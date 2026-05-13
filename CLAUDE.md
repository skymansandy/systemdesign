# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Sandybox System Design is a dedicated system design interview prep knowledge base built with **MkDocs** and the **Material for MkDocs** theme. Content is written in Markdown.

## Commands

- `mkdocs serve` — local dev server with live reload (http://127.0.0.1:8000)
- `mkdocs build` — generate static site into `site/`

## Content Structure

All content lives in `designs/` as Markdown files, organized **by topic**. The navigation tree is **manually defined** in `mkdocs.yml` under `nav:` — new pages must be added there to appear in the site.

Each topic has a **single `index.md`** file. Topics that cover both backend and mobile perspectives merge them into one document under a shared table of contents — backend-specific and mobile-specific sections appear as H2s within the same file. Shared context (requirements, API design) appears once.

```
designs/
├── index.md                          # Home page
├── home/
│   ├── best-practices/index.md
│   ├── building-blocks/index.md
│   ├── tech-tradeoffs/index.md
│   └── platform-fundamentals/index.md
├── social-media/
│   ├── chat-app/index.md
│   ├── news-feed/index.md
│   ├── instagram/index.md
│   ├── video-streaming/index.md
│   ├── multimedia-feed/index.md
│   ├── reddit-comments/index.md
│   └── message-expiration/index.md
├── e-commerce-payments/
│   ├── e-commerce/index.md
│   ├── mobile-payment/index.md
│   ├── stock-broking/index.md
│   └── hotel-reservation/index.md
├── productivity-location/
│   ├── offline-document-editor/index.md
│   ├── location-based-app/index.md
│   └── search-autocomplete/index.md
├── education-testing/
│   ├── live-class/index.md
│   └── test-taking/index.md
└── infrastructure-sdks/
    ├── image-loading/index.md
    ├── file-download-manager/index.md
    ├── network-layer/index.md
    ├── push-notification/index.md
    ├── analytics-sdk/index.md
    ├── feature-flags/index.md
    ├── design-system/index.md
    ├── app-update-migration/index.md
    └── pagination-library/index.md
```

## Adding Content

1. Create a folder under `designs/` named after the topic (e.g., `designs/url-shortener/`)
2. Add `index.md` inside that folder
3. Add the file path to the `nav:` section in `mkdocs.yml`

### Single-File Rule

Every topic is **one `index.md`**. If a topic has both backend and mobile dimensions, merge them:

- Shared sections (intro, requirements, API design) appear once
- Backend architecture is an H2 section
- Mobile client architecture is an H2 section
- Each section cross-references the other inline

### H1 Title Rule

The H1 (`#`) title is the **topic name** (e.g., `# Chat Application`). Perspective sections use H2 (`## Backend Architecture`, `## Mobile Client Architecture`).

### Document Structure — Unified Template

Every article follows this flow. Sections marked *(if applicable)* are included only when the topic has that dimension.

1. **H1: Topic Name** — 2-3 sentence hook: what this system is, why it's a compelling design problem, what makes it hard.

2. **Scoping the Problem** — The clarifying questions you'd ask, written as flowing prose ("The first thing I'd want to nail down is..."). Cover functional scope, non-functional priorities, and key constraints. No numbered tables — just the thought process. Include capacity estimation only if it meaningfully drives a design decision (e.g., write-heavy → LSM-tree, not just "here are some big numbers").

3. **API Design** — Protocol choice with reasoning and a comparison table. Key endpoints (3-5 max) with request/response shapes. Mention pagination and error strategy briefly — don't catalog every field.

4. **Backend Architecture** *(if applicable)* — High-level system diagram (Mermaid). Walk through the core components and why each exists. Data model, storage engine choice with reasoning. Deep-dive into the 2-3 genuinely hard problems unique to this system (e.g., fan-out for chat, ranking for feed, double-spend for payments).

5. **Mobile Client Architecture** *(if applicable)* — Architecture diagram (Mermaid). Clean architecture layers, KMP-aligned. Deep-dive into mobile-specific hard problems: offline-first, sync engine, local DB, lifecycle, background work, push notifications. Use as many diagrams as needed for data flows, state machines, sequence diagrams.

6. **Scalability, Reliability & Edge Cases** — How the system scales, what breaks first, and how to handle it. Non-obvious failure modes. Fold edge cases into this section rather than separating them.

7. **Wrap Up** — 3-5 bullet summary of key decisions. "What I'd improve with more time." Keep it tight.

8. **References** — External links for advanced deep-dives. Use these to offload content that's beyond expert-level (e.g., CRDT theory, Signal Protocol internals, Raft consensus). Blogs, conference talks, papers, official docs.

### Content Principles

- **Conversational interview tone** — Write as if the author is being interviewed and walking through their thought process. The narrative follows the chain of reasoning: "I'd start by...", "The reason I'd pick X over Y is...", "Now the tricky part here is...". It reads as an article, not a transcript — but the thinking unfolds naturally, not as a dry reference doc.
- **Expert-level depth, zero filler** — Cover enough to demonstrate near-expert knowledge on each topic. Every sentence should teach something or justify a decision. Cut anything that's just restating the obvious or padding sections.
- **Decision reasoning everywhere** — For every design choice, explain *why this* and *why not the alternatives*
- **KMP-aligned** — Mobile code samples use Kotlin (KMP-friendly); mention platform-specific implementations only where they diverge
- **Modern Android stack** — Compose, Ktor, Room/SQLDelight, Coroutines/Flow, WorkManager, Hilt/Koin
- **Industry insights inline** — Reference how well-known apps (WhatsApp, Signal, Instagram, Slack, Discord) solve specific problems. Weave these in where relevant, don't dump them in a separate section.
- **Offline-first** — Every mobile design must deeply cover offline scenarios, sync conflicts, and queue management
- **Diagrams as needed** — Use as many Mermaid diagrams as required to convey architecture, data flows, state machines, and sequences. No arbitrary limit.
- **Pro tips & edge cases** — Use `!!! tip "Pro Tip"` for interview-winning insights. Use `!!! warning "Edge Case"` for gotchas. These are high-signal — keep them.
- **Tables for comparisons** — Use comparison tables when contrasting options (protocols, databases, caching strategies). Don't use tables for requirements lists — use prose or compact bullets instead.
- **External links for advanced rabbit holes** — Don't write 500 words on CRDT theory or consensus algorithms. Link to authoritative external resources and move on.

### What NOT to Include

- **No verbose requirements tables** — Don't itemize FRs/NFRs in a table with a "Why It Matters" column. Weave requirements into the scoping narrative.
- **No capacity estimation theater** — Only include back-of-the-envelope math when it directly drives a design choice. Don't compute numbers just to compute them.
- **No exhaustive endpoint catalogs** — 3-5 key endpoints, not every CRUD operation
- **No filler prose** — Assume the reader is a senior engineer
- **No redundant sections** — If backend and mobile share context, write it once

## Markdown Features Available

The site is configured with these extensions — use them in content:

- **Admonitions**: `!!! note`, `!!! warning`, `!!! tip`, etc. (collapsible with `???`)
- **Tabbed content**: `=== "Tab 1"` / `=== "Tab 2"`
- **Code blocks**: fenced with language + line numbers; copy button enabled
- **Inline code highlighting**: `` `#!python print("hi")` ``
- **Mermaid diagrams**: fenced with ` ```mermaid `
- **Attribute lists** and **HTML in Markdown** (`md_in_html`)
