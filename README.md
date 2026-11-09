# TowPilot Dispatch – Schedule module

Spec and build notes for the **Schedule** section of the TowPilot DISPATCH dashboard: commission-only intake agents claim 2-hour coverage blocks and toggle online status; the routable list drives the Telnyx simultaneous ring group, with the Philippines BPO as fallback.

## Start here

- [docs/schedule-module-spec.md](docs/schedule-module-spec.md) – full spec: decisions, screens, data model, API, Telnyx integration, automation rules, UI styling, build order.

## Build order (from the spec)

1. Tables + migrations; cron that seeds 4 weeks of 2-hour blocks ahead.
2. `/api/routing/routable` and `/api/routing/surge` + Telnyx simultaneous ring with BPO fallback (unblocks the phone team).
3. Agent screen: online toggle + weekly grid + claim/release.
4. Admin Coverage + Agents screens.
5. Attribution + stats + commission rollup ($10 base, +$5 surge).
6. Surge detection + notifications + reminders.

## Ground rules

- Build inside the existing TowPilot DISPATCH dashboard; match its component library, auth, and database conventions. No new stack.
- Ask before changing any existing table.
- Agents are 1099 contractors: every alert is an optional opportunity, never a requirement. Keep copy framed that way.

## Open questions

- Max claims per agent per week?
- Separate payout export for commission (1099 reporting), or ride on existing provider payouts?
- Where is agent notification opt-in captured – existing onboarding or first login to Schedule?
