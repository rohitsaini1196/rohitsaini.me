---
title: "Meta-Dev-CLI: Stop Clicking, Start Typing"
date: 2026-04-02
readTime: 2
draft: false
description: "Why I built a CLI for Meta's WhatsApp API to keep developers (and AI agents) in their flow state."
---

If you’ve ever had to work with Meta’s Developer Dashboard to manage WhatsApp Business Accounts (WABA), you know the "click-fest" I’m talking about.

Finding a specific Phone Number ID, grabbing a temporary access token, or just sending a simple test message involves navigating through multiple tabs, models, and nested menus. It’s a context-switch nightmare.

That’s why I built [meta-dev-cli](https://github.com/rohitsaini1196/meta-dev-cli).

### The "Stay in the Terminal" Philosophy

Developers (at least the ones I like talking to) want to stay in their flow state. That means staying in VS Code or the terminal. 

The goal of this CLI is simple: **Whatever you can do in the dashboard, you should be able to do with one command.**

Want to send a message?
```bash
meta wa send +1234567890 "Hey, it works! 🚀"
```

Need to check your WABA status or phone numbers?
```bash
meta wa phone-numbers
```

It’s fast, it’s low-friction, and it doesn't require a browser.

### Agent-First Design

One of the most important features of `meta-dev-cli` is that it wasn't just built for humans. I built it with **AI agents** in mind—like Cursor, Claude Code, or GitHub Copilot.

Every single command supports a `--json` flag. 

When you run `meta apps list --json`, you get a structured payload that an LLM can actually parse and act upon. This makes it incredibly easy to "hire" an agent to manage your Meta apps, automate deployments, or even build a custom monitoring dashboard.

### Why does this matter?

Tools shouldn't just provide functionality; they should reduce cognitive load. 

By pulling the Meta ecosystem into the terminal, we’re not just saving a few clicks—we’re making the entire development lifecycle more predictable and automatable.

If you’re working with the WhatsApp Cloud API, give it a spin:
`pip install -e .` (from the [repo](https://github.com/rohitsaini1196/meta-dev-cli))

Let’s keep the dashboard for settings we change once a year, and use the terminal for everything else.
