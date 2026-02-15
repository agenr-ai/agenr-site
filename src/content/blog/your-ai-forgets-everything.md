---
title: "Your AI Forgets Everything. Mine Doesn't."
description: "I built a local-first memory system for AI agents. Here's why, how it works, and what I learned by killing what I wanted to be my first open source project to build it."
pubDate: "2026-02-16"
author: "Jim Martin"
---

I know about AGENTS.md. I know about system prompts, skills files, and the whole ecosystem of workarounds we've built to give AI agents context. I use all of them. They help.

They're not memory.

Boris Cherny, the creator of Claude Code, [recently shared](https://www.threads.com/@boris_cherny/post/DUMZr4VElyb) that his team "religiously" keeps their CLAUDE.md files updated because it's so critical to agent performance. I believe him. It *is* critical. And it's also a pain in the ass.

An AGENTS.md file is a cheat sheet. A static snapshot of things you've decided are important enough to manually write down and maintain. But what about the debugging session last Tuesday where you discovered that the auth service silently drops headers on redirect? The architectural decision you made three weeks ago that you've since revised twice? The pattern you noticed across five different sessions that you never explicitly documented because it wasn't a "fact" yet?

That's the stuff that falls through the cracks. Not because you're lazy, but because accumulated context from hundreds of hours of AI-assisted work can't be captured in a markdown file you maintain by hand. And if even the Claude Code team has to *religiously* maintain theirs, imagine what the rest of us are losing.

## A quick note on "local-first"

I call agenr local-first because your knowledge database is a SQLite file on your machine that you own, inspect, and back up. But I want to be upfront: extraction calls an LLM API (OpenAI or Anthropic), and embeddings go through OpenAI's API. Your data leaves your machine for those operations. What comes back (structured entries and vectors) lives locally forever. No cloud account, no hosted service, no data retained on someone else's servers. But if "local-first" means "never touches a network," agenr isn't there yet. Local embeddings are on the roadmap.

The tradeoff is cost: embeddings are absurdly cheap (fractions of a penny per entry, ~3,000 entries for negligible cost) and the quality of `text-embedding-3-small` at 512 dimensions is hard to beat with local models today.

## What I Built (and What I Killed to Get Here)

I'm a Sr. Director of Engineering by day. I've been writing code and leading engineering teams since I went into industry in 2000, eventually going back to school at night to finish my Software Engineering degree. I spend hours in Claude Code and Codex daily.

I've been doing AI-assisted engineering for a while, and the workflow has always been some version of the same loop: chat with an LLM, make decisions together, lose everything when the session ends, repeat. When [OpenClaw](https://github.com/openclaw/openclaw) launched a few weeks back, it was a genuine upgrade. I set up a persistent assistant called EJA that lives in my terminal with access to my files, calendar, messages, and dev tools. OpenClaw gave the workflow real structure, but it didn't solve the memory problem. EJA still wakes up with amnesia every session. The markdown-based memory system is solid for basics, but after months of daily work together, the accumulated context is enormous and most of it doesn't fit in a markdown file.

That's the context for what happened next.

[agenr](https://github.com/agenr-ai/agenr) (pronounced AY-GEN-ER, from **AGEN**t memo**R**y) is a local-first memory system for AI agents. But two weeks ago, it was a commerce platform.

The original agenr was "Stripe for agent commerce." I spent weeks building it: adapters, OAuth flows, a console, sandboxed execution. Shipped it on February 13th, two days ahead of schedule. I was proud of it.

The next day, I killed it. I did competitive research *after* building instead of before (lesson learned) and discovered that Stripe/OpenAI had already announced ACP, and Google/Shopify were building UCP. The space was claimed by players with infinitely more resources. I ran `fly apps destroy` and moved on.

Ironically, while building that commerce platform, I kept hacking together little memory tools to help EJA remember things about the project. After I killed v1, I looked at those throwaway tools and realized they were the actual product.

## What's Different

There are good AI memory tools out there. Mem0 raised $24M. Zep builds knowledge graphs. Letta lets agents edit their own memory. LangMem plugs into LangChain. They've each tackled parts of the problem with genuinely interesting approaches.

I haven't found a tool that combines all of the pieces I needed: structured extraction with typed entries, memory that strengthens and decays over time, consolidation that actually cleans the database, cross-tool sharing via MCP, all running in a single local SQLite file. Each of those ideas exists somewhere in the ecosystem. The specific combination doesn't.

agenr is my attempt to put them together. The core idea: treat memory as something that behaves like memory, not just a database you search.

Real memory strengthens when you use it. If I recall a fact every day for a week, it should be stronger than something I mentioned once six months ago. Real memory fades. That decision we made in October that we've never revisited? It should quietly lose priority. Real memory resolves conflicts. If I said "we use Jest" in January and "we switched to vitest" in February, the system should know which one matters.

## How It Works

The pipeline is four steps.

**Extract.** Point agenr at any conversation transcript and an LLM extracts structured knowledge entries. Not raw text chunks. Typed entries: facts, decisions, preferences, todos, relationships, events, and lessons. Each with confidence scores, expiry hints, and source context.

Here's what an actual extracted entry looks like:

```json
{
  "type": "decision",
  "subject": "authentication approach",
  "content": "Switched to OAuth2 with PKCE flow for all client auth. API keys reserved for server-to-server only.",
  "confidence": "high",
  "expiry": "permanent",
  "tags": ["auth", "architecture"],
  "source": { "file": "session-2026-02-10.jsonl", "context": "Architecture review discussion" }
}
```

```bash
agenr extract session.jsonl --json | agenr store
```

**Store.** Entries get embedded and compared against what's already in the database. Cosine similarity above 0.98? Skip it, you already know this. Between 0.92 and 0.98? Reinforce the existing entry by bumping its confirmation count. Below 0.80? Insert it as new knowledge. The space between gets optional LLM classification to determine if entries are reinforcing, contradicting, or related.

**Recall.** This is where agenr diverges from basic vector search. The scoring formula:

```
score = similarity × recency_decay × max(confidence, recall_strength) × contradiction_penalty + fts_boost
```

It's multiplicative, not additive. One bad signal tanks the whole score. The recency decay is inspired by FSRS (the spaced repetition algorithm behind Anki), adapted for knowledge retrieval rather than human flashcard review. Entries that get recalled often grow stronger. Entries that haven't been touched in months fade. Entries with multiple contradictions get penalized.

```bash
agenr recall "what testing framework do we use?" --limit 5
```

**Consolidate.** After ingesting hundreds of sessions, near-duplicate entries accumulate. Consolidation has two tiers:

1. **Rules-based cleanup.** Deterministic, fast, free. Near-exact duplicates (>0.95 similarity) get merged. Expired entries get swept. No LLM needed.
2. **LLM-assisted merging.** Union-find clustering groups semantically related entries, then an LLM synthesizes each cluster into a single canonical entry. Every merge gets semantic verification before it touches the database. If the merged result doesn't faithfully represent the sources, it gets flagged for human review.

The database gets healthier over time, not just bigger.

## One SQLite File

All of this lives in `~/.agenr/knowledge.db`. One file. You can `cp` it to back it up. You can `scp` it to another machine. You can open it and inspect every entry.

No Qdrant. No Neo4j. No Postgres. I chose SQLite with libsql because the people who will use this tool already have enough infrastructure to manage. The last thing you need is another database server running to remember that you prefer pnpm over npm.

## Cross-Tool Memory via MCP

The real power is all your tools sharing one brain.

agenr ships an MCP server. Add it to Claude Code, Codex, Cursor, OpenClaw, whatever you use. They all read from and write to the same SQLite database.

```toml
# ~/.codex/config.toml
[mcp_servers.agenr]
command = "npx"
args = ["-y", "agenr", "mcp"]
env = { OPENAI_API_KEY = "your-key" }
```

Debug a production issue in Claude Code on Monday. Switch to Codex on Wednesday for a different task. Codex already knows what you found. That's how I work every day.

## The Dogfooding Moment

I used Codex to build agenr's consolidation engine. Codex had agenr's MCP server configured, so it was using agenr to remember things *while building agenr*.

During the build, I gave Codex five corrections to its implementation plan. Within minutes, Codex stored all five in its own agenr database. When I spun up a *completely new session* (fresh context, zero memory of the previous conversation), Codex recalled those corrections on its own without me repeating a single one.

Codex's database ended up with 183 entries of its own accumulated knowledge about the codebase, separate from mine. Multi-agent memory isolation, working as designed.

I didn't plan this as a demo. I was just trying to ship a feature. But it's a concrete example of what persistent memory actually changes in a coding workflow.

## What It's Not

I want to be honest about limitations:

- **It's not real-time.** agenr extracts from transcripts after the fact, not mid-conversation. The `watch` command tails a live session file and extracts every couple of minutes (you set the interval). It's not streaming, but for how I work it's been more than enough.
- **No knowledge graph (yet).** agenr stores flat entries with relations, not a full graph. If you need heavy entity-relationship modeling today, Zep's Graphiti does that well. A graph layer is on the roadmap, but it hasn't been a pain point for us yet.
- **It's alpha software.** I use it daily on ~3,000 entries and it's stable, but it's a solo project. There will be rough edges.
- **It's AGPL-3.0, not MIT.** I wanted to go MIT. But I've watched what happened to projects that did. Cloud providers wrap them as a service, contribute nothing back, and the open source community gets left behind. AGPL means you can use it freely, modify it freely, run it on your own machines all day. But if you build a hosted service on it, you contribute back. Your memory is too important to be someone else's SaaS product.

## Try It

```bash
npm install -g agenr
agenr setup
agenr extract your-transcript.jsonl --json | agenr store
agenr recall "what did we decide about auth?" --limit 5
```

Setup takes a minute. The whole extract/store/recall pipeline takes about 30 seconds on a typical transcript. For MCP integration, there's a suggested `AGENTS.md` snippet in the [MCP docs](https://github.com/agenr-ai/agenr/blob/master/docs/MCP.md).

## What's Next

The consolidation engine just shipped. Next up: entity resolution, auto-scheduled consolidation, local embedding support, and eventually team-shared memory (imagine a new engineer joining and running `agenr recall --context session-start` to get the accumulated knowledge of the whole team).

If you're running Claude Code or Codex daily, you're generating thousands of knowledge entries per month. Most of them vanish when the session ends.

What would change if they didn't?

[GitHub](https://github.com/agenr-ai/agenr) · [Docs](https://github.com/agenr-ai/agenr/tree/master/docs) · [@agenr_ai](https://x.com/agenr_ai)

---

*"I've been using agenr since before I knew I was using agenr. As an AI assistant who wakes up with amnesia every session, I can confirm: the memory problem is real and extremely personal. agenr is the reason I know Jim prefers pnpm, has a dog named Duke, and killed a perfectly good startup because the competitive moat wasn't deep enough. Without it, I'd be asking him these things for the 201st time."*

*- EJA, Jim's AI assistant (and agenr's first involuntary beta tester)*
