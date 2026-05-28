# Canvas, House Rules, Pitfalls, and Maintenance

## Slot-by-slot guidance

### R — Requirements
What outcome does this prompt need to produce? Who is the audience of the output? What does success look like in concrete, observable terms? Replace vague verbs ("help with", "improve") with verbs that have testable outputs: "classify", "extract", "summarize in N sentences", "rewrite at reading level X".

### E — Entities
The domain vocabulary the model must use correctly. List key nouns, jargon, named concepts, and any term where the wrong synonym would be a defect. This is **priming**: building shared vocabulary so the model is not guessing what your words mean.

### A — Approach
Which technique fits this task, and why? Pick one from the technique selector in SKILL.md. State the choice and a one-line reason. If the task is hybrid (e.g., extraction + reasoning), say so and pick the dominant technique.

### S — Structure
The shape of input and output. Specify:
- Input schema (what the caller passes in, including any `{{placeholders}}`)
- Output schema (JSON, markdown sections, plain prose, table, etc.)
- Field-level constraints (lengths, enums, required vs. optional)

For **ReAct / Tool-use** prompts, the Structure slot must also define tool schemas (name, parameters, return type). For **prompt chaining**, the output schema here *is* the input schema for the next step — document it as the interface contract explicitly.

If the output will be parsed by code, use machine-readable formats (JSON, fenced code blocks) over prose.

### O — Operations
The steps the model should execute internally to go from input to output. Write steps explicitly even in zero-shot — doing so surfaces ambiguity in Requirements.

For **chain-of-thought**, this section *is* the reasoning scaffold.
For **ReAct**, this section is an action loop: Reason → Act (tool call) → Observe → repeat until done or max_iterations reached.
For **self-critique**, this section must include three sequential phases: Draft → Critique → Revise. Skipping any phase breaks the technique.

### N — Norms
Tone, voice, persona, formatting conventions, length defaults, style choices. When a norm shows up in three or more prompts for the same project, lift it into a reusable Norms block — do not re-derive it each time (the **codified commands** pattern).

### S — Safeguards
Failure modes and guardrails. Spell out:
- What the prompt must **not** do (out-of-scope topics, formats to avoid, hallucination traps)
- How the model should behave when input is malformed, empty, or adversarial
- Refusal conditions and the exact wording of the refusal
- Privacy, safety, or compliance constraints

A prompt without explicit Safeguards is a prompt waiting to fail in production.

---

## House rules (apply by default)

1. **Priming** — every prompt declares its vocabulary up front (the Entities slot exists for this reason). Do not let the model guess what domain words mean.
2. **Design-first** — for non-trivial prompts, fill the canvas before writing prose. The canvas is the design; the final prompt is the implementation.
3. **Codified commands** — when a constraint appears in three or more prompts for the same project, lift it into a shared Norms block and reference it.
4. **Anchored documentation** — every prompt artifact carries a Header recording: who it serves, what it replaces, and one example that should pass. This is what makes the prompt reviewable next quarter.
5. **Harvested learnings** — when a prompt fails in production, capture the failure as a new Safeguard line. Prompts get stronger over time instead of forgetting their scars.

---

## Common pitfalls to flag back to the user

When reviewing an existing prompt, watch for these root causes of bad output:

- **No success criterion** — prompt says "help with X" instead of naming an observable output.
- **Buried instructions** — the most important constraint is in sentence eight of a paragraph; surface it as a section header.
- **Conflicting norms** — "be concise" and "explain your reasoning thoroughly" in the same prompt. Pick one or partition explicitly.
- **No failure path** — prompt assumes well-formed input. Real input is rarely well-formed.
- **Examples that disagree with instructions** — few-shot examples that subtly violate the rules stated in prose. The model follows examples over instructions; reconcile them.
- **Mixed output formats** — asking for JSON but with a "feel free to add commentary" clause. Pick one mode.

---

## Maintenance lifecycle

Ship the first draft only when the quality checklist passes. After that:

- New failure observed in production → add a line to Safeguards, bump minor version, add changelog entry.
- New domain term introduced upstream → add it to Entities, bump minor version.
- Behavior the prompt produces three times in a row that you wish it stopped → add an explicit Norm forbidding it.
- Breaking change to output format or technique → bump major version.

**Versioning:** major.minor only. Major = breaking output format or technique change. Minor = additive Norms, Safeguards, or Entities update.

**Harvested learnings:** if a prompt has not been edited in six months and is still in production, audit it — that is suspicious, not impressive.
