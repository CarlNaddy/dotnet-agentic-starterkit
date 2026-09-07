# Starting a new project from this template

This repo is a **reference app + a curated Claude Code setup**. It works as a
GitHub *template repository* — you get a full, running .NET 10 / Blazor /
MudBlazor / EF Core + PostgreSQL monolith, then rename it and (optionally) strip
the sample feature.

> A `dotnet new` template is the eventual goal (parity plan **P7.2**). Until then
> this is the template-repo route, verified end-to-end.

## Prerequisites

- .NET 10 SDK (`dotnet --version` → 10.0.x) — <https://dotnet.microsoft.com/download/dotnet/10.0>
- Docker (for local PostgreSQL) — <https://docs.docker.com/get-docker/>
- Git, plus **bash** to run the scripts — on Windows use **Git Bash**
- Node 18+ *(optional)* — only for OpenSpec (step 7) — <https://nodejs.org>
- `flyctl` *(optional)* — only to deploy (P5.3); `scripts/new-project.sh` runs
  `scripts/install-flyctl.sh` for you (rerunnable standalone), `scripts/preflight.sh`
  reports it — <https://fly.io/docs/flyctl/install/>

`scripts/new-project.sh` runs `scripts/preflight.sh` first and stops if a
required tool is missing; run `bash scripts/preflight.sh` yourself any time.

**A tool the script installs for you may not be on `PATH` until you open a new
shell** — sometimes not until you sign out of the OS or reboot, because a
running process (your terminal, its parent, an IDE) keeps the `PATH` it started
with. Currently this applies to `flyctl` (installed by `scripts/install-flyctl.sh`,
which modifies your user `PATH`). `scripts/new-project.sh` prints a reminder at
the end of its output when this is the case, and `scripts/preflight.sh` reports
"installed but not on this shell's PATH" until the new environment is picked up.

**On Windows, without Git Bash yet:** every script has a `.ps1` counterpart —
`scripts/preflight.ps1`, `scripts/new-project.ps1` — that runs from
PowerShell/CMD with no bash needed to get that far. Each one delegates to Git
Bash if it finds one (checked against Git's own install, not just anything
named `bash.exe` on PATH — Windows ships an unrelated WSL `bash.exe` stub that
runs a separate Linux toolchain and can't see your Windows-side tools), or
tells you to install [Git for Windows](https://git-scm.com/downloads/win) if
it doesn't. Windows users can use the `.ps1` commands throughout this guide in
place of the `bash ...` ones shown.

## One-time (maintainer of *this* repo)

GitHub → **Settings → General → check "Template repository"**. Already done here —
the repo shows a **Use this template** button.

---

## Step 1 — Create and clone

On GitHub click **Use this template → Create a new repository**, then:

```bash
git clone https://github.com/<you>/<new-repo>.git
cd <new-repo>
```

## Step 2 — Rename (scripted)

```bash
bash scripts/new-project.sh Contoso.Portal
```

**Windows without Git Bash yet:**

```powershell
powershell -File scripts/new-project.ps1 Contoso.Portal
```

Same script either way — `new-project.ps1` delegates to `new-project.sh` via
Git Bash (see the Prerequisites note above).

The working tree must be clean. Both `new-project.sh` and `remove-sample.sh`
refuse to run if this repo's `origin` remote is still the canonical
`github.com/CarlNaddy/dotnet-agentic-starterkit` — a project created via "Use this
template" always gets its own new `origin`, so this only ever fires if
you're accidentally in the template repo itself, not a project made from it
(bypass with `I_UNDERSTAND_THIS_IS_THE_TEMPLATE=1`, template-maintenance
only). The script:

- replaces the `dotnet-agentic-starterkit` identifier in every tracked text file
  (namespaces, usings, `_Imports.razor`, `.slnx`, launch profiles, …);
- renames `dotnet-agentic-starterkit.csproj` → `Contoso.Portal.csproj`,
  `dotnet-agentic-starterkit.slnx` → `Contoso.Portal.slnx`, and
  `tests/dotnet-agentic-starterkit.Tests/` → `tests/Contoso.Portal.Tests/`;
- regenerates `<UserSecretsId>`;
- resets `README.md` to a short project stub;
- deletes this repo's history docs (`rails-parity-plan.md`, `setup-log.md`);
- installs the Claude Code plugins/skills from `.claude/settings.json`
  (idempotent — `check-plugins.sh --fix`; skipped with a warning if the
  `claude` CLI isn't on PATH, rerun any time);
- prints the remaining manual steps.

**Keeps the `Listing` sample feature** — it's the worked pattern every P3/P4
doc points at (auth, jobs, caching, file storage), so the renamed project runs
and has something to look at immediately, the same reasoning `rails new
--minimal` vs. the full `rails new` weighs. Strip it down to an empty
skeleton any time, standalone:

```bash
bash scripts/remove-sample.sh
```

This deletes `Components/Pages/Listings/`, `Data/Listing.cs`, `Data/Seed/`,
`Features/Listings/`, `Endpoints/ListingsApiEndpoints.cs`,
`Features/Jobs/ListingJobs.cs`, and their tests; trims the Listing-specific
lines out of `AppDbContext.cs`, `Program.cs`, and the Listings nav link; and
**regenerates `Data/Migrations/` from scratch** as a single fresh
`InitialCreate` — the old migration history is entangled with `Listing` (one
migration both creates the generic `StoredFiles` table and alters `Listings`
in the same `Up()`), so migrations can't just be deleted piecemeal. Safe to
run before Step 3 — `dotnet ef migrations add` never opens a real connection,
so it works before the connection string is configured. What survives either
way regardless of the sample: ASP.NET Core Identity, background jobs
(Hangfire wiring), caching/rate limiting (first-party, minus the
`Listing`-specific policy), file storage (`IFileStore`), Data Protection key
persistence, health checks.

The project **compiles** either way. A skeleton has no entities, domain
migrations, or seed data — like `rails new`.

## Step 3 — Point at your database

Edit `compose.yaml` — set the Postgres identifiers (and the host port if 5432 is
taken):

```yaml
    environment:
      POSTGRES_DB: contosoportal
      POSTGRES_USER: contosoportal
      POSTGRES_PASSWORD: dev_only_change_me
    ports:
      - "5432:5432"
```

Store the dev connection string in user-secrets (run in the folder with
`Contoso.Portal.csproj`):

```bash
dotnet user-secrets set "ConnectionStrings:Default" \
  "Host=localhost;Port=5432;Database=contosoportal;Username=contosoportal;Password=dev_only_change_me"
```

## Step 4 — Bring it up

```bash
docker compose up -d db
dotnet format Contoso.Portal.slnx   # normalise line endings from the rename
dotnet build
dotnet test                         # xUnit v3 via MTP
dotnet watch run                    # http://localhost:5xxx  →  Home
```

Also run `dotnet run -- seed` (applies migrations and seeds 5 listings) — the
nav has a **Listings** page. Removed the sample in Step 2 instead? Use
`dotnet ef database update` there instead of `seed`.

## Step 5 — Make it yours (`CLAUDE.md`)

- Retitle; delete the status blockquote and the **"Reuse — starting a new
  project"** section.
- Keep: Stack table, **Data access**, **MudBlazor rules**, **Conventions**
  (incl. Tests and Localization).
- Drop the `Listing` / `dotnet run -- seed` mentions (skeleton), and fix the
  `docs/ef-migrations.md` link — its conventions still apply; the worked example
  just refers to the (removed) sample.

## Step 6 — Add your first model

Like `rails g model` / `rails g scaffold`:

```bash
# create Data/<Entity>.cs, add DbSet<Entity> to AppDbContext, then:
dotnet ef migrations add InitialCreate -o Data/Migrations
dotnet ef database update
```

Use `dotnet-data:create-datadriven-aspnetcore` + `mudblazor:mudblazor` for the
CRUD UI. (Still have the sample? The `Listing` feature is the worked pattern.)

## Step 7 — (optional) Spec-driven development with OpenSpec

To plan features as specs before implementing them:

```bash
bash scripts/setup-openspec.sh      # needs Node 18+; installs the CLI, runs `openspec init`
```

`openspec init` creates `openspec/` (`specs/`, `changes/`, `archive/`) and wires
slash commands into Claude Code. Workflow:

```
/opsx:explore <idea>       weigh options
/opsx:propose <feature>    -> openspec/changes/<id>/ : proposal.md, specs/, design.md, tasks.md
/opsx:apply                implement the tasks
/opsx:archive              fold the spec deltas into openspec/specs/, archive the change
```

Implement the feature with the bundled skills (CRUD via
`dotnet-data:create-datadriven-aspnetcore` + `mudblazor:mudblazor`, etc.),
following the `Listing` feature as the pattern. `openspec update` refreshes the
agent guidance after CLI upgrades.

## Step 8 — Commit

```bash
git rm scripts/new-project.sh scripts/new-project.ps1 \
  scripts/_guard-not-template.sh docs/new-project.md   # templating helpers
git add -A
git commit -m "Initialize from template"
```

Keep `scripts/remove-sample.sh` in this commit if you haven't decided yet
whether to strip the sample — remove it by hand once you have (whichever way
you decide).

---

## What carries over vs. what to strip

| Carries over | Removed by the script |
|---|---|
| `.claude/settings.json` — plugins & marketplaces | `docs/rails-parity-plan.md`, `docs/setup-log.md` |
| `CLAUDE.md`, `Directory.Build.props`, `Directory.Packages.props`, `.editorconfig`, `.gitattributes`, `global.json` | — |
| `compose.yaml` shape | — |
| `Program.cs` wiring, `Endpoints/`, `Localization/`, `Resources/`, `tests/<Name>.Tests/` harness | — |
| `fly.toml` / `.github/workflows/deploy.yml` (P5.3) — `fly.toml`'s `app` name is deliberately **not** rewritten (excluded from the identifier rewrite; Fly app names have different rules than a C# identifier). **This repo's own `fly.toml` currently names the maintainer's live instance (`carlnaddy-dotnet-agentic-starterkit`) — not a placeholder — so you must replace it with your own `fly apps create` name before deploying, or you'll be pointing at an app you don't own.** `scripts/new-project.sh` installs `flyctl` (best-effort); the rest of the account-side setup is in `docs/deployment.md`'s P5.3 section. | — |
| `scripts/preflight.sh`, `scripts/preflight.ps1`, `scripts/_find-git-bash.ps1`, `scripts/check-plugins.sh`, `scripts/install-flyctl.sh`, `scripts/setup-openspec.sh`, `docs/ef-migrations.md` | *(keep these)* |
| `Components/Pages/Listings/`, `Data/Listing.cs`, `Data/Seed/`, `Features/Listings/`, `Endpoints/ListingsApiEndpoints.cs`, `Features/Jobs/ListingJobs.cs` — kept by default | *removed by `scripts/remove-sample.sh`, run separately, any time* |

Remove by hand once set up: `scripts/new-project.sh`, `scripts/new-project.ps1`,
`scripts/_guard-not-template.sh`, `docs/new-project.md` — and
`scripts/remove-sample.sh` too, once you've either run it or decided to keep
the sample for good.
Rewrite: `CLAUDE.md` title + "Reuse" section, `README.md` stub.
