# Masterplan: Equipment Serviceability Dashboard
*Last updated to reflect prototype v3 (post-validation session)*

---

## 1. App Overview & Objectives

A web-based internal dashboard that enables ground personnel to track and update the serviceability states of equipment in real time, and allows the Super Admin (Boss) to view live, accurate serviceability data across all equipment — eliminating the need for manual screenshot-based reporting.

**Core Problem Solved:** Replace slow, error-prone manual reporting with a live, always-accurate dashboard that anyone with access can open and immediately understand.

**Key Objectives:**
- Provide a clear, colour-coded view of equipment serviceability at a glance
- Track serviceability at 3 tiers: Category → Model → Sub-component
- Enable ground personnel to update statuses quickly and accurately
- Give the Super Admin a live, customisable view for reporting purposes
- Enforce accountability through mandatory defect remarks where relevant

---

## 2. Target Audience

| User Role | Description |
|-----------|-------------|
| **Super Admin (Boss)** | Full access. Views all equipment, manages the hierarchy (CRUD), can override auto-calculated statuses, customises their dashboard view. |
| **Ground Personnel** | Limited access. Updates serviceability states for their assigned categories only. Views only their assigned slice of the dashboard. |

**Scale:** ~10 users, ~15 equipment sets. Small internal tool — lean architecture is appropriate.

---

## 3. Core Features & Functionality

### 3.1 Three-Tier Equipment Hierarchy

All equipment is organised in a 3-level structure:

```
Tier 1 — Category        e.g. Phones
  Tier 2 — Model         e.g. iPhone 13
    Tier 3 — Sub-component  e.g. Camera Lens
```

- Super Admin can fully CRUD all tiers via the **Manage → Categories** page
- Ground Personnel cannot modify the hierarchy — view and status-update only
- Each tier shows a tier badge (`CAT` / `MODEL` / `SUB`) in the table view for quick identification

### 3.2 Serviceability States

Each item at any tier carries one of five states:

| State | Colour | Meaning |
|-------|--------|---------|
| Serviceable | Green `#66bb6a` | Fully operational, ready for use |
| Limitations | Amber `#ffa726` | Usable but has known defects — remarks required |
| Under Maintenance | Blue-grey `#78909c` | Currently being repaired or serviced |
| Unserviceable | Red `#ef5350` | Cannot be used; repairable |
| Attrited | Dark charcoal | Permanently damaged, beyond repair |

**Colour design notes:**
- Attrited uses a dark charcoal background with a visible `#444` border to remain legible against the near-black UI, while remaining visually distinct from Under Maintenance (blue-grey)
- Status colours are applied consistently as badges, strip bars, and legend dots throughout the UI
- The legend is hidden behind a hoverable `ⓘ` icon to reduce visual noise — tooltip appears on hover with white background

### 3.3 Status Roll-Up Logic ("Worst Status Wins")

Parent tiers automatically inherit the most severe status from their children:

**Severity order (most → least severe):** Attrited > Unserviceable > Under Maintenance > Limitations > Serviceable

- A Category's status = worst status across all its Models' sub-components
- A Model's status = worst status across all its Sub-components
- Roll-up is computed on the fly — not stored separately
- **Manual Override** (Super Admin only): Super Admin can bypass the auto-calculated status for any Category or Model. An override is indicated by a cyan `refresh` icon next to the badge. Overrides are stored on the entity object and respected during display; clearing the override reverts to auto-calculation.

### 3.4 Defect Remarks

- When status is **Limitations**: remarks field is **mandatory** — at least one entry required before saving
- When status is **any other state**: remarks field is **optional**
- Multiple remarks supported via **"+ Add Remark"** button — each remark is numbered sequentially (`Defect #1`, `Defect #2`, etc.)
- Numbering is DOM-based (not a running counter) — deleting a remark re-numbers the remaining ones correctly
- Remarks are saved alongside the status update with a timestamp
- In the table view, the first remark is shown as a preview pill; if more exist, a `+N more` badge is shown and clicking it opens the edit modal

### 3.5 Dashboard — Summary View (Cards)

The default landing view. Shows a health snapshot per category as expandable cards:

**Each card contains:**
- Category name + status badge
- Serviceable count + percentage (e.g. `5/9  56%`) — on one line, no "Ready rate" label
- Colour-coded status strip — green (serviceable) always anchors the left, severity increases rightward
- Breakdown dots showing counts per status
- Depth-controlled expansion: Model rows and Sub-component rows appear inline based on the depth filter

**Card interactions:**
- Click the pencil icon to open the status update modal for that item
- Cards expand/collapse based on the global depth filter (Category only / +Models / +Sub-components)

### 3.6 Dashboard — Table View

A toggleable alternative view. Accessed via the Summary/Table toggle in the secondary bar.

**Table structure:**
- Fixed column widths: Equipment/Component 35% · Status 15% · Last Updated 15% · Remarks 25% · Actions 10%
- `<colgroup>` used for reliable column sizing
- Tier rows are visually distinct:
  - **Category rows** — tinted surface background, 6px gap above each group, left cyan border accent
  - **Model rows** — indented, transparent background
  - **Sub-component rows** — further indented, tree-line connectors drawn via CSS pseudo-elements
- Tier badges (`CAT`, `MODEL`, `SUB`) labelled in distinct colours
- Remarks column: first remark previewed inline; `+N more` badge for additional remarks
- Edit action: pencil icon button (same style as Categories page)

### 3.7 Status Update Modal

Triggered by clicking any edit button (pencil icon) across summary cards, table rows, or sub-component rows.

**Modal behaviour:**
- Shows item name and breadcrumb path (`Category › Model › Sub-component`)
- **Sub-component level**: direct status selection — all 5 options shown as selectable rows
- **Model / Category level**: shows an "Auto-calculated (Worst Status Wins)" banner with the computed status badge; a "Manual override" checkbox unlocks the status selector; override indicator (`refresh` icon) appears on the badge when active
- Status selector is locked (greyed out) when override is off for model/category edits
- Defect remarks section: numbered entries, mandatory for Limitations, optional otherwise; `+ Add Remark` button appends a new numbered field
- Validation: saving with Limitations and no remarks shows an inline error; save is blocked
- All saves are immediate — no approval step

### 3.8 Customisable Dashboard View

The dashboard supports two filter dimensions via the persistent secondary bar:

**Dimension 1 — Category filter (what to show)**
- Dropdown to select a specific category or "All Categories"
- Role-aware: Ground Personnel only see their assigned categories in the dropdown

**Dimension 2 — Depth toggle (how deep to show)**
- Three chip buttons: `Category only` / `+ Model` / `+ Sub-component`
- Global — applies uniformly across all visible categories
- `getDepth()` helper function designed to be swapped for per-category depth map in a later phase

**Interaction:** Both filters apply simultaneously. Changing either updates the dashboard instantly.

**Session behaviour:** Filter state persists for the current session; resets on refresh.

---

## 4. Navigation & Layout

### Top Status Bar (DSTA-inspired)
A persistent full-width bar fixed at the top of every page containing:
- **☰ Menu button** (far left) — opens the sidebar drawer
- **Logo** (EquipTrack wordmark + icon)
- **Status pills** (centre): `● Online` · `THREATCON ORANGE` · `DEFCON 4` · live clock (updates every 10 seconds)
- **Role switcher** (right) — dropdown in a cyan-bordered select; changes the active role and filters the dashboard accordingly
- **User avatar** (far right)

### Collapsible Sidebar Drawer
Slides in from the left when ☰ is clicked; closes via `×`, clicking a nav item, or clicking the dim overlay.

**Nav items:**
- Monitor: Dashboard, All Equipment
- Manage: Categories (CRUD), Personnel (Phase 2), Settings (Phase 2)

Shows current user name, role label, and avatar at the bottom.

### Secondary Bar
Sits below the top status bar, sticky. Contains the category filter, depth chips, clear button, and the Summary/Table view toggle (right-aligned).

### Page Layout
- No persistent sidebar — full viewport width used for content
- Content area: `padding: 20px 24px`
- Categories page: max-width 860px, centred

---

## 5. Categories CRUD Page (Manage → Categories)

Super Admin only. Accordion-based hierarchy editor.

**Features:**
- Add / rename / delete Categories, Models, and Sub-components
- Accordion expand/collapse per category and per model
- Inline rename: click ✏ → input field appears; press Enter or click Save
- Delete: confirmation dialog before committing
- Add Model / Add Sub-component buttons render on a single line (`white-space: nowrap`)
- Any data changes immediately reflect on the Dashboard (shared data state)
- Syncs the category filter dropdown when categories are added/renamed/deleted

---

## 6. User Roles & Permissions

| Feature | Super Admin | Ground Personnel |
|---------|-------------|-----------------|
| View all equipment | ✅ | ❌ (assigned categories only) |
| Update serviceability status | ✅ | ✅ (assigned only) |
| CRUD Categories / Models / Sub-components | ✅ | ❌ |
| Override auto-calculated status | ✅ | ❌ |
| Customise dashboard view | ✅ | ✅ (their slice) |
| Access Categories CRUD page | ✅ | ❌ |
| Manage users & assignments | ✅ | ❌ |

**Role switching in prototype:** A dropdown in the top bar simulates role switching. Ground Personnel roles filter the category dropdown and hide Categories nav item.

---

## 7. Design System

### Colours
| Token | Value | Usage |
|-------|-------|-------|
| `--md-bg` | `#0d0d0d` | Page background |
| `--md-surface` | `#141414` | Cards, nav bar |
| `--md-surface2` | `#1a1a1a` | Secondary surfaces, table header |
| `--md-primary` | `#00e5ff` | Cyan — active states, badges, links |
| `--s-green` | `#66bb6a` | Serviceable |
| `--s-yellow` | `#ffa726` | Limitations |
| `--s-grey` | `#78909c` | Under Maintenance |
| `--s-red` | `#ef5350` | Unserviceable |
| `--s-black` | `#9e9e9e` / dark bg | Attrited |

### Typography
- Body: Roboto
- Monospace (timestamps, badges, codes): Roboto Mono
- Icons: Material Icons (Google Fonts CDN — no npm required)

### Key UI Patterns
- All edit actions use the same pencil `icon-btn` (28px circle, no border, pencil icon)
- Status badges: coloured dot + label text, rounded rectangle
- Tier badges in table: `CAT` (cyan tint) / `MODEL` (grey tint) / `SUB` (dim tint)
- Toast notifications: slide up from bottom centre, auto-dismiss after 2.6s
- Modals: slide-scale in animation, backdrop blur overlay

---

## 8. High-Level Technical Stack Recommendations

| Layer | Recommendation | Rationale |
|-------|---------------|-----------|
| **Frontend** | React + TypeScript | Component-based; TSX files already produced for developer handoff |
| **Styling** | CSS Modules or styled-components (MUI tokens) | Matches the Material Design dark theme used in prototype |
| **Icons** | `@mui/icons-material` | Matches Material Icons used throughout prototype |
| **Backend** | Supabase (BaaS) | Auth + DB + realtime out of the box; appropriate for ~10 users |
| **Database** | PostgreSQL (via Supabase) | Relational structure suits 3-tier hierarchy |
| **Authentication** | Username + Password (Supabase Auth, JWT) | Simple, internal tool |
| **Hosting** | Vercel (frontend) + Supabase (backend/DB) | Easy deployment, generous free tiers |

---

## 9. Conceptual Data Model

```
Users
  id, name, username, password_hash, role (super_admin | ground_personnel),
  assigned_category_ids[]

Categories  (Tier 1)
  id, name, override_status?, created_at

Models  (Tier 2)
  id, category_id, name, override_status?, updated_at, created_at

SubComponents  (Tier 3)
  id, model_id, name, status, updated_at, created_at

Remarks
  id, subcomponent_id, remark_text, created_at, created_by (user_id)
```

**Notes:**
- `override_status` on Category and Model is nullable — `null` means "use auto-calculated (Worst Status Wins)"
- Status is only stored at the Sub-component level; all parent statuses are computed
- Remarks belong to Sub-components only — model/category level has no remarks

---

## 10. Security Considerations

- All passwords stored as hashed values (bcrypt — never plain text)
- RBAC enforced on both frontend and backend
- Ground Personnel API calls validated server-side — can only modify their assigned categories
- Sessions expire after inactivity (configurable timeout)
- Single organisation instance — no multi-tenancy required

---

## 11. Development Phases / Milestones

### Phase 0 — Lo-Fi Prototype ✅ COMPLETE
- [x] Summary Dashboard view with dummy data and expandable cards
- [x] Table/Detail view with tree-line hierarchy and tier badges
- [x] Persistent secondary bar: category filter + depth chips + view toggle
- [x] Status update modal with override support
- [x] Numbered defect remarks (mandatory for Limitations)
- [x] Worst Status Wins roll-up logic
- [x] Manual override with refresh icon indicator
- [x] Categories CRUD page (accordion, inline rename, confirm delete)
- [x] DSTA-style top status bar (THREATCON · DEFCON · live clock)
- [x] Collapsible sidebar drawer with role switcher
- [x] Role-based view filtering (prototype simulation)
- [x] Material Icons throughout (no emoji)
- [x] Legend housed under hoverable ⓘ icon (white tooltip)
- [x] TSX component files for developer handoff

### Phase 1 — MVP
- [ ] User authentication (login/logout, Supabase Auth)
- [ ] Role-based access enforced server-side
- [ ] Full 3-tier CRUD wired to live database
- [ ] Status updates persisted to database
- [ ] Worst Status Wins computed from live data
- [ ] Summary and Table views rendering live data
- [ ] Session-based dashboard filtering

### Phase 2 — Enhancement
- [ ] Manual override persisted to database
- [ ] Saved dashboard view templates / persistent preferences
- [ ] Basic audit log / timestamp history per sub-component
- [ ] Ground Personnel assignment management UI
- [ ] Notifications / alerts when equipment becomes unserviceable

### Phase 3 — Future Expansion
- [ ] Export to PDF/Excel for formal reporting
- [ ] Mobile-optimised responsive layout
- [ ] Advanced analytics (trend over time, recurring defects)
- [ ] Configurable roll-up rules
- [ ] QR code scanning for quick status updates
- [ ] Multi-unit / multi-organisation support

---

## 12. Potential Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Override vs auto-roll-up conflict | Override stored on entity; `null` = auto. Refresh icon indicates overridden state. |
| Stale status data | "Last updated" timestamp shown prominently on all sub-component rows |
| Ground personnel updating wrong equipment | Category assignment enforced in both UI (filter) and API (server-side validation) |
| Status roll-up rules undefined | "Worst Status Wins" as safe default; revisit in Phase 2 |
| Remarks numbering after deletions | Numbering derived from DOM count, not a running counter — always accurate |
| CSS sticky header breaking | `overflow: hidden` on table wrapper removed; thead sticky works via explicit `top` offset |

---

## 13. TSX Component Files (Developer Handoff)

| File | Description |
|------|-------------|
| `CategoriesPage.tsx` | Full CRUD page for managing the 3-tier hierarchy |
| `DashboardPage.tsx` | Main dashboard — summary cards, table view, filters, status modal |

Both files export fully typed components with shared `Category`, `Model`, `SubComponent`, and `ServiceabilityStatus` TypeScript interfaces. Data is passed via props + `onDataChange` callback — swap for API mutations when backend is ready.

---

*This document is a living blueprint — update it as requirements evolve.*
