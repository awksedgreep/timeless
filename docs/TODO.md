# Timeless — Next Steps

## Bugs & Gaps

### Alert rule retention worker
Self-cleaning state rows are done (state deleted on `ok` transition), but there's no mechanism to prune old or orphaned alert **rules**. Plan: add `TIMELESS_ALERT_RETENTION` env var and a periodic worker that deletes rules with no matching series or rules that haven't fired in N days.

### Test coverage for pending segment flush
The 60-second periodic flush of in-progress segments (`flush_pending` in SegmentBuilder) has no dedicated tests. Should verify that data becomes queryable before the 4-hour segment bucket closes.

### `info()` undercounts during flush window
`Timeless.info/1` only counts points from `raw_segments` in SQLite. Points in SegmentBuilder memory (0-60 seconds old) aren't reflected in `total_points` or `buffer_points`. Less critical now that the pending flush exists, but the dashboard can show stale counts for up to 60 seconds after ingestion.

## Hardening

### Make `pending_flush_interval` configurable
The 60-second pending flush interval defaults in SegmentBuilder but isn't wirable through the supervisor or `TIMELESS_PENDING_FLUSH_INTERVAL` env var. Should follow the same pattern as `TIMELESS_SEGMENT_DURATION`.

### Import endpoint error reporting
`POST /api/v1/import` returns HTTP 200 with `{samples: N, errors: N}` for partial failures. Consider:
- HTTP 207 Multi-Status for partial failures
- Server-side logging of parse errors (currently silent)
- Return error details in response body for debugging

## Tooling

### Historical data backfill script
The real-time load generator (`/tmp/timeless_load.sh`) is useful for live testing, but for UI development a script that backfills days/weeks of historical data with realistic patterns would be more valuable. Include device reboots (uptime resets), traffic spikes, and SNR degradation events.

## Future

### timeless_ui
Separate Phoenix LiveView package for admin/monitoring. Requirements captured in `docs/timeless_ui_requirements.md`. Prototype started — embeds Timeless as a dependency, runs on port 4001.

### Clean up stale plan file
The bearer token auth plan at `.claude/plans/quiet-watching-cocoa.md` is fully implemented and can be deleted.
