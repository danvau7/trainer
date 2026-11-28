# KEP-3120: Network policies for training runtimes

**Initial working draft.** This proposal has not been submitted or approved
upstream. Field names and behavior below are candidates for discussion,
not supported Trainer APIs.

Author: Dan Ehrlich (@danvau7)

Tracking issue: [kubeflow/trainer#3120](https://github.com/kubeflow/trainer/issues/3120)

Implementation prototype: [kubeflow/trainer#4308](https://github.com/kubeflow/trainer/pull/4308)

## Summary

Add optional network policy configuration to `TrainingRuntimeSpec`, shared by
`TrainingRuntime` and `ClusterTrainingRuntime`. Platform authors describe the
access a runtime needs; Trainer creates separate Kubernetes NetworkPolicies for
each TrainJob and its private cache. Jobs using the same runtime share an access
profile while remaining isolated from each other.

The proposed `TrainJobNetworkPolicy` feature gate starts disabled at alpha.
The controller manages policy identity, ordering, repair and cleanup. Job
submitters keep their existing TrainJob and SDK workflows.

## Motivation

The prototype puts additional ingress and egress rules in controller-wide
configuration. This gives every isolated job the same exceptions. A runtime with
model or dataset downloaders may need access that another runtime should not have.

[Robert Bell's review](https://github.com/kubeflow/trainer/pull/4308#discussion_r4721595721)
requests runtime-specific rules and proposes a short KEP and feature gate.
His separate [EndpointSlice question](https://github.com/kubeflow/trainer/pull/4308#discussion_r4721736144)
asks us to justify infrastructure discovery. These are separate design decisions.

### Goals

- Configure approved access per runtime, with separate policies per TrainJob.
- Cover initializer, trainer and private-cache pods using controller-managed identity.
- Keep essential infrastructure reachable and make additional access explicit.
- Preserve isolation through pod replacement, retries and job lifecycle changes.
- Introduce the API behind a disabled-by-default alpha feature gate.

### Non-Goals

- A general Kubernetes resource-template engine, PVC provisioning or Gateway management.
- Authentication, encryption, namespace-administrator isolation or protection against
  a tenant allowed to spoof trusted labels or create bypass policies.
- Adding direct arbitrary network-policy overrides to TrainJob or its runtime patches.

## Proposal

A runtime opts in by setting `spec.networkPolicy` when the feature gate is enabled.
An empty object requests baseline isolation; an omitted field preserves existing
behavior. Platform authors can add common rules and, subject to API review, rules
targeting named ReplicatedJobs such as `dataset-initializer`.

Trainer uses the TrainJob's runtime snapshot to render policies, applies them before
workload resources, and repairs edited or deleted policies. A policy-application
error blocks new workload creation. Creating a policy object does not guarantee
that the CNI has finished enforcing it.

### User Stories

- A platform author permits model downloads for an initializer while workers use
  only their job's peers and private cache.
- A second runtime permits a different storage service without granting that
  access to every runtime managed by the operator.
- A job submitter selects an approved runtime without authoring NetworkPolicies.

### Risks and Mitigations

Only trusted runtime authors may grant access. Standard NetworkPolicies combine
allow rules; a restrictive policy cannot override another policy's broad allowance.
Do not preserve broad controller-wide exceptions and claim that runtime rules
narrow them.

Reserve and validate job identity labels, preserve object ownership, and reject
unsupported cases instead of silently leaving some workload pods unprotected.
The platform must supply an enforcing CNI and appropriate tenant RBAC/admission
controls. Shared services may require matching ingress rules managed by their owner.

## Design Details

### Candidate API

The initial shape reuses `networking.k8s.io/v1` ingress and egress rule types.
The controller supplies names, namespace, owner references, pod selectors and
policy types. The runtime does not provide arbitrary policy manifests or a
template language.

The following is a proposed `TrainingRuntime.spec` fragment. The referenced
ReplicatedJob must exist in that runtime's existing JobSet template.

```yaml
spec:
  networkPolicy:
    ingress: []
    egress: []
    replicatedJobs:
      - name: dataset-initializer
        egress:
          - to:
              - namespaceSelector:
                  matchLabels:
                    kubernetes.io/metadata.name: storage
                podSelector:
                  matchLabels:
                    app: minio
            ports:
              - protocol: TCP
                port: 9000
```

Common `ingress` and `egress` exceptions apply to that job and its private cache.
A `replicatedJobs` entry adds exceptions only to pods for that named ReplicatedJob.
For example, the fragment grants initializer pods access to the selected MinIO
pods without granting the same exception to trainer or cache pods. Within the
single peer entry, the namespace and pod selectors must both match.

Targeted rules add access; they do not override common allowances. Initializer
rules apply to separate initializer pods. Kubernetes NetworkPolicy cannot assign
different access to an init container and a main container sharing one pod.

Names must be unique and refer to existing ReplicatedJobs. Validate native rule
semantics, CIDRs, ports, protocols and list bounds, using CEL where appropriate.
The exact field names, targeting shape and limits remain open for review.

### Baseline and infrastructure access

Baseline policies allow communication within the same job and its private cache,
plus DNS, the Kubernetes API and Trainer status access when applicable.

- **DNS:** use scoped pod/namespace selectors for supported DNS deployments.
  Document administrator configuration for custom DNS or NodeLocal paths.
- **Trainer status:** prefer pod/namespace selectors and a named or configured
  target port. Any remaining need for EndpointSlice port discovery must be justified.
- **Kubernetes API:** retain automatic, narrowly scoped destination discovery as
  the candidate default in this draft. Permit the Service address and discovered
  backend addresses/ports as needed for the tested CNI's Service translation behavior.
  Do not substitute unrestricted outbound HTTPS access.

A managed control plane such as EKS is not represented by ordinary selectable
workload pods. The API Service can therefore require address-based rules. A private
API endpoint is still not a pod identity. Pod/namespace selectors remain the
preferred mechanism for ordinary in-cluster services.

EndpointSlice discovery keeps generated allowances current when relevant backend
addresses or ports change. It has costs: cache memory, read permissions, policy
refreshes and maintenance of CNI-specific behavior. New infrastructure watches
must be namespace-scoped, filter unrelated Services, and enqueue only jobs using
this feature. Existing unrelated controller watches retain their requirements.
Generate rules in deterministic order to avoid reconciliation churn.

Discovery failures must produce actionable errors and prevent new workloads from
starting with incomplete desired policies. Existing workloads retain their last
successfully applied policies; this is not proof of continued API connectivity.

The alternative is administrator-configured API destinations. This removes discovery
but transfers address maintenance to administrators. Which approach should be the
alpha default is an explicit review question; the watches are not assumed unnecessary.
No EKS validation or measured watch-overhead result is claimed.

### Ownership, lifecycle and compatibility

- The gate and runtime field must both opt in. Reject new policy configuration
  while the gate is disabled rather than silently accepting unenforced settings.
- Create namespaced policies owned by the TrainJob, including for a cluster-scoped
  runtime. Never adopt or delete a foreign same-name object. Conditional writes and
  delete preconditions must protect ownership against concurrent object replacement.
- Use the runtime snapshot so later runtime edits do not silently change a running
  job's access. Jobs whose snapshots lack policy configuration are not retrofitted.
  Infrastructure destination refreshes are distinct from changing runtime intent.
- Keep policies while completed or suspended jobs still have pods or private caches.
  Delete them with the owning TrainJob. Retained PVC data has a separate lifecycle;
  this proposal does not delete storage or take ownership of shared caches.
- Gate disablement or controller downgrade must not silently remove isolation.
  Existing policies remain until owner deletion or an explicit administrator cleanup.
  Document cleanup and required permissions before downgrade or feature removal.
- Keep `runtimePatches` restricted: do not expose the new policy fields there.
  Validate patches that could change reserved identity labels. Runtime write access
  and permission to select an unprotected runtime remain part of the trust boundary.

The prototype's controller-wide rule configuration is not a released API. The
implementation revision should replace workload exceptions with runtime-owned
rules; any operator-wide settings should describe infrastructure or feature enablement.
No controller, CRD, Helm or SDK code is changed by this draft.

### Test Plan

The following tests are planned for the revised implementation, not reported results.

#### Unit Tests

Cover omitted/empty fields, gate enforcement, rule validation, targeted initializer
access, same-runtime job separation, cache selection, deterministic endpoint order,
ownership conflicts, API errors and reserved-label/runtime-patch validation.

#### Integration Tests

Verify policy-before-workload ordering and failure handling, scoped metadata and
infrastructure watches, ownership replacement races, policy repair, snapshots,
retries, suspension, completion, cleanup and gate/rollback behavior.

#### E2E Tests

On an enforcing CNI, verify allowed same-job and denied cross-job traffic across
nodes and namespaces. Show that only the targeted initializer reaches approved
storage, that workers cannot inherit its exception, and that DNS/API/status access
works. Exercise pod recreation, native LWS cache pods and policy garbage collection.
Test both selector-based services and selectorless API destinations, IPv4/IPv6 where
supported, and endpoint rotation. Run an EKS case before claiming EKS support.
Test new connections; existing-connection behavior depends on the CNI.

### Graduation Criteria

Alpha requires an agreed API, a disabled-by-default gate, schema/webhook and
controller tests, enforcing-CNI coverage, and documented RBAC, trust and rollback
behavior. Beta requires runtime-author feedback, a supported CNI/platform matrix,
measured reconciliation overhead and settled infrastructure discovery semantics.
A release target and stable graduation criteria are not assigned in this draft.

## Implementation History

- Prototype implementation and exploratory validation: upstream PR #4308. Its
  controller-wide configuration does not implement this proposed runtime API.
- 2026-11-28: initial KEP draft prepared for iteration on the author's fork.
  No upstream acceptance or implementation status is implied.

## Drawbacks

The runtime API and controller gain policy-management responsibilities. Automatic
infrastructure discovery improves usability but adds watches and failure modes.
Role targeting requires reliable labels and makes the API larger than common
runtime-wide rules alone.

## Alternatives and Prior Art

- **Controller-wide exceptions:** simple deployment configuration, but grants the
  same additional access to every job. Retain only common infrastructure
  allowances if their scope is accepted.
- **Manually managed NetworkPolicies:** established and keeps Trainer smaller, but
  platform owners must coordinate per-job identity, ordering and cleanup themselves.
- **A separate policy controller or Kueue AdmissionCheck:** separates responsibilities
  and can coordinate prerequisites, but introduces deployment, identity and lifecycle
  coordination. Trainer should not require Kueue to provide basic isolation.
- **Argo Workflows resource templates:** an established declarative pattern for
  explicitly creating Kubernetes resources and optionally attaching ownership for
  cleanup. Argo also has PVC templates. This proposal is inspired by that pattern;
  it is not an Argo-standard NetworkPolicy API or a copy of its generic executor.
- **Kueue component policies:** the policies reviewed protect Kueue's controller/UI.
  They are not an automatic per-training-job isolation mechanism.
- **Static API destinations or CNI-specific API identities:** avoid EndpointSlice
  discovery, at the cost of administrator maintenance or portability. Standard
  NetworkPolicy does not offer a portable Service-name or API-server identity selector.

## Open Questions

1. Should alpha include ReplicatedJob targeting, or first introduce common runtime
   rules with targeting designed explicitly as the next step?
2. Which infrastructure allowances are baseline, and which require explicit runtime
   intent? Should selectorless API discovery remain the default?
3. What are the minimal RBAC and rollout/cleanup controls for disabled and enabled
   installations, and what policy changes are permitted for existing snapshots?
4. Which stable labels cover every supported initializer and private-cache pod,
   including long names and recreation? What are the explicit support boundaries
   for stateful workloads, host networking, meshes and secondary interfaces?

## References

- [Kubeflow KEP process and template](https://github.com/kubeflow/community/blob/master/proposals/README.md)
- [Trainer runtime guide](https://trainer.kubeflow.org/en/latest/operator-guides/runtime.html)
- [Kubernetes NetworkPolicy semantics and limitations](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Amazon EKS control-plane architecture](https://docs.aws.amazon.com/eks/latest/userguide/eks-architecture.html)
- [Argo Kubernetes resource templates](https://argo-workflows.readthedocs.io/en/latest/walk-through/kubernetes-resources/)
- [Argo volume claim templates](https://argo-workflows.readthedocs.io/en/latest/walk-through/volumes/)
- [Kueue AdmissionChecks](https://kueue.sigs.k8s.io/docs/concepts/admission_check/)
- [Kueue NetworkPolicy guide reviewed for this draft](https://github.com/kubernetes-sigs/kueue/blob/e1c47e65421b2688c808dfdb9e10c625c7df4571/site/content/en/docs/tasks/manage/productization/network_policies.md)
