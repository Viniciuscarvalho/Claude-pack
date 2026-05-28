# Worked Examples

Five compressed canvas examples — one per primary technique. Each highlights the slots most structurally distinctive for that technique.

---

## Example 1 — Extraction (few-shot)

**User asks:** "I need a prompt that pulls company name, role, and start date out of resumes."

**Canvas:**
- R: Extract three fields from a resume; output strict JSON; missing field → null.
- E: "resume" = unstructured text; "role" = job title at most recent employer; "start date" = ISO 8601 month if known.
- A: Few-shot — resume formats vary too much for zero-shot to land the JSON consistently.
- S: Input = raw resume text. Output = `{"company": str|null, "role": str|null, "start_date": "YYYY-MM"|null}`.
- O: 1) Locate most-recent-employer block, 2) read company, 3) read title, 4) read start month, 5) emit JSON.
- N: No prose, no markdown, no explanation — only the JSON object.
- S: If no employment block exists, return all nulls. Never invent a company. If multiple roles tie for "most recent", pick the one listed first.

**Technique justification:** few-shot beats zero-shot because resume layouts are inconsistent and JSON-only outputs are easier to elicit with examples.

---

## Example 2 — Classification (chain-of-thought)

**User asks:** "I want a prompt that decides whether a customer email is a billing issue, a technical issue, or other."

**Canvas:**
- R: Classify into one of three labels; emit reasoning then label, in that order.
- E: "billing" = charges, refunds, invoices, payment methods; "technical" = product not working; "other" = everything else.
- A: Chain-of-thought — three-way ambiguous cases need visible reasoning to be debuggable.
- S: Output = `## Reasoning` (2–4 sentences) then `## Label` (one of three exact strings).
- O: 1) Identify the customer's primary complaint, 2) match to category definitions, 3) if hybrid, pick the one the customer is asking *to be resolved*, 4) emit reasoning, 5) emit label.
- N: Reasoning is plain prose, no bullets. Label is exactly `billing` | `technical` | `other`, lowercase.
- S: If email is empty or non-English, label `other` and note the reason in reasoning. Never invent a fourth category.

**Technique justification:** CoT because hybrid cases need visible reasoning for the support team to audit borderline classifications.

---

## Example 3 — Research agent (ReAct / Tool-use)

**User asks:** "I need a system prompt for an agent that searches the web and returns a sourced summary."

**Key structural difference:** Operations is an action *loop*, not a linear sequence. Structure must define tool schemas, not just output format.

**Canvas:**
- R: Given a research question, produce a 3-bullet summary with inline source citations; halt after max 5 tool-call iterations.
- E: "research question" = user query; "observation" = raw tool result; "iteration" = one reason-act-observe cycle.
- A: ReAct — model must reason and call tools iteratively; output is an action sequence, not a reasoning trace.
- S (★ structurally distinctive):
  - Input = `{question: str}`
  - Tools: `search(query: str) → list[{title, url, snippet}]` · `fetch(url: str) → str`
  - Output = `{"summary": str (3 bullets, each ≤40 words), "sources": list[url]}`
- O (★ structurally distinctive — action loop):
  1. Reason: what information is still missing?
  2. Act: call `search` or `fetch` with a specific query.
  3. Observe: read the tool result.
  4. Update understanding.
  5. If answer is sufficient → emit final output. Else → repeat from step 1 (max 5 iterations).
- N: Cite sources inline with the bullet they support. Prefer primary sources over aggregators. Never fabricate a URL.
- S: If no relevant results after 3 iterations, surface what was found and note the gap. Never invent facts. If the question is ambiguous, ask one clarifying question before starting the loop.

**Technique justification:** ReAct because the agent must decide *which* tool to call and *when* to stop — that decision loop cannot be unrolled into static CoT steps.

---

## Example 4 — Fact-check pipeline (prompt chaining)

**User asks:** "I want a two-step pipeline: first extract all claims from an article, then fact-check each one."

**Key structural difference:** Structure slot defines the *interface contract* between steps — the output schema of Step 1 is the input schema of Step 2. Interface compatibility is the primary design constraint.

**Step 1 — Claim extraction canvas:**
- R: Extract all factual claims from a document as a structured list; each claim must be independently verifiable.
- E: "claim" = an assertable proposition that can be true or false; excludes opinions, recommendations, and predictions.
- A: Zero-shot — extraction of factual claims is well-understood.
- S (★ interface contract):
  - Input = `{document: str}`
  - Output = `{"claims": [{"id": int, "text": str, "location": str}]}`
  - *Step 2 receives this exact JSON as its input — any schema drift breaks the pipeline.*
- O: 1) Read document, 2) identify assertions presented as facts, 3) exclude opinions/recommendations, 4) emit JSON.
- N: Claim text is verbatim or close paraphrase. No editorial commentary.
- S: If document has no factual claims, return `{"claims": []}`. Never add claims not present in the document.

**Step 2 — Fact-check canvas (receives Step 1 output):**
- R: For each claim in the input list, return a verdict and a one-sentence rationale.
- E: "verdict" = `supported` | `refuted` | `unverifiable`; "rationale" = evidence-based one-liner.
- A: Chain-of-thought per claim — each verdict needs traceable reasoning.
- S (★ interface contract):
  - Input = `{"claims": [{"id": int, "text": str, "location": str}]}` ← exact schema from Step 1
  - Output = `{"results": [{"id": int, "verdict": str, "rationale": str}]}`
- O: For each claim: 1) state what would make it true, 2) assess available evidence, 3) assign verdict, 4) write rationale.
- N: Rationale is one sentence, active voice. Never use "it appears" or "seems" — commit to a verdict.
- S: If a claim cannot be assessed with available knowledge, verdict = `unverifiable`. Never invent a source.

**Technique justification:** prompt chaining because the tasks are structurally distinct (extraction vs. reasoning) and separating them makes each step testable in isolation.

---

## Example 5 — Rejection email (self-critique / reflection)

**User asks:** "Write a prompt that drafts a professional but warm rejection email for job applicants."

**Key structural difference:** Operations slot has a mandatory draft → critique → revise cycle. All three phases are non-negotiable — the final output is the *revised* version only.

**Canvas:**
- R: Draft a rejection email that is warm, specific to the role, and under 150 words; output the final version only.
- E: "warm" = empathetic without false hope; "specific" = references the role applied for; "false hope" = language implying future reconsideration when none is intended.
- A: Self-critique — high-stakes communication where first drafts regularly carry tone problems or inadvertent promises.
- S: Input = `{role: str, applicant_name: str, reason_hint: str (optional)}`. Output = email body only, no subject line, <150 words.
- O (★ three mandatory phases):
  1. **Draft:** write the rejection email without self-censoring.
  2. **Critique:** check for (a) false hope signals ("we'll keep your resume on file" unless instructed), (b) word count ≤150, (c) warmth-clarity balance — is the "no" unambiguous?, (d) any legally sensitive language (age, appearance, protected characteristics).
  3. **Revise:** apply all critique findings. Emit the revised version only — do not show the draft or critique.
- N: Address applicant by first name. No corporate jargon. Never promise future consideration unless the caller explicitly provides `keep_on_file: true`.
- S: If `reason_hint` is absent, keep reason vague but genuine. If `applicant_name` is missing, halt and request it before drafting. If critique finds legally sensitive language, remove it without flagging to the user — just fix it.

**Technique justification:** self-critique because first-draft rejection emails reliably fail on either warmth (too cold) or clarity (too much false hope) — a structured internal review loop catches both before delivery.
