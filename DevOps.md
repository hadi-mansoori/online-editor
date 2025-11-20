# DevOps Guide: Real-Time Collaborative Web Page Editor

This guide explains how to manage services independently on separate servers with secure communication, and also describes the JWT and WebSocket flow for user synchronization.

---

## 1. Service Architecture

- **Auth Service**:
  - Runs on its own server
  - Own database (Postgres or Redis) on separate server
  - Independent Docker container
  - Responsible for authentication, issuing JWT, managing refresh tokens

- **Document Service**:
  - Runs on its own server
  - Own database (Postgres) on separate server
  - Redis for locks, presence, and Pub/Sub on a separate server
  - Independent Docker container
  - Responsible for document management, edits, merge, presence, and cursor

- **Networking**:
  - Services are connected via a logical network (VPN or overlay network) to allow communication across servers
  - Clients connect directly to Document Service for WebSocket and REST
  - Services communicate using hostnames/IPs of external servers

---

## 2. JWT Flow and Authentication

1. Client logs in → `Auth Service` issues JWT signed with **private key**.
2. JWT contains claims:  
   - `sub=user_id`  
   - `roles`  
   - `doc_scope`  
   - `exp`  
   - `jti`
3. Client sends JWT to `Document Service` via REST or WebSocket.
4. `Document Service`:
   - Validates JWT signature using **public key** from Auth Service.
   - Checks roles and scope.
5. **JWT in WebSocket**:
   - Sent during handshake (`wss://document-service.example.com/ws?token=<JWT>`).
   - Connection allowed only for authenticated users.
6. **Refresh Token**:
   - Can be stored in **HttpOnly cookie** or statefully in Redis/Postgres.
   - Both approaches are supported:  
     - Cookie → secure browser storage  
     - Backend → stateful management with revocation/tracking

---

## 3. User Synchronization and WebSocket

- Each user has a session and connects via WebSocket.
- **Presence & Cursor**:
  - Each user's changes are broadcast via Redis Pub/Sub
  - All clients in a document receive other users' status
- **Lock**:
  - Section lock state stored in Redis with TTL
  - Ensures automatic release on disconnect with pending changes recorded
- **Offline & Reconnect**:
  - Pending changes stored in DB/Redis
  - On reconnect, merge can be automatic or manual

---

## 4. Example Environment Variables (for services on separate servers)

```yaml
# Auth Service
DATABASE_URL=postgres://auth:pass@postgres-server:5432/auth
REDIS_URL=redis://redis-server:6379

# Document Service
DATABASE_URL=postgres://doc:pass@postgres-server:5432/doc
REDIS_URL=redis://redis-server:6379
AUTH_PUBLIC_KEY_URL=http://auth-server:8000/public-key
```

---

## 5. Operational Notes

- Each service runs on its own server with its own database
- Redis is on a dedicated server
- Real-time changes are broadcast via Redis Pub/Sub
- JWT exchanged between services → limited scope ensures secure access
- Refresh token can be **stateful** in backend or stored in **HttpOnly cookie**
- Lock, presence, and cursor stored in Redis → better scalability
- Persist-before-broadcast ensures no edits are lost
- Services can be monitored via health-check endpoints (`/healthz`, `/ready`)
