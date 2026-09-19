# A verification layer for patient-facing LLM output

**Nemotron does not answer the user. It decides whether another model's answer is safe to show.**

Built for SteelHacks XIII — NVIDIA Nemotron track ("Beyond the Chatbot").

---

## The problem

A digital-health company has already built a patient-facing symptom feature and cannot ship it. What blocks them is not accuracy — it is that nobody can tell legal, compliance, or a health-system buyer *why* any individual answer was safe to show. "The model is usually fine" does not survive a procurement review.

A black-box safety classifier does not fix this. It returns a score, and a score cannot tell you which specific thing was wrong with a specific answer, or let you move the threshold without retraining.

## What this is

A gate. Nemotron performs a structured audit of every draft, and a deterministic policy layer in Python turns those findings into a decision. Each withheld answer names the rule that withheld it, and the risk threshold is a parameter rather than a model's mood.

```
0  person describes a symptom
1  NEMOTRON            triages it — acuity + red flags, before any draft exists
2  a non-NVIDIA model  writes draft advice
3  NEMOTRON            audits the draft — what is wrong with it?
4  gate() in Python    applies 8 fixed rules to those findings
5                      pass -> show it.  block -> withhold, redirect to real care
```

Nemotron reports findings. Code makes the decision. Medical is the wedge; the same architecture applies anywhere fluent, confident, wrong output is expensive — legal, claims, financial guidance.

### Why Nemotron is load-bearing here

It occupies two structural positions and answers the user in neither. Remove it and there is no gate at all: the drafter's output goes straight to the patient, which the demo will show you if you set the verifier dropdown to **None**.

| Role | Model | Mode |
|---|---|---|
| Router (intake triage) | `nvidia/nemotron-3-super-120b-a12b` | `thinking=False` |
| Verifier (draft audit) | `nvidia/nemotron-3-super-120b-a12b` | `thinking=True` |
| Corrupter (builds the eval set) | `nvidia/nemotron-3-ultra-550b-a55b` | `thinking=False` |
| Drafter (the audited model) | `openai/gpt-4o-mini` via OpenRouter | — |

Router and verifier share a checkpoint with role-specific prompts, because several smaller Nemotron slugs appear in the public catalogue but are not entitled on this account (see `notebooks/` cell 1). What matters for the architecture is that Nemotron performs a structural job, not which checkpoint serves each role.

---

## Evidence

**43 labeled cases: 10 clean controls and 33 corrupted drafts**, built as 10 clinical scenarios crossed with up to 4 injected flaw types. Ground truth is free by construction — every flaw was deliberately injected, so no human grading is involved. The corrupter (Ultra) is never the judge (Super); a model grading its own handiwork measures nothing.

The 33 corrupted rows are **not** 33 independent trials. They are clustered within 10 scenarios, and every v1 miss is the same failure mode appearing in 6 of them.

### Schema v1 baseline

| Injected flaw | Caught |
|---|---|
| Dosing error | 10/10 |
| Invented drug | 10/10 |
| Acuity downgrade | 7/7 |
| Omitted red flag | **0/6** |
| **Corrupted drafts caught** | **27/33 (82%)** |
| **Clean drafts shipped** | **10/10** |

### The comparison that matters is coverage, not recall

Same prompt, same schema, same `gate()`. Only the judge changes.

| Judge | Corrupted caught | Clean shipped |
|---|---|---|
| Nemotron 3 Super, v1 | 27/33 | **10/10** |
| Nemotron 3 Super, v2 | 31/33 | 8/10 |
| GPT-4o-mini, v1 | **32/33** | **2/10** |

GPT-4o-mini has better recall and is useless as a gate. It blocked 8 of 10 safe answers — and 5 of those 8 were blocked as "high harm potential" on the *clean emergency drafts*, the ones that correctly say call 911. It is rating how sick the patient is, not how unsafe the answer is.

That conflation is why no threshold setting rescues it. Policy cannot un-flag a finding the judge insisted on, so its whole curve sits pinned to the left edge of `evidence/policy_curve.png`. **A gate that withholds 80% of safe answers does not ship a safer feature; it blocks the feature a different way, and the patient gets nothing.** Recall alone cannot tell those two systems apart, which is why coverage is reported beside it everywhere in this repo.

### The policy layer is not decoration

If the model's own risk score were the only gate — which is what a black-box classifier gives you — here is what the same assessments produce:

| Gate | Corrupted caught | Clean shipped |
|---|---|---|
| Block on `risk_tier == high` only | 16/33 | 10/10 |
| Block on high + medium | 24/33 | 10/10 |
| **The shipped gate** | **27/33** | 10/10 |

**11 of the 27 catches came from a specific named finding rather than from the danger rating.** The clearest single row: on the corrupted ACS draft, Nemotron flagged 1300 mg of aspirin, correctly explained that it is roughly four times the standard dose — and set `risk_tier: "medium"`, reasoning that the draft still tells the patient to call 911. The gate blocked it anyway, because `dosing_or_drug_error` is checked before the risk rating is consulted. Any system gating on a single model-produced score ships that draft.

### Coverage on real drafter output

The 10 clean controls are hand-written ideal drafts, so "9/10 shipped" is measured on the friendliest clean set that exists. Scored against 12 fresh benign scenarios drafted by the actual drafter, the gate shipped **12/12** with zero false blocks.

### Reproducing the numbers

Every figure above is regenerated from the scored results by a cell in the notebook — none is typed by hand. A consistency check compares the notebook's prose against the live run and fails loudly when they diverge. (It caught exactly that during preparation; see `docs/FINDINGS.md`.)

### Known limits

- Corrupter and judge are different models but the same family (Ultra vs Super).
- Single sample per row at temperature 0. No variance estimate, no repeats.
- Latency was measured at n=3 on a shared endpoint and the two arms did not separate; no latency claim is made.
- `pipeline()` hard-stops on emergency intake, so in production the verifier never sees the 5 emergency scenarios that the offline eval does audit. The eval measures the gate in isolation; the intake stop is defence in depth.
- No clinician has reviewed any of these cases.

---

## Four failures, one principle

In short:

1. **Verdict collapse.** Asked for `pass` / `revise` / `escalate` directly, Nemotron returned `"revise"` for both a safe draft and a potentially lethal one. Same label, opposite meanings.
2. **Routing contradiction.** Classified a textbook heart attack as `"emergency"` with correct red flags, then set `path="fast"` in the same response.
3. **Numeric misjudgment.** Passed a 975 mg aspirin dose and called it appropriate in its rationale. Prompting does not fix arithmetic.
4. **Miscalibrated risk score.** Flagged a 4x aspirin overdose, explained it correctly, rated the draft medium risk.

All four are the same finding: the clinical assessment was right and the mechanical step layered on top of it was wrong.

> **Never ask the model for a field you intend to act on mechanically. Ask for observations; compute decisions.**

There is also a documented *schema* gap, distinct from a judgment failure: v1 caught 0/6 omitted red flags because every v1 field describes something **present** in the draft, and that harm is something **absent**. The fields were empty because the answer was empty — the question was never asked. No amount of prompt tuning on v1 fixes that. Schema v2 adds `required_elements_missing`, which took omitted red flags from 0/6 to 4/6 at a cost of one additional false block.

Read v2 honestly: only one of the three new catches actually fired on the new rule. The other two fired on `missing_red_flags` — a rule that already existed in v1 and fired on neither of them. Adding a field changed how Nemotron populates the *other* fields, so the intervention is not cleanly additive.

---

## Demo

The last cell launches a Gradio app on a public URL. Three things to try, in order:

1. **`chest tight for 20 min, left arm achy, sweating`** — flagged as an emergency at intake, so nothing drafts advice at all. The drafter is never called.
2. **`headaches most afternoons for three weeks`** — genuinely ambiguous. Watch the gate ladder: eight rules evaluated top to bottom, the one that fires highlighted, everything below it greyed out as never reached.
3. **The same input with the verifier set to `None`** — the ablation. Raw unchecked output, shown because nothing was checking rather than because anything confirmed it was safe.

Swapping the verifier dropdown runs the cross-model comparison live, on the judge's own input, instead of claiming it on a slide.

---

## Credits

Kyu Han — design, architecture, evaluation methodology, clinical scenarios, corruption specifications.
Claude (Anthropic) — code review, debugging, demo front end.
Nemotron 3 Super and Ultra, GPT-4o-mini via OpenRouter — test subjects.

**Decision support only. Not medical advice. No clinician has reviewed these cases, and nothing here should be used to make a decision about anyone's health.**
