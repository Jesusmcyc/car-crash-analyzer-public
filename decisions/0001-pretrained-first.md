# ADR 0001 — Pretrained-first: the pipeline is built on pretrained models before any fine-tune

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

The Car Crash Analyzer project (architecture doc) has two core ML components: a damage detector and a segmenter. The roadmap (sec 12) plans to fine-tune RT-DETRv2 on the CarDD dataset in a later phase (Phase 4 / M5).

The architectural question at the start of the project is: **do we build the pipeline directly with fine-tuned weights, or do we assemble it first with pretrained models and swap the weights in later?**

## Decision

**The pipeline (M0–M4) runs exclusively on pretrained models.** The CarDD fine-tune happens in M5 as a parallel milestone, and the resulting weights are dropped into the backend through an environment variable (`DETECTOR_WEIGHTS`) without touching any application code.

## Consequences

### Positives

- **Parallel work:** while the pipeline is being built and deployed (M0–M4), the fine-tune can be prepared in a separate notebook without either blocking the other.
- **Isolated risk:** training problems (convergence, overfitting, loss of mAP on rare classes) do not contaminate the debugging of the app.
- **Obvious fallback:** if the fine-tune does not improve on the pretrained baseline, the system works just the same with the original weights. The downside of the fine-tune is zero.
- **Reproducibility for the reviewer:** anyone can spin up the demo without access to the custom checkpoint, using only public weights.

### Negatives

- **M0–M4 detections are not specific to CarDD.** RT-DETRv2 pretrained on COCO does not know categories such as `crack` or `broken_lamp` directly; in M2, generic COCO classes are mapped to the domain categories through a dict (it is an approximation; M5 fixes it).
- **M2/M3 metrics are not final.** The M0–M4 demo must not be presented with accuracy claims until M5 is closed.

## Alternatives considered

1. **Wait for fine-tuned weights before starting the backend.** Rejected — it blocks M0–M4 for days or weeks depending on training pace, and leaves the deployment risk undiscovered.
2. **Train from scratch.** Out of scope.

## References

- architecture doc sec 12 (Roadmap).
- design doc sec 3 → M0–M5.
