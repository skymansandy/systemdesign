# Reddit Comment UI

Designing a Reddit-style threaded comment UI is a deceptively hard mobile problem. The core challenge isn't rendering text — it's rendering a **recursive tree** with variable depth, collapsible subtrees, vote state, lazy loading of deep threads, and smooth scrolling at 60 fps. Apps like Reddit, Hacker News, and Lemmy all solve this differently. The reason interviewers love it is that every decision — flattening strategy, indentation, collapse mechanics — has real performance implications on mobile devices with bounded screen width and memory.

---

## Scoping the Problem

The first thing I'd want to nail down is the maximum nesting depth. Unbounded depth is unusable on mobile screens — Reddit caps visual indent at ~10 levels. This single constraint drives the entire indentation and "continue this thread" strategy.

Next, I'd ask about comment volume. A post with 50 top-level comments vs. 50,000 fundamentally changes the pagination approach and initial load size. For a viral post, you simply can't send the full tree — partial tree loading with stubs becomes essential.

Other questions that meaningfully change the design:

- **Sorting model?** Best, Top, New, Controversial — sorting a tree is fundamentally different from sorting a flat list, because you're sorting *siblings at each level*, not a global list.
- **Collapse/expand subtrees?** This is the most complex interaction in threaded UIs. It changes visible item count without re-fetching, and must be single-frame fast.
- **Offline reading?** Caching a tree structure in SQLite requires a flattening/reconstruction strategy.
- **Real-time comment updates?** Live updates on a tree while the user is scrolling is a nightmare for scroll position stability. I'd scope this out and show a "3 new comments" banner instead.
- **"Continue this thread" vs "Load more replies"?** These are different operations — one paginates siblings, the other navigates to a deep subtree. Don't conflate them.
- **Rich text (markdown, links, media)?** Inline images and formatted text affect item height calculation dramatically.
- **Reply composing?** Reply-to-specific-comment requires context anchoring in the UI.

!!! tip "Pro Tip"
    Scope aggressively: *"I'll focus on threaded rendering, collapse/expand, voting, partial tree loading, and offline caching. I'll mention real-time updates and rich text as follow-ups."* This shows you understand which parts are architecturally interesting vs. feature work.

**Core scope:** Render threaded comments with collapse/expand, upvote/downvote with optimistic UI, sort comments (Best/Top/New), lazy-load deeper branches, reply to comments, paginate top-level comments, offline reading.

**Key non-functional priorities:**

- **60 fps scrolling** with 500+ visible comments — nested views are a jank magnet
- **< 800ms** to first meaningful comment
- **< 50 MB** for 2,000 comments in memory — deep trees with rich text balloon fast
- **< 16ms collapse latency** (single frame) — collapsing must feel instant
- **Offline caching** of last-viewed comment tree for spotty connectivity

---

## API Design

### Protocol Choice

REST is the right choice here. Comment trees are read-heavy, cacheable, and don't need real-time bidirectional communication.

| Protocol | Verdict | Reasoning |
|----------|---------|-----------|
| **REST** | **Use this** | Read-heavy, highly cacheable, simple pagination model |
| GraphQL | Viable alternative | Useful if clients need to control tree depth per request, but adds complexity |
| WebSocket | Not needed | Real-time comment updates are low-priority; polling or SSE suffice if needed |
| gRPC | Overkill | No inter-service communication benefit on the client side |

### Key Endpoints

```
GET /api/v1/posts/{post_id}/comments
  ?sort=best|top|new|controversial
  &cursor={opaque_cursor}
  &limit=20                          # top-level comments per page
  &depth=5                           # max nesting depth to return
  &child_limit=3                     # max direct children per comment

Response:
{
  "comments": [
    {
      "id": "c_abc123",
      "parent_id": null,              // null = top-level
      "author": { "username": "user_alpha", "avatar_url": "..." },
      "body": "This is a top-level comment...",
      "body_html": "<p>This is a top-level...</p>",
      "score": 890,
      "vote_state": 1,                // -1, 0, 1 for current user
      "created_at": "2025-03-15T10:30:00Z",
      "reply_count": 45,
      "depth": 0,
      "children": [
        { "id": "c_def456", "parent_id": "c_abc123", "depth": 1, ... }
      ],
      "has_more_children": true,
      "more_children_cursor": "cursor_xyz"
    }
  ],
  "next_cursor": "cursor_page2",
  "total_top_level": 342
}
```

```
GET /api/v1/comments/{comment_id}/children
  ?cursor={cursor}
  &limit=20
  &depth=3                            # relative depth from this comment

POST /api/v1/posts/{post_id}/comments
  { "parent_id": "c_abc123", "body": "My reply..." }

PUT /api/v1/comments/{comment_id}/vote
  { "direction": 1 }                  // 1 = upvote, -1 = downvote, 0 = remove
```

!!! tip "Pro Tip"
    The **`depth` and `child_limit` parameters** are the key mobile optimization. Instead of fetching the entire tree (which can be 10,000+ nodes for a viral post), the server returns a "skeleton" — top N levels with M children each — and the client lazily loads deeper branches. This is exactly how Reddit's API works with its `morechildren` endpoint.

---

## Mobile Client Architecture

### Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                        UI Layer                          │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ CommentList │  │ CommentItem  │  │ ReplyComposer   │  │
│  │ (LazyCol.) │  │ (depth-aware)│  │ (bottom sheet)  │  │
│  └─────┬──────┘  └──────┬───────┘  └────────┬────────┘  │
│        │                │                    │           │
│  ┌─────┴────────────────┴────────────────────┴────────┐  │
│  │              CommentTreeViewModel                   │  │
│  │  - flattenedList: List<CommentNode>                 │  │
│  │  - collapsedIds: Set<String>                        │  │
│  │  - voteStates: Map<String, VoteState>               │  │
│  └──────────────────────┬──────────────────────────────┘  │
└─────────────────────────┼────────────────────────────────┘
                          │
┌─────────────────────────┼────────────────────────────────┐
│                    Domain Layer                           │
│  ┌──────────────────────┴──────────────────────────────┐  │
│  │              CommentTreeManager                      │  │
│  │  - buildFlatList(tree, collapsedIds) → List<Node>   │  │
│  │  - insertReply(parentId, comment) → Updated tree    │  │
│  │  - toggleCollapse(commentId) → Updated flat list    │  │
│  └──────────────────────┬──────────────────────────────┘  │
└─────────────────────────┼────────────────────────────────┘
                          │
┌─────────────────────────┼────────────────────────────────┐
│                     Data Layer                            │
│  ┌──────────────┐  ┌────┴───────┐  ┌──────────────────┐  │
│  │ CommentRepo  │  │  TreeCache │  │  VoteRepository  │  │
│  │ (API + DB)   │  │ (in-memory)│  │  (optimistic)    │  │
│  └──────┬───────┘  └────────────┘  └──────────────────┘  │
│         │                                                 │
│  ┌──────┴───────┐  ┌──────────────┐                      │
│  │   Room DB    │  │  Retrofit /  │                      │
│  │ (flat table) │  │  Ktor Client │                      │
│  └──────────────┘  └──────────────┘                      │
└──────────────────────────────────────────────────────────┘
```

The core principle: **the UI reads from a single flat list derived from the tree**. The network fetches tree data, the domain layer flattens it, and the UI renders a simple `LazyColumn`. This clean separation makes collapse, sort changes, and partial loading all manageable.

### The Core Problem: Tree to Flat List

Mobile UI frameworks (RecyclerView, LazyColumn, UICollectionView) are built for **flat lists**, not trees. You cannot nest scroll containers without catastrophic performance. The entire design hinges on one decision: how to flatten the tree.

#### Why Not Nested LazyColumns?

| Approach | Verdict | Why |
|----------|---------|-----|
| **Flat list + depth field** | **Use this** | Single scroll container, predictable performance, works with RecyclerView/LazyColumn |
| Nested LazyColumns | Never | Each nested list has its own scroll state, viewport calculation, and layout pass. 5 levels = 5x layout work per frame |
| WebView with HTML/CSS | Avoid | Loses native scroll physics, accessibility, and makes vote interactions laggy |
| Canvas custom drawing | Overkill | Maximum control but enormous implementation cost; only justified for editors |

#### Flattening Strategy: Pre-order DFS

I'd convert the tree to a flat list using depth-first pre-order traversal. Each item carries its `depth` for indent calculation.

```kotlin
data class CommentNode(
    val comment: Comment,
    val depth: Int,
    val type: NodeType  // COMMENT, LOAD_MORE, COLLAPSED_STUB
)

fun flattenTree(
    comments: List<Comment>,
    collapsedIds: Set<String>,
    depth: Int = 0
): List<CommentNode> = buildList {
    for (comment in comments) {
        add(CommentNode(comment, depth, NodeType.COMMENT))

        if (comment.id in collapsedIds) {
            // Add a single "collapsed" stub instead of children
            add(CommentNode(comment, depth + 1, NodeType.COLLAPSED_STUB))
        } else {
            // Recurse into children
            addAll(flattenTree(comment.children, collapsedIds, depth + 1))

            // Add "load more" stub if there are unfetched children
            if (comment.hasMoreChildren) {
                add(CommentNode(comment, depth + 1, NodeType.LOAD_MORE))
            }
        }
    }
}
```

!!! warning "Edge Case"
    **Collapse performance**: When a user collapses a comment with 500 descendants, you need to remove 500 items from the flat list. Naive recomposition of the entire list causes jank. Solution: compute the **range** of indices to remove (it's always a contiguous range in DFS order) and use `removeRange()` + notify the adapter of the specific range change. In Compose, use `key()` on the comment ID so the framework diffs efficiently.

### Depth Indicator Rendering

Reddit uses **colored vertical bars** on the left edge, not just indentation. This is critical on mobile because pure indentation wastes horizontal space.

```kotlin
@Composable
fun CommentItem(node: CommentNode) {
    Row {
        // Depth indicators: colored vertical bars
        repeat(node.depth) { level ->
            Box(
                modifier = Modifier
                    .width(2.dp)
                    .fillMaxHeight()
                    .background(depthColors[level % depthColors.size])
            )
            Spacer(modifier = Modifier.width(8.dp))
        }

        // Comment content
        Column(modifier = Modifier.weight(1f)) {
            CommentHeader(node.comment)
            CommentBody(node.comment.bodyHtml)
            CommentActions(node.comment) // vote, reply, share
        }
    }
}

// Cap visual indent at depth 5 -- deeper comments still show bars but don't indent further
val effectiveDepth = minOf(node.depth, MAX_VISUAL_DEPTH)
val indentPadding = effectiveDepth * INDENT_PER_LEVEL_DP
```

!!! tip "Pro Tip"
    **Cap visual indentation at 4-5 levels.** After that, the comment text area becomes too narrow to read. Reddit shows a "Continue this thread" link that opens a new screen rooted at the deep comment. This also naturally bounds the tree depth the client needs to handle.

### Collapse/Expand Mechanics

Collapsing is the most latency-sensitive operation. The user taps and expects **instant** visual feedback. The approach: maintain a `Set<String>` of collapsed comment IDs and derive the flat list reactively.

```kotlin
class CommentTreeViewModel : ViewModel() {
    private val _tree = MutableStateFlow<List<Comment>>(emptyList())
    private val _collapsedIds = MutableStateFlow<Set<String>>(emptySet())

    // Derived flat list -- recomputed when tree or collapsed state changes
    val flatList: StateFlow<List<CommentNode>> = combine(_tree, _collapsedIds) { tree, collapsed ->
        flattenTree(tree, collapsed)
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(), emptyList())

    fun toggleCollapse(commentId: String) {
        _collapsedIds.update { ids ->
            if (commentId in ids) ids - commentId else ids + commentId
        }
    }
}
```

!!! warning "Edge Case"
    **Scroll position stability on collapse.** When the user collapses a comment *above* the current viewport, items below shift up. If you don't compensate, the content the user is reading jumps. Solution: before collapsing, record the first visible item's key and offset. After the flat list updates, use `LazyListState.scrollToItem()` to restore the position. Reddit's own app gets this wrong sometimes — don't repeat their mistake.

### Vote State Management

Votes must be **optimistic** — the score updates instantly, and the API call fires in the background.

```kotlin
data class VoteState(
    val displayScore: Int,      // what the user sees
    val userVote: Int,          // -1, 0, 1
    val pendingSync: Boolean    // true if API call in flight
)

fun onVote(commentId: String, direction: Int) {
    // 1. Compute new state immediately
    val current = voteStates[commentId]
    val scoreDelta = direction - current.userVote
    val newState = current.copy(
        displayScore = current.displayScore + scoreDelta,
        userVote = direction,
        pendingSync = true
    )
    // 2. Update UI instantly
    _voteStates.update { it + (commentId to newState) }

    // 3. Fire-and-forget API call with rollback on failure
    viewModelScope.launch {
        try {
            api.vote(commentId, direction)
            _voteStates.update { it + (commentId to newState.copy(pendingSync = false)) }
        } catch (e: Exception) {
            // Rollback to previous state
            _voteStates.update { it + (commentId to current.copy(pendingSync = false)) }
        }
    }
}
```

### Local Storage Schema

For offline caching, I'd store comments in a **flat table** (not nested JSON). The tree is reconstructed at read time.

```sql
CREATE TABLE comments (
    id              TEXT PRIMARY KEY,
    post_id         TEXT NOT NULL,
    parent_id       TEXT,                   -- NULL for top-level
    author          TEXT NOT NULL,
    body            TEXT NOT NULL,
    body_html       TEXT,
    score           INTEGER DEFAULT 0,
    user_vote       INTEGER DEFAULT 0,      -- -1, 0, 1
    depth           INTEGER NOT NULL,
    reply_count     INTEGER DEFAULT 0,
    has_more        INTEGER DEFAULT 0,      -- boolean: has unfetched children
    created_at      INTEGER NOT NULL,       -- epoch millis
    cached_at       INTEGER NOT NULL,       -- for LRU eviction
    sort_order      INTEGER NOT NULL        -- pre-computed position in sorted tree
);

CREATE INDEX idx_comments_post ON comments(post_id, sort_order);
CREATE INDEX idx_comments_parent ON comments(parent_id);
```

!!! tip "Pro Tip"
    The `sort_order` column is key. When the server returns comments in "Best" sort order, store the DFS traversal position. This lets you `SELECT * FROM comments WHERE post_id = ? ORDER BY sort_order` and get the correctly-sorted flat list without rebuilding the tree. When the user changes sort order, re-fetch and update `sort_order`.

### Partial Tree Loading

A viral post with 50,000 comments cannot be sent in one response. The server returns a **partial tree** with stubs.

```
Depth 0: Top-level comment A (20 top-level loaded, cursor for more)
  Depth 1: Reply B (3 of 45 children loaded)
    Depth 2: Reply C
    Depth 2: Reply D
    Depth 2: [LOAD_MORE: 42 more replies, cursor=xyz]
  Depth 1: Reply E
  Depth 1: [LOAD_MORE: 12 more replies, cursor=abc]
Depth 0: Top-level comment F
  ...
```

When the user taps "Load more":

1. Call `GET /comments/{parent_id}/children?cursor=xyz`
2. Insert the returned comments into the tree at the correct position
3. Replace the `LOAD_MORE` stub with the new comments
4. Re-flatten and update the list

!!! warning "Edge Case"
    **"Continue this thread" vs "Load more replies"** — these are different operations. "Load more replies" fetches *sibling* comments at the same depth (pagination). "Continue this thread" navigates to a new screen rooted at a deep comment (depth > max). Don't conflate them in your data model or UI — they have different API calls, different navigation flows, and different UX.

---

## Scalability, Reliability & Edge Cases

- **Deleted comments with children** — Show `[deleted]` placeholder, keep children visible. Removing a parent would orphan or hide all replies.
- **Rapid vote toggling** — Debounce API calls (300ms), always use latest local state. Prevents spamming the server while keeping the UI responsive.
- **Deep thread on narrow screen** — Cap indent at depth 5, show "Continue this thread." Text becomes unreadable below 4-5 indent levels.
- **Process death during reply compose** — Save draft to `SavedStateHandle` keyed by `parent_id`. Users lose work if the OS kills the app mid-reply.
- **Real-time new comments** — Don't auto-insert into the tree while scrolling. It breaks scroll position and confuses the user. Show a "3 new comments" banner at the top instead.
- **Comment with inline image** — Async height measurement with placeholder. Unknown heights cause list jumps; use aspect ratio hints from the server.
- **Comment with 0 score** — Show "0", not blank. Users need to know their vote changed the score.

---

## Wrap Up

- **Flatten the tree** into a single-level list using DFS pre-order traversal. Never nest scroll containers.
- **Depth indicators via colored bars**, not indentation alone. Cap visual depth at 4-5 levels.
- **Collapse is a local operation** — toggle a set of collapsed IDs and re-derive the flat list. Must be single-frame fast.
- **Partial tree loading** with `LOAD_MORE` stubs. The server controls how much tree to return via `depth` and `child_limit` params.
- **Optimistic votes** with rollback. The score updates before the API call completes.
- **Flat table storage** with `sort_order` column for offline caching. Reconstruct tree only when needed.

**With more time**, I'd design: real-time comment streaming with scroll-safe insertion, rich text rendering with inline media and code blocks, accessibility support (TalkBack navigation of nested trees), and animated collapse/expand transitions.

---

## References

- [Reddit's API documentation](https://www.reddit.com/dev/api/) — `morechildren` endpoint for partial tree loading
- [Building a Reddit-like Nested Comment System](https://blog.devgenius.io/) — Materialized path vs adjacency list trade-offs
- [Jetpack Compose LazyColumn performance](https://developer.android.com/develop/ui/compose/lists) — Google's official guidance on large list performance
- [How Reddit ranks comments](https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9) — Wilson score interval for "Best" sorting
