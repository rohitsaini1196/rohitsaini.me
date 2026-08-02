---
title: "Projects"
url: "/projects/"
description: "Things I've built, shipped, and experimented with."
subtitle: "Systems, tools, and side projects — live software and open-source code."
projects:
  - name: "ScamDB"
    description: "A searchable database of reported scam numbers and UPI IDs. The app took a weekend; scraping and cleaning the data took three months."
    live: "https://scamdb.in/"
    github: "https://github.com/rohitsaini1196/scamdb"
    writings:
      - label: "Writeup →"
        url: "/writing/scamdb/"
    tags: ["Next.js", "Supabase", "Search"]
    status: "Live"

  - name: "Parakhi"
    description: "Breaks an Indian product's MRP into what stays in India, what goes to tax, and what leaves the country, with a source tier on every number. No ground truth exists for FMCG cost structure, so the numbers are computed deterministically from hand-written category templates instead of generated."
    live: "https://parakhi.in"
    github: "https://github.com/rohitsaini1196/parakhi"
    writings:
      - label: "Writeup →"
        url: "/writing/parakhi/"
    tags: ["Next.js", "Postgres", "Prisma", "OpenAI"]
    status: "Live"

  - name: "ReelGen"
    description: "Turn a topic into a finished short-form video — AI storyboard, CLIP-matched stock footage, real color grading, animated captions. Local-first, about a penny per reel. Grew out of a Reddit-narration generator, which is where most of the audio and pacing bugs got found."
    github: "https://github.com/rohitsaini1196/ReelGen"
    writings:
      - label: "ReelGen →"
        url: "/writing/reelgen/"
      - label: "Brainrot →"
        url: "/writing/brainrot-debugging/"
    tags: ["Python", "FFmpeg", "CLIP"]
    status: "Active"

  - name: "Claude Session Tracker"
    description: "Local dashboard for the Claude Code tabs you forget about — surfaces sessions waiting on you or gone silent."
    github: "https://github.com/rohitsaini1196/claude-session-tracker"
    writings:
      - label: "Writeup →"
        url: "/writing/claude-code-session-tracker/"
    tags: ["CLI", "Developer Tool"]
    status: "Open Source"

  - name: "Meta-Dev-CLI"
    description: "CLI for Meta Developer Apps and the WhatsApp Cloud API — send messages, manage templates and webhooks from the terminal."
    github: "https://github.com/rohitsaini1196/meta-dev-cli"
    writings:
      - label: "Writeup →"
        url: "/writing/meta-dev-cli-stop-clicking-start-typing/"
    tags: ["CLI", "API"]
    status: "Open Source"
---
