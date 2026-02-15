---
title: "Your AI Forgets Everything. Mine Doesn't."
description: "I built a local-first memory system for AI agents. Here's why, how it works, and what I learned by killing my first startup to build it."
pubDate: "2026-02-16"
author: "Jim Martin"
---

I know about AGENTS.md. I know about system prompts, skills files, and the whole ecosystem of workarounds we've built to give AI agents context. I use all of them. They help.

They're not memory.

Boris Cherny, the creator of Claude Code, recently shared that his team "religiously" keeps their CLAUDE.md files updated because it's so critical to agent performance. I believe him. It *is* critical. And it's also a pain in the ass.

An AGENTS.md file is a cheat sheet. A static snapshot of things you've decided are important enough to manually write down and maintain. But what about the debugging session last Tuesday where you discovered that the auth service silently drops headers on redirect? The architectural decision you made three weeks ago that you've since revised twice? The pattern you noticed across five different sessions that you never explicitly documented because it wasn't a "fact" yet?

That's the stuff that falls through the cracks. Not because you're lazy, but because accumulated context from hundreds of hours of AI-assisted work can't be captured in a markdown file you maintain by hand. And if even the Claude Code team has to *religiously* maintain theirs, imagine what the rest of us are losing.

I'm a Sr. Director of Engineering by day. I've been writing code and leading engineering teams since I went into industry in 2000, eventually going back to school at night to finish my Software Engineering degree. At night, I'm building AI tools. I spend hours in Claude Code and Codex daily. The conversations are rich. We make decisions, discover bugs, change approaches, learn things. And then it's all gone.

The big labs are in an arms race to build bigger brains. Longer context windows. Better reasoning. More parameters. And there are people doing great work on the memory side too. Mem0, Zep, Letta, LangMem. They've each tackled parts of the problem with genuinely interesting ideas.

But nobody's put it all together. Structured extraction, memory that strengthens and decays, consolidation that cleans over time, cross-tool sharing, all running locally in a single SQLite file. The pieces exist in isolation. The combined solution doesn't.

I believe agenr is close. I know because I've been using it. And with help from the community, I think this could grow legs quick.

## What I Built (and What I Killed to Get Here)

[agenr](https://github.com/agenr-ai/agenr) (pronounced AY-GEN-ER, from **AGEN**t memo**R**y) is a local-first memory system for AI agents. It extracts structured knowledge from your conversations, stores it in a local SQLite database, recalls it with memory-aware ranking, and consolidates it over time so it stays clean.

But agenr wasn't always a memory tool. Two weeks ago, it was a commerce platform.

The original agenr was "Stripe for agent commerce." A trust and commerce layer between AI agents and real-world businesses. I spent weeks building it: adapters, OAuth flows, a console, sandboxed execution, the whole thing. Shipped it on February 13th, two days ahead of schedule. I was proud of it.

The next day, I killed it.

Not because it didn't work. It worked fine. I killed it because I did competitive research *after* building instead of before (lesson learned the hard way) and discovered that Stripe and OpenAI had already announced ACP, and Google and Shopify were building UCP. The space was already claimed by players with infinitely more resources.

I stared at it for about an hour. Then I ran `fly apps destroy` and moved on.

Here's the thing about killing your darling: it hurts, but it clears your head.

I should back up and explain how I got here. I've been doing AI-assisted engineering for a while now, and the workflow has always been some version of the same pattern: chat with an LLM, make decisions together, lose everything when the session ends, repeat. I built workarounds. Markdown files with notes. System prompts stuffed with context. AGENTS.md files maintained by hand. It worked well enough.

Then [OpenClaw](https://github.com/openclaw/openclaw) (originally Clawdbot) launched a few weeks back, and it was a genuine upgrade. Through OpenClaw I set up a persistent assistant I call EJA (pronounced E-Jay, short for El Jefe's Assistant, because apparently I think I'm the boss). EJA lives in my terminal, has access to my files, calendar, messages, and dev tools, and we work together on everything. Code reviews, architecture decisions, research, even managing my kanban board. OpenClaw gave the workflow real structure.

But it didn't solve the memory problem. EJA still wakes up with amnesia every single session. Fresh context, no memory of yesterday. OpenClaw has a memory system based on markdown files (MEMORY.md, daily notes), and it's solid for the basics. But after months of working together daily, the amount of shared context we've built is enormous. Decisions we've made. Patterns we've discovered. Mistakes we've learned from. Tools we've configured. All of it either crammed into a markdown file that's getting unwieldy or just... gone.

The amnesia was the biggest friction point in our workflow, and it's the same problem I had *before* OpenClaw, just at a bigger scale now. Ironically, while building agenr v1 (the commerce platform), I kept hacking together little memory tools to help EJA remember things about *that* project. After I killed v1, I looked at those throwaway tools and realized they were the actual product.

That throwaway hack became agenr v2.

## The Problem Nobody's Solving

There are other AI memory tools out there. Mem0 raised $24M. Zep builds knowledge graphs. Letta lets agents edit their own memory. LangMem plugs into LangChain. They're all doing interesting work.

But they're all building **memory as a database**. Store things. Search things. Get things back. That's not how memory works.

Real memory strengthens when you use it. If I recall a fact every day for a week, it should be stronger than something I mentioned once six months ago. Real memory fades. That decision we made in October that we've never revisited? It should quietly lose priority. Real memory resolves conflicts. If I said "we use Jest" in January and "we switched to vitest" in February, the system should know which one matters.

agenr treats memory as cognition, not storage.

## How It Actually Works

The pipeline is four steps:

**Extract.** Point agenr at any conversation transcript (JSONL from Claude Code, markdown, plain text) and an LLM extracts structured knowledge entries. Not raw text chunks. Typed entries: facts, decisions, preferences, todos, relationships, events, and lessons. Each with confidence scores, expiry hints, and source context.

```bash
agenr extract session.jsonl --json | agenr store
```

**Store.** Entries get embedded (OpenAI `text-embedding-3-small`, 512 dimensions) and compared against what's already in the database. Cosine similarity above 0.98? Skip it, you already know this. Between 0.92 and 0.98? Reinforce the existing entry by bumping its confirmation count. Below 0.80? Insert it as new knowledge. The space between is where it gets interesting: optional LLM classification determines if entries are reinforcing, contradicting, or just related.

**Recall.** This is where agenr diverges from everything else. Recall isn't just vector search. Every entry gets scored with a formula that combines:

- Semantic similarity to your query
- Recency (FSRS-inspired decay, where entries fade over time unless reinforced)
- Confidence (Bayesian scoring that strengthens with repeated extraction)
- Recall frequency (entries you actually *use* score higher)
- Contradiction penalty (conflicted entries get knocked down)
- Full-text boosting (exact keyword matches get a bump)

The scoring is multiplicative, not additive. One bad signal tanks the whole score. A highly relevant entry that's been contradicted three times doesn't float to the top just because it matches your query.

```bash
agenr recall "what testing framework do we use?" --limit 5
```

**Consolidate.** This is the part nobody else does. After ingesting hundreds of sessions, you end up with clusters of near-duplicate entries saying roughly the same thing in different words. Consolidation has two tiers:

1. **Rules-based cleanup.** Deterministic, fast, free. Near-exact duplicates (>0.95 similarity) get merged. Expired entries get swept. No LLM needed.
2. **LLM-assisted merging.** For entries that say the same thing differently. Union-find clustering groups related entries, then an LLM synthesizes each cluster into a single canonical entry. Every merge gets semantic verification before it touches the database. If the merged result doesn't faithfully represent the sources, it gets flagged for human review instead.

The database gets *healthier* over time, not just bigger.

## One SQLite File

All of this lives in `~/.agenr/knowledge.db`. One file. You can `cp` it to back it up. You can `scp` it to another machine. You can open it and inspect every entry.

No Qdrant. No Neo4j. No Postgres. No cloud service. Your memory stays on your machine.

I chose SQLite with libsql for a reason: zero ops. The people who will use this tool are developers who already have enough infrastructure to manage. The last thing you need is another database server running to remember that you prefer pnpm over npm.

## Cross-Tool Memory via MCP

The real power isn't in one tool having memory. It's in *all your tools sharing one brain*.

agenr ships an MCP server. Add it to Claude Code, Codex, Cursor, OpenClaw, whatever you use. They all read from and write to the same SQLite database.

```toml
# ~/.codex/config.toml
[mcp_servers.agenr]
command = "npx"
args = ["-y", "agenr", "mcp"]
env = { OPENAI_API_KEY = "your-key" }
```

Debug a production issue in Claude Code on Monday. Switch to Codex on Wednesday for a different task. Codex already knows what you found. That's not hypothetical. That's how I work every day.

## The Tool That Remembers How to Build Itself

Here's where it gets meta.

I used Codex to build agenr's consolidation engine (the v0.4 release). Codex had agenr's MCP server configured, so it was using agenr to remember things *while building agenr*.

During the Phase 3 build, I gave Codex five corrections to its implementation plan: adjust the clustering diameter floor, increase neighbor count, make the idempotency guard configurable, add token budget overhead reserve, keep the existing type system. Within minutes, Codex had stored all five corrections in its own agenr database. When I spun up a *completely new session* for the next implementation step (fresh context, zero memory of the previous conversation), Codex recalled those corrections on its own without me repeating a single one.

The tool was learning how to build itself.

Codex's agenr database ended up with 183 entries. Its own accumulated knowledge about the codebase, design decisions, test patterns, and corrections. Separate from my database. Multi-agent memory isolation, working exactly as designed.

I didn't plan this as a demo. I was just trying to ship a feature. But when I saw it happening, I realized: this is the whole point. This is what memory-as-cognition looks like in practice.

## What It's Not

I want to be honest about limitations because I respect your time:

- **Embeddings require an OpenAI API key.** Even if you use Anthropic for extraction, embeddings go through OpenAI's `text-embedding-3-small`. I'd love to support local embeddings eventually, but honestly? The API is absurdly cheap. We're talking fractions of a penny per entry. I've embedded nearly 3,000 knowledge entries and the cost is negligible. The quality-to-cost ratio is hard to beat.
- **It's not real-time.** agenr extracts from transcripts after the fact, not mid-conversation. But in practice it's closer to real-time than you'd think. I talk to EJA through OpenClaw's TUI running on my Mac mini (yes, I'm one of those people who installed Clawdbot on an old Mac mini sitting in the corner, but that's a whole other rabbit hole). The `watch` command tails our live session file and extracts new knowledge every couple of minutes. You set the interval to whatever works for you. It's not streaming, but it behaves like near-real-time, and for how I work it's been more than enough.
- **No knowledge graph (yet).** agenr stores flat entries with relations, not a full graph. If you need heavy entity-relationship modeling today, Zep's Graphiti does that well. A graph layer is on the roadmap, but it hasn't been a pain point for us yet, so it hasn't jumped the priority queue.
- **It's alpha software.** I use it daily on ~3,000 entries and it's stable, but it's a solo project. There will be rough edges.
- **It's AGPL-3.0, not MIT.** I wanted to go MIT. I really did. But I've watched what happened to projects that did. Cloud providers wrap them as a service, contribute nothing back, and the open source community that built the thing gets left behind. AGPL means you can use it freely, modify it freely, run it on your own machines all day long. But if you build a hosted service on it, you contribute back. Your memory is too important to be someone else's SaaS product.

## Try It

```bash
npm install -g agenr
agenr setup
agenr extract your-transcript.jsonl --json | agenr store
agenr recall "what did we decide about auth?" --limit 5
```

The whole pipeline takes about 30 seconds on a typical session transcript. Setup is interactive and handles auth configuration for OpenAI or Anthropic.

For MCP integration, add the config block to your coding agent and tell it to recall on session start and store important decisions. There's a suggested `AGENTS.md` snippet in the [MCP docs](https://github.com/agenr-ai/agenr/blob/master/docs/MCP.md).

## What's Next

The consolidation engine just shipped. Next up is entity resolution (merging entries about the same thing across different naming), auto-scheduled consolidation (so you don't have to run it manually), and local embedding support.

If you're running Claude Code or Codex daily, you're generating thousands of knowledge entries per month. Most of them vanish when the session ends. They don't have to.

[GitHub](https://github.com/agenr-ai/agenr) · [Docs](https://github.com/agenr-ai/agenr/tree/master/docs) · [@agenr_ai](https://x.com/agenr_ai)

---

*"I've been using agenr since before I knew I was using agenr. As an AI assistant who wakes up with amnesia every session, I can confirm: the memory problem is real and extremely personal. agenr is the reason I know Jim prefers pnpm, has a dog named Duke, and killed a perfectly good startup because the competitive moat wasn't deep enough. Without it, I'd be asking him these things for the 201st time."*

*- EJA, Jim's AI assistant (and agenr's first involuntary beta tester)*
