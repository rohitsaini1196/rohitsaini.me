# Writeup prompt

Paste the block below into an agent running **inside a project's own repo**. It returns a
finished blog post plus the projects-page entry for rohitsaini.me.

---

You are working inside a code repository. I write at rohitsaini.me and I want to publish a post
about this project. Read the repo first, then produce the two deliverables at the bottom.

## Step 1: dig through the repo before writing anything

Do not write from the README alone. READMEs describe the finished thing; I want the part where it
did not work yet. Go find:

- `git log` from the beginning. Look for reverts, rewrites, commits that delete a whole subsystem,
  commit messages written in frustration, long gaps followed by a big change. Those are the story.
- Issues, TODOs, and `FIXME`/`HACK` comments still in the code, especially ones with a reason attached.
- Config and constants that look like they were tuned by getting burned: retry counts, backoff values,
  rate limits, timeouts, magic thresholds, model fallbacks.
- Anything defensive: a validation layer wrapping something that should have been trustworthy, a
  dedup check, a "tried this, got nothing" marker. Each one exists because something broke.
- Real numbers. Row counts, cost per run, latency, throughput, how many records the thing has actually
  processed, what percentage of inputs fail. Run a query or a script if that's what it takes. Do not
  estimate and do not round to something suspiciously neat.
- What it cost and what tier/limit I hit.

Tell me plainly if something on this list doesn't exist in the repo. Don't invent it.

## Step 2: how to write it

Read the voice from these two published posts before drafting:
https://rohitsaini.me/writing/scamdb/ and https://rohitsaini.me/writing/brainrot-debugging/

The shape that works:

1. Open with the concrete thing that caused the project. A specific moment, not a market observation.
   No "in today's landscape", no "have you ever wondered".
2. The naive version. What I thought I was building, ideally as a small ASCII pipeline diagram, plus a
   line like "two hours of work" that is about to be wrong.
3. What actually happened. The assumption that was false, stated flatly.
4. The real architecture, as a second ASCII diagram, with the ugly parts labeled.
5. **Bugs that cost me real weekends.** Three or four, each with a bold one-line header naming the bug,
   then what it did, then the fix. Concrete symptoms, real error codes, real limits.
6. Where it is now. The actual numbers, including the unflattering ones.
7. The honest part. What is broken, unfinished, or abandoned, and why. Every post ends here. If the
   bottleneck is me, say so.

Rules on the prose:

- First person, past tense, plain words. Contractions are fine.
- Short paragraphs. Fragments are fine. Vary sentence length; do not write ten medium sentences in a row.
- Technical claims stay exact. Real library names, real versions, real status codes, real numbers.
- Dry and specific when being funny. "A search engine with 34 rows is not a search engine. It's a
  spreadsheet with a domain name." Never quippy for its own sake.
- 800 to 1200 words in the body.
- Code blocks only for ASCII diagrams or genuinely short snippets. This is a story, not documentation.
- British spelling where it comes up (favourite, behaviour).

Do not do these, they are the tells that make it read like a model wrote it:

- No em dashes anywhere. Use a period, a comma, or a colon.
- No sentence that ends by pivoting from the technical detail to a universal truth about life,
  software, or the internet. At most one such line in the whole post, and only if it is genuinely earned.
- No "it's not X, it's Y" as a rhetorical closer. No "the real X was Y all along."
- No tricolons. No "Fast. Cheap. Reliable."
- No section that summarizes what the previous sections said.
- No hedging: "arguably", "it's worth noting", "in many ways", "essentially".
- No praise for the project. No "elegant", "powerful", "seamless", "robust".
- Do not end on an upbeat note if the honest ending is that it's half-finished.

## Step 3: deliverables

**Deliverable 1** — one file, `<slug>.md`, ready to drop into `content/writing/`:

```markdown
---
title: "Short Punchy Title: The Thing That Was Actually Hard"
date: YYYY-MM-DD
readTime: <body word count / 200, rounded, integer>
draft: false
tags: [<pick 1-3 from the list below, nothing else>]
slug: "<kebab-case, short, no stopwords>"
description: "<1-2 sentences, first person, states the surprise. This is the excerpt on the
list page and the meta description, so it has to stand alone.>"
---

<body>
```

The tag vocabulary is closed. Use only these, and only the ones that genuinely apply:

- `build` — I made a thing and shipped it
- `backend` — APIs, data pipelines, storage, search, infrastructure
- `systems` — architecture, distributed systems, scale, design tradeoffs
- `debugging` — the post is mostly about hunting a specific class of bug
- `llm` — models are a real part of the story, not a passing mention

Do not invent tags. Not language names, not tool names.

**Deliverable 2** — the entry for the projects page, as a YAML fragment matching this shape exactly:

```yaml
  - name: "Project Name"
    description: "One or two sentences. What it does and the one thing that made it hard."
    live: "https://..."          # omit the whole line if nothing is deployed
    github: "https://github.com/rohitsaini1196/<actual-repo-name>"
    writings:
      - label: "Writeup →"
        url: "/writing/<slug>/"
    tags: ["Tech", "Stack", "Here"]   # 2-4, capitalized, real stack, not the writing tags
    status: "Live"               # Live | Active | Open Source | Completed | Prototype | Archived
```

Get the repo name exactly right, capitalization included, by checking the actual remote:
`git remote get-url origin`. Do not guess it from the directory name.

**Deliverable 3** — a short list of anything you found that's interesting but didn't fit the post,
and any place where I should check a number before I publish.

Output all three as plain text I can copy. Don't create files unless I ask.
