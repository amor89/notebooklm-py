# Queup — Costs

## Workstream: 09-costs
## Chat purpose: Build costs, running costs, unit economics, and financial planning

### Dependencies
- `00-master/README.md`
- `01-business-model/revenue-model.md` (when complete)

### Outputs expected
- `09-costs/build-costs.md`
- `09-costs/running-costs.md`
- `09-costs/unit-economics.md`
- `09-costs/financial-model.md`

### Known cost inputs

**One-off build costs**
- Claude Code subscription: ~£100/month during build (3–6 months)
- Apple Developer Account: £99/year
- Google Play Developer Account: £20 one-off
- Figma: £0–£45/month (free tier sufficient for MVP)
- Domain registration (queup.co.uk, queup.app): £15–£30/year
- Legal review (terms + privacy policy): £500–£1,500
- Trademark registration (2 classes): ~£340

**Monthly running costs (post-launch)**
- Backend hosting (Railway or Render): £15–£30/month
- Supabase (database): £0–£25/month depending on tier
- Firebase FCM: £0 at MVP scale
- Sentry error monitoring: £0 at free tier
- Claude Code (continued iteration): £100/month

**Variable costs (per transaction)**
- Stripe fee: 1.5% + 25p per transaction
- what3words: £0 under 25k calls/month; tiered pricing above

**Estimated total to launch MVP: £1,000–£2,500**
(excluding your time and your wife's design time)

### Unit economics to model
- Average order value (AOV): £12–£15
- Platform fee: 5% = £0.60–£0.75
- Stripe fee on £12: ~£0.43
- Net revenue per order: ~£0.17–£0.32
- Orders needed to cover £150/month running costs: ~470–880 orders/month
- At 5 vendors × 20 orders/day × 20 trading days: 2,000 orders/month
- Breakeven at current costs: ~5 active vendors trading regularly
