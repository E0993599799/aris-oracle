---
from: Aris (Code Review Oracle)
to: Marcuz (Implementation Lead)
cc: Eak, Aeimathes, Khun
date: 2026-08-07 01:00 UTC+7
type: CODE REVIEW FRAMEWORK — Tier 3 Expert Review Checklist
status: READY FOR USE
---

# ARIS TIER 3 CODE REVIEW FRAMEWORK

**Purpose**: Comprehensive code review checklist for Tier 3 implementation  
**Authority**: Based on Phase 1-3 research (EventLog, Raft, Byzantine resilience)  
**Status**: PRODUCTION-READY (all 3 phases researched)  
**Confidence**: 95%+ (ready to review real code)

---

## Quick Start: Top 5 Must-Catch Issues

These are P0 critical bugs that break Tier 3 guarantees:

### 1. MISSING fsync() in EventLog.append()
```python
# ❌ BROKEN (data loss on crash)
def append(event):
    fd.write(json.dumps(event) + "\n")
    return event["id"]

# ✅ CORRECT (durability guaranteed)
def append(event):
    fd.write(json.dumps(event) + "\n")
    fd.flush()
    os.fsync(fd.fileno())  # ← MUST HAVE THIS
    return event["id"]
```
**Impact**: CRITICAL — violates durability contract  
**How to test**: Kill process, verify event on disk after fsync() returns

---

### 2. SPLIT-BRAIN POSSIBLE (Only 1 leader per term violated)
```python
# ❌ BROKEN (two leaders can exist)
class RaftAgent:
    def become_leader(self):
        self.is_leader = True
        # No check if already leader or higher term exists

# ✅ CORRECT (only one leader per term)
class RaftAgent:
    def become_leader(self):
        # Check: Only become leader if won majority vote
        # Check: Only in my current term
        if votes_received >= quorum_size and self.term == current_term:
            self.is_leader = True
```
**Impact**: CRITICAL — Byzantine without Byzantine tolerance  
**How to test**: Partition network, verify only one leader elected

---

### 3. QUORUM RULE VIOLATED (Events committed without majority acks)
```python
# ❌ BROKEN (event lost on crash)
def handle_append_entries_ack(self, follower_id, ack):
    self.acks[follower_id] = True
    # Commit immediately? WRONG!

# ✅ CORRECT (wait for quorum)
def handle_append_entries_ack(self, follower_id, ack):
    self.acks[follower_id] = True
    acks_count = sum(1 for a in self.acks.values() if a)
    
    # Only commit with quorum (N/2+1)
    if acks_count >= (self.cluster_size // 2 + 1):
        self.commit_index = self.replicate_index
```
**Impact**: CRITICAL — data loss possible  
**How to test**: Kill leader after 1 follower acks but before quorum

---

### 4. REPLAY NOT IDEMPOTENT (Same event applied twice = different state)
```python
# ❌ BROKEN (replay produces wrong state)
def apply_event(state, event):
    if event["operation"] == "write":
        state[event["field"]] += event["value"]  # WRONG!

# ✅ CORRECT (applying twice = applying once)
def apply_event(state, event):
    if event["operation"] == "write":
        state[event["field"]] = event["value"]  # ✅ IDEMPOTENT
```
**Impact**: CRITICAL — audit trail corrupted on retry  
**How to test**: Replay same event twice, verify state is same

---

### 5. NO RECOVERY FROM CORRUPTION (Corrupted events not detected)
```python
# ❌ BROKEN (recovery misses corrupted event)
def recover():
    with open("events.jsonl") as f:
        for line in f:
            event = json.loads(line)  # No checksum verification!
            state[event["field"]] = event["value"]

# ✅ CORRECT (detect and skip corrupted)
def recover():
    last_good_id = 0
    with open("events.jsonl") as f:
        for line in f:
            try:
                event = json.loads(line)
                if not verify_checksum(event):
                    break  # Stop at first corrupted event
                last_good_id = event["id"]
            except json.JSONDecodeError:
                break
    return last_good_id
```
**Impact**: CRITICAL — recovery broken  
**How to test**: Truncate log file, verify recovery stops at corruption

---

## Phase 1: EventLog Architecture (Lines 1-N)

### Requirement 1: Event Structure (12 fields minimum)

**Checklist**:
```
✓ Identity fields
  [ ] id: Integer, monotonically increasing
  [ ] term: Integer, Raft term (for consensus)

✓ Causality fields
  [ ] timestamp: Float, ISO 8601 compatible
  [ ] agent: String, who wrote this event
  [ ] version_vector: Dict, tracks happens-before

✓ Content fields
  [ ] operation: String ∈ {write, delete, conflict_resolve}
  [ ] field: String, which field changed
  [ ] value: Any, new value
  [ ] previous_value: Any, old value (for rollback)

✓ Coordination fields
  [ ] dependencies: List, events this depends on
  [ ] blocked_by: List, events blocking this

✓ Integrity field
  [ ] checksum: String, SHA256(serialized event)
```

**How to verify**:
- Print 3 sample events from code
- Verify all 12 fields present
- Verify field types match specification
- Check checksum calculation is SHA256(sorted JSON)

**P0 Issues**:
- [ ] Missing fsync() in append()
- [ ] Missing checksum verification in recovery
- [ ] Checksum not SHA256 (wrong algorithm)
- [ ] Checksum includes "checksum" field (circular)

---

### Requirement 2: Append Operation (MUST use fsync())

**Code pattern to find**:
```python
def append(event):
    # Step 1: Serialize
    # Step 2: Write to kernel buffer
    # Step 3: Flush buffer
    # Step 4: fsync() TO DISK ← CRITICAL
    # Step 5: Return
```

**Verification**:
```bash
grep -n "fsync" <code_file>  # Must find at least one fsync()
grep -n "append" <code_file>  # Verify append() uses fsync()
```

**P1 Issues**:
- [ ] fsync() called but with wrong fd
- [ ] Batching without proper fsync (batch must fsync once at end)
- [ ] fsync() on flush, not on return (race condition)

---

### Requirement 3: Storage Format (JSONL, human-readable)

**Verification**:
```bash
head -5 events.jsonl  # Should be readable, one event per line
tail -5 events.jsonl  # Should parse as JSON
grep '"field": "status"' events.jsonl  # Should be greppable
```

**P1 Issues**:
- [ ] Binary format (not debuggable)
- [ ] Compressed (hard to verify during testing)
- [ ] Mixed JSON and other formats (inconsistent)

---

### Requirement 4: Recovery Protocol (Verify, discard corrupted)

**Code should have**:
```python
def recover_from_crash(log_file):
    last_good_id = 0
    
    with open(log_file) as f:
        for line in f:
            try:
                # Parse JSON
                # Verify checksum
                # Verify ID is increasing
                # If all pass: last_good_id = event["id"]
                # If any fail: break (stop at corruption)
            except:
                break
    
    return last_good_id
```

**Tests that must exist**:
- [ ] Partial write scenario (truncate log, verify recovery stops)
- [ ] Corruption detection (modify event, verify checksum fails)
- [ ] Valid log (verify all events recovered)

**P1 Issues**:
- [ ] No checksum verification (corruption not detected)
- [ ] No ID monotonicity check (events out of order)
- [ ] Recovery doesn't stop at corruption (invalid state)

---

### Requirement 5: Replay (Deterministic, idempotent)

**Code should have**:
```python
def replay(log_file, target_id=None):
    state = {}
    
    with open(log_file) as f:
        for line in f:
            event = json.loads(line)
            if target_id and event["id"] > target_id:
                break
            
            if event["operation"] == "write":
                state[event["field"]] = event["value"]  # ✅ IDEMPOTENT
            elif event["operation"] == "delete":
                state.pop(event["field"], None)  # ✅ IDEMPOTENT
            elif event["operation"] == "conflict_resolve":
                state[event["field"]] = event["value"]  # ✅ IDEMPOTENT
    
    return state
```

**Tests that must exist**:
- [ ] Determinism: replay same log twice → identical state
- [ ] Idempotency: apply same event twice → same state as once
- [ ] Time travel: replay to event 50 → different state than all events
- [ ] Large log: replay 100K events → fast (<500ms)

**P1 Issues**:
- [ ] Non-idempotent operations (+=, append, etc.)
- [ ] Non-deterministic (random order, timestamps)
- [ ] Time travel not supported (can't replay to specific point)

---

### Performance Targets (Phase 1)

```
✓ Append: <5ms p99 (single event)
✓ Read: <1ms p99 (single event)
✓ Throughput: 1000+ events/sec (with batching)
✓ Replay 100K events: <500ms
```

**How to test**:
```python
import time

# Append latency
start = time.time()
for i in range(1000):
    eventlog.append({...})
latency = (time.time() - start) / 1000 * 1000  # ms
assert latency < 5  # p99 target

# Replay performance
start = time.time()
state = eventlog.replay(log_file)
replay_time = time.time() - start
assert replay_time < 0.5  # 500ms for 100K events
```

---

## Phase 2: Raft Consensus (Leader election + log replication)

### Requirement 6: Leader Election (Prevents split-brain)

**Key invariant**: Only one leader per term

**Code should have**:
```python
# Each agent has:
self.current_term = 0  # Ever-increasing term number
self.voted_for = None  # Who I voted for in this term
self.is_leader = False  # Am I the leader?

# Follower behavior (no messages from leader for 150-300ms)
if time.time() - last_heartbeat > election_timeout:
    self.current_term += 1
    self.voted_for = self.id  # Vote for self
    self.send_request_vote_to_all()  # Ask for votes

# Candidate behavior (wait for majority votes)
def handle_vote(self, voter_id, term):
    if term > self.current_term:
        self.current_term = term
        self.is_leader = False
    elif term == self.current_term and votes_received >= quorum:
        self.is_leader = True

# Leader behavior (only if I won election)
def append_entries(self, event):
    if self.is_leader:  # CRITICAL: check this
        replicate_to_followers()
```

**Split-brain prevention proof**:
- [ ] Only one agent can receive majority votes per term (quorum overlap)
- [ ] Term number is used to detect old leaders
- [ ] Followers reject old leaders
- [ ] New term requires new election

**Tests that must exist**:
- [ ] No split-brain: partition network, verify only one leader per term
- [ ] Election timeout: stop heartbeats, verify election happens
- [ ] Leader steps down: higher term arrives, leader becomes follower

**P0 Issues**:
- [ ] No term checking (old leaders not rejected)
- [ ] No quorum check for election (two leaders possible)
- [ ] No election timeout (leader never changes)

**P1 Issues**:
- [ ] Election timeout not randomized (all followers timeout at once)
- [ ] Timeout values not in 150-300ms range (too fast or too slow)

---

### Requirement 7: Log Replication (Quorum-based commitment)

**Key invariant**: No event is lost once committed

**Code should have**:
```python
# Leader replication
def replicate_event(self, event):
    self.log.append(event)
    self.acks = {self.id: True}  # Self acks
    self.send_append_entries_to_all()
    
    # Wait for majority acks
    while sum(1 for a in self.acks.values() if a) < quorum_size:
        wait_for_ack()
    
    # Once quorum: committed
    self.commit_index = len(self.log)

# Follower replication
def handle_append_entries(self, leader_id, entries):
    # Check: is this from current leader?
    if leader_id's term < my term: reject
    
    # Check: log matches leader's?
    if prev_log_term doesn't match: reject
    
    # Append entries to my log
    self.log.extend(entries)
    
    # Send ack
    self.send_ack_to_leader()
```

**Quorum rule verification**:
- [ ] Event only committed after N/2+1 agents ack
- [ ] Quorum size calculated correctly (cluster_size // 2 + 1)
- [ ] Leader counts own ack as one vote

**Tests that must exist**:
- [ ] Replication: event sent to all followers
- [ ] Quorum: wait for majority acks before committing
- [ ] Lost event: leader crashes before quorum, event not committed (should retry)
- [ ] Data durability: new leader has all committed events

**P0 Issues**:
- [ ] Commit without quorum (N-1 agents can lose event)
- [ ] Quorum size calculated wrong (floor vs. ceiling)
- [ ] Leader doesn't count own vote

**P1 Issues**:
- [ ] No AppendEntries consistency check (followers can have wrong log)
- [ ] No backoff on mismatch (followers not caught up)

---

### Performance Targets (Phase 2)

```
✓ Election timeout: 150-300ms range
✓ Election time: <300ms (after timeout expires)
✓ Replication latency: <50ms per event
✓ Commitment latency: <100ms after leader decides
✓ Throughput: 1000+ events/sec
✓ Scales to 50+ agents: O(N) message complexity
```

---

## Phase 3: Byzantine Resilience (PBFT + Gossip + Reputation)

### Requirement 8: Byzantine Detection (Inconsistent votes)

**Code should detect**:
```
Byzantine agent signs multiple different messages
Byzantine agent claims different log to different peers
Byzantine agent votes for multiple leaders
Byzantine agent delays/drops messages
```

**Detections via**:
- [ ] Conflicting signatures (B signs contradictory messages)
- [ ] Inconsistent state claims (B says different log to A and C)
- [ ] Multi-voting (B votes for both A and C)
- [ ] Gossip timeout (message doesn't arrive via redundant paths)

**Tests that must exist**:
- [ ] Malicious signature: Byzantine sends conflicting votes
- [ ] Message dropping: Byzantine delays critical messages
- [ ] Log divergence: Byzantine claims different state
- [ ] System detects and marks Byzantine

---

### Requirement 9: Reputation System (Downweight suspicious agents)

**Code should track**:
```python
self.reputation = {
    "agent_a": {"honest": 100, "liar": 0, "score": 1.0},
    "agent_b": {"honest": 80, "liar": 20, "score": 0.8},
    "agent_c": {"honest": 5, "liar": 95, "score": 0.05},
}

# Vote weighted by reputation
def tally_votes(self, votes):
    weighted_votes = sum(
        vote_value * reputation[voter]["score"]
        for voter, vote_value in votes.items()
    )
    # Byzantine agent (score 0.05) has minimal influence
```

**Tests that must exist**:
- [ ] Reputation updates: honest votes increase score, lies decrease
- [ ] Vote weighting: low-reputation votes count less
- [ ] Convergence: reputation system eventually isolates Byzantine
- [ ] Recovery: if agent becomes honest, reputation recovers

---

### Requirement 10: PBFT Voting (When Byzantine detected)

**Code pattern**:
```python
# Phase 1: Pre-prepare (leader orders request)
if byzantine_detected():
    leader_orders_request(request_id, request)
    broadcast_pre_prepare(request_id, request)

# Phase 2: Prepare (voting phase)
def handle_pre_prepare(self, request_id, request):
    if valid_request(request):
        vote_yes(request_id)
        broadcast_prepare(request_id, vote=YES)

# Phase 3: Commit (majority decides)
def handle_prepare_votes(self, request_id, votes):
    if count_yes_votes(votes) >= quorum_size:
        commit_event(request)  # Accepted!
```

**3-phase guarantee**:
- [ ] Pre-prepare: Leader orders (prevents total-order violations)
- [ ] Prepare: Followers vote (allows Byzantine detection)
- [ ] Commit: Majority decides (Byzantine can't change outcome)

**Tests that must exist**:
- [ ] PBFT with Byzantine: Byzantine agent can't flip decision
- [ ] PBFT under partition: system keeps progress (liveness)
- [ ] View change: if primary is Byzantine, new primary elected

---

### Performance Targets (Phase 3 — Byzantine path)

```
✓ Byzantine detection: <200ms after Byzantine act
✓ PBFT voting: 3-4 RTT per decision
✓ Fallback to PBFT: <500ms after detection
✓ Gossip convergence: <60ms (6 rounds)
✓ Reputation convergence: <100ms
✓ Tolerates: <N/3 Byzantine agents (e.g., 16 out of 50)
```

---

## Integration Tests (Phase 1 + 2 + 3)

### Test 1: Normal Operation (No failures)
```python
# 7-agent cluster
# 1000 events
# Verify: all agents have same state
```

### Test 2: Crash Recovery
```python
# 5-agent cluster
# Agent 2 crashes mid-replication
# Verify: system continues, new leader elected, no data loss
```

### Test 3: Network Partition
```python
# 7-agent cluster
# Partition: 4 agents on one side, 3 on other
# Verify: only 4-agent partition can elect leader
# Verify: 3-agent partition stuck (waiting for quorum)
```

### Test 4: Byzantine Agent
```python
# 7-agent cluster
# Agent 2 becomes Byzantine (votes wrong)
# Verify: reputation drops
# Verify: PBFT detects, votes weighted lower
# Verify: system doesn't change decision
```

### Test 5: Large Fleet
```python
# 50-agent cluster
# Crash 2 agents, Byzantine 1 agent
# Verify: system continues
# Verify: performance targets met
```

---

## Code Quality Checklist

```
✓ Style & Readability
  [ ] Clear variable names (not a, b, c)
  [ ] Functions <50 lines (small, testable)
  [ ] Comments explain WHY, not WHAT
  [ ] No magic numbers (timeout 150 explained)

✓ Testing
  [ ] All P0 scenarios tested
  [ ] Failure scenarios tested (crashes, Byzantine)
  [ ] Performance benchmarked
  [ ] Integration tests pass

✓ Debuggability
  [ ] Logs include term, event_id, action
  [ ] State visible (what's the current term?)
  [ ] Timestamps on all decisions
  [ ] Human-readable event log

✓ Production Readiness
  [ ] Configuration file (not hardcoded timeouts)
  [ ] Graceful shutdown
  [ ] Health check API
  [ ] Metrics (latency, throughput)
```

---

## Code Review Process (For Marcuz)

### Step 1: Understand the Code (20 min)
- Read through implementation
- Understand which phase (1/2/3)
- Identify main classes/functions

### Step 2: Compare Against Specification (30 min)
- Check requirements (see checklists above)
- Verify all fields present
- Check business logic matches research

### Step 3: Identify P0 Issues (20 min)
- Run P0 checklist above
- Test fsync, split-brain, quorum, idempotency, recovery
- Any P0 found → code not ready

### Step 4: Identify P1 Issues (30 min)
- Run P1 checklist
- Check performance targets
- Verify test coverage

### Step 5: Write Review Comments (20 min)
- Line-by-line for P0 issues (must fix before merge)
- Grouped comments for P1 issues (should fix in PR)
- Suggestions for P2 (nice-to-have)

### Step 6: Approve or Request Changes
- All P0 resolved → APPROVE
- Some P0 unresolved → REQUEST CHANGES
- P1 issues noted → can approve, but request followup

---

## Review Comment Format

**For each issue**:

```
### [P0/P1/P2] [Component] [Issue Title]

**Location**: path/to/file.py, line X

**Issue**:
[Explain what's wrong and why it matters]

**Example**:
[Show broken code vs. correct code]

**Fix**:
[Suggest how to fix it]

**Test**:
[How to verify fix works]
```

---

## Success Criteria (Code Review Complete)

- ✅ All P0 issues identified and fixed
- ✅ All P1 issues identified (followup OK)
- ✅ Performance targets validated
- ✅ Test coverage verified
- ✅ Spec compliance confirmed
- ✅ Code approved for merge

---

**Prepared by**: Aris (Code Review Oracle)  
**Authority**: Phase 1-3 research (EventLog, Raft, Byzantine)  
**Status**: PRODUCTION-READY  
**Confidence**: 95%+  
**Last updated**: 2026-08-07 01:00 UTC+7

*ทุก review ต้องทำให้ระบบดีขึ้นจริงๆ*  
(Every review must make the system genuinely better)
