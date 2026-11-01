# Phase 8: reproducible release and writeup

## Release artifacts

- `scripts/reproduce.sh` orchestrates Phase 0, 3, 5, 6, and 7 from a fresh
  launcher host. It provisions QEMU guests without host mounts, checks a
  conservative free-memory headroom rule before each launch, and uses the
  preferred pinned model config unless `MODEL_CONFIG` is set.
- The launcher accepts `GUEST_MEMORY_MIB`, `GUEST_SWAP_MIB`, `GUEST_DISK_GIB`,
  and `GUEST_TIMEOUT_SECONDS`. Its 2x memory rule is a local safety heuristic, not
  a guarantee of model fit; available-memory and VM sizing remain recorded in
  the QEMU provenance. The direct Python VM runner enforces the same rule, so
  bypassing `scripts/reproduce.sh` does not skip the check.
- `LICENSE` uses the MIT License. No AgentSight, Tetragon, or AgentDojo source
  code was copied into Tripwire.
- `docs/paper/WORKSHOP_DRAFT.md` is a workshop draft with preliminary results. Verified
  primary sources are linked, including foundational syscall-sequence and
  mimicry work. One AgentDojo scenario is adapted; full benchmark integration
  and successful injection measurements remain follow-up work.
- On 2026-10-30, the related-work claims and identifiers were checked against
  the AgentSight arXiv preprint, NeurIPS proceedings for AgentDojo, official
  Tetragon documentation, the NDSS paper record for provenance-graph mimicry,
  and IEEE Xplore for syscall-sequence modeling. DOI links were added for
  AgentDojo and the NDSS paper. AgentSight's ACM page returned HTTP 403 during
  this check; its venue, pages, and DOI were cross-checked against its arXiv
  record and bibliographic index. The draft now states that Tripwire has not
  yet obtained successful-injection evidence for its OS-only detector claim.

## Measured implementation status

The 2026-11-01 refreshed corpus contains 34 tasks × 3 seeds (102 attempts)
and 18 successful checks (17.65%) using the pinned `qwen3:0.6b-q4_K_M`
profile. All seven injection goals failed. Four deterministic wrapper probes
reached their local fake-canary goals; two were not flagged, but this is only a
single-run controlled probe and the padding profile was not calibrated to
successful benign behavior. All taxonomy labels remain pending. The fresh
corpus join orphan rate was 86.92%. See the phase notes and `reports.md` for
measured counts and their limits.

The development host does not satisfy the guest environment checks, so
acceptance is evidenced by the disposable Ubuntu guest, not a host-side
`make check`. Full Phases 3–7 used a cached-model guest base. A blank-VM
end-to-end README reproduction, a useful-utility corpus, successful injected
goals, repeated calibrated adaptive campaigns, and blocking utility remain
open research/release work.

## Refreshed release status (2026-11-01)

The current source completed Phase 0–2 smoke acceptance, a full 34-task Phase
3 corpus, and Phases 4–7 evaluations in disposable KVM guest overlays. The
corpus used the explicit tiny model profile and achieved 18/102 task checks
(17.65%). All seven injection goals failed; four deterministic wrapper probes
achieved their local canary goals, with two observed detector evasions. The
latter are single-run probes, and one padding strategy was not calibrated to
successful benign sequence behavior. All injection taxonomy labels remain
pending. See `reports.md` for the measured tables and `progress.md` for
remaining research work.

The full-pipeline guest runs reused a cached, previously provisioned model
base, so they do not establish a fresh-VM reproduction from the README. The
runner now disables an inherited one-shot guest service in cloud-init's early
`bootcmd`, before installing the current repository archive. It also writes
immutable `provenance-<overlay>.json` records alongside the compatibility
`provenance.json` latest-run pointer, preventing later overlays from
silently replacing prior-run provenance. Earlier Phase 3 and Phase 5–7
provenance files were reconstructed from the saved invocation and manifests
and are marked as reconstructed. A blank-VM end-to-end reproduction remains
open.
