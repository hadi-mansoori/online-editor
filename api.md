# API Specification: Real-Time Collaborative Web Page Editor

All endpoints require **JWT authentication** (RS256 signed by Auth Service).  
Clients must include `Authorization: Bearer <JWT>` header for REST requests, or send JWT on WebSocket handshake.

---

## 1. Auth Service

### 1.1 Login
- **POST** `/auth/login`
- **Body:**
```json
{
  "username": "string",
  "password": "string"
}
```
- **Response:**
```json
{
  "access_token": "jwt",
  "refresh_token": "jwt",
  "expires_in": 3600
}
```
- **Notes:** Returns JWT signed with private key, including `sub=user_id`, `roles`, `doc_scope`.

### 1.2 Refresh Token
- **POST** `/auth/refresh`
- **Body:**
```json
{
  "refresh_token": "jwt"
}
```
- **Response:**
```json
{
  "access_token": "new_jwt",
  "refresh_token": "new_refresh_jwt",  
  "expires_in": 3600
}
```

---

## 2. Document Service (REST)

### 2.1 Get Document Metadata
- **GET** `/documents/{doc_id}`
- **Permissions:** viewer+  
- **Response:**
```json
{
  "doc_id": "string",
  "title": "string",
  "sections": [
    {"id": "s1", "locked_by": "user_id_or_null"}
  ],
  "version": 123
}
```

### 2.2 Get Section Content
- **GET** `/documents/{doc_id}/sections/{section_id}`
- **Permissions:** viewer+  
- **Response:**
```json
{
  "section_id": "s1",
  "content": "string or JSON",
  "locked_by": "user_id_or_null",
  "version": 5
}
```

### 2.3 Acquire Lock
- **POST** `/documents/{doc_id}/sections/{section_id}/lock`
- **Permissions:** editor+  
- **Response:**
```json
{
  "section_id": "s1",
  "locked_by": "user_id",
  "ttl": 60
}
```
- **Errors:** `409 Conflict` if already locked

### 2.4 Release Lock
- **POST** `/documents/{doc_id}/sections/{section_id}/unlock`
- **Permissions:** editor+ (owner of lock)  
- **Response:** `200 OK`

### 2.5 Merge Section (New Endpoint)
- **POST** `/documents/{doc_id}/sections/{section_id}/merge`
- **Permissions:** editor+  
- **Body:**
```json
{
  "strategy": "accept_remote" | "keep_mine" | "manual",
  "local_changes": { /* optional if manual */ }
}
```
- **Response:**
```json
{
  "status": "merged",
  "merged_content": { /* merged section content */ },
  "version": 6
}
```
- **Notes:**
  - `accept_remote`: server discards local pending changes and uses current remote state
  - `keep_mine`: server applies local changes, overwriting remote
  - `manual`: client provides diff for 3-way merge

### 2.6 List Collaborators / Presence
- **GET** `/documents/{doc_id}/collaborators`
- **Permissions:** viewer+  
- **Response:**
```json
{
  "collaborators": [
    {"user_id": "u1", "cursor": {"line": 5, "col": 12}, "role": "editor"}
  ]
```

### 2.7 Version History
- **GET** `/documents/{doc_id}/versions`
- **Permissions:** viewer+  
- **Response:**
```json
[
  {"version": 120, "timestamp": "2025-11-21T12:00:00Z", "author": "user_id"},
  {"version": 121, "timestamp": "2025-11-21T12:05:00Z", "author": "user_id"}
]
```
- **GET specific version** `/documents/{doc_id}/versions/{version}`

---

## 3. Document Service (WebSocket)

**WebSocket URL:** `wss://doc-service.example.com/ws?token=<JWT>&doc_id=<doc_id>`

### 3.1 Message Types

#### 3.1.1 `edit`
- **Sent by client** to apply changes
```json
{
  "type": "edit",
  "section_id": "s1",
  "content": "updated content",
  "version": 5
}
```
- **Response broadcast to other clients:**
```json
{
  "type": "edit",
  "section_id": "s1",
  "content": "updated content",
  "version": 6,
  "user_id": "u1"
}
```

#### 3.1.2 `lock_acquire` / `lock_release`
- Client sends lock/unlock requests  
- Server broadcasts lock state changes to all collaborators

#### 3.1.3 `merge_request`
- Sent by client when reconnecting after offline
```json
{
  "type": "merge_request",
  "section_id": "s1",
  "pending_changes": { ... },
  "strategy": "accept_remote" | "keep_mine" | "manual"
}
```
- Server responds:
```json
{
  "type": "merge_result",
  "section_id": "s1",
  "merged_content": { ... },
  "version": 6
}
```

#### 3.1.4 `cursor_update`
- Client updates cursor position
```json
{
  "type": "cursor_update",
  "section_id": "s1",
  "cursor": {"line": 5, "col": 12}
}
```
- Broadcast to all active collaborators

#### 3.1.5 `presence_update`
- Sent periodically or on join/leave
```json
{
  "type": "presence_update",
  "user_id": "u2",
  "role": "editor",
  "sections_locked": ["s1"]
}
```

---

## 4. Error Codes

- `401 Unauthorized` → JWT invalid or expired  
- `403 Forbidden` → insufficient permissions  
- `404 Not Found` → document or section not found  
- `409 Conflict` → lock acquisition failed  
- `422 Unprocessable Entity` → merge strategy invalid  
- `500 Internal Server Error` → unexpected

---

## 5. Example JWT Usage

```http
GET /documents/123 HTTP/1.1
Host: doc-service.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI...
```
- Document Service validates signature using **public key**, checks `doc_scope` and `roles`.

---

**Notes:**  
- REST endpoints used for metadata, version history, lock management, and explicit merge operations  
- WebSocket used for real-time editing, cursor updates, presence, and broadcast of changes  
- Audit logs record all operations (lock, edit, merge, reconnect)

