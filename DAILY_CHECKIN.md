# Daily Check-in — Standalone Module

A self-contained module: users tap "Claim" once a day and get a reward that grows with their streak. It does not need the rest of the loyalty system, so any project can use it on its own.

## How it works
- The user taps "Claim" once per calendar day.
- Each day in a row moves the streak up one step: day 1 → 10, day 2 → 10 … day 7 → 100.
- After the last day the cycle starts again at day 1.
- Missing a day resets the streak to day 1 (or keeps it, if `reset_on_miss = false`).

## Config
```json
{
  "rewards_by_day": [10, 10, 20, 20, 30, 30, 100],
  "reset_on_miss": true,
  "timezone": "Asia/Phnom_Penh",
  "start_at": "2026-12-01",
  "end_at": null
}
```
- The cycle length is `rewards_by_day.length`; there is no separate `cycle_days` setting.
- A reward value is a number by default. It can also be an object (`{ "type": "coupon", "id": "..." }`) when the host project gives something other than points.

## Data
| Table | Fields |
|---|---|
| `checkin_campaigns` | id, config (JSON), status (draft/active/paused/ended) |
| `checkin_streaks` | user_id, campaign_id, current_day, last_claim_date — unique (user_id, campaign_id) |
| `checkin_claims` | id, user_id, campaign_id, claim_date, day_index, reward — **unique (user_id, campaign_id, claim_date)** |

## Claim logic
1. `today` = the current date in the config `timezone`.
2. Load the streak. If `last_claim_date = today`, the claim is refused with "Already claimed today".
3. If `last_claim_date = yesterday`, then `day = current_day + 1`. Otherwise `day = 1` (or `current_day + 1` when `reset_on_miss = false`).
4. If `day > rewards_by_day.length`, then `day = 1`.
5. In one DB transaction: insert into `checkin_claims` and upsert `checkin_streaks`. The unique key blocks double taps.
6. Call the host project's `onReward(user, reward)` hook. The host decides what a reward is: points, a coupon or coins.

## Integration (how a project plugs it in)
- **Input:** an authenticated `user_id`.
- **Output:** the `onReward` hook, plus the `checkin.claimed` event.
- **API:** `GET /checkin/status` (current day, claimed today?, the 7-day strip), `POST /checkin/claim`.

## End-user friendliness
- Show the reward strip with the days done, today, and upcoming days.
- Tell the user clearly when a missed day resets the streak.
- Optional: a reminder notification before the day ends.

## Tests to write
- Claiming twice on the same day gives one row.
- A missed day resets the streak; with `reset_on_miss = false` it does not.
- After day 7 comes day 1.
- A claim at 23:59 and the next at 00:01 (in the campaign timezone) count as two days in a row.
