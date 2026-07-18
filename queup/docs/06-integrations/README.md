# Queup — Integrations

## Workstream: 06-integrations
## Chat purpose: Detailed integration guides for every third-party service

### Dependencies
- `00-master/README.md`
- `04-technical/README.md`

### Outputs expected
- `06-integrations/stripe-connect.md`
- `06-integrations/what3words.md`
- `06-integrations/firebase-fcm.md`
- `06-integrations/supabase-postgis.md`
- `06-integrations/figma-api.md`
- `06-integrations/expo-location.md`

### Services to document

| Service | Purpose | Free tier |
|---|---|---|
| Stripe | Payments + vendor payouts | No monthly fee. 1.5% + 25p per transaction |
| Stripe Connect | Vendor onboarding + split payments | Included with Stripe |
| what3words | GPS → 3-word address conversion | 25,000 calls/month free |
| Firebase FCM | Push notifications | Free at MVP scale |
| Supabase | Database, auth, realtime | Free tier: 500MB DB, 50k MAU |
| Expo | React Native build + OTA updates | Free for small teams |
| Sentry | Error monitoring | Free tier: 5k errors/month |

### Critical integration notes
- Stripe Connect requires vendors to complete identity verification before receiving payouts
- what3words API key must be restricted to server-side use only (never in client bundle)
- Firebase requires `google-services.json` (Android) and `GoogleService-Info.plist` (iOS) in Expo config
- Supabase realtime subscriptions power live order queue updates in vendor app
- Expo location background mode must be declared in `app.json` with usage description strings
