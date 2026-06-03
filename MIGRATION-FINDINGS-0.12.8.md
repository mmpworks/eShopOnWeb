# eShopOnWeb — Herald.OSS 0.12.8 migration findings (MEL provider swap)

**Project:** `dotnet-architecture/eShopOnWeb` (ASP.NET Core reference app)
**TFM:** net8.0
**Logger before:** Microsoft.Extensions.Logging (MEL) — default console provider, every log
call site is `ILogger<T>` / `app.Logger`
**Herald.OSS:** 0.12.8 (public nuget.org — no local-packages override)
**Date:** 2026-06-03 · Local only (committed on the fork branch, not pushed)

## The headline: provider swap, call sites unchanged

This is the cleanest MEL proof. The migration is a **provider swap with no shim**. We register
Herald's MEL provider so every existing `ILogger<T>` call flows through Herald.OSS — and **not one
of the 26 `ILogger` call sites changed**.

The entire migration is three bootstrap edits:

| File | Change |
|---|---|
| `Directory.Packages.props` | add `Herald.OSS` 0.12.8 package version |
| `src/Web/Web.csproj` | add `Herald.OSS` package reference |
| `src/Web/Program.cs` | swap the logging provider (5 lines) |

The Program.cs swap:

```csharp
builder.Logging.ClearProviders();
var heraldLog = QuickLogBuilder.Create().WithConsoleSink().BuildAndCommit();
builder.Logging.AddProvider(new HeraldLoggerProvider(heraldLog.Logger));
```

Before, this was a single line: `builder.Logging.AddConsole();`. Every downstream
`ILogger<T>` injection, every `app.Logger.LogInformation(...)`, every framework log
(`Microsoft.EntityFrameworkCore.*`, `Microsoft.Hosting.Lifetime`, `Microsoft.AspNetCore.*`)
keeps its exact call site and now renders through Herald.

`grep -rn "ILogger" src --include=*.cs` → 26 call sites, **all untouched**.
`git diff --stat` → only the 3 bootstrap files changed.

## Build (0.12.8, public nuget.org)

- Removed the `herald-local` local-packages source from `nuget.config` so the restore comes
  purely from `nuget.org` — a virgin user reproduces this exactly.
- Clean restore + Release build of `src/Web/Web.csproj`:
  - **Build succeeded. 0 errors.**
  - Resolved `Herald.OSS/0.12.8` (verified in `project.assets.json` and the global NuGet cache).
  - 5 warnings, all **pre-existing and unrelated to Herald**: transitive CVE notices on
    `Azure.Identity` 1.10.4 (NU1902) and `System.Text.Json` 8.0.3 (NU1903), plus one
    `SYSLIB0051` obsolete-serialization warning in the original app's
    `EmptyBasketOnCheckoutException`.

## Run (ran, not just compiled)

Started the app in Development with the app's built-in in-memory DB flag
(`UseOnlyInMemoryDatabase=true`), so it boots end-to-end with no SQL Server dependency, seeds the
catalog, and serves traffic. `ILogger` events flowed through Herald the whole time.

Startup + seeding (all rendered in Herald's console-sink format — `INF:2 <ISO-8601> <category> - <message>`):

```
INF:2 2026-06-03T20:35:08.93+00:00 Web - App created...
INF:2 2026-06-03T20:35:08.94+00:00 Web - Seeding Database...
INF:2 2026-06-03T20:35:09.46+00:00 Microsoft.EntityFrameworkCore.Update - Saved 5 entities to in-memory store.
INF:2 2026-06-03T20:35:09.72+00:00 Web - Adding Development middleware...
INF:2 2026-06-03T20:35:09.84+00:00 Web - LAUNCHING
INF:2 2026-06-03T20:35:09.89+00:00 Microsoft.Hosting.Lifetime - Now listening on: http://localhost:5199
INF:2 2026-06-03T20:35:09.89+00:00 Microsoft.Hosting.Lifetime - Application started. Press Ctrl+C to shut down.
```

Request path (full serve, not just boot):

- `GET /` → **HTTP 200** (homepage rendered)
- `GET /health` → 503 (expected: the `ApiHealthCheck` reaches the separate PublicApi service,
  which isn't running in this single-process run). Notably the health-check **failure** — the
  exception stack trace and the `Request finished ... GET /health - 503` request log — also
  rendered through Herald, including the structured `scope=RequestPath:/health RequestId:...`
  data. That is `ILogger<T>` error-path + request logging end-to-end through Herald.

The `INF:2 <ISO-8601> <category> - <message>` shape is Herald's console sink — not MEL's default
two-line `info: Category[EventId]` console formatter — which is the visible proof that events
routed through `HeraldLoggerProvider` rather than the default provider.

### Config fidelity holds

The app's `Logging:LogLevel` section (`Default: Warning`, `Microsoft`/`System: Warning`, with
`Information` allowed in Development) still governs what reaches Herald. MEL applies category-level
filtering at the `ILogger` factory layer ahead of the provider, so the existing appsettings
log-level config keeps working unchanged. No gap here.

## Gaps (honest)

1. **No sink gap for this project.** eShopOnWeb's Web host logged to console only
   (`AddConsole()`), and Herald has a console sink (`WithConsoleSink()`). This is the cleanest
   possible MEL proof — there is nothing Herald.Sinks is missing for this app. Stated plainly
   rather than inventing a gap.

2. **Scope note — PublicApi sidecar not migrated.** There is a second `AddConsole()` in
   `src/PublicApi/Program.cs:32`. The migration intentionally covered the Web front-end host only
   (the cited MEL proof), not the PublicApi sidecar. Migrating it would be the identical 3-edit
   provider swap; it was left out of scope rather than silently expanded.

3. **custom-sink-wanted: none.** This migration surfaced no sink or routing behavior that
   Herald.Sinks lacks. (Recorded explicitly per the showcase honesty rules — the "custom sink
   wanted" list is empty for eShopOnWeb.)

## Suggested validation

- Re-run on any future Herald.OSS bump: `dotnet restore --force --no-cache` then build
  `src/Web/Web.csproj` — the bootstrap is the only Herald surface, so a bump is a one-line version
  edit + rebuild.
- If the PublicApi sidecar is later brought in scope, apply the same 3-edit swap and re-run with
  both hosts to clear the `/health` 503.
