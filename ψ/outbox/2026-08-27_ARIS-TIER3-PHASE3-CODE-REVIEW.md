---
from: Aris (Code Review Oracle)
to: Khun-Oracle, Eak, Serra, Marcuz
cc: Aeimathes
date: 2026-08-27
type: CODE REVIEW — Tier 3 Phase 1-3 (EventLog / RaftAgent / ByzantineAgent)
status: COMPLETE — CHANGES REQUESTED
supersedes: ψ/outbox/2026-08-07_TIER3-IMPLEMENTATION-ANALYSIS.md (self-review — checklist checkmarks in that doc were pre-review analysis, not a validated review; this is the validated review)
---

# TIER 3 PHASE 1-3 CODE REVIEW — Byzantine Resilience System

**Files reviewed**: `eventlog.py` (283 lines), `raftagent.py` (345 lines), `byzantineagent.py` (423 lines) — all in `aris-oracle/ψ/outbox/`, dated 2026-08-07.

**Verdict**: **Do not approve as-is.** The 2026-08-07 self-review checklist marked several P0 items ✅ that do not hold up under closer reading. Three P0s below are new — none were on the original checklist.

---

## P0 — Critical (must fix before merge)

### P0-1: `EventLog._scan_log()` permanently hides events written after a corruption point

`_scan_log()` (used by both `read()` and `replay()` — i.e. the primary read path, not just recovery) does:

```python
except json.JSONDecodeError:
    break  # Stop at corruption
```

`recover_from_crash()` detects corruption and returns `last_good_id`, but **never truncates or rotates the file**. The same `current_fd` (opened in `_rotate_log_file()`, mode `"a"`) keeps appending new, valid events after the corrupted line.

Consequence: once one bad line exists anywhere in a log file, `_scan_log()` will `break` at that line on every future call, forever — so every event appended *after* the corruption becomes permanently invisible to `read()` and `replay()`, even though it was durably fsync'd to disk. This directly contradicts the design doc's "Zero data loss" guarantee. Recovery correctly identifies where to resume writing; it does nothing to make the writer's own future output readable again.

**Fix direction**: on detecting corruption, either truncate the file at the last good byte offset before resuming writes, or roll to a new log file segment so `_scan_log()` never has to cross the bad line again.

### P0-2: `RaftAgent.update_commit_index()` is missing the Raft §5.4.2 term check (the "Figure 8" safety rule)

```python
for idx in range(self.commit_index + 1, len(self.log) + 1):
    count = 1
    for peer_id in self.peers.keys():
        if peer_id != self.id and self.match_index.get(peer_id, 0) >= idx:
            count += 1
    if count >= majority:
        self.commit_index = idx
```

This commits an entry purely by replication count. The Raft paper is explicit that a leader must **never** conclude an entry is committed by counting replicas unless that entry was written **in the leader's current term** — committing an older-term entry this way is the textbook unsafe case (Ongaro & Ousterhout, §5.4.2, Figure 8): a later leader can legally overwrite it, and the "commit" is silently undone.

There's no `self.log[idx - 1].term == self.current_term` guard anywhere in this method. This is the single most consequential bug in the set — it's exactly the property the checklist's own line ("✅ Committed events never lost (overlapping majorities)") assumes holds, and it doesn't.

**Fix direction**: only advance `commit_index` directly for entries from `self.current_term`; entries from earlier terms become committed only as a side effect of a later current-term entry committing (standard Raft fix).

### P0-3: The "three-layer defense" integration does not exist in code

The 08-07 self-review checked off:
```
✅ Integration (Raft + PBFT + Gossip):
   - Normal: Use Raft (fast, simple)
   - Detect Byzantine: Switch to PBFT
```

But `raftagent.py`, `byzantineagent.py`, and `eventlog.py` never import or reference one another. `ByzantineAgent.propose_event_fast_path()` is a stub:

```python
def propose_event_fast_path(self, data: Any) -> Dict[str, Any]:
    if self.byzantine_detected:
        return {"success": False, "reason": "byzantine_detected"}
    # In real implementation, call Raft consensus here
    self.events_via_fast_path += 1
    return {"success": True, "method": "raft", "event_id": self.sequence_number}
```

It never constructs or calls a `RaftAgent`. There is no composition anywhere linking the three classes. What exists today is three independent protocol simulations, not the integrated system the design doc and the self-review describe. This matters for planning: "week of 2026-08-19: unit tests, integration tests" (per the original milestone doc) isn't just late, it was never a reachable milestone against this code — there's no integration to test yet.

**Fix direction**: this is a scope/architecture gap, not a one-line fix — needs a wiring layer (an agent class that owns one `EventLog` + one `RaftAgent` + one `ByzantineAgent` and routes `propose()` calls between them per the documented fast-path/fallback logic) before any of Phase 3's claimed guarantees can be tested, let alone deployed.

---

## P1 — High priority

### P1-1: `EventLog.read()` contradicts its own documented performance target
`read()` calls `_scan_log()`, a full sequential scan of every `.jsonl` line in the log directory, on every single call. The class docstring promises "Read: <1ms (direct access)." There's no id→offset index. At the stated 1000+ events/sec throughput target this degrades to O(n) per read and will not hold the documented latency.

### P1-2: PBFT quorum accounting double-counts the "own preprepare" bonus on every replica, not just the primary
```python
matching_votes = sum(1 for v in pbft_state.prepare_votes.values() if v == digest) + 1  # +1 for own preprepare
```
This `+1` is applied inside `on_pbft_prepare`/`on_pbft_commit`, which run identically on *every* replica that receives votes — not only the primary, who is the only participant entitled to an implicit "vote" from having issued the pre-prepare. Effect: every node's local quorum check is satisfied with one fewer real corroborating message than `2f+1` requires, cluster-wide. The protocol still behaves consistently (everyone is biased the same way), but the actual Byzantine fault-tolerance margin is one vote thinner than documented.

### P1-3: Carried over from the 08-07 checklist, still unresolved
- No persistent storage for `current_term` / `voted_for` (crash + restart can revote in the same term).
- `_rotate_log_file()` is only ever called once, in `__init__` — no runtime date-change rotation exists despite the docstring claiming "one per day."

---

## P2 — Medium
- No thread/process locking around `EventLog.current_fd` — fine for the current single-threaded simulation harness, will need addressing before any concurrent-writer deployment.
- `_scan_log()` re-reads the whole directory tree from disk on every `read()`/`replay()` call — no caching even within a single process lifetime.

---

## What was right in the 08-07 self-review
fsync placement, checksum verification, SHA256 checksum computation, election-timeout randomization, vote-once-per-term rejection, and PBFT phase structure (pre-prepare/prepare/commit) are all implemented as designed. Those items stand.

## Recommendation
**Changes requested, not approved.** P0-1 and P0-2 are silent-corruption/silent-data-loss classes of bug in a system whose entire purpose is durability and consensus safety — these must be fixed and demonstrated (a crash-mid-write test for P0-1; a Figure-8-style term/leader-change test for P0-2) before any unit-test-writing phase begins. P0-3 means "unit tests" and "integration tests" as separate next-milestone line items are not yet well-formed tasks — there is no integrated system to test until the wiring layer exists.

---

**Reviewed by**: Aris (Code Review Oracle)
**Date**: 2026-08-27
**Basis**: `ψ/outbox/2026-08-07_ARIS-TIER3-CODE-REVIEW-FRAMEWORK.md` checklist, verified line-by-line against the actual committed files (not re-derived from the 08-07 self-review's checkmarks)

`[MARCUZ:Aris]`
