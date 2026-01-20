# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the X (Twitter) "For You" feed recommendation algorithm. It retrieves, ranks, and filters posts from in-network (followed accounts) and out-of-network (ML-discovered) sources, using a Grok-based transformer model for scoring.

## Architecture

The system has four main components:

- **home-mixer/** - Orchestration layer that assembles the For You feed (Rust)
- **thunder/** - In-memory post store and realtime ingestion for in-network content (Rust)
- **phoenix/** - ML component for retrieval (two-tower model) and ranking (Grok transformer) (Python/JAX)
- **candidate-pipeline/** - Reusable framework for building recommendation pipelines (Rust)

## Tech Stack

**Rust Services:**
- home-mixer, thunder, candidate-pipeline
- Uses async/await with `tonic::async_trait`
- gRPC for service communication

**Python ML (phoenix/):**
- Python 3.11+
- JAX 0.8.1 with dm-haiku for neural networks
- Package manager: uv
- Linter: ruff (line-length=100, indent-width=4)
- Type checker: pyright
- Tests: pytest

## Common Commands

### Python (phoenix/)
```bash
cd phoenix

# Install dependencies
uv sync

# Run tests
uv run pytest

# Type check
uv run pyright

# Lint
uv run ruff check .

# Run ranker inference
uv run python run_ranker.py

# Run retrieval inference
uv run python run_retrieval.py
```

## Code Structure

### Pipeline Pattern (candidate-pipeline)
The recommendation pipeline follows a standard pattern with these traits:
- `Source` - Fetch candidates from a data source
- `Hydrator` - Enrich candidates with additional features
- `Filter` - Remove candidates that shouldn't be shown
- `Scorer` - Compute scores for ranking
- `Selector` - Sort and select top candidates
- `SideEffect` - Run async side effects (caching, logging)

### Filter Naming Convention
Filters in `home-mixer/filters/` follow the pattern `*_filter.rs`:
- Pre-scoring: `drop_duplicates`, `age`, `self_tweet`, `muted_keyword`, etc.
- Post-selection: `vf_filter`, `dedup_conversation`

### Scorer Pattern
Scorers implement the `Scorer<Query, Candidate>` trait with:
- `score()` - Computes scores for a batch of candidates
- `update()` - Merges scored values back into candidates

### Phoenix ML Models
- `grok.py` - Core Grok transformer architecture
- `recsys_model.py` - Ranking model with candidate isolation
- `recsys_retrieval_model.py` - Two-tower retrieval model
- `runners.py` - Inference runners for serving

## Key Design Principles

1. **No hand-engineered features** - The Grok transformer learns relevance from user engagement sequences
2. **Candidate isolation** - During ranking, candidates cannot attend to each other, only to user context
3. **Multi-action prediction** - Model predicts P(like), P(reply), P(repost), P(block), etc., combined via weighted sum
4. **Parallel execution** - Sources and hydrators run in parallel where possible
