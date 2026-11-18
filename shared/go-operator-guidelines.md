# Go operator guidelines (shared)

**Applies to:** all our Go controller-runtime operators. Each repo's `AGENTS.md` links here; each repo's `REVIEW.md` adds what is specific to that repo. \
**Source:** harvested from past cass-operator review comments (2021–2026), corrected by the owner.

**Scope:** judgement rules only. Anything a linter can enforce lives in `.golangci.yml`, and the codebase is fixed to comply with it (see [`pilot/cass-operator/lint-proposal.md`](../pilot/cass-operator/lint-proposal.md)). If a rule here becomes lintable, it moves to the linter config and is deleted from this file.

---

## 1. Reconcile loop

1. **Never block reconcile.** No long-running calls, waits or polling loops inside reconcile. Do the minimum, return, and requeue to check again.
2. **Reconcile runs very often.** In large clusters with many custom resources it runs constantly. Every extra API call, Secret read or config unmarshal is multiplied. Exit early when a feature isn't used, before doing any work for it.
3. **Know whether an error is transient or terminal.** If retrying with the same input cannot succeed, don't requeue in a loop. Use a terminal error (`reconcile.TerminalError`) or surface a status condition. Retrying with the same data that already failed only spins.
4. **Logging is not handling.** Logging an error and then returning an error result for something that will never succeed creates an endless reconcile.
5. **Use the sequence's own invariants before adding state.** If reaching the end of reconcile already means an earlier step succeeded, don't add a boolean to track it.
6. **Keep errors where they belong.** Return an error at the point it happens and let the retry handle it, rather than threading it into an unrelated process such as decommission.
7. **Respect controller lag.** The StatefulSet controller (or any other) may not have caught up yet. Don't act on objects whose status doesn't reflect the latest change. Show where in the sequence that situation is handled.
8. **State in goroutines doesn't persist.** Anything a goroutine computes is lost unless it's written to an object, for example a status or an annotation. Don't mutate outer variables from inside one and expect it to matter.
9. **A goroutine that outlives `Reconcile` doesn't use the reconcile context.** controller-runtime cancels that context when `Reconcile` returns whenever a reconciliation timeout is set. Derive a separate context (`context.WithoutCancel` plus its own timeout), and re-read objects before writing to them. When a PR changes controller options or defaults, check these goroutines.
10. **Fatal startup errors exit.** In `main`, an error that should stop the process calls `os.Exit(1)`. It must not carry on and start reconcilers.
11. **Expected transient conditions aren't errors.** A 409 conflict caused by a stale cache, or NotFound right after a create, resolves on its own. Requeue (for example `RequeueSoon`) without returning an error, so it doesn't produce a Warning event or an error log.
12. **A wait needs something that ends it.** A pass that returns done while waiting for a condition needs a watch that reports when the condition clears. Status-only changes don't pass `GenerationChangedPredicate`, and Secrets, ConfigMaps, upstream CRs and objects in remote clusters often aren't watched at all. Without a watch, use `RequeueAfter`. Don't rely on error retries to notice the change: the default backoff grows to 1000 seconds (client-go `DefaultTypedControllerRateLimiter`), so the condition can stay unnoticed for up to about 16 minutes after it clears.
13. **Every blocking state names its exit.** For each new error return, requeue, early done or latched condition, ask what clears it for each way it can be entered, and whether that code runs before this point in the pass. An event from an existing watch counts as an exit; name the watch and its predicate. The test that owns the behaviour covers the unhappy exit.

## 2. Kubernetes semantics

1. **Label values and annotation values have different rules.** Keys are similar, values are not. Use the Kubernetes validation helpers rather than hand-rolled regexes or blacklists.
2. **Object names derive from `metadata.name`.** Overrides used inside the application (for example a datacenter name override) must not leak into Kubernetes object names, or two resources can collide.
3. **Handle optional fields the API allows.** If Kubernetes allows ports without names, an empty template, and so on, the code must cope with them.
4. **Check that the platform actually supports what you rely on.** Examples: a StorageClass that supports modification, the StatefulSet controller accepting an in-place change, a CSI driver approving a change. Then reconcile until the change is confirmed, not just requested.
5. **Prefer OpenAPI validation (CRD markers) for static field rules.** Use webhooks for rules that need the old object, such as invalid transitions.
6. **Conditions are tri-state.** A condition that was never written reads as `Unknown`, not `False`. A check against `False` must be correct when the condition is unset, so ask which code writes it `False`. Prefer helpers whose names say how they treat the unset case.
7. **Option and default changes have consumers outside the diff.** When a PR changes a manager or controller option, a client or transport setting, or a default, list the code that depends on the old behaviour, even when it isn't in the diff (for example the goroutines in §1.9).
8. **Cache changes have readers and writers outside the diff.** When a PR changes what the client caches or how (scoping by label or field, disabling the cache for a type, transforms), list every read of that type, including exported helpers, and every code path that writes the label or field the cache is scoped on. A read outside the scope gets NotFound, not an error that says why. Typed, unstructured and metadata-only reads take different paths, so check which one each reader uses. A transform that drops fields from cached objects makes every `Update` of that type delete those fields on the server, because `Update` sends the whole object; list the `Update` calls for each transformed type.

## 3. API design (CRDs)

1. **Backward compatibility first.** Existing resources must keep working after an operator upgrade. API changes are called out as such in the CHANGELOG.
2. **One way to configure a thing.** Don't add a second mechanism for something already configurable elsewhere.
3. **No redundant switches.** An optional pointer struct doesn't need an extra `enabled` flag.
4. **Reuse existing API groups and packages** unless there is a stated reason for a new one.
5. **Don't hardcode versions or image references in Go code.** Put them in configuration, so changing them doesn't need a new operator release.
6. **Regenerate generated code and manifests** with the repo's pinned tool versions (`make manifests generate`), and commit the result.

## 4. Go style

Effective Go is the baseline; mechanical style is the linter's job. Reviewers look for what the linter can't see:

1. **Don't use pointers without a reason**, such as optional values, mutation or large structs.
2. **Use the right tool for the job,** for example `strings.Builder` for building strings rather than repeated `[]byte` casts.
3. **Name constants.** Annotation keys, label prefixes and magic strings become `const`s.
4. **Reuse existing helpers** rather than reimplementing naming or label logic. Look for existing functions before writing new ones.
5. **No new packages or files for a handful of helpers used in one place.**
6. **Names should say what something does.** If a function doesn't do what its name says, rename it or change it.

## 5. Comments

1. **Don't comment what the code already says.** No comments restating a type or a field. That is what a typed language and the editor are for.
2. **Reasoning usually goes in the commit message,** where `git blame` leads to it.
3. **Exception: a short comment for code that would otherwise confuse a reader,** such as a non-obvious rule, invariant or external constraint. Keep it to the minimum that prevents someone from "fixing" it. This is the exception, not the default.
4. **Delete commented-out code.**

## 6. Tests

Three levels. Each catches different problems; none replaces another.

| Level | Where | Framework | Use it for |
|---|---|---|---|
| **Unit** | Next to the code (`pkg/...`) | Go testing + testify (`require` for checks that must stop the test) | Pure logic: builders, parsing, calculations |
| **envtest** (integration) | In the controller and webhook packages, since the suite starts a real API server | Ginkgo/Gomega | Controller and webhook behaviour against a real API server: object creation, status updates, requeues, validation |
| **End-to-end** | `tests/<scenario>/` | Ginkgo/Gomega on a kind cluster | Real Cassandra behaviour: scaling, decommission, upgrades, version-specific behaviour |

**Prefer envtest for controller behaviour.** It gives far better feedback than unit tests with mocked clients and runs much faster than end-to-end tests. A controller change with only unit tests should explain why envtest wasn't possible.

**When a fake client is enough.** A fake client returns its own writes at once and has no cache, watches, resource versions or status subresource rules. It is fine for a single function whose result doesn't depend on timing or caching. Behaviour that spans reconcile passes, where a later pass reads back what an earlier one wrote, or that waits for or refreshes state from the API server, needs envtest: the API server's semantics are what the test has to show.

All levels:
1. **Tests must actually exercise the feature.** Check that fakes, such as a fake API server, expose the capability being tested. A test that passes because the fake lacks the feature proves nothing. Fakes must also be able to fail the way the real service does (errors and hangs per endpoint, unhealthy probes) for the failure paths a test claims to cover.
2. **Cover three cases for new behaviour:** existing behaviour unchanged, new behaviour works, and the failure path. "Existing behaviour unchanged" is normally shown by the existing tests still passing. Add a case for it only when no existing test covers that path.
3. **Test at the level that owns the behaviour** (table above).
4. **Keep test setup simple.** Inline data is often clearer than overriding methods. Use existing builders and helpers.
5. **Don't change test data unless the test needs it.** Test data doesn't have to mirror production.
6. **Check that assertions can fail.** For example, `-1 <= 1` passes when the value is an error sentinel.
7. **No standalone regression tests.** Behaviour is tested in the normal tests of the code, controller or scenario that owns it. A bug fix extends those tests, for example with a new table case or another assertion in the existing spec, so they cover the case they missed. Don't add a separate test named after the bug, the issue or the parameter that changed. If the behaviour matters, it belongs in the normal tests. If it doesn't, it needs no test. An existing test owns the behaviour when it calls the same function with the same harness. A new scenario for that function is a new table row, adding a field to the table if needed, not a new test function.
8. **No new tests when behaviour doesn't change.** A refactor, a rename or an internal restructuring is proven by the existing tests passing. The same goes for the default path of a new optional field that keeps the old behaviour when unset: only the new, set path needs a test. Don't add tests that only pin a constant, a default value or a field the diff touched, or that only check a function was called.
9. **Tests run with production settings.** The envtest manager uses the same controller options as `cmd/main.go` (reconciliation timeout, concurrency, panic recovery), and end-to-end suites deploy the operator with its default flags. Otherwise a changed default can't fail a test.
10. **End-to-end changes keep their coverage and clean up.** Cleanup still deletes everything the suite creates, in every namespace it uses. A suite moved to a new setup keeps the old setup covered, or says why not.
11. **Behaviour only the real service shows needs evidence.** If a change relies on how the managed application behaves (state transitions, timing, what survives a restart) and no test can show it with the fakes, the PR gives an end-to-end test or other evidence, or the reviewer asks the author for it.

Waiting and polling (envtest and end-to-end):
1. **Poll, don't sleep.** Use `Eventually` and the existing wait helpers.
2. **Assertions inside `Eventually` use the provided handle:** Gomega's `Eventually(func(g Gomega) { g.Expect(...) })`, or testify's `require.EventuallyWithT(t, func(c *assert.CollectT) { ... })`. A plain `Expect` or `require` inside the polled function fails the test on the first attempt, so the check stops being eventual.
3. **Timeouts should be realistic.** A step that needs ten minutes means the test has already failed.

## 7. Build and tooling

1. **Makefile values must be overridable,** using `?=` variables. Nothing hardcoded that breaks local builds.
2. **Call tools through their Makefile variables,** for example `$(HELM)`, and mark targets `.PHONY`.
3. **Don't duplicate what the build platform already does,** such as setting `GOARCH` when buildx `--platform` handles it.

## 8. Change hygiene

1. **No unrelated changes.** No mass re-indents, no touching files without a change, no reformatting outside the diff.
2. **Every change has an issue.** Follow-ups get a ticket in the repo, not a TODO.
3. **CHANGELOG entry** in the repo's format (see each repo's `REVIEW.md`).
4. **Classify changes as well as you can.** If nothing was broken, it isn't a bugfix. The category is the author's call, so reviewers don't flag it.
5. **Changed assertions in existing tests need a human.** If a PR modifies or removes an assertion in an existing test (a different expected value, a relaxed matcher, a deleted check, a skipped test), the reviewer lists it on the `Needs human:` line of its summary, and the PR requires human approval regardless of risk tier. Sometimes the change is correct because behaviour changed on purpose. But it is also the easiest way to hide a newly introduced bug. The PR description must say why the expectation changed. A renamed test whose expectations changed counts, and so does a setup change that makes an unchanged assertion check something else. The description explains the change only when it names the behaviour the assertion now pins; the PR's general goal is not enough. If the change also removes the only test of a behaviour, that missing coverage is a finding of its own.
6. **New lint suppressions need a human.** A new `//nolint` directive must name the specific linter and give a reason (`nolintlint` enforces both). Even then, the PR needs human approval regardless of risk tier. Suppressions are sometimes necessary, but each one switches off a check for good.
7. **User-facing changes come with user docs:** a docs note or a docs link, in the place and format the repo's `AGENTS.md` names. Internal changes need none. Annotations and labels the operator writes for its own bookkeeping, which users don't set, are internal.
8. **Don't leave user docs wrong.** Search the user docs for the changed behaviour, annotation or field. A statement there that the PR makes false is a defect, even when the PR doesn't touch docs.
9. **The description and CHANGELOG match the code.** They don't describe a design the code doesn't implement, claim more than the code fixes, or tick a checklist item with no matching change.

## 9. State, persistence and ownership

What gets written, when, and by whom. Mistakes here stall a resource or redo finished work. Each one looks harmless in a single pass.

1. **A write overwrites the in-memory object.** `Update`, `Patch`, `Status().Update` and `Status().Patch` decode the server's response into the object passed in (controller-runtime `pkg/client/typed_client.go`). Status changes made earlier in the pass and not yet written are lost when a mid-pass write targets the same object. Ask whether this happens on every pass (the status never advances) or once (it heals on the next pass).
2. **Write "finished" after the work it certifies.** If a completion marker (a finish time, a `Done` condition, a phase) is written before the follow-up steps, a failure in between is never retried, because the next pass sees "finished" and returns.
3. **Every object is reconciled again on restart, finished ones included.** On manager start the informer delivers a Create event for each existing object, and Create events pass `GenerationChangedPredicate`. A job-style controller needs an "already finished" check at the top, and its side effects must be safe to run twice.
4. **Hash what you apply.** With hash-based change detection, any change to the desired object after the hash is computed is stored in the hash but never applied. The next pass compares equal and never retries it. A change made after hashing has to be a deliberate, permanent rejection.
5. **Don't overwrite objects other actors also write.** `desired.DeepCopyInto(actual)` followed by `Update` replaces finalizers, labels, annotations, owner references and spec fields set by others: upstream operators, other controllers, users. Set only the fields this controller owns. When two controllers write the same object, check whether one undoes the other on every pass (a livelock) or only on a spec change.
6. **Every finalizer has a removal path that can always run.** Check that it runs when the parent is gone (a child garbage-collected through an owner reference can't wait for the parent's controller), when a remote cluster or client is unavailable, and for items still tracked in status after they've been removed from the spec. A helper used on the deletion path must not reject input that became invalid later.
7. **Webhooks don't block deletion or the operator's own patches.** `ValidateUpdate` skips objects with a `deletionTimestamp` and metadata-only changes. Otherwise a rule added in a newer version, or a check against the outside world, rejects the operator's finalizer and annotation patches, and a manual finalizer removal too.
8. **A panic repeats on every retry.** controller-runtime recovers it and turns it into an error, so the object panics on each attempt. Check lookups that can return `(nil, nil)` and pointer fields from API types before dereferencing them. A status write at the end of `Reconcile` that isn't in a `defer` is skipped on panic, so status never shows the failure.
9. **Errors reach the return.** Look for an `err` that is logged and then overwritten before anyone returns it, for a return of an earlier `err` that is known to be nil at that point, and for a result or error from a step function that the caller drops. Each one lets status report success after a failure.
10. **State recorded to skip work later has to stay true.** When an annotation, label or status records something so a later pass can skip a call, list the events that change the real state behind it. For each one, trace what the existing code already does to that object (deletes it, recreates it, rewrites the field) before calling the record stale.
11. **Destructive steps need positive evidence on every path.** When a check leads to deleting data or removing a member, list every path into it, including fallbacks behind annotations or flags, and the guards each path passes. A guard after an early `return` doesn't protect that path. Something missing from a response is not proof of completion unless the target was positively identified first. When a fallback swaps one data source for another, check that a later comparison doesn't compare the data with itself. When a destructive step moves into a path that retries or resumes, keep its identity check: a retry can meet a recreated object with the same name.
12. **An operation moved onto a shared path inherits that path's preconditions.** When a PR routes an operation through a generic path (a task runner, a per-pod loop, a helper), list what that path does before the operation runs: remote calls, readiness or status checks, lookups. Check that the operation still works for the inputs it exists for, for example a dead, Pending or unreachable target.

## 10. Evidence for the behaviour of dependencies

1. **Library behaviour comes from the library source.** A finding, or the dismissal of one, that depends on how a library behaves (controller-runtime, client-go, `net/http`) cites the source at the version in `go.mod`, or a small probe. "Probably" is not evidence.
2. **The managed application's behaviour too.** When a finding, or its dismissal, depends on how the application the operator manages behaves, read its source for the version in use or ask the author. Don't decide it from memory, in either direction.
3. **A failure chain with an unverified step is a question.** Write the chain that leads to a reported failure as steps. If any step is behaviour you couldn't read in source or show with a probe, the item goes to the author as a question, not as a finding. Name the unverified step.
