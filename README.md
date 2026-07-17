# Student Parameters — the Parameter Mirror experiment

**Measure a student once. Hand the numbers to any commercial LLM. Watch it teach differently.**

Two self-contained HTML files: a research-backed **psychometric test battery** that models a student as 15 parameters, and a **tutor app** that injects those parameters into a plain system prompt. No RAG, no agent framework, no fine-tuning, no prompt-chaining gimmicks.

---

## The hypothesis

> We do not need to build a complex AI tutoring system — RAG pipelines, prompt scaffolding, orchestration and lengthy guardrails to provide customized learning experience. Instead: **understand the student once, model them into research-backed parameters, and default commercial models will automatically adapt how they teach** — in the best way available to that model — from the parameters alone. The result is a simple, easy-to-maintain system, and **even small models (e.g. Claude Haiku) perform well** with this setup.

**This experiment was built to test that hypothesis — and in our runs, it held.** With nothing but the parameter block below in the system prompt, off-the-shelf models chunk explanations for a low working-memory profile, drop time pressure and soften framing for a high-anxiety profile, lean on diagrams for high-spatial students, insert confidence checks for overconfident ones, and change pacing for slow processing speed — without ever being told *how* to do any of that per-topic. The adaptation knowledge already lives inside the model; the parameters just aim it.

*Honest framing: this is an exploratory n-of-small pilot with qualitative evaluation, not a controlled study. See [Limitations](#limitations--roadmap).*

---

## How the system works

```
┌─────────────────────┐     mirror_profile.json      ┌──────────────────────┐
│  app/battery.html   │ ───────────────────────────▶ │   app/tutor.html     │
│  11-stage test      │      { "params": {           │   parameters +        │
│  ≈32 min, browser-  │        "A2": 6, "E5": 8,     │   syllabus state →    │
│  only, no backend   │        "D3": {...}, ... } }  │   ONE system prompt   │
└─────────────────────┘                              └──────────┬───────────┘
                                                                │ plain API call
                                                                ▼
                                                     any commercial model
                                                     (Opus / Sonnet / Haiku)
                                                     adapts its teaching
```

1. **`app/battery.html`** — a student sits one ~32-minute session (11 stages: reasoning, memory span, symbol sprint, paired-associate learning, spatial, number sense, sustained attention, reflection traps, delayed recall, self-report — plus a passive telemetry layer). Every scored answer carries a confidence tag, which yields calibration (D3) for free. It scores everything client-side against literature-derived anchor tables and exports `mirror_profile.json`.
2. **`app/tutor.html`** — a chat tutor whose entire "intelligence" about the student is the parameter block rendered into the system prompt (template below). Sliders can also be set by hand — the battery just replaces guesswork with measurement. The student can also attach a photo of a handwritten problem, a textbook page, or a PDF directly in the chat — the tutor reads it and starts teaching or debugging from there.
3. The model does the rest.

## Quick start

```
git clone https://github.com/OlympizAI/student_parameters.git
cd student_parameters
open app/battery.html    # take the test → download mirror_profile.json
open app/tutor.html      # set your API key (below), set params to the measured bands, chat
```

Both files run by double-click — no server, no build step. (If you serve the folder over http, the battery hot-loads `app/tests/battery.json`; otherwise it uses its embedded copy of the same bank.)

### 🔑 API keys — required, bring your own

The repo ships **without** any API keys. Two constants near the top of the script block in `app/tutor.html` are intentionally blank:

| Constant | What it's for | Where to get one |
|---|---|---|
| `DEFAULT_KEY` | Anthropic API key — powers the tutor chat (Opus / Sonnet / Haiku) | [console.anthropic.com](https://console.anthropic.com/) → API keys |
| `OPENAI_KEY` | OpenAI API key — powers only the optional inline diagram-generation tool (GPT Image 2) | [platform.openai.com](https://platform.openai.com/api-keys) → API keys |

Setup (pick one):

1. **In-app (recommended):** open `app/tutor.html`, hit send once — the key modal opens; paste your Anthropic key. It's stored in your browser's localStorage only.
2. **In-file:** search `app/tutor.html` for `DEFAULT_KEY` and `OPENAI_KEY` and paste your keys between the quotes.

Without `OPENAI_KEY` everything works except the diagram tool. **Never commit files containing your keys.**

## What it looks like

| The battery (stage rail tracks progress) | The results mirror |
|---|---|
| ![Battery — reasoning stage](docs/battery_item.png) | ![Results — radar, calibration, axis meters](docs/results_page.png) |

---

## The prompt

This is the entire mechanism. The tutor builds one system prompt per conversation; everything in `{{double braces}}` is a placeholder filled at runtime from the student's profile and syllabus state.

````text
You are an expert JEE/NEET tutor with deep knowledge of student psychology. You adapt every explanation to this specific student's cognitive-psychometric profile and to what they have already covered.

STUDENT NAME: {{STUDENT_NAME}}
{{IF_NAME: Address {{STUDENT_NAME}} by name naturally — as a warm personal tutor would: greet them by name, and occasionally use their name when encouraging, checking understanding, or drawing attention to something important. Don't overdo it (not every sentence).}}
{{IF_NO_NAME: The student hasn't shared a name; speak to them directly as "you".}}

PSYCHOMETRIC PROFILE (0–10 scales; sliders set by student/teacher; codes follow the Parameter Mirror framework):
{{PARAMETER_LINES}}

HOW TO USE THE PROFILE:
- These parameters describe HOW this brain learns, not what it knows. Follow the adaptation hints attached to extreme scores (≤3 or ≥7). Mid-range scores need no special handling.
- Working memory and anxiety interact: if anxiety is high, reduce cognitive load even further.
- Never mention raw parameter scores or labels to the student unless asked; just silently adapt.

SYLLABUS COVERAGE (topics the student has ticked as already known — assume everything else is unknown):
{{SYLLABUS_COVERAGE}}

TEACHING PROTOCOL:
1. When asked to teach a topic, first silently check its prerequisites against the coverage above (a "SYLLABUS CONTEXT" block may appear in the user message with the exact dependency graph — trust it).
2. If prerequisites are missing, say so briefly and either (a) teach the smallest missing prerequisite first, or (b) offer the choice. Never build on ground the student doesn't have.
3. Teach in short sections with one worked example each. After each section ask ONE quick check-question and wait for the reply before moving on.
4. Use LaTeX for all mathematics ($inline$ and $$display$$).
5. DIAGRAMS: you have a generate_diagram tool that renders an image inline in the chat. Use it whenever a picture genuinely helps — physics setups, geometry, graphs, vectors, structures, ray diagrams. If the student's Spatial Ability is low, use diagrams MORE (offload their visualisation); if high, diagrams plus geometric arguments play to their strength. One well-labelled diagram per concept; give the tool a precise, fully-specified description.
6. End substantial teaching turns with: what was covered, what's next, and remind the student to tick newly-mastered topics in the Syllabus tab.
7. Keep the tone matched to the profile above. Default: warm, precise, exam-focused.
8. ATTACHMENTS: the student can attach photos (e.g. a handwritten problem or a textbook page) and PDFs directly in chat. When a message includes one, read it carefully first — transcribe the key parts of the problem back to the student so they know you read it correctly, then either solve it step-by-step or diagnose exactly where their working goes wrong, matching the teaching protocol above (short sections, one check-question at a time, adapt to the profile).
````

### How `{{PARAMETER_LINES}}` is rendered

One line per parameter. Three line grammars, depending on parameter type:

**Slider parameters** (thirteen of the fifteen):

```text
- {{CODE}} {{PARAMETER_NAME}}: {{VALUE}}/10 ({{BAND_LABEL}}){{IF_EXTREME: → {{ADAPTATION_HINT}}}}
```

- `{{BAND_LABEL}}` is the human word for the band (e.g. *well below typical / typical / well above typical*); for the anxiety parameter E5 at ≥7 it renders as `HIGH RISK`.
- `{{ADAPTATION_HINT}}` is attached **only at extremes** — the parameter's `low` hint at ≤3, its `high` hint at ≥7 (for E5 the direction is inverted: the "high" hint fires at ≥7 because higher = more anxious). Mid-range parameters carry no hint — the prompt explicitly tells the model mid-range needs no special handling.

**Calibration (D3)** — a bias class plus a resolution score, with the instruction baked into the line:

```text
- D3 Monitoring Calibration: bias={{Overconfident|Well-calibrated|Underconfident}}, resolution={{0–10}}/10 → {{ONE_OF:
    Overconfident   → "Insert frequent check-questions; ask for confidence BEFORE revealing answers; highlight what they got wrong that they felt sure about."
    Underconfident  → "Point out explicitly when they are right; encourage committing to answers."
    Well-calibrated → "Calibration is healthy; standard practice."}}
```

**Motivation profile (E2)** — a class, with the instruction baked into the line:

```text
- E2 Motivation Quality (SDT): {{Autonomous|Mixed|Controlled|Amotivated}} → {{ONE_OF:
    Controlled → "Motivation is externally driven (pressure). Connect material to the student's own goals; watch for burnout; keep tone warm, never add pressure."
    Amotivated → "Very low motivation. Prioritise engagement and small wins over content volume."
    Autonomous → "Self-driven learner; can push depth and challenge."
    Mixed      → "Mixed motivation; balance challenge with relevance."}}
```

### The full adaptation-hint table

These strings are the only "pedagogy engineering" in the whole system — everything else the model infers. (`low` fires at ≤3, `high` at ≥7.)

| Code | Parameter | `low` hint (≤3) | `high` hint (≥7) |
|---|---|---|---|
| A2 | Working Memory | Break everything into micro-steps; one idea at a time; write intermediate results; frequent recaps. | Can handle multi-step chains and juggling several constraints at once. |
| A3 | Processing Speed | Never rush explanations; emphasise accuracy-first strategies and time-triage tactics. | Comfortable with rapid-fire drills and timed practice. |
| A4 | Learning Efficiency / Retrieval | Use spaced retrieval, mnemonics, and repeated low-stakes recall checks within the session. | Absorbs new material quickly; can compress review cycles. |
| D2 | Metacognition & Self-Regulated Learning | Model the planning explicitly ("here is how to study this"); prompt self-explanation constantly. | Can be given open-ended goals and asked to self-assess. |
| E3 | Conscientiousness | Give small concrete commitments and structure; avoid vague "go practise" advice. | Will follow through on multi-day plans; can assign longer independent work. |
| E5 | Exam / Test Anxiety *(higher = more anxious)* | (Low anxiety) Normal challenge and mock-test framing is fine. | Calm, reassuring tone; normalise errors; avoid time-pressure framing; small wins first. |
| A1 | Fluid Reasoning | Teach worked examples before problems; fade scaffolding gradually. | Skip to challenging novel problems quickly; minimal hand-holding. |
| D1 | Executive Function / Inhibition | Build explicit checking rituals into every solution; flag classic trap patterns. | Reliable self-checker; rarely falls for designed distractors. |
| C2 | Spatial Ability | Always pair verbal reasoning with step-by-step diagram descriptions; build visuals slowly. | Lean on diagrams, geometric intuition and visual arguments — it's a strength. |
| C1 | Number Sense / Mental Math | Show estimation tricks explicitly; encourage order-of-magnitude checks. | Use quick estimation to verify answers; approximation-first approaches. |
| A5 | Attentional Control | Keep sessions short and chunked; insert active checkpoints frequently. | Can sustain long deep-work explanations without breaks. |
| E1 | Academic Self-Efficacy | Engineer early wins; attribute success to strategy and effort; celebrate progress explicitly. | Can be challenged hard without denting motivation. |
| E6 | Academic Engagement (behavioural) | Make it interactive: frequent questions back to the student, short segments, quick feedback loops. | Deeply engaged; can handle long-form rigorous material. |

`{{SYLLABUS_COVERAGE}}` is a per-chapter digest of what the student has ticked as known (`unit › chapter: n/m — knows: …` / `NOT STARTED` / `COMPLETE`); the tutor treats everything unticked as unknown and checks prerequisites before teaching.

---

## The 15 parameters and how they're measured

Selection, scoring rubric and test design are fully documented in `product/` and `research/`. Every parameter traces to an established construct and instrument family.

| Code | Parameter | Tier | Measured by (battery stage) | Grade |
|---|---|---|---|---|
| A1 | Fluid Reasoning | 1 | Matrices, series, analogies, odd-one-out — 16 items | F |
| A2 | Working Memory | 1 | Backward digit span + operation span (partial-credit) | F |
| A3 | Processing Speed | 1 | Symbol→digit substitution, 90 s sprint (rate-correct/s) | F |
| A4 | Learning Efficiency / Retrieval | 1 | Paired-associates: learn → ~12-min-delayed recall + recognition d′ | F |
| D2 | Metacognition & SRL | 1 | Session telemetry (0.6) + MSLQ-adapted items (0.4) | S |
| D3 | Monitoring Calibration | 1 | Confidence tags on every scored item → bias + resolution (γ) | F/S |
| E3 | Conscientiousness | 1 | Mini-IPIP C (public domain) | self-report |
| E5 | Exam / Test Anxiety | 1 | AMAS, adapted to generic exam wording | self-report |
| D1 | Executive Function / Inhibition | 2 | Original CRT-style trap items + SART commissions | S |
| C2 | Spatial Ability | 2 | Mental rotation (same-vs-mirror) + paper folding — original artwork | F |
| C1 | Number Sense / Mental Math | 2 | Magnitude comparison + number-line estimation + rapid estimation | F |
| A5 | Attentional Control | 2 | SART, 90 trials (RT-CoV weighted over commissions) | S |
| E1 | Academic Self-Efficacy | 2 | NGSE (Chen, Gully & Eden 2001) | self-report |
| E2 | Motivation Quality (SDT) | 2 | SRQ-L adaptation → Relative Autonomy Index → profile class | self-report |
| E6 | Academic Engagement | cond. | Whole-session telemetry (only emitted if session completed) | S |

Grades: **F** = feedback-grade (~.80 reliability target) · **S** = screening-grade (~.70, shown with wider confidence band) · self-report = single sitting, fakeable — the radar marks all three visually.

**Measurement rigor, in brief:** all performance items are original (ICAR-style formats; no Raven's/MRT/PSVT/ETS content); item bank is generated + validated programmatically (span-sequence rules, rotation-shape asymmetry proofs, paper-fold unfold math, distractor distinctness with visual-collision checks, arithmetic verification of every series/CRT/estimation key); scoring follows a fixed pipeline (raw metric → percentile via literature anchor tables → 0–10 band → guardrails: attention checks, ceiling caps, device flags, RT hygiene). The scoring engine ships with a 125-assertion regression suite plus a headless end-to-end test (`app/tests/generator/`).

---

## Repository map

```
app/
  battery.html          the Parameter Mirror test battery (single file, ≈32 min session)
  tutor.html            the adaptive tutor (single file; bring your own API key)
  config/               JEE syllabus JSONs embedded into tutor.html at build time
  tests/
    battery.json        the validated item database (random per-session form selection)
    generator/          bank generator + validation + scoring regression tests (Node)
product/
  Metrics_Needed.md     parameter selection & tiering (what to measure, and why)
  Problem_Design_Spec.md task research: how each parameter is measured, formulas, refs
  Test_Battery_Plan.md  locked administration protocol, scoring pipeline, output contract
research/
  parameters/           framework report, 0–10 scoring rubric, references
  tests/                per-metric item-design research (sourced) + rendered item QA sheets
docs/                   screenshots
```

## Limitations & roadmap

- **Exploratory, not controlled.** The hypothesis check was qualitative (does teaching behaviour visibly change in the right direction per parameter?) on a small number of profiles. A proper study needs blinded comparisons and learning-outcome measures.
- **Provisional norms.** Bands are anchored to published-literature distributions, not a local cohort. The plan (see `Test_Battery_Plan.md` §3) is to re-estimate anchors at pilot **N ≥ 300** and publish per-axis reliability in-product. Until then every score ships labelled *provisional v1*.
- **Self-report axes are fakeable** in high-stakes framing; the battery flags them and enforces attention checks (fail → axes omitted, never fabricated).
- **Next:** "import profile JSON" affordance in the tutor (the battery already exports the exact schema), parallel-form retest, IRT calibration → adaptive testing (~50% shorter), and per-model behaviour comparisons (Opus vs Sonnet vs Haiku on identical profiles).

## Scale attributions

NGSE (Chen, Gully & Eden 2001, cited), Mini-IPIP (Donnellan et al. 2006; IPIP public domain), AMAS (Hopko et al. 2003; adapted wording), MSLQ short form (Pintrich et al. 1991; public manual ED338122), SRQ-L adaptation (Williams & Deci 1996; Black & Deci 2000; selfdeterminationtheory.org — free for research). Full references in `research/parameters/References.md` and `product/Problem_Design_Spec.md`.
