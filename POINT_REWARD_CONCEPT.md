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

### End-user friendliness
| Campaign | Friendly? | Watch out for |
|---|---|---|
| Daily check-in | Yes, a single tap | Show the 7-day streak visually, and tell users when a missed day resets it |
| Purchase | Yes, nothing to do | Pending points confuse people ("I bought, where are my points?"). Show "100 pts pending, available on Nov 17" |
| Mission | Yes, if progress is visible | Always show a progress bar like "2/3 orders". Avoid missions with many steps |
| Referral | Yes | Explain that points come after the friend's first order, not at sign-up, or users will think it's broken |

Rule of thumb: every rule that delays or blocks points needs a visible explanation in the app.
