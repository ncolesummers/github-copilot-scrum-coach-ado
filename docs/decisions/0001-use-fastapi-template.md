# ADR-0001: Use full-stack-fastapi-template as Project Scaffold

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-02-14 |
| **Author** | OIT Application Development |

## Context

SprintPilot needs a production-ready application scaffold with FastAPI (Python), React, PostgreSQL, Docker Compose, authentication, and a CI/CD baseline. Building this from scratch would consume several weeks before any domain-specific work could begin. The team has Python and TypeScript fluency and is already running FastAPI services in production.

The [full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) is a well-maintained, officially supported scaffold from the FastAPI project that ships with:

- FastAPI + SQLModel + PostgreSQL backend
- React + TypeScript + Vite + Tailwind CSS + shadcn/ui frontend
- JWT authentication with email/password (extensible to OAuth/SSO)
- Docker Compose with Traefik reverse proxy and automatic HTTPS
- GitHub Actions CI/CD
- Playwright E2E testing baseline
- Auto-generated API client from OpenAPI spec

Options considered:

| Option | Assessment |
|---|---|
| full-stack-fastapi-template | Matches stack exactly; actively maintained; ships with all required infrastructure |
| Django + DRF | Heavier ORM; synchronous by default; WebSocket support requires additional setup; not team's primary stack |
| Custom FastAPI scaffold | Full control; significant upfront investment; reinvents solved problems |
| Next.js full-stack | Frontend-first; less natural fit for Python-heavy agent backend |

## Decision

Use [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) as the project foundation, forked and extended with SprintPilot-specific models, routes, and components.

SprintPilot extends rather than modifies template internals where possible, to preserve the ability to pull upstream improvements.

## Consequences

**Positive:**
- Weeks of scaffold work skipped; auth, DB, Docker Compose, CI/CD are ready on day one.
- shadcn/ui component library already present; chat components integrate naturally.
- Auto-generated TypeScript API client reduces frontend/backend integration friction.
- Template's patterns (Pydantic models, SQLModel, dependency injection) are already understood by the team.

**Negative / Risks:**
- Template upgrades may require manual merge work.
- Template's `Item` model and generic CRUD routes are irrelevant to SprintPilot — they'll be left in place or removed cleanly to avoid confusion.
- Template uses local JWT auth by default; Azure AD SSO integration is a Phase 2+ extension (see §7.1 of PRD).
