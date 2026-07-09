# Circle of Care — Ops Repo

This repo is the source of truth for the Circle of Care autonomous bid-support pipeline. It contains the operating law specification (`OPERATING-LAW-AND-DEPLOYMENT-SPEC.md`) and the literal n8n workflow definitions (`workflows/`) that implement it.

n8n imports from this repo. This repo never imports from n8n. Nothing here is live until it's deliberately deployed following the order below.

## What's in `workflows/`

| File | Workflow | Depends on |
|---|---|---|
| `01-trigger.json` | Polls for new projects, hands off to Decode | — |
| `02-decode.json` | Reads project files, extracts facts with citations, emits `contacts.json` | 01 |
| `03-tier-map-resolution.json` | Resolves pricing, queues Manual Lever 1 exceptions | 02 |
| `04-build-master-set.json` | Drafts deliverables from decoded facts | 03 |
| `05-derive-tier-packages.json` | Slices master set into per-tier packages | 04 |
| `06-contact-research.json` | Public-record fact per contact via self-hosted SearXNG | 05 |
| `07-email-templates.json` | Fills the fixed brand shell with per-project values | 06 |
| `08-web-pipeline.json` | NDA/Intake/Payment webhook handler | — (client-facing, always live) |
| `09-nda-signature-handler.json` | Signature capture, backgrounded PDF generation | — (client-facing) |
| `10-payment-nda-watcher.json` | IMAP poll, reference-code payment matching | — (scheduled, every 2h) |
| `11-hold-release-processor.json` | Token minting after the 4-day hold, clamped to deadlines | — (scheduled, hourly) |
| `12-follow-up-watcher.json` | One NDA-drop, one payment-drop reminder | — (scheduled, twice daily) |
| `13-bounce-ooo-processor.json` | Bounce/OOO escalation chain | — (post-send) |
| `14-unsubscribe-endpoint.json` | Public unsubscribe link handler | — (client-facing, always live) |
| `15-launch.json` | Daily 7pm send sequence | — (scheduled, daily) |
| `16-carousel-builder.json` | Expired-RFP LinkedIn carousel | — (event-triggered) |
| `17-engagement-picker.json` | Daily 30-contact LinkedIn draft queue | — (scheduled, daily) |
| `18-linkedin-token-refresh.json` | OAuth token rotation to a plain file | — (scheduled, hourly) |
| `19-quality-gate.json` | Two-immutables check before deploy | 07 |
| `20-deploy.json` | Copies derived packages to `public_html`, enables Launch | 19 |

## One-time manual bootstrap (cannot be automated)

1. Generate an n8n API key through the n8n UI. Store it as the `N8N_API_KEY` environment variable on the server.
2. Set up LinkedIn OAuth app credentials (`LINKEDIN_CLIENT_ID`, `LINKEDIN_CLIENT_SECRET`, `LINKEDIN_REFRESH_TOKEN`) as environment variables.
3. Set SMTP/IMAP credentials in n8n's encrypted credential store, using fixed, predictable names (e.g. `circleofcare-smtp`, `circleofcare-imap`) so re-imports resolve without manual re-linking.

None of these can be created by an automated script — each requires a human action inside an authenticated session on the account that owns it.

## Deploy order

1. Scaffold the repo (done).
2. Import all workflow JSONs into n8n (in dependency order per the table above, or all at once — n8n resolves `executeWorkflow` references by internal ID once both workflows exist).
3. **Explicit activation step.** Imported workflows do not activate automatically. Every workflow with a trigger (schedule or webhook) must be explicitly set `active: true` via the n8n API, authenticated with `N8N_API_KEY`. Confirm this before proceeding — an imported-but-inactive workflow will silently refuse all webhook traffic.
4. Credential-binding audit — walk every credentialed node (email, IMAP, LinkedIn) and confirm it resolves to a real credential in this instance, not left in an unconfigured state from import.
5. Dry run against a dummy project folder. Confirm Quality Gate passes.
6. Live.

## What this repo does not contain

- Any real credential, token, or secret. All of those live in environment variables or n8n's encrypted store, never here.
- Client data, contact lists, or anything project-specific. Those live in `Working\` on the server, isolated per project, and are never committed.
- The actual n8n instance. This repo is the specification and the workflow definitions; the running system is a separate thing that imports from here.

## Placeholder workflow IDs

Several workflow files reference `PLACEHOLDER_*_WORKFLOW_ID` in their `executeWorkflow` nodes. These get filled in with real internal IDs once both the calling and called workflows exist inside the same n8n instance — that's a wiring step done after import, not something fakeable in a file that hasn't been imported anywhere yet.
