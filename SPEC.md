# Brainy - Smart Bookmark Vault Specification

Status: Draft v0.7.0 (language-agnostic)

Purpose: Brainy is a single-user bookmark knowledge base that ingests URLs, extracts and enriches content using AI, stores it with vector embeddings for hybrid search, maintains a knowledge graph for entity-based discovery, and answers natural language questions with citations from the user's collection.

## 1. Problem Statement

Bookmarking tools store URLs but lose context. When users revisit bookmarks, they must re-read content to recall why it was saved. Search is limited to titles and URLs, missing the semantic meaning of content. Cross-language content (English/Spanish) is especially hard to find.

Brainy solves this by:
- Extracting and storing full content from bookmarked pages
- Generating AI summaries for quick recall
- Building a knowledge graph of entities, concepts, and categories across bookmarks
- Enabling natural language questions answered with citations from the user's collection
- Supporting multilingual hybrid search (semantic + lexical) across English and Spanish content

Important boundary:

- **What Brainy IS**: A personal bookmark knowledge base with AI-powered search, summarization, and knowledge graph discovery.
- **What Brainy is NOT**: A multi-user collaboration tool, a general-purpose search engine, or a web crawler. It does not manage user authentication (single-user, private network).
- **Responsibility limit**: The system ingests URLs provided by the user. It does not discover or recommend new URLs. Graph merge/deduplication tooling exists but is a separate admin concern, not part of the core spec.

## 2. Goals and Non-Goals

### 2.1 Goals

- Ingest bookmarks from multiple sources (web UI, Chrome extension, iOS Shortcuts, API) with immediate acknowledgment
- Extract full content from web pages, YouTube videos, tweets, Instagram posts, and TikTok videos
- Generate AI summaries tailored to each content type
- Provide hybrid semantic+lexical search with query intent classification
- Answer natural language questions using bookmark content as context, with citations
- Maintain a knowledge graph of extracted entities, concepts, and categories
- Support multilingual search across English and Spanish content
- Detect paywalled content and fall back to archive sources

### 2.2 Non-Goals

- Multi-user support or authentication
- Real-time collaborative bookmarking
- Full-text indexing of non-bookmark content (e.g., local files)
- Graph merge/deduplication tooling (admin-only, not part of core spec)
- Mobile native apps (iOS Shortcuts integration is sufficient)
- Automatic bookmark discovery or recommendation

## 3. System Overview

### 3.1 Main Components

1. **HTTP API Server**
   - Serves all REST endpoints and SSE streams
   - Hosts the static web UI
   - Manages the job queue for async bookmark processing

2. **Job Queue & Workers**
   - 5 parallel workers process bookmarks asynchronously
   - Handles scraping, embedding, summarization, graph extraction, and chunking
   - Retry with exponential backoff for transient failures

3. **Relational Database (PostgreSQL + pgvector)**
   - Source of truth for all bookmark data
   - Vector indexes for semantic search
   - Full-text search indexes for English and Spanish
   - Content chunking for large documents

4. **Knowledge Graph (Neo4j)**
   - Stores extracted entities, categories, concepts, and topics
   - Provides entity-based discovery and filtering
   - Optional — system degrades gracefully without it

5. **AI Services (Embedding + Chat)**
   - Embedding generation for semantic search (OpenAI or Gemini)
   - Chat completions for answer generation, summarization, and entity extraction
   - Observability via Langfuse tracing

6. **Web UI**
   - Single-page application (Alpine.js + Tailwind CSS)
   - No build step — served as static HTML
   - Three views: Bookmarks, Ask, Add

7. **Chrome Extension**
   - Fast-mode single-click bookmark saving
   - Context menu and keyboard shortcut support

### 3.2 External Dependencies

- PostgreSQL with pgvector extension (vector search)
- Neo4j (knowledge graph, optional)
- OpenAI API or Google Gemini API (embeddings + chat)
- Tavily API (advanced scraping + paywall bypass, optional)
- Langfuse (AI observability, optional)

## 4. Design Principles

1. **Async-first ingestion.** Bookmark saving returns immediately with a job ID. All heavy processing (scraping, embedding, graph extraction, chunking) happens in background workers. The user never waits for content processing.

2. **Graceful degradation.** Every optional subsystem (knowledge graph, archive fallback, AI content cleaner, thumbnail extraction, summary generation) can fail without blocking bookmark creation. The core path (scrape -> embed -> store) always completes if the URL is reachable.

3. **Platform-aware extraction.** Each content platform (YouTube, Twitter, Instagram, TikTok) has a specialized extractor that understands the platform's content structure. All platform extractors fall back to generic webpage scraping on failure.

4. **Hybrid search with tunable weights.** Search combines semantic (vector cosine similarity) and lexical (full-text) signals via Reciprocal Rank Fusion (RRF). Weights are dynamically adjusted based on query intent classification.

5. **Knowledge graph as enrichment layer.** The graph database stores extracted entities, categories, and concepts as a discovery and filtering mechanism. The relational database remains the source of truth for bookmark data. Graph operations never block core CRUD.

6. **Fire-and-forget post-processing.** Graph entity extraction and content chunking run in background tasks after the bookmark is persisted. Their success or failure does not affect the bookmark's existence.

---

## 5. Core Domain Model

### 5.1 Entities

#### 5.1.1 Bookmark

The core entity. Represents a saved URL or freeform note with extracted content.

Fields:

- `id` (uuid, required) — Primary key, auto-generated
- `url` (string or null) — The bookmarked URL. Null for standalone notes. Non-null URLs are unique across all bookmarks.
- `title` (string or null) — Page title extracted during ingestion
- `content` (string, required) — Full extracted text content. Never null or empty.
- `summary` (string or null) — AI-generated summary, content-type-specific format
- `notes` (string or null) — User-provided freeform notes (markdown)
- `language` (string or null) — Detected language: `"en"`, `"es"`, or `"mixed"`
- `embedding` (vector or null) — Embedding vector (3072 dims for Gemini, 1536 for OpenAI)
- `content_type` (string) — One of: `"webpage"`, `"youtube"`, `"twitter"`, `"instagram"`, `"tiktok"`, `"article"`, `"note"`. Defaults to `"webpage"`.
- `metadata` (json_object or null) — Platform-specific metadata (see Metadata Schema in Section 9)
- `read_status` (boolean, required) — Whether the user has marked this as read. Defaults to `false`. Never null.
- `read_at` (timestamp or null) — When the bookmark was marked as read. Null when unread.
- `job_id` (uuid or null) — Reference to the job that created/last processed this bookmark. Set to null if job is deleted.
- `is_chunked` (boolean) — Whether content has been split into chunks. Defaults to `false`.
- `chunk_count` (integer) — Number of chunks. Defaults to `0`.
- `archive_url` (string or null) — Archive.today URL if content was retrieved via archive
- `archive_checked_at` (timestamp or null) — When archive availability was last checked
- `tsv` (tsvector, generated) — English full-text search vector, auto-computed from `title` + `content`
- `tsv_es` (tsvector, generated) — Spanish full-text search vector, auto-computed from `title` + `content`
- `note_tsv` (tsvector, generated) — English full-text search vector for `notes`
- `created_at` (timestamp) — When the bookmark was created. Defaults to now.
- `updated_at` (timestamp) — Auto-updated on every modification via database trigger.

#### 5.1.2 BookmarkChunk

A segment of a large bookmark's content, independently embedded for granular search.

Fields:

- `id` (uuid, required) — Primary key, auto-generated
- `bookmark_id` (uuid, required) — FK to Bookmark. Cascade deletes when bookmark is deleted.
- `chunk_index` (integer, required) — Zero-based position in the document
- `content` (string, required) — Chunk text
- `start_char` (integer, required) — Starting character position in original content
- `end_char` (integer, required) — Ending character position in original content
- `overlap_start` (integer or null) — Overlap with previous chunk
- `overlap_end` (integer or null) — Overlap with next chunk
- `embedding` (vector or null) — Chunk embedding vector
- `language` (string or null) — Detected language of chunk
- `tsv` (tsvector, generated) — English full-text search vector
- `tsv_es` (tsvector, generated) — Spanish full-text search vector
- `created_at` (timestamp) — Defaults to now

Unique constraint: `(bookmark_id, chunk_index)` — no duplicate chunk positions per bookmark.

#### 5.1.3 BookmarkJob

Tracks asynchronous bookmark processing.

Fields:

- `id` (uuid, required) — Primary key, auto-generated
- `url` (string, required) — URL being processed
- `notes` (string or null) — Notes associated with the job
- `status` (enum, required) — One of: `"pending"`, `"processing"`, `"completed"`, `"failed"`. Defaults to `"pending"`.
- `bookmark_id` (uuid or null) — FK to Bookmark created by this job. Cascade deletes.
- `retry_count` (integer) — Number of retry attempts. Defaults to `0`.
- `error_message` (string or null) — Error description when status is `"failed"`
- `metadata` (json_object) — Operation metadata (e.g., `{"operation": "reextract"}`). Defaults to `{}`.
- `created_at` (timestamp) — Defaults to now
- `started_at` (timestamp or null) — When processing began
- `completed_at` (timestamp or null) — When processing finished
- `updated_at` (timestamp) — Auto-updated on modification

#### 5.1.4 ArchiveCache

Caches archive.today availability checks to avoid redundant lookups.

Fields:

- `id` (uuid, required) — Primary key
- `original_url` (string, required, unique) — The original URL checked
- `archive_url` (string or null) — Archive URL if available
- `is_available` (boolean, required) — Whether an archive was found. Defaults to `false`.
- `checked_at` (timestamp, required) — When last checked
- `created_at` (timestamp, required) — When created

Cleanup: entries older than 30 days are automatically deleted.

### 5.2 Stable Identifiers and Normalization Rules

- **Bookmark ID**: UUID v4, auto-generated. Used for all API references and graph node identity.
- **URL uniqueness**: Enforced via partial unique index (`WHERE url IS NOT NULL`). Multiple null-URL bookmarks (standalone notes) are permitted.
- **Content type**: Lowercased string from a fixed set. Determined by URL pattern matching (see Section 14).
- **Graph node names**: Case-sensitive as extracted by the LLM. MERGE operations use exact name matching.
- **Job status**: PostgreSQL ENUM type — only `pending`, `processing`, `completed`, `failed` are valid.

---

## 6. System Invariants

Properties that must hold across ALL implementations, regardless of language, architecture, or internal design. An implementation that violates any invariant is incorrect by definition.

### 6.1 Safety Invariants

- **SAFE-001**: The system runs on localhost or a private network with no authentication. All API endpoints are unauthenticated. Deploying on a public network without additional authentication is a security violation. *Rationale: single-user system not designed for multi-tenant access.*

- **SAFE-002**: API tokens (OpenAI, Gemini, Tavily, Langfuse, Neo4j password) are never logged or included in API responses. *Rationale: prevents credential exposure in logs or client-visible output.*

### 6.2 System Invariants

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

- **INV-013**: Knowledge graph highlight spans (`<span class="kg-highlight">`) are never placed inside `<code>`, `<pre>`, `<script>`, `<style>`, or `<sup>` elements. *Rationale: highlighting inside code blocks would corrupt code examples; highlighting inside `<sup>` would break citation references.*

- **INV-014**: Every citation event sent via SSE corresponds to a `[N]` reference that actually appears in the LLM-generated answer text. Citation events are never sent for sources the LLM did not cite. *Rationale: showing unused sources would confuse users and undermine trust in the citation system.*

- **INV-015**: The knowledge graph type-to-color mapping is consistent across all UI surfaces (answer highlighting, citation pills, detail modal pills, autocomplete pills). Categories are always blue, concepts are always green, entities are always orange. *Rationale: users learn the color coding once and rely on it everywhere.*

### 6.3 Verification

Each invariant should be verifiable by:
1. Running after any state-mutating operation (bookmark create/update/delete)
2. Running as a continuous production check (periodic database consistency queries)
3. Including in property-based test suites with generated inputs

---

## 7. Behavioral Properties

Universal truths about function behavior that hold for ALL valid inputs. These are specified as properties for use with generative testing frameworks.

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

- **PROP-013**: `GenerateContentSummary`: Content type determines summary format. YouTube content always produces a descriptive title, introductory paragraph, and timestamped outlines in the content's language. Articles always produce 5-section numbered structure. Social media always produces 2-3 paragraphs. *Formal: `contentType == "youtube" => summary has title + intro paragraph + timestamps in MM:SS or HH:MM:SS` and `contentType in ("webpage", "article") => summary matches 5-section format` and `contentType in ("twitter", "tiktok") => paragraph_count(summary) <= 3`*

- **PROP-014**: `GenerateContentSummary`: No summary output begins with preamble phrases. *Formal: `for all summaries s: !starts_with(s, "Here is") && !starts_with(s, "Of course") && !starts_with(s, "Sure") && !starts_with(s, "I've created") && !starts_with(s, "Below is")`*

- **PROP-015**: `getSystemPromptForIntent`: Every non-command intent produces a system prompt that contains the base prompt as a prefix and includes citation `[N]` instructions. *Formal: `for all intents i where i != "command": starts_with(getSystemPromptForIntent(i), basePrompt) && contains(getSystemPromptForIntent(i), "[N]")`*

- **PROP-016**: `buildContextTemplate`: The context template for N search results contains exactly N source blocks numbered `[1]` through `[N]`, and ends with the user's question. *Formal: `for i in 1..N: contains(context, "Source [" + i + "]")` and `ends_with(context, "User Question: " + query)`*

- **PROP-017**: `extractCitedNumbers`: Returns only unique citation numbers that appear as `[N]` in the text, in order of first appearance. *Formal: `for all n in extractCitedNumbers(text): text contains "[" + n + "]"` and `len(extractCitedNumbers(text)) == len(set(extractCitedNumbers(text)))`*

- **PROP-018**: `formatAnswer`: Every `[N]` pattern in the input markdown is transformed into a `<sup>` element containing an `<a>` link targeting `#citation-N`. No `[N]` patterns remain as plain text in the output. *Formal: `count("[N]" in formatAnswer(text)) == 0` and `count("<sup>...[N]...</sup>" in formatAnswer(text)) == count("[N]" in text)`*

- **PROP-019**: `highlightTextNodes`: Highlighted spans never overlap. When two knowledge graph terms would overlap in the text, only the longer match is kept. If lengths are equal, entity > concept > category priority applies. *Formal: `for all spans s1, s2 in highlights: s1.range ∩ s2.range == ∅`*

- **PROP-020**: `highlightTextNodes`: Idempotent. Running the highlighting algorithm on already-highlighted content produces no additional spans (because `.kg-highlight` elements are in the exclusion list). *Formal: `highlightTextNodes(highlightTextNodes(element, terms), terms) == highlightTextNodes(element, terms)`*

- **PROP-021**: `renderContentPreview`: Content that matches any of the 11 markdown detection patterns is rendered via `marked.parse()`. Content that matches none is rendered with manual paragraph splitting and bold conversion. *Formal: `isMarkdown(content) => output contains "<div class=\"markdown-content\">"` and `!isMarkdown(content) => output contains "<p class=\"mb-4\">"}`*

- **PROP-022**: `getSnippet`: Snippets are always ≤ 200 characters (plus "..." suffix when truncated). Content is preferred over notes as the snippet source. *Formal: `len(snippet) <= 203` and `content != "" => snippet derived from content`*

- **PROP-023**: `GenerateContentSummary`: Summary language matches content language. When the input content is in a specific language, the summary output is in that same language. *Formal: `detected_language(content) == detected_language(summary)` for all content types*

---

## 8. State Machines

### 8.1 Job Lifecycle States

The job processing system tracks bookmark ingestion through a strict state machine.

1. `pending`
   - Initial state when a job is created via `POST /add` or `PUT /bookmark/{id}/reextract`
   - Job is queued but no worker has picked it up

2. `processing`
   - A worker has claimed the job and is actively processing it
   - Scraping, embedding, summarization, graph extraction, chunking in progress

3. `completed` (terminal)
   - All processing finished successfully
   - `bookmark_id` is set to the created/updated bookmark

4. `failed` (terminal)
   - Processing failed after all retry attempts exhausted
   - `error_message` contains the failure description

### 8.2 Transition Triggers

- `pending` → `processing`: Worker picks up job from queue
- `processing` → `completed`: All processing steps succeed
- `processing` → `failed`: Non-retryable error, or max retries (3) exhausted
- `processing` → `processing`: Retryable error triggers retry (stays in processing, increments `retry_count`)

### 8.3 Idempotency and Recovery Rules

- Job status transitions are forward-only (INV-007). No transition back to `pending`.
- Completed and failed jobs older than 7 days are automatically cleaned up.
- On server restart, in-flight `processing` jobs are NOT automatically recovered — they remain in `processing` state until manual intervention or cleanup.
- The `updated_at` timestamp tracks the last state change.

---

## 9. Interface Contracts

Contracts specify what crosses boundaries between components. Each contract survives reimplementation of either side.

### 9.1 Client → Backend: Add Bookmark

**Protocol**: HTTP POST

**Endpoint**: `/add` (alias: `/note` — identical behavior, provided for semantic clarity in UI code)

**Request schema**:
```
{
  "url": string (optional) — The URL to bookmark,
  "notes": string (optional) — Freeform notes or text,
  "created_at": string (optional) — ISO 8601 timestamp for backdating
}
```
At least one of `url` or `notes` must be provided.

**Response schema (202 Accepted)**:
```
{
  "success": true,
  "message": "Bookmark queued for processing",
  "job_id": string — UUID for polling job status
}
```

**Response schema (409 Conflict)**:
```
{
  "success": false,
  "message": "This URL has already been bookmarked",
  "id": string — UUID of existing bookmark
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

### 9.2 Client → Backend: Ask Question (SSE Stream)

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
  "number": integer — Citation reference number [N] in the answer text,
  "id": string — Bookmark UUID,
  "title": string — Bookmark title,
  "url": string — Original URL,
  "domain": string — URL domain,
  "content_type": string — "webpage", "youtube", "twitter", etc.,
  "created_at": string — ISO 8601 date
}
```

**Contract invariants**:
- Every citation `number` corresponds to a `[N]` reference in the answer text
- Only actually-cited sources are sent as citation events (not all search results)
- The stream always terminates with an `event: done` event
- Citation events are sent after all answer text events

---

### 9.3 Client → Backend: Ask Recent (SSE Stream)

**Protocol**: HTTP GET with Server-Sent Events response

**Endpoint**: `/answer/recent?q={query}&days={n}&limit={n}`

**Request** (query parameters):
- `q` (required): natural language question
- `days` (optional, default 7): number of days to look back
- `limit` (optional, default 20): max bookmarks to include as context

**SSE Event Types**: Same as `/answer` — `data:` chunks, `event: error`, `event: done`.

**Contract invariants**:
- Returns 400 if `q` is missing
- If no bookmarks found in the time range, sends a message without calling the LLM
- Bookmarks are ordered newest-first
- Citations are formatted as markdown in a final data event (not as separate citation events)

---

### 9.4 Client → Backend: List Bookmarks

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
  "snippet": string — First 200 characters of content, used as a preview in list views,
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

### 9.5 Client → Backend: Delete Bookmark

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

### 9.6 Client → Backend: Job Status (Polling)

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

### 9.7 Client → Backend: Job Status (SSE Stream)

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

### 9.8 Client → Backend: Bookmark Detail

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

### 9.9 Metadata Schema

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

### 9.10 Client → Backend: Toggle Read Status

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

### 9.11 Client → Backend: Update Notes

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

### 9.12 Client → Backend: Re-extract Content

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

### 9.13 Client → Backend: Check URL Existence

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

### 9.14 Client → Backend: Analyze URL

**Protocol**: HTTP POST

**Endpoint**: `/bookmarks/analyze-url`

**Request schema**:
```
{
  "url": string (required) — URL to analyze,
  "title": string (optional) — Pre-extracted page title,
  "content": string (optional) — Pre-extracted content,
  "fast_mode": boolean (optional) — Skip content fetching and AI analysis
}
```

**Response schema (200)**:
```
{
  "similar_bookmarks": [
    {
      "id": string,
      "url": string,
      "title": string,
      "excerpt": string — truncated to 200 chars,
      "similarity_score": float,
      "saved_date": string (ISO 8601),
      "categories": [string]
    }
  ],
  "categories": [{name, count, description, level}],
  "tags": {
    "categories": [{name, count, description}],
    "concepts": [{name, count}],
    "entities": [{name, count, type}]
  },
  "analysis": {
    "summary": string,
    "key_differences": [string],
    "unique_aspects": [string],
    "recommendation": string
  },
  "recommendation": "save" | "update" | "skip" | "duplicate"
}
```

**Contract invariants**:
- If exact URL match found, immediately returns with `recommendation: "duplicate"` and `similarity_score: 1.0`
- Uses 70/30 semantic/lexical weight for similarity search
- Tags are limited to top 10 per type, sorted by count
- Fast mode skips content fetching, embedding generation, and AI analysis
- Content is truncated to 24,000 chars for embedding generation

---

### 9.15 Client → Backend: Health Check

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

### 9.16 Client → Backend: Autocomplete

**Protocol**: HTTP GET

**Endpoint**: `/autocomplete?q={query}&limit={n}`

**Response schema (200)**:
```
[
  {
    "name": string — Node name (e.g., "React", "Technology"),
    "type": string — Node type: "Category" | "Concept" | "Topic" | "Person" | "Organization" | "Technology" | "Project",
    "count": integer — Number of bookmarks associated with this node,
    "description": string (optional) — Node description,
    "score": float (optional) — Relevance score (1.0 = exact match, 0.8 = starts-with, 0.6 = contains)
  }
]
```

**Contract invariants**:
- Returns empty array `[]` on any error or when graph service is unavailable
- `limit` is capped at 20
- Results are ordered by relevance score descending, then bookmark count descending

---

### 9.17 Client → Backend: Related Bookmarks

**Protocol**: HTTP GET

**Endpoint**: `/bookmarks/related?type={type}&name={name}`

**Request** (query parameters):
```
{
  "type": "category" | "concept" | "entity" (required),
  "name": string (required) — Graph node name to search by
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
  "type": string — Echo of the requested type,
  "name": string — Echo of the requested name
}
```

**Error schema (400)**: Missing or invalid `type` or `name` parameter.

**Error schema (503)**: Graph service unavailable.

**Contract invariants**:
- `type` must be one of: `"category"`, `"concept"`, `"entity"`
- Both `type` and `name` are required
- If the graph service is unavailable, returns 503 (not an empty result)
- The `score` field represents graph relationship strength

---

### 9.18 Client → Backend: Categories

**Protocol**: HTTP GET

**Endpoint**: `/categories`

**Request** (query parameters):
```
{
  "with_counts": "true" (optional) — Include bookmark and subcategory counts
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
  "count": integer — Total number of categories
}
```

**Error schema (501)**: Graph service not configured.

**Contract invariants**:
- Requires graph service; returns 501 if not available
- `bookmark_count` and `sub_category_count` are only present when `with_counts=true`

---

### 9.19 Client → Backend: Knowledge Graph Terms

**Protocol**: HTTP GET

**Endpoint**: `/knowledge-terms`

**Response schema (200)**:
```
{
  "categories": [{ "name": string, "count": integer, "level": integer }],
  "concepts": [{ "name": string, "count": integer }],
  "entities": [{ "name": string, "type": string, "count": integer }],
  "total": integer — sum of all three array lengths
}
```

**Error schema (501)**: Graph service not enabled (returns `{"error": "Graph service not enabled"}`).

**Contract invariants**:
- Categories are filtered to only those with `bookmark_count > 0`
- On category fetch error, returns empty array rather than failing
- `total` is the sum of all three array lengths

---

### 9.20 Client → Backend: Graph Explore

**Protocol**: HTTP POST

**Endpoint**: `/graph/explore`

**Request schema**:
```
{
  "bookmark_id": string (required) — UUID of bookmark to explore from,
  "depth": integer (optional, default 2) — Traversal depth
}
```

**Response schema (200)**:
```
{
  "related_bookmarks": [
    {
      "bookmark_id": string,
      "url": string,
      "title": string,
      "score": float,
      "graph_context": object,
      "categories": [string],
      "concepts": [string],
      "entities": [string]
    }
  ]
}
```

**Error responses**: 400 (missing bookmark_id), 500 (graph exploration failure).

---

### 9.21 Client → Backend: Graph Search

**Protocol**: HTTP POST

**Endpoint**: `/graph/search`

**Request schema**:
```
{
  "category": string (optional),
  "concept": string (optional)
}
```
Exactly one of `category` or `concept` must be provided. `category` takes precedence if both are set.

**Response schema (200)**:
```
{
  "results": [
    {
      "bookmark_id": string,
      "url": string,
      "title": string,
      "score": float,
      "graph_context": object,
      "categories": [string],
      "concepts": [string],
      "entities": [string]
    }
  ]
}
```

**Error responses**: 400 (neither field provided), 500 (graph search failure).

---

### 9.22 Client → Backend: Graph Index All

**Protocol**: HTTP POST

**Endpoint**: `/graph/index`

**Request**: Empty body.

**Response schema (202 Accepted)**:
```
{
  "message": "Started indexing N bookmarks",
  "count": integer
}
```

**Contract invariants**:
- Indexing runs asynchronously — the 202 response returns immediately
- Failures during async indexing are logged, not surfaced to the caller

---

### 9.23 Client → Backend: API Documentation

**Protocol**: HTTP GET

**Endpoints**:
- `GET /api` — Redirects (301) to `/api/docs`
- `GET /api/docs` — Serves Swagger UI (inline HTML page loading swagger-ui from CDN)
- `GET /api/openapi.yaml` — Serves the embedded OpenAPI 3.0 specification

**Contract invariants**:
- `/api/openapi.yaml` is served with `Content-Type: application/x-yaml` and `Access-Control-Allow-Origin: *`
- Swagger UI is configured with `tryItOutEnabled: true`

---

## 10. Configuration Specification

### 10.1 Source Precedence

Configuration values are resolved in this order (first wins):

1. Environment variables (set directly or via `.env` file loaded at startup by `godotenv`)
2. Built-in defaults (hardcoded in source)

### 10.2 Configuration Fields

#### Core Server

- `SERVER_PORT`: string, default `"8080"` — HTTP server listen port
- `DATABASE_URL`: string, **required** — PostgreSQL connection string (e.g., `postgres://user@localhost:5432/brainy_db?sslmode=disable`)

#### Embedding Provider Selection

- `EMBEDDING_PROVIDER`: string, default `"openai"` — Which embedding provider to use. Values: `"openai"` or `"gemini"`.

#### OpenAI Configuration

- `OPENAI_API_KEY`: string, **required when provider=openai** — OpenAI API key
- `OPENAI_CHAT_MODEL`: string, default `"gpt-4o-mini"` — Model for chat completions
- `OPENAI_EMBEDDING_MODEL`: string, default `"text-embedding-3-small"` — Model for embeddings (1536 dimensions)
- `OPENAI_SUMMARY_MODEL`: string, default `"gpt-4o-mini"` — Model for summary generation

#### Google Gemini Configuration

- `GEMINI_API_KEY`: string, **required for Gemini** (unless using Vertex AI) — Gemini API key
- `GEMINI_CHAT_MODEL`: string, default `"gemini-2.0-flash-exp"` — Model for chat completions
- `GEMINI_EMBEDDING_MODEL`: string, default `"text-embedding-004"` — Model for embeddings
- `GEMINI_SUMMARY_MODEL`: string, default `"gemini-2.0-flash-exp"` — Model for summary generation
- `GEMINI_EMBEDDING_DIMENSIONS`: string (parsed as int), default `"3072"` — Embedding vector dimensions. Valid: `768`, `1536`, `3072`.
- `GOOGLE_APPLICATION_CREDENTIALS`: string (file path), optional — Alternative to `GEMINI_API_KEY` for Google Cloud ADC
- `GOOGLE_GENAI_USE_VERTEXAI`: string, optional — Set to `"true"` to use Vertex AI instead of Gemini API
- `GOOGLE_CLOUD_PROJECT`: string, required if Vertex AI — Google Cloud project ID
- `GOOGLE_CLOUD_LOCATION`: string, default `"us-central1"` — Vertex AI region

#### Neo4j Knowledge Graph

- `ENABLE_GRAPH_EXTRACTION`: string, default `"true"` — Feature toggle. Set to `"false"` to disable graph features.
- `NEO4J_URI`: string, default `"bolt://localhost:7687"` — Neo4j connection URI
- `NEO4J_USER`: string, default `"neo4j"` — Neo4j username
- `NEO4J_PASSWORD`: string, default `"brainy_password"` — Neo4j password

#### Langfuse Observability

- `LANGFUSE_PUBLIC_KEY`: string, optional — Enables Langfuse tracing when both keys present
- `LANGFUSE_SECRET_KEY`: string, optional — Enables Langfuse tracing when both keys present
- `LANGFUSE_HOST`: string, optional — Custom Langfuse API endpoint (consumed by SDK)
- `LANGFUSE_ENABLED`: string, default `"true"` — Set to `"false"` to disable even when keys are present

#### Archive Fallback

- `ENABLE_ARCHIVE_FALLBACK`: string, default `"true"` — Toggle archive.today paywall bypass
- `ARCHIVE_DOMAINS`: string (comma-separated), optional — Custom archive domains

#### Tavily Integration

- `TAVILY_API_KEY`: string, optional — Tavily API key for advanced content extraction
- `TAVILY_ENABLED`: string, optional — Must be `"true"` to enable Tavily (requires API key)

#### Social Media

- `FACEBOOK_ACCESS_TOKEN`: string, optional — Used for Instagram content extraction
- `INSTAGRAM_ACCESS_TOKEN`: string, optional — Used for Instagram oEmbed in URL analysis

#### Debugging

- `PROMPT_LOGGING`: string, default `"true"` — Log AI prompts and responses to files. Set to `"false"` to disable.

### 10.3 Hardcoded Constants

These values are compiled into the binary and cannot be changed at runtime:

#### HTTP Server
- Read/Write/Idle Timeout: 5 minutes each
- Graceful shutdown timeout: 30 seconds
- Langfuse shutdown timeout: 5 seconds

#### Job Queue
- Workers: 5
- Buffer size: 100
- Max retries: 3
- Backoff: exponential (1s, 2s, 4s)
- SSE poll interval: 500ms

#### Neo4j Connection Pool
- Max connections: 50
- Acquisition timeout: 60 seconds
- Socket connect timeout: 30 seconds
- Write retry attempts: 3

#### OpenAI Limits
- Max embedding tokens: 6,000
- Max chars for embedding: 24,000 (6,000 × 4 chars/token)
- Content too long for summarization: 100,000 chars

#### Gemini Limits
- Max embedding tokens: 20,000
- Max chars for embedding: 80,000 (20,000 × 4 chars/token)
- Content too long for summarization: 200,000 chars

#### Content Processing
- Clean text max length: 500,000 chars
- Article content minimum: 100 chars
- Archive content max for AI cleaning: 50,000 chars

#### Content Chunking
- MaxChunkSize: 4,000 chars (~1,000 tokens)
- OverlapSize: 400 chars (~100 tokens)
- MinChunkSize: 100 chars
- ChunkingThreshold: 24,000 chars

#### Chat Completion Defaults
- Streaming chat: temperature 0.7, max tokens 2,000
- Summary (OpenAI): temperature 0.5, max tokens varies by type (YouTube: 4,000, social: 1,000, article: 2,000)
- Summary (Gemini): temperature 0.5, max tokens 8,000 (social: 1,000)
- Smart model routing (Gemini): large model for JSON extraction >10K tokens or content >20K chars

#### UI Constants
- KG term cache TTL: 5 minutes
- Citation ring highlight duration: 2 seconds
- Snippet max length: 200 chars
- Search debounce: 300ms
- Autocomplete blur delay: 200ms
- Highlighting debounce: 500ms
- KG term min length: 3 chars, max length: 50 chars
- Infinite scroll trigger: 100px from bottom
- Items per page: 20
- Notification auto-dismiss: 5 seconds

### 10.4 Validation Rules

Startup validation (blocks startup if failed):
- `DATABASE_URL` must be set and connectable
- At least one embedding provider must be configured (either `OPENAI_API_KEY` or `GEMINI_API_KEY`)
- If `EMBEDDING_PROVIDER=gemini`, `GEMINI_API_KEY` or `GOOGLE_APPLICATION_CREDENTIALS` must be set
- `GEMINI_EMBEDDING_DIMENSIONS` must be one of: 768, 1536, 3072

Per-operation validation:
- Neo4j connection failure disables graph features (does not crash server)
- Langfuse initialization failure disables tracing (does not crash server)
- Tavily initialization failure disables enhanced scraping (does not crash server)

---

## 11. Output Structure

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

## 12. Type Conventions

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

## 13. Error Handling

Errors are reported as JSON responses with appropriate HTTP status codes:

| Status Code | Meaning | When Used |
|-------------|---------|-----------|
| 400 | Bad Request | Missing required fields, invalid parameters |
| 404 | Not Found | Bookmark or job ID does not exist |
| 405 | Method Not Allowed | Wrong HTTP method for endpoint |
| 409 | Conflict | URL already bookmarked (duplicate) |
| 500 | Internal Server Error | Database errors, embedding generation failures |
| 501 | Not Implemented | Graph service endpoint when graph is not configured |
| 503 | Service Unavailable | Graph service temporarily unavailable |

SSE streams report errors as `event: error` events rather than HTTP status codes, since the connection is already established.

---

## 14. Content Type Detection

URL type detection follows a priority-ordered chain. The first match wins:

| Priority | Content Type | URL Pattern |
|----------|-------------|-------------|
| 1 | `youtube` | `youtube.com/watch`, `youtu.be/`, `youtube.com/shorts/`, `youtube.com/embed/`, `youtube.com/live/`, `m.youtube.com/watch` |
| 2 | `twitter` | `twitter.com/*/status/*`, `x.com/*/status/*`, `threadreaderapp.com/thread/*` |
| 3 | `instagram` | `instagram.com/p/*`, `instagram.com/reel/*`, `instagram.com/tv/*` |
| 4 | `tiktok` | `tiktok.com/@*/video/*` (15-19 digit IDs), `vt.tiktok.com/*`, `vm.tiktok.com/*` |
| 5 | `webpage` | Everything else (default) |

---

## 15. Functions

### 15.1 Hybrid Search Algorithm

#### Scoring Methods

The system uses two distinct scoring methods depending on the detected language of the query:

**Path A: Direct Similarity Blending** (default, English queries)
```
combined_score = MAX((cosine_similarity * 0.6) + (normalized_text_rank * 0.4), 0.01)
```
Where `cosine_similarity = 1 - cosine_distance` and `normalized_text_rank = full_text_rank / 10.0`.

**Path B: Reciprocal Rank Fusion** (non-English queries, e.g., Spanish)
```
rrf_score(rank, k) = 1.0 / (k + rank)   — where k defaults to 50
final_score = (rrf_score(semantic_rank, k) * semantic_weight) + (rrf_score(keyword_rank, k) * lexical_weight)
```

**Path selection**: When the query is detected as non-English (e.g., Spanish), the system uses Path B (RRF) which queries both the English (`tsv`) and Spanish (`tsv_es`) full-text search vectors and merges results via rank fusion. English queries use Path A (Direct Similarity Blending) with only the English `tsv` vector.

#### Query Intent Classification

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

#### Unified Search (Chunked + Non-chunked)

For large documents that have been chunked:
1. Search chunked bookmarks via `hybrid_search_chunks` (returns best chunk per bookmark)
2. Search non-chunked bookmarks via `hybrid_search_bookmarks`
3. Merge results, deduplicate by bookmark ID, sort by combined score
4. If chunk search fails, gracefully degrade to non-chunked results only

### 15.2 Answer Generation

The `/answer` endpoint generates AI-powered answers to user questions using search results as context. The answer system uses a two-layer prompt architecture: a base system prompt shared across all intents, plus intent-specific suffixes that tailor the AI's behavior.

#### Base System Prompt

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

#### Intent-Specific System Prompts

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

#### Context Template Format

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

#### Result Filtering

Before building the context, search results are filtered:
- Maximum 20 results included in the context
- Results with score > 0.3 are included even beyond the 20-result limit
- Results are ordered by combined score descending

#### Search Strategy Parameters

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

### 15.3 Answer Rendering Pipeline

The answer rendering pipeline transforms the raw LLM-generated markdown stream into a rich, interactive HTML experience. The pipeline has four sequential stages, each building on the previous one's output.

#### Stage 1: Markdown Rendering

Raw answer text (received as SSE chunks) is rendered to HTML using a GFM-compliant markdown parser.

**Parser configuration:**

| Setting | Value | Rationale |
|---------|-------|-----------|
| `breaks` | `true` | Convert single `\n` to `<br>` for line-level formatting |
| `gfm` | `true` | Support GitHub-Flavored Markdown (tables, strikethrough, task lists) |
| `smartLists` | `true` | Better list formatting with mixed numbering |
| `smartypants` | `true` | Typographic quotes and dashes |
| `sanitize` | `false` | Allow HTML passthrough (required for downstream processing) |

**Newline handling for SSE streams:** Streaming responses encode newlines as literal `\n` escape sequences within SSE `data:` fields. The client decodes these (`\\n` → `\n`) before markdown parsing. For non-streaming responses (line-by-line SSE events), the client inserts newlines between events unless the accumulated text already ends with a newline.

**Error fallback:** If the markdown parser is unavailable or throws an error, the answer is displayed as HTML-escaped plain text (`<` → `&lt;`, `>` → `&gt;`).

#### Stage 2: Citation Transformation

After markdown rendering, all `[N]` patterns in the HTML are transformed into interactive citation links.

**Transformation:**
```
[N] → <sup class="text-blue-600 font-semibold">
        <a href="#citation-N" onclick="scrollToCitation(N)">[N]</a>
      </sup>
```

**Citation click behavior:**
1. Smooth-scroll the citation card (`#citation-N`) into the viewport center
2. Add a blue ring highlight (`ring-2 ring-blue-500 ring-offset-2`) to the card
3. Remove the ring highlight after 2 seconds

**Citation card rendering:**

Citation cards appear in a "Sources" section below the answer. Each card contains:

| Element | Display | Behavior |
|---------|---------|----------|
| Number badge | Blue circle with citation number | Visual anchor |
| Thumbnail | 128×80 image from metadata, with fallback icon | Platform-aware |
| Title | Bookmark title, truncated | Clickable → opens detail modal |
| URL | Truncated to 100 chars with external link icon | Opens in new tab |
| Category pills | Blue rounded badges | Clickable → triggers related bookmark search |
| Concept pills | Green rounded badges | Clickable → triggers related bookmark search |
| Entity pills | Orange rounded badges with type label | Clickable → triggers related bookmark search |
| YouTube metadata | Channel name + duration (when applicable) | With red video icon |
| Date | Creation date | Static display |

**Citation filtering invariant:** Only sources whose `[N]` number appears in the LLM response text are rendered as citation cards. The backend extracts cited numbers via regex `\[(\d+)\]`, deduplicates them, and only sends citation SSE events for matched sources.

#### Stage 3: Knowledge Graph Term Highlighting

After the answer is fully rendered (streaming complete), knowledge graph terms are overlaid as interactive highlights on the rendered HTML. See Section 15.5 for the full specification.

#### Stage 4: Markdown CSS Styling

All rendered markdown content (answers, summaries, content previews) is styled via a `.markdown-content` CSS class that provides consistent typography:

| Element | Style |
|---------|-------|
| Body text | 1rem, line-height 1.75, gray-700 |
| `h1` | 1.875rem bold, gray-900 |
| `h2` | 1.5rem bold, gray-800 |
| `h3` | 1.25rem semibold, gray-800 |
| `strong`/`b` | font-weight 700, gray-900 (#111827) |
| `em` | Italic, gray-600 |
| Inline `code` | Gray-100 background, monospace, red-600 text |
| Code blocks (`pre`) | Dark background (#1f2937), gray-100 text, rounded |
| `blockquote` | Left blue border (4px), italic, gray-700 |
| Tables | Full-width, collapsed borders, alternating row shading |
| Links | Blue-600, underlined, darker on hover |
| `sup` (citations) | 0.75em, vertical-align super, bold blue links |
| Lists | Proper indentation (ml-6), disc/decimal markers, spacing |
| Images | Rounded, shadow, responsive max-width |

### 15.4 Content Display Formatting

Content and summaries are displayed differently depending on the UI context.

#### Markdown Auto-Detection

When displaying content that may or may not be markdown, the system tests against 11 regex patterns:

| Pattern | Matches |
|---------|---------|
| `/^#{1,6}\s/m` | Headers (`# Title`) |
| `/\*\*[^*]+\*\*/` | Bold (`**text**`) |
| `/\*[^*]+\*/` | Italic (`*text*`) |
| `/\[.+\]\(.+\)/` | Links (`[text](url)`) |
| `/^[\*\-]\s/m` | Unordered lists (`- item`) |
| `/^\d+\.\s/m` | Ordered lists (`1. item`) |
| `/^>\s/m` | Blockquotes (`> text`) |
| `/```[\s\S]*```/` | Fenced code blocks |
| `/`[^`]+`/` | Inline code |
| `/^\|.+\|/m` | Tables (`| cell |`) |
| `/^---+$/m` | Horizontal rules |

If **any** pattern matches, the content is rendered via the markdown parser and wrapped in `<div class="markdown-content">`. Otherwise, the plain-text fallback rendering is used.

#### Rendering by View Context

| Context | Source Field | Rendering | Truncation |
|---------|-------------|-----------|------------|
| Bookmark list card | `summary` (fallback: `snippet`) | Plain text via `x-text` | CSS 2-line clamp (`line-clamp-2`) |
| Ask view answer | LLM markdown stream | Full markdown pipeline (Stage 1-4) | None |
| Detail modal: summary | `summary` | Markdown auto-detect → `marked.parse()` or plain-text fallback | None |
| Detail modal: content | `content` | Markdown auto-detect → `marked.parse()` or plain-text fallback | 1500 chars default, expandable |
| Citation card | — | Structured HTML card | Title truncated, URL truncated to 100 chars |

#### Plain-Text Fallback Rendering

When content does not match markdown patterns, the fallback renderer:

1. Truncates to 1500 characters (for content preview) with `...` suffix
2. Normalizes whitespace: `\r\n` → `\n`, collapse triple+ newlines, collapse spaces
3. Converts `**text**` to `<strong class="font-semibold text-gray-900">text</strong>`
4. Converts `Label:` patterns at start of lines to bold: `**Label:**`
5. Splits on double newlines into `<p class="mb-4">` paragraphs
6. Converts remaining single `\n` to `<br>`

#### Snippet Generation (Backend)

Snippets are short plain-text previews generated server-side for use in bookmark list cards and search results.

**Generation rules:**

| Priority | Source | Behavior |
|----------|--------|----------|
| 1 | `content` field | First 200 characters + `...` if truncated |
| 2 | `notes` field | First 200 characters + `...` if truncated (only if content is empty) |
| 3 | Graph relationship | `"Related through: {categories}, {concepts}"` for graph-derived results |

**Search-highlighted snippets:** The database generates highlighted snippets via `ts_headline` with `<mark>` / `</mark>` tags for search term highlighting:

```
ts_headline('english', content, query,
  'StartSel=<mark>, StopSel=</mark>, MaxWords=35, MinWords=15, ShortWord=3, HighlightAll=FALSE')
```

### 15.5 Knowledge Graph Term Highlighting

After an answer finishes streaming, the system scans the rendered HTML for known knowledge graph terms (categories, concepts, entities) and wraps matching text in interactive, color-coded highlight spans.

#### Term Data Source

Terms are loaded from `GET /knowledge-terms` and cached client-side with a 5-minute TTL.

#### Color System

The knowledge graph type-to-color mapping is consistent across all UI surfaces:

| Type | Background | Text | Border | Icon | Usage |
|------|-----------|------|--------|------|-------|
| Category | `bg-blue-100` | `text-blue-700` | `border-blue-300` | 📁 | Answer highlights, citation pills, detail pills, autocomplete |
| Concept | `bg-green-100` | `text-green-700` | `border-green-300` | 💡 | Answer highlights, citation pills, detail pills, autocomplete |
| Entity | `bg-orange-100` | `text-orange-700` | `border-orange-300` | 🏷️ | Answer highlights, citation pills, detail pills, autocomplete |
| Topic | `bg-purple-100` | `text-purple-700` | — | — | Autocomplete pills, detail modal pills |
| Person | `bg-orange-100` | `text-orange-700` | — | — | Autocomplete pills (entity subtype) |
| Organization | `bg-red-100` | `text-red-700` | — | — | Autocomplete pills (entity subtype) |
| Technology | `bg-indigo-100` | `text-indigo-700` | — | — | Autocomplete pills (entity subtype) |
| Project | `bg-pink-100` | `text-pink-700` | — | — | Autocomplete pills (entity subtype) |

#### Highlighting Algorithm

The highlighting operates on the DOM (not on HTML strings) to avoid corrupting markup:

1. **Collect text nodes:** Use a `TreeWalker` (filter: `SHOW_TEXT`) to find all text nodes in the rendered answer container
2. **Skip excluded elements:** Reject text nodes whose parent matches `code`, `pre`, `script`, `style`, `sup`, or `.kg-highlight`
3. **Match terms:** For each text node, build regex patterns with word boundaries (`\b`) for each knowledge graph term. Sort terms by name length descending (longest first)
4. **Resolve overlaps:** When multiple terms match overlapping ranges in the same text node, keep only the longer match. If lengths are equal, entity (priority 3) > concept (priority 2) > category (priority 1)
5. **Replace text nodes:** Split the text node into fragments, wrapping matched ranges in `<span>` elements with the appropriate highlight class

**Timing:** Highlighting is applied after the answer stream completes, with a 500ms debounce. A fallback `forceHighlight()` function runs 100ms after stream completion as a safety net.

**Term filtering:** Terms shorter than 3 characters or longer than 50 characters are excluded. Terms that look like URLs are excluded.

#### Highlight Span Structure

Each highlighted term produces:

```html
<span class="kg-highlight kg-{type} {color-classes}"
      data-kg-term-encoded="{base64-encoded JSON term data}"
      data-kg-type="{type}"
      data-kg-name="{term name}"
      role="button"
      tabindex="0"
      aria-label="{type}: {term name} ({count} related bookmarks)"
      aria-expanded="false"
      aria-haspopup="dialog">
  {matched text}
</span>
```

#### Highlight Tooltip

Clicking or activating (Enter/Space) a highlighted term opens a floating tooltip:

| Element | Content |
|---------|---------|
| Header | Type icon + term name + type label |
| Bookmark count | "N related bookmarks" |
| Related bookmarks | Up to 3 bookmark links (title, clickable → detail modal) |
| Expand link | "Show all N bookmarks" when count > 3 |
| Explore button | Submits the term as a new question in the Ask view |

**Tooltip dismissal:** Clicking outside, pressing Escape, or clicking another highlight closes the tooltip.

**Accessibility:** The tooltip has `role="dialog"` and `aria-live="polite"`. The highlight span toggles `aria-expanded` on open/close.

#### Highlighting Invariants
- Highlights are never applied inside `<code>`, `<pre>`, `<script>`, `<style>`, or `<sup>` elements (INV-013)
- Running highlighting on already-highlighted content is idempotent — `.kg-highlight` elements are in the exclusion list (PROP-020)
- The color mapping is consistent across all UI surfaces (INV-015)

### 15.6 Content Chunking

#### Configuration

| Parameter | Default Value | Description |
|-----------|--------------|-------------|
| `MaxChunkSize` | 4000 chars (~1000 tokens) | Maximum characters per chunk |
| `OverlapSize` | 400 chars (~100 tokens) | Overlap between adjacent chunks |
| `MinChunkSize` | 100 chars | Minimum viable chunk size |
| `ChunkingThreshold` | 24000 chars | Content length above which chunking is applied |

#### Algorithm

1. If content fits in one chunk, return a single chunk
2. Sliding window: advance by `MaxChunkSize - OverlapSize` characters per step
3. At each step, find the best break point (sentence boundary preferred, then word boundary)
4. Sentence boundaries: `.`, `!`, `?` followed by space/newline (excluding single-letter abbreviations)
5. Ensure forward progress: minimum advance of `MinChunkSize` per step

### 15.7 Content Ingestion Pipeline

#### Processing Steps (per bookmark)

1. **URL type detection** — classify URL into content type
2. **Platform-specific extraction** — fetch content via specialized extractor
3. **Fallback to generic scraping** — if platform extractor fails
4. **Paywall detection** — check for paywalled content (known domains, JSON-LD, HTML patterns)
5. **Archive fallback** — if paywalled, try archive.today via web archive service or direct fetch
6. **Content cleaning** — AI-powered removal of archive UI artifacts (only applied when content was retrieved via archive fallback; skipped for directly-scraped content)
7. **Title resolution** — For URL bookmarks: readability title > OG title > first 100 chars of content. For notes-only bookmarks: always `"Quick Note"`. The UI may override the display title (e.g., showing first 50 chars of notes or "Untitled" when the stored title is empty).
8. **Embedding generation** — via configured embedding provider
9. **Summary generation** — AI summary using content-type-specific prompts (non-blocking, see Section 15.8). YouTube bookmarks get a video-focused prompt, tweets get a thread-focused prompt, and articles get a general article prompt.
10. **Database insert** — upsert bookmark with embedding vector
11. **Graph entity extraction** — async, 2-minute timeout, fire-and-forget
12. **Content chunking** — async, for content >24,000 chars (matching `ChunkingThreshold`), fire-and-forget

#### Retry Policy

- Maximum 3 retry attempts with exponential backoff (1s, 2s, 4s)
- Non-retryable errors: "invalid URL", "paywall detected", "content too large", "embedding limit exceeded"
- Retryable errors: HTTP 500-504, timeouts, connection resets

#### Paywall Detection

Four detection methods, ordered by reliability:

| Method | Confidence | Technique |
|--------|-----------|-----------|
| JSON-LD structured data | 1.0 | `isAccessibleForFree: false` in Article schema |
| Known domain list | 0.9 | 80+ hardcoded publication domains |
| HTML pattern matching | 0.7-0.95 | CSS classes, data attributes, paywall scripts |
| Content analysis | 0.6 | Truncation indicators in short articles (<500 chars) |

### 15.8 Summary Generation

The system generates AI-powered summaries for bookmarks as part of the ingestion pipeline. Summary generation is **non-blocking** — failure never prevents bookmark creation.

There are two distinct summarization paths:

#### Path 1: Content-Type-Aware Summary (User-Facing)

Generated during bookmark ingestion (pipeline step 9) and during re-extraction. Stored in the bookmark's `summary` field.

| Content Type | Prompt Strategy | Max Output Tokens | Temperature |
|---|---|---|---|
| `youtube` | Title + intro paragraph + timestamped standalone-summary outline | 4000 (OpenAI) / 8000 (Gemini) | 0.5 |
| `twitter` / `tiktok` | Main message, key facts, context, and tone in 2-3 paragraphs | 1000 | 0.5 |
| `article` / `webpage` | 5-section numbered structure | 2000 (OpenAI) / 8000 (Gemini) | 0.5 |
| default | Generic: main topic, key points, technical info, conclusions | 2000 (OpenAI) / 8000 (Gemini) | 0.5 |

#### YouTube Summary Format

YouTube summaries must produce a **navigable index** that also serves as a **standalone summary**:

1. **Title**: A descriptive, original title that captures the main theme and value proposition of the video. This is NOT the YouTube video title — it is a newly created title that summarizes the core thesis.
2. **Introductory paragraph**: A 2-4 sentence high-level summary providing context (who the speakers are, what's discussed, why it matters).
3. **Section header**: A label like "Structured Index" or "Episode Index" (in the content's language).
4. **Timestamped sections** organized by major topic transitions:
   - Timestamps in `MM:SS` format for videos under 1 hour, `HH:MM:SS` format for videos 1 hour or longer
   - Aim for 5-15 minute segments between timestamps
   - Descriptive section titles that capture the topic or question being discussed
   - Under each timestamp: 2-5 sentences describing what's discussed, including specific details, examples, numbers, names, and quotes from the transcript
   - Speaker attributions for important statements
   - Bullet points for specific sub-topics, tools, or references mentioned

**YouTube format invariants:**
- MUST contain at least one timestamp in `MM:SS` or `HH:MM:SS` format
- MUST start directly with the title (no preamble)
- MUST include an introductory summary paragraph after the title
- Each timestamp section MUST have a descriptive section title
- Timestamps MUST appear in chronological order
- MUST be in the same language as the video content
- Each section MUST be detailed enough to be a standalone summary of that segment

#### Article / Webpage Summary Format

```
1. **Main Topic**: What is this article about?
2. **Key Points**: 3-5 main arguments or findings
3. **Important Details**: Specific facts, figures, or examples
4. **Conclusions**: What conclusions does the author draw?
5. **Relevance**: Why is this information important or useful?
```

**Article format invariants:**
- MUST contain exactly 5 numbered sections in the specified order
- MUST start directly with `1. **Main Topic**:` (no preamble)
- Section 2 (Key Points) MUST contain 3-5 bullet points

#### Social Media Summary Format (Twitter / TikTok)

**Social media format invariants:**
- MUST be 2-3 paragraphs maximum
- MUST start directly with summary content (no preamble)
- MUST preserve all important information from the original post

**Behavioral rules (all content types):**
- Content type is determined by URL detection (see Section 14)
- If summary generation fails, the error is logged and the bookmark is created with a null/empty summary
- On re-extraction, if summary generation fails, the existing summary is preserved (null-coalescing update)
- Summary output MUST begin directly with content — never with introductory phrases
- Summary language MUST match the content language (PROP-023)

#### Path 2: Summarize-Then-Embed (Transient)

When content exceeds the embedding provider's maximum input length, it is summarized first, then the *summary* is embedded. This is a transient transformation — the original full content is stored.

| Step | Behavior |
|---|---|
| Content within provider limit | Embed directly, no summarization |
| Content exceeds provider limit | Summarize content first, then embed the summary |
| Content far exceeds limit (e.g., >100K chars) | Truncate to head+tail before summarizing |
| Summarization fails | Fall back to intelligent truncation at the last sentence boundary |

**Key invariant:** The summarize-then-embed output is **never stored**. The bookmark's `content` field always contains the original extracted content.

### 15.9 Knowledge Graph Schema

#### Node Types

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

#### Relationship Types

| Relationship | Source -> Target | Description |
|-------------|-----------------|-------------|
| `BELONGS_TO` | Bookmark -> Category | Classification |
| `ABOUT` | Bookmark -> Concept | Topic association |
| `MENTIONS` | Bookmark -> Entity | Entity reference |
| `SUB_CATEGORY_OF` | Category -> Category | Hierarchy |
| `RELATES_TO` | Any -> Any | General relationship |
| `SIMILAR_TO` | Any -> Any | Similarity |

#### Entity Extraction

- Content is sanitized based on detected type (social media vs. general)
- Content >50,000 chars is split at sentence boundary for parallel extraction
- LLM extraction uses JSON format (primary) with text format fallback
- Results are deduplicated by name across chunks
- 3 retry attempts with exponential backoff and jitter

### 15.10 Embedding Providers

#### Provider Abstraction

Each provider must implement:
- `GenerateEmbedding(text, taskType?) → vector` — generate an embedding vector
- `GenerateChatCompletion(prompt, options?) → text` — generate text via chat model

Provider configuration specifies:
- **Dimensions**: Fixed vector length (must be consistent across all bookmarks)
- **Max input length**: Character limit before content must be summarized or truncated
- **Task types**: Whether the provider supports task-type hints (document vs. query embeddings)

#### Long Content Handling

- Content exceeding the provider's max input is summarized first, then the summary is embedded
- If summarization input is very large, it is truncated using a head+tail strategy
- If summarization fails, intelligent truncation finds the last sentence boundary within the allowed range

#### Chat Model Routing

Dynamic model selection based on request characteristics:
- Large or complex tasks (JSON extraction, long inputs): route to a more capable model
- Small tasks: route to a faster/cheaper model
- On server error with the primary model: fallback to the secondary model
- On structured output failure: retry without structured output constraints

---

## 16. Reference Algorithms

### 16.1 Knowledge Graph Highlighting Algorithm

```text
function highlightTextNodes(container, terms):
  walker = createTreeWalker(container, SHOW_TEXT)
  textNodes = collectAll(walker)

  for each textNode in textNodes:
    parent = textNode.parentElement
    if parent matches "code, pre, script, style, sup, .kg-highlight":
      skip

    matches = []
    for each term in terms (sorted by name length DESC):
      if len(term.name) < 3 or len(term.name) > 50:
        skip
      regex = new RegExp("\\b" + escapeRegex(term.name) + "\\b", "gi")
      for each match of regex in textNode.textContent:
        matches.append({start, end, term})

    resolvedMatches = removeOverlaps(matches)
    // Keep longer match; if equal length: entity > concept > category

    if resolvedMatches is not empty:
      replaceTextNodeWithHighlightedFragments(textNode, resolvedMatches)
```

### 16.2 Content Chunking Algorithm

```text
function chunkContent(content, maxSize, overlap, minSize):
  if len(content) <= maxSize:
    return [singleChunk(content)]

  chunks = []
  position = 0
  while position < len(content):
    end = min(position + maxSize, len(content))

    if end < len(content):
      breakPoint = findSentenceBoundary(content, position + minSize, end)
      if breakPoint == -1:
        breakPoint = findWordBoundary(content, position + minSize, end)
      if breakPoint > position:
        end = breakPoint

    chunks.append(content[position:end])
    advance = max(end - position - overlap, minSize)
    position = position + advance

  return chunks
```

---

## 17. Failure Model and Recovery

### 17.1 Failure Classes

1. **Database failures**
   - PostgreSQL connection lost
   - Migration failures
   - Vector index corruption

2. **AI service failures**
   - Embedding generation timeout or error
   - Chat completion timeout or error
   - Token limit exceeded
   - Rate limiting

3. **Content extraction failures**
   - URL unreachable (DNS, network, HTTP errors)
   - Paywall blocking content
   - Platform-specific extractor failure
   - Content too large

4. **Graph database failures**
   - Neo4j connection lost
   - Entity extraction timeout (2-minute limit)
   - MERGE operation failure

5. **External service failures**
   - Tavily API unavailable
   - Archive.today unreachable
   - Langfuse tracing failure

### 17.2 Recovery Behavior

- **Database failures**: Fatal — server cannot start or serve requests without PostgreSQL
- **AI service failures**: Job retried with exponential backoff (up to 3 attempts). Non-retryable errors fail the job immediately.
- **Content extraction failures**: Retryable network errors are retried. Paywall detection triggers archive fallback. Platform extractor failures fall back to generic scraping.
- **Graph database failures**: Graph features are disabled. Core bookmark CRUD continues normally. Entity extraction failures are logged and ignored (fire-and-forget).
- **External service failures**: Tavily failure falls back to standard scraping. Archive failure skips paywall bypass. Langfuse failure disables tracing. None of these crash the server.

### 17.3 Restart Recovery

After restart:
- No in-flight job state is recovered. Jobs stuck in `processing` remain in that state.
- The server re-establishes connections to PostgreSQL and Neo4j.
- Completed and failed jobs older than 7 days are cleaned up.
- Archive cache entries older than 30 days are cleaned up.

---

## 18. Security and Safety

### 18.1 Trust Boundary

- The system is designed for **single-user, private network** deployment (localhost or Tailscale)
- All API endpoints are **unauthenticated** — no API keys, tokens, or session management
- The system trusts all incoming requests (no rate limiting, no input sanitization beyond what's needed for SQL/graph safety)
- Deploying on a public network without additional authentication is a security violation (SAFE-001)

### 18.2 Secret Handling

- API tokens are read from environment variables (never from config files checked into source control)
- Tokens are never logged or included in API responses (SAFE-002)
- Langfuse tracing excludes raw API keys from trace data
- The `.env` file should have restricted file permissions

### 18.3 Filesystem Safety

- The static web UI is served from a fixed directory path (`./web/static`)
- No user-controlled file paths are used in filesystem operations
- Prompt logging writes to a fixed directory when enabled

---

## 19. Observability

### 19.1 Logging

The server logs to stdout using Go's standard `log` package:
- All HTTP requests are logged via middleware (method, path, duration)
- Job processing events are logged (start, complete, fail, retry)
- AI service calls are logged (model, token usage, duration)
- Graph operations are logged (entity count, extraction duration)
- Errors include stack context and relevant IDs

### 19.2 Langfuse Tracing

When `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` are configured:
- All embedding generation calls are traced with model name, dimensions, and duration
- All chat completion calls are traced with model, prompt tokens, completion tokens
- All entity extraction operations are traced
- All summary generation operations are traced
- Traces include meaningful names for debugging (e.g., "generate-embedding", "extract-entities")

### 19.3 Prompt Logging

When `PROMPT_LOGGING=true`:
- AI prompts and responses are written to timestamped files for debugging
- Files are stored in a fixed directory
- Useful for debugging entity extraction and summary quality

---

## 20. Evaluation Tiers

### Tier 1: Durable Evaluations (survive reimplementation)

- **Safety checks**: Verify SAFE-001, SAFE-002 hold
- **Invariant checks**: Verify INV-001 through INV-015 hold after operations
- **Property-based tests**: Verify PROP-001 through PROP-023 with generated inputs
- **State machine tests**: Verify job lifecycle transitions (valid and invalid)
- **Contract conformance**: Verify all interface schemas (request/response shapes, status codes)
- **End-to-end behavioral checks**: URL -> bookmark -> searchable -> deletable lifecycle
- **Boundary tests**: Chunking thresholds, embedding dimension limits, pagination bounds, rendering pipeline stages

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

## 21. Validation Profiles

### 21.1 Core Conformance

Deterministic tests required for all conforming implementations. No external dependencies needed.

- Content type detection (Section 14)
- RRF score calculation
- Query classification
- Content chunking
- Citation extraction and formatting
- Highlight overlap resolution
- Markdown auto-detection
- Snippet generation

### 21.2 Extension Conformance

Required only for optional features that an implementation chooses to ship.

- If knowledge graph is implemented: entity extraction, graph node creation, highlight term loading
- If Tavily is implemented: enhanced scraping, JavaScript rendering fallback
- If multilingual search is implemented: Spanish full-text search, RRF path selection

### 21.3 Real Integration Profile

Environment-dependent checks recommended before production use.

- Embedding generation with real API keys (OpenAI or Gemini)
- Content scraping of live URLs
- Neo4j entity extraction round-trip
- Archive.today availability check
- Langfuse trace submission
- A skipped real-integration test should be reported as skipped, not silently passed.

---

## 22. Testing

### 22.1 Test data format

Tests are defined in `tests.yaml` as language-agnostic evaluations organized by durability tier and validation profile.

### 22.2 Input field mapping

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

### 22.3 Error test handling

For entries with `error: true`, assert the function raises/returns an error or the HTTP endpoint returns a 4xx/5xx status.

---

## 23. Live Evaluation Criteria

### 23.1 Operational Metrics

| Metric | Acceptable Range | Alert Threshold |
|--------|-----------------|-----------------|
| POST /add p99 latency | < 500ms (just job creation) | > 2s |
| GET /answer time-to-first-byte | < 3s | > 10s |
| GET /bookmarks p99 latency | < 1s | > 5s |
| Job processing time (median) | < 30s | > 120s |
| Job queue depth | < 50 | > 80 |
| Job failure rate | < 5% | > 15% |

### 23.2 Business Invariants (monitored continuously)

- **INV-001**: No bookmarks exist with null or empty `content`
- **INV-002**: No duplicate URLs exist among bookmarks (excluding null URLs)
- **INV-005**: For every chunked bookmark, `chunk_count` matches the actual number of chunk records
- **INV-007**: No jobs exist with a status outside the allowed set (`pending`, `processing`, `completed`, `failed`)

### 23.3 Cost Metrics

| Metric | Baseline | Alert Threshold |
|--------|----------|-----------------|
| Embedding tokens per bookmark | ~2-5K tokens (varies by provider) | > 10K tokens |
| Chat tokens per entity extraction | ~5K tokens | > 20K tokens |
| Chat tokens per summary | ~1K tokens | > 5K tokens |
| Archive API calls per bookmark | 0-1 (only for YouTube/paywalled) | > 3 |

---

## 24. Server Configuration

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

## 25. Web UI

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

**Answer streaming and rendering:**
- Opens SSE connection to `GET /answer?q=...`
- Text chunks appended in real-time, processed through the full Answer Rendering Pipeline (Section 15.3)
- Pulse skeleton shown while waiting for first chunk

**Knowledge graph highlighting:**
- Applied after streaming completes via the DOM-based highlighting algorithm (Section 15.5)

**Citation display:**
- "Sources" section appears below the answer with citation cards
- See Section 15.3 Stage 2 for citation card structure and interaction behavior
- Citations are lazy-loaded via IntersectionObserver with 100px rootMargin
- Only citations actually referenced as `[N]` in the answer text are shown (INV-014)

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
6. **Summary section:** Rendered via markdown auto-detection (see Section 15.4)
7. **Content preview:** Rendered via markdown auto-detection, truncated to 1500 chars with "Show Full Content" / "Show Less" toggle (see Section 15.4)
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

- **UI-INV-001**: The Bookmarks view always shows bookmarks ordered by creation date descending (newest first) unless a search query is active, in which case results are ordered by relevance score descending.

- **UI-INV-002**: A processing placeholder is always visible in the bookmark list while a job is in `pending` or `processing` state. The placeholder is replaced with the real bookmark card upon job completion.

- **UI-INV-003**: Citation numbers in the answer text always correspond to citation cards in the Sources section. Only actually-cited sources are displayed.

- **UI-INV-004**: The notification toast auto-dismisses after 5 seconds. Multiple notifications can be shown simultaneously.

- **UI-INV-005**: Knowledge graph highlighted terms in answers are never applied inside code blocks, `<pre>`, `<script>`, `<style>`, or superscript (`<sup>`) elements.

- **UI-INV-006**: Autocomplete dropdown closes on: blur (200ms delay), Escape key, or selecting a result. It never remains open when the search input loses focus.

---

## 26. Chrome Extension

The Chrome extension provides a minimal browser-integrated bookmark saving experience. It operates in "fast mode" by default — a single-click save that fires and forgets.

### Popup (Fast Mode — Default)

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

### Popup (Analysis Mode — Not yet specced)

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

- **EXT-INV-001**: The fast mode popup always closes immediately after sending the save message, regardless of success or failure. *Rationale: fire-and-forget UX — the user should never wait.*

- **EXT-INV-002**: Non-HTTP/HTTPS URLs (chrome://, file://, etc.) cannot be saved. The save button is disabled with an error message. *Rationale: only web content can be scraped and embedded.*

- **EXT-INV-003**: The keyboard shortcut `Cmd/Ctrl+Shift+B` triggers a save from any page, via either the Chrome commands API or the content script fallback. *Rationale: reliability — at least one path will work.*

---

## 27. Regeneration Confidence Checklist

- [x] Problem statement and non-goals are explicit (Sections 1-2)
- [x] All domain entities have precise field definitions (Section 5)
- [x] All system invariants are explicit — INV-001 through INV-015 (Section 6)
- [x] All safety invariants are formally stated — SAFE-001, SAFE-002 (Section 6.1)
- [x] All behavioral properties are formally stated — PROP-001 through PROP-023 (Section 7)
- [x] Job lifecycle state machine has explicit states, transitions, and triggers (Section 8)
- [x] All interface contracts have precise schemas — 23 endpoints documented (Section 9)
- [x] All functions have unambiguous behavior tables or pseudocode (Section 15)
- [x] All boundary conditions have exact threshold values (Section 10.3)
- [x] All error conditions are documented (Section 13)
- [x] All configuration fields have defaults and validation rules (Section 10)
- [x] Failure model covers all failure classes with recovery behaviors (Section 17)
- [x] Property-based tests cover function composition behaviors (Section 7)
- [x] State machine tests cover valid transitions (Section 8)
- [x] Live evaluation criteria would catch drift after regeneration (Section 23)
- [x] No critical behavior exists only as implicit knowledge
- [x] Answer rendering pipeline fully specified (Section 15.3)
- [x] Knowledge graph highlighting algorithm, color system, and tooltip behavior documented (Section 15.5)
- [x] Content display formatting rules documented for all view contexts (Section 15.4)

---

## 28. Implementation Checklist

### 28.1 Core Conformance (required)

- [ ] All domain model entities implemented (Section 5)
- [ ] All API endpoints implemented with correct request/response schemas (Section 9)
- [ ] All tests.yaml durable evaluations pass
- [ ] All property-based tests pass with generative framework
- [ ] All invariant checks pass
- [ ] All safety invariant checks pass
- [ ] Job lifecycle state machine implemented with correct transitions
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
- [ ] All configuration fields with defaults and validation
- [ ] Structured logging with request context

### 28.2 Extension Conformance (if applicable)

- [ ] Knowledge graph features work when Neo4j is available
- [ ] Tavily enhanced scraping works when configured
- [ ] Multilingual search works for Spanish content
- [ ] Langfuse tracing works when configured

### 28.3 Operational Readiness

- [ ] Answer rendering pipeline produces correct HTML from markdown stream
- [ ] Citation `[N]` references are transformed into clickable superscript links
- [ ] Knowledge graph term highlighting respects excluded elements (code, pre, sup)
- [ ] Highlight color system is consistent across all UI surfaces
- [ ] Content display uses markdown auto-detection with correct fallback rendering
- [ ] Live evaluation monitoring configured
- [ ] Failure recovery behaviors verified

---

## Appendix A. Original Implementation Reference

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

- **v0.7.0** - Full restructure to numbered section format. Added: Problem Statement and Non-Goals (Sections 1-2), Core Domain Model with entity field definitions (Section 5), Job Lifecycle State Machine (Section 8), Configuration Specification with full environment variable enumeration (Section 10), Failure Model and Recovery (Section 17), Security and Safety (Section 18), Observability (Section 19), Validation Profiles (Section 21), Reference Algorithms (Section 16). New endpoints: Analyze URL, Answer Recent, Graph Explore, Graph Search, Graph Index All, Knowledge Terms, API Documentation. Safety invariants SAFE-001 and SAFE-002 added. 23 total interface contracts documented (up from 14).
- **v0.6.1** - Updated YouTube Summary Format to match actual codebase output: added introductory summary paragraph, adaptive timestamp format (MM:SS / HH:MM:SS), rich per-section descriptions instead of sparse bullets, removed bracket requirement for section headers, added language-matching requirement. Added PROP-023 for summary language matching. Clarified provider-specific token budgets (OpenAI 4000 vs Gemini 8000). Updated tests.yaml to reflect richer YouTube summary format.
- **v0.6.0** - Added Answer Rendering Pipeline section (markdown rendering, citation transformation with scroll-to-card + ring animation, rendering stages). Added Knowledge Graph Term Highlighting section (TreeWalker algorithm, color system, overlap resolution, tooltip behavior, accessibility). Added Content Display Formatting section (markdown auto-detection patterns, rendering by view context, plain-text fallback, snippet generation). Added INV-013 through INV-015 for highlighting and citation invariants. Added PROP-017 through PROP-022 for rendering pipeline properties. Added tests for extractCitedNumbers, formatAnswer, KG highlighting, markdown detection, and content rendering.
- **v0.5.0** - Added precise summary output format templates (YouTube timestamped index, article 5-section structure, social media 2-3 paragraph), added Answer Generation section with intent-specific system prompts, context template format, result filtering rules, and search strategy parameters. Added PROP-013 through PROP-016 for summary format and prompt behavior. Added SummaryGeneration and AnswerGeneration test sections.
- **v0.4.0** - Clarified single-user auth model, documented search path switching (language detection), standardized Job Status to snake_case, added metadata schema per content type, documented snippet generation, clarified content cleaning scope (archive-only), documented re-extract full regeneration scope, documented content-type-specific summary prompts, clarified total sentinel -2 threshold, documented nodes[] graph filtering mechanism, documented related bookmarks score field, marked Analysis Mode as not yet specced
- **v0.3.2** - Added Summary Generation section with content-type-aware strategies and summarize-then-embed path
- **v0.3.1** - Added missing contracts (related bookmarks, categories), defined RecentBookmark and autocomplete schemas, fixed chunking threshold inconsistency, added notes title resolution
- **v0.3.0** - Abstracted technology references; added Original Implementation Reference section
- **v0.2.0** - Added Web UI and Chrome Extension specification with UI invariants
- **v0.1.0** - Initial specification extracted from existing codebase
