## Campaign Templates (how users earn points)
A campaign is a **template** that a shop or branch applies to itself. Each template has a `campaign_type` and its own config (JSONB) checked by its handler.

- **Template:** the type plus default settings, made once by the platform.
- **Campaign instance:** a template applied to a scope (`shop_id`, optional `branch_ids`; empty means every branch of the shop) with its own dates, budget and settings.
- Only activity at that shop/branch counts toward the instance (orders, missions, referrals).

Daily Check-in is **not** a template: it isn't tied to a shop or branch activity. It is a separate platform-level module, see [DAILY_CHECKIN.md](DAILY_CHECKIN.md).

### 1. Purchase
- Points earned from orders, triggered by an order-paid webhook/event (no manual claim).
- Config: `{ "rate": 1, "per_amount": 1, "min_order": 5, "max_points_per_order": 500, "multiplier": 2, "categories": ["food"], "credit_after_days": 7 }`
- Rules: points = floor(order_amount / per_amount) × rate × multiplier (promo campaigns use multipliers such as "double points weekend"); the order id is the idempotency key; points are **pending** until the return window ends (`credit_after_days`), then become available; refund/cancel → reverse the points.
- Extra: `point_transactions.status` (pending/available/reversed).

### 2. Mission
- A task or a set of tasks; points when the target is reached. Examples: "Buy 3 times this week", "Complete your profile", "Write 2 reviews", "Spend $50 this month".
- Config: `{ "steps": [{ "event": "order.paid", "target": 3 }], "period": "weekly", "repeatable": false, "auto_claim": false }`
- Rules: events update progress; when every step is done the mission is "completed" → user taps Claim (or it is auto-claimed); progress resets each period when the mission is repeatable.
- Extra table: `mission_progress` (user_id, campaign_id, step_index, progress, target, status: in_progress/completed/claimed, period_key).

### 3. Referral
- User shares a referral code/link; both people get points when the friend qualifies.
- Config: `{ "referrer_points": 200, "referee_points": 100, "qualify_event": "first_order.paid", "min_order": 10, "max_referrals": 20, "qualify_within_days": 30 }`
- Rules: each user has one `referral_code`; a referee can only be referred once and only as a new user; reward only after the qualifying action (e.g. first paid order), not just sign-up; anti-fraud: block self-referral and repeat device/IP/phone, cap per referrer.
- Extra table: `referrals` (referrer_id, referee_id UNIQUE, code, status: pending/qualified/rewarded/rejected, qualified_at).

### 4. Welcome Bonus
- One-time points when a new user signs up (or makes a first action) at the shop.
- Config: `{ "points": 100, "trigger": "account.created", "auto_claim": true, "claim_within_days": 30 }`
- Rules: once per user per campaign (idempotency key: user_id + campaign_id); only users created during the campaign window; anti-fraud: same device/phone rules as Referral.
- Can also be built as a Mission with a single `account.created` step; keep it as its own template only if you want a simpler admin setup.

### 5. Birthday
- Points given automatically in the user's birthday month or on the birthday.
- Config: `{ "points": 200, "grant_on": "birthday_month_start", "claim_within_days": 30, "min_account_age_days": 30 }`
- Rules: triggered by a daily scheduled job, not by a user action; once per user per year (idempotency key: user_id + campaign_id + year); the birthday can be edited only once (or locked after set) so users can't farm it; `min_account_age_days` stops new accounts from setting today's date.

### 6. Mini Game
- Users play a short game and win points (or a voucher). Types: spin wheel, scratch card, mystery box, lucky draw (random); tap/catch game, quiz (skill).
- Config: `{ "game": "spin_wheel", "plays_per_day": 1, "ticket_cost": 0, "prizes": [ {"points": 10, "weight": 50}, {"points": 50, "weight": 30}, {"points": 500, "weight": 2, "stock": 100}, {"type": "none", "weight": 18} ] }`
- Getting a turn: a free daily play, or tickets earned from other campaigns (e.g. "spend $20 → 1 spin").
- Flow: app calls `POST /minigame/{campaign_id}/play` → server checks plays left / tickets, picks the prize by weight, checks prize stock and budget, writes the play and the ledger row in one transaction → app plays the animation for that result.
- Receiving: Instant claim (`source = minigame`); voucher prizes go through Manual claim.
- Extra tables: `minigame_plays` (user_id, campaign_id, play_date, prize, idempotency_key UNIQUE), `minigame_tickets` (user_id, campaign_id, balance).
- Rules:
  - The server decides the result before the animation; the app never decides.
  - When a prize's stock runs out, its weight moves to the other prizes.
  - Show the odds in the app (e.g. "500 pts: 2%").
  - Paying money or points for a random game can count as gambling in some countries; keep plays free or earned until checked with local law.
  - Skill games: validate max possible score, min play time and a server-issued session token.
  - Caps: plays per day and points per user per day from all games.

### Not a campaign: Admin Adjustment
- A support/ops tool to add or remove points by hand (make up for a problem, fix a mistake). Not a template, not tied to a budget.
- Fields: `user_id`, `points` (+ or -), `reason` (required), `created_by` (admin id), optional `reference_id` (ticket/order).
- Rules: always credited immediately; every adjustment is logged in `point_transactions` with `source = admin_adjustment`; limit who can use it (role permission) and set a max per adjustment that needs a second approval.

### Not a campaign: Tier Multiplier (only if you have tiers)
- A modifier applied on top of any earning, e.g. Silver ×1.2, Gold ×1.5. Not a template.
- Rule: decide whether it applies to all sources or only Purchase (common: only Purchase).

## How Points Are Received (claim modes)
Every earning ends up in one of three modes:

| Mode | How it works | Used by |
|---|---|---|
| **Auto-credit** | Points go straight to the available balance, nothing to tap | Admin Adjustment, Welcome Bonus (when `auto_claim: true`), Mission (when `auto_claim: true`), Referral |
| **Pending → Available** | Points show as pending, become available after a waiting period; can be reversed while pending | Purchase (`credit_after_days`) |
| **Manual claim** | Reward is unlocked; user taps Claim to receive it; must claim before a deadline or it is lost | Mission (`auto_claim: false`), Daily Check-in, Birthday, Welcome Bonus (`auto_claim: false`) |

Per-type summary:

| Type | Trigger | Claim mode | Reversible? |
|---|---|---|---|
| Purchase | order paid (webhook) | Pending → Available | Yes, on refund/cancel |
| Mission | all steps completed | Manual claim or auto-credit | Optional, if the progress order is refunded |
| Referral | friend's qualifying action | Auto-credit (both users) | Optional, if the qualifying order is refunded |
| Welcome Bonus | sign-up / first action | Auto-credit or manual claim | No |
| Birthday | scheduled job | Manual claim (or auto-credit) | No |
| Daily Check-in | user taps check-in | Instant claim (the tap is the claim, no deadline, see [DAILY_CHECKIN.md](DAILY_CHECKIN.md)) | No |
| Mini Game | user plays (free or ticket) | Instant claim (vouchers: manual claim) | No |
| Admin Adjustment | admin action | Auto-credit | Yes, by another adjustment |

### Data model
Two tables. Every point movement is a row in the ledger; rewards waiting for a tap live in `claimable_rewards`.

| Table | Fields |
|---|---|
| `point_transactions` (ledger, append-only) | id, user_id, points (+/-), status (pending/available/reversed/expired), source (purchase/mission/referral/welcome/birthday/checkin/admin_adjustment/redemption/expiry), campaign_id, reference_id, idempotency_key **UNIQUE**, available_at, expires_at, created_at |
| `claimable_rewards` | id, user_id, campaign_id, source, points, status (claimable/claimed/expired/cancelled), claimable_from, claimable_until, reference_id, idempotency_key **UNIQUE**, claimed_at, transaction_id |
| `campaign_budgets` | campaign_id, total_points, used_points (checked and updated in the same transaction as the credit) |

- **Balance** = sum of `points` where status = available (and not expired). **Pending** = sum where status = pending. Cache it in a `user_balances` row if reads get heavy, but the ledger is the source of truth.
- Never update or delete a ledger row's points. A correction is a new row (reversal, adjustment).

### Idempotency keys
| Source | Key |
|---|---|
| Purchase | `purchase:{order_id}` (reversal: `purchase_reversal:{order_id}:{refund_id}`) |
| Mission | `mission:{campaign_id}:{user_id}:{period_key}` |
| Referral | `referral:{referral_id}:referrer` and `referral:{referral_id}:referee` |
| Welcome Bonus | `welcome:{campaign_id}:{user_id}` |
| Birthday | `birthday:{campaign_id}:{user_id}:{year}` |
| Daily Check-in | `checkin:{campaign_id}:{user_id}:{claim_date}` |
| Admin Adjustment | `admin:{adjustment_id}` |

A repeated event or a double tap hits the UNIQUE key and returns the earlier result instead of crediting twice.

### Mode 1: Auto-credit
Flow:
1. Event arrives (referral qualified, mission completed with `auto_claim: true`, admin action...).
2. In one DB transaction: check the campaign is active, check the per-user limit, reserve the budget (`used_points + points <= total_points`), insert a `point_transactions` row with status = available.
3. Commit, then emit `points.credited` and send a notification ("You received 200 pts from Referral").

If the budget or limit check fails: no points, and the reason is logged (`credit_skipped` with reason) so support can explain it. Admin Adjustment skips budget checks.

### Mode 2: Pending → Available
Flow:
1. Order paid → insert a row with status = pending, `available_at = paid_at + credit_after_days`. Budget is reserved now, so it can't be overspent later.
2. A scheduled job (e.g. every hour) moves rows with `available_at <= now` and status = pending to available, and emits `points.available`.
3. Refund/cancel while pending → status = reversed, release the budget. Partial refund → reverse the full row and insert a new pending row for the remaining amount (same `available_at`).
4. Refund after available → insert a negative row (`source = purchase`, reversal key). If the balance would go negative, decide one policy: allow a negative balance (future earnings pay it back) **or** take only what's left and log the rest. Recommended: allow negative, block redemption while negative.

The app shows: "100 pts pending, available on Nov 17".

### Mode 3: Manual claim
Lifecycle of a `claimable_rewards` row:

```
           unlock event
               │
               ▼
   ┌──────── claimable ────────┐
   │  user taps Claim           │ claimable_until passes
   ▼                            ▼
claimed                      expired
(ledger row created)    (no points, notify optional)

claimable → cancelled  (campaign cancelled by admin, or the source order refunded before claim)
```

Unlock: when the reward is earned (mission completed, birthday job runs, welcome trigger), insert a `claimable_rewards` row with status = claimable and `claimable_until = now + claim_within_days`. **Budget is not reserved yet**, and the user is notified ("You have a reward to claim").

Claim (`POST /rewards/{id}/claim`), all in one DB transaction:
1. Lock the row (`SELECT ... FOR UPDATE`). It must belong to the user.
2. status must be claimable; otherwise return the matching error below. If it's already claimed, return the earlier result (idempotent).
3. `now <= claimable_until`; otherwise mark it expired and return `REWARD_EXPIRED`.
4. Campaign status is active or ended (an ended campaign still honors rewards unlocked before the end, until `claimable_until`); paused → `CAMPAIGN_PAUSED`.
5. Check the per-user limit and reserve the budget.
6. Insert a `point_transactions` row (status = available), then set the reward to claimed with `claimed_at` and `transaction_id`.
7. Commit, then emit `reward.claimed`.

Expiry job (daily): set rows with status = claimable and `claimable_until < now` to expired. Optional reminder push 3 days and 1 day before `claimable_until`.

Design choice: reserving the budget at **claim** time (above) means a user can unlock a reward and then find the budget used up. If that's unacceptable, reserve at unlock instead and release it when the reward expires or is cancelled. Pick one per platform; recommended: reserve at unlock for Birthday/Welcome (small, predictable), at claim for Mission.

### Mode 4: Instant claim (Daily Check-in)
The tap both earns and claims, so there is no `claimable_rewards` row. The check-in module calls `onReward(user, reward)`, and the host inserts a `point_transactions` row (status = available, `source = checkin`) using the check-in idempotency key. Missing a day loses nothing already claimed; it only resets the streak.

### Claim errors (API + app message)
| Code | When | App message |
|---|---|---|
| `ALREADY_CLAIMED` | reward already claimed (returns the earlier result, HTTP 200) | "Already claimed" |
| `REWARD_EXPIRED` | past `claimable_until` | "This reward expired on {date}" |
| `REWARD_CANCELLED` | campaign cancelled / source order refunded | "This reward is no longer available" |
| `BUDGET_EXHAUSTED` | campaign budget used up | "Rewards for this campaign are used up" |
| `USER_LIMIT_REACHED` | per-user limit hit | "You've reached the limit for this campaign" |
| `CAMPAIGN_PAUSED` | campaign paused | "This campaign is paused, try again later" |
| `NOT_ELIGIBLE` | user no longer in the target audience | "You're not eligible for this reward" |

### API
| Endpoint | Purpose |
|---|---|
| `GET /points/balance` | available, pending, expiring soon (points + date) |
| `GET /points/transactions?cursor=` | history with source, campaign name, status |
| `GET /rewards/claimable` | rewards waiting for a tap, sorted by `claimable_until` |
| `POST /rewards/{id}/claim` | claim one reward (idempotent) |
| `POST /admin/points/adjustments` | admin add/remove points (role-protected, reason required) |

### Points expiry
- Each available row gets `expires_at` (e.g. 12 months after `available_at`). Rolling expiry ("expire if no activity for 12 months") is the other option; pick one.
- When points are spent, use the oldest `expires_at` first (FIFO).
- A daily job writes a negative `source = expiry` row for the unspent part of expired rows.
- Show "250 pts expire on Dec 31" in the app, and remind users 30 and 7 days before.

### Events
`points.credited`, `points.pending`, `points.available`, `points.reversed`, `points.expired`, `reward.unlocked`, `reward.claimed`, `reward.expired`. These are used for notifications, analytics and the user's activity feed.

### Tests to write
- The same order webhook twice → one ledger row.
- Double tap Claim (two concurrent requests) → one credit, the second request gets `ALREADY_CLAIMED`.
- Two users claim the last budget at the same time → only one succeeds, the other gets `BUDGET_EXHAUSTED`.
- Claim one second after `claimable_until` → `REWARD_EXPIRED`.
- Refund while pending → reversed, budget released; refund after available → negative row.
- Pending job moves only rows with `available_at <= now`.
- Expiry uses the oldest points first.

### Shared rules for all campaign types
Scope (shop, branches), start/end date, total points budget, per-user limit, target audience (all / tier / new users), status (draft/active/paused/ended), points expiry.

### End-user friendliness
| Campaign | Friendly? | Watch out for |
|---|---|---|
| Purchase | Yes, nothing to do | Pending points confuse people ("I bought, where are my points?"). Show "100 pts pending, available on Nov 17" |
| Mission | Yes, if progress is visible | Always show a progress bar like "2/3 orders". Avoid missions with many steps |
| Referral | Yes | Explain that points come after the friend's first order, not at sign-up, or users will think it's broken |
| Welcome Bonus | Yes | If it's a manual claim, show it right after sign-up or users miss it |
| Birthday | Yes | Remind the user (push/notification) and show the claim deadline |

Rule of thumb: every rule that delays or blocks points needs a visible explanation in the app.
