# Project: LP Evaluation Multi-Agent

This repo evaluates a landing page (LP) through the eyes of a specific target visitor persona,
using an orchestrator + three **generic** specialist subagents in `.claude/agents/`.

The three specialists are interchangeable evaluation workers. They own **no fixed area** of
their own — each one simply scores the LP against the single evaluation axis it is handed. All
the thinking lives in the orchestrator: from the persona and the LP it decides *which* three
top-level evaluation axes matter most for this visitor, then fleshes each one out into concrete,
persona-specific criteria, and hands each finished axis to one specialist. The specialist judges
the LP against exactly that axis — nothing more, nothing less.

This keeps "Breakdown & Delegate" the real thinking step: the orchestrator chooses the axes,
details them, and delegates; the specialists do exactly what they are told.

## Default behavior

When asked to evaluate, review, or audit a landing page (by URL):

1. **Always ask the user for the target visitor persona first** — every single time, before routing to the orchestrator.
   Use `AskUserQuestion` with a question such as:
   "Who is the target visitor for this LP? Please describe them — their role, goals, and what they care most about when landing on this page."
   Do not proceed until you have a persona description from the user.

2. Once you have both the URL and the persona description, route the work to `lp-orchestrator`.
   Pass both values in the prompt.

3. Do not evaluate the LP directly in the root session.

4. Show the orchestrator's report to the user **as returned** (full Markdown, including each axis's findings, actions and evidence). Do not condense it; at most add a short pointer to the weakest axis above it.

## Model assignment

- **lp-orchestrator**: uses `opus` (thinking-heavy coordination — choosing the axes and detailing them per persona)
- **lp-specialist-a/b/c**: use `sonnet` (focused single-axis scoring, exactly as instructed)

## The three slots

There are three evaluation slots — `lp-specialist-a`, `lp-specialist-b`, `lp-specialist-c` —
and they are **identical generic workers**. Nothing about *what* they evaluate is fixed. Each run,
the orchestrator decides the three top-level axes and assigns one to each slot.

The specialists never change and are never rewritten by the orchestrator. What changes each run is
the **evaluation axis** (name, core question, and detailed criteria) the orchestrator writes
and passes into each slot.

## How evaluation works

The orchestrator:
1. Fetches the LP once (for its own axis design).
2. **Breakdown & Delegate (thinking step)** — in two moves:
   - **Choose the three axes**: from the persona and LP, decide the three top-level evaluation
     axes that matter most for this visitor (see the selection rules below).
   - **Detail each axis**: expand each chosen axis into persona-specific criteria using the
     **two-pass ("両取り") method** below.
3. Dispatches the three generic specialists in parallel, passing each slot its complete axis plus
   the **LP URL** in the prompt (each specialist fetches the page itself). It does **not** write or
   edit the specialist definition files.
4. Collects their responses and merges them into one Markdown report.

### Choosing the three axes

The orchestrator picks the axes freshly each run. The three axes together must:

- be the **three most decision-relevant dimensions** for *this* persona on *this* LP — the things
  that most determine whether this visitor says yes or leaves;
- be **mutually distinct** — minimal overlap, so each specialist covers different ground;
- together **cover the visitor's path** from landing on the page to acting, so nothing that
  matters for this persona is left unexamined.

A common, reliable shape is *comprehension → credibility → action* (does the visitor understand
it, believe it, and act on it), and it is a fine default. But it is **not mandatory**: replace any
axis with a more decision-critical dimension when the persona or LP calls for it (e.g. price
justification, competitive differentiation, safety/compliance, onboarding effort). Give each axis
a short, clear name — that name is what the specialist and the final report use.

### Two-pass ("両取り") method for detailing each axis

For each chosen axis, the orchestrator thinks in two passes and combines them into the detailed
criteria:

1. **Persona-first pass** — before leaning on the LP, ask "for this persona, on this axis, what
   *should* ideally be checked?" Produce the ideal detailed criteria from the persona's goals,
   worries, and decision drivers. This keeps the criteria from missing things the LP omits.
2. **LP-informed pass** — then read what the LP actually contains, and concretize *the criteria
   only*: anchor each criterion to the real element on the page it should be judged against (e.g.
   point the refund criterion at the pricing block, the proof criterion at the testimonial strip).
   **Stop at anchoring.** Do **not** rule on whether the LP passes, fails, addresses, or omits a
   criterion, and do **not** label anything present or absent. That present/absent verdict — and how
   heavily it counts — is the specialist's judgment to make from the same LP content, not yours.
   Your job is to sharpen the yardstick, not to take the measurement; pre-deciding it here would
   collapse the specialist into a rubber stamp.

The combined result for each axis is: an **axis name**, a **core question** (what this axis
fundamentally measures), and **3–5 concrete criteria** that together *are* what the specialist
judges on. The criteria name *what to measure and where to look*, never the resulting score or
finding.

## Shared rules for all evaluation subagents

- **One axis per subagent.** Each specialist judges only the single axis it was handed; never comment on another slot's axis.
- **Scores are integers 1–5.** The overall score is the average of the three, to one decimal.
- **Evidence must be verbatim.** Every "Evidence" quote must be copied exactly from the LP,
  not paraphrased. If the needed content is missing from the LP, say so instead of inventing it.
- **No speculation.** Base every judgment on facts present in the LP content.
- **Output language: English**, plain and clear. Subagents return only their Markdown section
  (no preamble or closing remarks); the orchestrator assembles the final report.
- `WebFetch` returns a summary, not the raw page, so "verbatim" is not guaranteed. After the specialists return, the orchestrator sends all quotes to `lp-evidence-checker` (haiku, `curl` + `grep` against the raw HTML) and flags any quote it cannot find word for word.
- Each subagent fetches the LP itself from the URL it is given (with WebFetch), so its evidence quotes come straight from the page.
