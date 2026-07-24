# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

OrderHub is an internal company order management system: staff can create/view orders and manage
products and customers. It's a small single-tenant app (single SQL Server database) — don't apply
microservice/multi-tenant/high-concurrency patterns here.

This repo (`training-repo/`) is the app itself, nested inside a training repo. The parent directory
(`../documents/`) holds the training curriculum (`README.md`, `PROCESS.md`, `activities/`) — that
content is about the training exercises, not the app, and is generally not relevant to app changes.

## Tech stack

- .NET 8 / ASP.NET Core MVC (Razor Views + Bootstrap 5, no CDN — all front-end assets are local files)
- EF Core 8 + SQL Server (local instance, no Docker)
- Tests: xUnit with EF Core InMemory (no SQL Server needed to run tests)

## Common commands

```powershell
dotnet build                             # build the solution
dotnet test                              # run all tests
dotnet run --project src/OrderHub.Web    # run the site (http://localhost:5150)
```

Run a single test:

```powershell
dotnet test --filter "FullyQualifiedName~OrderServicePricingTests"
```

Reset the local dev database back to seed data:

```powershell
dotnet ef database drop -f -p src/OrderHub.Infrastructure -s src/OrderHub.Web
dotnet run --project src/OrderHub.Web   # re-runs migrate + seed on startup
```

The app auto-applies EF Core migrations and seeds data (20 customers, 50 products, 200 orders,
fixed random seed) on every startup — see `Program.cs` and `DbSeeder`.

## Architecture and conventions

Three layers, one-way dependency: `OrderHub.Web` → `OrderHub.Core` → `OrderHub.Infrastructure`
(Web references both Core and Infrastructure; Infrastructure implements Core's interfaces).

- `src/OrderHub.Web` — Controllers, ViewModels, Views. Wiring/display only.
- `src/OrderHub.Core` — Domain models, service interfaces, and all business logic
  (discounts, stock, status transitions).
- `src/OrderHub.Infrastructure` — EF Core `DbContext`, repositories, migrations, seed data.
- `tests/OrderHub.Tests` — xUnit, EF Core InMemory via `TestSetup` helpers.

Conventions to follow when adding/changing code:

- Controllers stay thin — they call a service and map the result to a ViewModel. No business logic
  or EF Core queries in a controller.
- Business logic lives in `OrderHub.Core/Services/*Service.cs`, exposed via an interface
  (`ICustomerService`, `IProductService`, `IOrderService`) and injected.
- Only repositories (`OrderHub.Infrastructure/Repositories`) touch `DbContext`. Services/controllers
  never use EF Core directly.
- Services return `ServiceResult<T>` (`OrderHub.Core/Common/ServiceResult.cs`) to express expected
  failures (e.g. "customer not found", "insufficient stock") — don't throw exceptions for those.
  Paged data uses `PagedResult<T>` (`OrderHub.Core/Common/PagedResult.cs`).
- Views bind to a ViewModel (`OrderHub.Web/ViewModels`), never a domain model directly; mapping is
  written by hand in the controller.
- User input is validated with DataAnnotations + `ModelState` — invalid input must never produce a
  500, it should redisplay the form with validation errors.
- Money is always `decimal`. Order pricing/discount logic belongs in `OrderService`
  (`CalculateSubtotal`/`CalculateTotal`/`GetDiscountRate`) — don't recompute discounts elsewhere.
- Reference implementations for a new feature: `ProductsController.cs` for a thin controller,
  `ProductService.cs`/`OrderService.cs` for the service layer, `OrderRepository.cs` for a repository
  with EF `Include`/paging.

## Important / dangerous files

- `src/OrderHub.Infrastructure/Migrations/**` — EF Core migration history, don't hand-edit.
- `src/OrderHub.Web/appsettings.json` / `appsettings.Development.json` — connection strings and
  config; confirm before changing.

## Don'ts

- Don't add a new NuGet package without checking first.
- Don't touch `DbContext` directly from a controller or service — go through a repository.
- Don't refactor unrelated code while working on a specific fix/feature.
- Don't read or write secrets (`*.pfx`, `appsettings.Production.json`, user-secrets).
