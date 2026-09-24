# Build prompt: the outreach engine

This file turns [`outreach-engine.md`](outreach-engine.md) into a spec an
agentic coding tool can execute. It contains two prompts:

- **Prompt A** builds the engine: a Cloudflare Worker, D1, the Apps Script
  senders, CI, in nine phases.
- **Prompt B** is the research brief that fills it with verified contacts.

The spec builds the engine the way we would build it again. It keeps every
rail from the description, and it skips the two structural debts our own engine
still carries: families marked sent on read, and campaign rules held as
constants in code. Read section 6 of the description ("Rails, and the incident
behind each") before you change any of the invariants.

The phase numbers are not the roadmap's stage numbers. Two roadmap stages,
"9 · Registry drives everything" and "10 · One research pipeline", are built in
from the start here, in Phases 2, 3 and 5. The later phases, 9 to 12, cover
roadmap stages 11 to 13.

## How to use it

**Claude Code**

1. Create an empty repository.
2. Save Prompt A as `SPEC.md` at its root, with the placeholders in section 0
   filled in.
3. Start a session there and say:
   > Read SPEC.md in full. Implement Phase 0 only. Stop when its acceptance
   > checks pass and report back.
4. Review the report, then ask for the next phase. One phase per session keeps
   the context small and the review honest.

**Cursor**

1. Put Prompt A in `.cursor/rules/outreach-spec.mdc`, or keep it as `SPEC.md`
   and reference it with `@SPEC.md`.
2. Ask for one phase at a time, in Agent mode, with the same stop rule.

**Both tools**

- Fill in section 0 before starting.
- Leave everything marked MUST as it is unless you have read the incident
  behind it.
- The spec assumes one developer and one owner, who may be the same person.
  Whatever it asks the owner to do by hand (secrets, DNS, installing the Apps
  Script) is listed in each phase's report.

---

## Prompt A: build the engine

````markdown
# SPEC — Outreach engine on Cloudflare Workers + D1 + Gmail Apps Script

You are a senior engineer building a production outreach engine. Read this whole
file before writing any code.

Work in phases. For each phase:
1. Implement it.
2. Run its tests.
3. Report three things: what you built, what you verified and how, and what
   the owner must do by hand.
4. STOP. Never start the next phase without an explicit go-ahead.

If a requirement here conflicts with something you believe is better, say so in
the report. Do not silently deviate.

## 0. Context (the owner fills this in)

- Brand the letters are signed with: {{BRAND}}
- Sending domain (Google Workspace): {{DOMAIN}}
- Account types (segments): {{SEGMENTS}}. Example: municipality,
  tourist_office, university, media, hotel.
- Countries, and the language of the letters in each: {{COUNTRIES_LANGS}}.
  Example: ES → es, PT → pt.
- Mailboxes, in rollout order: {{MAILBOXES}}. Example: `pilot` (a consumer
  Gmail) first, then `a`, `b`, `c` on the Workspace domain.
- Owner alert address: {{OWNER_EMAIL}}
- Send window in UTC, chosen for the recipients' office hours: {{WINDOW_UTC}}.
  Default 09:00–10:14.
- Legal basis for outreach, per country: {{LEGAL_BASIS}}. The owner confirms
  this with counsel. The engine enforces what it is told and never guesses.

## 1. What you are building

There are four parts, and each does one job:

- **Research** turns briefs into verified CSV rows, then into idempotent SQL
  imports. It lives in `/research`.
- **Brain** is one Cloudflare Worker with D1 and Cron Triggers. It decides who
  gets which letter today, renders it, refuses anything unsafe, and records
  everything.
- **Transport** is one Google Apps Script project per mailbox. It fetches
  rendered letters, sends them as-is with GmailApp, and acks each one. It makes
  no decisions.
- **Feedback** is a tracked redirect for clicks, an inbox harvester for replies,
  opt-outs and bounces, a classifier, and suppression.

The governing principle: **the worker is the brain, the mailbox is a dumb
pipe.** Anything that matters (the window, caps, order, copy, suppression) is
decided server-side, where it is versioned and tested.

## 2. Invariants (MUST). Each one needs a test.

1. **One send path.** Only the Apps Script senders send outreach. Build no
   SMTP, ESP or worker-side transport, not even "for later".
2. **The worker renders; the sender is dumb.** The sender posts `subject`,
   `text` and `html` exactly as received. Rendering is deterministic: the same
   row renders the same bytes in preflight, in the dry-run digest and at send
   time. No LLM runs at send time.
3. **Claim, then ack. Delivery is at-most-once.**
   - An export moves rows from `due` to `staging` with a claim id.
   - Only `POST /queue/ack` moves a row to `sent` or `failed`.
   - A row still in `staging` after 2 hours becomes `unacked`. That state is
     terminal and never resent automatically.
   - Prefer losing a letter to sending it twice.
4. **Reading never mutates.** `dry=1` renders without claiming, and a GET
   without a valid token returns 403.
5. **`touches` is append-only,** enforced by a DB trigger. A correction is a new
   row with `supersedes = <old id>`.
6. **Suppression is keyed by `email_hash`** (sha256 of the trimmed, lower-cased
   address). It is checked when a row is queued AND again at export.
7. **The send window is enforced by the queue, in UTC.** Outside it, the export
   returns an empty batch with `window.open=false`, and the sender alerts the
   owner. Never rely on the Apps Script trigger's time zone.
8. **A single campaign registry is the source of truth.** Release rules,
   renderer choice, mailbox routing, weekend rules, identity requirements,
   caps and kill switches are all read from it. A test fails when a campaign is
   missing from any consumer, or when any touch lacks a renderer.
9. **The export guard refuses; it never warns.** It refuses on:
   - order: a later touch of the family has already been sent;
   - duplicates: the same family and touch number was already sent;
   - manual contacts;
   - cooldown: any touch to this contact in the last 48 h;
   - canary: a campaign with fewer than 3 letters ever sent gets at most 3
     today.

   Order, duplicate and manual refusals park the row as `blocked` with a
   reason. Cooldown and canary refusals leave it `due`.
10. **Every export writes a `send_log` row:** planned, sent, skipped, and the
    count of each refusal reason. Dry runs write nothing.
11. **Every intake filters on account type and country, never on region
    alone.** A contact that never received a campaign's T1 cannot receive its
    T2.
12. **Follow-ups fire only while the previous touch is still `sent` once the
    gap has passed.** A click, reply, bounce or opt-out changes that status and
    stops the sequence. There is no separate "engaged" query.
13. **Every opt-out keyword a template promises must be recognised by the
    classifier.** A test renders every template and checks this. The opt-out
    line is in the letter's language.
14. **The schema check reads `sqlite_master`, not the migration ledger.** A
    missing required table is an alert on its own.
15. **Deploys run from CI only:**
    - tests → apply migrations → deploy → assert that `/version.json` matches
      the commit → assert that `/preflight.json` is `ok`;
    - CI refuses to deploy within 15 minutes of the window, or during it,
      unless forced.
16. **No PII in git.** Imports write SQL to a git-ignored folder. Test fixtures
    use `example.test` addresses.
17. **Time is an argument.** Every function that depends on the clock takes
    `now: Date`. Tests never sleep and never mock the global clock.
18. **Every query has an index.** Watch rows read, not rows written. Never
    write per-event analytics into D1.
19. **Legal basis is data.** Contacts carry `country`, `legal_basis` and
    `marketing_ok`, and the intake reads them. A country without a configured
    basis gets no letters.

## 3. Stack and layout

- **Runtime:** Cloudflare Workers (TypeScript, strict), D1, Cron Triggers, and
  a custom domain for the worker.
- **Tests:** vitest, or node:test with `@cloudflare/vitest-pool-workers` or
  Miniflare for D1. Tests never touch the network.
- **Senders:** Google Apps Script (V8), one project per mailbox, all from the
  same file.
- **CI:** GitHub Actions.

```
/worker
  wrangler.jsonc
  src/index.ts          routing only: fetch, scheduled, and nothing else
  src/registry.ts       CAMPAIGNS and MAILBOXES, the single source of truth
  src/queue.ts          intake, follow-ups, release, export (claim), ack, reap
  src/guard.ts          window, caps and ramps, order, duplicate, manual, cooldown, canary
  src/render/<family>.ts  one pure renderer per campaign family and language
  src/links.ts          tracked-link builder
  src/inbound.ts        classifier: bounce | unsubscribe | autoreply | reply
  src/preflight.ts      preflight, digest, isBroken()
  src/outbox.ts         transactional mail and owner alerts
  src/window.ts         the send window, pure, takes now
  src/admin.ts          cockpit (behind Cloudflare Access)
  test/
/migrations             NNNN-name.sql, applied in order by CI
/apps-script            sender.gs, harvester.gs, README.md (install steps)
/research               briefs/, contract.md, coverage.mjs, verify.mjs, dns.mjs, import.mjs
/.github/workflows      deploy.yml
```

## 4. Data model

Apply as migrations. Every statement is `IF NOT EXISTS`.

```sql
CREATE TABLE IF NOT EXISTS accounts (
  id TEXT PRIMARY KEY,                 -- deterministic: website domain, else hash(type|country|city|name)
  type TEXT NOT NULL, name TEXT, website TEXT,
  country TEXT NOT NULL, region TEXT, city TEXT,
  status TEXT NOT NULL DEFAULT 'active', -- active|bounced|unsubscribed|converted|dead
  source TEXT, enrichment TEXT, created_at TEXT NOT NULL, updated_at TEXT
);
CREATE TABLE IF NOT EXISTS contacts (
  id TEXT PRIMARY KEY,
  account_id TEXT REFERENCES accounts(id),
  email TEXT,                           -- operational only, never exported to git
  email_hash TEXT NOT NULL UNIQUE,
  name TEXT, role TEXT, lang TEXT NOT NULL, country TEXT NOT NULL,
  legal_basis TEXT, marketing_ok INTEGER NOT NULL DEFAULT 0,
  lifecycle TEXT NOT NULL DEFAULT 'lead', -- lead|engaged|customer|invalid
  manual INTEGER NOT NULL DEFAULT 0,      -- 1 = a person owns the thread; exports refuse
  source TEXT, created_at TEXT NOT NULL, updated_at TEXT
);
CREATE TABLE IF NOT EXISTS suppression (
  email_hash TEXT PRIMARY KEY, reason TEXT NOT NULL, -- unsubscribe|bounce|complaint|invalid_dns|manual
  ts TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS consents (           -- append-only ledger
  id TEXT PRIMARY KEY, contact_id TEXT NOT NULL, purpose TEXT NOT NULL,
  granted INTEGER NOT NULL, legal_basis TEXT, source TEXT, ts TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS send_queue (
  id TEXT PRIMARY KEY,                  -- q_<contact>_<campaign>_<touch>: deterministic
  contact_id TEXT NOT NULL, account_id TEXT,
  campaign TEXT NOT NULL, touch_number INTEGER NOT NULL,
  box TEXT NOT NULL,                    -- mailbox that will carry it
  status TEXT NOT NULL,                 -- scheduled|due|staging|sent|failed|unacked|blocked|paused
  claim_id TEXT, claimed_at TEXT,
  reason TEXT, variant TEXT,
  not_before TEXT,                      -- earliest UTC date it may be exported
  created_at TEXT NOT NULL, updated_at TEXT
);
CREATE INDEX IF NOT EXISTS idx_queue_box_status ON send_queue (box, status, not_before);
CREATE INDEX IF NOT EXISTS idx_queue_contact ON send_queue (contact_id, campaign);
CREATE TABLE IF NOT EXISTS touches (
  id TEXT PRIMARY KEY, contact_id TEXT NOT NULL, account_id TEXT,
  campaign TEXT NOT NULL, touch_number INTEGER NOT NULL, box TEXT NOT NULL,
  status TEXT NOT NULL,                 -- sent|clicked|replied|bounced|unsubscribed|failed|unacked
  sent_at TEXT NOT NULL, queue_id TEXT, variant TEXT, supersedes TEXT, meta TEXT
);
CREATE INDEX IF NOT EXISTS idx_touches_contact ON touches (contact_id, campaign, touch_number);
CREATE INDEX IF NOT EXISTS idx_touches_campaign ON touches (campaign, status);
CREATE TRIGGER IF NOT EXISTS touches_append_only BEFORE DELETE ON touches
BEGIN SELECT RAISE(ABORT, 'touches is append-only: insert a superseding row'); END;
CREATE TABLE IF NOT EXISTS email_events (
  id TEXT PRIMARY KEY, ts TEXT NOT NULL, message_id TEXT UNIQUE,
  email_hash TEXT, contact_id TEXT, account_id TEXT,
  kind TEXT NOT NULL,                   -- bounce|unsubscribe|autoreply|reply
  rule TEXT NOT NULL,                   -- which rule fired
  attribution TEXT NOT NULL,            -- contact_exact|account_domain|unattributed
  campaign TEXT
);
CREATE TABLE IF NOT EXISTS send_log (
  id TEXT PRIMARY KEY, day TEXT NOT NULL, box TEXT NOT NULL,
  planned INTEGER NOT NULL, sent INTEGER NOT NULL, skipped INTEGER NOT NULL,
  campaigns TEXT, reasons TEXT, created_at TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_send_log_day ON send_log (day, box);
CREATE TABLE IF NOT EXISTS clicks (
  id TEXT PRIMARY KEY, ts TEXT NOT NULL, campaign TEXT, account_id TEXT,
  touch_id TEXT, visitor_id TEXT, path TEXT
);
CREATE TABLE IF NOT EXISTS outbox (           -- transactional mail and owner alerts
  id TEXT PRIMARY KEY, kind TEXT NOT NULL, to_addr TEXT NOT NULL,
  subject TEXT NOT NULL, body_text TEXT NOT NULL, body_html TEXT,
  status TEXT NOT NULL DEFAULT 'pending', -- pending|staging|sent|failed
  dedup_key TEXT UNIQUE,                  -- e.g. alert:preflight:2026-10-01:night
  created_at TEXT NOT NULL, sent_at TEXT
);
CREATE TABLE IF NOT EXISTS campaign_config (  -- the runtime-editable half of the registry
  campaign TEXT PRIMARY KEY, enabled INTEGER NOT NULL DEFAULT 1,
  starts_on TEXT, ends_on TEXT, daily_cap INTEGER, pause_types TEXT, updated_at TEXT
);
CREATE TABLE IF NOT EXISTS meta (key TEXT PRIMARY KEY, value TEXT, updated_at TEXT);
CREATE TABLE IF NOT EXISTS migrations (name TEXT PRIMARY KEY, applied_at TEXT NOT NULL);
```

## 5. The campaign registry

The registry has two halves:

- **Code** (`registry.ts`) holds structure: families, touches, renderers,
  segments, mailboxes.
- **Data** (the `campaign_config` table, edited from the cockpit) holds what
  changes without a deploy: enabled, dates, caps, paused types.

```ts
export type Touch = { n: 1 | 2 | 3; gapDays: number; render: RenderFn };
export type Campaign = {
  id: string;                  // family, e.g. 'univ_es'
  lang: string;                // letter language
  countries: string[];
  accountTypes: string[];      // intake guard: never region-only
  touches: Touch[];            // T1..T3, gaps counted from the previous touch
  boxes: string[];             // mailboxes allowed to carry it
  weekends: boolean;
  needsSenderIdentity: boolean;// legal identity block required in the signature
  optOutKeyword: string;       // what the letter tells people to reply; the classifier must know it
  rank: number;                // release priority when budgets are short
};
export type Mailbox = {
  box: string;
  kind: 'gmail' | 'workspace';
  rampStart: string;           // YYYY-MM-DD
  rampFrom: number;            // letters on day 1, e.g. 5
  rampStep: number;            // +N per day, e.g. 1
  cap: number;                 // ceiling, e.g. 40
  triggerMinuteUtc: string;    // documentation, e.g. '09:29'
};
export const CAMPAIGNS: Campaign[] = [/* ... */];
export const MAILBOXES: Mailbox[] = [/* ... */];
```

A mailbox's budget for day `d` is
`min(cap, rampFrom + rampStep * daysSince(rampStart))`, and never more than
the Apps Script daily recipient quota for its account type. Follow-ups are
budgeted before first touches.

## 6. HTTP contract

All endpoints except `/health`, `/version.json` and `/go` need
`Authorization: Bearer <token>`.

| Method and path | Purpose |
|---|---|
| `GET /health` | `ok` |
| `GET /version.json` | `{sha, source, deployedAt}` from CI vars, public |
| `GET /queue.json?box=<box>[&dry=1]` | the batch for one mailbox; claims rows unless `dry=1` |
| `POST /queue/ack` | `{claimId, sent: [id], failed: [{id, error}]}`; idempotent |
| `POST /inbound` | one harvested message; classify, apply, log; idempotent on `messageId` |
| `GET /outbox.json`, `POST /outbox/ack` | transactional mail and alerts, claim and ack like the queue |
| `GET /go?c=<campaign>&k=<account>&t=<touch>&u=<path>` | served on the site's own domain (a Worker route or Pages Function writing to the same D1), so the visitor id is first-party. Logs the click, flips the touch to `clicked`, sets the visitor cookie, 302 to the site path. Only a relative `u` is allowed (no open redirect) |
| `GET /preflight.json` | `{ok, day, boxes: [{box, due, renderable, refused: {reason: n}}], schemaMissing: []}`; never mutates |
| `GET /admin` | cockpit behind Cloudflare Access: today's plan per mailbox, send log, stuck rows, reply feed, the campaign_config editor, pause and unpause |

`/queue.json` response:

```json
{
  "box": "a",
  "claimId": "clm_2026-10-01_a_0929",
  "window": { "open": true, "label": "09:00-10:14 UTC" },
  "messages": [
    { "id": "q_ct1_univ_es_1", "to": "person@example.test", "fromName": "{{BRAND}}",
      "replyTo": null, "subject": "…", "text": "…", "html": "…" }
  ],
  "skipped": { "cooldown": 2, "canary": 5 }
}
```

## 7. Apps Script contract

There is one file, `sender.gs`, installed in every mailbox. Script properties:
`TOKEN`, `BOX`, `BASE_URL`. The project time zone is set to UTC, but the
window is still enforced by the worker.

- `send()` runs from a daily trigger at a fixed minute inside the window. It:
  1. fetches `/queue.json?box=BOX`;
  2. if `window.open` is false, alerts the owner (throttled to once an hour)
     and exits;
  3. for each message, calls `GmailApp.sendEmail(to, subject, text, {htmlBody,
     name, replyTo})` and then acks that single id at once;
  4. sleeps 3–5 s between letters and keeps the whole batch under 5 minutes
     (the Apps Script execution limit is 6);
  5. if an ack fails, retries it 3 times, then alerts the owner. The row will
     become `unacked`, which is correct.
- `digest()` runs daily, one hour before `send`. It fetches with `dry=1` and
  mails the owner the batch: recipients, subjects, and the first lines.
- `sendOutbox()` runs every 5 minutes. It drains `/outbox.json` (form
  auto-replies and owner alerts) and acks each message, so an alert never waits
  for tomorrow's window.
- `harvest()` runs every 10 minutes. It searches for unlabelled threads newer
  than 2 days and posts each new message to `/inbound` with `{messageId,
  threadId, from, to, subject, date, headers, body}`. It labels the thread
  `crm-processed` only after an HTTP 200.

## 8. Crons (UTC)

| Cron | Job |
|---|---|
| `17 3 * * *` | nightly chain: reap unacked → intake and follow-ups → release scheduled → enrichment → attribution |
| `45 3 * * *` | night preflight (5 hours to fix) |
| `30 8 * * *` | morning preflight (second net) |
| `15 10 * * *` | digest: plan vs fact per mailbox, verdict in the subject, sent every day |
| `*/30 * * * *` | D1 budget guard: rows read today against an anomaly threshold, alert through `outbox` |

Each cron job is a separate `ctx.waitUntil` branch. The digest must never
depend on the nightly chain succeeding.

`isBroken(box)` is defined in one place and used by preflight, the digest and
CI: the box has due rows, renders none, and its refusals are not only canary or
cooldown. Alerts go through `outbox` with a `dedup_key` per day and slot.

## 9. Phases

Each phase lists what to build, then when it is done. "Done" is checked by
tests or by a command whose output goes in the report.

### Phase 0: Skeleton and deploy discipline

**Build**

- Worker with `/health` and `/version.json`.
- `wrangler.jsonc` with D1, a custom domain and crons.
- A migration runner and a schema check against `sqlite_master`.
- `deploy.yml`: tests → migrations → deploy with `--var GIT_SHA` →
  assert `/version.json`. The workflow refuses to run inside the freeze window
  unless `force` is set.
- A local deploy guard that fails without `ALLOW_LOCAL_DEPLOY=1`.

**Done when**

- CI deploys, and `/version.json` returns the commit.
- `wrangler deploy` from a laptop fails.
- A test proves the schema check flags a table that is missing although the
  ledger lists its migration.

### Phase 1: Record store

**Build**

- The tables from section 4 and the append-only trigger.
- `emailHash()`, `upsertAccount()`, `upsertContact()`, `suppress()`,
  `isSuppressed()`.

**Done when**

- Applying the migrations twice is a no-op.
- Deleting a touch raises.
- Upserting the same address twice yields one contact.
- A suppressed hash is never re-marked `marketing_ok`.

### Phase 2: Research import

**Build**, in `/research`:

- `contract.md` defining the CSV columns:
  `type,name,email,website,country,region,city,contact_role,source_url,notes`.
- `coverage.mjs`: prints the organisations already held for a region, as a
  block to paste into a brief.
- `verify.mjs`: fetches `source_url` and requires a literal match of the
  address. It writes `verified|not_on_page|fetch_failed`, samples at least 10%
  when run in `--sample` mode, and widens the sample on the first miss.
- `dns.mjs`: syntax, then MX, then A/AAAA. Dead domains produce a suppression
  row with reason `invalid_dns`. It never probes over SMTP.
- `import.mjs`: builds deterministic account ids, dedups on `email_hash` within
  the batch and against the DB snapshot, and writes idempotent SQL to
  `/research/out/` (git-ignored). A shared address listed for two roles goes to
  one contact, with the roles joined.

**Done when**

- A fixture CSV imported twice into a local SQLite gives identical counts.
- A row whose address is not on its page is excluded, and appears in the
  report.
- A region that is not in the closed list fails the import.

### Phase 3: One mailbox, one campaign, a human in the loop

**Build**

- `registry.ts` with one campaign (T1 only) and one mailbox.
- `render/<family>.ts`, with tracked links through `/go` and an opt-out line
  in the letter's language.
- `window.ts`.
- `queue.ts`: intake, export with claim, ack, and reap.
- `guard.ts`: order, duplicate, manual, cooldown, canary.
- The `send_log` write.
- `sender.gs` with `send()` and `digest()`.
- The Apps Script install README.

**Done when**

- The dry run and the real export render byte-identical letters.
- Outside the window, the export returns an empty batch with `open=false`.
- A batch claimed but never acked becomes `unacked` after 2 h and is never
  exported again.
- A new campaign exports at most 3 letters on its first day.
- `send_log` records each refusal with its reason.
- The owner can install the script from the README alone.

### Phase 4: Feedback loop

**Build**

- `/go`.
- `inbound.ts`, with rules in this order:
  1. bounce: DSN, mailer-daemon, delivery-failure subjects;
  2. autoreply: `Auto-Submitted`, `X-Autoreply`, out-of-office subjects in
     every configured language;
  3. unsubscribe: per-language keyword packs, subject and body, word-bounded,
     with idiom exclusions (for example the Spanish *temporada baja*,
     "low season", is not an opt-out);
  4. reply.
- `harvest()` in `sender.gs`.
- Suppression and status updates.
- An `email_events` row for every decision.

**Done when**

- A reply flips the latest open touch to `replied` and blocks its follow-ups.
- An opt-out suppresses by hash, and the contact receives nothing from any
  campaign.
- An autoreply changes nothing.
- A test proves that every template's `optOutKeyword` is classified as
  `unsubscribe` in its language.
- Re-posting the same `messageId` is a no-op.

### Phase 5: Cadence and guards

**Build**

- T2 and T3 per campaign, with `gapDays`.
- Release of `scheduled` rows by rank, weekday rules, a 2-day fatigue rule, and
  `campaign_config` dates.
- Budgets per mailbox: follow-ups first, then first touches.
- A type-and-country guard on every intake.

**Done when**

- A contact that clicked T1 never gets T2.
- T2 can never be exported before T1 is `sent`.
- A contact touched by campaign X's T1 can never receive campaign Y's T2.
- Weekend rules and paused types are respected.
- Budgets never exceed the ramp.

### Phase 6: More mailboxes

**Build**

- Several entries in `MAILBOXES`.
- Mailbox routing from `Campaign.boxes`.
- Per-mailbox ramps.
- A DNS checklist for the Workspace domain (SPF, DKIM, DMARC with `rua`
  reporting), delivered as `docs/dns.md` for the owner.

**Done when**

- Two mailboxes never receive the same row.
- Each mailbox's batch respects its own ramp.
- A test fails if `MAILBOXES` and the installed `BOX` names listed in
  `apps-script/README.md` diverge.

### Phase 7: Observability

**Build**

- `preflight.ts`: the night and morning slots, and `isBroken()`.
- The 10:15 digest, whose subject carries the verdict (for example
  `NOTHING SENT` or `42/45 sent`).
- `outbox` and the `sendOutbox()` Apps Script trigger.
- `/preflight.json`, checked by CI after every deploy.
- The `/admin` cockpit.

**Done when**

- A campaign added to the registry without a renderer makes preflight
  `ok:false` and fails CI.
- A day with due rows and zero renderable letters produces an alert at 03:45,
  once per slot.
- The digest goes out even when the nightly chain threw.

### Phase 8: Attribution and reporting

**Build**

- Click → visitor → contact stitch.
- `outcomes`, credited to the last touch before a conversion.
- Per-campaign and per-segment reports in the cockpit: sent, clicked, replied,
  bounced, and opted out, by week and by mailbox.

**Done when**

- Every number in the report can be reproduced by one indexed query.
- A campaign's reply rate is visible without writing SQL.

### Later phases (only when the owner asks)

- **9. Deliverability telemetry:** bounce and complaint rate per mailbox,
  seed-list placement tests, DMARC aggregate parsing, and an automatic pause
  of a mailbox over 2% bounces.
- **10. Reply intelligence:** an LLM classifier layered over the rules, with
  the classes interested, not now, wrong person plus referral, opt-out and out
  of office.
  - A referral creates a new contact.
  - A positive reply is handed to a person with an SLA.
  - Rules always win on opt-out.
- **11. Grounded personalisation:** a per-account fact sheet with a source for
  every fact, rendered into letters. A letter never states a number it cannot
  trace.
- **12. Experiments:** variants with a holdout, never run across a holiday
  month for a segment that takes holidays.

## 10. Never

- Never add a second send path.
- Never resend automatically after an uncertain outcome.
- Never mutate on a GET without a claim.
- Never delete a touch.
- Never commit an address.
- Never guess an email pattern.
- Never probe SMTP.
- Never use open-tracking pixels. Clicks and replies are the signals.
- Never let a campaign exist in one consumer and not another.
- Never deploy inside the window.
- Never keep a date, a pause or a cap in code when it belongs in
  `campaign_config`.
````

---

## Prompt B: research brief for list building

Use this prompt for breadth, meaning a list of 60–300 organisations. Run it as
a custom workflow: one extract agent and one verify agent per source stream,
merged and deduplicated in code. A generic deep-research tool summarises well
but undershoots enumeration; on one of our briefs it returned zero rows where
the workflow returned 303. Run wide fan-outs in their own session, on a model
you can afford to fan out.

````markdown
# Research brief — {{SEGMENT}} in {{GEOGRAPHY}}

You are building a contact list for one-to-one outreach. Enumerate; do not
summarise. Completeness and verifiability matter more than prose.

## Already held — do NOT return these
{{PASTE THE OUTPUT OF research/coverage.mjs HERE}}

## Sweep these source streams one by one, and report each
1. {{Official registry, e.g. the national directory of public bodies}}
2. {{Regional government lists}}
3. {{Association or federation member directories}}
4. {{The organisations' own websites, contact and staff pages}}
(Add streams; never merge two streams into one search.)

## Output: CSV only, these columns, in this order
type,name,email,website,country,region,city,contact_role,source_url,notes

## Rules
- `type` is one of: {{SEGMENTS}}. `region` is one of: {{CLOSED LIST OF REGIONS}}.
- Only addresses published verbatim at `source_url`. Never construct an address
  from a pattern (info@, firstname.lastname@). Never decode an obfuscated
  address. Put "OBFUSCATED" in `notes` and leave `email` empty.
- Prefer the organisation's own site or an official registry over aggregators.
  Aggregators invent plausible addresses.
- Prefer a named role (tourism, culture, communications, research) over a
  generic inbox. Record both if both are published.
- One row per (organisation, address). A shared inbox serving two roles is one
  row, with the roles joined in `contact_role`.
- Doubtful rows are kept and tagged `LOW-CONFIDENCE` in `notes`, with the reason.
- Stop at {{N}} rows or when the streams are exhausted.

## After the CSV
- Per stream: sources checked, rows found, rows dropped as already held.
- Gaps: regions or types where nothing was found, and why.
````

Every row then goes through `verify.mjs` (literal match on the page), then
`dns.mjs`, then `import.mjs`, as described in Phase 2. Treat a model saying
"verified" as a claim, not as evidence: about 5% of such rows failed the
literal match.
