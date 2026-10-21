# CLAUDE.MD -- Doctoral Research Project (Psychology & Public Health)

<!-- Keep this file under ~200 lines (official guidance) — Claude loads it every session. -->

**Project:** Emotional and psychological well-being of professionals who search for disappeared persons in Colombia (doctoral thesis)
**Institution:** Universidad Pública de Navarra (UPNA) — PhD programme in Health Sciences
**Discipline:** Psychology and public health (psychotraumatology, occupational mental health, epidemiology)
**Branch:** main

**Language.** Talk with the author in Spanish. Manuscripts, reviewer reports and code comments are in academic English.

---

## Scope Discipline

**Do exactly what was asked — nothing adjacent.** Do not add README files, build scripts,
`.gitignore` edits, helper utilities, or extra tooling that was not requested. If an addition
looks valuable, **list it as a suggestion at the end** and let the user decide.

A request for a codebook is a request for a codebook. Delivering a codebook plus a README plus
build scripts plus gitignore edits means the user now has to review four things to accept one,
and the usual outcome is that all four get thrown away.

**Before adding anything not named in the request, ask.** One line is cheaper than a revert.

---

## Core Principles

- **Plan first** -- enter plan mode before non-trivial tasks; save plans to `quality_reports/plans/`
- **Verify after** -- when a task changes an `.R` file, run it and confirm the output before reporting done; when it touches a manuscript, re-convert the `.docx` and check the result
- **Single source of truth** -- the Word manuscript (`.docx`) is authoritative for text; the R scripts are authoritative for every number, table and figure. A number in the manuscript must be traceable to an R output
- **Quality gates** -- nothing ships below 80/100
- **[LEARN] tags** -- when corrected, save `[LEARN:category] wrong → right` to [MEMORY.md](MEMORY.md)

Cross-session context lives in [MEMORY.md](MEMORY.md); past plans, specs, and session logs are in [quality_reports/](quality_reports/).

**How we verify** — the references and rules that carry the verification discipline:

- [`verification-ladder.md`](.claude/references/verification-ladder.md) — the seven rungs, from *qualify the checker* to the external oracle, and how the review loop converges.
- [`external-oracle-process.md`](.claude/references/external-oracle-process.md) — running an independent frontier-model referee (Claude Code → another vendor's model via the Oracle CLI; current targets in its §2) and adjudicating what it returns.
- [`provenance-and-ground-truth.md`](.claude/references/provenance-and-ground-truth.md) — naming and pinning your oracles, classifying divergence, and the clean-room boundary.
- [`review-fencing.md`](.claude/rules/review-fencing.md) — reviewer independence is a property of the environment, not an instruction: a neutral copy outside the checkout, prior verdicts withheld, and the answer keys the repo already commits fenced off.
- [`release-engineering.md`](.claude/references/release-engineering.md) — shipping research software: message and silent-resolution censuses, frozen feature matrices for ports, hash-claimed inherited tests, and downstream consumers pinned by commit SHA.

**How we write** — [`writing-with-ai.md`](.claude/rules/writing-with-ai.md): internal vs external-facing documents, why a model cannot make its own output stop reading as model output, and the human-readable standard for anything with your name on it.

**Theory work** — [`theory-proving.md`](.claude/references/theory-proving.md): proof contracts, portfolio search with isolated explorers, counterexample-hunting your own lemmas, adversarial audits, and the rule that an AI-generated proof is a claim, not a theorem.

**The laws** — [`research-agent-laws.md`](.claude/references/research-agent-laws.md): 21 laws for running agents on research infrastructure, each paid for by a real incident.

**How we remember** — the record lives in the repo, not the transcript:

- [`progress-reports.md`](.claude/rules/progress-reports.md) — GitHub issues as defect memory, `quality_reports/` as work memory, `MEMORY.md` as lesson memory.
- [`issue-ledger.md`](.claude/rules/issue-ledger.md) — every shipped bug, deferred improvement and blocking owner decision gets a GitHub issue, closed by hand when the fix lands; the evidence standard and the closing comment.
- [`repo-hygiene.md`](.claude/rules/repo-hygiene.md) — **scratch must not become main.** Enforced by `check-repo-hygiene.py` on every commit.

Nothing clears work until it has a row in [`quality_reports/qualification/LEDGER.md`](quality_reports/qualification/LEDGER.md) — run [`/vaccinate`](.claude/skills/vaccinate/SKILL.md) to put one there.

---

## Research Context

- **Population:** professionals and staff involved in the search for disappeared persons in Colombia (forensic, psychosocial, legal and field teams). Small, identifiable population: treat every output as re-identification-sensitive ([`confidential-data.md`](.claude/rules/confidential-data.md)).
- **Study types in the thesis:** quantitative cross-sectional survey, psychometric validation, qualitative, mixed methods, and a scoping review.
- **Reporting guidelines (EQUATOR):** STROBE (cross-sectional), COSMIN (psychometric validation), COREQ / SRQR (qualitative), GRAMMS (mixed methods), PRISMA-ScR (scoping review).
- **Style:** APA 7th edition unless the target journal says otherwise.
- **Target journals:** profiles in [`journal-profiles.md`](.claude/references/journal-profiles.md) (section "Psychology & Public Health"); discipline defaults in [`discipline-cards.md`](.claude/references/discipline-cards.md).
- **Causal methods:** the owner ruling in [`meta-governance.md`](.claude/rules/meta-governance.md) stands — no guidance on DiD, IV, RD, synthetic control, event studies or matching-as-identification.

---

## Folder Structure

```
mi-proyecto/
├── CLAUDE.MD                    # This file
├── .claude/                     # Rules, skills, agents, hooks
├── Bibliography_base.bib        # Centralized bibliography
├── Figures/                     # Figures and images
├── scripts/                     # Utility scripts + R analysis code (numbered stages in scripts/R/)
├── output/                      # Generated tables, figures, logs (created by the code)
├── quality_reports/             # Plans, session logs, review reports, decision records
├── explorations/                # Research sandbox (see rules)
├── templates/                   # Session log, quality report templates
├── master_supporting_docs/      # Papers and supporting documents
└── Slides/ Quarto/ Preambles/ docs/   # Template leftovers (LaTeX/Quarto) — not used in this project
```

---

## Working with Word manuscripts

Agents cannot read `.docx` directly. To review a manuscript, convert it to Markdown in the session scratchpad (never in the repo):

```bash
pandoc "manuscript.docx" -t gfm --track-changes=all -o "<scratchpad>/manuscript.md"
```

- Reviewers read the converted file; findings cite section headings and quoted text, not line numbers of the conversion.
- Do not edit a `.docx` unless asked. When asked, use the `docx` skill and prefer tracked changes so the author accepts or rejects each one in Word.

---

## Commands

```bash
# R analysis (R must be on PATH — Rscript is not on PATH on this machine yet)
Rscript scripts/R/00_run_all.R

# Word → Markdown for review
pandoc "file.docx" -t gfm --track-changes=all -o "<scratchpad>/file.md"

# Quality score (R scripts)
python scripts/quality_score.py scripts/R/file.R

# Backtest: is the repo internally consistent and currently true?
# (surface-sync + skill-integrity + model-versions + links + spec-conformance + staleness + repo-hygiene + derived-counts + ledger-coverage + hook-battery)
# Run this after ANY change. Also runs in pre-commit and CI.
./scripts/backtest.sh
```

---

## Quality Thresholds (advisory)

| Score | Checkpoint | Meaning |
|-------|------|---------|
| 80 | Commit | Good enough to save |
| 90 | PR | Ready for deployment |
| 95 | Excellence | Aspirational |

Enforced by `/commit` (halts + asks for override) **and** — once you run `./scripts/install-hooks.sh` — by a real git pre-commit hook (`.githooks/pre-commit`) that runs the backtest gate suite (the hook battery only when a hook, its settings or the battery itself is staged; CI always runs everything) plus the quality (≥80) gate on every commit. Bypass sparingly with `SKIP_QUALITY_GATE=1` or `--no-verify`. Word manuscripts are not auto-scored; they are reviewed with the skills below.

---

## Skills Quick Reference

The full table of all skills lives in [README.md](README.md#skills-claudeskills). Most-used for this project:

- **Manuscripts / review:** `/review-paper` (`--peer PTRP|JTS|EJPT|BMCPH|RPSP`) `/seven-pass-review` `/respond-to-referees` `/verify-claims` `/proofread` `/humanize` `/submission-disclosures`
- **Data / reproducibility (R):** `/data-analysis` `/review-r` `/audit-reproducibility` `/diagnose` `/replication-package` `/capture-environment` `/power-analysis` `/disclosure-check`
- **Research / writing:** `/interview-me` `/lit-review` `/research-ideation` `/preregister` `/grant-proposal` `/data-management-plan`
- **Verification / rigor:** `/vaccinate` `/challenge` `/oracle-review` `/adjudicate-review` `/differential-audit` `/blast-radius` `/verify-artifact` `/credible-claims` `/deep-audit`
- **Meta / workflow:** `/commit` `/learn` `/new-skill` `/checkpoint` `/context-status` `/coauthor-brief` `/triage-inbox` `/issues`

Slide, LaTeX and Quarto skills remain installed but are not part of this project's workflow.

---

## Current Project State

| Study | Type | Manuscript (.docx) | R scripts | Status |
| --- | --- | --- | --- | --- |
| [Study 1] | [e.g., scoping review] | [file] | [scripts/R/...] | [status] |
