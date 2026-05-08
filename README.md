# Sandybox — System Design

System design interview prep knowledge base built with [MkDocs](https://www.mkdocs.org/) and the [Material for MkDocs](https://squidfunnel.com/mkdocs-material/) theme.

## Topics

| Category | Topics |
|---|---|
| **Social & Media** | Chat App, News Feed, Instagram, Video Streaming, Multimedia Feed |
| **E-Commerce & Payments** | E-Commerce, Mobile Payment, Stock Broking, Hotel Reservation |
| **Productivity & Location** | Offline Document Editor, Location-Based App, Search Autocomplete |
| **Education & Testing** | Live Class App, Test Taking |
| **Infrastructure & SDKs** | Image Loading Library, File Download Manager, Network Layer, Push Notification, Analytics SDK, Feature Flags, Design System, App Update & Migration, Pagination Library |

Each topic includes a **backend architecture** (`generic.md`) and/or **mobile client architecture** (`mobile.md`) perspective, written as interview-ready walkthroughs.

## Getting Started

```bash
# Install dependencies
pip install -r requirements.txt

# Run local dev server
mkdocs serve

# Build static site
mkdocs build
```

## Content Structure

```
designs/
├── index.md
├── best-practices/
├── chat-app/
│   ├── generic.md    # Backend architecture
│   └── mobile.md     # Mobile client architecture
├── news-feed/
│   ├── generic.md
│   └── mobile.md
└── ...
```

## License

Personal knowledge base. Not for redistribution.
