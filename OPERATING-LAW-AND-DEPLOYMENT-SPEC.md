# Circle of Care — Autonomous Bid Support: Operating Law & Deployment Spec

This document and the project's own source files are the only two references any agent may act on. Where this document and reality disagree, this document wins and reality gets corrected.

---

## PART A — OPERATING LAW

### A0. What this business is

Circle of Care is the offshore back-end bid-support provider. For a government solicitation, an intel source identifies who downloaded/was awarded it. Circle of Care contacts those organizations offering paid preparation of the final documents they submit. Circle of Care is never the bidder and never the submitter — the client submits; Circle of Care prepares the ready-to-deliver documents based on client data. Never imply Circle of Care files or submits anything on the client's behalf.

Contacts are not all confirmed decision-makers. Everyone on the list gets contacted. No pre-filtering, no cleaning the list. Qualification happens by who responds, never by who gets removed.

Solicitation size never expands the build. Same template set regardless of domain or scale.

The pipeline is completely closed. No raw template, internal structure, or intermediate layout is ever shown or shared to clients. Clients interface exclusively with an interactive questionnaire generated dynamically per project. The final, completed document is the first asset that hits their secure portal view after submission.

### A1. Autonomy boundary

**Manual lever 1 — Tier Map exception.** Fires when a required component has no match in the default pricing table (A4), when the Deadline Law (A8) finds the solicitation already expired at intake, or when Decode cannot extract a clean funding figure for the ROI-based Full tier. Blocks pricing/launch for the affected scope only.

**Manual lever 2 — White-label release.** Full-bundle-for-free never fires on its own. Operator selects the contact and triggers release from the dashboard. No hold applies.

### A2. Workspace law (filesystem-enforced)

Three zones, one-direction flow: **Originals → Working → Delivered.**

**Hook-enforced, non-negotiable:** no renaming files; no moving files — copy only; new files named after their parent folder; generic names blocked.

**Clarification — operational state files are not Originals.** `contacts.json`, `send_log.json`, `suppression.json`, `pipeline.json`, `engagement_log.json`, `linkedin_queue.json`, `reminder_log.json`, and `ai_corrected_contacts.json` live inside Working/Delivered as live operational state, edited in place. The bounce-correction queue and reminder log are the deliberate, documented exceptions to the no-parallel-file rule, both solving lock collisions.

**One zone outside this law by necessity:** `public_html/intake_cache/` — a short-lived, public-facing staging area for live form autosave data.

### A3. New project intake, in order

1. **Source manifest snapshot:** before any AI call, a deterministic step snapshots the manifest of every file in the project root tree. A parallel index of active project directories/prefixes is maintained the same way — this is what the web pipeline validates incoming requests against (A9). Every downstream step operates exclusively against the frozen manifest, never a live re-scan. Secondary guard: hardcoded ignore pattern for known operational filenames. Files created after intake began are ineligible as evidence, forever.
2. **Decode:** read every recognized file, at any depth, recursively.
   - **File filter:** `.md`, `.json`, `.pdf`, `.docx`, `.xlsx` only. Anything else, or over 25MB, skipped and logged.
   - **Image files:** never text-processed. Filenames suggesting relevance flagged, not dropped.
   - **Duplicate-format files:** `.docx`/`.pdf` sharing a base filename, matched with paths canonicalized and case-folded for the comparison only — only the `.docx` processed.
   - **Per-file isolation, strictly sequential:** one file at a time, never in parallel. Buffer explicitly dereferenced before the next file loads.
   - **Chunked decode (context law):** structural-boundary splits, 25% overlap, deterministic merge, every fact carries a citation. No citation, no entry.
   - **Funding figure extraction:** unresolved → floor price + Manual Lever 1.
   - **Contact file emission:** locates the contact database anywhere in the tree, emits project-isolated `contacts.json`.
   - **Dynamic questionnaire schema emission:** every field name passes a strict allowlist — alphanumeric and underscore only, hard-capped at 40 characters — blocked from matching `__proto__`, `constructor`, or `prototype`.
3. Create the Project Document.
4. Resolve Tier Map (A4). Deadline Law check (A8).
5. Build deliverables (A5) → email templates → automation → web pipeline → dashboard → LinkedIn assets (Part C).
6. Quality Gate (A16).
7. Deploy, launch.

### A4. Tier Map — single source of truth

| Tier | Components | Price | Intake scope | Token unlocks |
|---|---|---|---|---|
| **Lowest** | Compliance checklist & requirements review | $0.00 — sent automatically, no gate, in E1 | Public record only | Checklist only |
| **Mid** | One standalone component | Read from the flat-rate table below | Base + component-specific | Selected document only |
| **Full** | Complete ready-to-submit proposal package | `Max(1.5% of total grant/solicitation value, $2,500 floor)`. No clean figure → floor + Manual Lever 1 | Complete intake matrix | Everything |

**Flat-rate table for Mid-tier components** — $1,500 is the floor across every standalone offering; nothing prices below it, benchmarked against real market rates for comparable proposal/grant-support work:

| Component | Price |
|---|---|
| Post-award closeout/reporting package | $1,500–$5,000, scaled to complexity |
| Standard turnkey drafting / formatting / credential alignment | $1,500 flat |

Unmatched component → exception queued (Manual Lever 1). Price identical across all five locations, all generated from the Tier Map. Currency canonicalization bound to the write operation, `.toFixed(2)` at the moment of write. Exception resolution recursively re-enters Build for affected components only.

### A5. Product deliverable standard

1. **Compliance checklist** — pass/fail, every scored item a live formula. Free, no gate, E1.
2. **End-to-end proposal drafting** — evidenced from the source manifest.
3. **SOW draft and budget template** — every rate a live formula.
4. **Sustainability plan** — where the program calls for one.
5. **Post-submission interview prep / closeout reporting** — where the track requires it.

### A6. Email sequence

| # | Type | Content | Audience | Timing |
|---|---|---|---|---|
| E1 | OPEN | Opportunity, free checklist no gate, public-record credibility | Everyone | Day 0, 7:00 PM operator's timezone |
| E2 | VALUE | Insight, ROI-justification | Everyone who received E1 | 24h after confirmed E1 delivery |
| E3 | CLOSE | NDA as gateway to paid tiers + Data Policy | Everyone who received E2 | 24h after confirmed E2 delivery |
| E4 | FULL | Full bundle paid upsell | Non-NDA holdouts who received E3 | 24h after confirmed E3 delivery |
| E5 | WHITE LABEL | Full access free | One contact, operator-selected | Event-triggered, no hold |

**Confirmed delivery:** SMTP accepted, no hard bounce by the next Bounce Processor run.

**Conversion check:** unbuffered, live, immediately pre-send. A contact who signed the NDA but hasn't paid is not eligible for E4 by design — that contact's path is the dedicated payment-drop reminder (B1), not the cold-outreach upsell.

**Overdue throttling:** capped per run, drained across runs.

**Data Policy — embedded verbatim in E3:**

> All information you provide is used solely to prepare the documents requested under this engagement. It is never sold or shared with any third party. All data is transmitted and stored on encrypted, access-controlled systems, and is covered under the signed NDA for the duration of the engagement and 24 months after its conclusion. You may request permanent deletion of your data at any time; it will be completed within 10 business days.

### A7. Deliverability & compliance law

Dedicated subdomain. SPF/DKIM/DMARC verified. Domain warmed. Throttled sends. `List-Unsubscribe` headers + visible link + physical address on every email. Unsubscribes/complaints → Global DNC automatically.

### A8. Deadline law

Future date → normal pipeline. Past date → Manual Lever 1, triggers Carousel Builder. Mid-sequence: no E1–E4 after due date. Hold clamp = earlier of hold expiry or 24h before due date. Post-expiry paid delivery ships regardless. Countdown never negative. Dry-run compresses schedule to fit short runways.

### A9. Flow after email click & intake loop

```
Click → NDA (mandatory for paid engagement) → Signature captured → Intake fields unlock
  → Paid path: upsell → payment → intake customizes
        → Token issued after hold (clamped per A8) → Portal serves final derived package
```

The free checklist already went out unconditionally in E1 — this flow governs paid tiers only.

Every incoming request to the web pipeline — NDA, Intake, or Payment — is checked at the top of the handler for a valid project identifier against the current index of active project directories (A3) before any file path is constructed. Missing, empty, or non-matching identifiers return `400` immediately.

Intake fields autosave into `public_html/intake_cache/`, using an explicit exclusive write lock (`LOCK_EX`). Never writes directly into `Working\`. A separate step validates against the project's schema — read with a brief retry-with-backoff (up to 3 attempts over ~150ms) — and copies the completed submission into the backend path.

**Signature capture:** typed name, timestamp, IP, consent action. This record is validated and saved synchronously into that project's own durable operational state, contact-hash-scoped — not only into the ephemeral `intake_cache/`. This instantly unlocks Intake. The secondary steps — the branded countersigned PDF via `pdf-lib` and the confirmation email — fork completely into background processing. On any exception, retries are staggered with an explicit delay, and the worker explicitly releases its execution token. The failing job is stripped of its heavy payload entirely while waiting: only minimal metadata is serialized to `Working/cache/retry_queue.tmp` — a pointer to the durable signature record, never a duplicate of it. The PDF's own output paths are scoped by contact hash plus a fresh random run identifier.

### A10. Delivery/portal standard

Fully automated. Token-gated. **Fails closed, never open.** Re-validates every request. Master set never web-reachable.

### A11. Bounce policy

| Bounce type | Action |
|---|---|
| Hard bounce, known-typo domain | Correction writes to `ai_corrected_contacts.json`, resend once. `contacts.json` untouched. |
| Hard bounce, no known typo | One official address, send once |
| Still hard bounce | One further official address, send once |
| Still hard bounce (both fail) | Stop, suppress, write Global DNC |
| Soft bounce | No special retry |
| Out-of-office | Parse alternate. Found → resend once. Not found/fails → official-address escalation. Alternates never tracked. |
| Spam/abuse complaint | Immediate suppression, Global DNC |
| Unsubscribe | Immediate, automatic, Global DNC |

**Read rule:** correction queue checked before every send. **"Known typo":** fixed lookup table only. **OOO circuit breaker:** fires at most once per original send. **Global Do-Not-Contact:** `Shared\global_dnc.txt`, append-only, SHA-256 hashed.

### A12. Security law

Master files never web-reachable; portal fails closed. Tokens cryptographically random, revocable. Credentials only as environment variables or n8n's encrypted store — never in the repo. All transport encrypted.

**Public web layer has no path into `Working\`, full stop.**

**Directory traversal defense:** `path.resolve()` + allowlist. Case-folding is scoped strictly to the comparison boundary — the original, case-preserved absolute path is what's passed to every actual filesystem operation.

**Webhook validation:** raw body as text, `JSON.parse()` in code-controlled try/catch, `400` on malformed. Project identifier validated before any file lookup. Intake two-layered: syntax, then schema, the schema read using retry-with-backoff.

**Duplicate-submission protection:** per-contact-hash lock, heartbeat-refreshed, 5-minute staleness.

### A13. Dashboard standard

Full-page admin gate. Panels shown only when populated: callback requests, white-label release, OOO redirects, complaint flags, Today's 30 (C2), carousel fallback (C1). Pipeline stages: **Contacted → Responded → NDA Signed → Intake Complete → Paid/Granted → Delivered.**

### A14. AI/compute constraints

Zero third-party AI API, all local via Qwen/Ollama. One box, 8GB RAM/100GB disk. Qwen 3–4B, Q4_K_M, `OLLAMA_NUM_CTX=2048`. Container hard-capped 4GB, never resident 24/7. **AI job concurrency = 1** — applies to inference calls specifically; deterministic Code-node work never occupies this slot. Layer 2 runs a genuinely different model family than the builder.

### A15. Non-negotiables

No invented figures. No contact excluded. Full bundle never auto-releases. Master file never web-reachable. No cloning a past project. Portal fails closed. No parallel delta file except the two documented exceptions. No third-party AI API. Global DNC checked before every send. No file halts a batch. No path without canonicalized, comparison-only case verification. No price non-canonical. No standalone component prices below the $1,500 floor. No duplicate submission processed twice. No send to an already-converted contact, checked live. No fact without a citation resolving inside the frozen manifest. No email without unsubscribe. No client data in any public post. Carousel capped at one/day. No credential in the repo. No AI-generated field name enters a schema without the allowlist, cap, and blocklist. No PDF generation ever blocks the live signing session. No credentialed node goes into production after a Git import without its binding confirmed. No OAuth token refresh depends on `process.env` reflecting a post-startup write. No background retry holds a heavy payload in memory across its delay window. No intake schema read is a single unbuffered attempt. No file path lookup is ever attempted against an unvalidated project identifier. No payment reference comparison runs on unnormalized text. No LinkedIn document upload assumes a single-request transfer regardless of size. No automated deployment script authenticates without a pre-generated, human-issued API credential.

### A16. Quality Gate — two-immutables check

Retry cap 3 per component. Cap hit → human exception, checked against the two immutables directly.

**Layer 1 (deterministic):** price-sync diff. Placeholder scan. Formula-liveness check. Numeric sweep. Portal fail-closed test. Unsubscribe validation. Date sanity check. Public-post scan. Schema field-name check.

**Layer 2 (independent, different model family):** claim-by-claim, manifest-validated citations only.

### A17. Final steps

Dry run → Quality Gate passes → clean Working → commit to Delivered → deploy and launch, automatic.

---

## PART B — N8N DEPLOYMENT SPEC

### B1. Pipeline → n8n mapping

| Step | n8n Component |
|---|---|
| Decode | Frozen manifest → sequential file walk, comparison-only case folding → chunked inference, 25% overlap → citations → `contacts.json` |
| Build master set | Template + inference fill, emits schema with allowlist/cap/blocklist |
| Web pipeline | Project identifier validated → `intake_cache/` only, exclusive-lock writes → schema validation (retry-with-backoff) → lock file → correction queue check → `200 OK` → background fork |
| Payment/NDA Watcher | Every 2 hours: IMAP poll. Both reference code and email text normalized (whitespace stripped, uppercased) before comparison — reference code first, amount as secondary guard. Scans abuse/FBL. |
| Follow-up Watcher | One NDA-drop, one payment-drop reminder each — writes exclusively to `reminder_log.json` |
| NDA Signature Handler | Validates synchronously, saves durable signature record → `200 OK` → forks PDF generation, staggered retry, memory cleared, retry queue holds a pointer only |
| Carousel Builder | Deadline Law expiry → decoded JSON → branded template → `pdf-lib` → LinkedIn's multi-step document upload flow → dashboard fallback if needed |
| Engagement Picker | Per-project dedup + 7-day global coherence window |
| LinkedIn Token Refresh | Writes token to a plain file; carousel node reads fresh via direct file read |
| Launch | Unbuffered live conversion check immediately pre-send |

### B2. Contact research — locked

Self-hosted SearXNG. Idempotent `docker stop`/`docker rm` against the named container (`circleofcare-searxng`) before each startup. Started on-demand, stopped after.

### B3. Disk/execution hygiene

Backups: flat rsync-style sync, no compression on live paths, 3:00 AM local, `nice -n 19`/`ionice -c 3`, 14-day retention. Launch aborts below 5GB free.

### B4. Sandbox allowlist

```
NODE_FUNCTION_ALLOW_EXTERNAL=exceljs,pdf-parse,mammoth,pdf-lib
NODE_FUNCTION_ALLOW_BUILTIN=fs,path
```

### B5. File type map

Per-page scanned-document guard: any single page under 100 characters fails closed.

### B6. Model sizing

Builder Qwen 3–4B; Layer 2 a different model family; contact research a smaller model. One loaded at a time.

### B7. Global DNC implementation

`Shared\global_dnc.txt`. Read: `fs.statSync` size+mtime dual signal per send. Write: one centralized, serialized workflow.

### B8. Bounce/OOO escalation logic

```
Fail → one official address → send once → fails → one further official address → send once → fails → stop, suppress, write Global DNC
```

### B9. System-hardening & reliability rules (consolidated)

Sequential file processing. Frozen-manifest-only citation lookups. Named-container SearXNG cleanup. Staggered PDF retries with explicit slot release and memory clearing. Live conversion checks, E4 correctly scoped. Schema field-name allowlist/cap/blocklist. Exclusive-lock autosave writes. Retry-with-backoff on schema reads. Path canonicalization comparison-only. Project identifier validated before any lookup. Payment reference matching normalized both sides. LinkedIn uploads use the real multi-step handshake. Post-import activation authenticated with a pre-generated API key. LinkedIn token via direct file read. $1,500 floor enforced across every standalone Mid-tier component, no exceptions.

---

## PART C — LINKEDIN LAYER

**C1:** Deadline-expiry-triggered carousel, manifest-cited public analysis only, branded template via `pdf-lib`, posted through LinkedIn's registered multi-step upload flow, one post/day, dashboard fallback.

**C2:** Daily 30-contact picks, per-project dedup + 7-day cross-project coherence window, manual post from the dashboard.

---

## PART D — GITHUB DEPLOYMENT LAW

The repo is the source of truth. Credentials never touch the repo. The LinkedIn token is read from a file at execution time. Webhook paths static.

**One-time manual bootstrap, at initial box setup only:** an n8n API key generated through the UI, stored as `N8N_API_KEY`. Every automated script authenticates with it.

**Deploy order:** repo scaffold → workflow JSONs → import → explicit activation step (authenticated via `N8N_API_KEY`), confirmed → credential-binding audit → dry run on a dummy project → Quality Gate passes → live.
