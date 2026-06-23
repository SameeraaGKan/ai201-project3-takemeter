# TakeMeter — Planning

## Community
r/MachineLearning. Discourse here ranges from rigorous technical pushback (citing ablations, benchmark issues, reproducibility problems) to confident unsupported predictions about where the field is headed, to genuine technical questions from practitioners. This range makes it a good fit: the same surface topic (e.g. "transformers are done") can be argued well or asserted with nothing behind it, and the community itself cares about that distinction — sloppy claims get called out in comments constantly.

## Labels

- **rigorous_critique**: The post makes a specific, checkable technical claim — citing a benchmark number, ablation result, methodology flaw, or reproducibility issue.
  - Example: "The reported 92% accuracy on this benchmark drops to 71% when you re-run with a different random seed and no early stopping."
  - Example: "Ablation in Table 3 shows removing the auxiliary loss only costs 0.4 F1, which suggests the architecture contribution is doing far less work than the abstract implies."

- **speculative_take**: A confident claim or prediction about a model, architecture, or company's future stated with no supporting evidence — assertion, not argument.
  - Example: "LLMs are basically a dead end for real reasoning, we're going to need a totally different architecture within 2 years."
  - Example: "Open source models will never catch up to frontier labs, the compute gap is just too large to close."

- **discussion_question**: A genuine question seeking input, advice, or experience from others — not an assertion at all.
  - Example: "Has anyone actually gotten LoRA fine-tuning to work well on a 7B model with under 16GB VRAM?"
  - Example: "What's the current best practice for handling class imbalance in a multi-label setting with ~50 classes?"

## Hard edge cases
A post like "transformers are basically solved, nothing new is coming in this direction" sounds like a speculative_take by tone, but if the author backs it with a specific benchmark-saturation citation, it becomes rigorous_critique.

Decision rule: if removing the confident/opinionated framing still leaves a specific, checkable claim or piece of evidence, label it **rigorous_critique**. If removing the framing leaves nothing — just an assertion — label it **speculative_take**.

A second hard case: questions that contain an embedded opinion, e.g. "Is RAG basically dead now that context windows are huge, or am I missing something?" Decision rule: if the post is primarily soliciting other people's views/experience (ends in a question, invites disagreement), label **discussion_question** even if it contains a take — the function of the post is asking, not asserting.

## Data collection plan
Source: r/MachineLearning post/comment text, restricted to discourse about model architectures, papers, benchmarks, and industry trends (excluding pure memes/off-topic). Target ~65-70 examples per label to keep balance under the 70% ceiling. If a label undershoots after the first collection pass, pull more comment threads specifically from contentious paper-discussion posts (these reliably produce rigorous_critique and speculative_take side-by-side).

## Evaluation metrics
Accuracy alone is insufficient because the three labels are not equally easy to distinguish — rigorous_critique vs. speculative_take both involve declarative, often emotionally loaded language, so a model could get high accuracy on discussion_question (which has a distinct question-mark/seeking-input signature) while failing to separate the other two. Per-class F1 and a confusion matrix are necessary to see whether errors concentrate on one label pair.

## Definition of success
All three per-class F1 scores ≥ 0.70, with the fine-tuned model beating the zero-shot Groq baseline by a meaningful margin (not just within noise of a few points), since the task is "ambiguous tone vs. cited evidence" — exactly the kind of distinction a general-purpose prompt tends to do unevenly. "Good enough for deployment" would mean the rigorous_critique vs. speculative_take boundary specifically holds up, since that's the actual analytical claim of the project.

## AI Tool Plan
- **Label stress-testing**: Asked Claude to generate borderline posts between rigorous_critique and speculative_take to pressure-test the decision rule above before committing to the taxonomy.
- **Annotation assistance**: Given time constraints, the training/test dataset for this run was generated with AI assistance rather than manually scraped and labeled one-by-one — disclosed fully in the README AI usage section. Real deployment would require manually annotated posts per the original spec.
- **Failure analysis**: Plan to paste the fine-tuned model's wrong predictions back to Claude after Section 4 runs, ask it to flag a systematic pattern, then verify manually before writing up the evaluation report.
