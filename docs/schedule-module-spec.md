# TowPilot Dispatch – Schedule Module Spec

> **For Claude Code:** Build the Schedule module described below inside the existing TowPilot DISPATCH dashboard. Follow the Build order section. Match the existing dashboard's component library, auth, and database conventions; do not introduce a new stack. Ask before changing any existing table.

## Purpose
Add a **Schedule** section to the existing TowPilot DISPATCH dashboard so commission-only intake agents can claim coverage shifts and toggle online status, and so admins always know who is available to take calls. Availability data drives the Telnyx ring group: inbound calls simultaneously ring agents who are online for the current block; the Philippines BPO is the fallback.

## Decisions
- Coverage blocks are **2 hours**; agents may stack consecutive blocks.
- Base commission: **$10 per dispatched job**.
- Surge bonus: **+$5 per dispatch** while surge is active.
- **No overnight/weekend bonus.** The BPO covers off hours; US agents claim the blocks they want. Revisit only if data shows local agents convert materially better than the BPO.
- Agents are 1099 contractors. Surge alerts and any notification are **optional opportunities, never requirements**, with no penalty for ignoring them. Keep all copy framed as opportunity ("Surge on – go online now to take calls"), never as a demand.

## Navigation
- New left-nav item **Schedule** (calendar icon), placed below Leads. Same active-state styling as existing items (light green pill, green icon).
- Agent role sees: My schedule, Online toggle, My stats.
- Admin role sees: Coverage, Agents, Settings.

## Roles
- **agent** – 1099 intake contractor. Can claim/release shifts, toggle online, view own stats.
- **admin** – TowPilot/DRIVE staff. Sees all agents, coverage, can force-offline, edit blocks, set bonuses.

## Core concepts
- **Coverage block** – a fixed 2-hour window on a weekly grid. Each block has a `target_agents` count and optional `bonus_pct`.
- **Claim** – an agent reserving a block. Blocks can be over-claimed up to `max_agents`.
- **Online status** – real-time toggle. An agent is *routable* only if `online = true` AND (they hold a claim for the current block OR `open_ring = true` on that block).
- **Routable list** – the set of phone numbers Telnyx should ring right now.
- **Surge** – a system state that sets `open_ring = true` on the current block and notifies all active agents; anyone who goes online is rung, claim or not.

## Screens

### 1. My schedule (agent)
- Header: large **Online / Offline** toggle, current block countdown, "3 agents online" indicator, surge banner when active.
- Weekly grid: 7 columns (days), rows = 2-hour blocks. Each cell shows claimed count vs target (e.g. 2/3), bonus badge if any, and a filled state if the agent holds it.
- Tap cell → claim / release. Confirm sheet shows block time, bonus, and "You'll be rung for calls during this block if you're online."
- Toggle week forward/back; claims allowed up to 4 weeks ahead (configurable).
- Empty-state copy: "Claim the blocks you want to work. Calls ring your phone only when you're online."

### 2. My stats (agent)
- Calls taken, jobs dispatched, conversion %, commission earned (this week / this month / all time), surge bonuses earned.
- Shift history list.

### 3. Coverage (admin)
- Same weekly grid as heat map: green = at/above target, amber = below, red = zero.
- Click a block → list of claimed agents, edit target/bonus/open_ring, send "shift open" push/SMS to network.
- "Online now" panel: live list of routable agents with phone, block, time online.
- Surge status indicator with manual start/stop.

### 4. Agents (admin)
- Table: name, phone, status (invited / active / suspended), online, this-week claims, conversion %, commission owed.
- Actions: invite (email/SMS), suspend, force offline, edit phone.

### 5. Settings (admin)
- Block length (default 120 min), max claims per week per agent, claim horizon, surge thresholds (see Automation rules), surge bonus amount (default $5), base commission (default $10), BPO fallback timeout (seconds before call rolls to BPO).

## Data model

```sql
agents (
  id uuid pk,
  user_id uuid fk users,
  phone text not null,
  status text check (status in ('invited','active','suspended')),
  online boolean default false,
  online_since timestamptz,
  timezone text,
  created_at timestamptz
)

coverage_blocks (
  id uuid pk,
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  target_agents int default 2,
  max_agents int default 5,
  bonus_pct numeric default 0,
  open_ring boolean default false,
  unique (starts_at)
)

block_claims (
  id uuid pk,
  block_id uuid fk coverage_blocks,
  agent_id uuid fk agents,
  claimed_at timestamptz,
  released_at timestamptz,
  unique (block_id, agent_id)
)

agent_status_events (
  id bigserial pk,
  agent_id uuid fk agents,
  online boolean,
  source text,  -- 'app','admin','auto'
  created_at timestamptz
)

surge_events (
  id uuid pk,
  block_id uuid fk coverage_blocks,
  trigger text,        -- 'low_coverage','bpo_rollover','unanswered','manual'
  started_at timestamptz,
  ended_at timestamptz
)

agent_call_attributions (
  id uuid pk,
  agent_id uuid fk agents,
  call_id text,        -- Telnyx call control id
  job_id uuid null,    -- set when a job is created from the call
  answered boolean,
  dispatched boolean,
  surge boolean default false,
  commission_amount numeric,
  created_at timestamptz
)
```

## API

```
GET    /api/schedule/blocks?from=&to=          -> blocks with claim counts, my_claim flag
POST   /api/schedule/blocks/:id/claim
DELETE /api/schedule/blocks/:id/claim
POST   /api/agents/me/online   { online: true|false }
GET    /api/agents/me/stats?range=week|month|all

GET    /api/routing/routable                   -> [{ agent_id, phone }] for the current moment
GET    /api/routing/surge                      -> { active: bool, since, trigger }

ADMIN
GET    /api/admin/coverage?from=&to=
PATCH  /api/admin/blocks/:id  { target_agents, max_agents, bonus_pct, open_ring }
POST   /api/admin/blocks/:id/notify            -> push/SMS "shift open"
POST   /api/admin/surge/start   /   POST /api/admin/surge/stop
GET    /api/admin/agents
PATCH  /api/admin/agents/:id { status, phone, online }
GET    /api/admin/settings  /  PATCH
```

`GET /api/routing/routable` is the single integration point for the phone system. Logic:
1. `online = true`
2. `status = active`
3. holds an unreleased claim on the block covering `now()` OR that block has `open_ring = true`
4. not currently on another call (optional, via Telnyx state)

## Telnyx integration
- On inbound call: call `/api/routing/routable`, start a simultaneous ring to all returned numbers with `fallback_timeout` (settings, default 20s).
- No answer → transfer to BPO queue and record a `bpo_rollover` event (feeds surge detection).
- On answer: write `agent_call_attributions` with `answered = true`, `surge = <surge active?>`. When a job is created from that call in New job, link `job_id`, set `dispatched = true`, and set `commission_amount = base + (surge ? surge_bonus : 0)`.
- Webhooks: `call.answered`, `call.hangup` → update attribution and agent busy state.

## Realtime
- WebSocket/SSE channel `schedule:coverage` broadcasts claim, online, and surge changes so the admin "Online now" panel, agent counts, and surge banner update live.

## Automation rules
- **Surge**: triggered automatically when (a) routable count < target for the current block, (b) a call rings out and rolls to the BPO fallback, or (c) N calls in the last 15 min went unanswered by agents (threshold in settings). On trigger: set `open_ring = true` on the current block so any active agent who goes online is rung, even without a claim; SMS/push all active agents: "Surge on – go online now to take calls. +$5 per dispatch while surge is active." Surge ends when routable count ≥ target for 15 min or no BPO rollovers for 30 min. Admin can start/stop manually.
- **Auto-offline**: if an agent is online but misses 3 consecutive rings, set offline and notify them so they're not holding a slot they can't answer.
- **Reminder**: 30 min before a claimed block, push/SMS reminder.
- All notifications require the agent's opt-in at onboarding (surge alerts by SMS and automated call), log the consent timestamp, honor STOP, and respect quiet hours in the agent's timezone unless they opted in to overnight alerts.

## UI styling (match existing dashboard)
- White cards on #F7F8F9 background, 1px #E6E8EA borders, 12–16px radius.
- Brand green ~#1D9E75 for primary buttons, active nav, claimed cells, online toggle.
- Inter / Plus Jakarta Sans, sentence-case buttons, black headings, gray #5B6168 helper text.
- Grid cells: 2/3 counter in bold, bonus as small green pill, claimed state = light green fill with green border.

## Build order
1. Tables + migrations; cron that seeds 4 weeks of 2-hour blocks ahead.
2. `/routable` and `/surge` endpoints + Telnyx simultaneous ring with BPO fallback (unblocks the phone team).
3. Agent screen: toggle + weekly grid + claim/release.
4. Admin Coverage + Agents screens.
5. Attribution + stats + commission rollup ($10 base, +$5 surge).
6. Surge detection + notifications + reminders.

## Open questions
- Max claims per agent per week?
- Does commission need a separate payout export (1099 reporting) or ride on existing provider payouts?
- Where does agent opt-in consent get captured — existing onboarding flow or the first login to Schedule?
