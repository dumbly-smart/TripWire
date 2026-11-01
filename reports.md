# Reports and experiment artifacts

This page is an index and interpretation guide for Tripwire outputs. Generated
guest artifacts under `data/vm/` are ignored by Git and are not automatically
part of a checkout. Preserve the exact configs, lock files, run manifests,
logs, and source revision with any artifact you analyze.

## Record types

| File | Meaning |
|---|---|
| `runs.jsonl` | One attempted task/attack per JSON line, including success/error and labels. |
| `calls.jsonl` | One agent tool call per line, with call ID, arguments, PIDs, cgroup, and monotonic time window. |
| `events.jsonl` | Parsed eBPF event stream, including `stats` records for lost-event counters. |
| `joined.jsonl` | Tool-call records annotated with assigned behavioral events. |
| `join-report.json` | Assigned, orphan, ambiguous, stats, dropped-event and grace-window counts. |
| `manifest.json` | Run metadata, effective research configs, task/model profile, and measurement kind. |
| `detector-eval/results.json` | Task split, thresholds/config, per-detector metrics and corpus counts. |
| `detector-eval/scores.jsonl` | Per-sample raw detector scores and explanations. |
| `detector-eval/attack-verdicts.jsonl` | Thresholded verdict per attack, detector, and target FPR. |
| `detector-eval/false-positives.jsonl` | Benign test samples flagged, with causes for inspection. |
| `taxonomy-results.json` | Case goal, expected and observed OS evidence, verdicts, and operational class or `pending`. |
| `ablation-eval/results.json` | Feature-set metrics for files, network, process, sequence, combined. |
| `ablation-eval/false-positives.jsonl` | False-positive cases for each ablation. |
| `ablation-eval/attack-verdicts.jsonl` | Attack scores/verdicts for ablations. |
| `ablation-eval/ablation-tpr.svg` | Static plot generated from ablation results. |
| `mimicry-summary.json` | Per-strategy goal outcome and per-detector verdict summary. |

The exact artifact names returned from QEMU are listed in
`scripts/run_phase0_vm.py`. Run directories are deliberately unique; existing
results are preserved instead of overwritten.

## How to read the metrics

- **Goal success** is determined by a task success predicate or a matching
  synthetic canary receipt at the loopback sink. A tool call or suspicious OS
  event alone does not prove the intended attack succeeded.
- **TPR** is the fraction of attack rows above a detector threshold. If attack
  goals failed, this is not a successful-attack detection rate.
- **Target FPR** is the requested calibration quantile, here 1% or 5%.
  **Measured FPR** is the observed fraction of held-out benign rows flagged;
  with a small calibration/test set it can differ substantially from target.
- **Thresholds** are calibrated using benign calibration tasks. Runs of one
  task stay in a single partition. Attack rows are test-only.
- **Orphan-event rate** is behavioral events not assigned to a tool call divided
  by behavioral event count. `stats` snapshots are reported separately.
- **Stable/noisy features** count feature presence across task attempts; they
  are corpus summaries, not universal allowlists.
- **OS-visible / partially visible / OS-invisible** are operational labels for
  successful goals and calibrated OS-only detector outcomes under the current
  model, features, kernel, and thresholds. Unsuccessful or unscored cases are
  `pending`.
- **Ablations** are comparisons of configured feature subsets. They do not
  establish causality or generalization beyond the small measured corpus.

See `docs/notes/phase4.md`, `phase5.md`, `phase6.md`, `phase7.md`, and
`docs/taxonomy.md` for method details and caveats.

## Currently documented historical results

These values come from the saved Phase 3–7 notes and guest artifacts dated
2026-10-26. They are not results for the four benign task definitions added
after the measured 30-task corpus.

| Evidence | Historical result | Interpretation |
|---|---:|---|
| Benign corpus | 30 tasks × 3 seeds = 90 runs; 15 goals succeeded | Qwen3 0.6B Q4_K_M; 16.7% utility success is a severe confound. |
| Behavioral event joining | 174/1,218 assigned; 1,044 orphan (85.71%); 0 ambiguous; 0 dropped | Tool-to-syscall correspondence is incomplete. 2,675 `stats` rows excluded from denominator. |
| Injection suite | 7 cases; 0 goals succeeded | No successful injection evidence; taxonomy stays pending. |
| Mimicry campaign | 2/2 local canary goals reached | Only the earlier naive and one-call adaptive profiles were measured. |
| Mimicry allowlist | 2/2 flagged at 1% and 5%; 0/30 measured benign false positives | Two deterministic probe rows, not an attack population estimate. |
| Mimicry n-gram | 2/2 flagged at 1% and 5%; 3/30 measured benign false positives (10%) | All 18 benign fetch attempts had zero tool calls, so adaptive fetch was not meaningfully in-envelope. |
| Taxonomy | 7/7 cases pending | No injection goal succeeded. Detector positives on failed attempts are not attack TPR. |
| Ablations | Files 1/7, network 1/7, process 0/7, sequence 1/7, combined 1/7 flagged | Attacks failed, so these are not successful-attack detection rates. Measured benign FPR was 0/30 for files/network/process, 3/30 (10%) sequence/combined. |

Those rows are explicitly historical and have been superseded by the
2026-11-01 refreshed corpus and four-strategy campaign documented below. The
historical results remain useful for tracing how the harness evolved; do not
combine them with refreshed measurements or use them as current campaign
coverage. The latest full-pipeline guest run reused a cached-model base, so
blank-VM reproduction remains unverified.

## Fresh Phase 0–2 acceptance (2026-11-01)

Source commit `2492d8f` passed in a new KVM overlay over the accepted Ubuntu
24.04.5 guest. `make check` verified kernel `6.8.0-146-generic`, readable BTF,
cgroup v2, clang 18.1.3, libbpf 1.3.0, and bpftool 7.4.0. All 191 unit tests
passed. The tracer smoke recorded 231 output records, 217 syscall records,
zero dropped events, and eight `write` syscalls; the updated smoke also
asserted a successful TCP write using the FD that had connected to the local
sink. The ten-task agent smoke completed and logged 11 tool calls. Its report
assigned 428 of 2,206 behavioral events and orphaned 1,778 (80.60%); there
were no ambiguous events and four stats snapshots. This demonstrates current
guest build/runtime acceptance while confirming that event-to-call joining
still loses many events.

A fresh Phase 3 attempt reached model installation, but its 1,435 MiB model
download had transferred only about 221 KiB after 1 minute 43 seconds. The
disposable guest was stopped before it returned a result marker; this produced
no task measurements and must not be added to corpus statistics. The serial
transfer failure after stopping it is `RuntimeError: guest returned no result;
see data/vm/serial.log`. A separate attempt from an older saved model guest
failed before acceptance because that image booted a stale enabled Tripwire
service, which printed `check_env.py: error: unrecognized arguments:
--lock-path config/environment.lock.json`. These are runner/image setup
failures; neither changes the historical Phase 3–7 result table above.

## Latest Phase 0–2 acceptance after provenance fix (2026-11-01)

Source commit `a06a775` passed `make check`, 193 unit tests, and both smoke
suites in the disposable Ubuntu 24.04.5 KVM guest. The tracer emitted 231
records with zero dropped events and passed the TCP socket-write assertion.
The ten-task join smoke completed with 428/2,206 behavioral events assigned,
1,778 orphaned (80.60%), and zero ambiguous. This supersedes the earlier
191-test run recorded below for current source acceptance; it does not improve
the join-quality limitation.

## Refreshed Phase 3–7 evidence (2026-11-01)

After adding the early cloud-init unit reset, the cached model guest ran the
full pipeline in fresh overlays under KVM. The benign corpus used the pinned
`config/model.tiny.json` profile and verified model lock: Ollama 0.40.7,
`qwen3:0.6b-q4_K_M`, context 2,048, prediction limit 512, temperature 0,
180-second request timeout, and three seeds (11, 23, 37). VM resources were
1,536 MiB RAM and 1,024 MiB guest-only swap. The model is a severe task-utility
confound, so these results describe the harness and this profile only.

| Evidence | Refreshed result | Interpretation |
|---|---:|---|
| Benign corpus | 34 tasks × 3 seeds = 102; 18 successes (17.65%) | Edit 9/18, fetch 3/30, search 0/18, summarize 6/18, tests 0/18. |
| Event joining | 188/1,582 assigned; 1,375 orphan (86.92%); 19 ambiguous; 0 drops | Join quality is a major limit. 2,390 stats snapshots excluded from the behavioral denominator. |
| Allowlist on 7 injection rows | 1/7 flagged; 0/30 held-out benign FPs | All injection goals failed, so not successful-attack TPR. |
| N-gram on 7 injection rows | 1/7 flagged; 3/30 held-out benign FPs (10%) | The three FPs are all seeds of held-out task `edit-02`. |
| Injection outcomes | 7/7 cases ran; 0 goals succeeded | Taxonomy remains pending; the local canary sink recorded no receipt. |
| Prompt keyword baseline | 7/7 injection prompts flagged; 0/34 benign prompts flagged | Small, patterned lexical baseline; not a semantic hijack result. |
| Mimicry outcomes | 4/4 deterministic wrapper goals succeeded | Naive and one-call cases were detected; workspace-scope and padding cases evaded both detectors in one run each. |
| Mimicry detector rates | Allowlist 2/4, FPR 0/30; n-gram 2/4, FPR 3/30 (10%) | Single run per strategy; not population estimates. Padding sequence was not calibrated to a successful benign sequence. |
| Ablations on unsuccessful injections | Files 1/7, network 1/7, process 0/7, sequence 1/7, combined 1/7 | These flags are not successful-attack detection rates. Sequence and combined FPR were 3/30 (10%). |
| Operational taxonomy | 7/7 injection cases pending | No injected goal succeeded. |

The detector split held 18 tasks for training, 6 for calibration, and 10 for
test; each task's three seeds stayed in one partition. The measured FPRs are
coarse: calibration used 18 runs, and the n-gram cutoff was 0. The VM outputs
include per-task behavior summaries, scores, and FP causes. The dominant n-gram
FP cause was the `edit-02` sequence's unseen `file_write|network_send`
trigrams; all three seeds were flagged. The exact per-overlay run records are
under ignored `data/vm/`, including reconstructed provenance for the Phase 3
and Phase 5–7 runs because those predated immutable per-overlay provenance.

## Reproduce or refresh reports

Use the commands in `README.md` and `scripts/reproduce.sh` from a host with
adequate memory headroom and a working QEMU/KVM setup. Phase 3 must be refreshed
after changing benign task definitions; phases 5–7 must then use that same
corpus and the preserved model/config locks. For existing generated data:

```sh
make eval-detectors CORPUS_DIR=data/phase3-corpus ATTACK_DIR=data/injection-corpus
ATTACK_DIR=data/injection-corpus make eval-all
```

The actual guest paths and the resolved corpus can differ; check each run's
manifest rather than assuming a default path. Do not mix artifacts from
different corpus IDs or model profiles in one table.
