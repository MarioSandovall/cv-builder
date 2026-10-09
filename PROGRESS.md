# CV Builder — Progress Tracker

**Last updated:** 2026-10-09
**Current phase:** P0 — Foundations
**Current session:** S0.3 — PostgreSQL in Docker
**Last session completed:** S0.2 — .NET solution and project scaffolding

> Full plan: `docs/superpowers/plans/2026-10-08-cv-builder-mvp-implementation-plan.md`
> Spec: `docs/superpowers/specs/2026-10-08-cv-builder-mvp-architecture-design.md`

## Session Checklist

### P0 — Foundations
- [x] S0.1 — Repository and Git initialization
- [x] S0.2 — .NET solution and project scaffolding
- [ ] S0.3 — PostgreSQL in Docker
- [ ] S0.4 — EF Core wiring and empty initial migration
- [ ] S0.5 — OpenAPI, Scalar, and health endpoint
- [ ] S0.6 — Folder scaffolding and CLAUDE.md refresh

### P1 — Resume CRUD
- [ ] S1.1 — Resume domain entity and migration
- [ ] S1.2 — CreateResume slice
- [ ] S1.3 — Validation filter and ProblemDetails errors
- [ ] S1.4 — Global exception handling
- [ ] S1.5 — GetResume and ListResumes slices
- [ ] S1.6 — UpdateResume and DeleteResume slices
- [ ] S1.7 — Testcontainers infrastructure
- [ ] S1.8 — Integration tests for the full Resume CRUD
- [ ] S1.9 — Frontend foundation and resume listing
- [ ] S1.10 — Frontend CRUD flows
- [ ] S1.11 — Introductory GitHub Actions CI (backend)

### P2 — CV Sections
- [ ] S2.1 — WorkExperience: domain, migration, backend slices
- [ ] S2.2 — WorkExperience: integration tests and frontend UI
- [ ] S2.3 — Education: backend slice
- [ ] S2.4 — Education: integration tests and frontend UI
- [ ] S2.5 — Skills: backend + frontend
- [ ] S2.6 — Languages: backend + frontend
- [ ] S2.7 — Certifications: backend + frontend
- [ ] S2.8 — Projects: backend + frontend
- [ ] S2.9 — Personal information fields on Resume
- [ ] S2.10 — UX polish for section panels

### P3 — PDF Generation
- [ ] S3.1 — IPdfRenderer and QuestPDF skeleton
- [ ] S3.2 — Baseline template rendering
- [ ] S3.3 — Export endpoint and frontend download

### P4 — Templates
- [ ] S4.1 — Template strategy refactor
- [ ] S4.2 — Second template
- [ ] S4.3 — Templates endpoint and frontend selector

### P5 — Duplicate
- [ ] S5.1 — Resume.Duplicate domain method and endpoint
- [ ] S5.2 — Duplicate button in frontend

### P6 — Docker Compose end-to-end
- [ ] S6.1 — Backend Dockerfile
- [ ] S6.2 — Frontend Dockerfile and nginx
- [ ] S6.3 — Full compose.yaml

### P7 — Frontend CI and documentation
- [ ] S7.1 — Frontend CI workflow
- [ ] S7.2 — README polish and deployment checklist

## Notes

- Architecture decisions log: see spec, section 15 (change history — add entries here when architecture changes).
- Deferred micro-decisions:
  - Assertion library (Shouldly vs FluentAssertions) — defaults to Shouldly in S1.7 unless overridden.
- Commit style: Conventional Commits. No secrets, no build artifacts, no empty/artificial commits.
