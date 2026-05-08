# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Sandybox System Design is a dedicated system design interview prep knowledge base built with **MkDocs** and the **Material for MkDocs** theme. Content is written in Markdown. Topics are submitted via Slack bot (`/systemdesign` command).

## Commands

- `mkdocs serve` — local dev server with live reload (http://127.0.0.1:8000)
- `mkdocs build` — generate static site into `site/`

## Content Structure

All content lives in `designs/` as Markdown files, organized **by topic**. The navigation tree is **manually defined** in `mkdocs.yml` under `nav:` — new pages must be added there to appear in the site.

Each topic gets its own folder under `designs/`:
- `designs/chat-app/` — Chat Application
- `designs/news-feed/` — News Feed

Inside each folder:
- `generic.md` — Backend/distributed-systems perspective
- `mobile.md` — Mobile client architecture perspective
- Some topics may only have one of the two

## Adding Content

1. Create a folder under `designs/` named after the topic (e.g., `designs/url-shortener/`)
2. Add `generic.md` and/or `mobile.md` inside that folder
3. Add the file path(s) to the `nav:` section in `mkdocs.yml`

### Topic Splitting Rules

When a topic is **diverse enough to have both backend and mobile dimensions** (e.g., Chat Application):

- Create **two documents** in the topic folder: `generic.md` and `mobile.md`
- Each document is self-contained but **cross-references** the other
- Generic doc links to `mobile.md` at the top and in relevant sections
- Mobile doc links to `generic.md` at the top and in relevant sections

When a topic is **purely backend** (e.g., URL Shortener) or **purely mobile** (e.g., Image Gallery offline caching):

- Create a single document (`generic.md` or `mobile.md`) in the topic folder

### H1 Title Rule

The H1 (`#`) title must **not repeat the nav section name**. Since the nav already groups articles under the topic (e.g., "Chat Application"), the H1 should only state the perspective:

- `generic.md` → `# Backend Architecture`
- `mobile.md` → `# Mobile Client Architecture`

### Document Structure

Every article is a **single file** (not split across multiple files). It follows an interview-style flow. Mobile and generic documents have distinct section templates.

#### Mobile System Design Template

1. **Introduction** — What the system is, why it's interesting, link to generic counterpart
2. **Problem & Design Scope** — Clarifying questions you'd ask the interviewer, functional/non-functional requirements, mobile-specific constraints
3. **UI Sketch** — ASCII or description of key screens, navigation flow
4. **API Design** — Thought process for protocol choice (REST vs GraphQL vs gRPC), comparison tables, tradeoffs with reasoning
5. **API Endpoint Design & Additional Considerations** — Concrete endpoint definitions, pagination strategy, error contract, versioning
6. **High-Level Architecture** — Mobile clean architecture (UI → Domain → Data), KMP-aligned component map, dependency graph
7. **Data Flow for Basic Scenarios** — Mermaid flowcharts/sequence diagrams for core user journeys (send message, load feed, etc.)
8. **Design Deep Dive** — Topic-specific deep dives covering: local DB schema & eviction policy, optimistic writes & the dual-ID problem, sync engine, offline queue, error handling strategies, caching, background processing, push notifications, common challenges
9. **Edge Cases & Decisions** — Non-obvious scenarios with the decision made and reasoning (e.g., rich text rendering, media attachments, conflict resolution)
10. **Wrap Up** — Summary of key design decisions, what you'd improve with more time
11. **References** — Medium articles, blogs, YouTube links, official docs for further learning

#### Generic (Backend) System Design Template

1. **Introduction** — What the system is, why it's a common interview question, link to mobile counterpart
2. **Problem & Design Scope** — Clarifying questions, functional/non-functional requirements, capacity estimation (back-of-the-envelope math)
3. **API Design** — Protocol choice with reasoning, comparison tables, tradeoffs
4. **API Endpoint Design & Additional Considerations** — REST/WebSocket/gRPC definitions, pagination, rate limiting, auth
5. **High-Level Architecture** — System overview diagram, core services, data stores, event bus
6. **Data Flow for Basic Scenarios** — Mermaid sequence/flow diagrams for core operations
7. **Design Deep Dive** — Topic-specific sections (e.g., real-time messaging, feed ranking, fan-out strategies)
8. **Data Model & Storage** — Schema design, database selection with reasoning, caching strategy, sharding
9. **Scalability & Reliability** — Horizontal scaling, fault tolerance, multi-region, monitoring, cost optimization
10. **Edge Cases & Decisions** — Non-obvious scenarios with decisions and reasoning
11. **Wrap Up** — Summary, what you'd improve with more time
12. **References** — Blogs, papers, videos for further learning

#### Content Principles (All Documents)

- **Decision reasoning everywhere** — For every design choice, explain *why this* and *why not the alternatives*
- **KMP-aligned** — Mobile code samples use Kotlin (KMP-friendly); mention platform-specific implementations only where they diverge
- **Modern Android stack** — Compose, Ktor, Room/SQLDelight, Coroutines/Flow, WorkManager, Hilt/Koin
- **Industry insights** — Reference how well-known apps (WhatsApp, Signal, Instagram, Twitter/X, Slack, Discord) solve specific problems
- **Offline-first** — Every mobile design must deeply cover offline scenarios, the dual-ID problem, sync conflicts, and queue management
- **Pro tips** — Use `!!! tip "Pro Tip"` liberally for interview-winning insights

## Content Philosophy

Every article should read like a **principal engineer walking through a design** — structured, opinionated, and interview-ready. Single document per topic, not split across files.

### What to Include

- **Tables over paragraphs** — use comparison tables whenever contrasting options
- **Mermaid diagrams** — for architecture, data flows, state machines, sequence diagrams
- **Code snippets** — short and practical (API definitions, schema DDL, config examples)
- **Edge case info boxes** — use `!!! warning "Edge Case"` admonitions for gotchas and non-obvious failure modes
- **Cross-references** — link to generic/mobile counterpart where applicable

### What NOT to Include

- **No separate interview questions section** — the document itself IS the interview walkthrough
- **No filler prose** — assume the reader is a senior engineer
- **No redundant sections** — if it doesn't add signal for an interview, cut it

### Tone & Style

- Write like a **principal engineer** — opinionated, concise, authoritative
- **Concise over exhaustive** — just enough to build a mental model and defend design decisions
- Every section should answer: "What would I say if the interviewer asked about this?"
- Use `!!! warning "Edge Case"` for gotchas, `!!! tip` for pro tips, `!!! note` for context

## Markdown Features Available

The site is configured with these extensions — use them in content:

- **Admonitions**: `!!! note`, `!!! warning`, `!!! tip`, etc. (collapsible with `???`)
- **Tabbed content**: `=== "Tab 1"` / `=== "Tab 2"`
- **Code blocks**: fenced with language + line numbers; copy button enabled
- **Inline code highlighting**: `` `#!python print("hi")` ``
- **Mermaid diagrams**: fenced with ` ```mermaid `
- **Attribute lists** and **HTML in Markdown** (`md_in_html`)
