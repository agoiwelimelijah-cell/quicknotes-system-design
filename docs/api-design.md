# QuickNotes REST API Design Specification

## 1. Overview
This document specifies the production RESTful API for **QuickNotes**, designed to transition the platform from a client-only prototype into a scalable cloud service serving over 1,000,000 active users.

* **Base URL:** `https://api.quicknotes.app/v1`
* **Transport:** HTTPS (TLS 1.3 enforced)
* **Authentication:** Bearer JSON Web Tokens (`Authorization: Bearer <jwt_token>`)
* **Content Negotiation:** `Content-Type: application/json`, `Accept: application/json`

---

## 2. API Endpoints

The QuickNotes API provides the following core endpoints:

| HTTP Method | Path | Description | Success Status | Auth Required |
| :--- | :--- | :--- | :--- | :---: |
| `POST` | `/api/v1/notes` | Create a new note for the authenticated user | `201 Created` | Yes |
| `GET` | `/api/v1/notes` | List paginated notes with optional search & filter | `200 OK` | Yes |
| `GET` | `/api/v1/notes/{id}` | Retrieve a specific note by ID | `200 OK` | Yes |
| `PUT` | `/api/v1/notes/{id}` | Fully update an existing note | `200 OK` | Yes |
| `PATCH` | `/api/v1/notes/{id}` | Partially update note fields (e.g. pin, archive, title) | `200 OK` | Yes |
| `DELETE` | `/api/v1/notes/{id}` | Permanently delete a note by ID | `204 No Content` | Yes |
| `GET` | `/api/v1/tags` | List all unique tags created by the user | `200 OK` | Yes |
| `POST` | `/api/v1/notes/{id}/tags` | Associate one or more tags with a note | `201 Created` | Yes |

---

## 3. Request and Response Examples

### 3.1 Create a Note (`POST /api/v1/notes`)

#### Request
* **Method & Path:** `POST /api/v1/notes`
* **Headers:**
  ```http
  Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
  Content-Type: application/json
  ```
* **Request Body:**
  ```json
  {
    "title": "System Architecture Review Meeting",
    "body": "Discussed sharding user_id, 3-tier caching hierarchy, and failover lag guard for read replicas.",
    "is_pinned": true,
    "tag_ids": [12, 18]
  }
  ```

#### Response
* **Status:** `201 Created`
* **Headers:**
  ```http
  Location: /api/v1/notes/4819
  Content-Type: application/json
  ```
* **Response Body:**
  ```json
  {
    "id": 4819,
    "user_id": 104,
    "title": "System Architecture Review Meeting",
    "body": "Discussed sharding user_id, 3-tier caching hierarchy, and failover lag guard for read replicas.",
    "is_pinned": true,
    "is_archived": false,
    "tags": [
      {
        "id": 12,
        "name": "architecture",
        "color_hex": "#4f46e5"
      },
      {
        "id": 18,
        "name": "work",
        "color_hex": "#10b981"
      }
    ],
    "created_at": "2026-10-13T10:30:00Z",
    "updated_at": "2026-10-13T10:30:00Z"
  }
  ```

---

### 3.2 List Notes (`GET /api/v1/notes`)

#### Request
* **Method & Path:** `GET /api/v1/notes?page=1&limit=10&is_archived=false&tag=architecture`
* **Headers:**
  ```http
  Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
  Accept: application/json
  ```

#### Response
* **Status:** `200 OK`
* **Headers:**
  ```http
  Content-Type: application/json
  X-Total-Count: 42
  X-Page: 1
  X-Page-Size: 10
  ```
* **Response Body:**
  ```json
  {
    "data": [
      {
        "id": 4819,
        "user_id": 104,
        "title": "System Architecture Review Meeting",
        "body": "Discussed sharding user_id, 3-tier caching hierarchy, and failover lag guard for read replicas.",
        "is_pinned": true,
        "is_archived": false,
        "tags": [
          {
            "id": 12,
            "name": "architecture",
            "color_hex": "#4f46e5"
          }
        ],
        "created_at": "2026-10-13T10:30:00Z",
        "updated_at": "2026-10-13T10:30:00Z"
      },
      {
        "id": 4702,
        "user_id": 104,
        "title": "Database Indexing Strategy",
        "body": "Ensure composite index on (user_id, is_archived, updated_at DESC) for fast feed scanning.",
        "is_pinned": false,
        "is_archived": false,
        "tags": [
          {
            "id": 12,
            "name": "architecture",
            "color_hex": "#4f46e5"
          }
        ],
        "created_at": "2026-10-12T14:15:00Z",
        "updated_at": "2026-10-12T15:20:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total_items": 42,
      "total_pages": 5,
      "has_next": true,
      "has_prev": false
    }
  }
  ```

---

## 4. Error Status Codes and Error Handling

The API adheres to standard HTTP status codes and formats errors using RFC 7807 Problem Details for HTTP APIs:

| HTTP Status Code | Name | Scenario |
| :--- | :--- | :--- |
| `400 Bad Request` | Client Validation Error | Title is missing, title exceeds 100 characters, or JSON body is malformed. |
| `401 Unauthorized` | Missing / Invalid Authentication | Missing `Authorization` header, invalid token signature, or expired JWT. |
| `403 Forbidden` | Access Denied | Authenticated user attempts to view, modify, or delete a note belonging to another user. |
| `404 Not Found` | Resource Not Found | Note ID does not exist or has already been deleted. |
| `429 Too Many Requests` | Rate Limit Exceeded | Client exceeds 120 requests per minute quota. |
| `500 Internal Server Error` | Server Exception | Unexpected database timeout, connection pool exhaustion, or internal unhandled exception. |

### Example Error Response Body (`400 Bad Request`)
* **Status:** `400 Bad Request`
* **Response Body:**
  ```json
  {
    "type": "https://api.quicknotes.app/v1/errors/validation-error",
    "title": "Validation Failed",
    "status": 400,
    "detail": "The request body failed field validation requirements.",
    "instance": "/api/v1/notes",
    "errors": [
      {
        "field": "title",
        "error": "Title is required and must not exceed 100 characters."
      }
    ],
    "timestamp": "2026-10-13T10:35:12Z"
  }
  ```

### Example Error Response Body (`404 Not Found`)
* **Status:** `404 Not Found`
* **Response Body:**
  ```json
  {
    "type": "https://api.quicknotes.app/v1/errors/not-found",
    "title": "Resource Not Found",
    "status": 404,
    "detail": "Note with ID 99999 was not found or was deleted.",
    "instance": "/api/v1/notes/99999",
    "timestamp": "2026-10-13T10:35:15Z"
  }
  ```
