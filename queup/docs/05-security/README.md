# Queup — Security

## Workstream: 05-security
## Chat purpose: Authentication, data protection, GDPR compliance, PCI DSS, app security

### Dependencies
- `00-master/README.md`
- `00-master/claude-project-instructions.md`
- `04-technical/README.md`

### Outputs expected from this chat
- `05-security/auth-design.md` ✓ complete
- `05-security/gdpr-compliance.md` ✓ complete
- `05-security/pci-dss-notes.md` ✓ complete
- `05-security/security-checklist.md` ✓ complete

### Key areas to cover

**Authentication**
- Supabase Auth handles JWT issuance and refresh
- All backend routes require valid JWT in Authorization header
- Vendor routes require vendor role claim
- Manager routes require admin role claim
- Social auth (Google, Apple) via Supabase OAuth providers

**GDPR (UK GDPR post-Brexit)**
- Data collected: name, email, location history, order history, payment method tokens
- Location data: vendor location stored temporarily (30-second updates). Customer location never stored — only used for proximity queries in real time.
- Right to erasure: customers and vendors can delete their accounts. Deletion must cascade through orders, menus, and payment records appropriately.
- Data processor agreements needed with: Supabase, Stripe, Firebase, what3words
- Privacy policy must be presented at sign-up and accepted explicitly
- Age gate: users must confirm they are 18+ (or 13+ with parental consent) at sign-up

**PCI DSS**
- Queup never touches raw card data. Stripe handles all card input via their mobile SDK.
- This puts Queup in SAQ A scope (the lightest PCI compliance tier)
- Stripe webhook signatures must always be validated — already in codebase
- Stripe secret keys must never appear in client-side code

**App security**
- Certificate pinning for production API calls
- Jailbreak/root detection (optional but recommended for payment app)
- Rate limiting on all public API endpoints
- Input validation on all user-supplied fields
- SQL injection prevention (Supabase parameterised queries by default)
- Expo updates (OTA) must be signed

### Regulatory notes for UK
- ICO registration required if processing personal data — register at ico.org.uk
- Financial Conduct Authority (FCA): Queup does not hold customer funds directly (Stripe does), so FCA authorisation is likely not required, but confirm with a solicitor
- Food Standards Agency: Queup is a technology platform, not a food business, so FSA registration is the vendor's responsibility, not Queup's
