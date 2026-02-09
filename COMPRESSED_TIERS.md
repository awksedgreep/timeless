# Compressed Tier Segments — Design Document

## Problem

Raw segments use gorilla + zstd compression achieving ~0.67 bytes/point.
Tier tables (hourly/daily/monthly rollups) store plain SQLite rows at ~78 bytes/row.
This means tier data is **116x less space-efficient** than raw data and becomes the
dominant storage cost at scale.

### Storage projections (10K devices × 100 metrics = 1M series)

| Data | 30-day retention | 90-day retention |
|------|-----------------|-----------------|
| Raw segments (compressed) | ~6 GB | ~17 GB |
| Hourly tier (uncompressed) | ~52 GB | ~156 GB |
| Daily tier (uncompressed) | ~7 GB | ~20 GB |

The hourly tier alone is 3-9x larger than all raw data.

### Storage projections (100K devices × 100 metrics = 10M series)

| Data | 30-day retention | 90-day retention |
|------|-----------------|-----------------|
| Raw segments (compressed) | ~60 GB | ~170 GB |
| Hourly tier (uncompressed) | ~520 GB | ~1.5 TB |

At 100K devices the hourly tier is untenable without compression.

## Solution: Tier Segments (Compressed Chunks)

Mirror the raw segment approach: instead of one SQLite row per (series_id, bucket),
store a compressed blob containing N buckets of aggregate data per series.

### Expected compression

Each chunk: N buckets × 56 bytes (7 values × 8 bytes) = N×56 bytes uncompressed.
With zstd on numeric data: ~5-10x compression.

| Tier | Chunk size | Uncompressed | Compressed (est.) |
|------|-----------|-------------|-------------------|
| Hourly | 24h (24 buckets) | 1,344 bytes | ~200-400 bytes |
| Daily | 30d (30 buckets) | 1,680 bytes | ~250-500 bytes |
| Monthly | 12mo (12 buckets) | 576 bytes | ~100-200 bytes |

### Projected savings at 1M series / 30-day hourly

- Current: ~52 GB
- Compressed (7x avg): ~7.5 GB
- Comparable to 90 days of raw data

## Schema Changes

### Tier DSL

```elixir
defmodule MyApp.MetricSchema do
  use Timeless.Schema

  raw_retention {90, :days}

  tier :hourly,
    resolution: :hour,
    chunk: {24, :hours},        # 24 hourly buckets per chunk
    aggregates: [:avg, :min, :max, :count, :sum, :last],
    retention: {30, :days}

  tier :daily,
    resolution: :day,
    chunk: {30, :days},         # 30 daily buckets per chunk
    aggregates: [:avg, :min, :max, :count, :sum, :last],
    retention: {365, :days}

  tier :monthly,
    resolution: {30, :days},
    chunk: {12, :months},       # 12 monthly buckets per chunk
    aggregates: [:avg, :min, :max, :count, :sum],
    retention: :forever
end
```

Default chunk sizes if not specified:
- Hourly → 24 hours (1 day)
- Daily → 30 days (1 month)
- Monthly → 12 months (1 year)

### SQLite Table Schema

```sql
-- Old (one row per bucket):
CREATE TABLE tier_hourly (
  series_id   INTEGER NOT NULL,
  bucket      INTEGER NOT NULL,
  avg         REAL,
  min         REAL,
  max         REAL,
  count       INTEGER,
  sum         REAL,
  last        REAL,
  PRIMARY KEY (series_id, bucket)
) WITHOUT ROWID

-- New (one row per chunk):
CREATE TABLE tier_hourly (
  series_id    INTEGER NOT NULL,
  chunk_start  INTEGER NOT NULL,
  chunk_end    INTEGER NOT NULL,
  bucket_count INTEGER NOT NULL,
  data         BLOB NOT NULL,
  PRIMARY KEY (series_id, chunk_start)
) WITHOUT ROWID
```

## Binary Chunk Format

Pragmatic approach: packed binary + zstd. No GorillaStream changes needed.

### Encoding

```
Header:
  resolution_seconds : uint32 (4 bytes)
  aggregate_count    : uint8  (1 byte) — number of aggregate fields
  bucket_count       : uint16 (2 bytes)

Body (repeated bucket_count times):
  bucket_timestamp   : int64  (8 bytes)
  avg                : float64 (8 bytes)
  min                : float64 (8 bytes)
  max                : float64 (8 bytes)
  count              : int64  (8 bytes)
  sum                : float64 (8 bytes)
  last               : float64 (8 bytes)  [if in aggregates list]

Total per bucket: 7 + aggregate_count × 8 bytes header + N × (8 + agg_count × 8) bytes
```

The entire binary is then zstd-compressed. The header allows future changes
to aggregate lists without breaking existing data.

### Decoding

```elixir
def decode_chunk(blob) do
  data = :ezstd.decompress(blob)
  <<resolution::32, agg_count::8, bucket_count::16, rest::binary>> = data
  decode_buckets(rest, agg_count, bucket_count, [])
end
```

### Why not gorilla compression?

GorillaStream handles (timestamp, float) pairs — single-valued. Tier data has
6-7 values per bucket. Options considered:

1. **Parallel gorilla streams** — one per aggregate. Best compression but requires
   GorillaStream changes or 6 separate compress/decompress calls per chunk.
   Could revisit as an optimization later.

2. **Packed binary + zstd** (chosen) — simple, uses existing ezstd dep, ~5-10x
   compression on numeric data. Good enough for v1, can upgrade later.

3. **Custom delta + XOR encoding** — most complex, best compression. Not worth
   the engineering cost for v1.

## Implementation Phases

### Phase 1: Chunk Codec Module

**File:** `lib/timeless/tier_chunk.ex`

- `encode(buckets, aggregates) :: binary` — takes list of bucket maps, returns zstd blob
- `decode(blob) :: [bucket_map]` — decompresses and unpacks
- `merge(existing_blob, new_buckets) :: binary` — read-modify-write for partial chunks
- Comprehensive tests with round-trip verification
- Bench: encode/decode speed, compression ratios at various chunk sizes

**No other files changed.** Pure codec module with tests.

### Phase 2: Schema DSL + Table Creation

**Files:** `lib/timeless/schema.ex`, `lib/timeless/segment_builder.ex`

- Add `:chunk` option to tier macro (duration tuple like retention)
- Add `chunk_seconds` field to `Timeless.Schema.Tier` struct
- Default chunk sizes: hourly→24h, daily→30d, monthly→12mo
- `create_tier_tables/2` creates new compressed table schema
- Handle migration: detect old schema (has `avg` column), read all rows,
  compress into chunks, recreate table. Run once on startup.

**Tests:** Schema DSL accepts chunk option, table creation with new schema.

### Phase 3: Rollup Writes Compressed Chunks

**Files:** `lib/timeless/rollup.ex`

Current flow per tier:
1. Read raw/source data since watermark
2. Aggregate into buckets
3. INSERT OR REPLACE individual rows
4. Advance watermark

New flow:
1. Read raw/source data since watermark (unchanged)
2. Aggregate into buckets (unchanged)
3. Group buckets by chunk boundary:
   `chunk_start = div(bucket, chunk_seconds) * chunk_seconds`
4. For each chunk:
   a. Read existing chunk blob from DB (may be nil for new chunk)
   b. Decode existing buckets
   c. Merge new buckets (new values overwrite existing for same bucket)
   d. Encode merged result
   e. INSERT OR REPLACE compressed chunk
5. Advance watermark (unchanged)

**Key concern:** Read-modify-write for partial chunks. This adds one SELECT
per chunk per rollup cycle. For hourly tier with 24h chunks, the current
chunk is updated once per rollup cycle (e.g., every 5 minutes). That's
24 updates to the same chunk over 24 hours before it's sealed. Acceptable.

**Tests:** Rollup produces compressed chunks, partial chunk updates work,
watermark semantics unchanged.

### Phase 4: Query Reads Compressed Chunks

**Files:** `lib/timeless/query.ex`

Current flow:
```sql
SELECT bucket, avg, min, max, count, sum, last
FROM tier_hourly
WHERE series_id = ?1 AND bucket >= ?2 AND bucket < ?3
```

New flow:
```sql
SELECT data, chunk_start, chunk_end
FROM tier_hourly
WHERE series_id = ?1 AND chunk_end > ?2 AND chunk_start < ?3
```

Then:
1. Decompress each chunk
2. Filter buckets to requested [from, to) range
3. Return filtered buckets

**Re-aggregation** (query bucket > tier bucket) works the same — it operates
on the decoded bucket list, not the storage format.

**Stitch logic** (tier + raw hybrid queries) unchanged — operates on
bucket-level data after decoding.

**Tests:** Query returns correct results from compressed chunks, range
filtering works, re-aggregation works, hybrid tier+raw stitching works.

### Phase 5: Retention on Chunk Boundaries

**Files:** `lib/timeless/retention.ex`

Current:
```sql
DELETE FROM tier_hourly WHERE bucket < ?1
```

New:
```sql
DELETE FROM tier_hourly WHERE chunk_end < ?1
```

This means retention granularity is chunk-sized. With 24h hourly chunks,
the oldest full day is dropped at a time. Acceptable — retention doesn't
need hour-level precision.

**Edge case:** A chunk that straddles the cutoff (some buckets expired,
some not). Two options:
- **Simple:** Only delete fully expired chunks. Slight over-retention. ← prefer this
- **Complex:** Read-modify-write to trim partial chunks. Not worth it.

**Tests:** Retention deletes only fully expired chunks, partial chunks preserved.

### Phase 6: Benchmark + Validate

**Files:** `lib/mix/tasks/bench_retention.ex` (update), new bench script

- Compare storage: uncompressed vs compressed tiers at 10K/100K series
- Compare rollup write speed (individual rows vs chunk read-modify-write)
- Compare query latency (direct row read vs chunk decompress + filter)
- Validate with existing test suite (all 185+ tests still pass)
- Bench compression ratios with realistic ISP metric data

### Phase 7: Migration Support

**Files:** `lib/timeless/segment_builder.ex`

On startup, if existing tier tables have the old schema (detected by column
presence of `avg`):
1. Read all rows from old table
2. Group by series_id and chunk boundary
3. Encode into compressed chunks
4. Create new table, insert chunks
5. Drop old table
6. Log migration stats (rows migrated, space saved)

This is a one-time operation. Existing deployments upgrade transparently.

## Backwards Compatibility

- Old schema (no `:chunk` option) → defaults to compressed with standard chunk sizes
- Migration runs automatically on first startup after upgrade
- No user action required
- The `Timeless.info/1` API returns the same data regardless of storage format
- Query API unchanged — callers don't know or care about storage format

## Open Questions

1. Should chunk size be configurable per tier or use sensible defaults only?
   **Decision: Configurable via schema DSL, with good defaults.**

2. Should we support mixed mode (some tiers compressed, some not)?
   **Decision: All tiers compressed. Simpler code, no reason not to.**

3. GorillaStream for tier data later?
   **Decision: Start with packed binary + zstd. Revisit if compression ratios
   are insufficient. The format is internal — can change without API impact.**

## Timeline

- Phase 1 (Codec): ~1 session
- Phase 2 (Schema): ~1 session
- Phase 3 (Rollup): ~1 session
- Phase 4 (Query): ~1 session
- Phase 5 (Retention): ~30 min (simple change)
- Phase 6 (Bench): ~1 session
- Phase 7 (Migration): ~1 session

Total: ~5-6 sessions, can be done incrementally. Each phase is independently
testable. No phase breaks existing functionality until Phase 3 switches
the write path.
