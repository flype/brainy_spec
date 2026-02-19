# Brainy Spec

The durable specification for **Brainy** — a personal bookmark knowledge base that uses AI and hybrid search to help you organize, search, and discover bookmarks across platforms like YouTube, Twitter/X, Instagram, TikTok, and the web.

## Why a spec repo?

This repository contains no implementation code. On purpose.

The idea comes from Chad Fowler's [Evaluations Are the Real Codebase](https://aicoding.leaflet.pub/3mb526js42k26), which argues that in the age of AI-assisted development, **code is cheap to generate but understanding and verifying behavior are expensive**. The durable asset isn't the codebase — it's the specification that defines what the system must do.

The core diagnostic: *if deleting your codebase feels terrifying, your evaluations are insufficient.*

Brainy Spec puts that idea into practice. Everything here — the system contracts, invariants, behavioral properties, and tiered test definitions — is designed to **survive reimplementation**. The code that implements Brainy can be rewritten in any language, by any team (human or AI), and these specs remain the source of truth.

## What's in here

- **`SPEC.md`** — The full system specification: API contracts, data models, processing pipelines, search algorithms, system invariants, and behavioral properties.
- **`tests.yaml`** — A three-tier test suite (durable evaluations, ephemeral tests, live evaluations) that any implementation must satisfy.

## What Brainy does

Brainy is an async-first bookmark system with:

- **Hybrid search** combining semantic (vector embeddings) and lexical (full-text) search
- **Knowledge graph** (Neo4j) for entity extraction and discovery
- **Graceful degradation** — optional subsystems can fail without blocking core functionality
- **Multi-platform content extraction** with paywall detection and archive fallbacks
- **Streaming answers** via SSE for question-answering over your bookmarks

## The three-tier evaluation model

Following Fowler's framework, the test suite is organized by durability:

1. **Durable evaluations** — Invariant checks, property-based tests, contract conformance, and end-to-end lifecycle tests. These survive reimplementation.
2. **Ephemeral tests** — Example-based tests tied to specific implementation details. Disposable when the code changes.
3. **Live evaluations** — Continuous production monitoring for drift detection, operational metrics, and business invariants.

## Current version

`v0.1.0`
