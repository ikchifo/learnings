# Kueue MultiKueue + TAS architecture and testing reference

A reference document capturing architectural knowledge and testing
patterns from the Kueue project, specifically around Topology-Aware
Scheduling (TAS) and MultiKueue.

---

## Kueue core concepts

Kueue is a Kubernetes-native job queueing system. It manages the
admission of batch workloads (Jobs, JobSets, RayClusters, etc.) into
a cluster by controlling when they can consume resources. Kueue does
not replace the scheduler; it gates job start via quota management.

Key objects:

- **Workload**: the kueue-internal representation of a batch job.
  Created automatically when an annotated Job/JobSet is submitted.
  Contains `PodSets` describing resource needs.
- **ClusterQueue**: cluster-scoped quota pool. Holds `ResourceGroups`
  (a list of ResourceFlavors and their quotas). Workloads are admitted
  from ClusterQueues.
- **LocalQueue**: namespace-scoped pointer to a ClusterQueue. Users
  submit jobs to a LocalQueue via the `kueue.x-k8s.io/queue-name`
  label.
- **ResourceFlavor**: names a class of nodes (e.g. on-demand, spot,
  GPU type) and carries `nodeLabels` used for node affinity injection.
  When `spec.topologyName` is set the flavor becomes TAS-enabled.
- **AdmissionCheck**: a gate on ClusterQueue admission. A Workload
  must pass all admission checks on a ClusterQueue before it is
  considered admitted. MultiKueue uses an AdmissionCheck to coordinate
  cross-cluster dispatch.

---

## Topology-Aware Scheduling (TAS)

TAS makes Kueue aware of the physical topology of the cluster (e.g.
which nodes share a rack or a network block) so it can fit PodSets
onto nodes within a common topology domain. This improves locality for
tightly-coupled workloads that need low-latency pod-to-pod
communication.

### The three required objects

1. **Topology CR** (`kueue.x-k8s.io/v1beta2, kind: Topology`): defines
   the hierarchy of topology levels as node label keys, from coarsest
   to finest. Example with three levels:

   ```yaml
   apiVersion: kueue.x-k8s.io/v1beta2
   kind: Topology
   metadata:
     name: default
   spec:
     levels:
     - nodeLabel: cloud.provider.com/topology-block
     - nodeLabel: cloud.provider.com/topology-rack
     - nodeLabel: kubernetes.io/hostname
   ```

   The label `kubernetes.io/hostname` must appear at the lowest (last)
   level if it is used at all.

2. **ResourceFlavor with `spec.topologyName`**: links the flavor to a
   Topology CR. At least one `nodeLabel` must also be set. Example:

   ```yaml
   apiVersion: kueue.x-k8s.io/v1beta2
   kind: ResourceFlavor
   metadata:
     name: tas-flavor
   spec:
     nodeLabels:
       cloud.provider.com/node-group: tas-node
     topologyName: default
   ```

3. **Pod annotations on the job/job template**: tell Kueue what
   topology constraint the PodSet has.

### Topology annotations

All constants live in `apis/kueue/v1beta2/topology_types.go` and use
the group `kueue.x-k8s.io`:

- `kueue.x-k8s.io/podset-required-topology`: all pods must be placed
  within a single domain at the named topology level. Admission fails
  if no domain can fit the entire PodSet.
- `kueue.x-k8s.io/podset-preferred-topology`: Kueue tries to fit all
  pods into one domain at the named level, then walks up the hierarchy
  if it cannot. If no single domain at any level fits, pods are spread
  across domains.
- `kueue.x-k8s.io/podset-unconstrained-topology`: no placement
  constraint. Kueue only checks that enough aggregate capacity exists.
  Useful for PodSets that do not need locality but still benefit from
  Kueue's capacity tracking.

In Go test code these constants are referenced as:

```go
kueue.PodSetRequiredTopologyAnnotation   // "kueue.x-k8s.io/podset-required-topology"
kueue.PodSetPreferredTopologyAnnotation  // "kueue.x-k8s.io/podset-preferred-topology"
kueue.PodSetUnconstrainedTopologyAnnotation
```

The import path for the package that defines these is
`sigs.k8s.io/kueue/apis/kueue/v1beta2`.

### Activating TAS

TAS is activated purely by setting `spec.topologyName` on a
ResourceFlavor. No feature gate flag is required. The Topology CR
named by `topologyName` must exist in the cluster before the
ResourceFlavor is created (or the flavor will not become active for
TAS).

### Topology levels in practice

A common three-level hierarchy:

```
block  →  rack  →  hostname
```

where `block` and `rack` use cloud-provider node labels and `hostname`
maps to the standard `kubernetes.io/hostname` label.

For simple single-machine or low-node-count setups, a one-level
topology with only `kubernetes.io/hostname` is sufficient.

Default label constants in `pkg/util/testing/defaults.go`:

```go
DefaultRackTopologyLevel  = "cloud.provider.com/topology-rack"
DefaultBlockTopologyLevel = "cloud.provider.com/topology-block"
```

Test wrapper helpers in `pkg/util/testing/v1beta2/wrappers.go`:

```go
MakeDefaultOneLevelTopology(name)   // hostname only
MakeDefaultTwoLevelTopology(name)   // block + rack
MakeDefaultThreeLevelTopology(name) // block + rack + hostname
```

---

## MultiKueue

MultiKueue implements a manager-worker multi-cluster dispatch pattern.
A manager cluster holds the authoritative Workload and dispatches the
actual job execution to one of several worker clusters.

### How dispatch works

1. A user submits a Job to the manager cluster referencing a
   LocalQueue backed by a ClusterQueue that has a MultiKueue
   AdmissionCheck.
2. Kueue on the manager creates a Workload object.
3. The MultiKueue controller copies the Job (and an associated
   Workload) to each registered worker cluster.
4. Each worker's local Kueue admits (or rejects) the Workload
   according to its own ClusterQueue quota.
5. The first worker that admits the Workload wins. MultiKueue marks
   the AdmissionCheck on the manager Workload as `Ready` with a
   message indicating the winning worker's name.
6. The manager deletes the Job copies on the losing workers.
7. Job status and Workload finish status flow back from the winning
   worker to the manager.

### Key objects

- **MultiKueueCluster**: points to a kubeconfig secret for a specific
  worker cluster.
- **MultiKueueConfig**: lists the MultiKueueCluster names that form a
  dispatch group.
- **AdmissionCheck** with
  `spec.controllerName: kueue.x-k8s.io/multikueue` and
  `spec.parameters` referencing a MultiKueueConfig: the gate on the
  manager ClusterQueue.

The controller name constant is:

```go
kueue.MultiKueueControllerName = "kueue.x-k8s.io/multikueue"
```

### RBAC setup on workers

Each worker cluster needs a ServiceAccount + ClusterRole +
ClusterRoleBinding that grant the manager permission to manage Jobs
and Workloads. The kubeconfig for this ServiceAccount is stored as a
Secret on the manager cluster and referenced by a MultiKueueCluster.

The helper `KubeconfigForMultiKueueSA` in `test/util/multikueue.go`
creates all three RBAC objects and returns a ready-to-use kubeconfig.

Two rule sets are available:

- **`MinimalMultiKueueRules()`**: covers only `batch/jobs`, `jobsets`,
  `kueue/workloads`, and `core/pods`. Suitable when only batch Jobs
  and JobSets are dispatched.
- **`DefaultMultiKueueRules()`**: covers the full set of integrations
  including KubeFlow training jobs, MPI jobs, RayClusters, RayJobs,
  AppWrappers, LeaderWorkerSets, StatefulSets, and Kubeflow Trainer
  jobs.

---

## E2E test architecture

### Directory structure

```
test/e2e/
  singlecluster/    # single-cluster tests (TAS, DRA, etc.)
  tas/              # TAS-specific single-cluster tests
  multikueue/       # multi-cluster tests (basic Job/JobSet/etc.)
  multikueue-dra/   # multi-cluster + DRA tests
  multikueue-tas/   # multi-cluster + TAS tests  ← new
  config/           # kustomize overlays
    default/
    dra/
    multikueue/
    multikueue-tas/
```

### Running tests

Single-cluster tests use `hack/testing/e2e-test.sh`. Multi-cluster
tests use `hack/testing/e2e-multikueue-test.sh`. The target test
folder is controlled by the `E2E_TARGET_FOLDER` env var, which
defaults to `multikueue`.

The Makefile target for MultiKueue+TAS E2E:

```makefile
run-test-multikueue-tas-e2e-%:
    MULTIKUEUE_TAS_E2E=true \
    E2E_TARGET_FOLDER="multikueue-tas" \
    ./hack/testing/e2e-multikueue-test.sh
```

### Overlay selection via `cluster_kueue_deploy`

`cluster_kueue_deploy` in `hack/testing/e2e-common.sh` selects which
kustomize overlay to deploy based on env vars:

```bash
if [[ -n ${CERTMANAGER_VERSION:-} ]]; then
    # deploy with cert-manager overlay
elif [[ -n ${DRA_EXAMPLE_DRIVER_VERSION:-} ]]; then
    build_and_apply_kueue_manifests "$1" ".../test/e2e/config/dra"
elif [[ -n ${MULTIKUEUE_TAS_E2E:-} ]]; then
    build_and_apply_kueue_manifests "$1" ".../test/e2e/config/multikueue-tas"
else
    build_and_apply_kueue_manifests "$1" ".../test/e2e/config/default"
fi
```

The `multikueue-tas` overlay enables only the `batch/job` integration
and sets `leaderElect: true` with two replicas of the controller.

### Kind cluster configs

`hack/testing/multikueue/worker-cluster.kind.yaml` already has the
TAS node labels on the worker node:

```yaml
- role: worker
  labels:
    cloud.provider.com/node-group: tas-node
```

This means the worker clusters are ready for TAS tests without any
additional node-labeling step.

---

## E2E test patterns

The following patterns are drawn from
`test/e2e/multikueue-tas/suite_test.go` and
`test/e2e/multikueue-tas/tas_test.go`.

### BeforeSuite

- Reads cluster names from env vars (`MANAGER_KIND_CLUSTER_NAME`,
  `WORKER1_KIND_CLUSTER_NAME`, `WORKER2_KIND_CLUSTER_NAME`).
- Creates three `client.WithWatch` clients (one per cluster) using
  `util.CreateClientUsingCluster("kind-<name>")`.
- Creates `rest.RESTClient` for each cluster (needed for pod exec).
- Calls `util.KubeconfigForMultiKueueSA` on each worker to create
  RBAC resources and get a kubeconfig, then stores it as a Secret on
  the manager with `util.MakeMultiKueueSecret`.
- Waits for Kueue to be fully available on all three clusters with
  `util.WaitForKueueAvailability`.

There is intentionally no `AfterSuite` in multi-cluster tests. With
`ginkgo -procs=2`, each process runs `AfterSuite` independently, and
if one process finishes early and deletes shared Secrets, the other
process fails. Cluster-scoped RBAC and kubeconfig Secrets are created
once in `BeforeSuite` and left for the OS/cluster teardown.

### BeforeEach

For each test:

1. Create namespaces on all three clusters. The worker namespaces use
   the same name as the manager namespace so that MultiKueue can
   mirror jobs correctly.
2. Create a Topology CR on all three clusters:
   ```go
   utiltestingapi.MakeDefaultOneLevelTopology("default")
   ```
3. Create a TAS ResourceFlavor on all three clusters:
   ```go
   utiltestingapi.MakeResourceFlavor("tas-flavor").
       NodeLabel(tasNodeGroupLabel, tasInstanceType).
       TopologyName(managerTopology.Name).
       Obj()
   ```
4. Create MultiKueue objects on manager only:
   - Two `MultiKueueCluster` objects pointing to the two kubeconfig
     Secrets.
   - One `MultiKueueConfig` listing both clusters.
   - One `AdmissionCheck` referencing the config.
5. Create ClusterQueues and LocalQueues on all three clusters.

### Asymmetric quotas for deterministic routing

Worker quotas are set differently so that a job requiring a specific
CPU amount will always be routed to the same worker:

- Manager ClusterQueue: 4 CPU (generous; acts only as a gate).
- Worker1 ClusterQueue: 2 CPU.
- Worker2 ClusterQueue: 1 CPU.

A job requesting 1500m CPU cannot fit on worker2 (1 CPU quota) but
fits on worker1 (2 CPU quota). This determinism avoids flaky tests
caused by race-based admission.

### AfterEach

Delete objects in reverse dependency order:

1. Namespaces (and their workloads/pods).
2. Worker ClusterQueues, ResourceFlavors, Topology CRs.
3. Manager ClusterQueue, ResourceFlavor, Topology CR.
4. Manager AdmissionCheck, MultiKueueConfig, MultiKueueClusters.

Use `ExpectObjectToBeDeletedWithTimeout` with `util.LongTimeout` for
objects that may take longer to clean up in CI (e.g. ClusterQueues
with active workloads).

### Generic helper for cross-cluster deletion

```go
type objAsPtr[T any] interface {
    client.Object
    *T
}

func expectObjectToBeDeletedOnWorkerClusters[PtrT objAsPtr[T], T any](
    ctx context.Context, obj PtrT,
) {
    gomega.Eventually(func(g gomega.Gomega) {
        util.ExpectObjectToBeDeleted(ctx, k8sWorker1Client, obj, false)
        util.ExpectObjectToBeDeleted(ctx, k8sWorker2Client, obj, false)
    }, util.Timeout, util.Interval).Should(gomega.Succeed())
}
```

The Go generics constraint ensures the function works for any
`client.Object` type (Workload, Job, etc.).

### Checking admission on a specific worker

`waitForJobAdmitted` polls the Workload's `AdmissionChecks` status
until the named check reaches `CheckStateReady`:

```go
func waitForJobAdmitted(wlLookupKey types.NamespacedName, acName, workerName string) {
    gomega.EventuallyWithOffset(1, func(g gomega.Gomega) {
        createdWorkload := &kueue.Workload{}
        g.Expect(k8sManagerClient.Get(ctx, wlLookupKey, createdWorkload)).
            To(gomega.Succeed())
        g.Expect(admissioncheck.FindAdmissionCheck(
            createdWorkload.Status.AdmissionChecks,
            kueue.AdmissionCheckReference(acName),
        )).To(gomega.BeComparableTo(&kueue.AdmissionCheckState{
            Name:    kueue.AdmissionCheckReference(acName),
            State:   kueue.CheckStateReady,
            Message: fmt.Sprintf(`The workload got reservation on "%s"`, workerName),
        }, cmpopts.IgnoreFields(kueue.AdmissionCheckState{},
            "LastTransitionTime", "PodSetUpdates")))
    }, util.LongTimeout, util.Interval).Should(gomega.Succeed())
}
```

### Workload name lookup

A Job's Workload name is deterministic and can be computed without
a cluster API call:

```go
wlLookupKey := types.NamespacedName{
    Name:      workloadjob.GetWorkloadNameForJob(job.Name, job.UID),
    Namespace: managerNs.Name,
}
```

`job.UID` is available immediately after `util.MustCreate` returns
because controller-runtime populates it from the API server response.

### Controllable pod lifecycle with agnhost

Tests use the `agnhost` image with specific behavior arguments to
control pod lifecycle:

```go
// Pod stays alive until terminated via HTTP /exit endpoint
BehaviorWaitForDeletion = []string{"netexec"}

// Pod exits immediately with no signal handling
BehaviorExitFast = []string{"entrypoint-tester"}
```

`GetAgnHostImage()` reads the image reference from
`hack/testing/agnhost/Dockerfile` (the `FROM` line), falling back to
the `E2E_TEST_AGNHOST_IMAGE` env var. `GetKuberayTestImage()` reads
`KUBERAY_RAY_IMAGE` from the environment (panics if absent).

To terminate a running agnhost pod in a test:

```go
util.WaitForActivePodsAndTerminate(
    ctx, k8sWorker1Client, worker1RestClient, worker1Cfg,
    job.Namespace, 1 /* expectedPods */, 0 /* exitCode */, listOpts,
)
```

---

## Key file paths

All paths are relative to the repository root
`/Users/ikennao/repos/OSS/kueue`.

| Purpose | Path |
|---|---|
| TAS annotation constants | `apis/kueue/v1beta2/topology_types.go` |
| MultiKueue constants | `apis/kueue/v1beta2/multikueue_types.go` |
| ResourceFlavor types | `apis/kueue/v1beta2/resourceflavor_types.go` |
| AdmissionCheck types | `apis/kueue/v1beta2/admissioncheck_types.go` |
| Test builder wrappers (v1beta2) | `pkg/util/testing/v1beta2/wrappers.go` |
| Test topology/flavor defaults | `pkg/util/testing/defaults.go` |
| Job test builder | `pkg/util/testingjobs/job/wrappers.go` |
| Job controller (GetWorkloadNameForJob) | `pkg/controller/jobs/job/job_controller.go` |
| RayCluster adapter + MultiKueue | `pkg/controller/jobs/raycluster/` |
| RBAC helpers for MultiKueue | `test/util/multikueue.go` |
| agnhost image + client helpers | `test/util/e2e.go` |
| Test timeout/behavior constants | `test/util/constants.go` |
| MultiKueue+TAS suite | `test/e2e/multikueue-tas/suite_test.go` |
| MultiKueue+TAS tests | `test/e2e/multikueue-tas/tas_test.go` |
| E2E kustomize overlay (MK+TAS) | `test/e2e/config/multikueue-tas/` |
| Shared E2E shell functions | `hack/testing/e2e-common.sh` |
| Multi-cluster test runner | `hack/testing/e2e-multikueue-test.sh` |
| Kind cluster configs (worker) | `hack/testing/multikueue/worker-cluster.kind.yaml` |
| Kind cluster configs (manager) | `hack/testing/multikueue/manager-cluster.kind.yaml` |
| Dev setup scripts | `site/static/examples/multikueue/dev/` |
| All E2E make targets | `Makefile-test.mk` |
