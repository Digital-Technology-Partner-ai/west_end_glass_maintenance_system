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

Steve confirmed on 2026-05-31 that the repository has been transferred from the `Virgelsnake` GitHub organisation to the DTP organisation, and that Joe/Joseph Ronnie has been actively working on it. Hudson verified the DTP repository at `Digital-Technology-Partner-ai/west_end_glass_maintenance_system` and cloned a canonical local working copy into DTP Coding Projects. Steve chose to keep the DTP GitHub repository public for now and approved a safe hygiene/lint pass.

## Current state

- **DTP software category:** client implementation
- **Development state:** MVP
- **Commercial status:** needs controlled pilot / deployment hardening
- **Last verified:** 2026-05-31 16:17 BST
- **Works locally:** partial
- **Tests:** backend unit tests pass; frontend lint passes; frontend build passes
- **Main risks:**
  - The DTP GitHub repository remains public by Steve's 2026-05-31 instruction. This is intentional for now, but the project is client-specific and should be revisited before wider deployment or external sharing.
  - Backend test collection across the whole repo fails if `test/test_whatsapp_template.py` runs without a real `backend_api/.env`; backend API tests pass when scoped to `backend_api/tests` with dummy non-secret environment values.
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
/opt/homebrew/bin/python3.12 -m venv /tmp/weg-backend-venv312
/tmp/weg-backend-venv312/bin/pip install -r /Users/hudsonrebel/DTP\ Coding\ Projects/west_end_glass_maintenance_system/backend_api/requirements.txt

cd /Users/hudsonrebel/DTP Coding Projects/west_end_glass_maintenance_system
META_WHATSAPP_TOKEN=dummy \
META_PHONE_NUMBER_ID=dummy \
META_VERIFY_TOKEN=dummy \
META_APP_SECRET=dummy \
ANTHROPIC_API_KEY=dummy \
JWT_SECRET_KEY=dummy \
ADMIN_PASSWORD=dummy \
/tmp/weg-backend-venv312/bin/python -m pytest backend_api/tests -q
```

Result on 2026-05-31 after the safe hygiene/lint pass:

```text
47 passed, 140 warnings in 1.76s
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

Results on 2026-05-31 after the safe hygiene/lint pass:

- `npm ci`: passed; reported 5 vulnerabilities, 3 moderate and 2 high.
- `npm run lint`: passed.
- `npm run build`: passed; Vite built `dist/` successfully and warned that one chunk is larger than 500 kB.

The safe lint pass fixed the `Tickets.jsx` hook/load ordering issue, unused frontend imports/variables, React Fast Refresh export-shape warnings by moving `useAuth` into a dedicated hook, and the ESM `__dirname` issue in `vite.config.js`.

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

Original Inbox/source copy inspected, then moved out of the live Inbox during closeout:

```text
/Users/hudsonrebel/My Drive/DTP Working Files/Projects/Active_Projects/Westend Glass/Machine service manager/processed-source/west_end_glass_maintenance_system__processed_20260531
```

Sync findings on 2026-05-31:

- The DTP org repository exists and Hudson has `ADMIN` permission.
- The Inbox repo originally still pointed at `https://github.com/Virgelsnake/west_end_glass_maintenance_system.git` and was 8 commits behind DTP `main`.
- Hudson repointed the Inbox repo origin to the DTP org URL and fast-forwarded it to commit `9fa1486`.
- The canonical DTP Coding Projects clone is clean at `9fa1486` before this briefing file is committed.
- The original Inbox copy was moved to the Working Files `processed-source/` folder after briefing closeout so the live DTP Inbox is not used as an archive. It remains a dirty provenance snapshot after fast-forward:
  - deleted tracked files: `.DS_Store`, `backend_api/.DS_Store`
  - the former untracked marketing files were moved to the Working Files project folder: `/Users/hudsonrebel/My Drive/DTP Working Files/Projects/Active_Projects/Westend Glass/Machine service manager/Marketing`
- The safe hygiene pass expanded `.gitignore` for macOS/Python generated files and removed tracked `.DS_Store` and `__pycache__` artefacts from the Git index using normal Git history, not history rewrite. Local generated files were left on disk.

## Related DTP records

- **Wiki product page:** None yet
- **Wiki project page:** `[[west-end-glass-machine-service-manager]]`
- **Working Files folder:** `/Users/hudsonrebel/My Drive/DTP Working Files/Projects/Active_Projects/Westend Glass/Machine service manager`
- **Processed source snapshot:** `/Users/hudsonrebel/My Drive/DTP Working Files/Projects/Active_Projects/Westend Glass/Machine service manager/processed-source/west_end_glass_maintenance_system__processed_20260531`
- **Kanban board:** `coding-projects`
- **Related client/project:** `[[west-end-glass]]`, `[[joe-ronnie]]`

## Open questions

1. Is the canonical client/project name for delivery still “West End Glass Machine Service Manager”, or should the current system be renamed to “West End Glass Maintenance System” across wiki/project records?
2. Should the remaining frontend dependency audit findings be fixed now, or held until Joe confirms whether dependency upgrades affect his current workstream?
3. Should the public repository visibility be revisited before any pilot deployment, client demo, or broader sharing?

## Next recommended actions

### Hudson-owned

- Keep the `coding-projects` pilot-readiness card updated with this safe-hygiene closeout and leave only dependency-audit / pilot-deployment items open.
- Before any live WhatsApp/Meta/Claude test, get Steve's explicit approval for the exact live action and expected side effect.

### Steve-decision items

- Decide whether to tackle the remaining frontend dependency audit findings now or defer until Joe's current workstream is stable.
- Confirm whether the client-facing name should remain “Machine Service Manager” or move to “Maintenance System”.

## Steve's notes

- 2026-05-31: Steve said the West End Glass maintenance system has been transferred from the `Virgelsnake` organisation to the DTP organisation. He asked Hudson to run the briefing process on the Inbox folder and check that the local files are in sync with the repo. Steve also noted that the repo should be the most up to date because Joe/Joseph Ronnie has been working on the project.
