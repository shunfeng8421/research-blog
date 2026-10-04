# Designing Conditional Binders at Competition Scale: A 20/20 Postmortem

*Shiqiang Chen — independent researcher (2026-10-04)*

I just completed a protein-design competition round at the **20/20 submission limit** (Track 3, Anthropic × Adaptyv 2026, Challenge 1: conditional EGFR binder). Here is what actually happened — the metrics that lied, the one that mattered, and the pipeline changes that got us across the finish line.

## The score that lied

Our automated factory — RFdiffusion (skeletons) → ProteinMPNN (sequences) → ColabFold (folding validation) — produces sequences scored by **pLDDT** (fold confidence), **solubility**, and a **composite**. The library filled with pLDDT 92–97 binders, composite scores 88–92. By the internal metrics, these were outstanding proteins.

Then we validated **binding** with multimer **ipTM**. The result was brutal: **pLDDT 92–97 collapsed to ipTM < 0.2** against EGFR. High fold confidence did not translate into binding. We had produced a library full of beautifully-folded proteins that mostly don't bind their target.

**Lesson 1**: in de novo binder design, *single-chain fold score (pLDDT) is a necessary but not sufficient filter*. The only metric that decides binding is multimer ipTM. Build your pipeline around ipTM, not pLDDT.

## The filter that mattered: novelty

Even a valid binder is rejected if it's not **de novo / zero-shot** — not a known protein, not a trivial modification. The competition's novelty check is strict.

Our hard-won finding: **sequence length was the novelty killer**. Our library was full of 67-residue binders; virtually all of them scored **2/4 novelty and were rejected** (in one batch, 8/8 rejected; another 1/11 passed). The only design class that reliably passed was **40–60 residue short binders** (10/11 in our second batch passed).

**Lesson 2**: for competition submissions, the *submission class* (length, novelty profile) is as important as binding quality. We changed the factory's contig from `[A1-171/0 40-70]` to `[A1-171/0 40-60]` to bias toward novelty-passing short binders.

## What a working pipeline looks like

```
RFdiffusion (target-aware skeleton, short contig)
  → ProteinMPNN (design only the binder chain)
  → ColabFold multimer (ipTM ≥ 0.5 is the submission gate)
  → Novelty check (short binders pass)
  → Submit (respect the per-challenge limit / 24h cooldown)
```

Key details that mattered:
- **Design only the binder chain** (`--pdb_path_chains B`), not the target.
- **ipTM ≥ 0.5 is the submission gate**, not composite score.
- **Short binders (40-60 aa)** pass novelty; longer ones don't.
- **Never reuse a known binder as a starting point** — it's de novo only.

## Why this matters beyond the competition

The two failures above — trusting pLDDT, ignoring novelty class — are the same failure from two angles: **validating against the wrong metric** and **optimizing for what the generator likes instead of what the downstream gate requires**.

This echoes a lesson from my security work: a repository named "sandbox" or "gateway" is not a guarantee of isolation or control — the metric (name, fold score) flatters; the real gate (binding, wired authentication) decides.

## Data & reproducibility

- 20/20 submissions achieved, Track 3 limit.
- Candidate library: 292 binders, top composite 92.2 (pLDDT 97.1 / pTM 0.819 / 98.5% soluble).
- Pipeline changes documented; ipTM screening tool (`multimer_screen.py`) is reusable for any target.
- Competition results will be published openly by the organizer (wet-lab validation of selected designs).

No AI-generated fluff. These are measured outcomes and reproducible pipeline changes.

---

*Next: Challenge 2 opens tomorrow. The factory is prepped, the ipTM screen is battle-tested, and the lesson is embedded: **validate against the gate that actually decides, not the metric that flatters.***
