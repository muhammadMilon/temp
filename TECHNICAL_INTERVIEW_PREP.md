# Nexora Studio — Technical Interview Preparation Handbook

> **Project:** Nexora Studio — SaaS drag-and-drop website builder  
> **Stack:** Next.js 16 (App Router), React 19, PostgreSQL, Prisma, NextAuth v5, Redux Toolkit, Stripe, Cloudinary, Tailwind CSS 4  
> **Purpose:** Interview-ready explanations tied to real architecture decisions in this codebase.

---

# 1. Project Overview & SaaS Architecture

## Easy Questions

### Q1: What is Nexora Studio and what problem does it solve?

**Answer:**

Nexora Studio is a **SaaS website builder** that lets users create production-ready websites without writing code from scratch. Users drag pre-built sections (hero, navbar, pricing, contact forms, etc.) onto a canvas, customize content and styles in a visual inspector, preview responsive layouts, and publish sites to a subdomain (with optional custom domain support).

The product solves three common pain points:

1. **Speed** — Starting from templates in PostgreSQL instead of blank HTML.
2. **Accessibility** — Non-developers can ship landing pages, portfolios, and SaaS marketing sites.
3. **Monetization** — Premium templates sold via Stripe with purchase records, invoices, and admin analytics.

Architecturally it is a **full-stack Next.js monolith** with server actions for mutations, Prisma for persistence, Redux for builder state, and edge-aware auth middleware. That keeps deployment simple (one app, one database) while still supporting marketplace, admin, and multi-tenant publishing.

---

### Q2: What are the main user roles and routes in the application?

**Answer:**

There are two primary roles defined in Prisma as `Role`: **user** and **admin**.

| Role | Typical routes | Capabilities |
|------|----------------|--------------|
| **user** | `/dashboard`, `/dashboard/projects/[slug]/builder`, `/dashboard/purchases`, `/templates` | Create projects, use builder, buy templates, publish sites |
| **admin** | `/admin`, `/admin/users`, `/admin/templates`, `/admin/transactions` | Platform analytics, user CRUD, block/unblock, soft delete |

Public routes include `/`, `/templates`, `/login`, `/register`, and published sites at `/p/[subdomain]`.

Auth is enforced in `proxy.js` (Next.js 16 middleware pattern) with logic in `lib/auth-callbacks.js`: unauthenticated users cannot access `/dashboard/*`; non-admins are redirected away from `/admin/*`; blocked users are sent to `/blocked`.

After admin login, credentials and Google OAuth flows redirect to **`/admin`** instead of the generic dashboard — a product decision implemented in `LoginForm.jsx` and `auth-callbacks.js`.

---

## Medium Questions

### Q1: Why did you choose a monolithic Next.js app instead of splitting frontend and backend microservices?

**Answer:**

For Nexora Studio’s current scale and team size, a **monolith with clear module boundaries** is the right trade-off.

**Reasons for monolith:**

- **Server Actions** (`"use server"` in `actions/*.js`) colocate business logic with the UI that triggers it — fewer network hops, simpler auth (session is already on the server).
- **Shared types and validation** — Zod schemas in `lib/validations.js` can be reused conceptually across forms and server handlers.
- **Faster iteration** — Builder, billing, and admin ship in one deploy; no API versioning drift between separate repos.
- **Lower ops cost** — One Vercel (or similar) deployment, one PostgreSQL instance, one Stripe webhook URL.

**Where boundaries still exist (future extraction points):**

- **Stripe webhooks** (`/api/webhooks/stripe`) — already isolated as an API route.
- **Invoice PDF generation** (`/api/invoices/[purchaseId]`) — could move to a worker service if CPU-heavy.
- **Published site rendering** (`/p/[subdomain]`) — could become a CDN-backed static edge later.

If traffic grew 10–100×, I would extract **webhook processing**, **PDF generation**, and **publish snapshot rendering** first — not the entire builder UI. The monolith stays until those paths prove they need independent scaling.

---

### Q2: How does data flow from a user editing the builder to a visitor seeing the published site?

**Answer:**

The flow has distinct phases: **edit (client)**, **persist (server)**, **publish (snapshot)**, **serve (public route)**.

1. **Edit** — User manipulates sections in `BuilderClient.jsx`. Redux (`builderSlice`) holds `document.sections`, selection, undo history, and `isDirty`.

2. **Persist** — Saving calls server actions (e.g. project/canvas updates in `actions/projects.js`) which write `Project.canvasData` or `Page.canvasData` as **JSON** in PostgreSQL via Prisma.

3. **Publish** — `publishProject` in `actions/publish.js` builds a **snapshot** via `buildPublishSnapshot()`: merges ordered pages, picks home page, copies SEO fields, and stores everything in `PublishedWebsite.snapshotData` with a unique `subdomain`.

4. **Serve** — Visitors hit `/p/[subdomain]`. The page loads `PublishedWebsite` by subdomain and renders sections through the same `SectionRenderer` used in preview — ensuring WYSIWYG consistency.

This design separates **mutable draft state** (project/pages) from **immutable published artifact** (snapshot). Republishing overwrites the snapshot without breaking in-progress edits in the builder.

---

## Hard Questions

### Q1: How would you evolve Nexora Studio from a single-tenant monolith to a multi-region SaaS serving millions of published sites?

**Answer:**

Today publishing stores a JSON snapshot and serves it dynamically from Next.js. At scale, that path becomes a **read-heavy, globally distributed** problem. A phased evolution:

**Phase 1 — Read path optimization (low risk)**

- On publish, **pre-render static HTML** (already partially supported via `utils/exportCode.js`) and upload to object storage (S3/R2) + CloudFront.
- `/p/[subdomain]` becomes a redirect or edge function that resolves subdomain → CDN URL.
- PostgreSQL remains source of truth; CDN is a cache layer with TTL invalidation on republish.

**Phase 2 — Write path isolation**

- Builder saves stay on primary DB region; use **debounced autosave** and optimistic UI to reduce write QPS.
- `CanvasRevision` table already supports version history — cap retention and archive old revisions to cold storage.

**Phase 3 — Multi-region**

- **Users/projects** — single primary region with read replicas for dashboard analytics (admin charts already aggregate in `actions/admin.js`).
- **Published sites** — geo-replicated object storage; custom domains via DNS to nearest PoP.
- **Stripe webhooks** — idempotent handlers (`fulfillTemplatePurchase` keyed by `stripeSessionId`) with a deduplication table to survive duplicate events across regions.

**Phase 4 — Tenant isolation & quotas**

- Per-user project limits, media storage quotas (Cloudinary already tracks `MediaAsset`).
- Rate limit form submissions (`FormSubmission`) per project/IP hash.

**Observability:** structured logs on server actions, webhook failures, publish latency; SLOs on publish time and TTFB for `/p/*`.

The key interview point: **don’t shard the builder on day one** — optimize the published read path first because that’s what scales with end-user traffic, not editor sessions.

---

### Q2: What are the biggest architectural risks in a drag-and-drop builder SaaS, and how does your codebase mitigate them?

**Answer:**

| Risk | Impact | Mitigation in Nexora Studio |
|------|--------|----------------------------|
| **Schema-less JSON canvas** | Breaking changes to section shape | `componentRegistry.js` + `createSection()` factory; version field on document; revisions in `CanvasRevision` |
| **Stale Prisma client in dev** | Runtime `findMany` undefined | `lib/prisma.js` refreshes client when delegates missing; `lib/prisma-models.js` safe accessors |
| **Double Stripe fulfillment** | Duplicate purchases | Unique `stripeSessionId`; lookup before create in `fulfillTemplatePurchase` |
| **Auth bypass on admin** | Data breach | `authorized` callback enforces `role === "admin"`; server actions re-check `session.user.id` |
| **XSS in user-generated content** | Published site attacks | Export/preview should escape user strings; forms submit server-side with validation |
| **Large JSON documents** | Slow saves, DB bloat | Page-level `canvasData` split; consider compressing or diff-based saves later |

**Production mindset:** every server action uses try/catch and returns `{ ok, error }` instead of throwing to the client; list endpoints return `[]` on failure (e.g. `listSavedBlocks` in `actions/blocks.js`) so the UI never hard-crashes on a missing model delegate.

Hard questions in interviews are about **knowing what breaks at scale** — this table shows you’ve already hit real issues (Prisma stale client, pending purchase UI) and fixed them with defensive patterns.

---

# 2. Next.js App Router & Server Architecture

## Easy Questions

### Q1: Why did you use the App Router instead of the Pages Router?

**Answer:**

Next.js App Router fits Nexora Studio because:

1. **Layouts** — `app/dashboard/layout.js` and `app/admin/layout.js` wrap nested routes with shared sidebars without re-mounting chrome on every navigation.

2. **Server Components by default** — Admin overview and template listing pages fetch Prisma data on the server, sending less JavaScript to the client. Interactive pieces (builder, charts) are isolated as `"use client"` components.

3. **Colocated routing** — `app/dashboard/projects/[slug]/builder/page.js` maps cleanly to product URLs users understand.

4. **Server Actions** — Mutations like `publishProject`, `createTemplateCheckout`, and `adminCreateUser` live beside the feature without maintaining a separate REST API layer.

5. **Next.js 16 conventions** — `proxy.js` replaces legacy middleware naming while keeping edge auth compatible with Auth.js.

The Pages Router would still work, but App Router reduces boilerplate for a dashboard-heavy SaaS with mixed server/client surfaces.

---

### Q2: What is the difference between a Server Component, Client Component, and Server Action in your project?

**Answer:**

| Pattern | Example in codebase | Runs where | Use case |
|---------|---------------------|------------|----------|
| **Server Component** | `app/admin/page.js`, `app/templates/page.js` | Server | Fetch users, stats, templates; no hooks |
| **Client Component** | `BuilderClient.jsx`, `TemplatesClient.jsx` | Browser | DnD, Redux, toasts, Recharts |
| **Server Action** | `actions/billing.js`, `actions/projects.js` | Server (RPC from client) | DB writes, Stripe session creation |

**Rule of thumb used in Nexora Studio:**

- Start as Server Component.
- Add `"use client"` only when you need state, effects, browser APIs, or event handlers.
- Put mutations in `actions/*.js` with `"use server"` — never expose raw Prisma to the client.

Example: `/templates` page server-loads `listPublicTemplates()`, passes data to `TemplatesClient` for filters and checkout — **data on server, interactivity on client**.

---

## Medium Questions

### Q1: How do you handle caching and revalidation after mutations?

**Answer:**

Next.js caches Server Component renders by default. After mutations, Nexora Studio calls **`revalidatePath()`** from server actions to invalidate stale UI.

Examples:

- `fulfillTemplatePurchase` revalidates `/dashboard/purchases` and `/dashboard/templates` so owned templates unlock immediately.
- `publishProject` revalidates project settings and public routes after subdomain assignment.
- Block save in `actions/blocks.js` revalidates the builder path for that project slug.

**What we don’t rely on:** long-lived client-side cache of server data for critical auth/billing state — `router.refresh()` after login redirects ensures session role (admin vs user) is current.

**Interview tip:** Mention **tag-based revalidation** (`revalidateTag`) as a future improvement if many pages depend on the same template catalog — today path-based revalidation is explicit and easier to reason about for a mid-size app.

---

### Q2: Explain the role of `proxy.js` in Next.js 16 and how auth integrates with it.

**Answer:**

In Next.js 16, **`proxy.js`** is the entry for request interception (successor pattern to `middleware.js`). Nexora Studio exports:

```js
export default async function proxy(request, context) {
  return edgeAuth(request, context);
}
```

with `matcher` covering `/dashboard/*`, `/admin/*`, `/login`, `/register`, `/blocked`.

Auth.js v5 integrates via the **`authorized` callback** in `lib/auth-callbacks.js` rather than wrapping custom logic inside `auth((req) => ...)`, which can break under the proxy model.

**What `authorized` does:**

- Blocks `/dashboard` and `/admin` for unauthenticated users.
- Redirects non-admins from `/admin` to `/dashboard`.
- Redirects logged-in users away from login/register (admins → `/admin`).
- Sends `blocked` users to `/blocked`.

**Edge vs Node:** `lib/auth.edge.js` powers proxy; full Prisma adapter runs on Node for credentials login. JWT/session callbacks copy `role` and `blocked` into the token so edge checks don’t need a DB round-trip per request.

---

## Hard Questions

### Q1: How would you structure Nexora Studio if you needed real-time collaboration (multiple users editing one project)?

**Answer:**

Current architecture is **single-user optimistic**: Redux holds local state; saves overwrite server JSON. Collaboration requires **operational transformation (OT)** or **CRDTs** plus a sync layer.

**Recommended approach:**

1. **Authority model** — PostgreSQL remains canonical; introduce a `ProjectCollaborator` table (userId, projectId, role: editor/viewer).

2. **Sync transport** — WebSocket or WebRTC room per `projectId` (Liveblocks, Partykit, or custom Socket.io). Broadcast section-level patches, not full document on every keystroke.

3. **Conflict resolution** — CRDT for section list order (Yjs `Y.Array`); LWW (last-write-wins) for simple text props with timestamps; lock inspector fields while another user edits the same section id.

4. **Persistence** — Debounce merged state to `canvasData` every N seconds; store periodic `CanvasRevision` entries for audit (already in schema).

5. **Next.js integration** — Builder page becomes client-only with a `CollaborationProvider`; Server Actions only for permissions and initial hydrate.

**Why not Server Actions alone:** they are request/response, not streaming duplex — fine for saves, wrong for 60fps cursor presence.

**Migration path:** ship **read-only share preview** first (lower risk), then two-editor beta with feature flag per project.

---

### Q2: What trade-offs exist between Server Actions and REST/GraphQL APIs for this codebase?

**Answer:**

| Dimension | Server Actions (current) | REST/GraphQL |
|-----------|--------------------------|--------------|
| **Auth** | Session automatic | Must attach cookies/tokens manually |
| **Type safety** | Informal `{ ok, data, error }` | OpenAPI/GraphQL schema |
| **Mobile/third-party** | Poor fit | Required for native apps |
| **Caching** | Implicit RSC cache | HTTP cache headers explicit |
| **Rate limiting** | Per-action middleware needed | Standard API gateway |
| **Testing** | Invoke functions in integration tests | Supertest/contract tests |

**Stay on Server Actions while:** the only client is your Next.js app and team velocity matters most.

**Extract APIs when:** you need public webhooks beyond Stripe, partner integrations, or a mobile app consuming the same backend.

**Hybrid pattern for Nexora Studio today:** Server Actions for dashboard mutations; **Route Handlers** for Stripe webhooks (`/api/webhooks/stripe`), invoice PDF download, and template ZIP export — because those need raw `Request`/`Response` (streams, binary, signature verification).

---

# 3. Authentication & Authorization (NextAuth v5)

## Easy Questions

### Q1: How does user authentication work in Nexora Studio?

**Answer:**

Authentication uses **NextAuth.js v5** (`next-auth` beta) with the **Prisma adapter** for account/session storage.

Supported flows:

1. **Credentials** — Email + bcrypt-hashed password (`bcryptjs`); validated with Zod in `lib/validations.js`.
2. **Google OAuth** — Optional; post-login redirect via `/api/auth/post-login` to respect `callbackUrl` and admin routing.
3. **Email verification** — `EmailVerificationToken` model; tokens created/consumed in `actions/auth.js` with safe fallbacks via `lib/db-tokens.js`.
4. **Password reset** — `PasswordResetToken` with expiry; `ForgotPasswordForm` / `ResetPasswordForm`.

Sessions use JWT strategy with callbacks in `lib/auth-callbacks.js` that attach **`id`**, **`role`**, and **`blocked`** to `session.user`.

Protected routes are enforced at the edge through `proxy.js` before page code runs — defense in depth with server actions checking `auth()` again.

---

### Q2: How do you prevent a normal user from accessing the admin panel?

**Answer:**

**Two layers:**

1. **Edge (`authorized` in `lib/auth-callbacks.js`)** — If path starts with `/admin` and `auth.user.role !== "admin"`, redirect to `/dashboard`. If no session, return `false` (Auth.js redirects to login).

2. **Server actions (`actions/admin.js`)** — Every admin function should call a shared guard (e.g. `requireAdmin()`) that verifies `session.user.role === "admin"` before Prisma queries. UI hiding alone is not security.

3. **Login redirect** — After credentials login, `LoginForm` reads session and sends admins to `/admin`, not `/dashboard`.

**Blocked users:** `blockedAt` on `User` sets `token.blocked`; middleware redirects to `/blocked` for dashboard/admin paths.

Interview answer: **never trust client-side role checks**; middleware + server action + DB role column must align.

---

## Medium Questions

### Q1: Why use JWT sessions with NextAuth instead of only database sessions?

**Answer:**

Nexora Studio uses JWT-enriched sessions because **edge middleware must authorize without a database call on every request**.

**Flow:**

- On login, `jwt` callback stores `id`, `role`, `blocked` in the token.
- `session` callback copies those into `session.user` for client components.
- `proxy.js` / `authorized` reads `auth.user.role` from that token at the edge.

**Trade-off:** JWT fields can become **stale** if an admin blocks a user or changes role until token refresh. Mitigations:

- Short session max age in Auth config.
- Critical server actions re-query `User` from DB for `blockedAt` and `role` when deleting users or processing payments.
- Force `signOut` / token refresh on role change (admin panel after block).

Database sessions alone would require edge-incompatible Prisma calls or a separate session validation service — acceptable at scale, heavier for current architecture.

---

### Q2: Walk through the email verification and password reset flows securely.

**Answer:**

**Email verification:**

1. On register, create user (unverified) and insert `EmailVerificationToken` with random token + expiry.
2. Send link to `/verify-email?token=...` (implementation in auth actions).
3. On submit, action looks up token, checks `expires > now`, marks `emailVerified`, deletes token rows for user.
4. `lib/db-tokens.js` wraps Prisma delegates — if model missing (stale client), returns graceful failure instead of crashing.

**Password reset:**

1. `ForgotPasswordForm` always returns generic success message (no email enumeration).
2. Create `PasswordResetToken` with short TTL (e.g. 1 hour).
3. Reset page validates token before showing password form.
4. On success, bcrypt-hash new password, delete all reset tokens for user.

**Security practices:**

- Tokens are **opaque random strings**, not user ids.
- **Single-use** — delete on consume.
- **Hashed passwords only** — never store plaintext.
- Rate limit forgot-password endpoint in production (not always in dev).

---

## Hard Questions

### Q1: How would you implement organization/team accounts (B2B) on top of the current User model?

**Answer:**

Current schema is **user-owned projects** (`Project.userId`). B2B needs an **Organization** layer:

**Schema additions:**

```
Organization (id, name, slug, plan)
OrganizationMember (orgId, userId, role: owner|admin|member)
Project.orgId (nullable during migration)
```

**Auth changes:**

- JWT carries `activeOrgId` and `orgRole`.
- Middleware checks org membership for `/dashboard/projects/*`.
- Invite flow: `OrganizationInvite` token similar to email verification.

**Billing:** Stripe Customer per organization, not per user; seat-based pricing.

**Builder permissions:** Only `editor+` can mutate `canvasData`; `viewer` gets read-only preview route.

**Migration:** Backfill each existing user with a personal org; set `Project.orgId` from owner.

**Interview point:** This is a **data model and authorization** change more than a UI change — every server action that does `where: { userId }` becomes `where: { orgId, ...membership check }`.

---

### Q2: What OWASP-style threats apply to a website builder, and how do you defend against them?

**Answer:**

| Threat | Vector in builder SaaS | Defense |
|--------|------------------------|---------|
| **Broken access control** | Guessing project UUID/slug | All queries scoped by `session.user.id`; admin guarded separately |
| **XSS** | User text in headings, rich text | Escape on export; sanitize HTML if allowing rich text; CSP on published sites |
| **CSRF** | Server Actions from malicious site | SameSite cookies; Origin check in production |
| **SSRF** | Custom domain verification fetching user URLs | Validate URLs; allowlist protocols |
| **Injection** | Form submissions, search | Prisma parameterized queries; Zod validation |
| **Sensitive data exposure** | Invoice PDF, purchase API | `/api/invoices/[id]` verifies session + `status: succeeded` |
| **Stripe webhook forgery** | Fake payment events | Verify signature with `stripe.webhooks.constructEvent` |

**Published sites** are untrusted content containers — treat them like user-generated static hosting: subdomains on separate cookie domain, optional sandboxed iframe preview, monitor for phishing reports.

---

# 4. Database Design with Prisma & PostgreSQL

## Easy Questions

### Q1: Why PostgreSQL and Prisma for this project?

**Answer:**

**PostgreSQL** fits because:

- **Relational integrity** — Users own projects; projects have pages; purchases reference templates and users with foreign keys.
- **JSON support** — `canvasData`, `snapshotData`, and `sectionData` are stored as `Json` columns for flexible builder documents without a separate document DB early on.
- **Enums** — `Role`, `PurchaseStatus`, `ProjectStatus` are enforced at DB level.
- **Indexes** — Admin analytics and listings use indexes on `userId`, `status`, `category`, `createdAt`.

**Prisma** provides:

- Type-safe queries in server actions.
- Migrations (`prisma migrate`) and `db push` for dev.
- Generated client with relation includes (`project include: { pages }`).

For a SaaS MVP, one Postgres instance is simpler than Postgres + MongoDB until JSON query patterns dominate.

---

### Q2: What is stored inside `canvasData` and why is it JSON?

**Answer:**

`canvasData` on `Project` and `Page` is a **builder document**, typically:

```json
{
  "version": 1,
  "sections": [
    { "id": "...", "type": "hero", "props": { "title": "...", ... } }
  }
}
```

Sections are created from `createSection(type)` in `componentRegistry.js` with default props from `componentLibrary.js`.

**Why JSON:**

- Section types evolve (new props, new components) without `ALTER TABLE` per field.
- Drag-and-drop UIs naturally produce nested trees.
- Export (`exportCode.js`) and preview (`SectionRenderer`) consume the same structure.

**Downside:** Harder to SQL-query “all projects using pricing v2.” Mitigation: store `version` on document; optional `ComponentDefinition` table for catalog metadata.

---

## Medium Questions

### Q1: Explain the relationship between Project, Page, PublishedWebsite, and Template.

**Answer:**

```
User
 ├── Project (draft workspace, canvasData, pages[])
 │     ├── Page (multi-page sites, per-page canvasData)
 │     ├── CanvasRevision (history snapshots)
 │     ├── SavedBlock (reusable sections)
 │     └── PublishedWebsite (1:1 when published)
 └── TemplatePurchase → Template
```

- **Project** — User’s editable site. Has slug unique per user. Status `draft` | `published`.
- **Page** — Optional multi-page support; each page has its own `canvasData` and SEO fields (`metaTitle`, `metaDescription`).
- **PublishedWebsite** — Public artifact: `subdomain`, optional `customDomain`, `snapshotData` (frozen copy), `domainVerified` flag.
- **Template** — Marketplace catalog entry with `canvasData`, `isPremium`, `priceCents`. Buying creates **TemplatePurchase** (`status: succeeded`).

**Creating project from template:** copies template `canvasData` into new `Project` — user then edits independently.

**Publishing:** does not mutate template; writes snapshot from project pages.

---

### Q2: How does soft delete work for users, and why prefer it over hard delete?

**Answer:**

`User.deletedAt` is a nullable timestamp set by `adminSoftDeleteUser` in `actions/admin.js`.

**Benefits:**

- **Audit** — Admin can see historical purchases and transactions tied to user id.
- **Legal** — Retain billing records; Stripe may require purchase history.
- **Recovery** — Undo mistaken deletion by clearing `deletedAt`.
- **Referential integrity** — Avoid cascade-deleting projects that might need export.

**Implementation pattern:**

- List users: `where: { deletedAt: null }`.
- Login: reject if `deletedAt` is set (add check in credentials provider).
- Unique email constraint: either keep email on soft-deleted row or anonymize email to `deleted+<id>@...` if re-registration must be allowed.

Hard delete is reserved for GDPR “right to erasure” after retention period — separate admin tooling.

---

## Hard Questions

### Q1: The canvas JSON column will grow large. How do you optimize storage, queries, and save performance?

**Answer:**

**Problems at scale:**

- Multi-MB JSON per project slows row updates and backups.
- Autosave every few seconds creates write amplification.
- Loading full document for dashboard list is wasteful.

**Optimizations:**

1. **Split hot/cold data** — Dashboard lists select `id, name, slug, thumbnail, updatedAt` only — never `canvasData`.

2. **Page-level documents** — Already split across `Page.canvasData`; save only active page on autosave.

3. **Diff-based saves** — Client sends patch `{ sectionId, propsDelta }` instead of full JSON; server merges server-side (requires version counter for conflict detection).

4. **Compression** — Store gzip blob in bytea or use Postgres JSONB + TOAST (automatic for large values) — monitor table bloat.

5. **Revision retention** — `CanvasRevision` capped (builder Redux already caps undo at 50); cron job deletes revisions older than 90 days.

6. **Read replicas** — Analytics queries in admin hit replica; writes stay on primary.

7. **Archival** — Inactive projects moved to `archived` status with canvas in object storage.

**Interview sound bite:** Optimize **what you SELECT** before sharding JSON into MongoDB.

---

### Q2: Design indexes and query patterns for admin analytics (revenue, user growth, template usage).

**Answer:**

Admin dashboard (`actions/admin.js`) aggregates:

- Total users, revenue, purchases, templates.
- Time-series: user growth, revenue by day, purchase distribution, template usage.

**Index strategy:**

| Query | Suggested index |
|-------|-----------------|
| Purchases by date/status | `TemplatePurchase(status, createdAt)` |
| Users over time | `User(createdAt)` |
| Revenue sums | `TemplatePurchase(status, amountCents)` partial where status = succeeded |
| Template popularity | `TemplatePurchase(templateId, status)` |

**Query patterns:**

- Use `groupBy` with `createdAt` truncated to day (`date_trunc` via `$queryRaw` or Prisma groupBy on pre-bucketed dates).
- Cache dashboard cards for 60s in production (`unstable_cache` with tag `admin-stats`).
- For heavy charts, materialized view `daily_revenue` refreshed nightly.

**Avoid:** loading all purchases into Node and reducing in JS — push aggregation to SQL.

---

# 5. Drag-and-Drop Builder System

## Easy Questions

### Q1: What libraries power drag-and-drop in the builder?

**Answer:**

Nexora Studio uses **@dnd-kit** (`@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`).

**Why dnd-kit over react-beautiful-dnd:**

- Actively maintained and compatible with **React 19**.
- Flexible sensors (pointer, keyboard) for accessibility.
- Works well with custom drag overlays (`SortableSection.jsx`).

**Flow:**

- Palette items drag new section types onto canvas.
- `SortableContext` reorders existing sections.
- On drop, Redux actions `addSection` or `insertSectionAt` update `document.sections`.
- `isDirty` flags unsaved changes until server persist.

DnD only handles **structure**; content editing happens in `BuilderInspector` via controlled fields.

---

### Q2: What is a "section" in your builder architecture?

**Answer:**

A **section** is the top-level building block — one row on the canvas (hero, navbar, footer, pricing, FAQ, contact, etc.).

Each section is an object:

- `id` — stable key (cuid from `createId()`).
- `type` — string key matching `componentRegistry`.
- `props` — serializable content/styles (text, colors, links, image URLs).

`SectionRenderer.jsx` maps `type` → React component for preview and published output.

`componentLibrary.js` defines palette metadata (label, icon, category).

**Adding a new section type:** register defaults in registry, add renderer component, optionally seed `ComponentDefinition` in DB for dynamic palette (schema supports this).

---

## Medium Questions

### Q1: How does undo/redo work in the builder?

**Answer:**

Implemented in **`redux/slices/builderSlice.js`** with two stacks:

- `past` — snapshots of `{ projectId, projectName, viewport, document }` before each mutating action.
- `future` — cleared when a new mutation happens after undo.

`pushHistory()` clones state via `JSON.parse(JSON.stringify(...))` — simple deep copy, acceptable for document size at current scale.

**Limit:** `HISTORY_LIMIT = 50` — oldest entry dropped to cap memory.

**Undo/redo reducers** pop from `past`/`future` and restore document.

**What’s not in undo:** server-side revisions (`CanvasRevision` from `actions/history.js`) are separate — long-term snapshots user can restore from history panel.

**Production improvement:** Immer patches or structural sharing to reduce memory; merge rapid typing into single history entry via debounced `pushHistory`.

---

### Q2: How do preview mode and responsive viewport switching work?

**Answer:**

**Viewport** — Redux `viewport` state (`desktop` | `tablet` | `mobile`). `BuilderViewportContext` applies CSS width constraints to the canvas wrapper so users see approximate breakpoints.

**Preview mode** — `previewMode` in builder slice toggles `BuilderPreviewMode.jsx`: hides chrome, renders clean `SectionRenderer` output similar to published site.

**Live preview route** — `/dashboard/projects/[slug]/preview` server-loads project and renders without edit handles — good for sharing work-in-progress before publish.

**Consistency principle:** Same `SectionRenderer` for builder preview, modal template preview, and published `/p/[subdomain]` — reduces “looks different live” bugs.

---

## Hard Questions

### Q1: How would you implement a plugin system so third parties can add custom section types safely?

**Answer:**

**Goals:** sandbox untrusted code, stable API, versioned contracts.

**Architecture:**

1. **Manifest-based plugins** — JSON manifest: `{ key, version, propsSchema, entryUrl }` stored in DB after admin review.

2. **Props schema** — JSON Schema validated on save (extend `ComponentDefinition.schema`).

3. **Rendering options:**
   - **Safe:** Only allow server-reviewed React components bundled at build time (curated marketplace).
   - **Sandboxed:** iframe sections with `sandbox` attribute and `postMessage` for height resize — no raw script in parent page.
   - **Avoid:** `eval(userJs)` in main builder thread.

4. **Runtime loading** — Dynamic `import()` from CDN with Subresource Integrity hash; CSP `script-src` allowlist.

5. **Export** — `exportCode.js` must know how to serialize plugin sections to HTML or embed iframe snippet.

**Phased rollout:** internal “custom HTML” section with sanitization first; external plugins later with review queue in admin.

---

### Q2: How do you keep builder performance smooth with 50+ sections and heavy images?

**Answer:**

**Client:**

- **Virtualize** section list if >30 visible blocks (react-window) — only mount DOM for viewport ± buffer.
- **Memoize** `SectionRenderer` with `React.memo` keyed by section id + props hash.
- **Debounce** inspector updates (300ms) before Redux commit for text fields.
- **Lazy load** images with `loading="lazy"`; Cloudinary transformations for thumbnails (`w_400,q_auto`).

**Network:**

- Autosave debounced 2–5s; abort in-flight save if newer edit exists.
- Send diffs instead of full JSON when implemented.

**Server:**

- Don’t revalidate entire site on every autosave — only builder path or skip revalidation until explicit save.

**Measure:** React Profiler + Lighthouse on preview route; track Time to Interactive on builder page.

---

# 6. State Management (Redux Toolkit)

## Easy Questions

### Q1: Why use Redux for the builder instead of only React useState?

**Answer:**

The builder has **cross-cutting state** that many distant components need:

- Current document (sections array).
- Selected section id (canvas + inspector).
- Undo/redo stacks.
- Viewport and preview mode.
- Dirty flag for save warnings.

Prop drilling through `BuilderClient` → canvas → section → inspector becomes unmaintainable.

**Redux Toolkit** provides:

- Predictable reducers in `builderSlice.js`.
- `useAppDispatch` / `useAppSelector` hooks in `hooks/useRedux.js`.
- DevTools for debugging complex drag sequences.

**What Redux is NOT used for:** server data like user profile or template catalog — those come from Server Components / props and `router.refresh()`.

---

### Q2: What happens when the builder page loads — how is Redux initialized?

**Answer:**

1. Server Component `builder/page.js` fetches project (and pages) from Prisma with `auth()` ownership check.

2. Passes `canvasData`, `projectId`, `viewport`, etc. as props to **`BuilderClient`** (client).

3. On mount, `BuilderClient` dispatches **`hydrateFromServer`** with payload.

4. `hydrateFromServer` resets `past`/`future`, sets `document` from `canvasData.sections` or `emptyCanvasDocument()`, clears `isDirty`.

5. User edits mark `isDirty: true`; save action persists back to Prisma and clears dirty state on success.

**Key point:** Redux is ephemeral per tab; **PostgreSQL is source of truth** after save.

---

## Medium Questions

### Q1: When would you choose React Context or Zustand instead of Redux here?

**Answer:**

| Tool | Fit for Nexora builder |
|------|------------------------|
| **Context** | Theme, viewport — already partially in `BuilderViewportContext`; avoid large document in context (re-render all consumers). |
| **Zustand** | Could replace Redux with less boilerplate; undo history still manual. |
| **Redux** | Strong fit for undo stacks, middleware (autosave logger), time-travel debugging. |

**Switching cost:** ~20+ reducers in builderSlice — migration only worth it if bundle size or boilerplate hurts metrics.

**Recommendation in interview:** Redux was chosen for **undo history + DevTools**, not because other tools can’t work.

---

### Q2: How do you prevent losing work if the user closes the tab?

**Answer:**

**Current mechanisms:**

- `isDirty` flag — warn with `beforeunload` event when true (implement in BuilderClient if not already).
- Explicit save button / autosave server action on interval.

**Enhancements:**

- **localStorage backup** — debounced write of `document` keyed by `projectId`; on hydrate, offer “restore unsaved local changes?” if newer than server `updatedAt`.
- **CanvasRevision** — server-side snapshots on milestone saves (publish, manual “Save version”).
- **Optimistic UI** — show “Saving…” / “Saved” toast (sonner).

**Conflict handling:** If server `updatedAt` > local backup timestamp, prefer server unless user chooses recovery.

---

## Hard Questions

### Q1: Design a sync strategy between Redux state and server `canvasData` when saves can fail or race.

**Answer:**

**States:** `idle | saving | saved | error | conflict`

**Algorithm:**

1. Client assigns monotonic `revision` number client-side on each successful server ack (server stores `project.revision` integer).

2. On save, send `{ revision, canvasData }`. Server uses transaction:

   ```sql
   UPDATE project SET canvasData = $1, revision = revision + 1
   WHERE id = $2 AND revision = $3
   RETURNING revision
   ```

3. If 0 rows updated → **conflict** — fetch latest, show merge UI or force overwrite with admin permission.

4. While `saving`, queue edits in Redux; on success, apply ack; on failure, retry exponential backoff (max 3).

5. Webhook-style **last-write-wins** only for single-user mode; unacceptable for collaboration.

**Nexora today:** simpler overwrite — document this as known limitation and path to revision column.

---

# 7. Template Marketplace & Stripe Payments

## Easy Questions

### Q1: How does a user purchase a premium template?

**Answer:**

1. User browses `/templates` or `/dashboard/templates` — data from `listPublicTemplates()` / marketplace actions with `owned` flag from `TemplatePurchase` where `status: succeeded`.

2. Clicks **Buy** on locked premium template → `createTemplateCheckout(templateId)` in `actions/billing.js`.

3. Server creates Stripe Checkout Session with template price, success URL includes `session_id`, cancel URL returns to templates.

4. User pays on Stripe hosted page.

5. **Fulfillment** happens via:
   - Stripe webhook `checkout.session.completed` → `fulfillTemplatePurchase`.
   - Success redirect → `completeCheckoutFromSession(sessionId)` as backup.

6. Purchase row set to `status: succeeded`; UI revalidates; template shows **Owned**.

7. User dashboard **Purchases** lists only succeeded purchases with invoice PDF link.

**No pending UI** — users don’t see “Awaiting confirmation” states in production dashboard.

---

### Q2: What data is stored in `TemplatePurchase`?

**Answer:**

| Field | Purpose |
|-------|---------|
| `userId` | Buyer |
| `templateId` | Product |
| `stripeSessionId` | Idempotency / webhook matching |
| `status` | `pending` → `succeeded` / `failed` / `refunded` |
| `amountCents`, `currency` | Invoice and analytics |

Unique constraint on `stripeSessionId` prevents duplicate rows from webhook + client callback racing.

Admin transactions page aggregates succeeded purchases for revenue charts.

---

## Medium Questions

### Q1: Why fulfill purchases in both the webhook and the success redirect callback?

**Answer:**

**Webhook is source of truth** — fires even if user closes browser after payment.

**Success callback (`completeCheckoutFromSession`)** improves UX:

- User lands on `/dashboard/purchases?paid=1&session_id=...` and sees purchase immediately.
- Handles dev environments where Stripe CLI webhook isn’t running.

**Idempotency:** `fulfillTemplatePurchase` looks up by `stripeSessionId`, existing `purchaseId`, or latest pending row for user+template — then **updates** instead of creating duplicates.

**Interview line:** “Never rely only on client return URL — always implement webhooks for money.”

---

### Q2: How are invoices generated and secured?

**Answer:**

**Generation:** `lib/invoice-pdf.js` uses **pdfkit** server-side to render PDF with user name, template name, price, date, invoice id (purchase id).

**Route:** `GET /api/invoices/[purchaseId]`:

1. `auth()` — must be logged in.
2. Load purchase where `id`, `userId` match session, `status === succeeded`.
3. Stream PDF response with `Content-Disposition: attachment`.

**Security:**

- Users cannot download other users’ invoices by guessing cuid (ownership check).
- Pending/failed purchases return 404 — no invoice before payment.

**Scale:** If CPU heavy, queue PDF job and store in S3; return signed URL.

---

## Hard Questions

### Q1: Design a refund and chargeback flow for template purchases.

**Answer:**

1. **Stripe Dashboard / API** — `refunds.create` on PaymentIntent tied to Checkout Session.

2. **Webhook** — listen for `charge.refunded` → set `TemplatePurchase.status = refunded`.

3. **Entitlement** — remove template from “owned” logic: `owned` only if succeeded and not refunded.

4. **Projects already created** — policy decision: allow keep vs disable export; store `grantedAt` snapshot on project create.

5. **Admin UI** — transaction detail with refund button (role-guarded).

6. **Chargebacks** — `charge.dispute.created` alerts admin; auto-flag user if abuse pattern.

7. **Analytics** — revenue charts subtract refunded amounts by day.

**Legal:** update Terms; invoice PDF marked “REFUNDED” if re-downloaded.

---

### Q2: How would you support subscriptions (Pro plan) alongside one-time template purchases?

**Answer:**

**Stripe Products:**

- One-time Prices — current templates.
- Recurring Price — “Nexora Pro” monthly/yearly.

**Schema:**

```
Subscription (userId, stripeSubscriptionId, status, currentPeriodEnd)
```

**Feature gating:**

- Free: N projects, watermark, basic templates.
- Pro: unlimited projects, premium templates included, custom domain.

**Implementation:**

- Checkout mode `subscription` for upgrade; Customer Portal for cancel.
- Webhooks: `customer.subscription.updated`, `deleted` → update DB entitlements.
- Middleware or server action `requirePro()` for premium features.

**Templates:** Either bundle all premium for Pro or keep à la carte — business decision affecting `listPublicTemplates` ownership logic (`owned` if purchased OR subscription active).

---

# 8. Publishing, Subdomains & Custom Domains

## Easy Questions

### Q1: What happens when a user clicks "Publish"?

**Answer:**

`publishProject` in `actions/publish.js`:

1. Verifies session owns the project.
2. Loads all `Page` records ordered by `sortOrder`.
3. Builds **snapshot** via `buildPublishSnapshot` — home page sections + metadata for all pages.
4. Assigns or validates **subdomain** (slugified, unique in `PublishedWebsite`).
5. Upserts `PublishedWebsite` with `snapshotData`, `isActive: true`, `publishedAt`.
6. Sets project `status` to `published`.
7. Logs activity via `logProjectActivity`.
8. Revalidates relevant paths.

Visitors access **`/p/[subdomain]`** which reads snapshot — not live draft canvas.

---

### Q2: What is the difference between project slug and published subdomain?

**Answer:**

- **Project slug** — Scoped per user (`@@unique([userId, slug])`). Used in dashboard URLs: `/dashboard/projects/my-startup/builder`.

- **Subdomain** — Globally unique on platform (`PublishedWebsite.subdomain`). Used in public URL: `/p/my-startup`.

They can match by default for UX but are **different columns** — renaming project slug shouldn’t break published URL without explicit subdomain update policy.

**Custom domain** — optional `customDomain` on `PublishedWebsite` with `domainVerified` and `domainVerifyToken` for DNS TXT/CNAME checks (settings UI in project settings).

---

## Medium Questions

### Q1: Why store a snapshot instead of rendering directly from Project.canvasData?

**Answer:**

**Snapshots decouple draft from live:**

- User can continue editing draft after publish without affecting live site until republish.
- Rollback = republish previous revision or restore from `CanvasRevision`.
- Performance — public route reads one JSON blob optimized for read.

**snapshotData structure** includes `version`, `homeSlug`, `pages[]`, and denormalized `sections` for fast first paint.

**Trade-off:** disk duplication — acceptable; snapshots are smaller than full revision history.

---

### Q2: How would custom domain SSL work in production?

**Answer:**

Typical stack (Vercel/Cloudflare):

1. User enters `www.example.com` in settings.
2. App shows DNS records (CNAME to platform host, TXT for verification).
3. User configures DNS; cron or on-demand check verifies `domainVerifyToken`.
4. Set `domainVerified: true`.
5. Platform provisions **TLS certificate** (Let’s Encrypt via host) on first request to domain.
6. Route traffic: Host header `www.example.com` → lookup `PublishedWebsite.customDomain` → render snapshot.

**Apex domains** may need A record + ALIAS; document limitations.

**Security:** only verified domains attach; prevent subdomain takeover by expiring unverified domains after 7 days.

---

## Hard Questions

### Q1: Scale published sites to 100k+ subdomains with low latency globally.

**Answer:**

1. **Static generation on publish** — HTML + assets to CDN; subdomain → edge KV map.
2. **Wildcard DNS** — `*.nexora.studio` → CDN.
3. **Separate origin** — `publish.nexora.com` isolated from app dashboard (different rate limits).
4. **Cache headers** — `Cache-Control: public, max-age=3600, stale-while-revalidate`.
5. **Invalidation** — on republish, purge CDN key for subdomain.
6. **Database** — `PublishedWebsite.subdomain` indexed; consider read-through cache (Redis) for hot lookups.
7. **Cold start** — Next.js SSR for `/p/*` only as fallback when CDN miss.

**Cost control:** bandwidth per user tier; disable publish for free tier abuse.

---

# 9. Media, Cloudinary & Asset Management

## Easy Questions

### Q1: How are images uploaded in the builder?

**Answer:**

**Cloudinary** handles uploads (unsigned preset for client-side or signed for server):

- Env: `CLOUDINARY_URL`, `NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME`, `NEXT_PUBLIC_CLOUDINARY_UPLOAD_PRESET`.
- `ImageUploadField.jsx` and `MediaLibraryPanel.jsx` upload files.
- On success, URL stored in section props and optionally recorded in **`MediaAsset`** Prisma model (`url`, `publicId`, `projectId`, `userId`).

**Benefits:** automatic optimization, transformations, CDN delivery — no binary blobs in Postgres.

---

### Q2: What is the MediaAsset table used for?

**Answer:**

`MediaAsset` tracks uploaded files per user/project:

- **Media library UI** — list, filter by folder, reuse across sections.
- **Quota enforcement** (future) — sum `sizeBytes` per user.
- **Cleanup** — delete from Cloudinary when asset removed (call Cloudinary destroy API by `publicId`).

It separates **asset metadata** from **canvas JSON** (which only stores URLs).

---

## Medium Questions

### Q1: How do you prevent unauthorized uploads to your Cloudinary account?

**Answer:**

1. **Unsigned preset** — restrict folder, max file size, allowed formats in Cloudinary dashboard; rate limit by IP.
2. **Authenticated uploads** — server generates signature for trusted users only.
3. **Session check** — upload initiated only from logged-in builder routes.
4. **Moderation** — optional AWS Rekognition or Cloudinary moderation add-on for NSFW.

Never expose API secret in client — only cloud name + preset for unsigned mode.

---

## Hard Questions

### Q1: User exports template ZIP with images — how do you bundle assets reliably?

**Answer:**

`exportTemplateFiles` in `utils/exportCode.js` produces `index.html`, `style.css`, `script.js`.

**Challenge:** canvas references `https://res.cloudinary.com/...` URLs.

**Options:**

1. **Leave CDN URLs** — ZIP works online only; simplest.
2. **Download assets** — server action fetches each URL, adds to ZIP `assets/` folder, rewrites paths (watch CORS and size limits).
3. **Self-host note** — document that offline use requires replace URLs.

`/api/templates/download` uses **jszip** — extend to parallel fetch images with timeout and max total MB.

---

# 10. Admin Dashboard & Analytics

## Easy Questions

### Q1: What can an admin do in the Nexora Studio admin panel?

**Answer:**

Routes under `/admin` with sidebar layout (`app/admin/layout.js`):

- **Overview** — stat cards (users, revenue, purchases, templates) + charts.
- **Users** — list, create user, block/unblock (`blockedAt`), soft delete (`deletedAt`).
- **Templates** — manage catalog (`AdminTemplatesList`).
- **Transactions** — view succeeded purchases.

All backed by `actions/admin.js` with real Prisma aggregations — **no mock chart data**.

---

### Q2: How are admin users created?

**Answer:**

1. **Seed script** — `prisma/seed.mjs` creates `admin@nexora-studio.com` with `role: admin` for development.
2. **Admin panel** — `adminCreateUser` can set role to `admin` or `user` (protect behind existing admin session).
3. **Promotion** — direct DB update in emergency only.

Login redirect sends `role === admin` to `/admin` automatically.

---

## Medium Questions

### Q1: What charts are shown on the admin overview and what data powers them?

**Answer:**

`AdminOverviewCharts.jsx` (Recharts):

1. **User growth** — time series from `User.createdAt` grouped by day.
2. **Revenue** — sum `TemplatePurchase.amountCents` where `status = succeeded` by day.
3. **Purchase distribution** — breakdown by status or category.
4. **Template usage** — purchase count or project creation per template.

Data from `actions/admin.js` functions like `getAdminStats`, `getRevenueSeries`, etc.

**Client:** Recharts needs `"use client"` — page server-fetches series, passes as props (no client-side fake data).

---

## Hard Questions

### Q1: Design audit logging and compliance for admin actions (GDPR, SOC2).

**Answer:**

**AuditLog table:**

```
id, actorId, action, targetType, targetId, metadata (Json), ipHash, createdAt
```

Log: user block, delete, role change, template price change, manual refund.

**Properties:**

- Append-only (no updates/deletes).
- Admin UI read-only search by target user.
- Retention 2 years.

**GDPR:**

- Export user data endpoint (projects, purchases, form submissions).
- Erase user — anonymize PII, keep purchase records as required by tax law.

**Access control:**

- Break-glass admin accounts with MFA.
- Separate `superadmin` role for destructive actions.

---

# 11. API Design, Server Actions & Error Handling

## Easy Questions

### Q1: When do you use Server Actions vs Route Handlers?

**Answer:**

| Use Server Actions | Use Route Handlers |
|--------------------|-------------------|
| Form submit, builder save | Stripe webhooks (raw body + signature) |
| Create project, block user | PDF binary stream (`/api/invoices/...`) |
| Checkout session create | ZIP download (`/api/templates/download`) |
| Auth mutations | Health check (`/api/health`) |
| | Public form submit (`/api/forms/submit`) |

**Rule:** need **streaming, binary, or third-party signature verification** → Route Handler. Everything else → Server Action with `{ ok, error }` pattern.

---

### Q2: What is your standard server action response shape?

**Answer:**

Consistent discriminated union:

```js
{ ok: true, data: ... }  // or project, url, etc.
{ ok: false, error: "Human readable message" }
```

**Benefits:**

- Client shows toast on `!ok` without try/catch for business logic.
- Unexpected errors caught in `catch`, logged, return generic message (don’t leak stack traces).

**Example:** `saveBlock` returns `{ ok: false, error: "Unauthorized" }` instead of throwing.

List actions return `[]` on failure (`listSavedBlocks`) so UI renders empty state.

---

## Medium Questions

### Q1: How do you handle Prisma client being stale in development?

**Answer:**

Known issue: Next.js hot reload keeps old `PrismaClient` singleton without new models.

**`lib/prisma.js` solution:**

- `REQUIRED_DELEGATES` array — `savedBlock`, `projectActivity`, etc.
- `isStaleClient()` checks `typeof client[name]?.findMany === "function"`.
- If stale, disconnect and create fresh client.

**`lib/prisma-models.js`:**

- `savedBlock()` returns delegate or `null` — callers bail gracefully.

**Developer workflow:** run `npx prisma generate` after schema change; restart dev server on Windows if EPERM.

**Production:** build runs `prisma generate` — stale client rare; still defensive for zero-downtime deploys during migrations.

---

## Hard Questions

### Q1: Design rate limiting and abuse protection for public endpoints.

**Answer:**

| Endpoint | Risk | Mitigation |
|----------|------|------------|
| `/api/forms/submit` | Spam | honeypot, captcha, per-project rate limit, `ipHash` in `FormSubmission` |
| `/api/templates/download` | Scraping | auth required for premium; rate limit per user |
| `/api/auth/*` | Brute force | lockout after N failures; slow bcrypt |
| `/register` | Bot signups | email verification required before publish |
| Webhooks | Replay | Stripe signature + idempotency |

**Implementation:** Upstash Redis sliding window; middleware or route wrapper `withRateLimit(handler, { max: 10, window: '1m' })`.

**Observability:** alert on 429 spikes per IP.

---

# 12. Deployment, DevOps & Production Readiness

## Easy Questions

### Q1: What environment variables are required for production?

**Answer:**

From `.env.example` (representative groups):

- **Database** — `DATABASE_URL` (PostgreSQL).
- **Auth** — `AUTH_SECRET`, `AUTH_URL`, Google OAuth ids if enabled.
- **Stripe** — `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, price ids.
- **Cloudinary** — cloud name, preset, API URL.
- **App** — `NEXT_PUBLIC_APP_URL` for callbacks and redirects.

`scripts/validate-env.mjs` runs on `predev` to catch missing vars early.

**Never commit** `.env` — use platform secret manager in production.

---

### Q2: What is your production build and deploy pipeline?

**Answer:**

```bash
npm run build   # prisma generate && next build
npm run start   # next start
```

**Build steps:**

1. `prisma generate` — fresh client with all models.
2. `next build` — Turbopack compile, static generation where possible.
3. Typecheck (Next 16 integrated).

**Database:**

- `prisma migrate deploy` in CI/CD before or during deploy.
- `db:seed` only for staging — not production unless initial bootstrap.

**Docker:** `docker compose` scripts for local Postgres.

**Hosting:** Vercel-compatible — serverless functions for API routes; ensure Prisma connection pooling (PgBouncer) for serverless.

---

## Medium Questions

### Q1: How do you run database migrations safely in production?

**Answer:**

1. **Backward-compatible migrations** — add column nullable first, deploy code, backfill, then enforce NOT NULL in later migration.
2. **`prisma migrate deploy`** in CI — not `db push` (push is dev convenience).
3. **Backup** before migrate — automated RDS/Neon snapshots.
4. **Zero-downtime** — two deploys for rename: add new column → dual-write → switch read → drop old.
5. **Rollback plan** — keep previous Docker image; migrations may not reverse cleanly — write `down` scripts for critical changes.

**Nexora:** JSON columns reduce migration frequency for builder props — still migrate for auth, purchases, indexes.

---

## Hard Questions

### Q1: Design a CI/CD pipeline with preview environments, E2E tests, and Stripe webhook testing.

**Answer:**

**Pipeline stages:**

1. **Lint** — `eslint`.
2. **Build** — `prisma generate && next build` with dummy env.
3. **Unit tests** — reducers, `exportCode.js`, validation schemas.
4. **Integration** — server actions against Testcontainers Postgres.
5. **E2E** — Playwright: register → create project → add section → publish → view `/p/subdomain`.
6. **Deploy preview** — Vercel preview URL per PR.
7. **Stripe** — Stripe CLI forward webhooks to preview URL in CI job; assert `fulfillTemplatePurchase`.

**Secrets:** preview DB branch (Neon), test Stripe keys.

**Smoke test post-deploy:** `/api/health` + login admin read-only check.

**Rollback:** instant previous deployment; feature flags for risky builder changes.

---

### Q2: How do you monitor and debug production issues in this stack?

**Answer:**

**Logging:**

- Structured JSON logs in server actions (`console.error('[saveBlock]', e)` → replace with pino).
- Request id in middleware for trace correlation.

**Error tracking:**

- Sentry for client + server with source maps.
- Capture Stripe webhook failures with event id.

**Metrics:**

- Latency p95 for publish, checkout, builder save.
- DB connection pool saturation.
- Failed auth attempts.

**Uptime:**

- `/api/health` checks DB `SELECT 1`.
- Synthetic monitor publishes test project in staging.

**Runbooks:**

- Prisma delegate undefined → regenerate client, redeploy.
- Webhook missed → manual `fulfillTemplatePurchase` from Stripe dashboard session id.
- User charged, no access → support uses admin transactions + session id lookup.

---

# Quick Reference — Elevator Pitch (30 seconds)

> **Nexora Studio** is a Next.js SaaS website builder where users drag sections on a Redux-powered canvas, save JSON documents to PostgreSQL via Prisma, buy premium templates through Stripe Checkout, and publish static snapshots to subdomain routes. Auth is NextAuth v5 with role-based admin, edge middleware via `proxy.js`, and server actions for mutations. Production concerns — idempotent payments, snapshot publishing, safe Prisma delegates, and invoice PDFs — are handled explicitly rather than with placeholder data.

---

# Study Plan Suggestion

| Week | Focus |
|------|--------|
| 1 | Topics 1–4 (architecture, Next.js, auth, database) |
| 2 | Topics 5–8 (builder, Redux, Stripe, publishing) |
| 3 | Topics 9–12 (media, admin, APIs, deployment) |
| 4 | Mock interviews: explain one hard question per topic without notes |

---

*Last updated for Nexora Studio codebase — Next.js 16, Prisma 6, NextAuth 5 beta.*
