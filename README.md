# CV Builder

A personal full-stack web application for creating, editing, managing, and exporting professional resumes as PDF documents.

Built as a learning project to practice modern .NET development, software architecture, Docker, testing, and CI/CD — while producing a real tool for everyday use.

## Tech Stack

- **Backend:** ASP.NET Core 10, C# 14, Minimal APIs, Entity Framework Core 10
- **Database:** PostgreSQL 17
- **Frontend:** React 19, Vite 8, TypeScript 6
- **PDF generation:** QuestPDF (Community License)
- **Infrastructure:** Docker + Docker Compose
- **Testing:** xUnit + Testcontainers for PostgreSQL + `WebApplicationFactory<Program>`
- **CI/CD:** GitHub Actions

## Architecture

Lightweight **Vertical Slice** in a single Web API project. Minimal APIs organized by feature folder. EF Core `DbContext` injected directly (no repositories, no Unit of Work class). QuestPDF behind an `IPdfRenderer` interface. Deliberate choice to avoid premature abstractions (no MediatR, no CQRS, no Generic Repository).

Full architectural decisions: [`docs/superpowers/specs/2026-10-08-cv-builder-mvp-architecture-design.md`](docs/superpowers/specs/2026-10-08-cv-builder-mvp-architecture-design.md).

## Status

**MVP in progress.** The project is built incrementally in ~45–60 minute sessions. Current progress and the full roadmap are tracked in [`PROGRESS.md`](PROGRESS.md). The session-by-session implementation plan lives in [`docs/superpowers/plans/`](docs/superpowers/plans/).

No authentication yet — the app is intended for local use during the MVP phase.

## Repository Layout

```
cv-builder/
├── backend/          # .NET 10 Web API (currently empty solution; projects added in P0)
├── frontend/         # React + Vite SPA
├── docs/             # specs, plans, decision log
├── CLAUDE.md         # guidance for Claude Code sessions
└── PROGRESS.md       # lightweight progress tracker
```

## Development

Setup and run instructions will be added as the project progresses (polished in session S7.2). For now, see the plan file for session-level details.

### Database (local)

The project runs PostgreSQL 17 in Docker. Credentials for local development live in a `.env` file (git-ignored).

```bash
cp .env.example .env       # one-time: create your local env file
docker compose up -d postgres
docker compose ps          # expect cvbuilder-postgres with status "healthy"
docker compose down        # stop (data persists in the named volume)
```

The named volume `cvbuilder-pgdata` persists data across container restarts and recreations. To wipe the database entirely, run `docker compose down -v`.

Postgres listens on `127.0.0.1:${POSTGRES_PORT}` (default `5432`) — the port is bound to loopback only, so it is not reachable from other machines on the network.

The credentials in `.env.example` are intended for local development only. Do not reuse them for production, staging, or any shared environment.
