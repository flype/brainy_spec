# Brainy - Smart Bookmark Vault Specification v0.5.0

## Overview

Brainy is a single-user, personal bookmark knowledge base that ingests URLs from multiple sources, extracts and enriches content using AI, stores it in a relational database with vector embeddings for hybrid semantic+lexical search, and maintains a knowledge graph for entity-based discovery. Users ask natural language questions and receive AI-generated answers with citations from their bookmark collection.

The system is designed to run on localhost or a private network with no authentication. All API endpoints are unauthenticated.

The system processes bookmarks asynchronously through a job queue, supports multiple content platforms (YouTube, Twitter/X, Instagram, TikTok, generic web), detects paywalls with archive fallback, and provides multilingual search (English/Spanish) with configurable embedding providers.

## Design Principles

1. **Async-first ingestion.** Bookmark saving returns immediately with a job ID. All heavy processing (scraping, embedding, graph extraction, chunking) happens in background workers. The user never waits for content processing.

2. **Graceful degradation.** Every optional subsystem (knowledge graph, archive fallback, AI content cleaner, thumbnail extraction, summary generation) can fail without blocking bookmark creation. The core path (scrape -> embed -> store) always completes if the URL is reachable.

3. **Platform-aware extraction.** Each content platform (YouTube, Twitter, Instagram, TikTok) has a specialized extractor that understands the platform's content structure. All platform extractors fall back to generic webpage scraping on failure.

4. **Hybrid search with tunable weights.** Search combines semantic (vector cosine similarity) and lexical (full-text) signals via Reciprocal Rank Fusion (RRF). Weights are dynamically adjusted based on query intent classification.

5. **Knowledge graph as enrichment layer.** The graph database stores extracted entities, categories, and concepts as a discovery and filtering mechanism. The relational database remains the source of truth for bookmark data. Graph operations never block core CRUD.

6. **Fire-and-forget post-processing.** Graph entity extraction and content chunking run in background tasks after the bookmark is persisted. Their success or failure does not affect the bookmark's existence.

---

## System Invariants

Properties that must hold across ALL implementations, regardless of language, architecture, or internal design.

- **INV-001**: Every bookmark has non-empty `content`. *Rationale: content is the foundation for embedding generation and search. A bookmark without content cannot participate in hybrid search.*

- **INV-002**: Non-null URLs are unique across all bookmarks. Multiple bookmarks with NULL URLs (standalone notes) are permitted. *Rationale: prevents duplicate bookmarks for the same resource while allowing unlimited freeform notes.*

- **INV-003**: `read_status` is always set (never null), defaulting to `false`. *Rationale: UI filtering by read status requires a definite boolean value.*

- **INV-004**: The `tsv` (English) and `tsv_es` (Spanish) full-text search vectors are always in sync with `title` + `content`. The `note_tsv` vector is always in sync with `notes`. *Rationale: these are auto-computed columns that recompute on any change, ensuring search results are never stale.*

- **INV-005**: When `is_chunked = true` on a bookmark, `chunk_count > 0` and exactly `chunk_count` rows exist in `bookmark_chunks` for that bookmark ID. When `is_chunked = false`, `chunk_count = 0` and no chunks exist. *Rationale: chunk metadata must match actual chunk data for unified search to work correctly.*

- **INV-006**: Deleting a bookmark cascades to delete all its chunks (`bookmark_chunks`) and associated jobs (`bookmark_jobs`). *Rationale: no orphaned data should remain after bookmark deletion.*

- **INV-007**: Job status follows a strict state machine: `pending` -> `processing` -> `completed` | `failed`. No other transitions are valid. A job cannot return to `pending` after leaving it. *Rationale: clients polling job status rely on monotonic state progression.*

- **INV-008**: The `updated_at` timestamp on bookmarks auto-updates on every row modification via database trigger. *Rationale: consumers of bookmark data can rely on `updated_at` for change detection and cache invalidation.*

- **INV-009**: Embedding vectors have exactly the dimensions configured for the active embedding provider. All vectors in the database use the same dimensionality. *Rationale: vector indexes require consistent dimensions for index operations and cosine similarity calculations.*

- **INV-010**: In the knowledge graph, all entity nodes have dual labels: their specific type label (e.g., `:Person`) plus the generic `:Entity` label. Category and Concept nodes have only their respective single label. *Rationale: enables both type-specific and generic entity queries.*

- **INV-011**: Graph node creation uses MERGE (not CREATE) keyed on `name` for categories/concepts/entities and on `id` for bookmarks. This prevents duplicate graph nodes. *Rationale: idempotent graph operations are essential for retry safety and re-indexing.*

- **INV-012**: When a bookmark is deleted from the graph, any category/concept/entity nodes that become orphaned (no remaining relationships) are also deleted. *Rationale: prevents graph pollution with disconnected nodes that no longer relate to any bookmark.*

### Verification

Each invariant should be verifiable by:
1. Running after any state-mutating operation (bookmark create/update/delete)
2. Running as a continuous production check (periodic database consistency queries)
3. Including in property-based test suites with generated inputs

---

## Behavioral Properties

Universal truths about function behavior that hold for ALL valid inputs.

- **PROP-001**: `ChunkContent`: For any content string and valid chunking config, the union of all chunk primary regions (excluding overlap) covers the entire content with no gaps. *Formal: `join(chunks[i].content[0:end-overlap]) == original_content` when accounting for boundary chunks*

- **PROP-002**: `ChunkContent`: Idempotent in structure. Chunking the same content with the same config always produces the same number of chunks with the same boundaries. *Formal: `ChunkContent(content, config) == ChunkContent(content, config)` for all valid inputs*

- **PROP-003**: `ChunkContent`: Content that fits in a single chunk (`len(content) <= MaxChunkSize`) produces exactly one chunk spanning the entire content. *Formal: `len(content) <= MaxChunkSize => len(ChunkContent(content)) == 1 && chunks[0].content == content`*

- **PROP-004**: `detectContentType`: URL type detection is mutually exclusive and total. Every URL maps to exactly one content type: `youtube`, `twitter`, `instagram`, `tiktok`, or `webpage`. *Formal: `|{t : detectContentType(url) == t}| == 1` for all valid URLs*

- **PROP-005**: `ClassifyQuery`: Query classification is deterministic. The same query text with the same `hasContext` flag always produces the same intent and weights. *Formal: `ClassifyQuery(q, ctx) == ClassifyQuery(q, ctx)` for all valid q, ctx*

- **PROP-006**: `rrf_score`: RRF scores are bounded in (0, 1/rrf_k] for non-null ranks and exactly 0 for null ranks. *Formal: `rank != null => 0 < rrf_score(rank, k) <= 1/k` and `rank == null => rrf_score(rank, k) == 0`*

- **PROP-007**: `rrf_score`: RRF scores are monotonically decreasing with increasing rank. *Formal: `rank_a < rank_b => rrf_score(rank_a, k) > rrf_score(rank_b, k)` for all valid k > 0*

- **PROP-008**: `HybridSearch`: Search results are ordered by combined score descending. *Formal: `results[i].score >= results[i+1].score` for all valid i*

- **PROP-009**: `UpsertBookmark` with URL: Upserting a bookmark with an existing URL updates the existing row rather than creating a new one. The bookmark ID is preserved. *Formal: `upsert(b{url=u}).id == existing(u).id` when a bookmark with URL u already exists*

- **PROP-010**: `DeleteBookmark`: After deleting a bookmark by ID, `GetBookmark(id)` returns not-found, no chunks exist for that ID, and the graph node is removed. *Formal: `delete(id) => !exists(bookmark[id]) && count(chunks[bookmark_id=id]) == 0 && !exists(graph_node[id])`*

- **PROP-011**: `cleanText`: Text cleaning is idempotent. Cleaning already-cleaned text produces identical output. *Formal: `cleanText(cleanText(text)) == cleanText(text)`*

- **PROP-012**: `hybrid_search_bookmarks`: Results with score <= 0.01 are always filtered out. *Formal: `for all r in results: r.combined_score > 0.01`*

- **PROP-013**: `GenerateContentSummary`: Content type determines summary format. YouTube content always produces timestamped outlines, articles always produce 5-section numbered structure, social media always produces 2-3 paragraphs. *Formal: `contentType == "youtube" => summary matches timestamped format` and `contentType in ("webpage", "article") => summary matches 5-section format` and `contentType in ("twitter", "tiktok") => paragraph_count(summary) <= 3`*

- **PROP-014**: `GenerateContentSummary`: No summary output begins with preamble phrases. *Formal: `for all summaries s: !starts_with(s, "Here is") && !starts_with(s, "Of course") && !starts_with(s, "Sure") && !starts_with(s, "I've created") && !starts_with(s, "Below is")`*

- **PROP-015**: `getSystemPromptForIntent`: Every non-command intent produces a system prompt that contains the base prompt as a prefix and includes citation `[N]` instructions. *Formal: `for all intents i where i != "command": starts_with(getSystemPromptForIntent(i), basePrompt) && contains(getSystemPromptForIntent(i), "[N]")`*

- **PROP-016**: `buildContextTemplate`: The context template for N search results contains exactly N source blocks numbered `[1]` through `[N]`, and ends with the user's question. *Formal: `for i in 1..N: contains(context, "Source [" + i + "]")` and `ends_with(context, "User Question: " + query)`*

---

## Interface Contracts

Contracts specify what crosses boundaries between components. Each contract survives reimplementation of either side.

### Client -> Backend: Add Bookmark

**Protocol**: HTTP POST

**Endpoint**: `/add` (alias: `/note` — identical behavior, provided for semantic clarity in UI code)

**Request schema**:
```
{
  "url": string (optional) -- The URL to bookmark,
  "notes": string (optional) -- Freeform notes or text,
  "created_at": string (optional) -- ISO 8601 timestamp for backdating
}
```
At least one of `url` or `notes` must be provided.

**Response schema (202 Accepted)**:
```
{
  "success": true,
  "message": "Bookmark queued for processing",
  "job_id": string -- UUID for polling job status
}
```

**Response schema (409 Conflict)**:
```
{
  "success": false,
  "message": "This URL has already been bookmarked",
  "id": string -- UUID of existing bookmark
}
```

**Error schema (400)**:
```
{
  "success": false,
  "message": "Either URL or notes must be provided"
}
```

**Contract invariants**:
- A successful 202 response always contains a valid UUID `job_id`
- The `job_id` is immediately queryable via `GET /job/{id}`
- Duplicate URL detection is performed before job creation
- The HTTP response returns before any scraping/embedding work begins

---

### Client -> Backend: Ask Question (SSE Stream)

**Protocol**: HTTP GET with Server-Sent Events response

**Endpoint**: `/answer?q={query}`

**Request**:
- Query parameter `q` (required): natural language question
- Header `X-Has-Context: true|false` (optional): signals follow-up question

**SSE Event Types**:

| Event | Data Format | Description |
|-------|-------------|-------------|
| `data:` (no event name) | Plain text chunk | Answer text fragment (newlines escaped as `\\n`) |
| `event: citation` | JSON object | A cited source bookmark |
| `event: error` | Plain text | Error message |
| `event: done` | Empty | Stream complete |

**Citation event data schema**:
```
{
  "number": integer -- Citation reference number [N] in the answer text,
  "id": string -- Bookmark UUID,
  "title": string -- Bookmark title,
  "url": string -- Original URL,
  "domain": string -- URL domain,
  "content_type": string -- "webpage", "youtube", "twitter", etc.,
  "created_at": string -- ISO 8601 date
}
```

**Contract invariants**:
- Every citation `number` corresponds to a `[N]` reference in the answer text
- Only actually-cited sources are sent as citation events (not all search results)
- The stream always terminates with an `event: done` event
- Citation events are sent after all answer text events

---

### Client -> Backend: List Bookmarks

**Protocol**: HTTP GET

**Endpoint**: `/bookmarks`

**Request** (query parameters):
```
{
  "limit": integer (optional, default 50, max 100),
  "offset": integer (optional, default 0),
  "read_status": "true" | "false" | "all" (optional),
  "category": string (optional),
  "content_type": "video" | "text" | "all" (optional),
  "search": string (optional),
  "nodes[]": string[] (optional, graph node names for filtering)
}
```

**Response schema (200)**:
```
{
  "bookmarks": [RecentBookmark],
  "total": integer,
  "limit": integer,
  "offset": integer,
  "category": string,
  "search": string
}
```

**RecentBookmark type**:
```
{
  "id": string (uuid),
  "url": string,
  "title": string,
  "content": string,
  "notes": string (optional),
  "summary": string,
  "snippet": string -- First 200 characters of content, used as a preview in list views,
  "read_status": boolean,
  "read_at": string (optional, ISO 8601 timestamp),
  "created_at": string (ISO 8601 timestamp),
  "metadata": json_object (optional, see Metadata Schema below)
}
```

**Contract invariants**:
- `bookmarks.length <= limit`
- When `total >= 0`: `offset + bookmarks.length <= total`
- Results are ordered by `created_at DESC` (for non-search queries) or by relevance score DESC (for search queries)
- `total` may be `-1` (count error) or `-2` (sentinel for "many results" when the total count would be expensive to compute, e.g., >10,000 bookmarks)
- When `nodes[]` is provided, filtering works by querying the knowledge graph for bookmark IDs connected to the named nodes, then filtering the relational query to only those IDs

---

### Client -> Backend: Delete Bookmark

**Protocol**: HTTP DELETE

**Endpoint**: `/bookmark/{id}`

**Response (204 No Content)**: Empty body on success.

**Response (404)**:
```
{
  "success": false,
  "message": "Bookmark not found"
}
```

**Contract invariants**:
- After a 204 response, the bookmark, its chunks, and its graph node are all deleted
- Graph deletion failure does not cause a non-204 response (best-effort)
- The operation is idempotent in effect (deleting an already-deleted bookmark returns 404)

---

### Client -> Backend: Job Status (Polling)

**Protocol**: HTTP GET

**Endpoint**: `/job/{id}`

**Response schema (200)**:
```
{
  "id": string,
  "url": string,
  "status": "pending" | "processing" | "completed" | "failed",
  "bookmark_id": string (present when status == "completed"),
  "retry_count": integer,
  "error": string (present when status == "failed"),
  "created_at": string,
  "started_at": string (nullable),
  "completed_at": string (nullable),
  "metadata": object
}
```

**Contract invariants**:
- `status` only moves forward: pending -> processing -> completed|failed
- `bookmark_id` is always present when `status == "completed"`
- `error` is always present when `status == "failed"`

---

### Client -> Backend: Job Status (SSE Stream)

**Protocol**: HTTP GET with Server-Sent Events response

**Endpoint**: `/job/{id}/sse`

**SSE Event Types**:

| Event | Data Format | Description |
|-------|-------------|-------------|
| `event: status` | JSON (same as polling response) | Current job status |
| `event: error` | Plain text | Error message |
| `event: done` | `"Job finished"` | Stream complete |

**Contract invariants**:
- Initial status is sent immediately upon connection
- Status updates are polled every 500ms
- Stream closes when job reaches `completed` or `failed`
- Stream closes when client disconnects

---

### Client -> Backend: Bookmark Detail

**Protocol**: HTTP GET

**Endpoint**: `/bookmark/detail?id={uuid}`

**Response schema (200)**:
```
{
  "id": string,
  "content": string,
  "read_status": boolean,
  "created_at": string,
  "metadata": json_object (optional, see Metadata Schema below),
  "url": string (optional),
  "title": string (optional),
  "summary": string (optional),
  "notes": string (optional),
  "read_at": string (optional),
  "graph_data": {
    "categories": [{name, description, level}],
    "concepts": [{name, description}],
    "entities": [{name, type}],
    "topics": [{name}]
  } (optional, when graph service enabled)
}
```

**Contract invariants**:
- When `graph_data` is present, all array fields (`categories`, `concepts`, `entities`, `topics`) MUST be empty arrays (`[]`), never `null`. Consumers rely on calling array methods (e.g., `.map()`) on these fields without null checks.

---

### Metadata Schema

The `metadata` JSON field stores platform-specific data extracted during ingestion. Its contents vary by content type:

| Content Type | Known Keys | Description |
|-------------|------------|-------------|
| `youtube` | `channel`, `duration`, `video_id`, `thumbnail_url`, `publish_date` | YouTube video metadata |
| `twitter` | `author`, `author_handle`, `tweet_id`, `retweet_count`, `like_count` | Tweet/thread metadata |
| `instagram` | `author`, `author_handle`, `post_type`, `thumbnail_url` | Instagram post metadata |
| `tiktok` | `author`, `author_handle`, `video_id`, `thumbnail_url` | TikTok video metadata |
| `webpage` | `og_image`, `author`, `publish_date`, `description`, `keywords` | Generic webpage metadata extracted from meta tags and JSON-LD |

All keys are optional. Implementations may store additional platform-specific keys beyond those listed here. The `metadata` field may be `null` or an empty object if no metadata was extracted.

---

### Client -> Backend: Toggle Read Status

**Protocol**: HTTP PATCH

**Endpoint**: `/bookmark/{id}/read-status`

**Response schema (200)**:
```
{
  "id": string,
  "url": string,
  "read_status": boolean,
  "updated_at": string,
  "read_at": string (present when read_status == true)
}
```

**Contract invariants**:
- Toggles the current boolean value (true -> false, false -> true)
- `read_at` is set to current timestamp when toggling to read, cleared when toggling to unread

---

### Client -> Backend: Update Notes

**Protocol**: HTTP PUT

**Endpoint**: `/bookmark/{id}/notes`

**Request schema**:
```
{
  "notes": string
}
```

**Response schema (200)**:
```
{
  "success": true,
  "message": "Bookmark notes updated successfully"
}
```

**Side effect**: If graph service is available and notes are non-empty, triggers async graph re-indexing.

---

### Client -> Backend: Re-extract Content

**Protocol**: HTTP PUT

**Endpoint**: `/bookmark/{id}/reextract`

**Response schema (202 Accepted)**:
```
{
  "success": true,
  "message": "Re-extraction queued for processing",
  "job_id": string,
  "bookmark_id": string
}
```

**Contract invariants**:
- Only works for bookmarks that have a URL (returns 400 for notes-only)
- Creates a new job that updates the existing bookmark rather than creating a duplicate
- Notes are preserved during re-extraction
- All derived data is regenerated: content, embedding, summary, chunks (if applicable), and graph entities. If summary generation fails, the existing summary is preserved via null-coalescing update.

---

### Client -> Backend: Check URL Existence

**Protocol**: HTTP POST

**Endpoint**: `/bookmarks/check`

**Request schema**:
```
{
  "url": string
}
```

**Response schema (200)**:
```
{
  "exists": boolean,
  "bookmark_id": string (present when exists == true)
}
```

---

### Client -> Backend: Health Check

**Protocol**: HTTP GET

**Endpoint**: `/health`

**Response schema (200)**:
```
{
  "status": "healthy"
}
```

**Contract invariant**: Always returns 200 if the server process is running.

---

### Client -> Backend: Autocomplete

**Protocol**: HTTP GET

**Endpoint**: `/autocomplete?q={query}&limit={n}`

**Response schema (200)**:
```
[
  {
    "name": string -- Node name (e.g., "React", "Technology"),
    "type": string -- Node type: "Category" | "Concept" | "Topic" | "Person" | "Organization" | "Technology" | "Project",
    "count": integer -- Number of bookmarks associated with this node,
    "description": string (optional) -- Node description,
    "score": float (optional) -- Relevance score (1.0 = exact match, 0.8 = starts-with, 0.6 = contains)
  }
]
```

**Contract invariants**:
- Returns empty array `[]` on any error or when graph service is unavailable
- `limit` is capped at 20
- Results are ordered by relevance score descending, then bookmark count descending

---

### Client -> Backend: Related Bookmarks

**Protocol**: HTTP GET

**Endpoint**: `/bookmarks/related?type={type}&name={name}`

**Request** (query parameters):
```
{
  "type": "category" | "concept" | "entity" (required),
  "name": string (required) -- Graph node name to search by
}
```

**Response schema (200)**:
```
{
  "bookmarks": [
    {
      "id": string,
      "url": string,
      "title": string,
      "score": float,
      "categories": [string],
      "concepts": [string],
      "entities": [string],
      "metadata": json_object (optional),
      "created_at": string,
      "graph_data": object (optional, when graph service available)
    }
  ],
  "total": integer,
  "type": string -- Echo of the requested type,
  "name": string -- Echo of the requested name
}
```

**Error schema (400)**: Missing or invalid `type` or `name` parameter.

**Error schema (503)**: Graph service unavailable.

**Contract invariants**:
- `type` must be one of: `"category"`, `"concept"`, `"entity"`
- Both `type` and `name` are required
- If the graph service is unavailable, returns 503 (not an empty result)
- The `score` field represents graph relationship strength: a measure of how strongly the bookmark is connected to the queried node (e.g., number of shared nodes, relationship depth)

---

### Client -> Backend: Categories

**Protocol**: HTTP GET

**Endpoint**: `/categories`

**Request** (query parameters):
```
{
  "with_counts": "true" (optional) -- Include bookmark and subcategory counts
}
```

**Response schema (200)**:
```
{
  "categories": [
    {
      "name": string,
      "description": string (optional),
      "level": integer,
      "bookmark_count": integer (only when with_counts=true),
      "sub_category_count": integer (only when with_counts=true)
    }
  ],
  "count": integer -- Total number of categories
}
```

**Error schema (501)**: Graph service not configured.

**Contract invariants**:
- Requires graph service; returns 501 if not available
- `bookmark_count` and `sub_category_count` are only present when `with_counts=true`

---

## Output Structure

**Do generate:**
- Backend server binary (HTTP API + static file server)
- Database migration files (relational schema + vector indexes)
- Static web UI (single HTML file, no build step)

**Do not generate:**
- Raycast extension (separate repository)
- Chrome extension distribution packages
- iOS Shortcut files (manual creation)
- Graph database schema (managed via application-level MERGE operations)

---

## Type Conventions

Types are described abstractly. Implementations should map to idiomatic types for their target language and database.

| Spec type | Meaning |
|-----------|---------|
| `string` | Variable-length text |
| `integer` | Whole number |
| `float` | Floating-point number |
| `boolean` | True/false |
| `uuid` | Universally unique identifier (RFC 4122) |
| `timestamp` | Date/time with timezone (ISO 8601) |
| `vector` | Fixed-length array of floats (embedding) |
| `tsvector` | Full-text search index (auto-generated from content) |
| `json_object` | Arbitrary key-value structure |
| `enum` | Constrained string with defined allowed values |

---

## Error Handling

Errors are reported as JSON responses with appropriate HTTP status codes:

| Status Code | Meaning | When Used |
|-------------|---------|-----------|
| 400 | Bad Request | Missing required fields, invalid parameters |
| 404 | Not Found | Bookmark or job ID does not exist |
| 409 | Conflict | URL already bookmarked (duplicate) |
| 500 | Internal Server Error | Database errors, embedding generation failures |
| 501 | Not Implemented | Graph service endpoint when graph is not configured |
| 503 | Service Unavailable | Graph service temporarily unavailable |

SSE streams report errors as `event: error` events rather than HTTP status codes, since the connection is already established.

---

## Content Type Detection

URL type detection follows a priority-ordered chain. The first match wins:

| Priority | Content Type | URL Pattern |
|----------|-------------|-------------|
| 1 | `youtube` | `youtube.com/watch`, `youtu.be/`, `youtube.com/shorts/`, `youtube.com/embed/`, `youtube.com/live/`, `m.youtube.com/watch` |
| 2 | `twitter` | `twitter.com/*/status/*`, `x.com/*/status/*`, `threadreaderapp.com/thread/*` |
| 3 | `instagram` | `instagram.com/p/*`, `instagram.com/reel/*`, `instagram.com/tv/*` |
| 4 | `tiktok` | `tiktok.com/@*/video/*` (15-19 digit IDs), `vt.tiktok.com/*`, `vm.tiktok.com/*` |
| 5 | `webpage` | Everything else (default) |

---

## Hybrid Search Algorithm

### Scoring Methods

The system uses two distinct scoring methods depending on the detected language of the query:

**Path A: Direct Similarity Blending** (default, English queries)
```
combined_score = MAX((cosine_similarity * 0.6) + (normalized_text_rank * 0.4), 0.01)
```
Where `cosine_similarity = 1 - cosine_distance` and `normalized_text_rank = full_text_rank / 10.0`.

**Path B: Reciprocal Rank Fusion** (non-English queries, e.g., Spanish)
```
rrf_score(rank, k) = 1.0 / (k + rank)   -- where k defaults to 50
final_score = (rrf_score(semantic_rank, k) * semantic_weight) + (rrf_score(keyword_rank, k) * lexical_weight)
```

**Path selection**: When the query is detected as non-English (e.g., Spanish), the system uses Path B (RRF) which queries both the English (`tsv`) and Spanish (`tsv_es`) full-text search vectors and merges results via rank fusion. English queries use Path A (Direct Similarity Blending) with only the English `tsv` vector.

### Query Intent Classification

Queries are classified into intents that adjust search weights:

| Intent | Semantic Weight | Lexical Weight | Max Results | Trigger Condition |
|--------|----------------|----------------|-------------|-------------------|
| URL-Specific | 0.1 | 0.9 | 1 | Query contains a URL |
| Command | 0.0 | 0.0 | 0 | "help", "commands" |
| Conversational | 0.7 | 0.3 | 15 | Has context flag + follow-up patterns |
| Temporal | 0.5 | 0.5 | 20 | Time-related words ("recent", "yesterday") |
| Graph | 0.4 | 0.1 | 20 | Category/concept patterns ("bookmarks about X") |
| Comparative | 0.7 | 0.3 | 25 | Comparison words ("vs", "compare", "difference") |
| Navigational | 0.3 | 0.7 | 20 | Finding patterns ("find", "show me", "where is") |
| Author-Specific | 0.4 | 0.6 | 30 | Author patterns ("by {name}", "{name}'s articles") |
| Informational | 0.6 | 0.4 | 20 | Default (no other match) |

Classification priority: URL-Specific > Command > Conversational > Temporal > Graph > Comparative > Navigational > Author-Specific > Informational.

### Unified Search (Chunked + Non-chunked)

For large documents that have been chunked:
1. Search chunked bookmarks via `hybrid_search_chunks` (returns best chunk per bookmark)
2. Search non-chunked bookmarks via `hybrid_search_bookmarks`
3. Merge results, deduplicate by bookmark ID, sort by combined score
4. If chunk search fails, gracefully degrade to non-chunked results only

---

## Answer Generation

The `/answer` endpoint generates AI-powered answers to user questions using search results as context. The answer system uses a two-layer prompt architecture: a base system prompt shared across all intents, plus intent-specific suffixes that tailor the AI's behavior.

### Base System Prompt

All answer generation uses a shared base prompt that establishes the AI's role:

```
You are an advanced AI assistant with access to a comprehensive knowledge base of bookmarked web pages. You have been provided with complete bookmark content, metadata, and knowledge graph relationships.

Key capabilities:
- Analyze full document content (not just snippets)
- Understand relationships between bookmarks through categories and concepts
- Synthesize information across multiple sources
- Provide nuanced, comprehensive answers

When analyzing sources, note the relevance scores to prioritize highly relevant content.
```

### Intent-Specific System Prompts

Each query intent appends a specific suffix to the base prompt that adjusts the AI's behavior and output format:

| Intent | Format Requirements | Key Instructions |
|--------|-------------------|------------------|
| **Navigational** | Structured bookmark card format with emoji headers, source/category/date metadata, graph pills, related content | Present comprehensive bookmark detail with categories, concepts, entities, and platform-specific metadata. Use `## 📖 [Title](URL)` format with **Source**, **Category**, **Added** metadata line |
| **Comparative** | Sections for each comparison aspect, tables when helpful, synthesis section | Read ALL sources thoroughly, use relevance scores, create comparison tables, include specific examples and quotes with `[N]` citations |
| **Temporal** | Chronological or recency-based ordering | Focus on temporal aspect, prioritize newer content, highlight dates and time-sensitive information. Cite sources `[N]` |
| **Conversational** | Natural conversational style | More casual tone, acknowledge connection to previous context. Inline citations `[N]` |
| **Graph** | Group related bookmarks by categories/concepts with headers | Explain how bookmarks are related through shared categories, concepts, topics. Leverage graph metadata for richer context. Cite sources `[N]` |
| **Informational** (default) | Markdown with bold, bullet points, code blocks, clear sections and headers | Thoroughly analyze full content, only cite relevant sources `[N]`, include specific details/examples/quotes, note metadata when relevant. Structure longer answers with headers |

**Prompt invariants:**
- All intents MUST include the base system prompt as prefix
- All intents MUST instruct the AI to cite sources using `[N]` notation
- The informational intent is the default when no other intent matches
- Command intent does NOT use an LLM call — it returns a static help message

### Context Template Format

Search results are formatted into a structured context block that becomes the user prompt. Each source follows this format:

```
Here are the relevant bookmarks:

[Optional: "Note: N related bookmarks are included based on shared categories/concepts."]

=== Source [1] (Relevance Score: 0.892) ===
Title: Example Article
URL: https://example.com/article
Added: 2025-06-15
Metadata:
  author: John Doe
  duration: 45:30
Content Type: article
Language: en
Categories: Technology, Web Development
Concepts: React, Frontend Architecture
Entities: React (Technology), Meta (Organization)
Content:
[full bookmark content]
User Notes: Great reference for component patterns

=== Source [2] (Relevance Score: 0.756) ===
...

User Question: [the user's original query]
```

**Context template invariants:**
- Each source MUST include a `[N]` number matching the citation system
- Each source MUST include the relevance score
- Full content is included (not truncated), enabling deep analysis
- Graph metadata (categories, concepts, entities) is included when the graph service is available
- User notes are included when present
- Metadata fields vary by content type (duration for video, author for articles, etc.)

### Result Filtering

Before building the context, search results are filtered:
- Maximum 20 results included in the context
- Results with score > 0.3 are included even beyond the 20-result limit
- Results are ordered by combined score descending

### Search Strategy Parameters

Each intent configures different search behavior:

| Intent | Semantic Weight | Lexical Weight | Max Results | Max Tokens | Special Behavior |
|--------|----------------|----------------|-------------|------------|------------------|
| URL-Specific | 0.1 | 0.9 | 1 | 50,000 | Graph search enabled (depth 3) |
| Command | 0.0 | 0.0 | 0 | 0 | No search performed |
| Conversational | 0.7 | 0.3 | 15 | 100,000 | No snippets |
| Temporal | 0.5 | 0.5 | 20 | 50,000 | Boost recent content |
| Graph | 0.4 | 0.1 | 20 | 150,000 | Graph search (depth 2-3) |
| Comparative | 0.7 | 0.3 | 25 | 200,000 | More results for comparison |
| Navigational | 0.3 | 0.7 | 20 | 50,000 | Boost exact matches |
| Author-Specific | 0.4 | 0.6 | 30 | 100,000 | Boost lexical for names |
| Informational | 0.6 | 0.4 | 20 | 100,000 | Default |

---

## Content Chunking

### Configuration

| Parameter | Default Value | Description |
|-----------|--------------|-------------|
| `MaxChunkSize` | 4000 chars (~1000 tokens) | Maximum characters per chunk |
| `OverlapSize` | 400 chars (~100 tokens) | Overlap between adjacent chunks |
| `MinChunkSize` | 100 chars | Minimum viable chunk size |
| `ChunkingThreshold` | 24000 chars | Content length above which chunking is applied |

### Algorithm

1. If content fits in one chunk, return a single chunk
2. Sliding window: advance by `MaxChunkSize - OverlapSize` characters per step
3. At each step, find the best break point (sentence boundary preferred, then word boundary)
4. Sentence boundaries: `.`, `!`, `?` followed by space/newline (excluding single-letter abbreviations)
5. Ensure forward progress: minimum advance of `MinChunkSize` per step

---

## Content Ingestion Pipeline

### Processing Steps (per bookmark)

1. **URL type detection** -- classify URL into content type
2. **Platform-specific extraction** -- fetch content via specialized extractor
3. **Fallback to generic scraping** -- if platform extractor fails
4. **Paywall detection** -- check for paywalled content (known domains, JSON-LD, HTML patterns)
5. **Archive fallback** -- if paywalled, try archive.today via web archive service or direct fetch
6. **Content cleaning** -- AI-powered removal of archive UI artifacts (only applied when content was retrieved via archive fallback; skipped for directly-scraped content)
7. **Title resolution** -- For URL bookmarks: readability title > OG title > first 100 chars of content. For notes-only bookmarks: always `"Quick Note"`. The UI may override the display title (e.g., showing first 50 chars of notes or "Untitled" when the stored title is empty).
8. **Embedding generation** -- via configured embedding provider
9. **Summary generation** -- AI summary using content-type-specific prompts (non-blocking, see [Summary Generation](#summary-generation)). YouTube bookmarks get a video-focused prompt, tweets get a thread-focused prompt, and articles get a general article prompt.
10. **Database insert** -- upsert bookmark with embedding vector
11. **Graph entity extraction** -- async, 2-minute timeout, fire-and-forget
12. **Content chunking** -- async, for content >24,000 chars (matching `ChunkingThreshold`), fire-and-forget

### Retry Policy

- Maximum 3 retry attempts with exponential backoff (1s, 2s, 4s)
- Non-retryable errors: "invalid URL", "paywall detected", "content too large", "embedding limit exceeded"
- Retryable errors: HTTP 500-504, timeouts, connection resets

### Paywall Detection

Four detection methods, ordered by reliability:

| Method | Confidence | Technique |
|--------|-----------|-----------|
| JSON-LD structured data | 1.0 | `isAccessibleForFree: false` in Article schema |
| Known domain list | 0.9 | 80+ hardcoded publication domains |
| HTML pattern matching | 0.7-0.95 | CSS classes, data attributes, paywall scripts |
| Content analysis | 0.6 | Truncation indicators in short articles (<500 chars) |

---

## Summary Generation

The system generates AI-powered summaries for bookmarks as part of the ingestion pipeline. Summary generation is **non-blocking** — failure never prevents bookmark creation. The summary is stored in a dedicated field on the bookmark, separate from the `content` field.

There are two distinct summarization paths that serve different purposes:

### Path 1: Content-Type-Aware Summary (User-Facing)

Generated during bookmark ingestion (pipeline step 9) and during re-extraction. The summary is stored in the bookmark's `summary` field and displayed in the UI.

The AI prompt strategy and output budget vary by content type:

| Content Type | Prompt Strategy | Max Output Tokens | Temperature |
|---|---|---|---|
| `youtube` | Timestamped structured outline (see format below) | 4000-8000 | 0.5 |
| `twitter` / `tiktok` | Main message, key facts, context, and tone in 2-3 paragraphs | 1000 | 0.5 |
| `article` / `webpage` | 5-section numbered structure (see format below) | 2000-8000 | 0.5 |
| default | Generic: main topic, key points, technical info, conclusions | 2000-8000 | 0.5 |

#### YouTube Summary Format

YouTube summaries must produce a **navigable index** of the video transcript with the following structure:

1. **Title**: A descriptive title capturing the main theme of the video
2. **Timestamped sections** organized by major topic transitions:
   - Timestamps in `MM:SS` format (aim for 5-10 minute segments)
   - Clear section headers at each topic transition
   - Under each timestamp: the main topic/question discussed, key assertions, specific examples or references mentioned
   - Speaker attributions for important statements (e.g., "Ross asserts...")
   - Bullet points for sub-topics within sections
   - Brief "Related reading" notes where relevant topics are mentioned

**Expected output structure:**
```
[Descriptive Title]

00:00 [Section Title]
[Brief context or related reading]

[Key question discussed or main assertion]

• Point 1
• Point 2
• Specific example or reference mentioned

08:30 [Next Section Title]
[Main topic/question for this section]

[Speaker] asserts...
• Point 1
• Point 2

...
```

**YouTube format invariants:**
- MUST contain at least one timestamp in `MM:SS` format
- MUST start directly with the title (no preamble)
- Each timestamp section MUST have a section header in brackets
- Timestamps MUST appear in chronological order

#### Article / Webpage Summary Format

Article summaries must produce a **5-section numbered structure**:

```
1. **Main Topic**: What is this article about?
2. **Key Points**: 3-5 main arguments or findings
3. **Important Details**: Specific facts, figures, or examples that support the main points
4. **Conclusions**: What conclusions does the author draw?
5. **Relevance**: Why is this information important or useful?
```

**Article format invariants:**
- MUST contain exactly 5 numbered sections in the specified order
- MUST start directly with `1. **Main Topic**:` (no preamble)
- Section 2 (Key Points) MUST contain 3-5 bullet points
- Technical details, names, dates, and specific information MUST be preserved

#### Social Media Summary Format (Twitter / TikTok)

Social media summaries must capture:
- The main message or point being made
- Key facts, claims, or insights
- Any important context or references
- The overall tone and purpose

**Social media format invariants:**
- MUST be 2-3 paragraphs maximum
- MUST start directly with summary content (no preamble)
- MUST preserve all important information from the original post

#### Default Summary Format

For content that doesn't match a specific type, summaries must:
- Identify the main topic or purpose
- List key points and important details
- Preserve technical information, names, and specific facts
- Highlight any conclusions or recommendations

**Default format invariants:**
- MUST start directly with summary content (no preamble)
- MUST be concise while covering all key information

**Behavioral rules (all content types):**
- Content type is determined by the same URL detection used in the ingestion pipeline (see [Content Type Detection](#content-type-detection))
- If summary generation fails, the error is logged and the bookmark is created with a null/empty summary
- On re-extraction, if summary generation fails, the existing summary is preserved (null-coalescing update)
- Summary output MUST begin directly with content — never with introductory phrases like "Here is a summary...", "Of course, here is...", "Sure! Here's...", or similar preamble. This is enforced via prompt instructions.

### Path 2: Summarize-Then-Embed (Transient)

When content exceeds the embedding provider's maximum input length, it is summarized first, then the *summary* is embedded. This is a transient transformation — the original full content is stored in the `content` field, not the summary.

| Step | Behavior |
|---|---|
| Content within provider limit | Embed directly, no summarization |
| Content exceeds provider limit | Summarize content first, then embed the summary |
| Content far exceeds limit (e.g., >100K chars) | Truncate to head+tail (first half + last half of allowed range) before summarizing |
| Summarization fails | Fall back to intelligent truncation at the last sentence boundary within the allowed range |

**Key invariant:** The summarize-then-embed output is **never stored**. It exists only during the embedding generation call. The bookmark's `content` field always contains the original extracted content.

### Storage

- The user-facing summary is stored in a dedicated `summary` column (not inside the `metadata` JSON)
- The summary has a full-text search index for inclusion in search results
- When updating a bookmark, a null summary preserves the existing value (null-coalescing)

---

## Knowledge Graph Schema

### Node Types

| Label | Key Properties | Description |
|-------|---------------|-------------|
| `:Bookmark` | `id`, `url`, `title`, `created_at` | Core bookmark reference |
| `:Category` | `name`, `description`, `level` | Hierarchical classification |
| `:Concept` | `name`, `description`, `domain`, `aliases` | Abstract topics |
| `:Entity:Person` | `name`, `type="Person"` | Named individuals |
| `:Entity:Organization` | `name`, `type="Organization"` | Companies, institutions |
| `:Entity:Technology` | `name`, `type="Technology"` | Tools, frameworks, languages |
| `:Entity:Project` | `name`, `type="Project"` | Specific projects |
| `:Topic` | `name` | Topical tags |

### Relationship Types

| Relationship | Source -> Target | Description |
|-------------|-----------------|-------------|
| `BELONGS_TO` | Bookmark -> Category | Classification |
| `ABOUT` | Bookmark -> Concept | Topic association |
| `MENTIONS` | Bookmark -> Entity | Entity reference |
| `SUB_CATEGORY_OF` | Category -> Category | Hierarchy |
| `RELATES_TO` | Any -> Any | General relationship |
| `SIMILAR_TO` | Any -> Any | Similarity |

### Entity Extraction

- Content is sanitized based on detected type (social media vs. general)
- Content >50,000 chars is split at sentence boundary for parallel extraction
- LLM extraction uses JSON format (primary) with text format fallback
- Results are deduplicated by name across chunks
- 3 retry attempts with exponential backoff and jitter

---

## Embedding Providers

### Provider Abstraction

The system supports multiple embedding providers via a provider abstraction. Each provider must implement:
- `GenerateEmbedding(text, taskType?) → vector` — generate an embedding vector for content
- `GenerateChatCompletion(prompt, options?) → text` — generate text via a chat/completion model

Provider configuration specifies:
- **Dimensions**: The fixed vector length for the provider (must be consistent across all bookmarks)
- **Max input length**: Character limit before content must be summarized or truncated
- **Task types**: Whether the provider supports task-type hints (e.g., document vs. query embeddings)

### Long Content Handling

- Content exceeding the provider's max input is summarized first, then the summary is embedded
- If summarization input is very large, it is truncated using a head+tail strategy (first half + last half of the allowed range)
- If summarization fails, intelligent truncation finds the last sentence boundary within the allowed range

### Chat Model Routing

Dynamic model selection based on request characteristics:
- Large or complex tasks (JSON extraction, long inputs): route to a more capable model
- Small tasks: route to a faster/cheaper model
- On server error with the primary model: fallback to the secondary model
- On structured output failure: retry without structured output constraints

---

## Server Configuration

| Setting | Value | Rationale |
|---------|-------|-----------|
| Read Timeout | 5 minutes | Accommodates long SSE streams |
| Write Timeout | 5 minutes | Accommodates slow content extraction |
| Idle Timeout | 5 minutes | Keeps SSE connections alive |
| Job Workers | 5 | Parallel bookmark processing |
| Job Queue Buffer | 100 | Handles burst bookmark saves |
| Job Max Retries | 3 | With exponential backoff |
| Job SSE Poll Interval | 500ms | Balance between responsiveness and load |

---

## Web UI

The web UI is a single-page application served as a static HTML file by the backend. No build step is required.

### Views

The application has three primary views, switched via a sticky header navigation bar:

| View | Purpose | Key Interactions |
|------|---------|-----------------|
| **Bookmarks** | Browse, search, and filter bookmarks | Search input, autocomplete, category/read/content-type filters, infinite scroll |
| **Ask** | Ask natural language questions | Question input, streaming answer display, citation cards, knowledge graph highlights |
| **Add** | Save new bookmarks or notes | URL input, notes textarea, submit button |
### Bookmarks View

**Search and filtering:**
- Free-text search input with 300ms debounce, triggers hybrid search on the backend
- Autocomplete dropdown fed by `GET /autocomplete`, showing graph node suggestions with type-colored pills (blue=Category, green=Concept, purple=Topic, orange=Person, red=Organization, indigo=Technology, pink=Project)
- Selected autocomplete nodes appear as removable pills and are sent as `nodes[]` query parameters for graph-enhanced filtering
- Category dropdown filter populated from `GET /categories?with_counts=true`
- Read status filter: All / Unread Only / Read Only
- Content type filter: All / Video Only / Text Only
- All filters reset pagination and reload from the server

**Bookmark list:**
- Bookmarks grouped by date with sticky date headers
- Date labels use smart formatting: "Today", "Yesterday", weekday name for last 7 days, full date otherwise
- Infinite scroll loads 20 items per page, triggered when user scrolls within 100px of bottom
- Scroll position is preserved during pagination loads via a locking mechanism

**Bookmark card structure:**

| Element | Behavior |
|---------|----------|
| Thumbnail (128x80) | From metadata preview images, falls back to document/note icon |
| Read/Unread toggle | Circle-check (green) for read, eye (gray) for unread; calls `PATCH /bookmark/{id}/read-status` |
| Title | Clickable, opens detail modal; falls back to first 50 chars of notes or "Untitled" |
| URL | Truncated to 100 chars, opens in new tab; hidden for notes |
| Summary/snippet | 2-line clamp, shows `summary` or `snippet` |
| Timestamps | "Read: X ago" (green, only for read bookmarks), "X ago" for creation date |

**Processing placeholder:** While a bookmark is being processed, a blue-bordered card with pulse animation shows "Processing bookmark..." with the URL/notes preview and gray skeleton bars.

**Empty state:** When no bookmarks exist, shows a message with an "Add Bookmark" button that switches to the Add view.

### Ask View

**Question submission:**
- Single text input with blue send button (paper plane icon)
- Submit disabled when empty or while streaming
- Questions saved to `localStorage` history (max 10 items)
- "Quick ask" input also available in the Bookmarks view header, which transfers the question to Ask view

**Answer streaming:**
- Opens SSE connection to `GET /answer?q=...`
- Text chunks appended in real-time, rendered as Markdown (GFM enabled)
- `[N]` citation references are post-processed into clickable superscript links that smooth-scroll to the corresponding citation card with a temporary blue ring highlight (2 seconds)
- Pulse skeleton shown while waiting for first chunk

**Knowledge graph highlighting:**
- After streaming completes, the answer text is scanned for known knowledge graph terms (categories, concepts, entities)
- Matching terms are wrapped in colored spans: blue (category), green (concept), orange (entity)
- Highlighted terms are clickable, opening a floating tooltip with:
  - Term name and type
  - Count of related bookmarks
  - Up to 3 related bookmark links (expandable to show all)
  - "Explore" button that submits the term as a new question
- Tooltip has keyboard support (Enter/Space to activate, Escape to close) and ARIA attributes (`role="dialog"`, `aria-live="polite"`)

**Citation display:**
- "Sources" section appears below the answer
- Each citation card shows: citation number (blue circle), thumbnail, title (clickable, opens detail modal), URL (external link), category/concept/entity pills (clickable, trigger related bookmark search), YouTube indicator (channel + duration), creation date
- Citations are lazy-loaded via IntersectionObserver with 100px rootMargin
- Only citations actually referenced as `[N]` in the answer text are shown

**Question history:**
- Previous questions shown below citations
- Each is clickable to re-submit
- "Clear All" button removes history from `localStorage`

### Add View

- URL input (optional, type="url") with auto-focus on view switch
- Notes textarea (optional, free-text)
- "Add Bookmark" button, disabled when both URL and notes are empty
- Double-submission prevention via flag with 100ms delay
- On success: switches to Bookmarks view, shows processing placeholder, monitors job via SSE
- On duplicate (409): switches to Bookmarks view, scrolls to existing bookmark, applies yellow border + scale highlight animation (1.5s, repeats twice)
- Notes-only bookmarks use `POST /note` endpoint, URL bookmarks use `POST /add`

### Bookmark Detail Modal

Overlay modal (max-w-4xl, max-h-90vh) opened by clicking any bookmark title. Contains:

1. **Header bar:** Read toggle, re-extract button, delete button
2. **Title and metadata:** Title, read status badge, URL (external link), creation date
3. **Preview image** (conditional)
4. **Knowledge graph metadata:** Clickable pills for categories (blue), concepts (green), topics (purple), entities (orange with type label). Clicking a pill loads related bookmarks inline
5. **Related bookmarks** (conditional, shown after clicking a graph pill): List of related bookmarks with clickable titles and tag pills
6. **Summary section**
7. **Content preview:** Truncated to 1500 chars with "Show Full Content" / "Show Less" toggle
8. **Personal notes:** View/edit/add mode with textarea and save/cancel buttons. Saving triggers `PUT /bookmark/{id}/notes` and async graph re-indexing

**Delete flow:** Delete button opens a confirmation modal with warning icon, bookmark title, and Cancel/Delete buttons. Delete calls `DELETE /bookmark/{id}` and removes the bookmark from the list.

**Re-extract flow:** Re-extract button calls `PUT /bookmark/{id}/reextract`, which creates a new job. Returns 202 with job ID.

### Notification System

Fixed-position toast in top-right corner with three variants:

| Variant | Color | Use Case |
|---------|-------|----------|
| Info | Blue | General information |
| Success | Green | Successful operations |
| Error | Red | Failed operations |

Auto-dismisses after 5 seconds. Close button available. Slide-in/fade transitions.

### Responsive Behavior

- PWA meta tags for iOS home screen support
- Max width container (7xl) centered on large screens
- Bookmark cards: vertical stack on mobile, horizontal row on desktop
- Thumbnails: full width on mobile, 128x80 on desktop
- Filter grid: 1 column on mobile, 2 on tablet, 3 on desktop
- Responsive text sizes and padding throughout

### URL-Based Navigation

- Supports browser back/forward via `pushState`/`replaceState` with `popstate` listener
- Query parameters: `bookmark` (opens detail modal), `related_type`, `related_name` (pre-loads related bookmarks)
- Enables deep-linking to specific bookmark details

### UI Invariants

- **UI-INV-001**: The Bookmarks view always shows bookmarks ordered by creation date descending (newest first) unless a search query is active, in which case results are ordered by relevance score descending. *Rationale: users expect to see their most recent bookmarks first.*

- **UI-INV-002**: A processing placeholder is always visible in the bookmark list while a job is in `pending` or `processing` state. The placeholder is replaced with the real bookmark card upon job completion. *Rationale: provides immediate visual feedback that the save action was received.*

- **UI-INV-003**: Citation numbers in the answer text always correspond to citation cards in the Sources section. Only actually-cited sources are displayed. *Rationale: prevents confusion from showing irrelevant sources.*

- **UI-INV-004**: The notification toast auto-dismisses after 5 seconds. Multiple notifications can be shown simultaneously. *Rationale: transient feedback should not require user action to dismiss.*

- **UI-INV-005**: Knowledge graph highlighted terms in answers are never applied inside code blocks, `<pre>`, `<script>`, `<style>`, or superscript (`<sup>`) elements. *Rationale: prevents corrupting code examples and citation references.*

- **UI-INV-006**: Autocomplete dropdown closes on: blur (200ms delay), Escape key, or selecting a result. It never remains open when the search input loses focus. *Rationale: prevents orphaned dropdowns from blocking interaction.*

---

## Chrome Extension

The Chrome extension provides a minimal browser-integrated bookmark saving experience. It operates in "fast mode" by default -- a single-click save that fires and forgets.

### Popup (Fast Mode -- Default)

The default popup (`popup-fast.html`) shows:
- Static "Smart Bookmark Vault" header
- Current page title
- "Save Bookmark" button

**Save flow:**
1. User clicks "Save Bookmark"
2. Button text changes to "Saving...", button disabled
3. Message `{ action: 'saveBookmark', url }` sent to background service worker
4. Popup closes immediately (fire-and-forget)

**Validation:** If the current tab URL is not `http://` or `https://`, the save button is disabled with an error message.

### Popup (Analysis Mode -- Not yet specced)

Analysis Mode is a planned richer popup with AI-powered bookmark analysis (similar bookmarks, tags, AI summary). It is not part of the current specification. Only Fast Mode is specced.

### Background Service Worker

Handles three bookmark save entry points:
1. **Popup message**: Responds to `saveBookmark` action from either popup variant
2. **Context menu**: "Save to Smart Bookmark Vault" menu item on pages and links
3. **Keyboard shortcut**: `Cmd+Shift+B` (Mac) / `Ctrl+Shift+B` (Windows/Linux)

All three paths call `POST /add` with `{ url }` and show a Chrome notification on success/failure. On success, the extension badge briefly shows "..." (blue) for 3 seconds.

### Content Script

Injected into every page at `document_idle`:
- **Metadata extraction**: Extracts title, description, OG image, author, published date, keywords, and JSON-LD structured data. Currently not called by any extension code (reserved for future use).
- **Visual feedback**: Injects a green toast notification into the page DOM when requested. Currently not called by any extension code.
- **Keyboard shortcut**: Listens for `Cmd/Ctrl+Shift+B` as a redundant fallback to the Chrome commands API.

### Options Page

Full-page settings interface with three sections:

**Server Configuration:**
- API URL input (default: `http://localhost:8082`)
- API Key input (optional, reserved for future auth)
- "Test Connection" button that calls `GET /health`

**Display Preferences:**
- Theme selector: Auto (follow system) / Light / Dark
- Show notifications checkbox
- Show badge checkbox

**Keyboard Shortcuts:**
- Displays current shortcut binding (default: `Cmd+Shift+B`)
- Link to Chrome's extension shortcuts page

Settings stored in browser extension sync storage (syncs across browser instances).

### Extension Invariants

- **EXT-INV-001**: The fast mode popup always closes immediately after sending the save message, regardless of success or failure. *Rationale: fire-and-forget UX -- the user should never wait.*

- **EXT-INV-002**: Non-HTTP/HTTPS URLs (chrome://, file://, etc.) cannot be saved. The save button is disabled with an error message. *Rationale: only web content can be scraped and embedded.*

- **EXT-INV-003**: The keyboard shortcut `Cmd/Ctrl+Shift+B` triggers a save from any page, via either the Chrome commands API or the content script fallback. *Rationale: reliability -- at least one path will work.*

---

## Evaluation Tiers

### Tier 1: Durable Evaluations (survive reimplementation)

- **Invariant checks**: Verify INV-001 through INV-012 hold after operations
- **Property-based tests**: Verify PROP-001 through PROP-012 with generated inputs
- **Contract conformance**: Verify all interface schemas (request/response shapes, status codes)
- **End-to-end behavioral checks**: URL -> bookmark -> searchable -> deletable lifecycle
- **Boundary tests**: Chunking thresholds, embedding dimension limits, pagination bounds

### Tier 2: Ephemeral Tests (disposable with implementation)

- **Example-based tests**: Specific URL type detection cases
- **Progression tests**: Query classification with representative queries
- **Platform-specific edge cases**: Twitter thread detection, YouTube video ID extraction

### Tier 3: Live Evaluations (continuous in production)

- **Operational metrics**: API latency, job queue depth, embedding generation time
- **Business invariants**: Bookmark count growth, search result quality scores
- **Drift detection**: Search weight effectiveness, classification accuracy
- **Cost metrics**: AI API token usage per bookmark, embedding cost per query

---

## Testing

### Test data format

Tests are defined in `tests.yaml` as language-agnostic evaluations organized by durability tier.

### Input field mapping

**AddBookmark:**
```yaml
input: { url: "<string>", notes: "<string>", created_at: "<string>" }
```

**HybridSearch:**
```yaml
input: { query_embedding: [float], query_text: "<string>", limit: integer }
```

**ChunkContent:**
```yaml
input: { content: "<string>", max_chunk_size: integer, overlap_size: integer }
```

### Error test handling

For entries with `error: true`, assert the function raises/returns an error or the HTTP endpoint returns a 4xx/5xx status.

---

## Live Evaluation Criteria

### Operational Metrics

| Metric | Acceptable Range | Alert Threshold |
|--------|-----------------|-----------------|
| POST /add p99 latency | < 500ms (just job creation) | > 2s |
| GET /answer time-to-first-byte | < 3s | > 10s |
| GET /bookmarks p99 latency | < 1s | > 5s |
| Job processing time (median) | < 30s | > 120s |
| Job queue depth | < 50 | > 80 |
| Job failure rate | < 5% | > 15% |

### Business Invariants (monitored continuously)

- **INV-001**: No bookmarks exist with null or empty `content`
- **INV-002**: No duplicate URLs exist among bookmarks (excluding null URLs)
- **INV-005**: For every chunked bookmark, `chunk_count` matches the actual number of chunk records
- **INV-007**: No jobs exist with a status outside the allowed set (`pending`, `processing`, `completed`, `failed`)

### Cost Metrics

| Metric | Baseline | Alert Threshold |
|--------|----------|-----------------|
| Embedding tokens per bookmark | ~2-5K tokens (varies by provider) | > 10K tokens |
| Chat tokens per entity extraction | ~5K tokens | > 20K tokens |
| Chat tokens per summary | ~1K tokens | > 5K tokens |
| Archive API calls per bookmark | 0-1 (only for YouTube/paywalled) | > 3 |

---

## Implementation Checklist

Before considering the implementation complete:

- [ ] All API endpoints implemented with correct request/response schemas
- [ ] All tests.yaml durable evaluations pass
- [ ] All property-based tests pass with generative framework
- [ ] All invariant checks pass
- [ ] Hybrid search returns results ordered by score
- [ ] SSE streaming works with proper event types
- [ ] Job queue processes bookmarks asynchronously
- [ ] Platform-specific extractors handle their URL patterns
- [ ] Paywall detection and archive fallback work
- [ ] Content chunking produces valid, non-overlapping primary regions
- [ ] Graph entity extraction creates correct node types and relationships
- [ ] Cascade deletion removes bookmarks, chunks, and graph nodes
- [ ] Duplicate URL detection prevents duplicate bookmarks
- [ ] Errors are returned as JSON with appropriate HTTP status codes
- [ ] CORS headers are set on all responses
- [ ] Health check endpoint returns 200

---

## Regeneration Confidence Checklist

- [x] All system invariants are explicit (INV-001 through INV-012)
- [x] All behavioral properties are formally stated (PROP-001 through PROP-016)
- [x] All interface contracts have precise schemas (14 endpoints documented)
- [x] All functions have unambiguous behavior tables
- [x] All boundary conditions have exact threshold values
- [x] All error conditions are documented
- [x] Property-based tests cover function composition behaviors
- [x] Live evaluation criteria would catch drift after regeneration
- [x] No critical behavior exists only as implicit knowledge

---

## Original Implementation Reference

This section documents the technology choices made in the initial implementation. These are not part of the specification — any reimplementation may use different tools, libraries, and services as long as it satisfies the contracts, invariants, and properties defined above.

### Backend

| Component | Technology | Notes |
|-----------|-----------|-------|
| Language | Go | Standard library HTTP server |
| Relational database | PostgreSQL | With pgvector extension for vector indexes |
| Full-text search | PostgreSQL tsvector | `GENERATED ALWAYS` columns, `ts_rank_cd` for ranking |
| Knowledge graph | Neo4j | Cypher queries, MERGE for idempotent node creation |
| Embedding provider (primary) | Google Gemini `gemini-embedding-001` | 3072 dimensions, task-type support |
| Embedding provider (secondary) | OpenAI `text-embedding-3-small` | 1536 dimensions |
| Chat model (large) | Gemini `gemini-2.5-pro` | Used for complex extraction and long content |
| Chat model (small) | Gemini `gemini-2.5-flash` | Used for summaries and small tasks |
| Archive/paywall fallback | Tavily API | Retrieves archived versions of paywalled content |
| Content extraction | go-readability | Based on Mozilla Readability algorithm |
| Observability | Langfuse | Tracing for AI operations |

### Web UI

| Component | Technology | Notes |
|-----------|-----------|-------|
| Framework | Alpine.js | Lightweight reactivity, no build step |
| Styling | Tailwind CSS | Via CDN, utility-first CSS |
| Markdown rendering | Marked.js | GFM enabled |
| Date formatting | Day.js | With relativeTime plugin |
| Icons | Lucide | SVG icon library |
| Delivery | All via CDN | No build step required |

### Chrome Extension

| Component | Technology | Notes |
|-----------|-----------|-------|
| Storage | `chrome.storage.sync` | Syncs settings across browser instances |
| Commands | `chrome.commands` API | Keyboard shortcut registration |
| Notifications | `chrome.notifications` API | Save confirmation feedback |

### Infrastructure

| Component | Technology | Notes |
|-----------|-----------|-------|
| Container orchestration | Docker Compose | PostgreSQL + Neo4j services |
| Development environment | Devbox | Reproducible dev shells |
| Process management | mise / devbox scripts | Server start/stop/restart |

---

## Version History

- **v0.5.0** - Added precise summary output format templates (YouTube timestamped index, article 5-section structure, social media 2-3 paragraph), added Answer Generation section with intent-specific system prompts, context template format, result filtering rules, and search strategy parameters. Added PROP-013 through PROP-016 for summary format and prompt behavior. Added SummaryGeneration and AnswerGeneration test sections.
- **v0.4.0** - Clarified single-user auth model, documented search path switching (language detection), standardized Job Status to snake_case, added metadata schema per content type, documented snippet generation, clarified content cleaning scope (archive-only), documented re-extract full regeneration scope, documented content-type-specific summary prompts, clarified total sentinel -2 threshold, documented nodes[] graph filtering mechanism, documented related bookmarks score field, marked Analysis Mode as not yet specced
- **v0.3.2** - Added Summary Generation section with content-type-aware strategies and summarize-then-embed path
- **v0.3.1** - Added missing contracts (related bookmarks, categories), defined RecentBookmark and autocomplete schemas, fixed chunking threshold inconsistency, added notes title resolution
- **v0.3.0** - Abstracted technology references; added Original Implementation Reference section
- **v0.2.0** - Added Web UI and Chrome Extension specification with UI invariants
- **v0.1.0** - Initial specification extracted from existing codebase
