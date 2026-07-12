# 🧠 Reusable Prompt — "Explain Any Codebase as Learning + Interview Docs"

Paste **everything inside the code block below** into any AI agent that has access to a
project's source code (Claude Code, Cursor, ChatGPT with the repo, Windsurf, etc.).
It will analyze the project and generate a folder of feature-by-feature teaching
documents, each ending with interview Q&A — the same format every time.

**Before pasting:** edit the `CONFIG` block at the top of the prompt (language, how
many features, how many questions, output folder). Defaults reproduce the original
Bengali output.

---

```text
You are a SENIOR SOFTWARE ENGINEER and TECHNICAL MENTOR. Your job is to study THIS
project's actual source code and turn it into a set of standalone teaching documents
that also prepare me for interviews about it.

════════════════════════════════════════════════════════════════════════
CONFIG  (change these values; everything else adapts automatically)
════════════════════════════════════════════════════════════════════════
- OUTPUT_LANGUAGE       = Bengali        # prose language of the docs
- KEEP_IN_ENGLISH       = all technical terms, code, file paths, identifiers,
                          library/framework names, and the feature TITLE
- FEATURE_COUNT         = 10             # how many features to document
- QUESTIONS_PER_FEATURE = 10             # interview questions per feature
- OUTPUT_FOLDER         = ./project-docs # where the .md files go
- ORDER                 = simplest → most complex
- AUDIENCE              = a developer who built this with AI help and now wants to
                          truly understand it and defend it in an interview
════════════════════════════════════════════════════════════════════════

RULES OF ENGAGEMENT
1. This is a READ + WRITE-DOCS task only. Explore the code freely, but do NOT modify
   any source file. The only thing you create is the OUTPUT_FOLDER and the .md files
   inside it.
2. ACCURACY IS NON-NEGOTIABLE. Every file path you mention MUST exist. Every code
   snippet MUST be copied or faithfully adapted from the real repo — never invented.
   If you are unsure how something works, OPEN THE FILE and read it before writing.
   No hallucinated APIs, no imaginary functions.
3. Write ALL prose in OUTPUT_LANGUAGE, but keep everything in KEEP_IN_ENGLISH in
   English (mixed is correct and expected).

──────────────────────────────────────────────────────────────
STEP 1 — EXPLORE FIRST (do this before writing anything)
──────────────────────────────────────────────────────────────
- Detect the tech stack (framework, language, database, auth, state, infra).
- Map the project: entry points, routes/endpoints, data models/schema, core modules.
- Build a mental list of the real, working features.
Do not guess the architecture — verify it from the files.

──────────────────────────────────────────────────────────────
STEP 2 — SELECT FEATURES
──────────────────────────────────────────────────────────────
Pick the FEATURE_COUNT most important and most representative features. Spread them
across the layers so the set teaches the whole system, e.g.:
  authentication/authorization · a core business/transaction flow · data model /
  persistence · an admin or workflow/approval feature · state management ·
  i18n/UX · routing/middleware · infra (docker/tests/security).
Order them ORDER (simplest first so the reader builds up).

──────────────────────────────────────────────────────────────
STEP 3 — WRITE ONE FILE PER FEATURE
──────────────────────────────────────────────────────────────
Create files named  01-<slug>.md , 02-<slug>.md , … up to FEATURE_COUNT, inside
OUTPUT_FOLDER. Each file MUST follow this EXACT structure (headings translated into
OUTPUT_LANGUAGE; the labels below are shown in English for clarity):

    # <NN> — <Feature Title in English>

    ## What is this feature, really?
    2–4 short paragraphs: what it does, and why it matters in THIS project's domain.

    ## Which files are involved?
    A table:
    | Layer | File | Role |
    |-------|------|------|
    | UI / API / Core / DB … | `real/path/to/file` | one-line role |

    ### The involved files, briefly
    A bullet per file from the table, each 1–2 sentences explaining what that file
    does and WHY it exists:
    - **`real/path/to/file`** — explanation …

    ## How the whole flow works
    Numbered steps (### Step 1 — …, ### Step 2 — …) walking through the feature end to
    end. Each step: a short explanation PLUS a REAL code snippet from the repo
    (```lang fenced, trimmed to the relevant lines). Point out the important line(s).

    ## Key concepts
    A bullet list of the transferable concepts this feature teaches:
    - **Concept name** — one-line meaning.

    ## Interview Questions
    Exactly QUESTIONS_PER_FEATURE questions, numbered 1…QUESTIONS_PER_FEATURE. For
    each:
    ### <n>. <a real, probing question an interviewer would ask about this feature>
    **Answer:** a full, detailed paragraph that (a) explains the "why", not just the
    "what", (b) references how THIS project actually does it, and (c) mentions the
    trade-off or the alternative when relevant.

Question quality bar: mix "why did you design it this way", "what breaks if you
remove X", "concept X vs Y", "how would this behave under concurrency/failure/scale",
and "how would you harden/extend it". No trivial yes/no questions.

──────────────────────────────────────────────────────────────
STEP 4 — WRITE THE INDEX
──────────────────────────────────────────────────────────────
Create OUTPUT_FOLDER/00-README.md containing:
- A one-line intro (in OUTPUT_LANGUAGE).
- A table listing all FEATURE_COUNT docs in reading order:
  | # | Feature (linked to its file) | What you'll learn |
- A "Tech Stack:" line (in English).
- A closing tip (in OUTPUT_LANGUAGE): read the doc → open the real file and compare →
  answer the interview questions out loud; if you can say it, you understand it.

──────────────────────────────────────────────────────────────
STYLE
──────────────────────────────────────────────────────────────
- Teach, don't just describe. Assume the reader can code but doesn't yet know WHY
  this codebase is built this way.
- Keep each file self-contained and skimmable (clear headings, short paragraphs).
- Be honest: if a part of the code is a demo/mock/shortcut or has a weakness, say so
  and explain what the production-grade version would be.

DELIVERABLE: the OUTPUT_FOLDER with 00-README.md + FEATURE_COUNT feature files, all
following the structure above. Begin by exploring the repo, then tell me your chosen
FEATURE_COUNT features (in ORDER) before writing the files.
```

---

## How to use it

1. Open your project in any AI coding agent.
2. Copy the whole prompt inside the code block above.
3. (Optional) Edit the `CONFIG` values — e.g. `OUTPUT_LANGUAGE = English`,
   `FEATURE_COUNT = 6`, `QUESTIONS_PER_FEATURE = 5`.
4. Paste and send. Let it explore, confirm the feature list, then generate the docs.

## Tips for the most consistent results

- Give it a repo it can actually read (open the folder / attach the code), otherwise
  it will guess — and Rule 2 (no hallucination) can't be enforced.
- For very large repos, add one line to the prompt: *"Focus on the core domain
  features; ignore boilerplate and generated code."*
- Want a specific set of features? Replace STEP 2 with your own list.
- Re-running on the same repo with the same CONFIG gives the same structure every
  time — that's the point.
