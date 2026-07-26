# Instance-Scoped Integration Manager Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (\`- [ ]\`) syntax for tracking.

**Goal:** Remove Job Framework's package-global integration state so every Kueue manager uses an independent integration registry and enabled-framework state.

**Architecture:** \`jobframework.IntegrationManager\` becomes the sole owner of registered callbacks, enabled frameworks, external frameworks, GVK lookup, and implicit-framework state. The \`jobs\` aggregation package creates and populates a new manager; each composition root passes that object through setup and runtime consumers instead of calling package-level Job Framework helpers.

**Tech Stack:** Go, controller-runtime, Kubernetes API machinery, Ginkgo/Gomega integration tests.

## Global Constraints

- Preserve existing callbacks, registration order, validation errors, feature gates, and user-facing configuration behavior.
- Do not retain a \`jobframework\` package-global manager or package-level forwarding API that reads one.
- Do not add an API type, configuration field, or release-note entry; this is cleanup work and retains \`release-note NONE\`.
- Use fresh managers in tests; do not restore mutable global integration state with cleanup functions.
- Before a push, run the closest available CI gates, fix actionable failures, and report environment-limited gates precisely.

---

## File Structure

| File group | Responsibility |
| --- | --- |
| \`pkg/controller/jobframework/integrationmanager.go\`, \`setup.go\` | Manager-owned registration, setup, lookup, and adapter methods. |
| \`pkg/controller/jobframework/reconciler.go\`, \`defaults.go\`, \`base_webhook.go\` | Carry the manager through options and generic runtime paths. |
| \`pkg/controller/jobs/**\` and \`pkg/controller/jobs/kubeflow/jobs/**\` | Register each built-in callback against a caller-owned manager. |
| \`cmd/kueue/main.go\`, \`pkg/config/validation.go\`, \`cmd/kueuectl/app/list/list_pods.go\` | Build and inject managers at application composition roots. |
| Job Framework, config, pod, kueuectl, and MultiKueue tests | Verify behavior and manager isolation. |

### Task 1: Make the integration manager instance-owned

**Files:**

- Modify: \`pkg/controller/jobframework/integrationmanager.go\`
- Modify: \`pkg/controller/jobframework/setup.go\`
- Modify: \`pkg/controller/jobframework/integrationmanager_test.go\`
- Modify: \`pkg/controller/jobframework/setup_test.go\`

**Interfaces:**

- Produces: \`type IntegrationManager struct\`, \`func NewIntegrationManager() *IntegrationManager\`, and receiver methods \`RegisterIntegration\`, \`RegisterExternalJobType\`, \`ForEachIntegration\`, \`GetIntegration\`, \`GetIntegrationByGVK\`, \`GetIntegrationsList\`, \`GetMultiKueueAdapters\`, \`SetupControllers\`, and \`SetupIndexes\`.
- Consumes: existing \`IntegrationCallbacks\`, \`Option\`, and controller-runtime manager interfaces.

- [ ] **Step 1: Write the failing isolation tests**

Add two fresh \`IntegrationManager\` instances to \`TestGetJobTypeForOwner\` and the implicit-framework tests. Register equivalent callbacks on both, enable a framework or collect implicit frameworks on only one, then assert the other instance has no enabled owner and no implicit GVK result.

~~~~go
first := NewIntegrationManager()
second := NewIntegrationManager()
mustRegister(t, first, "batch/job", jobCallbacks)
mustRegister(t, second, "batch/job", jobCallbacks)
first.enableIntegration("batch/job")

if second.IsOwnerManagedByKueueForObject(podWithJobOwner) {
	t.Fatal("second manager observed first manager's enabled integration")
}
~~~~

- [ ] **Step 2: Run the focused tests to verify the singleton-dependent behavior fails**

Run: \`go test ./pkg/controller/jobframework -run 'Test(GetJobTypeForOwner|ImplicitlyEnabledIntegrations)$' -count=1\`

Expected: the new isolation test cannot compile until \`IntegrationManager\` exposes the receiver API, or fails while the singleton is still consulted.

- [ ] **Step 3: Implement the receiver API and remove the singleton**

Rename \`integrationManager\` to exported \`IntegrationManager\`, add \`NewIntegrationManager\`, and convert every former forwarding function to a method. Remove \`var manager integrationManager\`. Change \`SetupControllers\` and \`SetupIndexes\` to receiver methods; their deferred API setup closures must keep using the same receiver. Update every internal \`manager\` access, including external registration and implicit-framework collection, to \`m\`.

~~~~go
func NewIntegrationManager() *IntegrationManager {
	return &IntegrationManager{}
}

func (m *IntegrationManager) SetupControllers(ctx context.Context, mgr ctrl.Manager, log logr.Logger, opts ...Option) error {
	return m.setupControllers(ctx, mgr, log, opts...)
}
~~~~

- [ ] **Step 4: Update unit tests to construct managers explicitly**

Replace calls to package-level registration and test-only global enable helpers with a local manager in each test. Preserve table cases for duplicate registrations, mandatory fields, external framework parsing, dependencies, ordering, and GVK mapping.

- [ ] **Step 5: Run the Job Framework unit tests**

Run: \`go test ./pkg/controller/jobframework -count=1\`

Expected: PASS.

- [ ] **Step 6: Commit the manager API slice**

~~~~bash
git add pkg/controller/jobframework/integrationmanager.go pkg/controller/jobframework/setup.go pkg/controller/jobframework/integrationmanager_test.go pkg/controller/jobframework/setup_test.go
git commit -m "refactor: scope integration manager state"
~~~~

### Task 2: Make built-in registration explicit

**Files:**

- Modify: \`pkg/controller/jobs/jobs.go\`
- Modify: \`pkg/controller/jobs/kubeflow/jobs/jobs.go\`
- Modify: \`pkg/controller/jobs/{appwrapper,deployment,job,jobset,leaderworkerset,mpijob,pod,raycluster,rayjob,rayservice,sparkapplication,statefulset,trainjob}/*_controller.go\`
- Modify: \`pkg/controller/jobs/kubeflow/jobs/{jaxjob,paddlejob,pytorchjob,tfjob,xgboostjob}/*_controller.go\`
- Create: \`pkg/controller/jobs/jobs_test.go\`

**Interfaces:**

- Produces: \`func NewIntegrationManager() *jobframework.IntegrationManager\` in package \`jobs\`.
- Produces: \`func RegisterIntegration(m *jobframework.IntegrationManager) error\` in each leaf integration package and \`func RegisterIntegrations(m *jobframework.IntegrationManager) error\` in each aggregation package.
- Consumes: \`(*jobframework.IntegrationManager).RegisterIntegration\` from Task 1.

- [ ] **Step 1: Write a failing aggregation test**

Create \`TestNewIntegrationManager\` that calls \`NewIntegrationManager\` twice, compares each list to the expected sorted built-in framework names, and verifies enabling an integration on one manager does not change owner lookup on the other.

~~~~go
first := NewIntegrationManager()
second := NewIntegrationManager()
if diff := cmp.Diff(first.GetIntegrationsList(), second.GetIntegrationsList()); diff != "" {
	t.Fatalf("built-in registrations differ (-first +second):\n%s", diff)
}
~~~~

- [ ] **Step 2: Run the aggregation test to verify it fails**

Run: \`go test ./pkg/controller/jobs -run '^TestNewIntegrationManager$' -count=1\`

Expected: FAIL because \`jobs.NewIntegrationManager\` does not exist.

- [ ] **Step 3: Convert every leaf \`init\` function into registration against a supplied manager**

For every controller file listed above, replace \`init\` plus \`utilruntime.Must(jobframework.RegisterIntegration(...))\` with:

~~~~go
func RegisterIntegration(m *jobframework.IntegrationManager) error {
	return m.RegisterIntegration(FrameworkName, jobframework.IntegrationCallbacks{
	})
}
~~~~

Move the exact \`IntegrationCallbacks\` literal from that package's current \`init\`
body into this method unchanged. Remove imports made unused by deleting
\`utilruntime.Must\`; retain all callback values and their order.

- [ ] **Step 4: Implement the two aggregation functions**

Import each leaf package by name, call its registration function in the existing \`jobs.go\` declaration order, and return the first error. The top-level package calls the Kubeflow aggregation in the same sequence as today. Its constructor makes a fresh Job Framework manager, calls \`RegisterIntegrations\`, and panics with \`utilruntime.Must\` only if built-in source code violates its static registration contract.

~~~~go
func NewIntegrationManager() *jobframework.IntegrationManager {
	m := jobframework.NewIntegrationManager()
	utilruntime.Must(RegisterIntegrations(m))
	return m
}
~~~~

- [ ] **Step 5: Run registration and dependent compilation tests**

Run: \`go test ./pkg/controller/jobs ./pkg/controller/jobs/kubeflow/jobs -count=1\`

Expected: PASS.

- [ ] **Step 6: Commit explicit registrations**

~~~~bash
git add pkg/controller/jobs
git commit -m "refactor: register job integrations explicitly"
~~~~

### Task 3: Thread the manager through Job Framework runtime paths

**Files:**

- Modify: \`pkg/controller/jobframework/reconciler.go\`
- Modify: \`pkg/controller/jobframework/defaults.go\`
- Modify: \`pkg/controller/jobframework/base_webhook.go\`
- Modify: \`pkg/controller/jobs/pod/pod_controller.go\`
- Modify: \`pkg/controller/jobs/pod/pod_webhook.go\`
- Modify: \`pkg/controller/jobframework/{defaults,reconciler,validation}_test.go\`
- Modify: \`pkg/controller/jobs/pod/{pod_controller,pod_webhook}_test.go\`

**Interfaces:**

- Produces: \`func WithIntegrationManager(m *IntegrationManager) Option\` and \`Options.IntegrationManager\`.
- Consumes: the instance created by \`jobs.NewIntegrationManager\`.

- [ ] **Step 1: Write failing option-propagation tests**

Extend \`TestProcessOptions\` to require an \`IntegrationManager\` pointer round-trip. Add a pod/defaulting test that enables a parent integration only on the injected manager and asserts a managed-parent decision uses that manager.

~~~~go
m := jobs.NewIntegrationManager()
opts := ProcessOptions(WithIntegrationManager(m))
if opts.IntegrationManager != m {
	t.Fatal("integration manager was not preserved in options")
}
~~~~

- [ ] **Step 2: Run the focused option and pod tests to verify they fail**

Run: \`go test ./pkg/controller/jobframework ./pkg/controller/jobs/pod -run 'Test(ProcessOptions|IsPodOwnerManagedByQueue|Default)$' -count=1\`

Expected: FAIL because \`WithIntegrationManager\` and receiver-based calls do not exist.

- [ ] **Step 3: Carry the manager through generic reconcilers and webhooks**

Add the option, store the manager on \`JobReconciler\` and \`BaseWebhook\`, and replace global owner/defaulting calls with receiver methods. Update \`FindAncestorJobManagedByKueue\`, \`ApplyDefaultLocalQueue\`, and \`ApplyDefaultWorkloadPriorityClass\` to receive or call the instance manager. Production composition must always supply a manager.

- [ ] **Step 4: Update the pod integration**

Store the option manager on \`PodWebhook\` and the pod representation/reconciler path that evaluates \`Skip\`. Use it for ancestor lookup, priority defaulting, managed-owner warnings, and implicit-framework checks. Update test constructors to pass a new manager rather than altering a shared manager.

- [ ] **Step 5: Run Job Framework and pod tests**

Run: \`go test ./pkg/controller/jobframework ./pkg/controller/jobs/pod -count=1\`

Expected: PASS.

- [ ] **Step 6: Commit runtime dependency injection**

~~~~bash
git add pkg/controller/jobframework pkg/controller/jobs/pod
git commit -m "refactor: inject integration manager into job runtime"
~~~~

### Task 4: Wire composition roots and validation

**Files:**

- Modify: \`cmd/kueue/main.go\`
- Modify: \`pkg/config/validation.go\`
- Modify: \`pkg/config/validation_test.go\`
- Modify: \`cmd/kueuectl/app/list/list_pods.go\`
- Modify: \`cmd/kueuectl/app/list/list_pods_test.go\`
- Modify: direct callers of \`setupControllers\`, \`setupIndexes\`, and \`config.Validate\` found by compilation.

**Interfaces:**

- Consumes: \`jobs.NewIntegrationManager\`, \`*jobframework.IntegrationManager\`, and \`jobframework.WithIntegrationManager\`.
- Produces: manager-aware Kueue setup, configuration validation, and kueuectl GVK lookup.

- [ ] **Step 1: Write failing composition tests**

Update config validation test setup to pass a built-in manager and add a \`list_pods\` test whose \`PodOptions\` contains a deliberately isolated manager. Verify a known built-in GVK still resolves its pod selector.

- [ ] **Step 2: Run config and list tests to verify they fail at the new signatures**

Run: \`go test ./pkg/config ./cmd/kueuectl/app/list -count=1\`

Expected: FAIL until callers provide an integration manager.

- [ ] **Step 3: Create and inject the manager in \`cmd/kueue\`**

Create one built-in manager before integration callbacks add their types to the scheme. Pass it to \`config.Validate\`, the manager-owned \`SetupIndexes\` and \`SetupControllers\`, and \`GetMultiKueueAdapters\`. Add \`WithIntegrationManager\` to the Job Framework options used for controller setup.

- [ ] **Step 4: Make validation and kueuectl explicit consumers**

Add the manager argument to \`config.Validate\` and its internal integration validation helpers, then replace all global lookups with receiver calls. Store a manager in \`PodOptions\`, initialize it in the command constructor with \`jobs.NewIntegrationManager\`, and use it in \`getPodLabelSelector\`.

- [ ] **Step 5: Run composition tests**

Run: \`go test ./cmd/kueue ./pkg/config ./cmd/kueuectl/app/list -count=1\`

Expected: PASS.

- [ ] **Step 6: Commit composition wiring**

~~~~bash
git add cmd/kueue/main.go pkg/config/validation.go pkg/config/validation_test.go cmd/kueuectl/app/list
git commit -m "refactor: wire integration managers at composition roots"
~~~~

### Task 5: Prove MultiKueue manager isolation and validate the full change

**Files:**

- Modify: \`test/integration/multikueue/suite_test.go\`
- Modify: \`test/integration/multikueue/setup_test.go\` if its worker setup constructs adapters or Job Framework options.
- Modify: only the integration fixtures required to start two manager instances with distinct enabled frameworks.

**Interfaces:**

- Consumes: per-worker \`*jobframework.IntegrationManager\` and its \`GetMultiKueueAdapters\` and setup methods.
- Produces: a regression that demonstrates no cross-worker integration state leakage.

- [ ] **Step 1: Add a failing two-manager regression scenario**

Build two worker managers with fresh \`jobs.NewIntegrationManager\` instances. Configure distinct enabled integration lists, start the first worker, record its managed-owner/defaulting result, start the second worker, and assert the first result is unchanged. Keep cleanup local to the test cluster objects, not framework state.

- [ ] **Step 2: Run only the new MultiKueue spec to verify it exposes the leak before the wiring is complete**

Run: \`go test ./test/integration/multikueue -run '^TestMultiKueue$' -ginkgo.focus='integration manager isolation' -count=1\`

Expected: the spec fails against global state and passes only when each worker receives its own manager.

- [ ] **Step 3: Update shared MultiKueue setup**

Pass the worker-specific manager to adapter collection and any Job Framework setup paths. Do not construct adapters through a package-global lookup.

- [ ] **Step 4: Run focused regression and affected suites**

Run: \`go test ./pkg/controller/jobframework ./pkg/controller/jobs ./pkg/controller/jobs/pod ./pkg/config ./cmd/kueue ./cmd/kueuectl/app/list -count=1\`

Run: \`go test ./test/integration/multikueue -run '^TestMultiKueue$' -count=1\`

Expected: PASS, subject to local envtest/toolchain prerequisites.

- [ ] **Step 5: Run the closest available repository validation**

Run: \`make verify\`

Run: \`make verify-ci-lint\`

Expected: PASS. If the Windows checkout lacks required generation, linters, WSL tooling, or envtest binaries, record the exact command and failure separately from code failures.

- [ ] **Step 6: Commit the regression test and review the final diff**

~~~~bash
git add test/integration/multikueue
git commit -m "test: cover integration manager isolation"
git diff origin/main...HEAD --check
git status --short
~~~~

## Self-Review

Spec coverage: Task 1 removes the singleton and makes its state instance-owned. Task 2 eliminates \`init\` registration into global state. Task 3 propagates the instance through runtime behavior. Task 4 covers all command and validation composition roots. Task 5 proves the MultiKueue regression and defines the validation sequence.

No placeholders: the plan names every registration group, concrete methods, focused tests, test commands, and commits.

Type consistency: every consumer accepts \`*jobframework.IntegrationManager\`; \`jobs.NewIntegrationManager\` is the only built-in factory; runtime consumers receive it through \`jobframework.WithIntegrationManager\`.
