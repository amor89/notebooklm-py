# Queup — Business Model

## Workstream: 01-business-model
## Chat purpose: Define and model revenue streams, pricing, unit economics, and P&L projections

### Dependencies
- `00-master/README.md`
- `00-master/claude-project-instructions.md`

### Outputs expected from this chat
- `01-business-model/revenue-model.md`
- `01-business-model/pricing-strategy.md`
- `01-business-model/unit-economics.md`
- `01-business-model/projections-3yr.md`

### Context to include when starting this chat

Queup serves four vendor segments:
- Food trucks and stalls at festivals (seasonal, May–September peak)
- Fixed pitch food trucks at markets and business districts (year-round)
- Home food businesses — bakers, meal prep, home cooks (year-round)
- Pop-up caterers at private events

UK market:
- ~10,000 registered food trucks
- ~150,000–200,000 home food businesses
- ~25,000 market stalls operating regularly
- UK street food market valued at £1.2bn, growing ~7% per year

Revenue model options discussed:
- Transaction commission (5% per order — current default in codebase)
- Monthly vendor subscription (£20–£50/month)
- Event organiser licence (£200–£1,000 per event)
- Combination model

Key financial constraint: at £12 average order value, 3% commission (£0.36) is less than Stripe's fee (~£0.43). Commission must be at least 5% or supplemented with a per-order minimum fee.

### Questions to resolve in this chat
1. What is the right commission rate for launch?
2. At what scale does a subscription model make sense?
3. What is the 3-year revenue projection at 1% market penetration?
4. What is the break-even vendor count?
