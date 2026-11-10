# Point Claiming & Reward System — Concept Plan

## Context
A greenfield project (repo is empty) for a platform that awards points through **campaigns** using several claim types, and lets users redeem points for vouchers, physical items, digital items, and tiers/badges. This plan sets the domain model, flows, and build order before any code is written. Stack: **Node/TypeScript + PostgreSQL** (suggested: NestJS or Express + Prisma).

## Core Principles
1. **Ledger, not balance column** — every point change is an immutable `point_transactions` row; balance = SUM (cached in `wallets` and updated inside the same DB transaction).
2. **Idempotency** — every claim/redeem takes an `idempotency_key`; unique constraints stop double-claims.
3. **Campaign-driven** — earning rules live in data (campaign + rule config), not hardcoded.
4. **Expiry is configurable per campaign** (none / fixed date / N days after earn); redemption spends points FIFO, oldest first.

## Domain Model
| Table | Key fields |
|---|---|
| `users` | id, tier_id |
| `wallets` | user_id, balance, lifetime_earned (cache) |
| `campaigns` | id, name, status(draft/active/paused/ended), start_at, end_at, budget_points, points_issued, expiry_type, expiry_value |
| `campaign_rules` | campaign_id, claim_type, config JSONB, points, per_user_limit, total_limit, period(once/daily/weekly) |
| `claims` | id, user_id, rule_id, status(pending/approved/rejected), evidence JSONB, idempotency_key UNIQUE |
| `claim_codes` | code UNIQUE, rule_id, max_uses, used_count, expires_at |
| `point_transactions` | id, user_id, amount(+/-), type(earn/redeem/expire/adjust/refund), source_type, source_id, remaining (for FIFO earn lots), expires_at, created_at |
| `rewards` | id, type(voucher/digital/physical/tier/badge/raffle/ticket/donation/points_cash), cash_price, cost_points, stock (null = unlimited), per_user_limit, active window, metadata JSONB |
| `reward_inventory` | reward_id, payload (voucher code / gift-card key), assigned_to, assigned_at |
| `redemptions` | id, user_id, reward_id, points_spent, status(pending/fulfilled/shipped/cancelled/refunded), shipping JSONB, idempotency_key UNIQUE |
| `tiers` / `badges` / `user_badges` | thresholds by lifetime points or unlocked by redemption |

## Claim Types (pluggable strategy)
Interface: `ClaimHandler { validate(user, rule, input): Promise<void>; requiresApproval: boolean }`, registered by `claim_type`.
- `code` — redeem a promo/QR code (checks `claim_codes`)
- `purchase` — points from order amount (rate in config, triggered by an order webhook)
- `action` — check-in, referral, review, profile completion (triggered by an event)
- `manual` — admin grants or deducts points
- `submission` — user uploads proof (receipt, screenshot) → pending → admin approves
Adding a new type = one handler + config schema (validated with zod).

## Campaign Types (how users earn points)
Each campaign has a `campaign_type` and its own config (JSONB) checked by its handler.

### 1. Daily Check-in
- User taps "Claim" once per day; points go up with each day in a row (streak).
- Config: `{ "rewards_by_day": [10,10,20,20,30,30,100], "reset_on_miss": true, "timezone": "Asia/Phnom_Penh", "cycle_days": 7 }`
- Rules: one claim per user per calendar day (unique `user_id + campaign_id + claim_date`); missing a day resets the streak (or keeps it if `reset_on_miss=false`); day 7 bonus, then the cycle restarts.
- Extra table: `user_streaks` (user_id, campaign_id, current_streak, last_claim_date).

### 2. Purchase
- Points earned from orders, triggered by an order-paid webhook/event (no manual claim).
- Config: `{ "rate": 1, "per_amount": 1, "min_order": 5, "max_points_per_order": 500, "multiplier": 2, "categories": ["food"], "credit_after_days": 7 }`
- Rules: points = floor(order_amount / per_amount) × rate × multiplier (promo campaigns use multipliers such as "double points weekend"); the order id is the idempotency key; points are **pending** until the return window ends (`credit_after_days`), then become available; refund/cancel → reverse the points.
- Extra: `point_transactions.status` (pending/available/reversed).

### 3. Mission
- A task or a set of tasks; points when the target is reached. Examples: "Buy 3 times this week", "Complete your profile", "Write 2 reviews", "Spend $50 this month".
- Config: `{ "steps": [{ "event": "order.paid", "target": 3 }], "period": "weekly", "repeatable": false, "auto_claim": false }`
- Rules: events update progress; when every step is done the mission is "completed" → user taps Claim (or it is auto-claimed); progress resets each period when the mission is repeatable.
- Extra table: `mission_progress` (user_id, campaign_id, step_index, progress, target, status: in_progress/completed/claimed, period_key).

### 4. Referral
- User shares a referral code/link; both people get points when the friend qualifies.
- Config: `{ "referrer_points": 200, "referee_points": 100, "qualify_event": "first_order.paid", "min_order": 10, "max_referrals": 20, "qualify_within_days": 30 }`
- Rules: each user has one `referral_code`; a referee can only be referred once and only as a new user; reward only after the qualifying action (e.g. first paid order), not just sign-up; anti-fraud: block self-referral and repeat device/IP/phone, cap per referrer.
- Extra table: `referrals` (referrer_id, referee_id UNIQUE, code, status: pending/qualified/rewarded/rejected, qualified_at).

### Shared rules for all campaign types
Start/end date, total points budget, per-user limit, target audience (all / tier / new users), status (draft/active/paused/ended), points expiry.

## Reward Types (what users can claim with points)
| Reward type | Examples | How the user claims it | Fulfilment | Key config |
|---|---|---|---|---|
| Voucher / discount | % off, fixed amount off, free shipping | Redeem in catalogue → gets a code | Instant: code from `reward_inventory` or generated | discount value, min spend, valid days, single/multi use |
| Digital item | Gift card, e-wallet top-up, game item, subscription | Redeem → key/PIN shown + emailed | Instant (or via a provider API) | provider, denomination, key pool |
| Physical item | Merch, gadgets | Redeem → enter shipping address | Pending → packed → shipped → delivered (admin/3PL updates) | stock, variants (size/colour), shipping regions |
| Tier / membership | Silver, Gold, Platinum | Automatic at a lifetime-points threshold, or buy with points | Instant: update `users.tier_id` + perks | threshold, perks, duration |
| Badge / achievement | "First claim", "10 check-ins" | Automatic when a condition is met, or redeem | Instant: insert `user_badges` | condition, icon |
| Lucky draw / raffle entry | 1 entry = 100 points | Redeem → entry recorded | Winner picked at draw date | entries per user, draw date, prizes |
| Experience / event ticket | Event pass, workshop seat | Redeem → booking with QR ticket | Instant ticket, checked at venue | capacity, date, location |
| Donation / charity | Turn points into a donation | Redeem → donation receipt | Batched payout to the partner | partner, conversion rate |
| Points + cash | Item at 500 pts + $5 | Redeem → pay the cash part at checkout | After payment succeeds | points cost + cash price |

All types share one `rewards` table (`type` + `metadata` JSONB) and one `RewardFulfiller` interface per type, so a new reward type = one fulfiller + metadata schema.

## Flows
**Claim:** request → load active campaign + rule → handler.validate → check limits (per user/period, total, campaign budget) → in one DB tx: insert claim, insert earn transaction with expires_at, update wallet and campaign.points_issued (`SELECT … FOR UPDATE`) → emit `points.earned`.

**Redeem:** request → check reward active, stock, per-user limit, balance → in one tx: lock wallet, take earn lots FIFO (decrement `remaining`), insert redeem transaction, decrement stock / assign an inventory row, create redemption → fulfil by type:
- voucher/digital: assign code right away → fulfilled
- physical: pending → shipped (admin updates)
- tier/badge: grant right away
Cancel/refund → refund transaction (restores points) + stock back.

**Expiry job:** daily cron finds lots with `remaining > 0 AND expires_at < now` → expire transaction. Optional: notify users N days before.

**Tier recalculation:** on `points.earned`, compare lifetime_earned with tier thresholds.

## API (sketch)
- User: `GET /wallet`, `GET /wallet/transactions`, `GET /campaigns`, `POST /claims`, `GET /rewards`, `POST /redemptions`, `GET /redemptions`
- Admin: CRUD for campaigns/rules/rewards/codes, `POST /admin/claims/:id/approve|reject`, `POST /admin/points/adjust`, `PATCH /admin/redemptions/:id`, reports (points issued vs redeemed, liability)

## Anti-abuse
Rate limiting on claim/redeem; unique keys; campaign budget cap; per-device/IP limits for code claims; audit log for admin adjustments.

## Build Order
1. Schema + ledger/wallet service (earn/spend/refund with tests)
2. Campaigns + rules + `code` and `manual` claim types
3. Rewards catalogue + redemption (voucher/digital first)
4. Physical fulfilment + tiers/badges
5. `purchase`/`action`/`submission` types + approval queue
6. Expiry cron, notifications, admin reports

## Verification
- Unit tests: FIFO spend, expiry, limit checks, each claim handler.
- Concurrency tests: 50 parallel redeems on stock=1 → exactly 1 succeeds; same idempotency key twice → one transaction.
- Invariant check: `wallet.balance == SUM(point_transactions.amount)` for every user.
- E2E: create campaign → claim code → redeem voucher → cancel → balance restored.

## Open Questions
- Default expiry policy (you answered "Other" — currently per campaign).
- Can points go negative (e.g. a refunded purchase after points are spent)?
- Who approves submissions, and is there an SLA?
