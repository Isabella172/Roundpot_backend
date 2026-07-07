# Roundpot — Backend Architecture

A digital susu / rotating savings group (ROSCA) platform built with Django + Django REST Framework. This document is the architectural source of truth.

---

## 1. Tech Stack

- **Framework:** Django + Django REST Framework
- **Database:** PostgreSQL (money and state machines demand real transactions — don't use SQLite beyond local prototyping)
- **Async tasks / scheduling:** Celery + Redis (broker + result backend)
- **State machines:** `django-fsm` (or a hand-rolled transition mixin — see §5)
- **Payments:** MTN MoMo Collections + Disbursements API (sandbox first)
- **Auth:** JWT (`djangorestframework-simplejwt`)
- **Docs:** `drf-spectacular` for OpenAPI schema
- **Money handling:** `Decimal` everywhere, stored in minor units (pesewas) as integers — never floats
- **Timezone:** all timestamps stored in UTC; Ghana (Africa/Accra, GMT, no DST) computed at display/business-logic layer

**Branding note:** the product name is **Roundpot**. Internally the model is called `Circle` (a group of members), but every `Circle` a user creates is presented to them in the UI/copy as "your pot" — e.g. "Create a Pot," "3 of 5 paid into this round's pot," "Your pot is ready." Keep the model name `Circle` in code (it's the clearer engineering term), but reserve "pot" for user-facing language.

---

## 2. Django Apps — What Each One Does

Nine apps, each with a single responsibility. Keep money logic (`ledger`, `payments`) isolated from everything else — that boundary is what keeps this system auditable. Namespace them under a `roundpot` project (e.g. `roundpot/apps/groups`, `roundpot/apps/ledger`) so the Django project itself is named `roundpot`, not a generic `backend` or `core`.

| # | App | Responsibility |
|---|-----|-----------------|
| 1 | `accounts` | Custom user model, auth, KYC fields, phone/momo verification |
| 2 | `groups` | Circle creation, membership, owner-configurable rules |
| 3 | `cycles` | A full rotation of a group; generates the round schedule |
| 4 | `contributions` | Per-member, per-round payment obligations + their state machine |
| 5 | `payouts` | Disbursement of a round's pot to the scheduled recipient |
| 6 | `payments` | MTN MoMo integration — Collections, Disbursements, webhooks, reconciliation |
| 7 | `ledger` | Append-only double-entry bookkeeping — the single source of truth for all money movement |
| 8 | `reputation` | Trust score computation, tiers, unlock rules |
| 9 | `disputes` | Raised complaints, evidence, admin resolution |
| 10 | `notifications` | Reminders, alerts, multi-channel dispatch (in-app, SMS, push later) |

Each app gets its own `models.py`, `serializers.py`, `views.py`, `permissions.py`, `services.py` (business logic lives here, not in views), and `tasks.py` (Celery jobs). Keep views thin — they call services, services call the ledger.

---

## 3. Core Data Model (by app)

### `accounts`
- `User` (extends `AbstractUser`): phone number (unique, verified flag), national ID type + number, momo number, momo verified flag, date joined.
- `KYCDocument`: user FK, document type, file, verification status, reviewed_by, reviewed_at.

### `groups`
- `Circle`: name, description, owner (FK User), contribution_amount (integer, pesewas), currency (default GHS), frequency (`weekly` / `monthly`), cycle_size (auto-derived from member count, or set explicitly), rotation_method (`fixed` / `random_draw` / `auction`), visibility (`private` / `invite_only` / `discoverable`), status (`forming` / `active` / `completed` / `disbanded`).
- **Owner-configurable rule fields** (this is the part you specifically asked for — see §6 for full logic):
  - `payment_window_hours` (default 24) — how long a member has to pay once a round opens.
  - `payout_delay_days` (default 0) — how many days after full collection before the owner-set payout actually fires. Lets an owner say "collect by day 1, pay out on day 3" instead of instant payout.
  - `grace_period_hours` (default 0) — extra buffer after the window closes before a payment is officially "late."
  - `penalty_type` (`flat` / `percentage` / `escalating`) and `penalty_value`.
  - `max_penalty_cap` (percentage of contribution — prevents runaway penalties).
  - `buffer_pool_percentage` (default 0) — % skimmed from every contribution into a shared shortfall-cover pool.
  - `business_days_only` (bool) — if true, due dates that land on a weekend/public holiday roll to the next business day.
- `Membership`: user FK, circle FK, rotation_position (nullable until assigned), status (`invited` / `active` / `defaulted` / `exited`), guarantor (nullable self-FK to another Membership), joined_at.

### `cycles`
- `Cycle`: circle FK, cycle_number, status (`forming` / `active` / `completed` / `at_risk`), started_at, expected_end_at.
- `Round`: cycle FK, round_number, recipient (FK Membership), window_opens_at, window_closes_at (= opens_at + payment_window_hours), payout_due_at (= closes_at + payout_delay_days), status (`upcoming` / `collecting` / `ready` / `disbursing` / `completed` / `partial`).

### `contributions`
- `Contribution`: round FK, member FK, amount_due, amount_paid, status (`pending` / `paid` / `late_paid` / `overdue` / `defaulted`), due_at, paid_at (nullable), penalty_applied (amount), penalty_waived_by (nullable admin FK), payment (nullable FK to `payments.Transaction`).
- `ContributionStatusLog`: contribution FK, from_status, to_status, changed_by (`system` / user FK), reason, timestamp — **never overwrite history, always append.**

### `payouts`
- `Payout`: round FK (one-to-one), recipient FK, expected_amount, actual_amount, shortfall_covered_by_buffer (amount), status (`scheduled` / `collecting` / `ready` / `disbursing` / `completed` / `partial` / `failed`), disbursed_at, transaction (FK to `payments.Transaction`).

### `payments`
- `Transaction`: type (`collection` / `disbursement`), direction, amount, provider (`mtn_momo`), provider_reference, idempotency_key (unique), status (`initiated` / `pending` / `confirmed` / `failed` / `reversed`), raw_webhook_payload (JSON, for audit), created_at, confirmed_at.
- `WebhookEvent`: raw payload, signature_verified (bool), processed (bool), received_at — log every webhook hit before processing, so nothing is ever silently lost.

### `ledger`
- `LedgerAccount`: owner_type (`user` / `circle` / `platform_buffer` / `platform_fee`), owner_id, account_type (`wallet` / `escrow` / `buffer_pool`), balance is **never stored directly** — always derived by summing entries.
- `LedgerEntry`: transaction reference, debit_account FK, credit_account FK, amount, description, created_at. Immutable — no update, no delete, ever. Every contribution, payout, penalty, and refund produces exactly one balanced entry pair here.

### `reputation`
- `ReputationScore`: user FK (one-to-one), score (0–100), tier (`bronze` / `silver` / `gold` / `platinum`), on_time_rate, cycles_completed, defaults_count, last_recalculated_at.
- `ReputationEvent`: user FK, event_type (`on_time_payment` / `late_payment` / `default` / `cycle_completed` / `dispute_lost`), score_delta, timestamp.

### `disputes`
- `Dispute`: raised_by FK, against_member FK (nullable), circle FK, contribution FK (nullable), type, description, evidence (file/JSON), status (`open` / `under_review` / `resolved` / `dismissed`), resolved_by, resolution_notes.

### `notifications`
- `Notification`: user FK, type, channel (`in_app` / `sms` / `push`), payload, sent_at, read_at, delivery_status.

---

## 4. Owner-Configurable Payout & Payment Window System

This is the mechanism you specifically want — walked through end to end.

**When a round opens:**
1. `Round.window_opens_at` is set (either at cycle start, or immediately after the previous round's payout completes).
2. `window_closes_at = window_opens_at + circle.payment_window_hours` (owner sets this per circle — default 24 hours, but an owner running a monthly circle might set 72 hours; a fast weekly circle might set 12).
3. Every member gets a `Contribution` row with `due_at = window_closes_at`.
4. `payout_due_at = window_closes_at + circle.grace_period_hours + circle.payout_delay_days` — this is the field that answers "owner can fix the number of days the [recipient] will take the money." An owner can set `payout_delay_days = 0` for instant payout the moment collection completes, or `payout_delay_days = 2` to build in a manual review buffer before money moves.

**Celery beat jobs (run every few minutes, not hourly — timing precision matters for money):**
- `check_overdue_contributions`: any `Contribution` past `due_at` and still `pending` → transitions to `overdue`, penalty calculation triggers (§6).
- `check_round_ready`: any `Round` where all contributions are `paid`/`late_paid` → transitions to `ready`, schedules payout at `payout_due_at`.
- `trigger_scheduled_payouts`: any `Payout` where `now() >= payout_due_at` and status is `ready` → fires disbursement.
- `reconcile_pending_transactions`: any `Transaction` stuck in `pending` beyond a threshold (e.g., 10 minutes) → actively queries MTN's status endpoint instead of waiting for a webhook that may never arrive.

---

## 5. State Machines

Enforce these at the model layer (django-fsm or a custom transition mixin) so an invalid jump — e.g. `pending` straight to `completed` — is impossible even if a bug tries to force it.

**Contribution:**
`pending → paid` (happy path)
`pending → overdue → late_paid` (paid within grace period, penalty may apply)
`pending → overdue → defaulted` (grace period expires, never paid)
`defaulted → paid` (admin manually resolves after the fact — must still be possible, real life is messy)

**Round:**
`upcoming → collecting → ready → disbursing → completed`
`collecting → partial` (window closes, not everyone paid — see §6.4 for what happens next)

**Payout:**
`scheduled → collecting → ready → disbursing → completed`
`disbursing → failed → disbursing` (retry path — disbursement can fail on MTN's side and must be retryable without double-paying, hence idempotency keys)

**Cycle:**
`forming → active → completed`
`active → at_risk` (triggered automatically if default rate in the cycle crosses a threshold, e.g. 20%)

---

## 6. Penalty Logic — All The Edge Cases

This is the part worth getting genuinely right, because it's where most susu apps quietly become unfair or exploitable.

### 6.1 Base penalty calculation
On the `overdue` transition:
```
if penalty_type == 'flat': penalty = penalty_value
if penalty_type == 'percentage': penalty = amount_due * penalty_value
if penalty_type == 'escalating': penalty = base_percentage * hours_overdue_in_full_days
penalty = min(penalty, amount_due * max_penalty_cap)   # never let a penalty exceed the cap
```

### 6.2 Timing edge cases
- **Payment made in the same second as the deadline:** never trust client-reported time. The authoritative timestamp is the one on the confirmed webhook/transaction record from MTN, not when the user tapped "pay." Compare that timestamp against `due_at` server-side only.
- **Grace period stacking:** `grace_period_hours` applies once, after `window_closes_at` — it is not renewed on each retry attempt.
- **Weekend/holiday due dates:** if `business_days_only` is true, when a calculated `due_at` lands on a weekend or a Ghanaian public holiday, roll forward to the next business day *at group creation time*, not dynamically — so members see a fixed, predictable schedule, not one that shifts based on when they check the app.
- **Owner changes rules mid-cycle:** rule changes on `Circle` (payment window, penalty type, etc.) must **never** retroactively affect an already-open `Round`. Snapshot the relevant rule values onto the `Round` record itself when it opens. New rules apply only to rounds created after the change. This prevents an owner from unfairly tightening rules on people mid-payment.

### 6.3 Payment edge cases
- **Webhook lost / delayed:** the member paid, MTN confirmed on their end, but your webhook never arrived. This is why `reconcile_pending_transactions` actively polls MTN's status endpoint rather than passively waiting — a payment should never be marked `defaulted` while a `Transaction` is sitting in `pending`.
- **Duplicate/double payment:** idempotency keys on every payment request prevent double-charging from client retries. If a genuine duplicate lands anyway (rare provider-side bug), the second payment triggers an automatic ledger entry crediting the member's wallet for the excess, flagged for admin review — never silently absorbed.
- **Partial payment (momo transaction partially fails or user sends wrong amount):** `Contribution.amount_paid` is tracked separately from `amount_due`. A contribution isn't `paid` until `amount_paid >= amount_due`. Underpayment keeps the contribution `pending`/`overdue` with the shortfall clearly displayed to the member, not silently accepted as "close enough."
- **Penalty paid but original contribution still outstanding:** penalties are tracked as a separate ledger line item, not folded into `amount_due`, so a member always sees exactly what they owe in principal vs. penalty.
- **Admin waives a penalty:** `penalty_waived_by` + reason are logged. The ledger entry for the penalty is reversed with a matching reversal entry — never deleted, so the audit trail always shows both the original charge and the waiver.

### 6.4 Round-level shortfall (not everyone paid by payout time)
Three configurable policies an owner can choose per circle:
- **Hold:** payout waits until 100% collected, even past `payout_due_at`. Simple, but delays the honest majority.
- **Buffer-covered:** the `buffer_pool_percentage` skimmed from every contribution covers the shortfall so the recipient still gets close to the full amount on schedule; the defaulting member now owes the buffer pool directly, tracked as a debt in the ledger against their account.
- **Partial payout:** recipient gets whatever was collected, and the remainder is paid out as a follow-up disbursement the moment the defaulting member eventually pays. `Payout.status = 'partial'` reflects this clearly rather than pretending the round is `completed`.

### 6.5 Member exit edge cases
- **Member leaves before receiving their payout:** they're removed from the rotation; remaining rounds are renumbered; their historical contributions stay in the ledger untouched (never delete financial history).
- **Member leaves after already receiving their payout (the risky case):** they still owe future contributions. `Membership.status = 'exited'` does not erase the debt — a standing `LedgerEntry` liability against that user persists, visible on their profile and factored hard into their reputation score. This is the exact scenario real ROSCAs fail on informally; your system should make the obligation explicit and trackable even if you can't force enforcement.
- **Guarantor forfeiture:** if a guaranteed member defaults, the guarantor's reputation takes a proportional hit, and if the circle's rules say so, the guarantor becomes liable for the shortfall — logged as its own ledger liability, distinct from the defaulter's.

### 6.6 Concurrency edge cases
- Two Celery workers processing the same overdue check simultaneously → wrap the status transition in `select_for_update()` inside a DB transaction so a contribution can't be double-penalized.
- A member pays at the exact moment the `check_overdue_contributions` job runs → the job re-checks `Contribution.status` inside the same locked transaction before applying a penalty, so a payment that lands mid-job-run doesn't get penalized after the fact.

---

## 7. API Structure (high level)

REST, versioned from day one (`/api/v1/...`). Rough resource map:

```
/auth/                     (register, login, refresh, KYC upload)
/circles/                  (CRUD, join, invite)
/circles/{id}/members/
/circles/{id}/cycles/
/cycles/{id}/rounds/
/rounds/{id}/contributions/
/contributions/{id}/pay/   (initiates MoMo collection)
/payouts/{id}/
/payments/webhooks/momo/   (MTN callback — signature-verified, no auth header)
/ledger/me/                (a member's own transaction history, read-only)
/reputation/me/
/disputes/
/notifications/
```

---

## 8. Build Order

1. `accounts` + `groups` + `cycles` models, no money logic — get the shape of a rotation right first.
2. `contributions` + `payouts` state machines, manually triggered ("mark as paid" admin action) — prove the state machine logic before touching real payments.
3. `ledger` — implement double-entry bookkeeping against the manual flow above. Don't skip this step or bolt it on later; retrofitting a ledger is painful.
4. `payments` — MTN MoMo sandbox, webhooks, idempotency, reconciliation job.
5. Wire §6's penalty logic fully, including the Celery beat schedule.
6. `reputation` + `disputes` + `notifications`.
7. Web frontend, once the API is stable and documented via drf-spectacular.

---

## 9. Honest Compliance Note

This system is architected as if it custodies member funds (a real e-money/wallet model), because that's the only way to learn real fintech ledger design. In production, Ghana requires a Bank of Ghana e-money issuer license or a partnership with one to legally hold customer funds this way — this project is a portfolio/learning build, not a licensed financial product, and that boundary is worth stating explicitly anywhere you show this off.
