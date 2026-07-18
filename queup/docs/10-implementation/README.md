# Queup — Implementation

## Workstream: 10-implementation
## Chat purpose: Sprint plan, build milestones, App Store submission, and launch checklist

### Dependencies
- `00-master/README.md`
- `04-technical/README.md`
- `02-customer-journey/customer-app-flow.md`
- `03-design-ux/figma-structure.md`

### Outputs expected
- `10-implementation/sprint-plan.md`
- `10-implementation/milestone-tracker.md`
- `10-implementation/app-store-submission.md`
- `10-implementation/launch-checklist.md`
- `10-implementation/beta-test-plan.md`

### Build phases summary

**Phase 1 — Foundation (Months 1–2)**
- Rename repo from streetserve to queup
- Update all package names to @queup/shared, @queup/backend etc.
- Set up Supabase project with PostGIS enabled
- Run all database migrations
- Configure Stripe account and Stripe Connect
- Configure Firebase project
- Get what3words API key
- Backend fully deployed and health check passing

**Phase 2 — Vendor App (Months 3–4)**
- All vendor app screens built and connected to backend
- Location broadcast tested on real device
- Order queue working with Supabase realtime
- Stripe Connect onboarding flow working end-to-end
- Push notifications firing correctly

**Phase 3 — Customer App (Months 4–5)**
- All customer app screens built
- Map + what3words integration working
- Stripe checkout flow working end-to-end
- Order status updates working in real time
- Push notifications firing at each status change

**Phase 4 — Launch Prep (Month 6)**
- Beta test with 3–5 real vendors at a local market or festival
- Fix all critical bugs from beta
- App Store and Google Play submissions
- Terms of service and privacy policy live on website
- GDPR compliance verified
- Sentry error monitoring active

### App Store submission checklist
- [ ] Apple Developer Account active (£99/year)
- [ ] App icons at all required sizes (use Expo's icon generation)
- [ ] Screenshots for all required device sizes (iPhone 6.7", 6.5", 5.5")
- [ ] App description written (max 4,000 characters)
- [ ] Keywords researched and entered (max 100 characters)
- [ ] Privacy policy URL live and accessible
- [ ] Age rating questionnaire completed
- [ ] TestFlight beta tested with at least 5 users
- [ ] All App Store Review Guidelines checked

### Google Play submission checklist
- [ ] Google Play Developer Account active (£20 one-off)
- [ ] Feature graphic (1024 × 500px)
- [ ] Screenshots for phone and 7" tablet
- [ ] Short description (max 80 characters)
- [ ] Full description (max 4,000 characters)
- [ ] Content rating questionnaire completed
- [ ] Privacy policy URL live
- [ ] Internal testing track used before production release

### Timeline to first festival
To trade at a summer festival, submissions must begin by April at the latest.
App Store review takes 1–7 days on average but can take longer on first submission.
Budget 2 weeks of buffer for review rejections and resubmission.
