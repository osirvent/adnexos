# PocketBase Backend Guidelines

This backend is a PocketBase application extended with custom Go plugins. Use this file as a reference when porting the
business logic to another project.

## Application bootstrap
- `cmd/adnexos/main.go` configures a standard PocketBase instance, serves static assets, enables automigrations, and
  registers the custom plugin logic via `plugin.Register(app)`. 【F:backend/cmd/adnexos/main.go†L1-L36】
- The plugin registration is the main extension point. Any new project should call `plugin.Register` (or its equivalent)
after creating the PocketBase app and before `app.Start()`.

## Plugin structure
- Custom logic lives in `internal/plugin`. The package exposes `Register(app core.App) error` which wires up:
  - HTTP routes under `/api/collections/groups/...` that implement join, leave, and settle flows for group records.
  - Record-level hooks for `expenses`, `groups`, and `settings` collections (create, update, list, view) to enforce
    business rules and decorate responses.
  - A cron task that removes expired invites nightly.
  These registrations are centralized inside `plugin.Register` so a downstream project can copy the package and invoke it
  from its own `main.go`. 【F:backend/internal/plugin/plugin.go†L1-L45】

### Custom routes
- `groupJoinRoute`, `groupLeaveRoute`, and `groupSettleRoute` are registered as authenticated POST routes. They load the
  authenticated user from `RequestInfo`, fetch the target records, run custom validation, and persist updates through the
  PocketBase data access APIs. 【F:backend/internal/plugin/groups.go†L13-L121】
- The settle route converts group expenses into payments via the `service.Settle` helper before persisting them as records
  in the `payments` collection. 【F:backend/internal/plugin/groups.go†L63-L121】【F:backend/internal/service/settle.go†L1-L78】

### Record hooks
- `onExpensesBeforeCreate`/`onExpensesBeforeUpdate` ensure every expense member belongs to the owning group. 【F:backend/internal/plugin/expenses.go†L9-L45】
- `onGroupsBeforeUpdate`, `onGroupsView`, and `onGroupsList` protect group membership, compute per-user balances, and
  add computed data (`balance`, `costs`, `membersBalance`) to responses when requested via `fields` query parameters.
  【F:backend/internal/plugin/groups.go†L123-L221】
- `onSettingsCreate` prevents duplicate user settings records. 【F:backend/internal/plugin/settings.go†L10-L26】

### Cron jobs and maintenance
- `invitesRemove` runs daily (`0 1 * * *`) and deletes expired invite records directly via the database handle. Adapt the
  query or schedule when porting to projects with different retention rules. 【F:backend/internal/plugin/plugin.go†L27-L38】【F:backend/internal/plugin/invites.go†L1-L17】

## Shared services
- `internal/service/settle` converts expenses into payment transfers. It exposes a reusable `Settle` type that computes
  member balances and emits payment instructions. Copy or wrap this package when migrating the balance-settlement logic.
  【F:backend/internal/service/settle.go†L1-L78】

## Migration considerations
- This project relies on PocketBase automigrations (`migratecmd.Automigrate`). If you port the plugin to another project,
  ensure the target instance either enables automigration or ships the matching SQL migrations under `migrations/`.
