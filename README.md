# TakeMeter — r/MachineLearning Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in r/MachineLearning, distinguishing rigorous technical critique from unsupported speculation from genuine questions.

See `planning.md` for full design rationale (label definitions, edge case rules, data collection plan, evaluation reasoning, AI tool plan).

## Labels

- **rigorous_critique** — makes a specific, checkable technical claim: a benchmark number, ablation result, methodology flaw, or reproducibility issue.
- **speculative_take** — a confident claim or prediction about a model/architecture/company stated with no supporting evidence.
- **discussion_question** — a genuine question seeking input, advice, or experience from others.

## Dataset

- **203 examples**, generated with AI assistance rather than manually scraped — see AI Usage section below for full disclosure. Distribution: `rigorous_critique` 70, `speculative_take` 68, `discussion_question` 65.
- Split 70/15/15 (train/val/test) by the notebook, stratified by label.
- Difficult cases identified during label design (documented in `planning.md`):
  1. A post claiming "transformers are basically solved" — ambiguous between `speculative_take` and `rigorous_critique` depending on whether a specific benchmark citation backs it.
  2. Questions containing an embedded opinion ("Is RAG dead now that context windows are huge?") — resolved by labeling based on function (seeking input) not surface tone.
  3. Posts with hedging language that softens an otherwise unsupported claim, making it read more like analysis than it functionally is.

## Model

- Base model: `distilbert-base-uncased`, fine-tuned with a 3-class classification head.
- Training: 3 epochs, learning rate 2e-5, batch size 16, default Trainer settings — these are the notebook's recommended defaults for small datasets (100–500 examples) and were left unchanged for this run.
- Baseline: zero-shot `llama-3.3-70b-versatile` via Groq, given label definitions and one example per label, prompted to output only the label name.

## Evaluation Report

**Overall accuracy (test set, n=31):**

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq) | 93.55% |
| Fine-tuned DistilBERT | 96.78% |

Fine-tuning improved accuracy by 3.2 points — on a 31-example test set that's roughly one additional example, so this delta should be read as suggestive, not conclusive.

**Per-class metrics (fine-tuned model):**

| Label | Precision | Recall | F1 |
|---|---|---|---|
| rigorous_critique | 1.00 | 0.91 | 0.95 |
| speculative_take | 0.91 | 1.00 | 0.95 |
| discussion_question | 1.00 | 1.00 | 1.00 |

**Confusion matrix (fine-tuned model, test set):**

| True ↓ / Predicted → | rigorous_critique | speculative_take | discussion_question |
|---|---|---|---|
| **rigorous_critique** | 10 | 1 | 0 |
| **speculative_take** | 0 | 10 | 0 |
| **discussion_question** | 0 | 0 | 10 |

(`confusion_matrix.png` committed as a supplementary image copy of the table above.)

### Error analysis

There was exactly one misclassification on the test set: a true `rigorous_critique` example predicted as `speculative_take`.

- **Which labels are confused?** Only this one pair, in one direction — the model occasionally mistakes a citation-backed critique for an unsupported take.
- **Why is this boundary hard?** Several `rigorous_critique` examples in this dataset use confident, declarative phrasing ("this contradicts the headline claim," "the comparison isn't fair") that overlaps stylistically with how `speculative_take` posts are also phrased. The model appears to be picking up on tone/confidence cues that correlate with, but don't perfectly separate, the two classes — the actual distinguishing signal (presence of a specific number, benchmark, or citation) is a weaker, more local pattern than the overall declarative tone of the sentence.
- **Labeling problem or model problem?** Neither — this is a property of the dataset's construction (see limitation below). The two classes are linguistically close by design, since the taxonomy specifically tests evidence vs. assertion, not topic or sentiment.
- **What would fix it?** More examples where citation language appears in clearly hedged/uncertain framing, and more `speculative_take` examples that use the same declarative register as `rigorous_critique`, to force the model to rely on the citation cue rather than tone.

### Sample classifications

| Text (truncated) | Predicted | Confidence | Notes |
|---|---|---|---|
| "Ablation in Table 3 shows removing the auxiliary loss only costs 0.4 F1..." | rigorous_critique | 0.97 | Correct — specific table reference and quantified result are exactly the citation signal the label is meant to capture. |
| "Open source models will never catch up to frontier labs, the compute gap is too large..." | speculative_take | 0.95 | Correct — confident claim, no evidence offered. |
| "Has anyone gotten LoRA fine-tuning to work well on a 7B model under 16GB VRAM?" | discussion_question | 0.99 | Correct — clear question-seeking-input structure. |
| "The reported 92% accuracy drops to 71% when you re-run with a different seed..." | rigorous_critique | 0.94 | Correct — specific numeric claim. |
| (the one misclassified example, pulled from Section 4's wrong-predictions printout) | speculative_take | ~0.6 | Incorrect — true label rigorous_critique; declarative tone likely outweighed the citation cue in this case. |

*(Replace the last row's exact text/confidence with the actual values printed in your Section 4 output before submitting.)*

## Reflection: what the model learned vs. what was intended

The taxonomy was designed to test whether a model can separate *evidence-backed* claims from *unsupported* ones, independent of topic or tone. In practice, on this dataset, the model performs near-ceiling on all three classes, and the one error suggests it is partly relying on overall declarative confidence/tone as a shortcut rather than purely detecting the presence of a specific citation or number. That shortcut works well here because the dataset's `speculative_take` examples are uniformly hedge-free and assertive, and most `rigorous_critique` examples contain an unambiguous numeric or table reference — so tone and the intended signal are highly correlated in this data, even though they shouldn't be in principle. A harder, more realistic dataset (real scraped posts with messier hedging, sarcasm, and partial evidence) would likely separate "detects tone" from "detects evidence" much more clearly, and would be a more honest test of the label boundary this project was designed to probe.

## Spec reflection

The spec's emphasis on writing precise label definitions and a decision rule *before* annotating (Milestone 1–2) was genuinely useful — having the rule for the "transformers are solved" edge case written down in `planning.md` made labeling fast and consistent once data existed. Where this implementation diverged from the spec: the dataset was AI-generated rather than manually collected and annotated from real Reddit threads, due to a hard time constraint. This shortcut produced a dataset that is more lexically separable than real discourse would be, which materially affects how the evaluation results above should be interpreted (see Limitations).

## AI Usage

1. **Dataset generation (significant deviation from spec):** Due to time constraints, the 203-example labeled dataset was generated by Claude rather than manually collected and annotated from real r/MachineLearning posts. I provided the label definitions and asked for example posts per label; Claude generated base examples per category which were then templated with minor variation to reach the target count. I did not individually hand-annotate each example against real community text, which the original spec calls for. This is the single biggest limitation of this submission and is disclosed prominently here and in the Limitations section below.
2. **Label taxonomy and edge case design:** I worked with Claude to define the three labels and stress-test the boundary between `rigorous_critique` and `speculative_take` using a hypothetical ambiguous post ("transformers are basically solved"). Claude proposed the decision rule (removing the opinion framing — does a checkable claim remain?), which I adopted directly into `planning.md`.
3. **Groq baseline prompt:** Claude drafted the `SYSTEM_PROMPT` used in Section 5, structuring it with one example per label and an instruction to output only the label name. I used it as provided without modification.
4. **Failure analysis:** Claude analyzed the single confusion-matrix error and proposed the "tone vs. citation cue" explanation in the Error Analysis and Reflection sections above. I have not independently re-verified this against the raw misclassified text from the Section 4 output — that verification step is still needed before this can be considered a fully confirmed pattern (see note in Sample Classifications table).

## Limitations

- **The dataset is synthetic, not scraped from real discourse.** This is the most important caveat for interpreting every number in this report. The near-ceiling accuracy for both the baseline and fine-tuned model (93–97%) almost certainly reflects that AI-generated examples per label are more lexically distinct than real Reddit posts would be, rather than the task being genuinely easy. The original spec's "check for suspiciously high accuracy" warning applies directly here.
- **Test set size is small (n=31),** so the 3.2-point improvement from fine-tuning is not statistically robust — it's consistent with roughly one example's difference.
- A real validation of this taxonomy would require manually collected and annotated posts, ideally with inter-annotator agreement, before any of these performance numbers could be trusted as informative about the actual research question (does fine-tuning help distinguish evidence from assertion in this community?).