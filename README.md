# Project_Hubbb

Client-developer project marketplace for Pakistan. Clients post software projects, an AI suggests a fair
price band and timeline, verified developers bid and negotiate, the client pays into platform escrow, and
work is delivered and paid out in 3 stages (SRS 20%, working demo 40%, source code 40%). All messages are
moderated so contact details are not exchanged off-platform.

Full plan, rules and task prompts: [`docs/PROMPT_PACK.md`](docs/PROMPT_PACK.md).
Decisions (single source of truth): [`docs/decisions.md`](docs/decisions.md).
Current status and next task: [`docs/PROGRESS.md`](docs/PROGRESS.md).

## Repository layout

| Folder | What | Stack | Owner |
|---|---|---|---|
| `backend/` | Main API, business rules, payments, jobs | Java 21, Spring Boot 3, Maven, PostgreSQL 16, Flyway | Founder A |
| `ai-service/` | Estimate, moderation scoring, chatbot | Python 3.11, FastAPI | Founder B |
| `frontend/` | Website and dashboards | Next.js (TypeScript), Tailwind, Playwright | Founder B |
| `contracts/` | OpenAPI specs + example payloads | OpenAPI 3.1 | Both (PR reviewed by both) |
| `docs/` | Requirements, decisions, ADRs, testing, legal drafts | Markdown | Both (PR reviewed by both) |

## Prerequisites

Git, Docker Desktop, JDK 21, Node.js 22, Python 3.11, VS Code (or any editor).

## Running locally

```bash
cp .env.example .env          # then fill in local values (never commit .env)
docker compose up -d          # postgres, mailpit, minio
docker compose --profile scan up -d clamav   # optional virus scanner (needs ~1-2 GB RAM)
```

| Service | Port(s) | URL / use |
|---|---|---|
| PostgreSQL 16 | 5432 | `jdbc:postgresql://localhost:5432/marketplace` |
| Mailpit (fake email inbox for dev/test) | 1025 (SMTP), 8025 (web) | http://localhost:8025 - all OTP and notification emails land here |
| MinIO (S3-compatible file storage) | 9000 (API), 9001 (console) | http://localhost:9001 |
| ClamAV (profile `scan`) | 3310 | virus scanning |
| Backend (from task A1) | 8080 | http://localhost:8080/api/v1/ping |
| AI service (from task B1) | 8000 | http://localhost:8000/health |
| Frontend (from task F1) | 3000 | http://localhost:3000 |

How to run each app (`backend/`, `ai-service/`, `frontend/`) is documented in that folder's own README
once its first task is merged.

Image tags for Mailpit, MinIO and ClamAV are not pinned yet; pin them to exact versions before production (A10).

## Branching and workflow

- `main` is always working. Never commit directly to `main`.
- One branch per task, named exactly as in the pack: `chore/G0-repo-setup`, `feat/A1-foundation`, `feat/B1-ai-skeleton`, ...
- Flow: task -> branch -> implement + tests -> PR -> review by the other founder -> merge. See [`CONTRIBUTING.md`](CONTRIBUTING.md).
- Changes to `contracts/` or `docs/decisions.md` must be approved by both founders.
- Never commit secrets, `.env`, build output or `node_modules`.

## Ownership

| Path | Founder A | Founder B |
|---|---|---|
| `backend/` | owner | reviewer |
| `ai-service/` | reviewer | owner |
| `frontend/` | reviewer | owner |
| `contracts/`, `docs/`, root config | co-owner | co-owner |

GitHub usernames are set in [`.github/CODEOWNERS`](.github/CODEOWNERS).
