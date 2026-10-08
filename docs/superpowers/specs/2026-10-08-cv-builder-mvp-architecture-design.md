# CV Builder — MVP Architecture Design

**Date:** 2026-10-08
**Status:** Approved by user (Option C: pragmatic approach with strong learning focus).
**Authors:** Claude (Senior Software Architect) + user.

---

## 1. Context

CV Builder is a personal web application for creating, editing, managing, and exporting resumes as PDF. It replaces equivalent paid services.

The project is simultaneously:

- **A real application** with real users (the author first).
- **A learning vehicle** for practicing .NET 10, EF Core, PostgreSQL, Docker, testing, and continuous integration.

### Context constraints

- **Single developer.**
- **Sessions of ~45–60 minutes.** Each implementation task must fit in one session.
- **Monorepo** with `backend/` and `frontend/`.
- Repository is **not yet initialized as git** (first task in the plan).
- Backend: `CvBuilder.slnx` solution file exists but is empty.
- Frontend: untouched Vite + React 19 + TypeScript scaffold.

### What we want to practice deliberately

- ASP.NET Core 10 + Minimal APIs + modern C# features.
- Dependency Injection and SOLID principles applied *when they add value*.
- EF Core migrations, Fluent API configurations, projection queries.
- PostgreSQL in Docker.
- Docker: multi-stage Dockerfiles, networking, volumes, Compose, environment variables.
- Unit testing with xUnit; integration testing with Testcontainers.
- REST API design, validation, global error handling, OpenAPI.
- CI/CD with GitHub Actions.

### What we will NOT introduce "to learn"

If a pattern does not add value to the domain, we will explore it in a different project. The following will **not** be introduced into CV Builder purely for learning: MediatR, CQRS, multi-project Clean Architecture, Generic Repository, explicit Unit of Work, AutoMapper, Specification pattern, Domain events, Event sourcing.

---

## 2. Approved architectural decisions

| Decision | Choice |
|---|---|
| Backend framework | ASP.NET Core 10 Web API |
| Backend architecture | Lightweight Vertical Slice, **single Web API project** |
| API style | Minimal APIs organized by feature folder |
| Language | C# 14 (.NET 10 features) |
| ORM | Entity Framework Core (latest stable compatible with .NET 10) |
| Database | PostgreSQL 17 |
| Data access pattern | `DbContext` injected directly — no repositories |
| Validation | DataAnnotations + global endpoint filter; FluentValidation reactively if a complex cross-field rule appears |
| Error handling | `AddProblemDetails()` + `UseExceptionHandler()` + custom `IExceptionHandler` |
| Dependency Injection | Native .NET DI; grouped registration per module via extension methods |
| Logging | Native `ILogger<T>`; Serilog only if structured sinks are needed |
| OpenAPI | Native `Microsoft.AspNetCore.OpenApi` + Scalar as UI |
| Configuration | `appsettings.*.json` + user-secrets + env vars; strongly-typed `IOptions<T>` classes |
| PDF | `IPdfRenderer` abstraction, `QuestPdfRenderer` implementation |
| Testing | xUnit; integration with `WebApplicationFactory<Program>` + Testcontainers for PostgreSQL |
| Frontend | React 19 + Vite 8 + TypeScript 6 (keep current scaffold versions for now) |
| Dev infrastructure | Docker Compose with `postgres`, `api`, `frontend` |
| CI/CD | GitHub Actions |

### Explicit non-choices for MVP

- Authentication / authorization (added post-MVP).
- Multi-tenancy.
- AI features.
- Public deployment (until auth exists).
- Distributed caching.
- Messaging / background jobs (QuestPDF renders in-process; acceptable for MVP).

---

## 3. Repository structure

```
cv-builder/
├── CLAUDE.md
├── README.md
├── .gitignore
├── .editorconfig
├── .env.example
├── compose.yaml
├── docs/
│   └── superpowers/
│       ├── specs/
│       │   └── 2026-10-08-cv-builder-mvp-architecture-design.md
│       └── plans/            # implementation plans
│
├── .github/
│   └── workflows/            # CI
│
├── backend/
│   ├── CvBuilder.slnx
│   ├── src/
│   │   └── CvBuilder.Api/
│   │       ├── CvBuilder.Api.csproj
│   │       ├── Program.cs
│   │       ├── appsettings.json
│   │       ├── appsettings.Development.json
│   │       ├── Dockerfile
│   │       ├── Features/
│   │       │   ├── Resumes/
│   │       │   │   ├── CreateResume/
│   │       │   │   ├── GetResume/
│   │       │   │   ├── ListResumes/
│   │       │   │   ├── UpdateResume/
│   │       │   │   ├── DeleteResume/
│   │       │   │   ├── DuplicateResume/
│   │       │   │   └── ExportResumePdf/
│   │       │   ├── WorkExperiences/
│   │       │   ├── Education/
│   │       │   ├── Skills/
│   │       │   ├── Languages/
│   │       │   ├── Certifications/
│   │       │   ├── Projects/
│   │       │   └── Templates/
│   │       ├── Domain/
│   │       │   └── <entities + behavior>.cs
│   │       ├── Persistence/
│   │       │   ├── AppDbContext.cs
│   │       │   ├── Configurations/
│   │       │   └── Migrations/
│   │       ├── Pdf/
│   │       │   ├── IPdfRenderer.cs
│   │       │   └── QuestPdfRenderer.cs
│   │       └── Common/
│   │           ├── Errors/
│   │           ├── Validation/
│   │           └── Extensions/
│   └── tests/
│       └── CvBuilder.Api.Tests/
│           ├── Features/
│           ├── Integration/
│           └── Support/
│
└── frontend/
    ├── Dockerfile
    ├── nginx.conf
    ├── package.json
    ├── vite.config.ts
    └── src/
```

---

## 4. Anatomy of a slice

Folder `Features/<Area>/<Operation>/`. Contents:

- `*Endpoint.cs` — defines the route and handler (static function or inline).
- `*Request.cs` — body/query record (with DataAnnotations).
- `*Response.cs` — record returned by the endpoint.
- `*Validator.cs` *(optional)* — only if validation exceeds DataAnnotations.
- `*Handler.cs` *(optional)* — only when logic does not fit comfortably in the endpoint.

**Rules:**

- No cross-slice references. Shared code lives in `Common/`, `Domain/`, `Persistence/`, or `Pdf/`.
- A functional change should touch **one** slice folder in 95% of cases.
- If logic is duplicated across 3+ slices, extract to `Common/` or a domain method.

---

## 5. Data layer

- `AppDbContext` as the single access point.
- `DbSet<T>` for each root entity.
- Configuration via `IEntityTypeConfiguration<T>` per entity, applied with `ApplyConfigurationsFromAssembly()`.
- `AsNoTracking()` as default for read queries.
- Projection with `.Select(…)` directly to response types for list endpoints.
- Migrations from day 1; **no** `EnsureCreated`.
- Implicit transactions via `SaveChangesAsync` (one unit of work per request).
- ID convention: `Guid.CreateVersion7()` (.NET 10) for natural ordering.

---

## 6. Domain model

Entities are POCOs in `Domain/` with **behavior where it makes sense** (not anemic):

- Private constructors + static factory methods (`Resume.Create(title)`).
- Private setters; mutations via methods (`resume.Rename(newTitle)`, `resume.Duplicate()`).
- Invariant validation inside domain methods (throw specific exception).

No heavy tactical DDD (value objects everywhere, rich aggregates, domain events) is introduced until the domain demands it.

---

## 7. API design

### REST conventions

```
GET    /api/resumes
POST   /api/resumes
GET    /api/resumes/{id}
PUT    /api/resumes/{id}
DELETE /api/resumes/{id}
POST   /api/resumes/{id}/duplicate
GET    /api/resumes/{id}/pdf

GET    /api/resumes/{id}/work-experiences
POST   /api/resumes/{id}/work-experiences
PUT    /api/resumes/{id}/work-experiences/{weId}
DELETE /api/resumes/{id}/work-experiences/{weId}

GET    /api/templates
```

Status codes:

- `201 Created` with `Location` header on resource creation.
- `204 No Content` on successful PUT/DELETE with no body.
- `404 Not Found` for unknown IDs.
- `400 Bad Request` for shape/format validation; `422 Unprocessable Entity` for business rule violations.
- `500 Internal Server Error` only for unhandled exceptions (via `IExceptionHandler`).

### Validation

- MVP: DataAnnotations on `Request` records.
- Global `ValidationFilter` endpoint filter that runs `Validator.TryValidateObject` and returns `ValidationProblem()` on failure.
- Complex cross-field rules → introduce FluentValidation **only in that feature**.

### Error handling

Custom `IExceptionHandler` maps:

- `NotFoundException` → 404
- `DomainValidationException` → 422
- rest → 500 with generic `ProblemDetails`, logged with stack trace.

### OpenAPI

- `builder.Services.AddOpenApi()`.
- `app.MapOpenApi()` + Scalar at `/scalar`.
- Document each endpoint with `.WithName()`, `.WithTags()`, `.Produces<T>()`, `.ProducesValidationProblem()`.

### Service registration

Extension methods per module:

```csharp
builder.Services.AddPersistence(builder.Configuration);
builder.Services.AddPdfGeneration();
builder.Services.AddApiValidation();
```

And endpoint mapping:

```csharp
app.MapResumeEndpoints();
app.MapTemplateEndpoints();
```

---

## 8. Testing

- **Unit tests**: domain (`Resume.Duplicate()` produces a new entity with equivalent content, etc.) and validators. xUnit + `Shouldly` (or FluentAssertions — decided during setup).
- **Integration tests**: `WebApplicationFactory<Program>` + Testcontainers for PostgreSQL. One container per fixture; migrations applied on startup. End-to-end tests per slice.
- **No** EF InMemory provider. Real Postgres from day 1 — it is the learning target and avoids divergence bugs.
- Coverage: pragmatic. Happy paths and expected error paths covered. No pursuit of 100%.

---

## 9. PDF generation

- `IPdfRenderer` interface in `Pdf/`:
  ```csharp
  public interface IPdfRenderer
  {
      Task<byte[]> RenderAsync(Resume resume, string templateKey, CancellationToken ct);
  }
  ```
- `QuestPdfRenderer` implementation delegates to *template strategies*.
- Template resolver (`IPdfTemplate` + keyed registration) introduced when the second template appears.
- QuestPDF Community License: document non-commercial usage in README.

---

## 10. Reference flow — `CreateResume`

**Request:** `POST /api/resumes` with body `{ "title": "Developer CV" }`.

**Files involved:**

- `Features/Resumes/CreateResume/CreateResumeRequest.cs`
- `Features/Resumes/CreateResume/CreateResumeResponse.cs`
- `Features/Resumes/CreateResume/CreateResumeEndpoint.cs`
- `Domain/Resume.cs`
- `Persistence/AppDbContext.cs`

**Flow:**

1. Model binding → `CreateResumeRequest`.
2. `ValidationFilter` runs DataAnnotations → 400 on failure.
3. Endpoint handler calls `Resume.Create(request.Title)`.
4. `db.Resumes.Add(resume)` + `await db.SaveChangesAsync(ct)`.
5. `TypedResults.Created($"/api/resumes/{resume.Id}", new CreateResumeResponse(resume.Id))`.

No service, no repository, no mediator. If the flow later grows logic (publish an event, invalidate a cache), it is added *then*, in that slice.

---

## 11. Docker strategy

### Progressive introduction (pedagogical motivation)

| Phase | What is dockerized | Docker concept learned |
|---|---|---|
| P0 | Postgres only | Official image, named volume, port mapping, env vars |
| P1 | Postgres + Testcontainers | Programmatic ephemeral containers for integration tests |
| P6 | Postgres + API | Multi-stage backend Dockerfile, service networking, healthcheck + `depends_on: condition: service_healthy` |
| P6 | + Frontend | Multi-stage frontend Dockerfile (build → nginx), proxy to API |
| P6 | Full Compose | `profiles` dev/prod, override files, `.env` |

### Stable decisions

- Single `compose.yaml` at the root with *profiles* `dev` and `prod`.
- Named volume `cvbuilder-pgdata` for Postgres persistence.
- `.env` outside git; `.env.example` versioned.
- `ConnectionStrings__Default` via env var in containers.
- During active development: Postgres in Docker, backend/frontend with `dotnet watch` and `npm run dev` outside Docker. Full Compose used for end-to-end smoke tests and production.

---

## 12. CI/CD (GitHub Actions)

### Minimal pipeline (introduced late, when there is something real to validate)

- Workflow at `.github/workflows/ci.yml`.
- Trigger: `push` and `pull_request` to `main`.
- Jobs:
  - **Backend**: `dotnet restore`, `dotnet build`, `dotnet test` (requires Docker on the runner for Testcontainers; use `ubuntu-latest`).
  - **Frontend**: `npm ci`, `npm run lint`, `npm run build`.
- Optional later: Docker image builds, Trivy scan, publish to GHCR.

---

## 13. High-level phases

Titles only. The session-by-session breakdown is produced in the implementation plan (via `writing-plans` skill).

- **P0 — Foundations**: git init, `.gitignore`, `.editorconfig`, README, .NET projects + solution, folder structure, Postgres in Docker, initial EF Core, health check, OpenAPI.
- **P1 — Resume CRUD (full vertical)**: Resume entity, migration, full CRUD, validation, error handling, integration tests, frontend consumes the API.
- **P2 — CV sections**: one slice per section (WorkExperience, Education, Skills, Languages, Certifications, Projects).
- **P3 — PDF generation**: `IPdfRenderer`, `QuestPdfRenderer`, one baseline template, export endpoint.
- **P4 — Templates**: multiple templates, selector in frontend.
- **P5 — Duplicate**: domain logic + endpoint + UI.
- **P6 — End-to-end Docker Compose**: backend and frontend Dockerfiles, full compose.
- **P7 — CI/CD + deployment prep**: GitHub Actions, documentation, environment variables, deployment checklist.
- **Post-MVP**: authentication, AI assist, advanced template customization, cloud deployment.

---

## 14. Risks and mitigations

| Risk | Mitigation |
|---|---|
| QuestPDF commercial license if published | Document restriction in README; evaluate alternative (iText, PdfSharpCore) if monetized |
| No auth → API is public if exposed | Keep local until authentication is added post-MVP; when exposed, temporary reverse proxy with basic auth |
| React 19 / Vite 8 / TS 6 are very recent → unstable tooling | Downgrade to stable if blocking; log in decision journal |
| Testcontainers requires Docker in CI | Use `ubuntu-latest`, which already includes Docker; no exotic matrix |
| Duplication across slices | Refactor reactively to `Common/` or domain methods |
| QuestPDF renders synchronously and is heavy | Acceptable in MVP; move to background job (Hangfire / Quartz) if it grows |
| Migration discipline | One migration per commit; never edit already-merged migrations |

---

## 15. Spec evolution policy

- This document reflects architectural decisions, **not implementation details**.
- A change of architectural decision (e.g., "we are moving to repositories") requires a new entry in a "Change history" section and an explicit commit.
- The implementation plan (sessions) is a separate artifact: it lives in `docs/superpowers/plans/`.
- Routine code updates do not require touching this spec.

---

## 16. Next step

Transition to the `writing-plans` skill to produce the session-by-session implementation plan, honoring:

- Sessions ≤ 60 minutes.
- Docker introduced from P0.
- 10-step per-session workflow defined by the user.
- Explicit approval before each session; no auto-chaining.
