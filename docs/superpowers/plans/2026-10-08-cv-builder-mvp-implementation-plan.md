# CV Builder MVP — Implementation Plan

> **For agentic workers:** This plan is executed under the user's 11-step per-session workflow (review → identify task → explain → mini-plan → **wait for approval** → implement → test → summarize → update PROGRESS.md → **approve commit message → commit + push** → recommend next). Full details in Appendix C. Do NOT auto-chain sessions. One session per interaction. Checkbox (`- [ ]`) items track completion.

**Goal:** Build a full-stack CV Builder MVP (ASP.NET Core 10 + PostgreSQL + React) that lets a single user create, edit, duplicate, and export resumes as PDF, delivered incrementally over ~45–60 minute sessions.

**Scope:** 40 sessions across 8 phases (P0–P7). Count breakdown: P0 = 6, P1 = 11, P2 = 10, P3 = 3, P4 = 3, P5 = 2, P6 = 3, P7 = 2. Sessions may split into sub-sessions (`Sx.y a / b`) if the mini-plan reveals the work does not fit in 60 min.

**Architecture:** Lightweight Vertical Slice in a single Web API project. Minimal APIs. EF Core `DbContext` injected directly (no repositories). QuestPDF behind an `IPdfRenderer` interface. React 19 + Vite SPA. PostgreSQL in Docker from day 1. Testcontainers for integration tests. GitHub Actions CI.

**Tech Stack:** .NET 10, C# 14, ASP.NET Core 10, EF Core 10 (Npgsql provider), PostgreSQL 17, QuestPDF (Community License), xUnit, Testcontainers for PostgreSQL, `WebApplicationFactory<Program>`, Microsoft.AspNetCore.OpenApi, Scalar.AspNetCore, React 19, Vite 8, TypeScript 6, Node LTS, Docker + Docker Compose, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-08-cv-builder-mvp-architecture-design.md`

---

## Global Constraints

- **Backend framework:** ASP.NET Core 10, C# 14.
- **Database:** PostgreSQL 17.
- **ORM:** Entity Framework Core 10 with `Npgsql.EntityFrameworkCore.PostgreSQL`.
- **API style:** Minimal APIs only. No controllers.
- **Data access:** `AppDbContext` injected directly. No repositories, no Unit of Work class.
- **ID type:** `Guid` generated with `Guid.CreateVersion7()`.
- **Validation (MVP):** DataAnnotations + global `ValidationFilter` endpoint filter.
- **Error handling:** `AddProblemDetails()` + `UseExceptionHandler()` + custom `IExceptionHandler`.
- **OpenAPI:** `Microsoft.AspNetCore.OpenApi` + Scalar UI at `/scalar`.
- **Testing:** xUnit; integration via `WebApplicationFactory<Program>` + Testcontainers for PostgreSQL. No EF InMemory provider.
- **Frontend versions:** React 19, Vite 8, TypeScript 6 (current scaffold).
- **Docker volume name:** `cvbuilder-pgdata`.
- **Environment files:** `.env` is git-ignored; `.env.example` is committed.
- **Connection string env var:** `ConnectionStrings__Default` (used in Docker and production).
- **Local credentials:** stored in `.NET User Secrets` (per project `UserSecretsId`). Never commit DB passwords, API keys, or connection strings with credentials to `appsettings*.json` or any other tracked file. `appsettings.Development.json` may contain non-sensitive overrides only.
- **Session length:** ≤ 60 minutes. If, after the mini-plan in step 4 of the session workflow, a session clearly cannot fit (unknowns, discovered complexity, surprise refactor), **split it** into sub-sessions (`S1.7a`, `S1.7b`, …) rather than overrun. Overrunning sessions is a plan smell, not a badge.
- **Commit style:** Conventional Commits (`feat:`, `chore:`, `test:`, `docs:`, `refactor:`, `fix:`, `ci:`).
- **Git discipline:** small, meaningful commits that represent real progress. No empty or artificial commits. Never commit secrets, `.env` files, generated build artifacts (`bin/`, `obj/`, `dist/`, `node_modules/`, `TestResults/`). Every commit requires explicit user approval; after approval, commit **and push** to the GitHub remote.
- **No code gets written until a session is explicitly approved.**

---

## File Structure (what the plan produces, cumulatively)

```
cv-builder/
├── .editorconfig                               # P0
├── .env.example                                # P0
├── .gitignore                                  # P0
├── CLAUDE.md                                   # pre-existing, updated through phases
├── PROGRESS.md                                 # P0 (lightweight tracker)
├── README.md                                   # P0, polished in P7
├── compose.yaml                                # P0 (Postgres only), expanded in P6
├── docs/
│   └── superpowers/
│       ├── specs/2026-10-08-cv-builder-mvp-architecture-design.md
│       └── plans/2026-10-08-cv-builder-mvp-implementation-plan.md  (this file)
├── .github/workflows/
│   ├── backend-ci.yml                          # P1 (S1.11)
│   └── frontend-ci.yml                         # P7
├── backend/
│   ├── CvBuilder.slnx
│   ├── src/CvBuilder.Api/
│   │   ├── CvBuilder.Api.csproj                # P0
│   │   ├── Program.cs                          # P0, expanded through phases
│   │   ├── appsettings.json                    # P0
│   │   ├── appsettings.Development.json        # P0
│   │   ├── Dockerfile                          # P6
│   │   ├── Features/
│   │   │   ├── Resumes/{CreateResume,GetResume,ListResumes,UpdateResume,DeleteResume,DuplicateResume,ExportResumePdf}/   # P1, P3, P5
│   │   │   ├── WorkExperiences/                # P2
│   │   │   ├── Education/                      # P2
│   │   │   ├── Skills/                         # P2
│   │   │   ├── Languages/                      # P2
│   │   │   ├── Certifications/                 # P2
│   │   │   ├── Projects/                       # P2
│   │   │   └── Templates/                      # P4
│   │   ├── Domain/
│   │   │   ├── Resume.cs                       # P1
│   │   │   ├── WorkExperience.cs               # P2
│   │   │   └── ...                             # P2
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs                 # P0
│   │   │   ├── Configurations/                 # P1, P2
│   │   │   └── Migrations/                     # P0 (empty), P1, P2
│   │   ├── Pdf/
│   │   │   ├── IPdfRenderer.cs                 # P3
│   │   │   ├── QuestPdfRenderer.cs             # P3
│   │   │   ├── IPdfTemplate.cs                 # P4
│   │   │   └── Templates/                      # P3, P4
│   │   ├── Common/
│   │   │   ├── Errors/                         # P1
│   │   │   ├── Validation/                     # P1
│   │   │   └── Extensions/                     # P0+
│   │   └── Properties/launchSettings.json      # P0
│   └── tests/CvBuilder.Api.Tests/
│       ├── CvBuilder.Api.Tests.csproj          # P0
│       ├── Features/                           # P1+
│       ├── Integration/
│       │   ├── CvBuilderApiFactory.cs          # P1
│       │   └── ...Tests.cs                     # P1+
│       └── Support/
│           └── PostgresFixture.cs              # P1
└── frontend/
    ├── Dockerfile                              # P6
    ├── nginx.conf                              # P6
    ├── package.json                            # pre-existing, dependencies added through phases
    ├── vite.config.ts                          # pre-existing, proxy added in P1
    └── src/
        ├── main.tsx                            # existing, repurposed in P1
        ├── App.tsx                             # existing, replaced in P1
        ├── api/                                # P1
        ├── features/
        │   ├── resumes/                        # P1, P5
        │   ├── sections/                       # P2
        │   └── templates/                      # P4
        └── router.tsx                          # P1
```

---

## Session Workflow (per the user's requirement)

Each session runs through this 11-step workflow (full detail in Appendix C):

1. Review current project state (`PROGRESS.md`, last commit, open TODOs).
2. Identify the next incomplete task from this plan.
3. Explain what we will implement and *why*.
4. Present a short implementation mini-plan (expand this session into TDD micro-steps if it is a code session). Split into sub-sessions if 60 min is not realistic.
5. **Wait for explicit user approval.**
6. Implement only the approved task.
7. Run the relevant build and tests.
8. Summarize what was completed.
9. Update `PROGRESS.md` (ships in the same commit as the session work).
10. Propose the Conventional Commit message, wait for approval, then `git commit && git push`.
11. Recommend the next session's objective. Do not start it.

Sessions are not auto-chained. Each starts fresh.

---

# PHASE P0 — Foundations

**Goal:** From an empty repo to a running Web API that responds to `/health` and `/scalar`, with a Postgres container up and migrations applied.

**Phase outcome:** `curl http://localhost:5080/health` returns `Healthy`; Scalar UI loads at `/scalar`; `docker compose ps` shows a healthy `postgres` container; `dotnet test` runs the (empty) test project successfully.

---

### Session S0.1 — Repository and Git initialization

- **Objective:** Initialize local git, set up ignore rules, create `README.md` skeleton, seed `PROGRESS.md`, connect to the existing GitHub remote (`https://github.com/MarioSandovall/cv-builder`), and push the initial commit.
- **Estimated duration:** 45 min
- **Prerequisites:** None locally. The remote repo at `https://github.com/MarioSandovall/cv-builder` already exists. If the remote has any commits (e.g., an auto-generated README), decide inside the session whether to merge (`git pull --rebase origin main --allow-unrelated-histories`) or ask the user to delete the remote content and recreate it empty. Do NOT force-push without explicit approval.

**Tasks:**

- [ ] Run `git init` in the repo root.
- [ ] Run `git branch -M main` to ensure the primary branch is `main`.
- [ ] Create `.gitignore` at root, covering: `bin/`, `obj/`, `*.user`, `.vs/`, `.vscode/`, `.idea/`, `node_modules/`, `dist/`, `build/`, `.env`, `.env.*.local`, `TestResults/`, `*.pfx`, `*.key`, `coverage/`, `.DS_Store`, `Thumbs.db`.
- [ ] Create `.editorconfig` at root with: UTF-8, LF for `*.sh`/`*.yml`, CRLF for `*.cs`, 4-space indent for C#, 2-space for JSON/YAML/TS/TSX, final newline, trim trailing whitespace.
- [ ] Create `README.md` with: project title, one-paragraph description, tech stack list, link to spec and plan, "status: in development" note.
- [ ] Create `PROGRESS.md` at root (template provided in Appendix A below).
- [ ] Stage and commit the spec, plan docs, `CLAUDE.md`, and setup files together.
- [ ] Add the existing GitHub remote: `git remote add origin https://github.com/MarioSandovall/cv-builder.git`.
- [ ] Inspect remote state: `git ls-remote origin`. If it reports no refs, go straight to push. If it reports an existing `main` (e.g., default README), run `git pull --rebase origin main --allow-unrelated-histories`, resolve any conflicts, and only then push.
- [ ] Push: `git push -u origin main`.
- [ ] Verify: `git remote -v`, `git branch -vv` (should show `main` tracking `origin/main`).

**Files touched:**
- Create: `.gitignore`, `.editorconfig`, `README.md`, `PROGRESS.md`.
- Already present (will be staged in this commit): `CLAUDE.md`, `docs/superpowers/specs/2026-10-08-cv-builder-mvp-architecture-design.md`, `docs/superpowers/plans/2026-10-08-cv-builder-mvp-implementation-plan.md`.

**Acceptance criteria:**
- [ ] `git log --oneline` shows at least one commit on `main`.
- [ ] `git status` is clean.
- [ ] `git remote -v` points to `https://github.com/MarioSandovall/cv-builder`.
- [ ] `git branch -vv` shows `main` tracking `origin/main`.
- [ ] The commit is visible on GitHub.
- [ ] `.env` style file (if created later) is ignored — verify by `echo FOO=bar > .env && git status` (then remove the file).
- [ ] `PROGRESS.md` lists P0 as current phase and S0.2 as next session.

**Verification:**
```bash
git log --oneline
git remote -v
git branch -vv
git check-ignore -v .env
gh repo view MarioSandovall/cv-builder --web
```

**Technical concepts:** `.gitignore` precedence, `.editorconfig` for consistent whitespace, connecting a local repo to an existing remote, `-u` upstream flag on first push, `--allow-unrelated-histories` when the remote has an independent initial commit.

**Suggested commit message:**
```
chore: initialize repository with ignore rules, editorconfig, and planning docs
```

---

### Session S0.2 — .NET solution and project scaffolding

- **Objective:** Create `CvBuilder.Api` Web API project and `CvBuilder.Api.Tests` xUnit project, add both to the `CvBuilder.slnx` solution.
- **Estimated duration:** 45 min
- **Prerequisites:** S0.1 complete. .NET 10 SDK installed (`dotnet --version` ≥ `10.0.100`).

**Tasks:**

- [ ] From `backend/`, create API project: `dotnet new webapi -n CvBuilder.Api -o src/CvBuilder.Api --use-minimal-apis --no-https` (confirm flags against installed templates; adjust if `--use-minimal-apis` is default in .NET 10).
- [ ] Remove the auto-generated `WeatherForecast` endpoints from `Program.cs` — leave just the builder skeleton.
- [ ] Create xUnit test project: `dotnet new xunit -n CvBuilder.Api.Tests -o tests/CvBuilder.Api.Tests`.
- [ ] Add project reference: `dotnet add tests/CvBuilder.Api.Tests reference src/CvBuilder.Api`.
- [ ] Add both to the solution: `dotnet sln CvBuilder.slnx add src/CvBuilder.Api/CvBuilder.Api.csproj tests/CvBuilder.Api.Tests/CvBuilder.Api.Tests.csproj`.
- [ ] In `CvBuilder.Api.csproj`, set `<InvariantGlobalization>true</InvariantGlobalization>` and set `<Nullable>enable</Nullable>` (likely default) and `<ImplicitUsings>enable</ImplicitUsings>`.
- [ ] Add `<InternalsVisibleTo Include="CvBuilder.Api.Tests" />` to API project to allow tests to see internals.
- [ ] Expose `Program` as partial for `WebApplicationFactory<Program>` by adding `public partial class Program;` at the bottom of `Program.cs`.
- [ ] Run `dotnet build` from `backend/`.
- [ ] Run `dotnet test` from `backend/` (empty but should pass).

**Files touched:**
- Create: `backend/src/CvBuilder.Api/*` (project skeleton), `backend/tests/CvBuilder.Api.Tests/*` (project skeleton).
- Modify: `backend/CvBuilder.slnx`.

**Acceptance criteria:**
- [ ] `dotnet build` succeeds with 0 warnings.
- [ ] `dotnet test` runs 0 tests and reports success.
- [ ] `backend/CvBuilder.slnx` references both projects.
- [ ] `Program.cs` is a clean skeleton (no WeatherForecast).

**Verification:**
```bash
cd backend
dotnet build
dotnet test
dotnet sln CvBuilder.slnx list
```

**Technical concepts:** `slnx` XML solution format in .NET 10, `WebApplication` builder, `InternalsVisibleTo` for test access, `partial class Program` convention for `WebApplicationFactory<T>`.

**Suggested commit message:**
```
chore(backend): scaffold CvBuilder.Api and CvBuilder.Api.Tests projects
```

---

### Session S0.3 — PostgreSQL in Docker

- **Objective:** Introduce the first Docker service — a PostgreSQL 17 container — with persistent volume, env file, and health check.
- **Estimated duration:** 45 min
- **Prerequisites:** S0.2 complete. Docker Desktop running (`docker --version` works).

**Tasks:**

- [ ] Create `.env.example` at repo root with keys: `POSTGRES_USER=cvbuilder`, `POSTGRES_PASSWORD=cvbuilder_dev`, `POSTGRES_DB=cvbuilder`, `POSTGRES_PORT=5432`.
- [ ] Copy `.env.example` → `.env` (local, not committed).
- [ ] Create `compose.yaml` at repo root with a single `postgres` service:

```yaml
services:
  postgres:
    image: postgres:17-alpine
    container_name: cvbuilder-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      - cvbuilder-pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  cvbuilder-pgdata:
```

- [ ] Run `docker compose up -d postgres` and wait for healthy status.
- [ ] Verify connection with `docker exec -it cvbuilder-postgres psql -U cvbuilder -d cvbuilder -c "\conninfo"`.
- [ ] Document in README how to start/stop Postgres.

**Files touched:**
- Create: `compose.yaml`, `.env.example`, `.env` (local only).
- Modify: `README.md`.

**Acceptance criteria:**
- [ ] `docker compose ps` shows `cvbuilder-postgres` with status `healthy`.
- [ ] `psql` connection via `docker exec` succeeds.
- [ ] `docker volume ls | grep cvbuilder-pgdata` returns a line.
- [ ] `.env` is not tracked by git.

**Verification:**
```bash
docker compose up -d postgres
docker compose ps
docker exec -it cvbuilder-postgres psql -U cvbuilder -d cvbuilder -c "SELECT version();"
docker compose down          # stop
docker compose up -d postgres  # restart — data persists
```

**Technical concepts:** Docker Compose service definition, named volumes vs bind mounts, env-var interpolation, health checks, image tagging (`:17-alpine`), port publishing.

**Suggested commit message:**
```
chore(docker): add PostgreSQL compose service with persistent volume
```

---

### Session S0.4 — EF Core wiring and empty initial migration

- **Objective:** Add EF Core 10 and Npgsql provider, create `AppDbContext`, wire DI, create an empty initial migration, apply it against the Postgres container.
- **Estimated duration:** 60 min
- **Prerequisites:** S0.3 complete. Postgres container healthy.

**Tasks:**

- [ ] Install `dotnet ef` tool globally if not present: `dotnet tool install --global dotnet-ef --version 10.*`.
- [ ] From `src/CvBuilder.Api`, add packages:
  - `dotnet add package Microsoft.EntityFrameworkCore.Design`
  - `dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL`
- [ ] Create `Persistence/AppDbContext.cs`:

```csharp
namespace CvBuilder.Api.Persistence;

using Microsoft.EntityFrameworkCore;

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}
```

- [ ] Create `Common/Extensions/PersistenceServiceCollectionExtensions.cs`:

```csharp
namespace CvBuilder.Api.Common.Extensions;

using CvBuilder.Api.Persistence;
using Microsoft.EntityFrameworkCore;

public static class PersistenceServiceCollectionExtensions
{
    public static IServiceCollection AddPersistence(this IServiceCollection services, IConfiguration configuration)
    {
        var connectionString = configuration.GetConnectionString("Default")
            ?? throw new InvalidOperationException("ConnectionStrings:Default is not configured.");

        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(connectionString, npg => npg.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)));

        return services;
    }
}
```

- [ ] In `Program.cs` add `builder.Services.AddPersistence(builder.Configuration);`.
- [ ] Enable User Secrets for the API project: from `src/CvBuilder.Api/`, run `dotnet user-secrets init`. This adds `<UserSecretsId>` to the `.csproj`.
- [ ] Store the development connection string in User Secrets (NOT in `appsettings.Development.json`):
  ```bash
  dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;Port=5432;Database=cvbuilder;Username=cvbuilder;Password=<same value from .env>"
  ```
- [ ] Leave `appsettings.Development.json` without a connection string; if you want a placeholder entry for discoverability, use an empty string or a comment pointing to User Secrets. **Do not commit real credentials to any `appsettings*.json`.**
- [ ] For running `dotnet ef` commands (which also need the connection string), User Secrets are picked up automatically because the API project is the startup project.
- [ ] Create initial empty migration: `dotnet ef migrations add InitialCreate --project src/CvBuilder.Api --startup-project src/CvBuilder.Api --output-dir Persistence/Migrations`.
- [ ] Apply migration: `dotnet ef database update --project src/CvBuilder.Api --startup-project src/CvBuilder.Api`.
- [ ] Verify in psql: `\dt` should show the `__EFMigrationsHistory` table.

**Files touched:**
- Create: `Persistence/AppDbContext.cs`, `Common/Extensions/PersistenceServiceCollectionExtensions.cs`, `Persistence/Migrations/*InitialCreate*.cs`.
- Modify: `Program.cs`, `CvBuilder.Api.csproj` (packages + `UserSecretsId`).
- NOT modified for credentials: `appsettings.Development.json` stays free of DB passwords. User Secrets store (`%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` on Windows) holds the local connection string and is outside the repo.

**Acceptance criteria:**
- [ ] `dotnet build` succeeds.
- [ ] `dotnet ef database update` reports success and `__EFMigrationsHistory` row exists.
- [ ] API starts (`dotnet run --project src/CvBuilder.Api`) without exceptions.

**Verification:**
```bash
dotnet ef migrations list --project src/CvBuilder.Api
docker exec -it cvbuilder-postgres psql -U cvbuilder -d cvbuilder -c "\dt"
```

**Technical concepts:** EF Core design-time vs runtime, `IDesignTimeDbContextFactory` (not needed when startup project provides config), migrations history table, Npgsql provider, DI extension method pattern, `.NET User Secrets` for keeping local credentials out of source control.

**Suggested commit message:**
```
feat(backend): wire EF Core with Npgsql and create empty initial migration
```

---

### Session S0.5 — OpenAPI, Scalar, and health endpoint

- **Objective:** Enable OpenAPI generation, mount Scalar UI, add a `/health` endpoint, confirm end-to-end.
- **Estimated duration:** 45 min
- **Prerequisites:** S0.4 complete.

**Tasks:**

- [ ] Add packages to `CvBuilder.Api`:
  - `dotnet add package Microsoft.AspNetCore.OpenApi`
  - `dotnet add package Scalar.AspNetCore`
- [ ] In `Program.cs`:

```csharp
builder.Services.AddOpenApi();
// ... after app build:
app.MapOpenApi();
if (app.Environment.IsDevelopment())
{
    app.MapScalarApiReference(); // defaults to /scalar
}
```

- [ ] Add health endpoint: `app.MapGet("/health", () => Results.Ok("Healthy")).WithName("Health").WithTags("Infrastructure");`.
- [ ] In `launchSettings.json`, set the HTTP profile to `applicationUrl: "http://localhost:5080"` for a stable port.
- [ ] Run `dotnet run --project src/CvBuilder.Api`.
- [ ] Open `http://localhost:5080/health` → `"Healthy"`.
- [ ] Open `http://localhost:5080/scalar` → Scalar UI renders.
- [ ] Open `http://localhost:5080/openapi/v1.json` → JSON OpenAPI document.

**Files touched:**
- Modify: `Program.cs`, `Properties/launchSettings.json`, `CvBuilder.Api.csproj`.

**Acceptance criteria:**
- [ ] `/health` returns 200 with body `"Healthy"`.
- [ ] `/scalar` renders a UI listing the Health endpoint.
- [ ] `/openapi/v1.json` returns a valid OpenAPI 3.1 document.

**Verification:**
```bash
dotnet run --project src/CvBuilder.Api &
curl -i http://localhost:5080/health
curl -s http://localhost:5080/openapi/v1.json | head -40
```

**Technical concepts:** Native .NET 10 OpenAPI generation (`AddOpenApi`), differences vs Swashbuckle, Scalar as a modern alternative to Swagger UI, `MapGet` with endpoint metadata.

**Suggested commit message:**
```
feat(api): add OpenAPI, Scalar UI, and health endpoint
```

---

### Session S0.6 — Folder scaffolding and CLAUDE.md refresh

- **Objective:** Create the agreed folder skeleton (`Features/`, `Domain/`, `Pdf/`, `Common/`), add placeholder `.gitkeep` files, update `CLAUDE.md` to reflect the now-populated repo.
- **Estimated duration:** 30 min
- **Prerequisites:** S0.5 complete.

**Tasks:**

- [ ] Under `src/CvBuilder.Api/`, create empty directories: `Features/`, `Domain/`, `Pdf/`, `Common/Errors/`, `Common/Validation/`.
- [ ] Add `.gitkeep` files in each so git tracks them (git ignores empty dirs).
- [ ] Under `tests/CvBuilder.Api.Tests/`, create: `Features/`, `Integration/`, `Support/`, each with `.gitkeep`.
- [ ] Update `CLAUDE.md`:
  - Remove the "state of the project" paragraph that said scaffolding is untouched.
  - Add a short "architecture at a glance" section pointing to the spec.
  - Add a "where to find things" paragraph listing the top-level folders.
- [ ] Update `PROGRESS.md` to mark P0 complete and set P1 as current.

**Files touched:**
- Create: `.gitkeep` placeholders across new folders.
- Modify: `CLAUDE.md`, `PROGRESS.md`.

**Acceptance criteria:**
- [ ] `git status` shows all folders tracked (via `.gitkeep`).
- [ ] `CLAUDE.md` no longer says the frontend/backend are untouched scaffolds.
- [ ] `PROGRESS.md` reflects P0 done.

**Verification:**
```bash
find backend/src/CvBuilder.Api -type d -name 'Features' -o -name 'Domain' -o -name 'Pdf' -o -name 'Common'
cat PROGRESS.md | head -20
```

**Technical concepts:** `.gitkeep` convention, keeping agent-facing docs in sync with the codebase.

**Suggested commit message:**
```
chore: scaffold feature folders and refresh CLAUDE.md
```

---

# PHASE P1 — Resume CRUD (full vertical slice)

**Goal:** Deliver a complete, tested vertical for the `Resume` aggregate — create, read, list, update, delete — with the frontend consuming it.

**Phase outcome:** User can go to the browser, create a resume with a title, see it in a list, open it, rename it, and delete it. Integration tests cover all five endpoints against a real Postgres container.

---

### Session S1.1 — Resume domain entity and migration

- **Objective:** Introduce the `Resume` entity with factory + rename behavior, Fluent configuration, and the first non-empty migration.
- **Estimated duration:** 45 min
- **Prerequisites:** P0 complete.

**Tasks:**

- [ ] Create `Domain/Resume.cs`:

```csharp
namespace CvBuilder.Api.Domain;

public sealed class Resume
{
    public Guid Id { get; private set; }
    public string Title { get; private set; } = default!;
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset UpdatedAt { get; private set; }

    private Resume() { } // EF

    public static Resume Create(string title)
    {
        var now = DateTimeOffset.UtcNow;
        return new Resume
        {
            Id = Guid.CreateVersion7(),
            Title = title.Trim(),
            CreatedAt = now,
            UpdatedAt = now,
        };
    }

    public void Rename(string title)
    {
        Title = title.Trim();
        UpdatedAt = DateTimeOffset.UtcNow;
    }
}
```

- [ ] Create `Persistence/Configurations/ResumeConfiguration.cs`:

```csharp
namespace CvBuilder.Api.Persistence.Configurations;

using CvBuilder.Api.Domain;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public sealed class ResumeConfiguration : IEntityTypeConfiguration<Resume>
{
    public void Configure(EntityTypeBuilder<Resume> b)
    {
        b.ToTable("resumes");
        b.HasKey(x => x.Id);
        b.Property(x => x.Title).HasMaxLength(120).IsRequired();
        b.Property(x => x.CreatedAt).IsRequired();
        b.Property(x => x.UpdatedAt).IsRequired();
    }
}
```

- [ ] Add `DbSet<Resume> Resumes` to `AppDbContext`.
- [ ] Create migration: `dotnet ef migrations add AddResume --project src/CvBuilder.Api`.
- [ ] Apply: `dotnet ef database update --project src/CvBuilder.Api`.
- [ ] Verify in psql: `\d resumes` shows the table.

**Files touched:**
- Create: `Domain/Resume.cs`, `Persistence/Configurations/ResumeConfiguration.cs`, migration files.
- Modify: `Persistence/AppDbContext.cs`.

**Acceptance criteria:**
- [ ] `dotnet build` passes.
- [ ] Migration applies without errors.
- [ ] `psql -c "\d resumes"` lists `id`, `title`, `created_at`, `updated_at` with correct types.

**Verification:**
```bash
dotnet ef migrations list --project src/CvBuilder.Api
docker exec cvbuilder-postgres psql -U cvbuilder -d cvbuilder -c "\d resumes"
```

**Technical concepts:** Rich domain entities (private setters + factory), `IEntityTypeConfiguration<T>` pattern, snake_case table naming (optional convention), `Guid.CreateVersion7` for time-ordered IDs.

**Suggested commit message:**
```
feat(domain): add Resume entity with migration
```

---

### Session S1.2 — CreateResume slice (endpoint + tests of handler)

- **Objective:** First vertical slice end-to-end: request → validation → DbContext → response. Unit test for the domain factory.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.1 complete.

**Tasks:**

- [ ] Create `Features/Resumes/CreateResume/CreateResumeRequest.cs`:

```csharp
namespace CvBuilder.Api.Features.Resumes.CreateResume;
using System.ComponentModel.DataAnnotations;

public sealed record CreateResumeRequest(
    [Required, StringLength(120, MinimumLength = 1)] string Title);
```

- [ ] Create `Features/Resumes/CreateResume/CreateResumeResponse.cs`:

```csharp
namespace CvBuilder.Api.Features.Resumes.CreateResume;

public sealed record CreateResumeResponse(Guid Id);
```

- [ ] Create `Features/Resumes/CreateResume/CreateResumeEndpoint.cs`:

```csharp
namespace CvBuilder.Api.Features.Resumes.CreateResume;

using CvBuilder.Api.Domain;
using CvBuilder.Api.Persistence;
using Microsoft.AspNetCore.Http.HttpResults;

public static class CreateResumeEndpoint
{
    public static IEndpointRouteBuilder MapCreateResume(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/resumes", Handle)
            .WithName("CreateResume")
            .WithTags("Resumes")
            .Produces<CreateResumeResponse>(StatusCodes.Status201Created)
            .ProducesValidationProblem();
        return app;
    }

    private static async Task<Created<CreateResumeResponse>> Handle(
        CreateResumeRequest request, AppDbContext db, CancellationToken ct)
    {
        var resume = Resume.Create(request.Title);
        db.Resumes.Add(resume);
        await db.SaveChangesAsync(ct);
        return TypedResults.Created($"/api/resumes/{resume.Id}", new CreateResumeResponse(resume.Id));
    }
}
```

- [ ] Create `Common/Extensions/EndpointRouteBuilderExtensions.cs` with `MapResumeEndpoints` aggregator that calls `MapCreateResume()`.
- [ ] In `Program.cs`, call `app.MapResumeEndpoints();`.
- [ ] Add unit test `tests/CvBuilder.Api.Tests/Features/Resumes/ResumeDomainTests.cs`:
  - `Create_sets_trimmed_title_and_timestamps()`
  - `Rename_updates_updatedAt_and_title()`
- [ ] Run unit tests.
- [ ] Smoke test with `curl` or Scalar: `POST /api/resumes {"title":"Test"}` → 201 with id.

**Files touched:**
- Create: `Features/Resumes/CreateResume/*`, `Common/Extensions/EndpointRouteBuilderExtensions.cs`, `tests/.../ResumeDomainTests.cs`.
- Modify: `Program.cs`.

**Acceptance criteria:**
- [ ] Unit tests pass.
- [ ] `POST /api/resumes` with valid body returns 201 and the ID is persisted (verify in psql).
- [ ] `POST /api/resumes` with empty title returns 400 (we will formalize this in S1.3 — accept the current behavior from the framework for now).

**Verification:**
```bash
dotnet test
curl -i -X POST http://localhost:5080/api/resumes -H 'content-type: application/json' -d '{"title":"My CV"}'
docker exec cvbuilder-postgres psql -U cvbuilder -d cvbuilder -c "SELECT id, title FROM resumes;"
```

**Technical concepts:** Vertical slice anatomy, `TypedResults.Created`, DI resolution of `DbContext` in Minimal API handlers, endpoint aggregator extension pattern.

**Suggested commit message:**
```
feat(resumes): add CreateResume slice with domain unit tests
```

---

### Session S1.3 — Validation filter and ProblemDetails errors

- **Objective:** Replace Minimal API default validation behavior with an explicit `ValidationFilter` endpoint filter that runs DataAnnotations and returns `ValidationProblem()`.
- **Estimated duration:** 45 min
- **Prerequisites:** S1.2 complete.

**Tasks:**

- [ ] Add `builder.Services.AddProblemDetails();` in `Program.cs`.
- [ ] Create `Common/Validation/ValidationFilter.cs`:

```csharp
namespace CvBuilder.Api.Common.Validation;

using System.ComponentModel.DataAnnotations;

public sealed class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var argument = ctx.Arguments.OfType<T>().FirstOrDefault();
        if (argument is null)
            return Results.Problem(statusCode: 400, title: "Missing request body.");

        var errors = new List<ValidationResult>();
        var isValid = Validator.TryValidateObject(argument, new ValidationContext(argument), errors, validateAllProperties: true);
        if (!isValid)
        {
            var dict = errors
                .SelectMany(e => e.MemberNames.DefaultIfEmpty(string.Empty).Select(m => (member: m, message: e.ErrorMessage ?? "Invalid")))
                .GroupBy(x => x.member)
                .ToDictionary(g => g.Key, g => g.Select(x => x.message).ToArray());
            return Results.ValidationProblem(dict);
        }

        return await next(ctx);
    }
}
```

- [ ] Apply filter to the CreateResume endpoint: `.AddEndpointFilter<ValidationFilter<CreateResumeRequest>>()`.
- [ ] (Optional for ergonomics) Create an extension `WithValidation<T>()` to shorten the call.
- [ ] Add an integration test (will fully wire in S1.7, but add stub if time permits).
- [ ] Manual smoke: `POST /api/resumes {"title":""}` → 400 with ProblemDetails body including `errors.Title`.

**Files touched:**
- Create: `Common/Validation/ValidationFilter.cs`.
- Modify: `Features/Resumes/CreateResume/CreateResumeEndpoint.cs`, `Program.cs`.

**Acceptance criteria:**
- [ ] Invalid body returns 400 with Content-Type `application/problem+json` and includes field-level errors.
- [ ] Valid body still returns 201.

**Verification:**
```bash
curl -i -X POST http://localhost:5080/api/resumes -H 'content-type: application/json' -d '{"title":""}'
# Expect: HTTP/1.1 400 Bad Request, application/problem+json, errors.Title present
```

**Technical concepts:** `IEndpointFilter<T>`, `Results.ValidationProblem`, `ProblemDetails` RFC 7807, generic open filter type registration.

**Suggested commit message:**
```
feat(api): add generic DataAnnotations validation filter with ProblemDetails
```

---

### Session S1.4 — Global exception handling

- **Objective:** Introduce `NotFoundException` and `DomainValidationException`, wire a custom `IExceptionHandler` that maps them to 404/422.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.3 complete.

**Tasks:**

- [ ] Create `Common/Errors/NotFoundException.cs` (constructors taking resource name + id).
- [ ] Create `Common/Errors/DomainValidationException.cs` (constructor taking message).
- [ ] Create `Common/Errors/GlobalExceptionHandler.cs` implementing `IExceptionHandler`:

```csharp
namespace CvBuilder.Api.Common.Errors;

using Microsoft.AspNetCore.Diagnostics;

public sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        var (status, title) = ex switch
        {
            NotFoundException nf          => (StatusCodes.Status404NotFound, nf.Message),
            DomainValidationException dv  => (StatusCodes.Status422UnprocessableEntity, dv.Message),
            _                             => (StatusCodes.Status500InternalServerError, "An unexpected error occurred.")
        };

        if (status == 500)
            logger.LogError(ex, "Unhandled exception at {Path}", ctx.Request.Path);

        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status,
            Title = title,
            Instance = ctx.Request.Path,
        }, cancellationToken: ct);
        return true;
    }
}
```

- [ ] Register in `Program.cs`:

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
// after build:
app.UseExceptionHandler();
```

- [ ] Add a temporary endpoint `/_dev/throw` in Dev only that throws a `NotFoundException` to verify.
- [ ] Verify: `GET /_dev/throw` → 404 with ProblemDetails. Remove the dev endpoint after confirming.

**Files touched:**
- Create: `Common/Errors/NotFoundException.cs`, `Common/Errors/DomainValidationException.cs`, `Common/Errors/GlobalExceptionHandler.cs`.
- Modify: `Program.cs`.

**Acceptance criteria:**
- [ ] Throwing `NotFoundException` from an endpoint yields 404 with ProblemDetails.
- [ ] Throwing `DomainValidationException` yields 422.
- [ ] Unknown exceptions yield 500 and are logged.

**Verification:** manual via temporary endpoint; permanent coverage comes with S1.5+ endpoints.

**Technical concepts:** `IExceptionHandler` (ASP.NET Core 8+), `UseExceptionHandler` middleware, mapping domain errors to HTTP semantics without leaking stack traces.

**Suggested commit message:**
```
feat(api): add global exception handler mapping domain errors to HTTP
```

---

### Session S1.5 — GetResume and ListResumes slices

- **Objective:** Two read endpoints with projection to response DTOs; use `AsNoTracking()`.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.4 complete.

**Tasks:**

- [ ] Create `Features/Resumes/GetResume/`:
  - `GetResumeResponse.cs` (record with Id, Title, CreatedAt, UpdatedAt).
  - `GetResumeEndpoint.cs`:
    - `MapGet("/api/resumes/{id:guid}", Handle)`. Handler returns `Ok<GetResumeResponse>`. 404 is produced by the global exception handler, not by a typed `NotFound` result — so the handler signature is `Task<Ok<GetResumeResponse>>`, **not** `Results<Ok<GetResumeResponse>, NotFound>`.
    - Query: `db.Resumes.AsNoTracking().Where(r => r.Id == id).Select(r => new GetResumeResponse(...)).FirstOrDefaultAsync(ct)`.
    - If null → `throw new NotFoundException("Resume", id);`. The `GlobalExceptionHandler` from S1.4 maps it to 404 ProblemDetails. This keeps the choice "throw domain errors; the handler converts them" consistent across all mutating/reading endpoints.
    - Document both 200 and 404 in OpenAPI via `.Produces<GetResumeResponse>(200)` and `.ProducesProblem(404)`.
- [ ] Create `Features/Resumes/ListResumes/`:
  - `ListResumesResponse.cs` (array of items; keep simple — no pagination yet).
  - `ListResumesEndpoint.cs`: `MapGet("/api/resumes", Handle)` ordered by `CreatedAt desc`.
- [ ] Register both in `MapResumeEndpoints`.
- [ ] Smoke test both endpoints.

**Files touched:**
- Create: `Features/Resumes/GetResume/*`, `Features/Resumes/ListResumes/*`.
- Modify: `Common/Extensions/EndpointRouteBuilderExtensions.cs`.

**Acceptance criteria:**
- [ ] `GET /api/resumes/{id}` returns 200 for known id, 404 for unknown.
- [ ] `GET /api/resumes` returns 200 with an array (possibly empty).

**Verification:**
```bash
curl -s http://localhost:5080/api/resumes | jq
curl -i http://localhost:5080/api/resumes/00000000-0000-0000-0000-000000000000
# Expect: 404 ProblemDetails
```

**Technical concepts:** `AsNoTracking` for reads, LINQ projection with `Select`, `Results<T1, T2>` typed unions.

**Suggested commit message:**
```
feat(resumes): add GetResume and ListResumes endpoints
```

---

### Session S1.6 — UpdateResume and DeleteResume slices

- **Objective:** Mutation endpoints using domain methods; both load the entity (tracked) and persist.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.5 complete.

**Tasks:**

- [ ] Create `Features/Resumes/UpdateResume/`:
  - `UpdateResumeRequest.cs` (`[Required, StringLength(120, MinimumLength = 1)] Title`).
  - `UpdateResumeEndpoint.cs`:
    - `MapPut("/api/resumes/{id:guid}", Handle)`. Handler returns `NoContent` (the typed result `Task<NoContent>`). 404 is produced by the global exception handler when the entity is missing; **not** modeled as a typed `NotFound` result on the handler signature.
    - Load `resume = await db.Resumes.FindAsync([id], ct); if (resume is null) throw new NotFoundException("Resume", id);`.
    - `resume.Rename(request.Title); await db.SaveChangesAsync(ct);`.
    - OpenAPI metadata: `.Produces(204)` + `.ProducesProblem(404)` + `.ProducesValidationProblem()`.
  - Apply `ValidationFilter<UpdateResumeRequest>`.
- [ ] Create `Features/Resumes/DeleteResume/`:
  - `DeleteResumeEndpoint.cs`:
    - `MapDelete("/api/resumes/{id:guid}", Handle)`. Handler returns `NoContent` (same pattern as Update).
    - Load + `db.Resumes.Remove(resume)` + save (parallels Update). 404 via `NotFoundException` → global handler.
    - OpenAPI metadata: `.Produces(204)` + `.ProducesProblem(404)`.
- [ ] Register both.
- [ ] Smoke test.

**Files touched:**
- Create: `Features/Resumes/UpdateResume/*`, `Features/Resumes/DeleteResume/*`.
- Modify: `Common/Extensions/EndpointRouteBuilderExtensions.cs`.

**Acceptance criteria:**
- [ ] PUT with valid body → 204; subsequent GET shows new title and newer `UpdatedAt`.
- [ ] PUT for unknown id → 404.
- [ ] DELETE for existing id → 204; subsequent GET → 404.
- [ ] DELETE for unknown id → 404.

**Verification:** `curl` suite exercising the four cases.

**Technical concepts:** EF Core change tracking for updates, `DbContext.FindAsync` vs `FirstOrDefaultAsync`, deciding between tracked-and-save vs `ExecuteDeleteAsync`.

**Suggested commit message:**
```
feat(resumes): add UpdateResume and DeleteResume endpoints
```

---

### Session S1.7 — Testcontainers infrastructure

- **Objective:** Set up `WebApplicationFactory<Program>` wired to a Testcontainers PostgreSQL instance; add first integration test for `CreateResume`.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.6 complete. Docker Desktop running.

**Tasks:**

- [ ] In `CvBuilder.Api.Tests`, add packages:
  - `dotnet add package Microsoft.AspNetCore.Mvc.Testing`
  - `dotnet add package Testcontainers.PostgreSql`
  - `dotnet add package Shouldly` *(assertion lib choice is deferred, but we need one — defaulting to Shouldly; justify in session)*
- [ ] Create `tests/Support/PostgresFixture.cs` (IAsyncLifetime) that spins up `PostgreSqlBuilder` container and exposes `ConnectionString`.
- [ ] Create `tests/Integration/CvBuilderApiFactory.cs` extending `WebApplicationFactory<Program>`, overriding `ConfigureWebHost` to replace the `DbContextOptions<AppDbContext>` registration with one that points to the test container's connection string, and applying migrations in `InitializeAsync`.
- [ ] Create a collection fixture so one container is shared across tests in that collection.
- [ ] Add `tests/Integration/CreateResumeTests.cs` with:
  - `Post_with_valid_body_returns_201_and_persists()`.
  - `Post_with_empty_title_returns_400_with_problem_details()`.
- [ ] Run `dotnet test`.

**Files touched:**
- Create: `tests/Support/PostgresFixture.cs`, `tests/Integration/CvBuilderApiFactory.cs`, `tests/Integration/CreateResumeTests.cs`.
- Modify: `CvBuilder.Api.Tests.csproj` (packages).

**Acceptance criteria:**
- [ ] `dotnet test` spins up a Postgres container, applies migrations, runs tests, tears down container.
- [ ] Both tests pass.
- [ ] No leftover containers after test run (`docker ps -a` shows no stuck Testcontainers).

**Verification:**
```bash
dotnet test backend/tests/CvBuilder.Api.Tests
docker ps -a | grep testcontainers
```

**Technical concepts:** `WebApplicationFactory<TEntryPoint>`, `IAsyncLifetime`, Testcontainers lifecycle, collection fixtures for sharing expensive setup, replacing DI registrations in tests.

**Suggested commit message:**
```
test(api): add Testcontainers-backed integration test infrastructure
```

---

### Session S1.8 — Integration tests for the full Resume CRUD

- **Objective:** Cover Get, List, Update, Delete with integration tests reusing the fixture from S1.7.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.7 complete.

**Tasks:**

- [ ] Add `tests/Integration/GetResumeTests.cs`:
  - `Returns_201_values_matching_created()`.
  - `Returns_404_for_unknown_id_with_problem_details()`.
- [ ] Add `tests/Integration/ListResumesTests.cs`:
  - `Empty_db_returns_empty_array()`.
  - `Returns_newest_first()`.
- [ ] Add `tests/Integration/UpdateResumeTests.cs`:
  - `Updates_title_and_updatedAt_bumps()`.
  - `Returns_404_for_unknown_id()`.
  - `Returns_400_for_empty_title()`.
- [ ] Add `tests/Integration/DeleteResumeTests.cs`:
  - `Deletes_existing_resume_returns_204()`.
  - `Returns_404_for_unknown_id()`.
- [ ] Run full suite; it should remain under 1 minute due to shared container.

**Files touched:**
- Create: `tests/Integration/{Get,List,Update,Delete}ResumeTests.cs`.

**Acceptance criteria:**
- [ ] All tests pass.
- [ ] Each test cleans up or isolates its data (truncate between tests OR each test uses a unique title; prefer truncate for determinism — add a helper in `CvBuilderApiFactory` that executes `TRUNCATE resumes CASCADE`).

**Verification:** `dotnet test` reports a growing green suite.

**Technical concepts:** Test isolation strategies (truncate vs new container vs transactional rollback), test naming (behavior-driven), avoiding snapshot coupling.

**Suggested commit message:**
```
test(resumes): cover full CRUD with integration tests
```

---

### Session S1.9 — Frontend foundation and resume listing

- **Objective:** Strip the Vite starter from `App.tsx`, add a typed API client, a router, and a "resumes list" page that calls `GET /api/resumes`.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.8 complete. Backend running.

**Tasks:**

- [ ] In `frontend/`, add `react-router` v7 (or latest): `npm i react-router`. Add `zod` for runtime response parsing: `npm i zod`.
- [ ] In `vite.config.ts`, add a dev proxy for `/api` → `http://localhost:5080`.
- [ ] Create `src/api/client.ts` with `apiFetch<T>(path, init?)` that:
  - Prefixes `/api`,
  - Parses JSON,
  - Throws typed errors on non-2xx (reading ProblemDetails if present).
- [ ] Create `src/api/resumes.ts` with `listResumes()` returning `ResumeSummary[]` (zod schema + `.parse`).
- [ ] Replace `App.tsx` and `main.tsx` to wire a `RouterProvider`.
- [ ] Create `src/features/resumes/ResumesListPage.tsx` that fetches and renders titles with timestamps.
- [ ] Create a bare `src/features/resumes/CreateResumePage.tsx` stub (filled in S1.10).
- [ ] Delete `App.css`, `assets/` scaffold assets we no longer use.
- [ ] Update `index.css` to a minimal reset.
- [ ] Visually verify: start backend + Postgres + `npm run dev`, open the app, see empty state (or test rows if any from prior sessions).

**Files touched:**
- Create: `src/api/{client,resumes}.ts`, `src/features/resumes/*.tsx`, `src/router.tsx`.
- Delete / replace: `src/App.tsx`, `src/App.css`, `src/assets/*`, scaffold content in `src/main.tsx`, `src/index.css`.
- Modify: `vite.config.ts`, `package.json`, `index.html` title.

**Acceptance criteria:**
- [ ] `npm run build` succeeds.
- [ ] `npm run lint` passes.
- [ ] The browser shows the resumes list without network errors.

**Verification:**
```bash
cd frontend
npm run lint
npm run build
# then run dev and inspect in browser
```

**Technical concepts:** Vite dev proxy, zod response parsing (lightweight runtime validation), file-based feature folders in React, minimal CSS reset vs frameworks.

**Suggested commit message:**
```
feat(frontend): add API client, router, and resumes list page
```

---

### Session S1.10 — Frontend CRUD flows

- **Objective:** Wire create, edit (rename), delete UIs; the user can round-trip a resume end-to-end in the browser.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.9 complete.

**Tasks:**

- [ ] Extend `src/api/resumes.ts` with `createResume`, `getResume`, `updateResume`, `deleteResume`.
- [ ] Implement `CreateResumePage.tsx` with a title input + submit, redirect to the edit page on success.
- [ ] Implement `EditResumePage.tsx` that loads the resume, lets the user rename and save.
- [ ] Add a delete button in the list (with confirm).
- [ ] Show inline error messages from ProblemDetails (parse `errors` dict for field-level errors).
- [ ] Browser smoke test: create → rename → delete.

**Files touched:**
- Modify: `src/api/resumes.ts`, `src/features/resumes/*.tsx`, `src/router.tsx`.

**Acceptance criteria:**
- [ ] Full round trip works in the browser.
- [ ] Validation errors from the API show up in the UI inline, not as toasts.
- [ ] `npm run build && npm run lint` pass.

**Verification:** manual browser testing + build.

**Technical concepts:** Controlled form inputs in React 19, loader/action patterns in react-router, consuming ProblemDetails errors in the UI.

**Suggested commit message:**
```
feat(frontend): add create/edit/delete resume flows
```

---

### Session S1.11 — Introductory GitHub Actions CI (backend)

- **Objective:** Introduce CI early, now that we have meaningful tests (S1.7–S1.8). A workflow runs `restore → build → test` for the backend on every push and PR. This catches regressions before P2 begins stacking new slices on top.
- **Estimated duration:** 60 min
- **Prerequisites:** S1.10 complete. GitHub repo wired (S0.1). Testcontainers works locally (S1.7).

**Tasks:**

- [ ] Create `.github/workflows/backend-ci.yml`:

```yaml
name: backend-ci
on:
  push:
    branches: [main]
    paths: ['backend/**', '.github/workflows/backend-ci.yml']
  pull_request:
    branches: [main]
    paths: ['backend/**', '.github/workflows/backend-ci.yml']
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.x'
      - name: Restore
        run: dotnet restore backend/CvBuilder.slnx
      - name: Build
        run: dotnet build backend/CvBuilder.slnx --no-restore -c Release
      - name: Test
        run: dotnet test backend/CvBuilder.slnx --no-build -c Release --logger "console;verbosity=normal"
```

- [ ] Verify Docker is available on `ubuntu-latest` (it is, out of the box) so Testcontainers can start Postgres.
- [ ] Open a short-lived PR (e.g., a tiny README edit on a feature branch) to confirm the workflow runs on PR events.
- [ ] Merge the branch; confirm the workflow runs on `main` push.
- [ ] Investigate and fix any first-run failures (likely around dotnet SDK version pin or Testcontainers startup time).

**Files touched:**
- Create: `.github/workflows/backend-ci.yml`.

**Acceptance criteria:**
- [ ] The workflow is green on `main` after the first push.
- [ ] The workflow triggers on PRs targeting `main`.
- [ ] All existing integration tests run and pass in CI.

**Verification:**
```bash
gh run list --workflow=backend-ci.yml -L 3
gh run view --log-failed    # if any failed
```

**Technical concepts:** GitHub Actions anatomy (`on`, `jobs`, `steps`, `runs-on`), path filters to avoid unrelated triggers, `actions/setup-dotnet`, Testcontainers under `ubuntu-latest`, CI as a safety net before stacking features.

**Suggested commit message:**
```
ci: add backend GitHub Actions workflow (build + test)
```

---

# PHASE P2 — CV Sections

**Goal:** Add six resume sections (WorkExperience, Education, Skills, Languages, Certifications, Projects) using the pattern established in P1. Each section lives at `/api/resumes/{resumeId}/<section>/{itemId}`.

**Approach:** Pair backend + frontend for each section when possible. Reuse the ValidationFilter and ExceptionHandler from P1. Add migrations incrementally.

> **Pattern reference**: all sessions in P2 follow the shape established in S1.1 (entity + config + migration), S1.2 (slice: request/response/endpoint + domain), S1.3 (ValidationFilter applied), S1.6 (update + delete pattern), S1.8 (integration tests). When this plan says "follow the P1 pattern", it means: a `Create`, `Update`, `Delete` slice, a `List` under the parent resume, DataAnnotations on the request, integration tests for happy path + 404 + 400.

---

### Session S2.1 — WorkExperience: domain, migration, backend slices

- **Objective:** Add `WorkExperience` as a child of `Resume`. Backend CRUD slices nested at `/api/resumes/{resumeId}/work-experiences`.
- **Estimated duration:** 60 min
- **Prerequisites:** P1 complete.

**Tasks:**

- [ ] Create `Domain/WorkExperience.cs`: fields `Id`, `ResumeId`, `Company`, `Role`, `StartDate` (DateOnly), `EndDate` (DateOnly?), `Description`, `Order`. Factory + `Update(...)` method.
- [ ] Update `Domain/Resume.cs`: add `private readonly List<WorkExperience> _workExperiences = new();` and `public IReadOnlyList<WorkExperience> WorkExperiences => _workExperiences;`. Add `AddWorkExperience(...)` method.
- [ ] Create `Persistence/Configurations/WorkExperienceConfiguration.cs` — map FK `ResumeId`, cascade delete, index on `(ResumeId, Order)`.
- [ ] Add migration `AddWorkExperience`.
- [ ] Create slices under `Features/WorkExperiences/{Create,List,Update,Delete}WorkExperience/` — endpoints prefixed with `/api/resumes/{resumeId:guid}/work-experiences`.
- [ ] Register endpoints via a new `MapWorkExperienceEndpoints` extension.
- [ ] Smoke test with curl.

**Files touched:**
- Create: `Domain/WorkExperience.cs`, `Persistence/Configurations/WorkExperienceConfiguration.cs`, `Features/WorkExperiences/**`, new migration.
- Modify: `Domain/Resume.cs`, `Persistence/AppDbContext.cs` (add DbSet), `Common/Extensions/EndpointRouteBuilderExtensions.cs`, `Program.cs`.

**Acceptance criteria:**
- [ ] CRUD endpoints work via curl.
- [ ] Deleting a resume cascades to work experiences (verify in psql).
- [ ] Build + existing tests still pass.

**Verification:**
```bash
curl -X POST http://localhost:5080/api/resumes/<id>/work-experiences -H 'content-type: application/json' -d '{"company":"Acme","role":"Dev","startDate":"2020-01-01"}'
docker exec cvbuilder-postgres psql -U cvbuilder -d cvbuilder -c "SELECT * FROM work_experiences;"
```

**Technical concepts:** Owned vs child entities, cascade delete, `DateOnly` with Npgsql, EF Core backing fields (`_workExperiences`), aggregate root keeping ownership of children.

**Suggested commit message:**
```
feat(sections): add WorkExperience aggregate with CRUD endpoints
```

---

### Session S2.2 — WorkExperience: integration tests and frontend UI

- **Objective:** Integration tests for the WorkExperience CRUD + a UI panel inside the resume edit page.
- **Estimated duration:** 60 min
- **Prerequisites:** S2.1 complete.

**Tasks:**

- [ ] Add integration tests under `tests/Integration/WorkExperiences/` for the four endpoints, mirroring S1.8.
- [ ] Add frontend client methods `listWorkExperiences`, `createWorkExperience`, `updateWorkExperience`, `deleteWorkExperience`.
- [ ] Add a `WorkExperiencesPanel.tsx` component inside the resume edit page: list items with inline edit + add + delete.
- [ ] Smoke test in the browser.

**Files touched:**
- Create: tests under `tests/Integration/WorkExperiences/`, `src/features/sections/WorkExperiencesPanel.tsx`, `src/api/workExperiences.ts`.
- Modify: `src/features/resumes/EditResumePage.tsx`.

**Acceptance criteria:**
- [ ] All integration tests pass.
- [ ] A work experience can be added, edited, reordered (if trivial), and deleted in the browser.

**Verification:** `dotnet test` + manual browser.

**Technical concepts:** Nested resources in REST, UI patterns for list + edit panels, optimistic vs pessimistic updates (go with pessimistic — simpler).

**Suggested commit message:**
```
feat(frontend+tests): add WorkExperience UI and integration tests
```

---

### Session S2.3 — Education: backend slice (reuses pattern)

- **Objective:** Add `Education` section. Pattern now established; expected faster.
- **Estimated duration:** 45 min
- **Prerequisites:** S2.2 complete.

**Tasks:**

- [ ] `Domain/Education.cs`: `Institution`, `Degree`, `FieldOfStudy`, `StartDate`, `EndDate?`, `Description`, `Order`.
- [ ] Configuration + migration.
- [ ] Slices under `Features/Education/`.
- [ ] Register endpoints.

**Files touched:** Analogous to S2.1 but for Education.

**Acceptance criteria:**
- [ ] CRUD endpoints respond correctly.
- [ ] Migration applied.
- [ ] Existing tests pass.

**Technical concepts:** Reinforcing the slice pattern; nothing new.

**Suggested commit message:**
```
feat(sections): add Education aggregate with CRUD endpoints
```

---

### Session S2.4 — Education: integration tests and frontend UI

- **Objective:** Mirror S2.2 for Education.
- **Estimated duration:** 60 min
- **Prerequisites:** S2.3.

**Tasks:**

- [ ] Integration tests mirroring S2.2.
- [ ] `EducationPanel.tsx` inside the resume edit page.

**Acceptance criteria / Verification / Concepts:** mirror S2.2.

**Suggested commit message:**
```
feat(frontend+tests): add Education UI and integration tests
```

---

### Session S2.5 — Skills: backend + frontend

- **Objective:** Simpler section — just name + level enum. Can bundle back + front in one session.
- **Estimated duration:** 60 min
- **Prerequisites:** S2.4.

**Tasks:**

- [ ] `Domain/Skill.cs`: `Name` (max 60), `Level` (enum: Beginner, Intermediate, Advanced, Expert), `Order`.
- [ ] Configuration + migration.
- [ ] Slices (Create/List/Update/Delete).
- [ ] Integration tests (lighter than larger sections — happy path + 404).
- [ ] Frontend `SkillsPanel.tsx` with a compact chip-style UI.

**Files touched:** backend slice set, migration, integration test file, `SkillsPanel.tsx`, `src/api/skills.ts`.

**Acceptance criteria:**
- [ ] Round-trip works end-to-end in the browser.
- [ ] Tests pass.

**Technical concepts:** Enums in EF Core (`HasConversion<string>()` for readable storage).

**Suggested commit message:**
```
feat(sections): add Skills section (backend + frontend)
```

---

### Session S2.6 — Languages: backend + frontend

- **Objective:** `Language` section (Name, Proficiency enum: Basic/Conversational/Fluent/Native).
- **Estimated duration:** 60 min
- **Prerequisites:** S2.5.

**Tasks:** mirror S2.5 for Language.

**Suggested commit message:**
```
feat(sections): add Languages section (backend + frontend)
```

---

### Session S2.7 — Certifications: backend + frontend

- **Objective:** `Certification` section (Name, Issuer, IssueDate, ExpirationDate?, CredentialUrl?).
- **Estimated duration:** 60 min
- **Prerequisites:** S2.6.

**Tasks:** mirror S2.5 for Certification. Add URL validation (`[Url]` on CredentialUrl).

**Suggested commit message:**
```
feat(sections): add Certifications section (backend + frontend)
```

---

### Session S2.8 — Projects: backend + frontend

- **Objective:** `Project` section (Name, Description, TechStack list of strings, Url?).
- **Estimated duration:** 60 min
- **Prerequisites:** S2.7.

**Tasks:** mirror S2.5. For `TechStack`, use `HasConversion` to persist as `text[]` (Npgsql supports arrays natively).

**Technical concepts:** PostgreSQL array columns via EF Core + Npgsql.

**Suggested commit message:**
```
feat(sections): add Projects section with tech stack array
```

---

### Session S2.9 — Personal information fields on Resume

- **Objective:** Add `FullName`, `Headline`, `Email`, `Phone?`, `Location?`, `Summary?`, `Website?` directly to the `Resume` entity (not a separate aggregate — they are 1-to-1).
- **Estimated duration:** 60 min
- **Prerequisites:** S2.8.

**Tasks:**

- [ ] Add properties to `Domain/Resume.cs`; add `UpdatePersonalInfo(...)` method.
- [ ] Update `ResumeConfiguration.cs` with max lengths and nullability.
- [ ] Migration.
- [ ] Extend `UpdateResume` slice (or add a new `UpdateResumeDetails` slice) — decide during session; prefer extending to avoid slice proliferation unless the request bodies diverge too much. Confirm by writing out both options at step 4.
- [ ] Extend `GetResumeResponse` to include these fields.
- [ ] Frontend: personal info form in `EditResumePage.tsx`.
- [ ] Integration tests for the extended endpoint.

**Files touched:** `Domain/Resume.cs`, config, migration, update slice, response record, frontend edit page.

**Acceptance criteria:**
- [ ] Personal info persists and round-trips in the UI.
- [ ] Validation on `Email` (`[EmailAddress]`), `Website` (`[Url]`).

**Technical concepts:** When to add fields to an existing entity vs create a new one, when to extend a slice vs create a new one.

**Suggested commit message:**
```
feat(resumes): add personal info fields to Resume
```

---

### Session S2.10 — UX polish for section panels

- **Objective:** Section reordering (drag-and-drop OR simple up/down buttons — pick the simpler one; `Order` field already exists), empty states, loading states, consistent styling across sections.
- **Estimated duration:** 60 min
- **Prerequisites:** S2.9.

**Tasks:**

- [ ] Add reorder endpoints per section: `PUT /api/resumes/{id}/work-experiences/reorder` taking an ordered array of ids. Repeat for each section OR add a single generic reorder endpoint using a discriminator (prefer per-section to stay slice-pure).
- [ ] Frontend: add up/down buttons on each list item, calling the reorder endpoint.
- [ ] Add minimal CSS to match a cohesive visual.

**Files touched:** reorder slices + frontend panels + stylesheet.

**Acceptance criteria:**
- [ ] Items can be reordered; order persists across refresh.

**Technical concepts:** Reorder patterns (dense vs sparse `Order`), batching reorders in a single request.

**Suggested commit message:**
```
feat(frontend): add section reordering and visual polish
```

---

# PHASE P3 — PDF Generation

**Goal:** Produce a downloadable PDF from any resume via `GET /api/resumes/{id}/pdf`, rendered with QuestPDF.

---

### Session S3.1 — IPdfRenderer and QuestPDF skeleton

- **Objective:** Install QuestPDF, declare `IPdfRenderer` abstraction, implement a renderer that produces a placeholder single-page PDF.
- **Estimated duration:** 60 min
- **Prerequisites:** P2 complete.

**Tasks:**

- [ ] `dotnet add package QuestPDF` in `CvBuilder.Api`.
- [ ] Register QuestPDF license: in `Program.cs`, `QuestPDF.Settings.License = QuestPDF.Infrastructure.LicenseType.Community;` and document in README.
- [ ] Create `Pdf/IPdfRenderer.cs`:

```csharp
namespace CvBuilder.Api.Pdf;

using CvBuilder.Api.Domain;

public interface IPdfRenderer
{
    Task<byte[]> RenderAsync(Resume resume, string templateKey, CancellationToken ct);
}
```

- [ ] Create `Pdf/QuestPdfRenderer.cs` with a minimal document that writes "Resume: {Title}" on page 1.
- [ ] Create `Common/Extensions/PdfServiceCollectionExtensions.cs` with `AddPdfGeneration()`.
- [ ] Register in `Program.cs`.

**Files touched:** `Pdf/IPdfRenderer.cs`, `Pdf/QuestPdfRenderer.cs`, `Common/Extensions/PdfServiceCollectionExtensions.cs`, `Program.cs`, `README.md`.

**Acceptance criteria:**
- [ ] Build succeeds.
- [ ] Unit test: `QuestPdfRenderer.RenderAsync` returns non-empty byte array for a sample resume (not asserting PDF structure, just non-emptiness).

**Technical concepts:** QuestPDF fluent API, license activation, interface-based swap-out, byte-stream output.

**Suggested commit message:**
```
feat(pdf): add IPdfRenderer and QuestPDF skeleton
```

---

### Session S3.2 — Baseline template rendering

- **Objective:** Expand the renderer to produce a complete baseline template using all resume data (personal info, work experience, education, skills, languages, certifications, projects).
- **Estimated duration:** 60 min
- **Prerequisites:** S3.1.

**Tasks:**

- [ ] Create `Pdf/Templates/BaselineTemplate.cs` — a class with a method `Compose(IDocumentContainer, Resume)`.
- [ ] Wire the renderer to invoke the baseline template when `templateKey == "baseline"` or any value for now.
- [ ] Design the layout: header (name + headline + contact), summary paragraph, two columns for sections, consistent typography.
- [ ] Verify by running an endpoint (S3.3) OR by writing a test that saves `*.pdf` to disk for manual inspection in dev.

**Files touched:** `Pdf/Templates/BaselineTemplate.cs`, `Pdf/QuestPdfRenderer.cs`.

**Acceptance criteria:**
- [ ] Byte array ≥ 10 KB for a non-trivial resume.
- [ ] Visually inspecting the output PDF (one-time dev step) shows all sections.

**Technical concepts:** QuestPDF composition (Container, Column, Row, Text), typography scale, page breaking.

**Suggested commit message:**
```
feat(pdf): implement baseline template with all resume sections
```

---

### Session S3.3 — Export endpoint and frontend download

- **Objective:** Add `GET /api/resumes/{id}/pdf` that streams the PDF, and a "Download PDF" button in the frontend.
- **Estimated duration:** 60 min
- **Prerequisites:** S3.2.

**Tasks:**

- [ ] Create `Features/Resumes/ExportResumePdf/ExportResumePdfEndpoint.cs`:
  - Loads full resume with all children (`.Include(...)` chain).
  - If null → `NotFoundException`.
  - Calls `IPdfRenderer.RenderAsync`.
  - Returns `Results.File(bytes, "application/pdf", fileDownloadName: $"{resume.Title}.pdf")`.
- [ ] Register endpoint.
- [ ] Frontend: `downloadResumePdf(id)` triggers a window.open or anchor download.
- [ ] Integration test: `GET /api/resumes/{id}/pdf` returns 200 with `application/pdf` content type and non-empty body.

**Files touched:** `Features/Resumes/ExportResumePdf/*`, frontend edit page, `src/api/resumes.ts`.

**Acceptance criteria:**
- [ ] Download works in browser.
- [ ] File opens as a valid PDF in a reader.
- [ ] Integration test asserts content type + non-empty body.

**Technical concepts:** `Results.File`, `.Include` + `.ThenInclude` for eager loading, serving binary content via Minimal APIs, browser file-download semantics.

**Suggested commit message:**
```
feat(pdf): add export endpoint and download button
```

---

# PHASE P4 — Templates

**Goal:** Multiple PDF templates with a selector in the UI.

---

### Session S4.1 — Template strategy refactor

- **Objective:** Introduce `IPdfTemplate` with a keyed registry; refactor `QuestPdfRenderer` to resolve the template by key.
- **Estimated duration:** 60 min
- **Prerequisites:** P3 complete.

**Tasks:**

- [ ] Create `Pdf/IPdfTemplate.cs` with `string Key { get; }`, `string DisplayName { get; }`, `void Compose(IDocumentContainer container, Resume resume);`.
- [ ] Refactor `BaselineTemplate` to implement `IPdfTemplate` (Key = `"baseline"`).
- [ ] Update `QuestPdfRenderer`: inject `IEnumerable<IPdfTemplate>`, select by key, throw `NotFoundException("Template", key)` if missing.
- [ ] Register templates in DI (`services.AddSingleton<IPdfTemplate, BaselineTemplate>()`).
- [ ] All existing tests still pass.

**Files touched:** `Pdf/IPdfTemplate.cs`, `Pdf/Templates/BaselineTemplate.cs`, `Pdf/QuestPdfRenderer.cs`, `Common/Extensions/PdfServiceCollectionExtensions.cs`.

**Acceptance criteria:**
- [ ] Existing PDF generation continues to work.
- [ ] Unknown template key returns 404.

**Technical concepts:** Strategy pattern via DI collection, singleton template instances.

**Suggested commit message:**
```
refactor(pdf): introduce IPdfTemplate strategy with keyed resolution
```

---

### Session S4.2 — Second template

- **Objective:** Implement a visually distinct second template (`"compact"` or `"modern"` — pick one). Query-parameter wiring on the export endpoint is deferred to S4.3; this session only adds the template and verifies it via the renderer directly.
- **Estimated duration:** 60 min
- **Prerequisites:** S4.1.

**Tasks:**

- [ ] Create `Pdf/Templates/CompactTemplate.cs` implementing `IPdfTemplate` (Key = `"compact"`, DisplayName = `"Compact"`).
- [ ] Register in DI next to `BaselineTemplate`.
- [ ] Add a unit test `tests/Features/Pdf/QuestPdfRendererTests.cs` (or extend the one from S3.1) that:
  - Calls `renderer.RenderAsync(sampleResume, "compact", ct)` and asserts a non-empty byte array.
  - Calls `renderer.RenderAsync(sampleResume, "baseline", ct)` and asserts the two byte arrays **differ** (byte-level inequality is enough to prove the key selected a different template).
  - Calls `renderer.RenderAsync(sampleResume, "unknown", ct)` and asserts `NotFoundException` is thrown.

**Files touched:** `Pdf/Templates/CompactTemplate.cs`, `Common/Extensions/PdfServiceCollectionExtensions.cs` (DI registration), `tests/Features/Pdf/QuestPdfRendererTests.cs`.

**Acceptance criteria:**
- [ ] Both templates resolve by key and produce non-empty output.
- [ ] Byte output differs between templates for the same resume.
- [ ] Unknown template key throws `NotFoundException` (which S4.3 will surface as HTTP 404 via the global handler).

**Technical concepts:** Reinforce the strategy pattern; practice QuestPDF layout APIs; keep per-session scope tight — expose the choice to the HTTP layer only when its own session (S4.3) arrives.

**Suggested commit message:**
```
feat(pdf): add second template (compact) with renderer tests
```

---

### Session S4.3 — Templates endpoint and frontend selector

- **Objective:** `GET /api/templates` returns available templates; frontend lets the user pick before downloading.
- **Estimated duration:** 60 min
- **Prerequisites:** S4.2.

**Tasks:**

- [ ] Create `Features/Templates/ListTemplatesEndpoint.cs` returning `{ key, displayName }[]` sourced from the registered `IEnumerable<IPdfTemplate>`.
- [ ] Update `ExportResumePdfEndpoint`:
  - Read `?template=` query parameter (default `"baseline"`).
  - Pass to `IPdfRenderer.RenderAsync`. If the renderer throws `NotFoundException` (unknown key), the global exception handler returns 404 ProblemDetails.
  - OpenAPI metadata: `.Produces<FileResult>(200)` + `.ProducesProblem(404)`.
- [ ] Add a template selector (`<select>` or chip list) in the frontend edit page; store the chosen template in local state; wire it into the download URL.
- [ ] Add integration test for `GET /api/templates` (returns the two templates with their keys).
- [ ] Add integration test for `GET /api/resumes/{id}/pdf?template=unknown` → 404 ProblemDetails.

**Files touched:** `Features/Templates/*`, `Features/Resumes/ExportResumePdf/ExportResumePdfEndpoint.cs`, frontend edit page, `src/api/templates.ts`, integration test files.

**Acceptance criteria:**
- [ ] UI shows the templates list; downloading uses the chosen template.
- [ ] Default (no `?template=`) still renders the baseline template.
- [ ] Unknown template key returns 404 ProblemDetails.

**Technical concepts:** Query parameter binding in Minimal APIs with defaults, surfacing domain exceptions through the global handler, UI state for ephemeral choices.

**Suggested commit message:**
```
feat(templates): add templates endpoint and selector UI
```

---

# PHASE P5 — Duplicate

**Goal:** User can duplicate a resume with one click to produce a copy they can customize for a different application.

---

### Session S5.1 — Resume.Duplicate domain method and endpoint

- **Objective:** Add `Resume.Duplicate()` that returns a new `Resume` with deep-copied children; expose via `POST /api/resumes/{id}/duplicate`.
- **Estimated duration:** 60 min
- **Prerequisites:** P4 complete.

**Tasks:**

- [ ] Add `Duplicate()` method on `Resume` that returns a new `Resume` with:
  - New Id (`Guid.CreateVersion7()`).
  - Title `$"{this.Title} (copy)"`.
  - New timestamps.
  - Deep copies of all section lists (new ids for every child).
- [ ] Add equivalent domain methods on child entities (`WorkExperience.CloneFor(resumeId)`, etc.) OR do the deep copy inline in `Resume.Duplicate()` — prefer per-child methods for testability.
- [ ] Create `Features/Resumes/DuplicateResume/DuplicateResumeEndpoint.cs`:
  - Loads full resume with all children.
  - Calls `resume.Duplicate()`.
  - Adds + saves.
  - Returns 201 with new id.
- [ ] Unit tests for `Resume.Duplicate` (independent object, all children copied, new ids, title suffix).
- [ ] Integration test for the endpoint.

**Files touched:** `Domain/Resume.cs` (+ children), `Features/Resumes/DuplicateResume/*`, test files.

**Acceptance criteria:**
- [ ] Duplicating a resume produces a brand-new resume with equivalent content and no shared references.
- [ ] All children are duplicated with fresh ids.

**Technical concepts:** Deep-copy semantics in domain methods, aggregate-root cloning.

**Suggested commit message:**
```
feat(resumes): add Duplicate domain method and endpoint
```

---

### Session S5.2 — Duplicate button in frontend

- **Objective:** UI affordance for duplication; redirect to the new resume's edit page after success.
- **Estimated duration:** 30 min
- **Prerequisites:** S5.1.

**Tasks:**

- [ ] Add `duplicateResume(id)` to `src/api/resumes.ts`.
- [ ] Add a "Duplicate" button on the resumes list (and optionally on the edit page).
- [ ] On click, call the API, then navigate to `/resumes/<new-id>/edit`.

**Acceptance criteria:**
- [ ] Clicking duplicate produces a copy and opens its edit page.

**Technical concepts:** react-router programmatic navigation.

**Suggested commit message:**
```
feat(frontend): add duplicate action in resumes list
```

---

# PHASE P6 — Docker Compose end-to-end

**Goal:** Both the backend and the frontend run inside Compose along with Postgres; developer can bring the whole stack up with one command.

---

### Session S6.1 — Backend Dockerfile

- **Objective:** Multi-stage Dockerfile that builds the API and produces a slim runtime image.
- **Estimated duration:** 60 min
- **Prerequisites:** P5 complete.

**Tasks:**

- [ ] Create `backend/src/CvBuilder.Api/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY backend/CvBuilder.slnx ./backend/
COPY backend/src/CvBuilder.Api/CvBuilder.Api.csproj ./backend/src/CvBuilder.Api/
COPY backend/tests/CvBuilder.Api.Tests/CvBuilder.Api.Tests.csproj ./backend/tests/CvBuilder.Api.Tests/
RUN dotnet restore ./backend/CvBuilder.slnx
COPY backend/ ./backend/
RUN dotnet publish ./backend/src/CvBuilder.Api/CvBuilder.Api.csproj -c Release -o /app /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:10.0-alpine AS runtime
WORKDIR /app
COPY --from=build /app ./
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "CvBuilder.Api.dll"]
```

- [ ] Add `.dockerignore` at repo root covering `bin/`, `obj/`, `node_modules/`, `.git/`, `.vs/`, `TestResults/`, `*.user`, `.env`.
- [ ] Build locally: `docker build -f backend/src/CvBuilder.Api/Dockerfile -t cvbuilder-api:dev .` (context = repo root).
- [ ] Run container manually (with env var pointing to host Postgres): verify `/health` responds.

**Files touched:** `backend/src/CvBuilder.Api/Dockerfile`, `.dockerignore`.

**Acceptance criteria:**
- [ ] `docker build` succeeds and produces an image.
- [ ] Image size is reasonable (< 300 MB with alpine).
- [ ] Container responds to `/health` when connected to Postgres.

**Technical concepts:** Multi-stage builds, layer caching with early `COPY *.csproj + restore`, alpine runtime, `ASPNETCORE_URLS`, `.dockerignore` for build-context size.

**Suggested commit message:**
```
chore(docker): add multi-stage backend Dockerfile
```

---

### Session S6.2 — Frontend Dockerfile and nginx

- **Objective:** Multi-stage frontend image — Node build → nginx serve.
- **Estimated duration:** 60 min
- **Prerequisites:** S6.1.

**Tasks:**

- [ ] Create `frontend/nginx.conf` serving `/usr/share/nginx/html` and proxying `/api` to `http://api:8080`.
- [ ] Create `frontend/Dockerfile`:

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

FROM nginx:1.27-alpine AS runtime
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

- [ ] Build locally and run attached to a Docker network with Postgres + API.

**Files touched:** `frontend/Dockerfile`, `frontend/nginx.conf`.

**Acceptance criteria:**
- [ ] `docker build` succeeds for frontend.
- [ ] Image serves the SPA and proxies `/api` requests to the API container.

**Technical concepts:** Static SPA hosting via nginx, reverse-proxy config, container DNS inside a Compose network.

**Suggested commit message:**
```
chore(docker): add multi-stage frontend Dockerfile with nginx
```

---

### Session S6.3 — Full compose.yaml

- **Objective:** Extend `compose.yaml` to include `api` and `frontend` services alongside `postgres`; introduce `profiles` to separate dev-only concerns.
- **Estimated duration:** 60 min
- **Prerequisites:** S6.2.

**Tasks:**

- [ ] Expand `compose.yaml` to include three services. Add:
  - `api`: builds from `backend/src/CvBuilder.Api/Dockerfile`, depends on `postgres` with `condition: service_healthy`, env `ConnectionStrings__Default=Host=postgres;...`, exposes port 8080 → host 5080.
  - `frontend`: builds from `frontend/Dockerfile`, depends on `api`, exposes port 80 → host 5173.
  - Networks (default is fine).
- [ ] Add migration-on-startup strategy: either (a) a short `migrate` service that runs `dotnet ef database update` and exits, or (b) call `db.Database.MigrateAsync()` in `Program.cs` behind an env flag. Go with (b) — simpler.
- [ ] Document `docker compose up --build` in README.
- [ ] Verify end-to-end: fresh clone, `docker compose up --build`, open `http://localhost:5173`.

**Files touched:** `compose.yaml`, `Program.cs` (migration hook), `README.md`.

**Acceptance criteria:**
- [ ] `docker compose up --build` brings up all three services; the browser loads the SPA and performs round-trip CRUD.
- [ ] Data persists across `docker compose down && docker compose up`.

**Technical concepts:** Service dependency ordering (healthcheck-based), container-to-container networking, runtime migration vs sidecar migration, Compose profiles.

**Suggested commit message:**
```
chore(docker): wire full compose with api, frontend, and postgres
```

---

# PHASE P7 — Frontend CI and documentation

**Goal:** The frontend has its own CI workflow; the README is a polished entry point; a deployment checklist exists. (Backend CI was introduced in S1.11.)

---

### Session S7.1 — Frontend CI workflow

- **Objective:** `frontend-ci.yml` runs `npm ci`, `lint`, `build` on push + PR.
- **Estimated duration:** 45 min
- **Prerequisites:** P6 complete. Backend CI from S1.11 is green.

**Tasks:**

- [ ] Create `.github/workflows/frontend-ci.yml`:

```yaml
name: frontend-ci
on:
  push:
    branches: [main]
    paths: ['frontend/**', '.github/workflows/frontend-ci.yml']
  pull_request:
    branches: [main]
    paths: ['frontend/**', '.github/workflows/frontend-ci.yml']
jobs:
  lint-build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 'lts/*'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run build
```

- [ ] Open a short-lived PR touching the frontend to confirm the workflow runs.
- [ ] Confirm it also runs on `main` push.

**Files touched:** `.github/workflows/frontend-ci.yml`.

**Acceptance criteria:**
- [ ] Workflow passes on main.
- [ ] Workflow runs on PR events touching `frontend/**`.

**Technical concepts:** Node setup action + npm cache keyed to lockfile, path-filtered triggers to avoid redundant runs, `working-directory` default.

**Suggested commit message:**
```
ci: add GitHub Actions workflow for frontend lint and build
```

---

### Session S7.2 — README polish and deployment checklist

- **Objective:** README is the single doorway for a newcomer. Add quickstart, architecture summary, and a deployment checklist (not an actual deployment).
- **Estimated duration:** 45 min
- **Prerequisites:** S7.1.

**Tasks:**

- [ ] Rewrite `README.md` with sections:
  - Overview (what it does).
  - Prerequisites (.NET 10, Node LTS, Docker).
  - Quickstart (local dev: `docker compose up -d postgres`, `dotnet run`, `npm run dev`).
  - One-command bring-up (`docker compose up --build`).
  - Project layout (one paragraph; link to spec).
  - Testing (`dotnet test`, `npm run lint && npm run build`).
  - Architecture summary (link to spec).
  - Status (MVP, no auth, not production-ready).
- [ ] Create `docs/deployment-checklist.md`:
  - Set `ASPNETCORE_ENVIRONMENT=Production`.
  - Supply `ConnectionStrings__Default` via secrets.
  - Rotate `POSTGRES_PASSWORD`.
  - Decide auth strategy (not in MVP — flag).
  - QuestPDF license considerations.
  - Volume backup plan.
  - Reverse proxy with TLS (Caddy / nginx / Traefik).
- [ ] Mark P7 (and overall MVP) complete in PROGRESS.md.

**Files touched:** `README.md`, `docs/deployment-checklist.md`, `PROGRESS.md`.

**Acceptance criteria:**
- [ ] A new contributor could read the README and run the app in < 5 minutes.

**Technical concepts:** Minimal docs that pay for themselves.

**Suggested commit message:**
```
docs: expand README and add deployment checklist
```

---

## Appendix A — PROGRESS.md Template

The file saved at `C:\Projects\Personal\cv-builder\PROGRESS.md` on S0.1 uses the content below. Keep it short; update after each session.

```markdown
# CV Builder — Progress Tracker

**Last updated:** YYYY-MM-DD
**Current phase:** P0 — Foundations
**Current session:** S0.1 — Repository and Git initialization
**Last session completed:** none

> Full plan: `docs/superpowers/plans/2026-10-08-cv-builder-mvp-implementation-plan.md`
> Spec: `docs/superpowers/specs/2026-10-08-cv-builder-mvp-architecture-design.md`

## Session Checklist

### P0 — Foundations
- [ ] S0.1 — Repository and Git initialization
- [ ] S0.2 — .NET solution and project scaffolding
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

- Architecture decisions log: see spec, section 15 (change history).
- Deferred micro-decisions: assertion library choice (Shouldly vs FluentAssertions) is pending; defaulting to Shouldly in S1.7 unless overridden.
```

---

## Appendix B — Deferred decisions log

Micro-decisions explicitly deferred by the user; capture them here when they fall:

| Decision | Pending since | Default on first use | Final choice | Session |
|---|---|---|---|---|
| Assertion library (Shouldly vs FluentAssertions) | spec §8 | Shouldly | — | S1.7 |

---

## Appendix C — Session execution workflow (recap)

Each session follows this flow verbatim (user requirement):

1. **Review state** — read `PROGRESS.md`, run `git log --oneline -5` and `git status`, note any open TODOs.
2. **Identify next task** — find the first unchecked session in `PROGRESS.md`.
3. **Explain** — in chat, summarize what we will do and why (1–2 paragraphs).
4. **Mini-plan** — expand the session's checkboxes into concrete TDD micro-steps (write test → run → implement → run) and clearly state the files that will change. If the mini-plan reveals the work cannot fit in 60 min, propose splitting into `Sx.y a / b` and get approval on the split before continuing.
5. **Wait for approval** — hard gate. Do not touch files.
6. **Implement** — only after explicit "yes".
7. **Build + test** — run the relevant commands and verify green.
8. **Summarize what shipped** — modified files, any surprises, any decisions logged.
9. **Update `PROGRESS.md`** — tick the completed session (or sub-session), update "Last updated", "Last session completed", and "Current session". This update is part of the *same* working tree change that will be committed next, so it ships in a single commit together with the session's code.
10. **Git review and commit cycle** (per user's git workflow):
    1. Run `git status` + `git diff --stat` to show what changed (including the `PROGRESS.md` edit from step 9).
    2. Confirm no secrets, no `.env`, no build artifacts (`bin/`, `obj/`, `dist/`, `node_modules/`, `TestResults/`) are staged.
    3. Propose a Conventional Commit message (format: `type(scope): short imperative summary`).
    4. **Wait for user approval** of the message.
    5. On approval: stage the specific files (not `git add .`), including `PROGRESS.md`, create the commit, then `git push origin main`.
    6. Confirm the commit is visible on GitHub (`gh repo view MarioSandovall/cv-builder --web` or paste URL).
11. **Recommend next session** — propose the next session's objective; do NOT auto-start it.

**Guardrails that always apply during the git cycle:**

- No `--force` pushes.
- No `--no-verify`, no hook bypasses.
- No amending commits that are already pushed.
- No commits of generated files; verify by inspecting `git diff --cached`.
- No empty or artificial commits (if the session produced no meaningful change, flag it instead of fabricating a commit).
