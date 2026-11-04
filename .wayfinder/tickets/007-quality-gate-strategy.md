---
title: How do we prove the port did not degrade the factory?
label: wayfinder:grilling
status: closed
assignee: yaroslav.y
blocked_by: [004]
---
## Question

How do we prove that the GitLab port behaves like upstream super-board on every lane: Ready → Building → QA → Review → merged, plus stop/resume, halts and the merge gate? Upstream is the behavioural reference; upstream tests on GitHub no longer apply.

Settle:
- how the ~20 gh-stub tests and the shared eval stub are ported to glab/GitLab-shaped fakes
- whether a shared fake replaces the per-test stubs
- how faithful the fakes must be to real gitlab.com responses (recorded fixtures?)
- the e2e smoke scenario on a sandbox gitlab.com group
- how evals show no regression against upstream results

## Resolution

The user decided this in a grilling session on 2026-11-04.

1. **Baseline first.** Run the unchanged upstream v3.3.0 test suite and all evals, and record each test's pass/fail and each eval's pass rate. The port must be no worse than this baseline. Tests that already fail upstream don't count against it.
2. **Tests translate 1:1.** Each upstream test keeps its scenario and assertions; only the forge shape changes (`glab`, GitLab fields, `lane:*` labels). A test may be dropped only when its behaviour is GitHub-only, such as Projects v2 option-ID reminting or draft issues. Every drop goes in a **dropped-behaviours ledger** with the reason.
3. **One shared, stateful `glab` fake** (`tests/_fake/glab`) replaces the ~20 per-test `gh` stubs and `evals/_stubs/bin/gh`.
   - It models issues (labels, assignee), MRs, notes and board lists.
   - It reproduces the known glab traps: auto-merge by default, a 409 on an `--sha` mismatch, quick actions, and auto-close only on the default branch.
   - Env knobs inject failures, and a call log keeps call-count assertions and eval graders working.
4. **Fixture fidelity.** Fake JSON is recorded from the gitlab.com sandbox, not hand-written. An opt-in live contract test replays key calls against the sandbox and diffs response shapes against the fixtures.
5. **E2E smoke** on a sandbox group with 2 projects.
   - Cards:
     - a `feature` card in repo A
     - a `bug` card in repo B
     - a `qa` card
     - a card with `blocked-by:` on the first
     - a card with two `lane:*` labels
     - a card the user assigned by hand without `loop:in-*`
   - Expected:
     - the first three cards reach Done with merged MRs and passing `verify_commands`
     - the dependent card starts only after the first one
     - both anomalous cards are skipped
     - `stop` mid-wave followed by `run` resumes cleanly
6. **Evals:** each eval runs 3 times on the same model, and the port's pass rate must be at least the baseline's.
7. **When things run:** tests run in GitHub Actions on every push. Evals and the e2e smoke run manually before the port is called done and after major changes.
