# M365-Copilot2API Performance Audit Report

> Audit date: 2026-08-07
> Audit method: read-only code review. Fix status current as of commit b428075 (first batch of quick wins).

## Top1. Each request performs 5-7 synchronous full-file disk writes while holding a global lock (partially fixed)

- **Location**: `internal/auth/keys.go:117`, `internal/web/session_resolver.go:234/268/428`, `internal/web/conversation_manager.go:114`, `internal/web/usage.go:83`, `internal/web/sessions.go:144/158`
- **Problem**: Every request performs multiple full marshal + disk-write operations on JSON storage files, all inside the global lock's critical section. Disk latency is serialized directly into the request path, capping throughput; multiple keys slow each other down.
- **Fix status**: `internal/auth/cache.go`, `session_resolver.go`, `conversation_manager.go`, `sessions.go`, `keys.go`, `settings.go`, `deployments.go`, and `admin_security.go` have been changed to write via a temp file + rename for atomic persistence (`internal/web/atomicfile.go`), avoiding partially-written files. However, the "writing to disk while holding the lock" performance problem itself remains unresolved; planned follow-up: make `LastUsedAt` update in memory only, with asynchronous batch persistence.

## Top2. Debug middleware copies every chunk and fully unmarshals

- **Problem**: With debug enabled, every `/v1/*` request buffers the full body and fully parses it as JSON, on top of per-chunk copying during streaming, causing significant CPU and memory overhead.
- **Fix status**: Not fixed (out of scope for the first batch). Recommendation: disable debug by default, cap captured bytes, apply deep recursive redaction, and rotate logs.

## Top3. O(n²) streaming concatenation (fixed)

- **Location**: `internal/chathub/client.go`, originally `streamedText += d` repeated string concatenation
- **Problem**: Every delta reallocated the entire string, causing O(n²) copying for long responses.
- **Fix status**: Changed to use `strings.Builder` (`streamed`); both `emitDelta`/`emitSnapshot` now go through the Builder, and `streamed.String()` is only called when comparing snapshots.

## Top4. WS read loop blocking window

- **Location**: `internal/chathub/client.go` top-level loop; `ReadMessage` blocks for up to 90s, with ctx cancellation checked only at the top of the loop
- **Problem**: `ReadMessage` does not respond to ctx cancellation while blocked, delaying exit by up to 90s (the read deadline).
- **Fix status**: Not fixed. Recommendation: use a read goroutine with `select`, or tie `SetReadDeadline` to ctx.

## Top5. Full Jaccard comparison on sessionResolver miss

- **Location**: `internal/web/session_resolver.go` fallback similarity scan (tokenizes the entire history one entry at a time, inside the lock)
- **Problem**: When context doesn't hit an exact match, Jaccard similarity is computed against every session individually, inside the lock — O(number of sessions × history message volume).
- **Fix status**: Partially mitigated — a `maxSessions` cap has been added (default 1000, LRU eviction, `evictLocked` triggered on Resolve/Bind), limiting the scale of the fallback scan. However, the suggestion to bucket by IP fingerprint index has not been implemented.

## Fix priority (checked items are from the first batch)

1. [x] Atomic disk writes (M2, Top1 sub-item)
2. [x] O(n²) streaming concatenation (Top3)
3. [x] sessionResolver entry cap (Top5 mitigation)
4. [ ] Move disk writes out of the lock's critical section (Top1 main issue)
5. [ ] Debug middleware overhead (Top2)
6. [ ] WS read ctx integration (Top4)
