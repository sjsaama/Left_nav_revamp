# PRD: Left Nav, Local DB, and API Architecture

**Status:** Final  
**Scope:** Appointment setting split, ordering window semantics, local DB retention, and API contracts for the consult/left-pane experience.  
**Out of scope:** 150-consult logic (removed); exact request/response schemas (to be defined in API spec).

---

## 1. Overview

This PRD defines the product and technical requirements for:

- **Splitting** the single “appointment window” setting into **EHR look-ahead** and **ordering window**
- **Left nav and date-based navigation** (default view, date strip, removal of “All” infinite scroll)
- **Local DB retention and eviction** (window guarantee, practice retention ceiling, fetch_filter TTL)
- **API behavior** for `fetchAll`, `fetchFilter`, and `fetchConsult` (window-aware sync, unified storage, merge awareness)

The outcome is a **strong guarantee** that the doctor’s working window (e.g. yesterday–today–tomorrow) is always complete on device—online and offline—with clear date-first navigation and predictable eviction rules.

---

## 2. Problem Statement

### 2.1 Current Architecture (Previous State)

| Aspect | Current behavior |
|--------|------------------|
| **Local DB** | Single consults table; hard cap ~150 consults; eviction by recency only. |
| **Data sources** | `fetch_all` polled ~60s; primary for home list. `fetch_filter` used for search/filtered lists; results often temporary (HOME, SEARCH, HOME_AND_SEARCH flags). |
| **Settings** | One overloaded “appointment window” setting drives both EHR look-ahead (polling, recaps) and UI ordering window (left pane, plus-button). |
| **UI** | Home/left pane and “All” tab read from local cache (≤150). Plus-button uses same cache; ordering indirectly influenced by window setting. |

### 2.2 Pain Points

1. **In-window data not guaranteed**  
   Even with a small UI window (e.g. yesterday–today–tomorrow), the 150-cap can evict some of those appointments. Doctors miss appointments in the left pane and in the plus → select-patient list.

2. **Overloaded setting**  
   One control drives EHR pull depth and default UI window. Power users who want long look-ahead (e.g. 30 days for recaps) get a huge UI window that local DB cannot faithfully cache.

3. **“All” + infinite scroll doesn’t match usage**  
   Doctors think in **dates**, not “150 recent rows.” The product should optimize for a clear working window and date-based navigation, not generic rolling history.

4. **Weak offline guarantees**  
   Offline, users only have whatever ≤150 consults were last synced. There is no guarantee that **today’s full schedule** is on device.

5. **Search/filter data ephemeral**  
   `fetch_filter` results are treated as temporary; no clear persistence or TTL, leading to inconsistent behavior when revisiting dates or search results.

### 2.3 Current API Behavior (fetch_all & fetch_filter)

Reference for the **as-is** contract before the changes in §5–6.

#### Fetch All Consults (`/fetch_all`)

| Aspect | Details |
|--------|---------|
| **Purpose** | Primary sync mechanism — keeps local DB in sync with backend |
| **Method** | `POST` |
| **When called** | Every ~60 seconds while app is active (polling); pagination when user scrolls beyond local limit (150) |
| **Request** | `{ "token": "string", "last_fetch_timestamp": "string", "last_consult_id": "string" }` |
| **Key response fields** | `consults[]`, `deleted_consult_ids[]`, `reset_settings`, `last_page` |
| **Offline behavior** | No API calls; shows locally cached consults (150 max; may be more if user was online and scrolled/searched within ~24h) |
| **Local storage** | Max 150 consults; timestamp-based conflict resolution; cleanup when over limit (exact policy TBD, e.g. after 24h beyond 150) |
| **Critical notes** | Core sync mechanism; 1GB storage limit enforced; pagination available |

#### Fetch Filter (`/fetch_filter`)

| Aspect | Details |
|--------|---------|
| **Purpose** | Search entire backend DB (beyond local 150 consults) |
| **Method** | `POST` |
| **When called** | User searches or filters consults; date range filters; provider filters; pagination in search results |
| **Request** | Search params, date ranges, provider filters, pagination |
| **Key response fields** | `consults[]`, `last_page` |
| **Offline behavior** | Online feature only |
| **Local storage** | Results flagged as `HOME`, `SEARCH`, or `HOME_AND_SEARCH`; temporary (cleaned up on exit or over time) |
| **Critical notes** | 50 results per page. |

---

## 3. Goals & Success Criteria

| Goal | Success criteria |
|------|-------------------|
| **Strong working-window guarantee** | For the configured ordering window (e.g. -1, 0, +1 days), all consults in that window are on device and never evicted (subject to practice retention). |
| **Separate EHR vs UI semantics** | Two distinct settings: EHR look-ahead (backend) and ordering window (UI + local retention). |
| **Date-first navigation** | Default view and navigation are date-anchored; “All” unbounded scroll is removed. |
| **Predictable offline behavior** | Device always has full data for the current ordering window; out-of-window data has clear persistence (e.g. 3-day TTL for fetch_filter-only). |
| **Unified consult storage** | Single consults table; both `fetch_all` and `fetch_filter` write to it; retention driven by business rules (window, practice retention, TTL, global cap). |

---

## 4. Settings: EHR Look-Ahead vs Ordering Window

### 4.1 Split Into Two Settings

**A. EHR look-ahead (backend / ops-managed)**

- **Meaning:** How far into the future to pull data from EHR for appointment poll and recap generation.
- **Behavior:** Ignore negative values; use the maximum positive value as look-ahead. Negative (look-back) is irrelevant for this setting.
- **Scope:** Backend only. Used for EHR appointment poll and recap generation window.
- **Not** used for UI or local DB retention.

**B. Ordering window (ops-managed; later user-configurable in left pane)**

- **Meaning:** The date window the doctor sees by default in the UI (left nav, plus-button “select patient”).
- **Behavior:** Uses minus and plus offsets from “today” (e.g. `[-1, 0, +1]` = yesterday, today, tomorrow). Can be represented as a continuous range `[minOffset, maxOffset]` (e.g. pastDays=1, futureDays=1).
- **Scope:** Drives default left pane view, plus-button list, and **local DB retention guarantee** (see §6).
- **Constraint:** Must lie **inside** the practice’s contracted retention period. If configuration attempts to exceed it (e.g. -60 days when contract is 30), ops must clamp or deny.

### 4.2 Migration

- Introduce a **new setting** for EHR look-ahead.
- On day one, **copy** existing “appointment window” values into both new settings.
- Thereafter: EHR uses only the positive part of look-ahead; ordering window is set or shrunk per desired UI behavior.
- Power users: e.g. large look-ahead (30 days) for recaps, small ordering window (-1, 0, 1) with search/date picker for far-future days.

---

## 5. API Requirements

### 5.1 Practice Retention (Contractual)

- Each practice has a **contracted retention period** (data retained on Marvix for X days).
- **Backend:** Must not serve consults/appointments older than that period. Applies to all consult-list and filter APIs.
- **Ordering window config:** Must not exceed the retention boundary; ops clamp/deny invalid ranges.
- **Local DB:** Must not keep or surface data older than the practice retention period (see §6.3).

### 5.2 fetchAll (window-aware sync)

| Requirement | Description |
|-------------|-------------|
| **Ordering-window awareness** | Request or server-side config must convey the ordering window (or its date range). Server returns **all** consults whose appointment date falls in that window—**no truncation by count**. |
| **Slice separation** | Conceptually: first slice = ordering window (complete); optional subsequent slices = out-of-window, paginated. Avoid complex SQL (e.g. unions with two different limit conditions); keep queries simple. |
| **Delta sync** | Continue to support delta updates (changes since `last_fetch_timestamp` / `last_consult_id`): new, updated, cancelled, merged consults—both in and out of window. |
| **Response** | Include `consults[]`, `deleted_consult_ids[]`; retain `reset_settings` if still in use. No fixed “page 1 = exactly N items” for the window slice. |

### 5.3 fetchFilter (search and date-based fetch)

| Requirement | Description |
|-------------|-------------|
| **Search** | When user searches (e.g. by patient name): offline = local only; online = local + server; merge server results into local DB and UI (Gmail-style). |
| **Date / out-of-window** | When user selects a date outside the ordering window (e.g. date strip or calendar): call with that date range; return consults for that range; results are upserted into the same local consults table. |
| **Persistence** | Results from `fetch_filter` are **persisted**. Rows that exist **only** because of a search or date-based fetch (and are **outside** the current ordering window) are **evicted after 3 days** (see §6.4). Rows that fall inside the ordering window are protected by the window guarantee. |

### 5.4 fetchConsult (merge awareness)

- If a consult was deleted via merge: response must support a **deleted** flag and a **link to the new consult** (e.g. `merged_into_consult_id`).
- Frontend and backend must be merge-aware: operations on a merged-away consult redirect to the target consult. No new request/response parameters mandated beyond this contract.

### 5.5 fetch_all & fetch_filter: Current vs New

| Aspect | Current (§2.3) | New (§5.2, §5.3, §6) |
|--------|-----------------|----------------------|
| **fetch_all role** | Primary sync; pagination when scrolling past 150 | Window-aware sync; **all** consults in ordering window returned (no count truncation); delta sync retained |
| **fetch_all local** | Max 150; evict by recency | No eviction for in-window consults; global cap + practice retention + 3-day TTL for fetch_filter-only |
| **fetch_filter role** | Search/filter beyond 150; online only | Search + date-based fetch (e.g. date strip); same API, results persisted |
| **fetch_filter local** | Temporary; HOME/SEARCH/HOME_AND_SEARCH flags; cleaned on exit | **Unified table**; upsert into same consults table; out-of-window fetch_filter-only rows **evicted after 3 days** |
| **Offline** | Local cache only (≤150) | **Full ordering window** always on device; search/out-of-window = local first, then fetch when online |

---

## 6. Local DB Architecture

### 6.1 Structural Changes

- **Single consults table** for both `fetch_all` and `fetch_filter` (unified storage; no separate “search table”).
- **Schema augmentation:**
  - `appointment_date` / `consult_date` is **central** for queries and retention.
  - **Derived (or stored):** `in_ordering_window: boolean` — true when `appointment_date` is within the current ordering window (today ± configured offsets), recomputed when window or date changes.
  - **Merge fields:** `deleted_due_to_merge: boolean`, `merge_target_consult_id: string | null`.
  - **Optional:** `fetched_via_filter_at` (or similar) to implement 3-day TTL for fetch_filter-only rows.
- **Indexes:** Support fast date-based queries (`appointment_date`) and efficient eviction scans (e.g. `in_ordering_window`, `appointment_date`).

### 6.2 Retention Hierarchy

Constraints apply in this order:

1. **Practice retention ceiling (hard cap)**  
   No consult may be kept or surfaced if its `appointment_date` is older than the practice’s contracted retention period. This overrides all other retention rules.

2. **Ordering-window guarantee**  
   Never delete consults whose `appointment_date` lies within the current ordering window **and** within the practice retention period. These are the “working window” consults.

3. **Global cap (storage)**  
   A global upper bound (e.g. 500 consults; tuned for memory, e.g. under 1GB) applies to **all** consults. When total count exceeds the cap, eviction applies only to non-window, eligible rows (see §6.3–6.4).

### 6.3 Eviction Rules

- **Never evict:** Rows with `in_ordering_window = true` and `appointment_date` within practice retention.
- **Eviction candidates:** Rows with `in_ordering_window = false` (and within practice retention). Sort by oldest `appointment_date` and/or last-accessed (LRU); delete until total ≤ global cap.
- **Sliding window:** As the calendar day or ordering window changes, some dates leave the window; their consults become eligible for eviction when over cap.

### 6.4 fetch_filter–Only Data: 3-Day TTL

- Rows that are in the DB **only** because of a `fetch_filter` call (search or date-based) and are **outside** the current ordering window are **evicted after 3 days** (e.g. using `fetched_via_filter_at` or equivalent).
- Rows that are in the ordering window are **not** evicted early regardless of how they were fetched.
- All rows remain subject to practice retention and global cap.

### 6.5 Window Change and Hydration

- When the ordering window **changes** (ops/user config or day rollover):
  1. Recompute `in_ordering_window` for all rows.
  2. Identify newly in-window dates that are not yet fully hydrated.
  3. Call fetch APIs (e.g. `fetch_all` or `fetch_filter` by date range) to pull consults for those dates.
  4. Upsert into local DB; then run eviction (respecting window guarantee and practice retention).

---

## 7. UI & Behavior Changes

### 7.1 Left Nav & Default View

- **Remove** the “All” unbounded infinite-scroll model. Navigation is **date-based** and **ordering-window–based**.
- **Default view on load:** Show all consults whose appointment date lies in the current ordering window (e.g. -1, 0, +1 → yesterday, today, tomorrow). Order follows configured order (e.g. today, tomorrow, yesterday).
- This is the **schedule view**: for EHR practices, appointments in this slice; for non-EHR, walk-in consults for those dates.

### 7.2 Window Picker via Settings Icon

- **No date strip or date picker** in the left pane. The default view is driven solely by the **ordering window** (see §4).
- A **settings icon** (e.g. gear) in the left pane header opens a **window picker** (View Options modal) where the user can set or change the ordering window (e.g. days back, days ahead, and optionally the order of past/today/future).
- The ordering window is ops-managed initially; the same UI will later support user-configurable window (per §4.1 B).
- When the day rolls over, the logical window slides (e.g. [-1, 0, 1] re-anchors around the new “today”); default view updates accordingly. **Out-of-window** dates are reached via **search or filters** (which trigger `fetch_filter`), not a date picker.

### 7.3 Plus-Button (“Select Patient”)

- Uses the **same** ordering-window data as the left pane default view (from local DB; no server call required for in-window consults).
- If a user needs a patient/appointment outside the window, they use **search or filters** (which may trigger `fetch_filter`).

### 7.4 Offline Semantics

- **In-window:** Device always has full data for the current ordering window (within practice retention). Left pane and plus-button can rely on local DB only.
- **Out-of-window / search:** Show local data first; when online, call `fetch_filter` and merge results. Out-of-window data is evicted per §6.3–6.4 (3-day TTL for fetch_filter-only; otherwise by cap).

### 7.5 EHR vs Non-EHR Practices

- **With EHR:** EHR look-ahead defines what the server can offer; ordering window defines what’s shown by default and protected locally. Users who pre-chart far ahead keep large look-ahead and use search/date to reach far-future days.
- **Without EHR:** Consults are working appointments; default view is window-only; older consults leave the window and are reached via **search or filters**.

---

## 8. Open Clarifications (Implementation Phase)

- **Ordering window representation:** Standardize on continuous range `[minOffset, maxOffset]` vs allow sparse sets (e.g. -1, 0, 2, 7). Recommendation: continuous range.
- **Exact fetchAll contract:** Separate “window hydration” call vs single `fetchAll` with window as first slice and pagination for the rest. Meeting agreed on behavior; exact API shape TBD.
- **Provider / department filters:** For multi-provider practices, whether ordering window and retention are per provider or global. To be defined.
- **Global cap value:** Final number (e.g. 500) and any platform-specific tuning (e.g. iOS vs Android vs web) to be set during implementation.
- **Current behavior (to be deprecated):** Cleanup of local consults beyond 150 after 24 hours — exact policy was TBD; replaced by new retention rules in §6.

---

## 9. Summary

| Area | Before | After |
|------|--------|--------|
| **Settings** | Single “appointment window” for EHR + UI | **EHR look-ahead** (backend) + **ordering window** (UI + local retention) |
| **Local DB** | ~150 cap; eviction by recency only | **Window guarantee** (never evict in-window); **practice retention** ceiling; **global cap**; **3-day TTL** for fetch_filter-only out-of-window |
| **fetchAll** | Truncated by count; no window semantics | **All consults in ordering window** returned; delta sync; simple queries |
| **fetchFilter** | Temporary; separate treatment | **Unified table**; persisted; **evict after 3 days** if out-of-window and fetch_filter-only |
| **fetchConsult** | — | **Merge-aware** (deleted + merge target ID) |
| **Left nav** | “All” + infinite scroll; cache-bound | **Ordering window** default; **settings icon → window picker**; search/filters for out-of-window |
| **Offline** | No guarantee for today’s schedule | **Full ordering window** always on device (within practice retention) |

---

*This PRD is the single source of truth for the Left Nav, Local DB, and API architecture. API request/response schemas and implementation details (indexes, exact field names) should be specified in separate technical specs.*
