---
name: prompt-architect
description: Engineer high-quality prompts using the REASONS Canvas (Requirements, Entities, Approach, Structure, Operations, Norms, Safeguards), based on Thoughtworks' SPDD methodology and GenAI patterns from martinfowler.com. TRIGGER when the user asks to write, create, design, draft, improve, refactor, or review a prompt for an LLM — including system prompts, agent instructions, prompt templates, or one-shot prompts for code generation, content writing, data extraction, classification, summarization, or analysis. Also trigger when the user mentions prompt engineering or techniques (zero-shot, few-shot, CoT, ReAct, chaining, self-critique), wants to debug a prompt, or asks for "the best prompt" for any task. Trigger for casual requests ("help me write a prompt to...", "make me a prompt that..."). DO NOT TRIGGER for: (a) conceptual/educational questions about techniques — "what is chain-of-thought?" → answer directly; (b) skill-writing tasks — improving a SKILL.md or agent skill file → use create-skill; (c) questions about output a prompt produced, not the prompt itself.
---

# Prompt Architect

Treat every prompt as a first-class artifact: **Specified** (goal, inputs, outputs explicit) · **Reviewable** (peer can predict model behaviour) · **Testable** (named pass and fail examples) · **Versioned** (changes tracked with changelog).

## Workflow

**If the user supplies an existing prompt → Review mode:**
1. Reverse-engineer the REASONS Canvas from the existing prompt
2. Run the quality checklist against it
3. Surface the delta — missing, conflicting, or wrong slots
4. Propose targeted changes — full rewrite only if 3+ checklist items fail
5. Rejoin at Step 5 (self-review) below

**If writing from scratch → follow in order:**

1. **Capture intent.** Restate in one sentence: *"You want a prompt that takes X and produces Y for audience Z, evaluated by criterion W."* If you cannot complete that sentence, ask exactly one clarifying question. Tiebreaker priority when multiple gaps are equal-weight: ① output consumer (human vs. machine-parsed) — determines Structure; ② task complexity (single vs. multi-step) — determines Technique; ③ failure tolerance (prototype vs. production) — determines Safeguards depth.
2. **Fill the REASONS Canvas.** Work through all seven slots. See `references/canvas.md` for slot-by-slot guidance. **Non-nullable: R, S (Structure), S (Safeguards)** — marking any of these N/A requires explicit written justification. E, A, O, N may be N/A when genuinely not applicable.
3. **Pick the technique.** Use the table below. State the choice and a one-line justification.
4. **Assemble the prompt.** Use the output format below exactly.
5. **Self-review.** Run the quality checklist. Apply the two-track failure protocol.

## REASONS Canvas

| Slot | Fill with |
|------|-----------|
| **R** Requirements | Observable outcome, audience, success criterion — no vague verbs like "help with" |
| **E** Entities | Domain vocabulary, jargon, named concepts the model must use correctly |
| **A** Approach | Chosen technique + one-line justification |
| **S** Structure | Input schema, output schema, field-level constraints |
| **O** Operations | Step-by-step task decomposition; the CoT scaffold lives here |
| **N** Norms | Tone, voice, persona, length defaults, formatting conventions |
| **S** Safeguards | What prompt must NOT do; malformed-input behaviour; refusal conditions |

## Technique selection

| Task signal | Technique | When to use |
|---|---|---|
| Single-step, common task | **Zero-shot** | Simple summaries, translations, obvious classifications |
| Unusual format or ambiguous labels | **Few-shot** (2–5 examples) | Custom JSON shapes, domain-specific classification |
| Multi-step reasoning, math, planning | **Chain-of-thought** | Explicit `## Reasoning` then `## Answer` |
| High-stakes correctness, multiple samples affordable | **Self-consistency** | Sample N, majority vote — math and reasoning eval |
| Writing a prompt that generates other prompts | **Meta-prompting** | Prompt-of-prompts with canvas embedded |
| Knowledge outside training or frequently changing | **RAG template** | Context-injection template; a system pattern, not just prompt content |
| Model must reason AND call tools in a loop | **ReAct / Tool-use** | Agent system prompts, tool-calling pipelines; output is an action sequence, not a reasoning trace |
| Output of this prompt feeds the next prompt | **Prompt chaining** | Multi-step pipelines; design constraint is interface compatibility between steps |
| Task benefits from draft → critique → revise | **Self-critique / Reflection** | High-stakes writing, code review; sequential, not parallel like self-consistency |

Default to zero-shot. Move up only when zero-shot fails a checklist item.

## Quality checklist

**Two-track failure protocol:**
- **Self-correctable** (output format, conflicting norms, length): iterate internally, max 2 passes; deliver with a note if still imperfect
- **User-input-required** (goal not testable, vocabulary not primeable): surface the specific failing item, ask exactly one targeted question, wait before drafting

- [ ] **Goal is testable.** Can I name one input that should pass and one that should fail? If not → user-input-required.
- [ ] **Output is parseable.** Could a downstream script consume this without regex acrobatics? If not → self-correctable.
- [ ] **Vocabulary is primed.** Every domain term is in Entities or universally understood. If not → user-input-required.
- [ ] **Technique fits the task.** Approach slot has a one-line justification matching the table. If not → self-correctable.
- [ ] **Failure modes named.** Safeguards lists at least one out-of-scope and one malformed-input case. If not → self-correctable.
- [ ] **Length is honest.** No single canvas slot exceeds 150 words (→ decompose into sub-prompt). Norms has ≤7 rules (→ model trades them off beyond that). Operations has ≤6 steps (→ use CoT or prompt chaining). If violated → self-correctable.
- [ ] **Adversarial sanity check.** Off-topic input, prompt-injection, empty string — degrades gracefully? If not → self-correctable.

## Output format

~~~markdown
# [short-name-in-kebab-case]

## Header
- **Purpose:** one sentence
- **Audience / consumer:** who or what reads the output
- **Prompt type:** system-only | user-template | system+user
- **Technique:** zero-shot | few-shot | CoT | self-consistency | meta | RAG-template | ReAct | chaining | self-critique
- **Target model:** claude-sonnet-4-6 | gpt-4o | model-agnostic | ...
- **Version:** v1.0
- **Changelog:** v1.0 — initial

## REASONS Canvas
- **R:** ...  **E:** ...  **A:** ...  **S (Structure):** ...  **O:** ...  **N:** ...  **S (Safeguards):** ...

## The Prompt
[If Prompt type is system+user, use ### System and ### User template sub-sections]
```
[prompt text, ready to copy-paste]
```

## Example pass
> Must exercise at least one Safeguards or Operations constraint — not the happy path alone.
Input: ...  Expected output: ...

## Example fail
> Must represent the most likely real-world failure mode (from Common pitfalls or Safeguards) — not the most obvious adversarial case.
Input: ...  Expected behavior: ...
~~~

## Using the output (read this before you paste anything)

The artifact this skill produces has three concerns — store it whole, use it selectively, and maintain it at the canvas level.

**Suggested storage layout:**

```
prompts/
  [short-name-in-kebab-case].md     ← the whole artifact lives here
    ├── Header                      (metadata, version, changelog)
    ├── REASONS Canvas              (the design — edit this first)
    ├── The Prompt                  ← paste only this block into your tool
    ├── Example pass
    └── Example fail
```

**Two rules:**

1. **Store the whole document, paste only the prompt block.** The canvas, header, and examples are the engineering record. The `## The Prompt` block is the deployable unit. Keep them together in version control; only the prompt block goes into your LLM tool, API call, or system-prompt field.

2. **When the prompt misbehaves, edit the canvas first, then re-derive the prompt block.** A failure in production is almost always a gap in Requirements or Safeguards — not a wording problem in the prompt prose. Find the failing canvas slot, fix it, then regenerate the prompt block from the updated canvas. Editing the prompt block directly without updating the canvas disconnects design from implementation and the artifact loses its engineering record.

## References

- `references/canvas.md` — slot-by-slot guidance, house rules, common pitfalls, maintenance lifecycle
- `references/examples.md` — five worked examples: extraction (few-shot), classification (CoT), research agent (ReAct), fact-check pipeline (chaining), rejection email (self-critique)
