# Brainy - Smart Bookmark Vault Specification v0.1.0

## Overview

Brainy is a personal bookmark knowledge base that ingests URLs from multiple sources, extracts and enriches content using AI, stores it in PostgreSQL with vector embeddings for hybrid semantic+lexical search, and maintains a Neo4j knowledge graph for entity-based discovery. Users ask natural language questions and receive AI-generated answers with citations from their bookmark collection.

The system processes bookmarks asynchronously through a job queue, supports multiple content platforms (YouTube, Twitter/X, Instagram, TikTok, generic web), detects paywalls with archive fallback, and provides multilingual search (English/Spanish) with configurable embedding providers (OpenAI, Gemini).

## Design Principles

1. **Async-first ingestion.** Bookmark saving returns immediately with a job ID. All heavy processing (scraping, embedding, graph extraction, chunking) happens in background workers. The user never waits for content processing.

2. **Graceful degradation.** Every optional subsystem (Neo4j graph, Tavily archive fallback, Gemini cleaner, thumbnail extraction, summary generation) can fail without blocking bookmark creation. The core path (scrape -> embed -> store) always completes if the URL is reachable.

3. **Platform-aware extraction.** Each content platform (YouTube, Twitter, Instagram, TikTok) has a specialized extractor that understands the platform's content structure. All platform extractors fall back to generic webpage scraping on failure.

4. **Hybrid search with tunable weights.** Search combines semantic (vector cosine similarity) and lexical (PostgreSQL full-text) signals via Reciprocal Rank Fusion (RRF). Weights are dynamically adjusted based on query intent classification.

5. **Knowledge graph as enrichment layer.** Neo4j stores extracted entities, categories, and concepts as a discovery and filtering mechanism. PostgreSQL remains the source of truth for bookmark data. Graph operations never block core CRUD.

6. **Fire-and-forget post-processing.** Graph entity extraction and content chunking run in detached goroutines after the bookmark is persisted. Their success or failure does not affect the bookmark's existence.

---

## System Invariants

Properties that must hold across ALL implementations, regardless of language, architecture, or internal design.

- **INV-001**: Every bookmark has non-empty `content`. *Rationale: content is the foundation for embedding generation and search. A bookmark without content cannot participate in hybrid search.*

- **INV-002**: Non-null URLs are unique across all bookmarks. Multiple bookmarks with NULL URLs (standalone notes) are permitted. *Rationale: prevents duplicate bookmarks for the same resource while allowing unlimited freeform notes.*

- **INV-003**: `read_status` is always set (never null), defaulting to `false`. *Rationale: UI filtering by read status requires a definite boolean value.*

- **INV-004**: The `tsv` (English) and `tsv_es` (Spanish) full-text search vectors are always in sync with `title` + `content`. The `note_tsv` vector is always in sync with `notes`. *Rationale: these are PostgreSQL GENERATED ALWAYS columns that auto-recompute on any change, ensuring search results are never stale.*

- **INV-005**: When `is_chunked = true` on a bookmark, `chunk_count > 0` and exactly `chunk_count` rows exist in `bookmark_chunks` for that bookmark ID. When `is_chunked = false`, `chunk_count = 0` and no chunks exist. *Rationale: chunk metadata must match actual chunk data for unified search to work correctly.*

- **INV-006**: Deleting a bookmark cascades to delete all its chunks (`bookmark_chunks`) and associated jobs (`bookmark_jobs`). *Rationale: no orphaned data should remain after bookmark deletion.*

- **INV-007**: Job status follows a strict state machine: `pending` -> `processing` -> `completed` | `failed`. No other transitions are valid. A job cannot return to `pending` after leaving it. *Rationale: clients polling job status rely on monotonic state progression.*

- **INV-008**: The `updated_at` timestamp on bookmarks auto-updates on every row modification via database trigger. *Rationale: consumers of bookmark data can rely on `updated_at` for change detection and cache invalidation.*

- **INV-009**: Embedding vectors have exactly the dimensions configured for the active provider: 3072 for Gemini, 1536 for OpenAI. All vectors in the database use the same dimensionality. *Rationale: pgvector requires consistent dimensions for index operations and cosine similarity calculations.*

- **INV-010**: In the Neo4j knowledge graph, all entity nodes have dual labels: their specific type label (e.g., `:Person`) plus the generic `:Entity` label. Category and Concept nodes have only their respective single label. *Rationale: enables both type-specific and generic entity queries.*

- **INV-011**: Neo4j node creation uses MERGE (not CREATE) keyed on `name` for categories/concepts/entities and on `id` for bookmarks. This prevents duplicate graph nodes. *Rationale: idempotent graph operations are essential for retry safety and re-indexing.*

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

---

## Interface Contracts

Contracts specify what crosses boundaries between components. Each contract survives reimplementation of either side.

### Client -> Backend: Add Bookmark

**Protocol**: HTTP POST

**Endpoint**: `/add`

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

**Contract invariants**:
- `bookmarks.length <= limit`
- When `total >= 0`: `offset + bookmarks.length <= total`
- Results are ordered by `created_at DESC` (for non-search queries) or by relevance score DESC (for search queries)
- `total` may be `-1` (count error) or `-2` (sentinel for "many results")

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
  "bookmarkId": string (present when status == "completed"),
  "retryCount": integer,
  "error": string (present when status == "failed"),
  "createdAt": string,
  "startedAt": string (nullable),
  "completedAt": string (nullable),
  "metadata": object
}
```

**Contract invariants**:
- `status` only moves forward: pending -> processing -> completed|failed
- `bookmarkId` is always present when `status == "completed"`
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
  "metadata": object,
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

**Response schema (200)**: Array of suggestion objects from graph service.

**Contract invariants**:
- Returns empty array `[]` on any error or when graph service is unavailable
- `limit` is capped at 20

---

## Output Structure

**Do generate:**
- Go backend binary (`backend/cmd/server/main.go`)
- Database migration SQL files (`backend/internal/db/migrations/`)
- Static web UI (`backend/web/static/index.html`)

**Do not generate:**
- Raycast extension (separate repository)
- Chrome extension distribution packages
- iOS Shortcut files (manual creation)
- Neo4j schema (managed via application-level MERGE operations)

---

## Type Conventions

Since this spec targets a Go implementation with PostgreSQL:

| Spec type | Go type | PostgreSQL type |
|-----------|---------|-----------------|
| `string` | `string` | `TEXT` |
| `integer` | `int` | `INTEGER` |
| `float` | `float64` | `NUMERIC` / `FLOAT` |
| `boolean` | `bool` | `BOOLEAN` |
| `uuid` | `string` (UUID format) | `UUID` |
| `timestamp` | `time.Time` | `TIMESTAMPTZ` |
| `vector` | `[]float32` | `vector(N)` via pgvector |
| `tsvector` | N/A (DB-generated) | `TSVECTOR` (GENERATED) |
| `json_object` | `map[string]interface{}` | `JSONB` |
| `enum` | `string` (constrained) | Custom ENUM type |

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

The system uses two distinct scoring methods depending on the search path:

**Path A: Direct Similarity Blending** (default `hybrid_search_bookmarks`)
```
combined_score = MAX((cosine_similarity * 0.6) + (normalized_ts_rank * 0.4), 0.01)
```
Where `cosine_similarity = 1 - cosine_distance` and `normalized_ts_rank = ts_rank_cd / 10.0`.

**Path B: Reciprocal Rank Fusion** (multilingual `hybrid_search_multilingual`)
```
rrf_score(rank, k) = 1.0 / (k + rank)   -- where k defaults to 50
final_score = (rrf_score(semantic_rank, k) * semantic_weight) + (rrf_score(keyword_rank, k) * lexical_weight)
```

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
5. **Archive fallback** -- if paywalled, try archive.today via Tavily or direct fetch
6. **Content cleaning** -- AI-powered removal of archive UI artifacts (Gemini preferred, OpenAI fallback)
7. **Title resolution** -- readability title > OG title > first 100 chars of content
8. **Embedding generation** -- via configured provider (OpenAI or Gemini)
9. **Summary generation** -- content-type-aware AI summary (non-blocking)
10. **Database insert** -- upsert bookmark with embedding vector
11. **Graph entity extraction** -- async, 2-minute timeout, fire-and-forget
12. **Content chunking** -- async, for content >5000 chars, fire-and-forget

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

### Duplicate Detection and Merging

| Threshold | Default | Purpose |
|-----------|---------|---------|
| String similarity | 0.85 | Minimum Levenshtein/Jaro-Winkler average |
| Vector similarity | 0.90 | Minimum embedding cosine similarity |
| Auto-merge | 0.95 | Above this, merge without review |

Combined score: `string_sim * 0.4 + vector_sim * 0.6` (when embeddings available), otherwise string similarity alone.

---

## Embedding Providers

### Provider Configuration

| Provider | Model | Dimensions | Max Input | Task Types |
|----------|-------|-----------|-----------|------------|
| OpenAI | `text-embedding-3-small` | 1536 | 24,000 chars (~6K tokens) | No |
| Gemini | `text-embedding-004` | 3072 (configurable: 768, 1536) | 80,000 chars (~20K tokens) | Yes (RETRIEVAL_DOCUMENT, RETRIEVAL_QUERY, etc.) |

### Long Content Handling

- OpenAI: Content >24K chars is summarized first, then embedded. Summarization input >100K chars is truncated to first 50K + last 50K.
- Gemini: Content >80K chars is summarized first, then embedded. Summarization input >200K chars is truncated to first 100K + last 100K.
- If summarization fails, intelligent truncation finds the last sentence boundary within the allowed range.

### Chat Model Selection (Gemini)

Dynamic model routing based on request characteristics:
- JSON extraction with >10K max tokens or >20K input chars: `gemini-2.5-pro`
- Regular tasks with >50K chars or >10K tokens: `gemini-2.5-pro`
- Small tasks (<10K chars, <=4K tokens): `gemini-2.5-flash`
- On 500 error with Pro: fallback to Flash
- On JSON mode failure: retry without ResponseMIMEType

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

- **INV-001**: `SELECT COUNT(*) FROM bookmarks WHERE content IS NULL OR content = ''` should always be 0
- **INV-002**: `SELECT url, COUNT(*) FROM bookmarks WHERE url IS NOT NULL GROUP BY url HAVING COUNT(*) > 1` should return 0 rows
- **INV-005**: `SELECT b.id FROM bookmarks b WHERE b.is_chunked = true AND b.chunk_count != (SELECT COUNT(*) FROM bookmark_chunks c WHERE c.bookmark_id = b.id)` should return 0 rows
- **INV-007**: `SELECT * FROM bookmark_jobs WHERE status NOT IN ('pending', 'processing', 'completed', 'failed')` should return 0 rows

### Cost Metrics

| Metric | Baseline | Alert Threshold |
|--------|----------|-----------------|
| Embedding tokens per bookmark | ~2K tokens (OpenAI) / ~5K tokens (Gemini) | > 10K tokens |
| Chat tokens per entity extraction | ~5K tokens | > 20K tokens |
| Chat tokens per summary | ~1K tokens | > 5K tokens |
| Tavily API calls per bookmark | 0-1 (only for YouTube/paywalled) | > 3 |

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
- [x] All behavioral properties are formally stated (PROP-001 through PROP-012)
- [x] All interface contracts have precise schemas (12 endpoints documented)
- [x] All functions have unambiguous behavior tables
- [x] All boundary conditions have exact threshold values
- [x] All error conditions are documented
- [x] Property-based tests cover function composition behaviors
- [x] Live evaluation criteria would catch drift after regeneration
- [x] No critical behavior exists only as implicit knowledge

---

## Version History

- **v0.1.0** - Initial specification extracted from existing codebase
