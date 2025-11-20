# Design Document: Real-Time Collaborative Web Page Editor (Lock-Based, Two-Service Architecture)

## 1. Introduction
This document describes the backend design for a web page editor supporting real-time collaborative editing for small groups.  
Goal: Support concurrent editing, per-page version history, live presence, granular permissions, offline handling, and operational readiness.

---

## 2. Assumptions
- Max 5 concurrent users per document, ~1000 total editors.  
- Network mostly stable, but offline/disconnects possible.  
- Operations on text/JSON are lightweight; images up to 5–10MB.  
- Self-hosted Redis and Postgres for state and persistence.

---

## 3. System Architecture
**Two services:**

1. **Auth Service**
   - Issues and validates JWT tokens signed with a **private key** (RS256 recommended)
   - Manages users, roles, and refresh tokens
   - Responsible for scope-limited JWTs per document

2. **Document Service**
   - Manages documents, sections, locks, edits, snapshots
   - Tracks presence and cursors
   - Handles merge logic on reconnect
   - Subscribes to Redis Pub/Sub for broadcasting changes
   - Validates JWTs using **public key** from Auth Service

**Token flow:**  
- Client authenticates → Auth Service returns signed JWT  
- Client sends JWT to Document Service with WebSocket or REST requests  
- Document Service validates JWT signature, expiration, and claims  
- Permissions enforced per document/section based on roles

---

## 4. Concurrency Model
**Model:** Lock-based section-level editing

**Justification:**
- Predictable behavior, avoids conflicts in same section  
- Low latency (<200ms typical network) using Redis locks and Pub/Sub  
- Simpler client/server complexity than OT or CRDT for small groups  
- Persistent operation log in Postgres for durability

**Workflow:**
1. User acquires lock on a section (`LOCKED`)  
2. Lock has TTL in Redis  
3. If user goes offline and TTL expires → lock enters `PREVIOUSLY_LOCKED_OFFLINE`, audit log created  
4. Other users may acquire lock and edit  
5. On reconnect, system supports:
   - Accept remote changes
   - Override remote changes
   - Manual 3-way merge

---

## 5. Lock State Machine


- `UNLOCKED`: Section available  
- `LOCKED`: Section owned by user  
- `PREVIOUSLY_LOCKED_OFFLINE`: User offline, lock released but pending changes logged

---

## 6. Persistence & Storage

**Redis:**
- Stores locks with TTL (`lock:{doc_id}:{section_id}`)
- Value JSON: `{ owner_user_id, session_id, acquired_at, lease_expires_at, version_snapshot_id }`
- Supports atomic check-and-set

**Postgres:**
- `documents` → document metadata, current version  
- `snapshots` → full document snapshots for time-travel  
- `edits` → operation log of changes  
- `locks_audit` → lock acquisition/release/expired/offline/stolen  
- `audit_logs` → append-only user actions and changes

**Images:** Stored in self-hosted object store or filesystem, metadata in Postgres

---

## 7. Offline & Reconnect Handling

- Offline lock expiry → lock released, record created in `PREVIOUSLY_LOCKED_OFFLINE`  
- Record includes: `lastLockedBy`, `lastLockedAt`, `pendingChangesReference`  
- On reconnect:
  - If section unchanged → apply pending operations
  - If section changed → merge options:
    1. Accept remote changes
    2. Keep mine (override)
    3. Manual merge (3-way diff/JSON merge)

---

## 8. Versioning & Merge

- Each edit versioned (`edits.version_to`)  
- Snapshots stored for revert/time-travel  
- Merge strategy on reconnect:
  - base = snapshot_at_lock_acquire  
  - remote = current document state  
  - local = pendingOps  
  - Merge can be automatic or require manual resolution

---

## 9. Presence & Cursors

- Active collaborators per document tracked  
- Cursor/selection positions broadcasted via Pub/Sub  
- Lock acquire/release and offline eviction broadcasted to all clients

---

## 10. Durability & No Data Loss

- Persist-before-broadcast: ensure all edits saved before notifying other users  
- Images stored on filesystem/object store; metadata in DB  
- Pending operations saved during offline, applied/merged on reconnect  
- Audit logs track all edits, lock events, offline eviction

---

## 11. Security & Permissions

- **JWT Issuance:**  
  - Auth Service issues JWTs signed with **private key** (RS256 recommended)  
  - Token includes claims: `sub=user_id`, `roles`, `doc_scope`, `exp`, `jti`  
  - Refresh tokens or rotation strategy supported  

- **JWT Validation in Document Service:**  
  - Document Service uses **public key** corresponding to Auth Service private key  
  - Validates signature, expiration, and claims  
  - Ensures token authenticity (cannot be forged)  

- **Permissions Enforcement:**  
  - Viewer → read-only access  
  - Editor → write in unlocked or lock-owned sections  
  - Owner → full control  
  - Enforcement applies to both **REST endpoints** and **WebSocket messages**  

- **Audit Logging:**  
  - Every edit, lock acquire/release, and offline eviction is logged  
  - Logs include user ID, document ID, operation type, timestamp, and session ID  

---

## 12. Observability & Operations

**Metrics:**
- Latency per document  
- Dropped messages  
- Sync errors  
- Active connections  
- Messages/sec  
- Persistence lag  
- Operation queue length

**Health endpoints:**
- `/healthz` → checks Redis + Postgres connectivity  
- `/ready` → service ready for requests

**Alerts examples:**
- Persistence lag > 500ms >1 min  
- Active connections drop >50% in 1 min  
- Unprocessed operation queue length > threshold

**Logging:**
- Structured logs with correlation IDs  
- Retention: Postgres 90 days, archive to compressed files

---

## 13. Scaling & Performance

- Redis + Pub/Sub for cross-instance broadcasting  
- Persist-before-broadcast latency ~50–100ms  
- 5 concurrent users per document, 1000 total editors → sharding for Redis/Postgres if needed  
- Images → object store/self-hosted filesystem

---

## 14. Tradeoffs

**Pros:**
- Predictable and conflict-free section editing  
- Offline-friendly with merge strategy  
- Simple client/server implementation

**Cons:**
- One editor per section limits concurrency  
- Some merges require manual resolution  
- Persist-before-broadcast adds small latency

---

## 15. Conclusion

This design:
- Ensures no data loss  
- Handles offline + reconnect scenarios with merge options  
- Supports 1000 concurrent editors with <200ms latency  
- Provides audit logs, security via JWT signed with private key, and operational observability
