---
title: West End Glass Maintenance System code briefing
type: code-briefing
status: active
owner: Hudson
development_state: MVP
software_category: client implementation
updated: 2026-05-31
---

# West End Glass Maintenance System — code briefing

## Summary

This repository is the West End Glass Maintenance System: a WhatsApp-led field-service and machine-maintenance ticketing platform for West End Glass. Field technicians interact through WhatsApp/NFC-tag initiated machine IDs; administrators use a React web dashboard to manage tickets, users, machines, daily checks, manuals, audit logs, and reporting.

Steve confirmed on 2026-05-31 that the repository has been transferred from the `Virgelsnake` GitHub organisation to the DTP organisation, and that Joe/Joseph Ronnie has been actively working on it. Hudson verified the DTP repository at `Digital-Technology-Partner-ai/west_end_glass_maintenance_system` and cloned a canonical local working copy into DTP Coding Projects. The original Inbox copy remains provenance/source material and currently contains additional untracked marketing assets.

## Current state

- **DTP software category:** client implementation
- **Development state:** MVP
- **Commercial status:** needs controlled pilot / deployment hardening
- **Last verified:** 2026-05-31 15:48 BST
- **Works locally:** partial
- **Tests:** backend unit tests pass; frontend build passes; frontend lint fails on pre-existing code-quality issues
- **Main risks:**
  - The DTP GitHub repository is public at the time of verification. This may be intentional, but it should be confirmed because the project is client-specific and includes implementation details.
  - The repository tracks generated/system files: 43 `__pycache__` files and 2 `.DS_Store` files are currently tracked.
  - Backend test collection across the whole repo fails if `test/test_whatsapp_template.py` runs without a real `backend_api/.env`; backend API tests pass when scoped to `backend_api/tests` with dummy non-secret environment values.
  - Frontend lint currently fails with 18 errors and 1 warning, including a recent `Tickets.jsx` hook/immutability issue.
  - `npm ci` reports 5 frontend dependency vulnerabilities: 3 moderate and 2 high. No fix was applied during briefing.
  - Deployment uses real external services and WhatsApp/Meta/Claude credentials. Do not run live WhatsApp tests or mutate production data without Steve's explicit approval.

## How to run

Verified from the canonical local copy:

```bash
cd /Users/hudsonrebel/DTP Coding Projects/west_end_glass_maintenance_system
```

Docker Compose path from `QUICKSTART.md` and `docker-compose.yml`:

```bash
docker compose up -d
```

Expected local services from the repo documentation/config:

- MongoDB service: `weg-mongo` on the internal Docker network only
- FastAPI backend: container `weg-api`, host port `8835` mapped to container port `8000`
- Admin frontend: container `weg-frontend`, host port `3030` mapped to container port `3000`

This full Docker run was not executed during briefing because it requires a real `backend_api/.env` containing operational credentials. The frontend production build was verified independently.

Frontend development commands:

```bash
cd /Users/hudsonrebel/DTP Coding Projects/west_end_glass_maintenance_system/backend_admin_page
npm ci
npm run dev
npm run build
```

Backend local development commands from `backend_api/README.md`:

```bash
cd /Users/hudsonrebel/DTP Coding Projects/west_end_glass_maintenance_system/backend_api
pip install -r requirements.txt
```

The backend imports require environment variables. Use dummy values for local unit tests only; do not commit `.env` files.

## How to test

Verified commands and results:

```bash
# backend test environment created outside the repo
/opt/homebrew/bin/python3.12 -m venv /tmp/weg_backend_venv
/tmp/weg_backend_venv/bin/pip install -r /Users/hudsonrebel/DTP\ Coding\ Projects/west_end_glass_maintenance_system/backend_api/requirements.txt

cd /Users/hudsonrebel/DTP Coding Projects/west_end_glass_maintenance_system
META_WHATSAPP_TOKEN=*** \
META_PHONE_NUMBER_ID=dummy \
META_VERIFY_TOKEN=*** \
META_APP_SECRET=*** \
ANTHROPIC_API_KEY=*** \
JWT_SECRET_KEY=*** \
ADMIN_PASSWORD=*** \
/tmp/weg_backend_venv/bin/python -m pytest backend_api/tests -q
```

Result on 2026-05-31:

```text
47 passed, 140 warnings in 2.65s
```

Whole-repo pytest was also attempted:

```bash
/tmp/weg_backend_venv/bin/python -m pytest backend_api/tests test -q
```

Result: failed during collection because `test/test_whatsapp_template.py` exits when `backend_api/.env` is absent. This appears to be a real test-harness limitation rather than an application failure.

Frontend verification:

```bash
cd /Users/hudsonrebel/DTP Coding Projects/west_end_glass_maintenance_system/backend_admin_page
npm ci
npm run lint
npm run build
```

Results on 2026-05-31:

- `npm ci`: passed; reported 5 vulnerabilities, 3 moderate and 2 high.
- `npm run lint`: failed with 18 errors and 1 warning.
- `npm run build`: passed; Vite built `dist/` successfully and warned that one chunk is larger than 500 kB.

The most significant lint finding is in `backend_admin_page/src/pages/Tickets.jsx`: `load` is accessed before declaration inside a `useEffect`, reported by `react-hooks/immutability`. Other failures are unused variables, React Fast Refresh export-shape warnings, and `__dirname` in `vite.config.js`.

## Architecture map

- `backend_api/` — FastAPI backend for admin authentication, technician authentication, webhook handling, WhatsApp message processing, tickets, machines, users, audit, photos, manuals, daily checks, ticket types, dashboard data, and simulation endpoints.
- `backend_api/app/main.py` — FastAPI app wiring, startup/shutdown hooks, router inclusion, `/settings/public`, and `/health`.
- `backend_api/app/models/` — Pydantic/Mongo-facing models for admin users, machines, tickets, users, messages, manuals, dailies, audits, and ticket types.
- `backend_api/app/routers/` — API route modules for admin auth/admins/audit/dashboard/dailies/machines/manuals/messages/photos/simulate/technician auth/technician tickets/ticket types/tickets/users/webhook.
- `backend_api/app/services/` — domain services for audit logging, Claude agent handling, daily scheduling, interactive handling, message processing, ticket logic, and WhatsApp integration.
- `backend_admin_page/` — React/Vite admin frontend.
- `backend_admin_page/src/pages/` — main dashboard pages: Admins, AuditLog, Calendar, Dailys, Dashboard, Files, Login, Machines, SettingsPage, TicketDetail, Tickets, Users, plus technician pages.
- `backend_admin_page/src/components/` — shared UI components including layout/sidebar/navbar, protected routes, conversation view, photo gallery, attachment modal, and ticket step editors.
- `tools/` — CLI tools for admin tasks, importing daily checklists, resetting/seeding data, and simulating WhatsApp conversations.
- `test/` — WhatsApp template/text scripts. These are not safe default tests because they expect live `.env` credentials and may send real WhatsApp messages.
- `deploy/` — production compose file.
- `Documentation/` — API specification, technical overview by Joe, import material, WhatsApp webhook notes, and project contribution/code-of-conduct files.

High-level data flow:

1. Technician taps an NFC tag or sends a machine ID through WhatsApp.
2. Meta Cloud API sends the webhook to the FastAPI backend.
3. Backend authenticates the phone/user, finds the relevant machine/ticket, and routes the message through the ticket/message/Claude-agent flow.
4. The system writes messages, step completions, photos, and audit events into MongoDB/file storage.
5. Administrators manage the workflow through the React dashboard.

## Dependencies and services

Environment variable names identified from `backend_api/app/config.py`:

- `MONGODB_URL`
- `MONGODB_DB_NAME`
- `META_APP_ID`
- `META_WHATSAPP_TOKEN`
- `META_PHONE_NUMBER_ID`
- `META_VERIFY_TOKEN`
- `META_APP_SECRET`
- `META_WABA_ID`
- `WHATSAPP_BUSINESS_NUMBER`
- `WEST_END_CALL_BACK_URL`
- `WEST_END_CALL_BACK_TOKEN`
- `ANTHROPIC_API_KEY`
- `JWT_SECRET_KEY`
- `JWT_ALGORITHM`
- `JWT_ACCESS_TOKEN_EXPIRE_MINUTES`
- `ADMIN_USERNAME`
- `ADMIN_PASSWORD`
- `PHOTO_STORAGE_PATH`
- `MANUAL_STORAGE_PATH`
- `CORS_ORIGINS`

External/service dependencies:

- MongoDB 7
- Meta WhatsApp Cloud API / WhatsApp Business account
- Anthropic Claude API
- JWT admin authentication
- File storage volumes for photos and manuals
- Docker Compose for local/containerised operation
- Node/Vite/React frontend stack
- Python FastAPI backend stack

Do not store environment variable values in the repo, wiki, Kanban, chat, or briefing files.

## Repository hygiene

Canonical local repo:

```text
/Users/hudsonrebel/DTP Coding Projects/west_end_glass_maintenance_system
```

GitHub repo:

```text
https://github.com/Digital-Technology-Partner-ai/west_end_glass_maintenance_system
```

Inbox/source copy inspected:

```text
/Users/hudsonrebel/My Drive/DTP Inbox/west_end_glass_maintenance_system
```

Sync findings on 2026-05-31:

- The DTP org repository exists and Hudson has `ADMIN` permission.
- The Inbox repo originally still pointed at `https://github.com/Virgelsnake/west_end_glass_maintenance_system.git` and was 8 commits behind DTP `main`.
- Hudson repointed the Inbox repo origin to the DTP org URL and fast-forwarded it to commit `9fa1486`.
- The canonical DTP Coding Projects clone is clean at `9fa1486` before this briefing file is committed.
- The Inbox copy remains dirty after fast-forward:
  - deleted tracked files: `.DS_Store`, `backend_api/.DS_Store`
  - untracked files: `Documentation/marketing/west-end-glass-sales-overview.excalidraw`, `Documentation/marketing/west-end-glass-sales-overview.png`
- The repo currently tracks 43 `__pycache__` files and 2 `.DS_Store` files. This should be cleaned with a normal Git hygiene commit if Steve approves the cleanup, not with history rewrite.
- Backend test verification created untracked `__pycache__` files in the canonical clone. Tracked generated-file modifications were restored; untracked generated files were left in place rather than deleted without approval.

## Related DTP records

- **Wiki product page:** None yet
- **Wiki project page:** `[[west-end-glass-machine-service-manager]]`
- **Working Files folder:** `/Users/hudsonrebel/My Drive/DTP Working Files/Projects/Active_Projects/Westend Glass/Machine service manager`
- **Inbox/source folder:** `/Users/hudsonrebel/My Drive/DTP Inbox/west_end_glass_maintenance_system`
- **Kanban board:** `coding-projects`
- **Related client/project:** `[[west-end-glass]]`, `[[joe-ronnie]]`

## Open questions

1. Should the DTP GitHub repository remain public, or should it be made private now that it is under the DTP organisation?
2. Should the untracked Inbox marketing assets be moved into the West End Glass Working Files project folder, committed to the repository, or left as Inbox provenance for now?
3. Should Hudson proceed with a repo hygiene PR/commit to untrack `__pycache__` and `.DS_Store`, expand `.gitignore`, and remove verification residue?
4. Should frontend lint fixes and dependency audit fixes be treated as immediate pilot-readiness work, or deferred until Joe has finished the current feature branch/workstream?
5. Is the canonical client/project name for delivery still “West End Glass Machine Service Manager”, or should the current system be renamed to “West End Glass Maintenance System” across wiki/project records?

## Next recommended actions

### Hudson-owned

- Create/update the DTP wiki project record with the DTP GitHub URL, canonical local repo path, latest verification results, and current open risks.
- Create a `coding-projects` follow-up card for pilot-readiness cleanup covering frontend lint, dependency audit triage, and generated-file repo hygiene.
- If Steve approves, run a scoped hygiene pass: update `.gitignore`, `git rm --cached` generated/system files, remove local verification residue, rerun backend tests/frontend build, then commit and push.

### Steve-decision items

- Decide whether the GitHub repo should be public or private.
- Decide what to do with the Inbox-only marketing assets.
- Decide whether Hudson should touch Joe's current codebase now, or leave engineering cleanup until Joe confirms his current work is stable.

## Steve's notes

- 2026-05-31: Steve said the West End Glass maintenance system has been transferred from the `Virgelsnake` organisation to the DTP organisation. He asked Hudson to run the briefing process on the Inbox folder and check that the local files are in sync with the repo. Steve also noted that the repo should be the most up to date because Joe/Joseph Ronnie has been working on the project.
