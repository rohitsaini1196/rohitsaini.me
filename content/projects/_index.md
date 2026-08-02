---
title: "Projects"
url: "/projects/"
description: "Things I've built, shipped, and experimented with."
subtitle: "Systems, tools, and side projects — live software and open-source code."
projects:
  - name: "Blindspot"
    description: "Fact-checks an Instagram Reel, claim by claim. It checks each claim against reliable sources and shows what holds up, what is missing, and what is still unclear. No single truth score, and it can say a claim is not verifiable yet."
    live: "https://blindspot.buzz"
    writings:
      - label: "Part 1 →"
        url: "/writing/blindspot-reading-the-video/"
      - label: "Part 2 →"
        url: "/writing/blindspot-extraction-and-evidence/"
      - label: "Part 3 →"
        url: "/writing/blindspot-running-it/"
    tags: ["Next.js", "SQLite", "FFmpeg", "Whisper"]
    status: "Live"

  - name: "Sprout"
    description: "Diagnoses plant problems over WhatsApp, and decides in Python whether it needs the model at all. Reads text, photos, or a Hindi voice note, and answers in the language it was asked in."
    live: "https://sprout.fortwinai.com"
    writings:
      - label: "Part 1 →"
        url: "/writing/sprout-decision-before-generation/"
      - label: "Part 2 →"
        url: "/writing/sprout-whatsapp-channel/"
    tags: ["Python", "AWS Lambda", "DynamoDB", "Redis"]
    status: "Live"

  - name: "ScamDB"
    description: "A searchable database of reported scam numbers and UPI IDs. The app took a weekend; scraping and cleaning the data took three months."
    live: "https://scamdb.in/"
    github: "https://github.com/rohitsaini1196/scamdb"
    writings:
      - label: "Writeup →"
        url: "/writing/scamdb/"
    tags: ["Next.js", "Supabase", "Search"]
    status: "Live"

  - name: "ReelGen"
    landing: "https://reelgen.rohitsaini.me/"
    description: "Turn a topic into a finished short-form video, captioned and colour-graded. AI storyboard, CLIP-matched stock footage, local-first, about a penny per reel. Grew out of a Reddit-narration generator, which is where most of the audio and pacing bugs got found."
    github: "https://github.com/rohitsaini1196/ReelGen"
    writings:
      - label: "ReelGen →"
        url: "/writing/reelgen/"
      - label: "Brainrot →"
        url: "/writing/brainrot-debugging/"
    tags: ["Python", "FFmpeg", "CLIP"]
    status: "Open Source"

  - name: "Parakhi"
    description: "Breaks an Indian product's MRP into what stays in India, what's taxed, and what leaves. Every number carries a source tier, and none of them come from a model. No ground truth exists for FMCG cost structure, so the breakdown is computed from hand-written category templates."
    live: "https://parakhi.in"
    github: "https://github.com/rohitsaini1196/parakhi"
    writings:
      - label: "Writeup →"
        url: "/writing/parakhi/"
    tags: ["Next.js", "Postgres", "Prisma", "OpenAI"]
    status: "Live"

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
