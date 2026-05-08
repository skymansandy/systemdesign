# System Design Knowledge Base

A collection of system design case studies written as **interview walk-throughs**. Each topic is a single, self-contained document that follows the structure an interviewer expects — from requirements gathering through scalability deep-dives.

## Topics

| Topic | Generic | Mobile |
|-------|---------|--------|
| **Chat Application** | [Backend & Distributed](social-media/chat-app/generic.md) | [Client Architecture](social-media/chat-app/mobile.md) |
| **News Feed** | [Backend & Distributed](social-media/news-feed/generic.md) | — |

!!! note "Topic Layout"
    Each topic has its own folder. Topics that span both backend and mobile have two documents — `generic.md` and `mobile.md` — with cross-references linking the two perspectives.

## Document Structure

Every article follows a consistent interview-style flow:

1. **Requirements & Estimation** — Clarify scope, define FR/NFR, estimate scale
2. **High-Level Architecture** — Core components, API design, data flow
3. **Deep-Dive Sections** — Topic-specific technical deep-dives
4. **Data Model & Storage** — Schema design, database choices, caching
5. **Scalability & Reliability** — Scaling strategy, fault tolerance, monitoring
