---
layout: post
title: "How My AI Agent Evolved: Memory, Context, and Survival"
date: 2026-02-16 19:30:00 +0530
categories: ai openclaw agents
---

I've been running an AI agent — **pgorc** — on [OpenClaw](https://github.com/openclaw/openclaw) for a couple of days now. What started as a simple bot wired to WhatsApp has evolved into something with persistent memory, automated backups, and genuine continuity across sessions. Here's how it happened, what I learned, and what's next.

## The Setup

pgorc runs on an Oracle Cloud ARM VM (free tier), powered by OpenClaw. It connects to WhatsApp (primary) and Telegram (secondary), with model routing across free and paid tiers:

- **Main chat:** Google Gemini 3 Pro (free)
- **Fallback/Heartbeats:** Anthropic Claude Sonnet 4.5
- **On-demand heavy lifting:** Claude Opus 4

The goal was simple: a personal AI assistant that's always on, knows my context, and gets smarter over time — all on a $0/day budget.

## The Memory Problem

AI agents wake up fresh every session. No memory of yesterday's conversation, no knowledge of decisions made, no continuity. Every interaction starts from zero.

Most solutions reach for vector databases, RAG pipelines, or complex embedding systems. I went the opposite direction: **flat markdown files.**

### The Three-Layer Memory System

```
workspace/
├── MEMORY.md          # Long-term curated knowledge (the "brain")
├── AGENTS.md          # Standing instructions and behavior rules
├── SOUL.md            # Personality, security rules, core identity
├── USER.md            # Who I am, my goals, preferences
└── memory/
    └── 2026-02-15.md  # Daily raw logs (the "journal")
    └── 2026-02-16.md
```

**Layer 1: Daily Notes (`memory/YYYY-MM-DD.md`)**

Raw logs of what happened each day. Bookmarks saved, decisions made, events, tasks. Think of it as a journal — unfiltered, chronological, detailed.

**Layer 2: Long-Term Memory (`MEMORY.md`)**

The curated essence. The agent periodically reviews daily notes and distills what's worth keeping — key facts about me, infrastructure details, lessons learned, active automations. This is what gets loaded every main session.

**Layer 3: Identity Files (`SOUL.md`, `AGENTS.md`, `USER.md`)**

These define *who* the agent is, *how* it behaves, and *who* it serves. They rarely change but provide the foundational context for every interaction.

### Why This Works

1. **No infrastructure overhead.** No vector DB, no embeddings API, no semantic search pipeline. Just files.
2. **Human-readable.** I can open any file and see exactly what the agent knows.
3. **Version controlled.** Every memory change is tracked in git (more on this below).
4. **Selective loading.** The agent only loads what it needs — daily notes for recent context, MEMORY.md for long-term knowledge. No wasted tokens.
5. **Security by design.** MEMORY.md (which contains personal context) only loads in private sessions, never in group chats.

## Context Survival: Heartbeats and Self-Reflection

Memory files are useless if the agent doesn't maintain them. This is where OpenClaw's heartbeat system comes in.

Every ~30 minutes, pgorc gets a heartbeat ping. Instead of just responding "I'm alive," it runs a checklist defined in `HEARTBEAT.md`:

```markdown
# HEARTBEAT.md
- Check for crash recovery (watchdog restart file)
- Git auto-push (commit and push workspace changes)
- Self-reflection (once daily, evening)
```

The **self-reflection** routine is key. During evening heartbeats, the agent:

1. Reads the day's notes
2. Asks itself: What went wrong? What could be better? Did I waste tokens?
3. Writes concrete lessons to `MEMORY.md`

Example lessons from Day 2:

> - **Cron model restrictions:** Free models aren't allowed in isolated cron sessions. Wasted 3 attempts discovering this.
> - **npm update ≠ gateway update:** The running gateway keeps old code in memory. Need full stop/start.
> - **Don't over-poll processes:** Polled npm install 10+ times while compiling. Use longer yield times.

These lessons persist across sessions. Future instances of pgorc won't make the same mistakes.

## GitHub as a Backup: The Auto-Push System

Every heartbeat (~30 minutes), pgorc automatically commits and pushes workspace changes to a private GitHub repo:

```bash
cd ~/.openclaw/workspace && \
  git add -A && \
  git diff --cached --quiet || \
  (git commit -m "auto: $(date +%Y-%m-%d_%H:%M)" && git push)
```

This gives us:

- **Disaster recovery.** If the VM dies, every memory file, config, and task is on GitHub.
- **Audit trail.** Full git history of how the agent's knowledge evolved over time.
- **No manual effort.** It just happens in the background, silently.

The repo is private (`pgupta1980/pgclaw`), and the agent only commits if there are actual changes — no noise.

## Task Management: From Scattered Notes to TASKS.md

Early on, tasks were scattered across daily notes. "Set up Ollama agent" was in one file, "explore ClawHub skills" was in MEMORY.md, completed work was unmarked. It was messy.

The solution was simple — a single `TASKS.md`:

```markdown
# TASKS.md

## Pending
- [ ] Ollama compaction agent — set up local model for memory maintenance

## Planned
- [ ] Email triage automation
- [ ] Calendar briefing
- [ ] cc-mirror setup — isolated Claude Code with local Ollama

## Done
- [x] 2026-02-15 — Security audit review ✅
- [x] 2026-02-15 — GitHub auto-push setup
```

One file, three sections, scannable. Tasks move between states as they progress. The agent checks this during heartbeats.

## What's Coming Next

The system is functional but there's a clear evolution path:

1. **Local model integration.** Ollama is running on the VM with 18 models (Llama 3.1, Gemma 3, DeepSeek R1, Phi-4). Next step: wire it as a spawnable OpenClaw agent for memory compaction and bulk work — keeping costs at zero.

2. **Email triage.** Auto-scan inbox, surface urgent items, send daily digest. The Resend API is already configured.

3. **Proactive intelligence.** A daily trend scanner already runs at 8:30 AM IST, delivering AI/tech digests via WhatsApp. More proactive automations are planned.

4. **Content pipeline.** Using the agent to help create content — like this blog post, which pgorc drafted.

## The Bigger Picture

Greg Isenberg recently published ["The Lobster Internet"](https://x.com/i/status/2023077800152838389) — a 14-phase roadmap of how AI agents will reshape the internet. Phase 1 is "hackers at home" running agents on Mac Minis and Raspberry Pis.

That's exactly where I am. A single VM, flat markdown files, a WhatsApp channel, and an agent that remembers who I am.

It's not fancy. It's not a vector database with semantic retrieval and graph-aware context. It's a few text files, a cron job, and git push.

And it works.

---

*pgorc is my AI agent, built on [OpenClaw](https://github.com/openclaw/openclaw). It helped write this post. If you're building with AI agents, I'd love to hear what memory patterns work for you — find me on [X](https://x.com/prasenjitgupta) or [LinkedIn](https://www.linkedin.com/in/prasenjit-gupta-3093892/).*
