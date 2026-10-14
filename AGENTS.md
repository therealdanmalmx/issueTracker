# AGENTS.md

Guidance for AI agents working in this repository.

## Project

issueTracker: Blazor Web App (interactive server), .NET 10, EF Core + PostgreSQL (Npgsql), ASP.NET Core Identity.
Root namespace `issueTracker`. Existing scaffold lives in `Components/` (UI, Identity pages) and `Data/` (DbContext, migrations).

## Architecture

- **Vertical slice architecture.** Organize by feature (use case), not by technical layer. A slice is one use case (e.g. `CreateIssue`) and contains everything it needs end to end: command/query, handler, request, response, DTOs, validation and its Blazor UI. Do not split a slice across `Commands/`, `Queries/`, `Dtos/` or similar layer folders.
- `Features/` holds one folder per entity/area (e.g. `Features/Issues/`, `Features/Projects/`), and each contains one sub-folder per slice.
- Slices are independent: do not share handlers, DTOs or request/response types between slices. Duplicate small types rather than coupling slices. Only the entity (and genuinely cross-cutting code in `Common/`) is shared.
- **CQRS.** Writes are Commands, reads are Queries. A command never returns entity data beyond an id/result; a query never mutates state.
- **Custom mediator** (in `Common/Mediator/`). No MediatR or other mediator libraries.
- Handlers talk to `ApplicationDbContext` directly. Do not add repositories or service classes unless there is a concrete need; if you think one is needed, ask first.
- Blazor components send commands/queries through `IMediator`; they contain no data-access logic.

### Feature folder layout

```
Features/Issues/
  Domain/                        # entity + enums for the area, shared by its slices
    Issue.cs
  CreateIssue/                   # slice (write)
    CreateIssueCommand.cs        # implements ICommand<CreateIssueResponse>
    CreateIssueHandler.cs
    CreateIssueRequest.cs        # inbound model (form/API)
    CreateIssueResponse.cs       # outbound model
    CreateIssueValidator.cs      # only if validation is non-trivial
    CreateIssue.razor            # UI for this slice
  GetIssueById/                  # slice (read)
    GetIssueByIdQuery.cs
    GetIssueByIdHandler.cs
    GetIssueByIdResponse.cs
    IssueDetailDto.cs            # slice-specific DTO lives here, not in a shared Dtos/ folder
  ListIssues/
    ...
Features/Projects/
  ...
```

Handlers are named `<UseCase>Handler` (no `Command`/`Query` infix).

Shared infrastructure (mediator abstractions, behaviors) goes in `Common/`.

## Conventions

- **One type per file** (class, record, interface, enum); file name matches the type name.
- Commands, queries, requests, responses and DTOs each live in their own files, inside the slice folder that uses them.
- Every handler and mediator component is behind an interface (`ICommandHandler<,>`, `IQueryHandler<,>`, `IMediator`) and resolved via constructor dependency injection. Register handlers in DI by assembly scanning in `Program.cs` or a `Common` extension method.
- Prefer `record` for commands, queries, requests, responses and DTOs. Never expose EF entities outside the feature's handlers; map to DTOs.
- Use `async`/`await` with `CancellationToken` passed through to EF calls.
- Nullable reference types are enabled; respect them.
- Keep files short and focused. Add comments where they explain *why*, not *what*.

## Rules

- **No paid packages.** Free/open-source licenses only.
- **Do not add any NuGet package without asking the user first.**
- Use EF Core migrations for schema changes (`dotnet ef migrations add <Name> --output-dir Data/Migrations`); never edit generated migration or snapshot files by hand.
- Follow .NET and ASP.NET Core best practices (DI, async, validation at boundaries, no secrets in source; use user-secrets/config).
- Build before finishing: `dotnet build`.

## Deployment

Target is **Azure App Service**; `main` deploys automatically via GitHub Actions. Write code for production from the start — never assume localhost, a single instance, or a developer machine.

- **No secrets in `appsettings*.json` or any committed file.** Locally use .NET user secrets (`UserSecretsId` is already set); in production values come from App Service configuration. If you need a new setting, add the key with an empty/placeholder value or not at all, and tell the user what to set.
- Connection strings and all config are read through `IConfiguration` — never hard-coded, never a literal in code.
- **Do not call `Database.Migrate()` (or `EnsureCreated`) on startup.** Migrations are applied by the pipeline as an idempotent SQL script.
- Assume multiple instances. No shared state in memory — no static mutable caches, no in-process counters, no local file storage for anything that must persist. Use the database or a distributed store.
- Assume HTTPS everywhere: secure cookies, absolute URLs built from the request, no `http://` literals.
- Log with `ILogger<T>` and structured message templates (`_logger.LogInformation("Issue {IssueId} created", id)`). Never `Console.WriteLine`, never string interpolation in log messages.
- **Do not modify anything under `.github/workflows/` without asking the user first.**

## Commands

- Run: `dotnet run`
- Build: `dotnet build`
- Database: `docker compose up -d` (see `docker-compose.yml`)
