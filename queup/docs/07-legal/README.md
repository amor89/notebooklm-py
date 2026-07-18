# Queup — Legal

## Workstream: 07-legal
## Chat purpose: Terms of service, privacy policy, trademark, IP protection, and regulatory compliance

### Dependencies
- `00-master/README.md`
- `05-security/gdpr-compliance.md` (when complete)

### Outputs expected
- `07-legal/terms-of-service-draft.md`
- `07-legal/privacy-policy-draft.md`
- `07-legal/trademark-research.md`
- `07-legal/ip-protection.md`
- `07-legal/vendor-agreement.md`

### Key legal questions to answer

**Trademark**
- Search "Queup" on the UK IPO register (ipo.gov.uk) before launch
- Register in Class 42 (Software as a service) and Class 43 (Food and drink services)
- Cost: ~£170 per class for UK trademark registration
- Timeline: ~4 months for registration

**IP and copyright**
- The codebase is proprietary. Do not open-source.
- The name "Queup", logo, and visual identity should be trademarked
- Consider whether the what3words integration constitutes any IP dependency risk
- Document authorship of original code (you) and design (your wife)

**Platform liability**
- Queup is a technology platform, not a food business
- Queup does not prepare, store, or deliver food
- Vendor is responsible for: food hygiene rating, public liability insurance, food safety compliance
- Terms of service must make this distinction explicit
- Consider requiring vendors to upload proof of food hygiene registration at onboarding

**Vendor agreement**
- Vendors agree to: platform fee structure, prohibited items policy, accuracy of menu information
- Queup reserves right to suspend vendors for: complaints, non-compliance, fraudulent activity
- Dispute resolution process for customer complaints about vendor

**Consumer law (UK)**
- Consumer Rights Act 2015 applies
- Customers have right to refund if order not fulfilled
- Refund process must be documented and automated where possible

### Immediate actions (before launch)
1. Trademark search for "Queup"
2. Register queup.co.uk and queup.app domains
3. Draft basic terms and privacy policy (this chat)
4. Have a solicitor review before going live (budget £500–£1,500)
5. Register with ICO as a data controller
