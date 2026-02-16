# ADR-002: Chunk-Based File Splitting Strategy

## Status
Accepted

## Context
Files contain up to 1,000,000 lines. We need to split them into manageable units for parallel processing without loading the entire file into memory.

## Decision
Split files into chunks of 10,000 lines each using Apache Camel's tokenizer with streaming mode enabled.

## Rationale
- **10,000 lines per chunk** balances parallelism and message overhead:
  - 1M lines / 10K = 100 chunks (manageable queue depth)
  - Each chunk is small enough for efficient processing
  - Large enough to amortize per-message overhead
- **Streaming mode** ensures constant memory usage regardless of file size
- **Metadata enrichment** (chunkId, totalChunks, fileName, timestamp) enables tracking and idempotency

## Alternatives Considered
- **1,000 lines/chunk**: Too many messages (1,000), excessive broker overhead
- **100,000 lines/chunk**: Too few chunks (10), limits parallelism
- **Byte-based splitting**: Risk of splitting mid-line in CSV/text formats

## Consequences
- Fixed chunk size may not be optimal for all file formats
- Chunk size is configurable via `CHUNK_SIZE` environment variable
- Last chunk may contain fewer than 10,000 lines
