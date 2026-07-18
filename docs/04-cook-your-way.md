# Cook Your Way (CYW) — An Offline-First, Inventory-Aware Cooking App

**Author:** Nicolas Yammouny (MMC-DEV Lab)

---

## What It Is

Cook Your Way connects three things that are usually built as separate apps —
recipes, a household food inventory, and ingredient-aware sourcing — into one
offline-first system. The core design question wasn't "how do we show recipes,"
it was "how does the app know what's already in the kitchen, what's about to
expire, and what a real dish looks like from exactly what's on hand."

---

## Architecture

**Recipe engine** — full CRUD (create, read, update, delete) and search from day
one, not a static content list.

**Inventory layer** — food-type-aware FIFO logic, not date-only tracking. Different
food categories have different real shelf-life rules (fresh-frozen vs. store-frozen,
fish vs. beef, cooked leftovers), and the inventory model reflects that rather than
treating all expiry the same way.

**Leftover-matching logic** — given a partial, incomplete set of ingredients (half a
protein, wilted herbs, leftover grain), the system suggests a real, complete dish
rather than just flagging "these items are expiring."

**Sourcing layer** — only activated after the inventory and leftover layers have
already been checked; the app reasons about what's genuinely missing before ever
suggesting a purchase, with no forced vendor selection.

---

## Stack

- **Backend:** Python, FastAPI — async, auto-documented
- **Database:** SQLite for development, with a PostgreSQL migration path for
  production
- **Frontend:** React + Vite — PWA-native, Capacitor-ready
- **Offline layer:** IndexedDB client storage with a sync queue, so the app
  functions fully without a live connection
- **Mobile packaging:** Capacitor, wrapping the PWA into a native APK
- **Planned (v2) vision input:** camera-based inventory recognition via
  vision-capable models, designed into the architecture from v1 so it can be added
  without a rebuild

---

## Design Principles

Consistent with the rest of the ecosystem this is built alongside:

- **Offline-first** — the app works without an internet connection, not as a
  fallback mode but as the default assumption
- **User owns their data** — no selling, no behavioral profiling
- **No forced vendors** — the app suggests, the user decides
- **Explainable** — no black-box suggestion logic; the reasoning behind a
  suggestion is always visible
