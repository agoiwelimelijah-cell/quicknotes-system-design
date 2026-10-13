# QuickNotes: System Design & API Client

A scalable system architecture design, production REST API specification, relational data model, and interactive frontend API prototype for **QuickNotes**—scaling from a browser-only app to a reliable cloud service for 1,000,000 users.

---

## 📋 Table of Contents
1. [Project Overview](#project-overview)
2. [How to Run the API Client](#how-to-run-the-api-client)
3. [System Design Documents](#system-design-documents)
4. [API Client Features & Practice Endpoints](#api-client-features--practice-endpoints)
5. [What I Learned](#what-i-learned)
6. [Repository Structure](#repository-structure)

---

## 🌟 Project Overview
As the Lead Engineer for QuickNotes, this project delivers two core milestones:
1. **Frontend API Client:** A resilient, secure frontend client connecting to REST endpoints to demonstrate asynchronous GET, POST, and DELETE requests, client-side validation, UI state management, and strict XSS defense.
2. **Backend Engineering Blueprints:** Comprehensive technical design documents that the backend engineering team will build from to support 1 million registered users.

---

## 🚀 How to Run the API Client

The API client runs directly in any modern web browser without requiring complex build steps or dependencies.

### Option 1: Direct File Opening
Double-click `index.html` in your file explorer, or right-click and choose **Open with > Chrome / Firefox / Safari / Edge**.

### Option 2: Using VS Code Live Server
1. Open this repository folder in Visual Studio Code.
2. Install the **Live Server** extension (by Ritwick Dey).
3. Right-click on `index.html` and select **"Open with Live Server"**.
4. The client will open at `http://127.0.0.1:5500`.

### Option 3: Using Python HTTP Server
Run from the repository root:
```bash
python3 -m http.server 3000
```
Then navigate to `http://localhost:3000` in your web browser.

### Option 4: Using Node.js / Vite
```bash
npm install
npm run dev
```

---

## 📚 System Design Documents

All technical design documents are located inside the [`docs/`](docs/) directory:

1. **[docs/api-design.md](docs/api-design.md)**  
   * Complete REST API specification for QuickNotes.
   * 8 documented endpoints (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) with request/response schemas.
   * Full HTTP error catalog (`400`, `401`, `403`, `404`, `429`, `500`) following RFC 7807 problem details.

2. **[docs/data-model.md](docs/data-model.md)**  
   * Relational entity model for `users`, `notes`, `tags`, and `note_tags`.
   * Explanation of one-to-many and many-to-many relationships.
   * Production PostgreSQL DDL `CREATE TABLE` statements with cascading foreign keys.
   * Composite indexes for user feed scanning (`idx_notes_user_feed`).
   * Example SQL queries including tag aggregation via `LEFT JOIN`.
   * Detailed engineering justification for choosing SQL over NoSQL.

3. **[docs/architecture.md](docs/architecture.md)**  
   * Functional and non-functional requirements.
   * Back-of-the-envelope capacity estimates for 1M registered users (200,000 DAU; 4.6 avg / 23.1 peak writes/s; 46.3 avg / 231.5 peak reads/s; 328.5 GB/year storage).
   * ASCII architecture diagram showing DNS, CDN, ALB, Stateless App Server fleet, Redis Cache, PostgreSQL Primary & Read Replica, and SQS Queue + Worker.
   * One-sentence problem statements per component.
   * Step-by-step request flows for `GET /notes` and `POST /notes`.
   * Analysis of architectural trade-offs and single-point-of-failure (SPOF) mitigation strategies.

---

## ⚡ API Client Features & Practice Endpoints

* **Base Practice API:** `https://jsonplaceholder.typicode.com/posts`
* **GET (Load 10 Notes):** Requests `GET /posts?_limit=10`, disables buttons during transit, displays loading spinner, and renders note items with titles and bodies.
* **POST (Create Note):** Validates title (required, max 100 characters), sends `{ title, body, userId: 1 }`, prepends newly created note to top of feed, and displays HTTP status code and new ID (e.g. `Note created (status 201, id 101)`).
* **DELETE (Remove Note):** Sends `DELETE /posts/{id}` and removes the note from the DOM upon success.
* **State Management:** Distinct UI styling for **Loading**, **Success**, **Error**, and **Empty** states.
* **Security Discipline:** All user text is injected using `textContent` only to prevent Cross-Site Scripting (XSS).

---

## 💡 What I Learned

1. **Defensive DOM Security (textContent vs. innerHTML):**  
   Dynamically rendering user-provided content like note titles and bodies using `textContent` is essential to prevent stored XSS attacks. By avoiding `innerHTML` when interpolating user data, malicious `<script>` tags or inline event handlers are neutralized before reaching the browser parser.

2. **Back-of-the-Envelope Capacity Estimations Dictate Architecture:**  
   Calculating the 10:1 read-to-write ratio (46.3 reads/s vs. 4.6 writes/s) proved that the system is read-heavy. This directly informed the need for an in-memory Redis caching layer and read replica separation, preventing repetitive read queries from overwhelming the primary relational database.

3. **Handling Asynchronous UI States Gracefully:**  
   Users must never wonder whether a button click was registered. Disabling buttons during in-flight network requests, providing clear feedback banners ("Loading notes...", "Note created"), and designing an empty state prevents accidental duplicate submissions and enhances user confidence.

4. **Normalized Data Modeling with Composite Indexes:**  
   Structuring notes and tags into normalized tables with a junction table (`note_tags`) provides referential integrity. Creating a composite B-Tree index on `(user_id, is_archived, is_pinned DESC, updated_at DESC)` ensures user feed queries execute in $<1\text{ms}$ at 1 million user scale.

---

## 📁 Repository Structure

```
quicknotes-system-design/
├── index.html              # HTML5 application skeleton
├── api.js                  # Frontend REST API client (GET, POST, DELETE)
├── style.css               # Styling with success/error/loading states
├── docs/
│   ├── api-design.md       # REST API specification & endpoints
│   ├── data-model.md       # PostgreSQL schema, DDL, queries & indexes
│   └── architecture.md     # 1M user scaling architecture & estimates
├── day7/
│   └── photo-app-scaling.md# Day 7 SnapShare scaling plan deliverable
└── README.md               # Project documentation & overview
```
