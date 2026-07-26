# Instance-Scoped Integration Manager Design

## Goal

Eliminate the package-global Job Framework integration manager so that every
Kueue manager running in one process owns independent integration state. This
prevents a manager configured for one set of integrations from changing
another manager's behavior in MultiKueue integration tests.

## Problem

`pkg/controller/jobframework/integrationmanager.go` currently declares one
package-global `integrationManager`. Its state includes the registered
callbacks, enabled integrations, external integrations, GVK lookup map, and
implicitly enabled integrations. `SetupControllers` mutates that singleton by
recording enabled and external integrations.

The global state is observed by controller setup, index setup, owner
resolution, defaulting, pod reconciliation, configuration validation,
MultiKueue adapter construction, and `kueuectl list pods`. Consequently, two
controller managers created in a single binary can see each other's enabled
integration state.

## Design

### Integration-manager API

Export `jobframework.IntegrationManager` and construct it explicitly. It owns
all integration maps, the ordered integration list, GVK lookup, external
frameworks, enabled integrations, and implicitly enabled integrations. The
package-global `manager` variable and package-level forwarding functions are
removed.

The manager exposes methods for registration, scheme iteration, controller and
index setup, configuration lookup, MultiKueue adapter collection, owner
resolution, and implicit-integration lookup. Methods that currently depend on
the enabled set use the receiver, so an integration enabled by one manager is
not visible to another.

### Registration

Built-in job integrations no longer register through package `init()` into a
singleton. Each job package exposes registration against a supplied
`*jobframework.IntegrationManager`. `pkg/controller/jobs` imports those job
packages normally and provides the single built-in registration entry point.

`jobs.NewIntegrationManager()` creates a fresh framework manager and registers
all built-in integrations. A caller needing two managers calls it twice; no
maps or enabled sets are shared between the results.

### Composition and propagation

`cmd/kueue` creates one manager before adding integration types to its scheme,
then passes that manager through configuration validation, index setup,
controller setup, and MultiKueue adapter construction. The command may retain
its own process-lifetime reference because it creates one controller manager;
the Job Framework package itself retains none.

`cmd/kueuectl` creates an independent built-in manager for the `list pods`
GVK lookup. `pkg/config.Validate` receives the manager needed for built-in
framework and adapter validation rather than consulting a global registry.

Add `WithIntegrationManager` to Job Framework options. Reconcilers and
webhooks retain the supplied manager and use it for owner traversal,
queue/default priority defaulting, and implicit-framework behavior. This keeps
all runtime reads on the same manager instance that performed setup.

### Compatibility and error behavior

Registration keeps the existing duplicate-name and mandatory-callback errors.
The external-framework parser and adapter-collision checks retain their current
error messages and validation semantics. Built-in registrations preserve their
existing order and callbacks; only ownership changes.

No feature gate, API type, configuration field, or user-facing behavior is
introduced. The change is a cleanup and should retain `release-note NONE`.

## File-Level Scope

Primary changes:

- `pkg/controller/jobframework/integrationmanager.go`: instance API and
  receiver methods; remove the global singleton.
- `pkg/controller/jobframework/setup.go`, `reconciler.go`, `defaults.go`, and
  option definitions: pass and consume the instance manager.
- `pkg/controller/jobs/jobs.go` and every built-in integration package:
  explicit registration instead of `init()` mutation.
- `cmd/kueue/main.go`: construct, register, and inject the manager into all
  composition paths.
- `pkg/config/validation.go`: accept the manager for integration validation.
- `cmd/kueuectl/app/list/list_pods.go`: use a command-owned manager for GVK
  lookup.
- affected unit and integration tests, especially
  `pkg/controller/jobframework/integrationmanager_test.go` and
  `test/integration/multikueue` setup.

## Testing Strategy

1. Convert existing integration-manager tests to use freshly constructed
   managers. Cover registration validation, deterministic framework listing,
   external-framework registration, dependencies, GVK lookup, and implicit
   integration collection.
2. Add isolation tests that construct two managers, enable or configure
   distinct integrations, and assert owner resolution and implicitly enabled
   framework lookup on one instance do not change after configuring the other.
3. Update job-framework, configuration, `kueuectl`, and pod
   controller/webhook tests to inject the appropriate manager rather than use
   test helpers that mutate global state.
4. Add a MultiKueue regression scenario, patterned after PR #13190's
   two-manager formatter-isolation test: configure two in-process managers
   with different enabled integrations and prove the first manager's
   defaulting/owner-management result remains unchanged after the second
   starts.

## Validation

Run focused Job Framework, configuration, kueuectl, and MultiKueue tests first.
Before any branch push, run the closest repository CI gates available, fix
actionable failures, and report any environment-limited gates separately from
application failures.
