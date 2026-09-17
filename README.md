# Referral Intake & Case Automation

An n8n-based automation that takes website form submissions and safely turns them into case records — designed around one core constraint: **a downstream system failing should never mean losing or duplicating a submission.**

This was built as a self-directed portfolio project to demonstrate the specific engineering concerns that come up in real business automation work: idempotency, partial-failure recovery, and data integrity under unreliable external dependencies. No fake clients, users, or business results are represented here — this is a self-built, tested system using synthetic data.

**[Demo video](https://github.com/user-attachments/assets/75b9dc0c-1c5b-498c-aba7-cb29ec407568)** 

---

## The problem this solves

A common pattern in small-business automation: a website form collects a referral or intake request, and that data needs to reliably become a case record — logged, routed to the right person, and confirmed by notification — without a human manually re-entering it.

The naive version of this ("form → database → email") breaks in predictable ways:
- The same submission arrives twice (a double-click, a network retry, a resubmit) and creates two cases for one person.
- A downstream step — creating the case record, sending the notification — fails partway through, and the submission is either silently lost or gets reprocessed from scratch, creating duplicates.

This project handles both failure modes deliberately, rather than assuming the happy path.

---

## Architecture

```
Website Form
     │
     ▼
Webhook (n8n)
     │
     ▼
Insert into `submissions`  ──[duplicate]──► Respond: "already submitted"
     │ (dedup enforced at DB level)
     ▼
Update status → processing
     │
     ▼
Insert into `cases`  ──[fails]──► Respond: "received, processing delayed"
     │ (status → case_created)
     ▼
Send notification email  ──[fails]──► Respond: "received, notification delayed"
     │
     ▼
Update status → completed
     │
     ▼
Respond: "success"


Separate Retry Workflow (scheduled)
     │
     ├─► Find rows stuck at `processing`  → create case → notify → complete
     └─► Find rows stuck at `case_created` → notify only → complete
```

Two workflows, not one:
- **Main workflow** — triggered by the webhook, handles a submission in real time.
- **Retry workflow** — triggered on a schedule, independently scans for anything that got stuck and finishes the job. This has to be separate, since a webhook-triggered workflow only runs when a new submission arrives; recovering an old, stuck one needs a different trigger entirely.

---

## Key design decisions

### Idempotency: a time-windowed hash, not a raw unique constraint

Two submissions are considered "the same case" if they share the same normalized `email + request_type + full_name` **within a rolling 7-day window** — not forever, and not just on exact timestamp match.

- A plain `UNIQUE` constraint on the identity fields alone would incorrectly block a legitimate second request from the same person months later.
- Checking for duplicates in application code first, then inserting, creates a race condition: two near-simultaneous submissions can both pass the check before either is inserted, and both get through.

The fix used here: a `dedup_bucket` column, computed as the identity hash concatenated with a 7-day time bucket, with a **partial unique index** on that column. This pushes the guarantee into the database itself — enforced atomically, not through a check-then-insert pattern in workflow logic.

```sql
CREATE UNIQUE INDEX uniq_dedup_bucket
    ON submissions (dedup_bucket)
    WHERE status != 'duplicate';
```

### A multi-stage status lifecycle, not a binary success/fail

```
received → processing → case_created → completed
```

The reason for three intermediate states instead of one generic "failed" state: a submission can get stuck at genuinely different points, and each requires a different recovery action.

- Stuck at `processing` → the case record was never created. Recovery must create it.
- Stuck at `case_created` → the case exists, only the notification failed. Recovery must **only** resend the notification — re-running case creation here would create a duplicate case linked to the same submission.

The retry workflow queries these two states separately and takes a different, narrower recovery action for each, rather than blindly retrying "everything" from scratch.

### Parameterized queries throughout

All SQL run from n8n uses placeholder parameters (`$1`, `$2`, ...) rather than string-interpolated values, so submitted form data is always treated as data by Postgres — never as executable SQL, regardless of what a user types into the form.

### Failure isolation at every external step

Every node that talks to something outside the system's own database (the case-insert, the notification API) is configured with `On Error: Continue Using Error Output`. A failure at any one step is caught and routed to its own response path — it never crashes the whole execution, and it never silently loses the submission already safely captured in `submissions`.

---

## Schema

```sql
CREATE TABLE submissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    raw_payload     JSONB NOT NULL,
    source_ip       TEXT,
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    request_type    TEXT NOT NULL,
    dedup_hash      TEXT NOT NULL,
    dedup_bucket    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'received',
    case_record_id  UUID,
    error_detail    TEXT
);

CREATE TABLE cases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submission_id   UUID NOT NULL REFERENCES submissions(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    request_type    TEXT NOT NULL,
    assigned_to     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`raw_payload` retains the original submission untouched, regardless of what happens downstream — useful both for debugging and as an audit trail.

---

## What's deliberately not built yet

**Retry escalation after repeated failures.** Currently, a row stuck in `processing` or `case_created` will be retried indefinitely on every scheduled run. In a real deployment, this needs a cap: a `retry_count` column, incremented on each failed retry, with a threshold (e.g. 5 attempts) after which the row moves to a terminal `failed_needs_review` status and triggers an alert to a human operator rather than retrying forever. This was reasoned through during development but left as a documented next step rather than built, to keep the core mechanism's scope tight and demonstrable.

**Real Sheets/AppSheet/Drive integration.** This version uses Postgres tables to represent the "case record store," since the core problem — safe intake under partial failure — is identical regardless of what the downstream system actually is. Swapping the Postgres case-insert for a Google Sheets/AppSheet API call is a mechanical change to one node, not a redesign.

---

## Stack

- **n8n** (self-hosted, Docker) — workflow orchestration
- **PostgreSQL** — persistent state, dedup enforcement, status tracking
- **Resend API** — transactional email for notifications
- Plain HTML/JS test form — simulates the website intake point

## Running it locally

1. `cp .env.example .env` and fill in real values
2. `docker compose up -d`
3. Run the schema SQL against the `po_intake` database
4. Import both workflow JSON files into n8n, reconnect credentials
5. Serve `webhook-tester.html` (e.g. `python -m http.server 8000`) and submit against your webhook's production URL
