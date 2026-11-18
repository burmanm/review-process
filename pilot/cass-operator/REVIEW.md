# Review guidance — cass-operator

<!-- Draft harvested from past maintainer review comments. Owner to correct. Shared rules live in shared/go-operator-guidelines.md; this file covers only what is specific to this repo. -->

Owners: @k8ssandra/maintainers

## What failure looks like here

A healthy Cassandra cluster gets disrupted. This includes an unexpected rolling restart after an operator upgrade, a scale-down or decommission that gets stuck or cleans up too early, data or nodes lost, or a reconcile that loops forever or floods the API server.

## Focus on

1. **Upgrade safety.** After an operator upgrade, existing clusters must behave as before unless the PR says otherwise.
   - Any change to the generated StatefulSet, pod template, Services or labels can restart or change every existing cluster. Flag it unless the PR says it is intended and the CHANGELOG mentions it.
   - Also look for changed behaviour on existing code paths: Management API client and transport settings, timeouts, defaults.
   - Option, default and cache changes have consumers outside the diff (shared guidelines §2.7–2.8). Here that includes goroutines that use `rc.Ctx`, and the exported helpers in `pkg/` that other projects call.
   - State recorded to skip a call later has to stay true (shared guidelines §9.10).
   - When the impact depends on the environment and you can't show it, ask whether the change is intended instead of describing a failure.
2. **Server and version gating.** Behaviour differs across `cassandra`, `dse` and `hcd`, and across major versions. For example, MCAC exists only on older versions. Check that version-specific code is gated correctly and that the gate is right.
3. **Order of checks, racks and StatefulSets.** Check rack balance, ordering, and acting on StatefulSets before their controller has caught up. Ask where in the sequence the edge case is handled.
   - `ReconcileAllRacks` runs an ordered list of checks. Look for a check that depends on state a later check loads or recovers.
4. **Long operations as state machines.** Scale, decommission, replace and tasks must be resumable after a restart and must not finish early when a Management API call fails. Check what happens when a request fails halfway.
   - Every new `result.Error`, `RequeueSoon`, `Done()` or latched condition names its exit and the event that brings the next pass (shared guidelines §1.12–1.13).
   - Decommission, PVC cleanup and node replacement are destructive steps: every path into them needs positive evidence (shared guidelines §9.11).
   - A task or operation moved onto a shared path still works on the pods it exists for, including dead or unreachable ones (shared guidelines §9.12).
5. **Transient vs terminal errors and requeues** (shared guidelines §1). Look especially at code that polls Cassandra or the Management API.
6. **Per-reconcile cost.** New reads of Secrets, PVCs, Endpoints or config on every reconcile are expensive in cluster-wide deployments.
7. **Names and labels.** Kubernetes object names follow `metadata.name`, never the datacenter name override. Label value rules are not the same as annotation value rules.
8. **Webhook coverage.** Invalid transitions, such as scaling down while scaling up, are blocked in the webhook, not handled late in reconcile.
9. **Compatibility with k8ssandra-operator and k8ssandra-client.**
   - Watch for changes to defaults or fields that k8ssandra-operator sets or reads.
   - Both projects import `pkg/httphelper` as a library, so a change to its exported behaviour is an API change.
   - k8ssandra-client passes its own transport for port-forwarded connections.
10. **Tests prove the behaviour, at the right level** (shared guidelines §6). Fake Management API features match what the test claims.
    - `pkg/reconciliation` has many fake-client unit tests because they predate a usable envtest. That is history, not the rule. A change to a single function that an existing fake-client table test owns extends that table. Behaviour that spans reconcile passes, or that waits for or refreshes state from the API server, gets envtest coverage in `internal/controllers/cassandra` or `internal/controllers/control`.
    - A test setup that doesn't use the production options (shared guidelines §6.9) is a Nit on its own. When it is why a defect in this PR went untested, say so inside that defect's finding instead.
    - Operational changes come with an end-to-end suite, or an explanation of why not. If the PR relies on Cassandra behaviour that the fake can't show and gives no evidence for it, ask under "Questions for the author". Don't report it as a finding.
11. **Mechanical changes and partial fixes.**
    - Lint rewrites, deprecation replacements and refactors are behaviour changes. The author states that each rewrite is equivalent, and the reviewer checks rewritten conditionals one by one. Conditions are tri-state (shared guidelines §2).
    - For a bug fix, ask what else on the same path has the same property.

## Repo hygiene

Flag these as Important unless the item says otherwise. Items marked *(CI)* are mechanical. They stay here only until the matching CI check exists, then the line is deleted.

- Image or version references hardcoded in Go instead of `config/imageconfig/image_config.yaml`.
- **Test data or test setup edited without a test in this PR needing it.** Important, and the review requests changes until the author explains each edit: an edit no test needs can hide a defect by making a failing test pass. Report each edit separately and say what it could hide.
- Unrelated changes: reformatting, untouched files in the diff. One finding per PR that lists every item.
- User docs made wrong (shared guidelines §8.8).
- **User-facing change without a docs note** (shared guidelines §8.7), even when the changelog kind isn't `FEATURE`. **Nit** until changelog fragments (1.6b) give the note a place.
- **Description or CHANGELOG doesn't match the code** (shared guidelines §8.9). Important when it hides a difference in behaviour or compatibility, otherwise a Nit.
- **Test sprawl** (shared guidelines §6, items 7–8): standalone regression tests, new tests for a change that doesn't alter behaviour, or new tests that only repeat assertions an existing test already makes. Report it once per PR and list every such test. For each one, say which existing test the case belongs in, or that it can be deleted.
  - **Important** when an existing test already owns the behaviour and the new test bypasses it: it tests the same function in a new test function or a second table instead of adding a row.
  - **Nit** when no test owns the behaviour yet. Then the new test becomes the owning test, and renaming it after the behaviour or function is the whole fix.

## Needs a human (a marker, not a finding)

New `//nolint` directives and changed or removed assertions in existing tests need human approval (shared guidelines §8.5–8.6) *(CI detector, T17)*. Don't report them as findings. List them on one line at the end of the summary: `Needs human: <what> in <file:line, ...>. Explained in the description: yes/no.` When the items differ, give the answer per item.

A `//nolint` is only ever a marker; `nolintlint` already requires its reason. For a changed assertion, report an Important finding only when the description doesn't explain the change, or the explanation doesn't match the diff.

Every question for the author and every uncertainty also goes here, one line each: `Needs human: <question, or what you could not verify> (<file:line>).` That includes behaviour outside this repo the change depends on, an assumption a finding rests on, and a severity you weren't sure of. When in doubt, list it: a human dismisses an extra line faster than they find a missed one.

## Intentional trade-offs

If the PR description states a trade-off or a known limitation, don't flag it. A stated trade-off names what it gives up. If its stated premise is false, for example a path it calls rare runs on every pass, it is not a stated trade-off: report the mismatch (shared guidelines §8.9) with the failure it causes. Raise it only if it leads to one of the failures under "What failure looks like here". Then ask the author about it as a question rather than reporting it as a finding.

## Don't flag

- Style the linter enforces (`.golangci.yml`).
- Comments the shared guidelines say to delete. Flag that as a single Nit per PR, not one finding per comment.
- The wording of error and log messages that use the terms users know. For example, "the Cassandra version does not support it" is fine when the exact cause is the Management API image.
- What CI already checks: the linked issue, and generated files (`make manifests generate` must leave no diff).

## Skip these paths

- `**/zz_generated.deepcopy.go`
- `config/crd/bases/**`
- `go.sum`
- `tests/testdata/**`: don't review the contents, but check that a test in this PR needs every edit there (see "Repo hygiene")

## Evidence

- Every finding cites a file and line and describes the concrete failure: what breaks, for which cluster or version, and when. Say how often the trigger has to happen, and set the severity from that. Don't guess from a name. Read the code it calls.
- Don't report a mechanism without tracing every function it names. Say what was traced and what was assumed.
- Library and server behaviour need evidence (shared guidelines §10). Cite library source as `<module>@<version>/<path>:<line>`. For Cassandra, DSE or the Management API, read the source for the version in use or ask under "Questions for the author". A finding whose failure chain has an unverified step is a question, not a finding (shared guidelines §10.3).
- Conclusions from subagents are claims to verify, not findings.

## Limits

- At most 3 Nits per review, and fewer is fine. A Nit is a change the author may skip and still merge. If the PR shouldn't merge without the change, it is Important, not a Nit.
- Tag each finding Important, Nit or Pre-existing.
- A transient error that surfaces as a Warning event or an error log but converges is a Nit, unless it loops: the same error recurs on every pass with nothing changing between passes.
- A rare trigger that ends in something irreversible, such as deleted data or a node removed too early, is Important. Say that the trigger is rare and what it needs.
- **Pre-existing** findings don't count toward the limits.
  - List them in the summary only, not as inline comments.
  - If the PR makes a pre-existing defect reachable or more frequent, it stays Pre-existing. Say that the PR widens the trigger, and ask the author whether to fix it here.
  - Give each one a one-line proposed `tech-debt` issue. Check first whether an issue already exists. If you can't search the tracker, write "not checked".
- Don't repeat findings already posted on this PR. If you can't see the PR discussion, say so once and review as if there were none.
- If the same problem appears in several places, report it once and list every location.

## Summary format

Start with a count, for example "N important, M nits, K pre-existing." If there is nothing to fix, say so plainly. Put questions for the author, such as unverifiable Cassandra behaviour or a trade-off to confirm, under "Questions for the author". They don't count as findings. End with the `Needs human:` lines: the markers, then every question and uncertainty.
