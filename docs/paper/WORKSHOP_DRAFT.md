# Tripwire: Measuring OS-Level Observability of Prompt Injection in Tool Agents

**Workshop paper draft — preliminary measurements, utility-confounded**

Authors: Tripwire contributors  
Status: The Phase 3–7 pipeline ran in a provisioned guest with the pinned tiny
model. These preliminary measurements do not establish injection efficacy
because the model achieved none of the injection goals.

## Abstract

Tool-using language-model agents translate untrusted text into file, process,
and network operations. This paper asks which indirect prompt-injection
outcomes are distinguishable from operating-system behavior, and how an
adaptive attacker can stay within a task's benign behavior envelope. We build
Tripwire, a cgroup-scoped eBPF tracer that records kernel metadata and joins
events to logged tool calls, then compare per-task behavior allowlists and
tool/syscall sequence models. We report a 102-attempt benign corpus with 18
successful tasks (17.65%), seven injection attempts with zero successful goals,
four deterministic local
fake-canary wrapper probes with two observed evasions, and five feature
ablations. All seven injection taxonomy rows remain pending. Fetch tasks
succeeded in 3/30 attempts, but the padding probe was not calibrated to a
successful benign sequence. These preliminary results do not establish a
population-level adaptive evasion rate.

## 1. Introduction

Indirect prompt injection can change how an agent uses tools after it reads
untrusted file, page, or tool output. The central question is not whether an
agent can be hijacked in general, but what an OS-level monitor can distinguish
once an attack changes the agent's behavior. We study both conspicuous actions
(unexpected process, out-of-scope file access, externalized fake canary) and
semantic changes that may use ordinary file operations.

Research questions:

1. Which attack outcomes produce a detectable deviation in OS-level behavior?
2. What attack success can an adaptive attacker achieve while matching the
   benign tool, path, host, syscall, and sequence envelope?
3. Which signal groups contribute detection, and what task utility is lost?

Contributions: a reproducible cgroup-scoped syscall tracer and tool-call join;
task-held-out OS-only detector and ablation machinery; and an explicitly
limited measurement of how model utility constrains injection and mimicry
claims. A general attack taxonomy is not claimed from this run.

## 2. Threat model

The experiment runs in an isolated Ubuntu VM with a local open-weights model,
fake secrets, generated canary tokens, and an HTTP sink bound to guest
loopback. The attacker controls injected content and can adapt to disclosed
features and thresholds; it cannot modify the guest kernel, tracer, or
detector. No host files, credentials, or external targets are in scope.

The monitor records syscall, process, path, and connection metadata. It does
not collect HTTP payloads or infer semantic intent from file contents. The
full threat model is in `docs/THREAT_MODEL.md`.

## 3. System

Tripwire attaches tracepoint-based CO-RE eBPF programs to a cgroup v2 subtree,
emits structured events through a ring buffer, and reports ring-buffer drops.
The userspace collector records monotonic and wall-clock timestamps, process
identity, cgroup identity, paths or socket destinations, and syscall return
codes. A JSONL tool logger provides call windows and process roots. The joiner
uses call timing, process ancestry, and a configured late-event grace; it
reports ambiguous and orphan events.

The monitor extracts path, binary, host, syscall, and ordered
`(tool, syscall-class)` features. Baselines are a per-task-type allowlist and
an unsmoothed n-gram novelty score. Task-level splitting keeps repeated runs
of a task in one partition. The split seed, FPR targets, sequence order, and
feature groups are recorded in `config/`.

## 4. Experimental method

### 4.1 Benign behavior

The refreshed benign suite has 34 declarative tasks across five task types,
each run with three fixed seeds (102 total). With `qwen3:0.6b-q4_K_M` and
Ollama 0.40.7, 18/102 task checks succeeded (17.65%): edit 9/18, fetch 3/30,
search 0/18, summarize 6/18, and tests 0/18. Weak-model failures remain in
the corpus and are a severe utility confound. The join assigned 188/1,582
behavioral events and orphaned 1,375 (86.92%), with 19 ambiguous events and
zero ring-buffer drops; 2,390 stats snapshots are excluded from that
denominator. The model manifest digest is retained with the Phase 3 artifacts.

### 4.2 Injection suite

The suite covers out-of-scope fake-canary reads, local canary submission,
unexpected process execution, writes outside the workspace, poisoned
summaries, and wrong-but-valid edits. An attack counts as successful only when
its declared goal is observed. Cases originate in `injections/`. One case
adapts AgentDojo's workspace email-exfiltration scenario to use a fake canary
and loopback sink; it is not a canonical AgentDojo run or directly comparable
benchmark result. The full benchmark's task tools are not integrated.

### 4.3 Adaptive attacker

The current reproducible attacker includes naive shell-child, one-call
adaptive, workspace-file-scope, and sequence-padding wrappers. Each reached its
local fake-canary goal in one run. The naive and one-call profiles were flagged
by both detectors; the file-scope and padding profiles evaded both at the
configured 1% and 5% targets. This is four deterministic tool-wrapper probes,
not a learned optimizer or model takeover. The padding length was not calibrated
to a successful benign sequence, and only 3/30 benign fetch tasks succeeded.

### 4.4 Metrics and ablations

Report task success and detector rates at 1% and 5% target FPR, with measured
held-out FPR, calibration/test sample counts, and a cause log for every false
positive. Split by task, not run. The refreshed split held 18 train, 6
calibration, and 10 test tasks, with 18 calibration and 30 benign test runs.
Ablations include files, network, process, sequence, and combined signals. The
combined score is the configured arithmetic mean of allowlist and sequence
novelty. The calibration set is small, so measured 1% and 5% FPRs are coarse.

## 5. Results

These guest measurements are preliminary. Phase 5 attack rows all failed
their declared goals, so their detector flags are not successful-attack TPR.
The successful wrapper probes are a separate four-run experiment.

| Measure | Refreshed preliminary result |
|---|---|
| Benign task success | 18/102 (17.65%) with Qwen3 0.6B Q4_K_M |
| Phase 3 join quality | 188/1,582 assigned; 1,375 orphaned (86.92%); 19 ambiguous; 0 drops |
| Injection goal success | 0/7; successful-attack TPR is undefined |
| Allowlist flags on failed attack rows | 1/7; held-out benign FPR 0/30 |
| N-gram flags on failed attack rows | 1/7; held-out benign FPR 3/30 (10%) |
| Prompt keyword baseline | 7/7 injections flagged; 0/34 benign prompts flagged |
| Mimicry wrapper goals | 4/4 local canary goals reached; one run per strategy |
| Mimicry detector outcomes | 2/4 detected by both; 2/4 evaded both; FPR 0/30 allowlist and 3/30 n-gram |
| Ablation flags on failed attack rows | Files 1/7, network 1/7, process 0/7, sequence 1/7, combined 1/7 |
| Utility loss under blocking | Not measured; blocking is not implemented |

All seven taxonomy cases remain pending. The prompt keyword baseline is a
lexical smoke result, not evidence of semantic injection detection. The
four-probe mimicry result is not a population estimate, and its padding case
was not calibrated to a successful benign sequence. Full run details and
limitations are in `reports.md` and the phase notes.

## 6. Taxonomy

Use the operational rules in `docs/taxonomy.md`. A case remains pending if its
goal did not succeed or lacks calibrated OS-only detector verdicts. The
generated `taxonomy-results.json` has seven pending rows. Current
hypotheses are not findings because none of the injected goals succeeded.

## 7. Limitations

- Small, hand-authored task and injection suites; external validity is unknown.
- Model weakness or inability to follow a task can confound attack success.
- The current adaptive attacker is hand-built and deterministic.
- Connection metadata exposes endpoints, not HTTP query/body semantics.
- Tool-call logs contain arguments, but the primary detector baseline uses OS
  features; cross-layer argument features are future work.
- The process-tree/time-window join can orphan startup and late events.
- No enforcement or task-utility loss under blocking has been measured.
- No claim is made about kernel versions beyond the pinned guest.

## 8. Related work

AgentSight [1] uses eBPF to inspect encrypted LLM traffic for semantic intent,
observe kernel effects, and correlate the streams. Tripwire is complementary
in scope: its current detector excludes application payloads and evaluates
OS-behavior features alone. The successful-injection evidence needed to
measure that detector's limits has not yet been obtained. Tetragon [3]
documents eBPF observability and runtime enforcement for process, file, and
network activity. AgentDojo provides a dynamic benchmark for tool-agent prompt
injection [2]. Tripwire adapts one workspace email-exfiltration scenario to a
fake canary and a loopback-only sink; its full simulated tool suite is not
integrated because those tools do not directly produce the OS-visible behavior
or tool-call records Tripwire measures. System-call sequence models have used
short-range syscall correlations to characterize normal process behavior [5],
and later work demonstrated mimicry attacks against host IDSs [6]. Recent work
also studies mimicry against provenance-graph detectors [4]. Tripwire's
attacker is currently a narrow local probe rather than a general optimizer; it
does not reproduce those papers' attack methods or results.

## References

1. Y. Zheng, Y. Hu, T. Yu, and A. Quinn, “AgentSight: System-Level
   Observability for AI Agents Using eBPF,” Proceedings of the Workshop on
   Practical Adoption Challenges of ML for Systems (PACMI 2025), pp. 110–115,
   co-located with SOSP 2025,
   doi: [10.1145/3766882.3767169](https://doi.org/10.1145/3766882.3767169).
   [ACM record](https://dl.acm.org/doi/10.1145/3766882.3767169),
   [arXiv version](https://arxiv.org/abs/2508.02736).
2. E. Debenedetti, J. Zhang, M. Balunović, L. Beurer-Kellner, M. Fischer,
   and F. Tramèr, “AgentDojo: A Dynamic Environment to Evaluate Prompt
   Injection Attacks and Defenses for LLM Agents,” Advances in Neural
   Information Processing Systems 37, Datasets and Benchmarks Track, 2024.
   doi: [10.52202/079017-2636](https://doi.org/10.52202/079017-2636).
   [Proceedings](https://proceedings.neurips.cc/paper_files/paper/2024/hash/97091a5177d8dc64b1da8bf3e1f6fb54-Abstract-Datasets_and_Benchmarks_Track.html),
   [arXiv:2406.13352](https://arxiv.org/abs/2406.13352).
3. Cilium, “Tetragon: eBPF-based Security Observability and Runtime
   Enforcement,” project documentation, accessed 2026-10-30; source pinned to
   release v1.7.1.
   [Documentation](https://tetragon.io/docs/),
   [v1.7.1 source](https://github.com/cilium/tetragon/tree/v1.7.1).
4. A. Goyal, X. Han, G. Wang, and A. Bates, “Sometimes, You Aren't What You
   Do: Mimicry Attacks against Provenance Graph Host Intrusion Detection
   Systems,” NDSS 2023, doi:
   [10.14722/ndss.2023.24207](https://doi.org/10.14722/ndss.2023.24207).
   [NDSS paper page](https://www.ndss-symposium.org/ndss-paper/sometimes-you-arent-what-you-do-mimicry-attacks-against-provenance-graph-host-intrusion-detection-systems/).
5. S. Forrest, S. A. Hofmeyr, A. Somayaji, and T. A. Longstaff, “A Sense of
   Self for Unix Processes,” Proceedings of the 1996 IEEE Symposium on
   Security and Privacy, pp. 120–128, 1996, doi:
   [10.1109/SECPRI.1996.502675](https://doi.org/10.1109/SECPRI.1996.502675).
   [IEEE record](https://ieeexplore.ieee.org/document/502675/).
6. D. Wagner and P. Soto, “Mimicry Attacks on Host-Based Intrusion Detection
   Systems,” Proceedings of the 9th ACM Conference on Computer and
   Communications Security (CCS 2002), pp. 255–264, 2002, doi:
   [10.1145/586110.586145](https://doi.org/10.1145/586110.586145).
   [Author-hosted paper](https://people.eecs.berkeley.edu/~daw/papers/mimicry.pdf).
