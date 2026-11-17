# Review guidance — cass-operator

<!-- Draft harvested from past maintainer review comments. Owner to correct. Shared rules live in shared/go-operator-guidelines.md; this file covers only what is specific to this repo. -->

Owners: @k8ssandra/maintainers

## What failure looks like here

A healthy Cassandra cluster gets disrupted. This includes an unexpected rolling restart after an operator upgrade, a scale-down or decommission that gets stuck or cleans up too early, data or nodes lost, or a reconcile that loops forever or floods the API server.

## Cassandra behaviour the code relies on

Many findings here depend on how Cassandra behaves, not on Go code. Use these facts. Check anything else against the source (see [Evidence](#evidence)).

- **Decommission timing.** A node reports `LEFT` in its own metadata when its announce period (`RING_DELAY`) starts, before other datacenters have heard it left. The Management API decommission job finishes only after that period. Local `LEFT` alone is not proof that the rest of the ring knows.
- **Peers after `LEFT`.** OSS Cassandra keeps a `LEFT` node in the peers' metadata for days, and the code treats HCD the same way. DSE removes it at once. A node missing from peer metadata counts as decommissioned only on DSE, and only after the node itself reported `LEFT`. On OSS, a missing entry proves nothing.
- **Runtime settings don't survive a restart.** Settings the operator applies through the Management API at runtime, such as full query logging, are lost when Cassandra restarts. That includes a container restart inside the same Pod (liveness probe, OOM kill). Nothing re-applies them from `cassandra.yaml`.

## Focus on

1. **Upgrade safety.** After an operator upgrade, existing clusters must behave as before unless the PR says otherwise.
   - Any change to the generated StatefulSet, pod template, Services or labels can restart or change every existing cluster. Flag it unless the PR says it is intended and the CHANGELOG mentions it.
   - Also look for changed behaviour on existing code paths: Management API client and transport settings, timeouts, defaults.
   - **Option and default changes have consumers outside the diff.** When a PR changes a controller-runtime or manager option, a client or transport setting, or a default, list the code that depends on the old behaviour, even when it isn't in the diff. For the reconcile context, that means goroutines that use `rc.Ctx` ([shared §1](../../shared/go-operator-guidelines.md#1-reconcile-loop)).
   - **Cache options have readers and writers outside the diff.** For a change to `ByObject` (label or field scoping), `DisableFor` or `DefaultTransform`, list every typed `Get` and `List` of that type, including exported helpers in `pkg/httphelper`. A cached `Get` of an object outside the selector returns NotFound. Also list every code path that adds or removes the selecting label: two writers that disagree make the label flap.
   - **State cached on an object goes stale.** When a PR records node runtime state in an annotation, label or status to skip a Management API call, list what changes that state without changing the object: a container restart in the same Pod, Cassandra restarted by the Management API, `nodetool` run by a user, and pods created before the upgrade, which don't have the annotation yet. Each one needs an invalidation, or the PR states it as a trade-off.
   - When the impact depends on the environment and you can't show it, ask whether the change is intended instead of describing a failure.
2. **Server and version gating.** Behaviour differs across `cassandra`, `dse` and `hcd`, and across major versions. For example, MCAC exists only on older versions. Check that version-specific code is gated correctly and that the gate is right.
3. **Order of checks, racks and StatefulSets.** Check rack balance, ordering, and acting on StatefulSets before their controller has caught up. Ask where in the sequence the edge case is handled.
   - `ReconcileAllRacks` runs an ordered list of checks. Look for a check that depends on state a later check loads or recovers.
4. **Long operations as state machines.** Scale, decommission, replace and tasks must be resumable after a restart and must not finish early when a Management API call fails. Check what happens when a request fails halfway.
   - **Every new blocking state names its exit.** For each new `result.Error`, `RequeueSoon`, `Done()` or latched condition, ask what clears it for each way it can be entered, and whether that code runs before this point in the pass. A `Done()` also names the watch event that brings the next pass: status-only changes don't trigger one. The predicates are `GenerationChanged || AnnotationChanged` for `CassandraDatacenter` and `GenerationChanged` only for `CassandraTask`. The test that owns the behaviour covers the unhappy exit.
   - **Destructive steps need positive evidence on every path.** When a PR changes a check that leads to deleting PVCs, scaling a StatefulSet down or marking a decommission done, list every path into that check, including fallbacks behind annotations such as `skip-local-decommission-check`, and the guards that run on each path. A guard placed after an early `return` doesn't protect that path. "Not found" proves completion only when the node was identified by a non-empty HostID. When a fallback replaces one data source with another, check that a later check doesn't end up comparing the data with itself.
5. **Transient vs terminal errors and requeues** ([shared guidelines §1](../../shared/go-operator-guidelines.md#1-reconcile-loop)). Look especially at code that polls Cassandra or the Management API.
6. **Per-reconcile cost.** New reads of Secrets, PVCs, Endpoints or config on every reconcile are expensive in cluster-wide deployments.
7. **Names and labels.** Kubernetes object names follow `metadata.name`, never the datacenter name override. Label value rules are not the same as annotation value rules.
8. **Webhook coverage.** Invalid transitions, such as scaling down while scaling up, are blocked in the webhook, not handled late in reconcile.
9. **Compatibility with k8ssandra-operator and k8ssandra-client.**
   - Watch for changes to defaults or fields that k8ssandra-operator sets or reads.
   - Both projects import `pkg/httphelper` as a library, so a change to its exported behaviour is an API change.
   - k8ssandra-client passes its own transport for port-forwarded connections.
10. **Tests prove the behaviour, at the right level.** Fake Management API features match what the test claims.
    - Changes to controller wiring (watches, predicates, manager, cache or client options) and to webhooks come with envtest coverage. Reconcile logic in `pkg/reconciliation` is tested with unit tests, the fake client and the fake Management API.
    - Operational changes come with an end-to-end suite, or an explanation of why not. If the PR depends on Cassandra behaviour that the fake can't show and gives no evidence for it, ask under "Questions for the author". Don't report it as a finding.
    - End-to-end changes: cleanup still deletes every namespace and datacenter the suite creates. A suite moved to a new topology keeps the old topology covered, or says why not.
11. **Mechanical changes and partial fixes.**
    - Lint rewrites, deprecation replacements and refactors are behaviour changes. The author states that each rewrite is equivalent, and the reviewer checks rewritten conditionals one by one. Conditions are tri-state ([shared §2](../../shared/go-operator-guidelines.md#2-kubernetes-semantics)).
    - For a bug fix, ask what else on the same path has the same property.

## Repo hygiene

Flag these as Important unless the item says otherwise. Items marked *(CI)* are mechanical. They stay here only until the matching CI check exists, then the line is deleted ([plan §3.1](../../00-plan.md#31-deterministic-checks-are-code-not-review)).

- **No changelog entry** *(CI, once the changelog check exists; there is none yet)*. Until changelog fragments (1.6b) land, the entry goes under `## unreleased` in `CHANGELOG.md`. Every change outside docs needs one, including a PR titled as a test fix that changes production code. Don't flag the category: which category fits is a judgement call.
- Image or version references hardcoded in Go instead of `config/imageconfig/image_config.yaml`.
- Unrelated changes: reformatting, untouched files in the diff, test data edited without the test needing it.
- **User docs made wrong.** Search `docs/` for the changed behaviour, annotation or field. A statement there that the PR makes false is Important, even when the PR doesn't touch docs.
- **User-facing change without a docs note**: a change users can see (fields, annotations users set, behaviour, metrics) with no docs note or docs link, even when the changelog kind isn't `FEATURE` ([plan §5.6](../../00-plan.md#56-user-facing-documentation-docs-sync)). **Nit** until changelog fragments (1.6b) give the note a place. Annotations and labels the operator writes for its own bookkeeping, which users don't set, aren't user-facing.
- **Description or CHANGELOG doesn't match the code**: it describes a design or mechanism the code doesn't implement, claims more than the code fixes, or ticks a checklist item with no matching change. Important when it hides a difference in behaviour or compatibility, otherwise a Nit.
- **Test sprawl** ([shared §6, items 7–8](../../shared/go-operator-guidelines.md#6-tests)): standalone regression tests, new tests for a change that doesn't alter behaviour, or new tests that only repeat assertions an existing test already makes. Report it once per PR and list every such test. For each one, say which existing test the case belongs in, or that it can be deleted.
  - An existing test **owns** the behaviour when it calls the same function with the same harness. A new scenario for that function is a new table row, adding a field to the table if needed, not a new test function.
  - **Important** when an existing test already owns the behaviour and the new test bypasses it.
  - **Nit** when no test owns the behaviour yet. Then the new test becomes the owning test, and renaming it after the behaviour or function is the whole fix.

## Needs a human (a marker, not a finding)

Some changes need human approval whatever the risk tier, even when they are correct. Don't report them as findings. List them on one line at the end of the summary: `Needs human: <what> in <file:line, ...>. Explained in the description: yes/no.`

- **New `//nolint` directives** *(CI detector, T17)*. The directive must name the linter and give a reason.
- **Changed or removed assertions in existing tests** *(CI detector, T17)*: different expected values, relaxed matchers, deleted checks, skipped tests. A renamed test whose expectations changed counts, and so does a setup change that makes an unchanged assertion check something else.

The description explains a change only when it names the behaviour the assertion now pins. The PR's general goal is not enough. Report an Important finding only when the description doesn't explain the change, or the explanation doesn't match the diff. If the change also removes the only test of a behaviour, report the missing coverage as a finding and still list the assertion here.

## Intentional trade-offs

If the PR description states a trade-off or a known limitation, don't flag it. Raise it only if it leads to one of the failures under "What failure looks like here". Then ask the author about it as a question rather than reporting it as a finding.

## Don't flag

- Style the linter enforces (`.golangci.yml`).
- Comments the shared guidelines say to delete. Flag that as a single Nit per PR, not one finding per comment.
- The wording of error and log messages that use the terms users know. For example, "the Cassandra version does not support it" is fine when the exact cause is the Management API image.
- What CI already checks: the linked issue, and generated files (`make manifests generate` must leave no diff).

## Skip these paths

- `**/zz_generated.deepcopy.go`
- `config/crd/bases/**`
- `go.sum`
- `tests/testdata/**`, unless the PR's purpose is test data

## Evidence

- Every finding cites a file and line and describes the concrete failure: what breaks, for which cluster or version, and when. Say how often the trigger has to happen, and set the severity from that. Don't guess from a name. Read the code it calls.
- Don't report a mechanism without tracing every function it names. Say what was traced and what was assumed.
- A finding, or the dismissal of one, that depends on how a library behaves (controller-runtime, client-go, `net/http`) cites the library source at the version in `go.mod` (`$(go env GOMODCACHE)/<module>@<version>/<path>:<line>`), or a small probe. "Probably" is not evidence.
- The same applies to Cassandra, DSE and Management API behaviour that isn't listed under [Cassandra behaviour the code relies on](#cassandra-behaviour-the-code-relies-on). Cite the Cassandra or `management-api-for-apache-cassandra` source. If you can't check it, ask it under "Questions for the author" instead of dismissing the finding.
- Conclusions from subagents are claims to verify, not findings.

## Limits

- At most 3 Nits per review.
- Tag each finding Important, Nit or Pre-existing.
- A transient error that surfaces as a Warning event or an error log but converges is a Nit, unless it loops.
- A rare trigger that ends in something irreversible is Important: a PVC deleted, a node removed before its data streamed, or a node removed before the ring heard it left. Say that the trigger is rare and what it needs.
- **Pre-existing** findings don't count toward the limits.
  - List them in the summary only, not as inline comments.
  - Give each one a one-line proposed `tech-debt` issue. Check first whether an issue already exists. If you can't search the tracker, write "not checked".
- Don't repeat findings already posted on this PR. If you can't see the PR discussion, say so once and review as if there were none.
- If the same problem appears in several places, report it once and list every location.

## Summary format

Start with a count, for example "N important, M nits, K pre-existing." If there is nothing to fix, say so plainly. Put questions for the author, such as unverifiable Cassandra behaviour or a trade-off to confirm, under "Questions for the author". They don't count as findings. End with the `Needs human:` line when it applies.
