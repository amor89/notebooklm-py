# Queup — Database Schema

**Workstream:** 04-technical
**Status:** Complete (v1 — MVP schema)
**Engine:** PostgreSQL 15 (Supabase) + PostGIS
**Feeds into:** `05-security/gdpr-compliance.md`, `05-security/auth-design.md`, `06-integrations/supabase-postgis.md`

---

## 1. Conventions

- **Primary keys:** `uuid` via `gen_random_uuid()`.
- **Money:** integer **pence**. Never `numeric`, never `float`. `850` = £8.50.
- **Timestamps:** `timestamptz`, stored UTC. `created_at`/`updated_at` on every mutable table.
- **Enums:** Postgres `CHECK` constraints (not native enums) so values can evolve via migration without type churn.
- **Soft rules in code, hard rules in SQL:** every domain invariant that can be a constraint *is* one (FKs, `CHECK`, `NOT NULL`, unique). The DB is the last line of defence.
- **Migrations only:** schema changes are additive migration files in `backend/src/db/migrations/`, numbered and forward-only. Never edit a shipped migration.
- **RLS:** Row-Level Security is enabled on every table holding user data (§10).

Enable extensions **before** any table migration:

```sql
create extension if not exists postgis;
create extension if not exists pgcrypto; -- gen_random_uuid()
```

---

## 2. Enumerated domains

| Domain | Allowed values | Notes |
|---|---|---|
| `vendor_mode` | `festival`, `fixed_pitch`, `popup`, `home_food` | First three = Segment A; `home_food` = Segment B |
| `location_type` | `w3w`, `postal` | `w3w` for Segment A, `postal` for Segment B |
| `order_status` | `pending`, `accepted`, `preparing`, `ready`, `collected`, `rejected` | One-directional; `rejected` only from `pending` |
| `vendor_status` | `pending_review`, `active`, `suspended` | Manager-controlled |
| `user_role` | `customer`, `vendor`, `admin` | Mirrors JWT role claim |

---

## 3. `customers`

Application profile for a buyer. Auth identity lives in Supabase `auth.users`; this row is keyed to it 1:1.

```sql
create table customers (
  id            uuid primary key references auth.users(id) on delete cascade,
  email         text not null unique,
  full_name     text not null,
  phone         text,
  home_postcode text,                         -- optional, for Segment B distance when GPS denied
  marketing_opt_in boolean not null default false,
  age_confirmed boolean not null default false, -- 18+ (or 13+ w/ consent) gate at sign-up
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);
```

- **No card data.** Payment methods live in Stripe; we store only the Stripe customer id (see `payment_events` / vendor mapping below), never a PAN.
- **Customer GPS is never stored.** `home_postcode` is a coarse, user-supplied fallback for Segment B distance, not live location.

---

## 4. `vendors`

The heart of the location model. Holds both location representations; a `CHECK` guarantees the right one is populated for the mode.

```sql
create table vendors (
  id             uuid primary key default gen_random_uuid(),
  owner_id       uuid not null references auth.users(id) on delete cascade,
  business_name  text not null,
  category       text not null,               -- burgers, pizza, bakery, ...
  description    text,
  hero_image_url text,
  dietary_tags   text[] not null default '{}', -- vegan, halal, gluten_free, ...

  vendor_mode    text not null check (vendor_mode in ('festival','fixed_pitch','popup','home_food')),
  location_type  text not null check (location_type in ('w3w','postal')),

  -- Segment A (w3w): transient current position only. NO history table.
  geom           geography(Point, 4326),      -- current GPS point, overwritten each update
  w3w_address    text,                        -- e.g. '///filled.count.soap'
  last_seen_at   timestamptz,                 -- when geom/w3w was last refreshed

  -- Segment B (postal): permanent address = personal data
  postal_address    jsonb,                    -- { line1, line2, city, postcode }
  postcode_centroid geography(Point, 4326),   -- derived from postcode for distance calc

  status         text not null default 'pending_review'
                   check (status in ('pending_review','active','suspended')),
  stripe_account_id text,                     -- Stripe Connect account
  stripe_onboarded  boolean not null default false,
  platform_fee_bps  int not null default 500, -- 5.00% = 500 basis points

  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now(),

  -- Location integrity: the mode dictates which representation is valid.
  constraint location_type_matches_mode check (
    (vendor_mode = 'home_food' and location_type = 'postal')
    or (vendor_mode in ('festival','fixed_pitch','popup') and location_type = 'w3w')
  ),
  -- Segment B must have a postal address; Segment A must not rely on one.
  constraint postal_requires_address check (
    location_type <> 'postal' or postal_address is not null
  )
);

create index vendors_geom_gix          on vendors using gist (geom);
create index vendors_centroid_gix      on vendors using gist (postcode_centroid);
create index vendors_status_idx        on vendors (status);
create index vendors_owner_idx         on vendors (owner_id);
```

**Why `geom` is a single overwritten point and not a history table:** Segment A vendor position is broadcast live but deliberately *not* retained. Each `/location/update` overwrites `geom`, `w3w_address`, `last_seen_at`. This is the data-minimisation stance the privacy policy commits to (see `05-security/gdpr-compliance.md`). Segment B `postal_address` is the opposite — permanent, and classed as personal data.

---

## 5. `menu_items`

```sql
create table menu_items (
  id           uuid primary key default gen_random_uuid(),
  vendor_id    uuid not null references vendors(id) on delete cascade,
  name         text not null,
  description  text,
  price_pence  int not null check (price_pence >= 0),   -- integer pence, never float
  category     text not null default 'Mains',           -- Mains, Sides, Drinks, Extras
  dietary_tags text[] not null default '{}',
  image_url    text,
  is_available boolean not null default true,           -- out-of-stock toggle
  sort_order   int not null default 0,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);

create index menu_items_vendor_idx on menu_items (vendor_id);
```

---

## 6. `orders` and `order_items`

Server-authoritative totals; the client-supplied total is never trusted.

```sql
create table orders (
  id                uuid primary key default gen_random_uuid(),
  customer_id       uuid not null references customers(id) on delete restrict,
  vendor_id         uuid not null references vendors(id)   on delete restrict,

  status            text not null default 'pending'
                      check (status in ('pending','accepted','preparing','ready','collected','rejected')),
  collection_code   text,                                  -- 4-char, set server-side on payment success

  subtotal_pence    int not null check (subtotal_pence >= 0),
  platform_fee_pence int not null check (platform_fee_pence >= 0),
  total_pence       int not null check (total_pence >= 0),

  payment_intent_id text,                                  -- Stripe PaymentIntent
  payment_confirmed boolean not null default false,

  -- Segment B pre-order slot (null for Segment A "order now")
  slot_id           uuid references collection_slots(id) on delete set null,

  special_instructions text,
  idempotency_key   text,                                  -- client-supplied, dedupes creation

  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now(),

  constraint uq_order_idem unique (customer_id, idempotency_key),
  constraint total_is_subtotal_plus_fee check (total_pence = subtotal_pence + platform_fee_pence)
);

create index orders_vendor_status_idx on orders (vendor_id, status);
create index orders_customer_idx      on orders (customer_id);
create unique index orders_active_code_uq
  on orders (vendor_id, collection_code)
  where status in ('pending','accepted','preparing','ready');

create table order_items (
  id              uuid primary key default gen_random_uuid(),
  order_id        uuid not null references orders(id) on delete cascade,
  menu_item_id    uuid not null references menu_items(id) on delete restrict,
  name_snapshot   text not null,        -- name at time of order (menu may change later)
  price_pence     int not null check (price_pence >= 0), -- price at time of order
  quantity        int not null check (quantity > 0),
  special_instructions text
);

create index order_items_order_idx on order_items (order_id);
```

- **Price/name snapshots** on `order_items` freeze what the customer actually bought, independent of later menu edits.
- **`orders_active_code_uq`** enforces that a collection code is unique among a vendor's *active* orders; once `collected`/`rejected` the code can be reused.
- **Status graph is enforced in the service layer** (illegal transition → `409`); see §8.

---

## 7. Supporting tables

### 7.1 `collection_slots` (Segment B pre-order)

```sql
create table collection_slots (
  id           uuid primary key default gen_random_uuid(),
  vendor_id    uuid not null references vendors(id) on delete cascade,
  slot_start   timestamptz not null,
  slot_end     timestamptz not null,
  capacity     int not null check (capacity > 0),
  booked_count int not null default 0 check (booked_count >= 0),
  created_at   timestamptz not null default now(),
  constraint slot_time_valid check (slot_end > slot_start),
  constraint slot_not_overbooked check (booked_count <= capacity)
);
create index collection_slots_vendor_time_idx on collection_slots (vendor_id, slot_start);
```

### 7.2 `payment_events` (Stripe webhook idempotency + audit)

```sql
create table payment_events (
  id             uuid primary key default gen_random_uuid(),
  stripe_event_id text not null unique,        -- dedupe: replayed events skipped
  event_type     text not null,               -- payment_intent.succeeded, charge.refunded, ...
  order_id       uuid references orders(id) on delete set null,
  payload_digest text,                         -- hash of payload, not the raw PII
  processed_at   timestamptz not null default now()
);
```

### 7.3 `devices` (push tokens)

```sql
create table devices (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references auth.users(id) on delete cascade,
  fcm_token   text not null,
  platform    text not null check (platform in ('ios','android')),
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now(),
  constraint uq_device_token unique (user_id, fcm_token)
);
```

### 7.4 `events` (manager geofenced zones)

```sql
create table events (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  boundary    geography(Polygon, 4326),        -- geofence for organiser view
  starts_at   timestamptz,
  ends_at     timestamptz,
  created_by  uuid references auth.users(id) on delete set null,
  created_at  timestamptz not null default now()
);
create index events_boundary_gix on events using gist (boundary);
```

---

## 8. Order status transition rule

Enforced in `orders` service before any `UPDATE`:

```
pending ──▶ accepted ──▶ preparing ──▶ ready ──▶ collected   (terminal)
   │
   └──▶ rejected   (terminal; triggers Stripe refund)
```

- Forward-only. No `ready → preparing`, no reopening `collected`.
- `rejected` is reachable only from `pending`.
- Any other transition → `409 Conflict`, no state change.

A DB trigger mirrors this as defence-in-depth so a direct SQL write can't corrupt the graph:

```sql
create or replace function enforce_order_status() returns trigger as $$
begin
  if old.status = new.status then return new; end if;
  if not (
       (old.status='pending'   and new.status in ('accepted','rejected'))
    or (old.status='accepted'  and new.status='preparing')
    or (old.status='preparing' and new.status='ready')
    or (old.status='ready'     and new.status='collected')
  ) then
    raise exception 'illegal order status transition: % -> %', old.status, new.status;
  end if;
  return new;
end $$ language plpgsql;

create trigger trg_order_status
  before update of status on orders
  for each row execute function enforce_order_status();
```

---

## 9. PostGIS RPC — nearby vendors (Segment A)

Exposed to the backend as a Supabase RPC. Coordinate-based radius scan for `w3w` vendors that are `active` and recently seen.

```sql
create or replace function vendors_within_radius(
  in_lat double precision,
  in_lng double precision,
  in_radius_m integer default 500
)
returns table (
  id uuid, business_name text, category text, w3w_address text,
  distance_m double precision, last_seen_at timestamptz
)
language sql stable as $$
  select v.id, v.business_name, v.category, v.w3w_address,
         st_distance(v.geom, st_makepoint(in_lng, in_lat)::geography) as distance_m,
         v.last_seen_at
  from vendors v
  where v.location_type = 'w3w'
    and v.status = 'active'
    and v.geom is not null
    and v.last_seen_at > now() - interval '10 minutes'   -- only "live" vendors
    and st_dwithin(v.geom, st_makepoint(in_lng, in_lat)::geography, in_radius_m)
  order by distance_m asc;
$$;
```

Segment B discovery does **not** use this RPC — it's an ordinary query ordered by `st_distance(postcode_centroid, customer_centroid)` converted to miles at the display edge, no radius scan, no live GPS.

---

## 10. Row-Level Security (defence-in-depth)

RLS is enabled on every user-data table. The backend uses the service role for trusted writes; RLS still protects any path that runs under a user JWT (including Supabase Realtime). Representative policies:

```sql
alter table customers   enable row level security;
alter table vendors     enable row level security;
alter table orders      enable row level security;
alter table order_items enable row level security;
alter table devices     enable row level security;

-- A customer sees only their own profile.
create policy customer_self on customers
  for select using (id = auth.uid());

-- A customer sees only their own orders; a vendor sees only orders for their vendor.
create policy order_visibility on orders
  for select using (
    customer_id = auth.uid()
    or vendor_id in (select id from vendors where owner_id = auth.uid())
  );

-- Only the vendor owner can mutate their own menu / vendor row.
create policy vendor_owner_write on vendors
  for update using (owner_id = auth.uid());

-- A user manages only their own device tokens.
create policy device_self on devices
  for all using (user_id = auth.uid());

-- Public discovery: active vendors are readable (read-only) by anon/auth for the map & list.
create policy vendors_public_read on vendors
  for select using (status = 'active');
```

Full role model and JWT claims → `05-security/auth-design.md`. Manager/admin access is granted by the `admin` role claim and dedicated policies, never by exposing the service key to the browser.

---

## 11. Data classification (for GDPR)

| Table / column | Class | Retention |
|---|---|---|
| `customers.email`, `full_name`, `phone` | Personal | Life of account; erased on deletion |
| `customers.home_postcode` | Personal (coarse) | Life of account |
| Customer live GPS | **Never stored** | n/a |
| `vendors.geom` / `w3w_address` (Segment A) | Transient location | Overwritten each update; not historised |
| `vendors.postal_address` (Segment B) | **Personal** | Life of vendor account |
| `orders`, `order_items` | Personal + transactional | Retained for tax/accounting (6 yrs) then anonymised |
| `payment_events` | Transactional (no PAN) | Retained for reconciliation |
| Card data (PAN/CVV) | **Never stored** — Stripe only | n/a |

This table is the input to `05-security/gdpr-compliance.md` §data-map and drives the erasure cascade design there.

---

## 12. Referential integrity summary

- `customers.id` → `auth.users.id` (cascade delete): removing the auth user removes the profile.
- `vendors.owner_id` → `auth.users.id` (cascade).
- `orders.customer_id` / `vendor_id` → `restrict`: an order can't be created against a missing party, and accounts with live orders are anonymised rather than hard-deleted (see erasure design in `05-security`).
- `order_items.order_id` → `orders` (cascade); `menu_item_id` → `restrict` (with `name_snapshot`/`price_pence` preserving the historical record even if the menu item is later removed via a nulling migration).
