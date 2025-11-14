# Detailed architectural / systems diagram — PWA Task Tracker (plane-ready)

Below is a clear, exam-friendly architecture explanation you can include with your diagram. It covers components, data flow, failure modes, DB migrations, cache-busting and how the app safely reconciles after deploys (no data loss, no duplicate tasks). I also include two ASCII diagrams you can copy into your diagram slide and a short “grader checklist” showing how to test the required behavior.

I’ll refer to your existing app where relevant (it registers a service worker and uses IndexedDB). See the registration and IndexedDB code in your `index.html`.  Your PWA manifest is in `manifest.json`. 

---

# 1 — High-level components

1. **Client UI (PWA shell)**

   * HTML/CSS/JS, offline UI, optimistic updates. (Your `index.html` contains form, offline indicator, and IndexedDB logic.) 

2. **Service Worker (SW)**

   * Controls precaching of static assets (app shell), runtime caching policies for other resources, intercepts network for offline-first behaviors, performs cache-busting/upgrade steps, and coordinates background sync / retry queue.

3. **Local Storage (IndexedDB, versioned)**

   * Persistent local store of tasks and an *operation queue* (outbox). Schema versioning + incremental migrations implemented in `onupgradeneeded`.

4. **Sync Engine (client-side)**

   * Manages a local operation queue, batches operations, retries with exponential backoff, handles idempotency & deduplication, and runs reconciliation with the server on reconnect or after deploy.

5. **Server API**

   * RESTful endpoints for CRUD (`/tasks`), idempotency support (accept client-supplied UUIDs), and a lightweight endpoint for server-side migration detection (optional /sync-meta).

6. **Deployment pipeline / release tag**

   * Each deploy produces: asset fingerprinting (hash), SW cache name version, and API release version. These are used to trigger safe client migrations.

---

# 2 — Design goals & constraints

* Work fully offline (airplane mode) for CRUD.
* No data loss when reconnecting or after a new deploy.
* No duplicate tasks after sync.
* Support DB schema migrations across app versions.
* Explicit caching strategy & cache-busting on release.

---

# 3 — Key design decisions (brief)

* **Optimistic UI + local persistence:** UI immediately reflects changes and writes to IndexedDB (store: `tasks`) and to an *outbox* (store: `queue`) for later sync. (See IndexedDB usage in your page.) 
* **Client-generated stable UUIDs for tasks:** Each new task gets a universally unique id (e.g., `uuid.v4()` or `Date.now().toString()` combined with random) so it is idempotent across retries and safe through migrations.
* **Outbox pattern + operation logs:** Store operations (create/update/delete) in `queue` with metadata: op-type, task-id, payload, timestamp, client-version. This permits replay and conflict resolution.
* **Server dedupe by task-id:** Server treats `task.id` as idempotency key — if a create with same id arrives, server returns existing resource rather than inserting duplicate.
* **Service worker caching strategy:**

  * Precaching (install): app shell assets (index.html, js, css, manifest, icons) — *cache-first* for instant load.
  * Runtime caching: API calls use *network-first with fallback to cache* for reads; static images / icons use *cache-first*.
  * Background sync or custom retry in client when SW `sync` not available.
* **Cache-busting / SW lifecycle:** Use cache names with version/fingerprint, `skipWaiting()` + `clients.claim()` flow plus notification to the client to run DB migration if needed. Remove old caches during `activate` event.
* **DB migrations:** Implement incremental migrations inside `onupgradeneeded` and maintain a `schema_version` in the DB. Migrations are applied stepwise (oldVersion -> newVersion). A migration log is stored in DB to allow rollback or inspection.

---

# 4 — Detailed flows (with sequence steps)

## A — App start (fresh or returning)

1. Browser loads `index.html` (app shell). 
2. SW registered (`/sw.js`) and either installs new SW or uses existing. Registration code in `index.html`. 
3. IndexedDB `open(DB_NAME, DB_VERSION)` runs. `onupgradeneeded` applies migrations (if `oldVersion < newVersion`) in small, deterministic steps. Each migration updates schema and optionally transforms data in the `tasks` store. (See `initDB()` skeleton in your page.) 
4. UI loads tasks from IndexedDB and renders.

## B — Offline CRUD (user adds/edits/deletes while airplane)

1. User creates task → UI updates immediately.
2. Client writes new task object to `tasks` store (persistent) and appends an `op` to `queue` store: `{opId, type: 'create', taskId, payload, ts, clientVersion}`.
3. Service Worker intercepts navigation requests and serves app shell from cache (precached). API network requests fail quickly (because offline) — client handles that by leaving items in queue. UI shows offline indicator (you already have that). 

## C — Reconnect / Sync (normal)

1. Network returns → Sync engine dequeues ops in timestamp order (or grouped by taskId) and sends batched requests to server.
2. Server processes each op idempotently: `create` uses client-supplied `taskId` as idempotency key; `update` applies if the server-side `lastModified` is older than incoming op timestamp; `delete` removes. Server returns canonical task objects including server `lastModified` timestamps.
3. Client applies server responses to local DB (update `tasks` store), removes successfully-synced ops from `queue`.
4. If server reports conflict (concurrent modification), client uses resolution policy (default: server-last-write-wins OR present merge UI to user). For tasks, last-write-wins by `lastModified` timestamp is usually acceptable.

## D — Deploy & cache-bust (after new release)

1. New deploy creates new asset fingerprints and a new `CACHE_VERSION` constant used inside SW (e.g., `const CACHE_NAME = 'app-shell-v2025-11-14-01'`).
2. New SW is served. Browser downloads new SW; in `install` the SW precaches new assets. In `activate` SW cleans up old caches and calls `clients.claim()`. Use `self.skipWaiting()` if you want immediate takeover (but warn user to avoid mid-edit kill).
3. **Critical step — data migration coordination:**

   * SW activation fires an event and broadcasts a `SW_UPDATED` message to clients (via `postMessage`).
   * Client receives `SW_UPDATED` and runs DB migration checks (if client DB version < required version). Then client runs `onupgradeneeded` or migration scripts. Migrations are **idempotent** and stepwise. Store an `applied_migrations` array in DB for auditing.
   * Client retains the existing outbox and replays queued ops after migration and after confirming network. Because tasks keep stable client-generated IDs, replays are idempotent and will not duplicate tasks on server.

## E — Deploy while offline / mid-sync

* If a deploy occurred while client is offline and the user later reconnects, the client will load the new SW and run migrations locally on next page load — queued ops are still present and will be replayed. Server dedupe ensures no duplicates. If schema changed, migration script transforms store items (and queue) into new schema.

---

# 5 — Data model & IDEMPOTENCY

* Each task: `{ id: "<client-uuid>", text, completed, createdAt, updatedAt }`
* Each outbox op: `{ opId: "<uuid>", opType: "create|update|delete", taskId, payload, ts, attemptCount }`
* **Deduplication:** Server uses `task.id` as unique key and rejects duplicate creates (responds with existing task). Server also supports `If-Unmodified-Since` / `If-Match` style headers or `lastModified` field for safer updates.

---

# 6 — DB Migrations (practical recipe)

* Use `indexedDB.open(DB_NAME, NEW_VERSION)` → `onupgradeneeded` receives `oldVersion` and `newVersion`.
* Implement a migration registry:

  ```js
  const MIGRATIONS = {
    1: function(db) { /* create objectStore 'tasks', 'queue' */ },
    2: function(db) { /* add index 'completed' */ },
    3: function(db) { /* add migration to rename field */ }
  };
  ```
* In `onupgradeneeded`, iterate `for (let v = oldVersion + 1; v <= newVersion; v++) MIGRATIONS[v](db)`.
* If data transformation required (e.g., normalize `task.text` into `content`), run transformations inside a transaction on the store.
* Keep migrations small and reversible where possible; log each applied migration in a `meta` store.

Your `initDB()` already calls `indexedDB.open(DB_NAME, DB_VERSION)` and has a place to add migrations. 

---

# 7 — Service Worker caching & cache-busting recipe

1. **Cache naming:** `const APP_SHELL_CACHE = 'app-shell-v<build-hash>';` embed the build hash at build time.
2. **Install:** precache the app shell assets (index.html, CSS, JS bundles, manifest, icons).
3. **Activate:** delete caches not matching the current `APP_SHELL_CACHE`. Then call `self.clients.claim()`.
4. **Fetch handler (pseudo):**

   * `GET /` and static assets → *cache-first* (so app loads offline).
   * `GET /api/tasks` → *network-first* with cache fallback (shows last-seen tasks when server unreachable).
   * `POST/PUT/DELETE /api/tasks` → forward to network when online; if network fails, the client writes to `queue` and the SW returns a synthetic 202 Accepted response (or allows client to handle offline).
5. **Update flow:** when new SW activates, `postMessage({type: 'SW_UPDATED', cacheVersion})` to clients so they can run any required DB migration or prompt user to reload.

---

# 8 — Reconciliation & avoiding duplicates

* **Client-generated stable IDs** prevents duplicate creates.
* **Outbox ops are monotonic and contain opId** so server can respond with success and client can remove op from queue.
* **Server API** uses `UPSERT` semantics keyed on `task.id` or an explicit idempotency key header.
* **If network error & partial batch success:** server returns per-op result; client removes only succeeded ops and retries failed ones. Use a `retryCount` and exponential backoff to avoid flooding.

---

# 9 — Edge cases & mitigations

* **Concurrent edits across devices:** server stores `lastModified` and resolves with last-write-wins, or produce merge UI. For tasks simple LWW is acceptable.
* **SW replaced while queue flush in progress:** queue stays in IndexedDB; new SW won’t remove queue. After new SW activates, client resumes replay.
* **DB corruption / abort:** detect DB exceptions and offer user export or reset — but warn about data loss. Provide an optional server-side backup endpoint to upload full dump on sync.
* **Clock skew across clients:** use server timestamps as truth. Each op should include both client ts and server will stamp its own `lastModified`.

---

# 10 — Sequence diagram (offline create → reconnect → deploy during offline → sync)

```
User      Client App      IndexedDB (tasks, queue)      SW Cache      Server API
 |            |                  |                         |              |
 | Add task   |→ create task obj | add to tasks store      |              |
 |            |  create op in q  | add op to queue         |              |
 |            |  optimistic UI   |                         |              |
 | (offline)  |                  |                         |              |
--- reconnect -------------------------------------------------------------->|
 |            | dequeue ops      |                         | server processes
 |            | send batched req  |                         | dedupe by task.id
 |            | apply server resp | update tasks store      | return canonical
 |            | remove succeeded | remove ops               | 
 |            |                  |                         | 
--- deploy happens on server while offline client still queued ------------->|
 |            | new SW available |                         |              |
 |            | new SW installed  | purge old caches        |              |
 |            | postMessage: SW_UPDATED -> run migration   |              |
 |            | migrate DB schema |                         |              |
 | reconnect  | replay queue      |                         | server accepts idempotent ops
```

---

# 11 — Practical grader checklist (how to test & what you should see)

1. **Start with network=Online**

   * Add tasks, ensure they appear and server stores them (check API).
2. **Switch to Airplane (offline)**

   * Add / complete / delete several tasks. UI should update immediately; they remain after reload. The offline indicator should show. 
3. **Switch network to slow-3G in devtools then back online**

   * Observe queued ops upload, no duplicates on server, tasks reconciled.
4. **Deploy a new version (simulate by bumping `CACHE_VERSION` and `DB_VERSION`)**

   * New SW should activate, cache should update and old caches removed. Client should run migration on next load and preserve outbox. After reconnect, no data loss and no duplicate tasks.
5. **Edge test:** While offline, add a task, then simulate deploy, then reconnect — verify server has one task only and client queue cleared.

---

# 12 — Implementation notes you can include on your diagram

* **Outbox key idea:** atomic local write (task + op in same transaction) so no operation gets lost.
* **Migration guarantee:** migrations run in `onupgradeneeded` and are idempotent — safe even if interrupted. Keep small incremental migrations numbered. Example in your `initDB()` shows where to implement this. 
* **Cache versioning:** embed build hash (or timestamp) into `CACHE_NAME`. SW `activate` cleans old caches.
* **Message bus:** use `postMessage` between SW and clients for lifecycle events (e.g., `SW_UPDATED`, `SYNC_COMPLETE`).
* **Testing hooks:** expose `window.debugFlushQueue()` to manually trigger queue replay for automated tests.

---

# 13 — Short, copy-ready diagram captions (for the diagram boxes)

* **App Shell (cache-first)** — index.html, CSS, JS bundles, manifest. (Precached)  
* **IndexedDB (versioned)** — `tasks`, `queue`, `meta`. Migrations run at DB version bump. 
* **Service Worker** — precache, runtime caching, cache-bust on deploy, postMessage on update.
* **Outbox / Sync Engine** — durability + replay, idempotent ops.
* **Server API** — idempotent endpoints, dedupe by task id, return canonical state with `lastModified`.

---

If you want, I can:

* produce a high-resolution diagram image (SVG/PNG) of the above architecture you can drop into your slide; **or**
* convert the ASCII diagrams into a nicer flowchart (Mermaid or draw.io XML) you can import into your presentation.

Which would you prefer?
