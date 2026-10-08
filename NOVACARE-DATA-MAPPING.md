# NovaCare — Data Mapping (current state)

Updated 2026-10-08. Field-by-field reference for where each piece of data lives across the
five systems, and why it is built that way.

Systems: Webflow `6a844bc4ca0928415fbbd845` · Make scenario `5981328` ·
Supabase `bjjgckoseqvaxlpjglpj` · HubSpot portal `247088766` · Retool `NovaCare Admin Portal`

### Verification status for this revision

| System | Verified | How |
|---|---|---|
| Webflow | ✅ 2026-10-08 | Live DOM read of the published `/book` form |
| Make | ✅ 2026-10-08 | Scenario blueprint fetched; active, valid, 7 modules |
| Supabase | ⚠️ not re-verified | MCP connection rejects both `postgres` and `supabase_read_only_user` with `28P01 password authentication failed` |
| HubSpot | ⚠️ not re-verified | Not queried this revision |
| Retool | ⚠️ not re-verified | MCP server disconnected / awaiting re-authorisation |

Sections marked ⚠️ reflect the last confirmed state (2026-08-19) and should be re-checked
before being relied on.

---

## 1. Inbound pipeline — Webflow → Make → Supabase → Brevo → HubSpot

Order matters: the patient email is sent **before** the HubSpot step, so patient
communication does not depend on the CRM succeeding.

### Patient Name
| Layer | Value |
|---|---|
| Webflow inputs | `patient_first_name`, `patient_last_name` (both required) |
| Make keys | `payload.data.patient_first_name`, `payload.data.patient_last_name` |
| Supabase | `patients.full_name` (TEXT) ← `trim(first + " " + last)` |
| HubSpot | `firstname`, `lastname` |

HubSpot has **no writable full-name property**. The "Name" in the contact list is computed
from `firstname` + `lastname`; there is no `name` property to map.

### Patient Email
| Layer | Value |
|---|---|
| Webflow input | `patient_email` — `type="email"`, required |
| Make key | `payload.data.patient_email` |
| Supabase | `patients.email` (TEXT, **UNIQUE**) ← `lower(trim(...))` |
| HubSpot | `email` |

Validated twice, deliberately. The browser rejects malformed addresses on submit (UX), and
Make re-checks with `^[^@\s]+@[^@\s]+\.[A-Za-z]{2,}$` before any write (integrity) — a
browser check only protects the form, not the webhook URL.

Normalised to lowercase/trimmed so casing variants cannot create duplicate patients.

### Patient Phone
`patient_phone` → `payload.data.patient_phone` → `patients.phone` (TEXT, nullable) → HubSpot `phone`

### Patient State
`patient_state` → `payload.data.patient_state` → `patients.state` (TEXT) → HubSpot `state`

Column is TEXT, not `varchar(2)` — full names like "Metro Manila" are valid.

### Service Specialty
`service_type` (select, required) → `payload.data.service_type` → `appointments.service_type`
(TEXT, NOT NULL)

```
ifempty( switch(service_type; "Select one..."; ""; service_type) ; "General Consultation" )
```

Two guards, because the placeholder is not empty. The select's first option has an **empty
value**, so the browser blocks submission until a real choice is made. Make additionally maps
the literal string `"Select one..."` to empty before the fallback — a plain `ifempty` would
not catch it, and that string was reaching the database before this was added.

### Preferred Consultation Date
| Layer | Value |
|---|---|
| Webflow input | `preferred_date` — required, `placeholder="YYYY-MM-DD"`, `pattern="\d{4}-\d{2}-\d{2}"` |
| Make | `ifempty(preferred_date; formatDate(now; "YYYY-MM-DD"))` |
| Supabase | `appointments.preferred_date` (DATE, nullable) |

Webflow's form field types are limited to text, email, password, tel, number and url — there
is **no date type**, and `type` is a reserved word so it cannot be added as a custom
attribute either. A native date picker would need custom code, which is gated behind a paid
Site plan. The `pattern` attribute is what makes this safe: the column is a real Postgres
`DATE`, so an unparseable value would fail the insert.

The today's-date fallback is protection, not a workaround — a submission arriving without a
date (e.g. posted directly to the webhook) would otherwise error the insert.

⚠️ Make's org timezone is **America/New_York** while the Webflow site is **Asia/Manila**.
This only affects the fallback, which could record the previous day for a late-evening
Manila submission.

### Patient Record ID (FK)
Supabase `patients.id` (UUID) → `appointments.patient_id` (UUID FK), sourced from the
**Search Rows** module (module 4) — not from the insert's own output. See the dedupe note.

### Patient Confirmation Email — Brevo
| Layer | Value |
|---|---|
| Module | `sendinblue:SendEmail` v2 (Brevo), connection `sendinblue2` |
| To | `lower(trim(patient_email))`, name from first + last |
| Sender / Reply-To | NovaCare Health &lt;phoeberhonegangoso@gmail.com&gt; |
| Subject | We've received your NovaCare booking request |
| Body tokens | first name, service type (same guard as above), preferred date, email |
| On error | `builtin:Resume` — a mail failure cannot take the pipeline down |

Gmail cannot be the sender. Three separate dead ends, all confirmed:

1. Make's Gmail module needs the restricted scope `https://mail.google.com/`; a connection
   with only `gmail.send` fails at runtime with `[403] insufficient authentication scopes`
2. Adding that scope manually returns Google's "This app is blocked"
3. Make refuses plain SMTP for Google accounts, and its Google Restricted type errors with
   *"not possible to use restricted scopes with customer @gmail.com accounts"*

Free Brevo sends from a `*.brevosend.com` address and appends its own footer; both go away
with domain authentication on a paid plan.

### Appointment Status
Default `'pending'`. `appointments.status` is a **Postgres enum** (`appointment_status`),
not TEXT. Allowed values only: `pending` · `confirmed` · `completed` · `cancelled`

### Timestamps
`created_at` (TIMESTAMPTZ) on both tables, default `timezone('utc', now())`.
HubSpot `createdate` is set by HubSpot.

### ✅ Duplicate-email dedupe — FIXED 2026-08-19
A returning patient used to throw
`409 duplicate key value violates unique constraint "patients_email_key"`, which tripped the
scenario's `maxErrors: 3` and auto-deactivated it.

Make's `upsertARecord` upserts on the **primary key only** and exposes no conflict target, so
it cannot dedupe on email. The working pattern:

1. Insert into `patients` — on a repeat email this errors, swallowed by `builtin:Resume`
2. **Search Rows** on `patients` where `email` = submitted email, limit 1
3. Appointment uses `{{4.id}}` from that search

Either way there is always a patient id, so new and returning patients both succeed.
Verified: one patient record with two linked appointments, no error.

Do NOT open the Search Rows module in the Designer and press OK without checking the Search
criteria filter survived — the criteria live in a filter, not in the column boxes.

### Error handling, by design
| Module | Handler | Why |
|---|---|---|
| 3 — insert patient | `Resume` | Duplicate email is expected, not fatal |
| 5 — insert appointment | none | If this fails the booking is lost; it *should* abort loudly |
| 6 — Brevo email | `Resume` | Mail failure must not block the CRM step |
| 7 — HubSpot contact | `Ignore` | CRM is downstream of the source of truth |

⚠️ A `Resume` handler makes the run report **SUCCESS** and still consume the operation even
when the module failed. The error appears only as "Handled error" on the module bubble in
the Make UI. The API's `executions_get-detail` returns just `{status}` on the free plan, so
the UI is the only place to see it. Always check the bubble, not the green tick.

### ⛔ HubSpot Deals — NOT BUILT
No deal module exists in the scenario; no deal has ever been created. `dealname`,
`dealstage` and `associations.contacts` are unimplemented. The portal has only the default
sales pipeline (Qualified To Buy → Contract Sent → Closed Won), which is wrong for clinical
intake, and pipelines cannot be created via the HubSpot MCP.

Deliberately deferred: those stages would duplicate `appointments.status`, and two systems
owning the same state drift apart. If HubSpot needs stage visibility, sync the status onto
the **contact** as a custom property rather than building a deal pipeline.

### ⛔ `intake_notes` — NOT BUILT
`appointments.intake_notes` (TEXT, nullable) exists; no form field or mapping writes it.

### Unmapped payload field
The published form now posts a `cf-turnstile-response` hidden field (Cloudflare Turnstile
bot protection, added by Webflow). It is ignored by the scenario and needs no mapping.

---

## 2. Internal operations — Supabase → Retool ⚠️ *not re-verified this revision*

Backed by `getAppointments.ts`, joining
`appointments a JOIN patients p ON p.id = a.patient_id`, ordered `created_at DESC`.
Name search is server-side and debounced 300ms; all other filtering is client-side.

| Source | Retool column | Rendering |
|---|---|---|
| `patients.full_name` | Patient Name | text, server-side search target |
| `patients.email` | Email | `mailto:` link |
| `patients.phone` | Phone | `tel:` link, em dash when null |
| `patients.state` | State | neutral badge |
| `appointments.service_type` | Service Type | text + client-side dropdown filter |
| `appointments.preferred_date` | Preferred Date | formatted date, em dash when null |
| `appointments.status` | Status | editable dropdown + client-side filter |
| `appointments.created_at` | Created At | timestamp, default sort DESC |
| `appointments.id` | (row key) | `UPDATE appointments SET status = $1 WHERE id = $2` |

**Metric cards** computed over all fetched rows (not the filtered subset): Total Intakes,
Pending Triage, Confirmed Today.

**Row click** opens a detail drawer with contact and appointment blocks plus a status
dropdown. The status cell stops event propagation so changing status does not open the
drawer.

**Status updates are optimistic**: local row state flips immediately, the mutation runs in
the background, and only the affected row rolls back on failure. No refetch on success.

The rollback must not restore a captured snapshot of the whole array — the `columns` memo
has an empty dependency list, so a captured `rows` would be the stale initial value and a
failed update would empty the table. Previous status is captured *inside* the functional
state updater for this reason.

---

## Known gaps

| Gap | Impact |
|---|---|
| No HubSpot deals | CRM has contacts but no triage pipeline (deliberate — see above) |
| `intake_notes` unpopulated | Column exists, nothing writes it |
| Supabase MCP auth failing | Database cannot be inspected or re-verified |
| Retool MCP disconnected | Dashboard cannot be inspected or re-verified |
| Make/Webflow timezone mismatch | Affects the preferred-date fallback only |
| Webflow free plan | 50 total form submissions; custom code unavailable |
