# Admin Roles & Laundry Assignment — Plan

Status: **implemented** — reference docs: `laundry_admin/docs/ROLES.md`.
Scope: `laundry_admin` (portal + `/api/admin/*`). The mobile API (customers / drivers) is not affected.

---

## 1. Review: what exists today

| Role (DB value) | Label today | Laundry link | What it can do |
| --- | --- | --- | --- |
| `SUPER_ADMIN` | Super admin | `StaffUser.vendorId = null` | Everything, every laundry, with a laundry switcher (cookie) |
| `ADMIN` | Admin | **exactly one** `StaffUser.vendorId` | Runs one laundry: catalogue, drivers, customers, orders, finance |
| `USER` | User / operator | **exactly one** `StaffUser.vendorId` | Dashboard + orders (view/operate) |

How scoping works now (`src/server/auth/scope.ts`):
- `vendorWhere(session)` → `{ vendorId: session.vendorId }` for ADMIN/USER, or the switcher's laundry / `{}` for SUPER_ADMIN.
- `vendorForWrite`, `assertVendorAccess`, `vendorGuard` (in `orders/staff.ts`) all compare against **one** `session.vendorId`.
- ~33 files read `session.vendorId` or these helpers.

**Gap vs. what you want:** a laundry admin can only be linked to one laundry, and has no switcher.
Everything else (3 roles, operator = one laundry, data filtered per laundry) already holds.

---

## 2. Target model

| Role | Label | Laundries | Switcher |
| --- | --- | --- | --- |
| `SUPER_ADMIN` | Super admin | all (none assigned) | All laundries + each laundry |
| `ADMIN` | **Laundry admin** | **1..N**, assigned by super admin | "All my laundries" + each assigned laundry (hidden if only 1) |
| `USER` | **Laundry operator** | **exactly 1**, assigned by super admin | none |

Capability matrix stays as it is (`src/lib/permissions.ts`) — roles decide **what** you can do,
laundry assignments decide **where**.

### Decision: keep the DB enum values
Keep `SUPER_ADMIN / ADMIN / USER` in the database and change only the labels (en/ar) to
"Laundry admin" / "Laundry operator". Renaming the Postgres enum adds a migration and touches
every test and seed for no behavioural gain. *(If you prefer `LAUNDRY_ADMIN` / `LAUNDRY_OPERATOR`
in code too, say so — it's a mechanical rename.)*

---

## 3. Data model

New join table, replacing `StaffUser.vendorId`:

```prisma
/// Laundries a staff member works in. SUPER_ADMIN: none (platform-wide).
/// ADMIN: one or more. USER: exactly one.
model StaffVendor {
  staffUserId String
  vendorId    String
  createdAt   DateTime  @default(now())
  staffUser   StaffUser @relation(fields: [staffUserId], references: [id], onDelete: Cascade)
  vendor      Vendor    @relation(fields: [vendorId], references: [id], onDelete: Cascade)

  @@id([staffUserId, vendorId])
  @@index([vendorId])
}
```

Migration (one file, safe for existing data):
1. Create `StaffVendor`.
2. `INSERT INTO "StaffVendor" SELECT id, "vendorId", now() FROM "StaffUser" WHERE "vendorId" IS NOT NULL;`
3. Drop `StaffUser.vendorId`.

The "operator = exactly 1 / admin ≥ 1 / super admin = 0" rule is enforced in the save action
(`saveStaffAction`) — a pure `validateAssignment(role, vendorIds)` in `src/lib/permissions.ts`,
unit-tested.

---

## 4. Best approach for "APIs return data specific to role and user"

**One resolved `Scope` per request, and every query goes through it.** No page, action or route
handler decides scoping on its own.

```ts
// src/server/auth/scope.ts
interface Scope {
  /** null = platform-wide (super admin). Otherwise the laundries this user is assigned to. */
  allowed: string[] | null;
  /** The laundry picked in the switcher, always ∈ allowed. null = "all (my) laundries". */
  active: string | null;
}

getScope(session)          // React cache()'d per request; reads assignments + switcher cookie
vendorWhere(session)       // → {} | { vendorId: id } | { vendorId: { in: allowed } }
vendorIdsSql(session)      // same thing as a Prisma.sql fragment for raw queries (dashboard)
vendorForWrite(session)    // active, or the only allowed one, else "Pick a laundry first"
assertVendorAccess(session, rowVendorId)  // row must be ∈ allowed → otherwise "Not found"
```

Why this approach:
- **Same call sites as today.** Every reader already calls `vendorWhere` / `assertVendorAccess`;
  only the helper's return shape grows (`string` → `string | { in: string[] }`). Small, auditable diff.
- **Deny by default.** A laundry admin with "All my laundries" gets `{ vendorId: { in: [...] } }`,
  never `{}`. Only `allowed === null` (super admin) can produce an unfiltered query.
- **Cookie can't escalate.** The switcher cookie is validated against `allowed` on every request;
  a tampered cookie falls back to "all my laundries", never to another laundry.
- **Not found, not forbidden.** Out-of-scope rows keep returning "Not found" (no existence leak).
- **Live revocation.** Assignments are re-read with the session each request (already the pattern
  for `isActive`/`role`), so removing a laundry from an admin takes effect immediately.

Considered and rejected: Postgres row-level security (strong, but needs per-request
`SET app.vendor_ids` on the pooled `pg` connection, and changes how every raw query and the
LISTEN/NOTIFY bus work — too much for the gain right now). Can be layered on later because all
scoping already flows through one helper.

`Session` changes: `vendorId: string | null` → `vendorIds: string[]` (empty for super admin),
plus `isPlatform` derived from role.

---

## 5. Changes by area

### Auth / scope
- `session.ts` — load `vendors: { select: { vendorId } }` with the user; expose `vendorIds`.
- `scope.ts` — `getScope`, new return shapes, cookie validated against allowed set.
- `actions/auth.ts` `setVendorFilter` — accept only a laundry in the user's allowed set.
- `orders/staff.ts` `vendorGuard` — use `assertVendorAccess` (set membership).

### Queries / actions that use `session.vendorId` directly (~15 sites)
Replace each with the scope helper. Notable ones:
- `queries/dashboard.ts` — raw SQL `o."vendorId" = X` → `= ANY(${ids})`.
- `queries/customers.ts`, `actions/people.ts` (customer edit) — `orders.some.vendorId in allowed`.
- `api/admin/events/route.ts` (SSE) — filter notifications by the allowed set, not one id.
- `api/admin/finance/export`, `api/admin/reports/export` — scope via helper; reports' laundry
  filter dropdown lists only allowed laundries.
- Pages that hide the "Laundry" column with `!session.vendorId` → show it whenever the current
  scope spans more than one laundry (`showVendor = scope.active === null && allowedCount > 1`).
- Catalogue / finance settings — already require one laundry; a multi-laundry admin gets the
  same "pick a laundry" prompt a super admin gets today.
- Dashboard "laundry comparison" and report "by laundry" — also shown to a laundry admin looking
  at all their laundries (only their laundries in it).

### Portal UI
- **Switcher** (`layout.tsx`, `vendor-switcher.tsx`): shown for super admin (>1 laundry) and for a
  laundry admin with >1 assigned laundry. Options = allowed laundries; "All" label becomes
  "All my laundries" for a laundry admin.
- **Staff page / dialog** (super admin only, unchanged capability):
  - Role select with new labels and a one-line description per role.
  - Laundry admin → multi-select of laundries (≥1). Operator → single select. Super admin → hidden.
  - Table "Laundries" column lists all assigned laundries (chips).
- Sidebar role label uses the new names. `messages/en.json` + `ar.json` updated.

### New endpoint: `GET /api/admin/me`
Returns the current staff member's role, capabilities, assigned laundries and active laundry —
the single source the UI (and any future admin client) can use:
```json
{ "id": "...", "name": "...", "role": "ADMIN",
  "capabilities": ["orders.view", "..."],
  "laundries": [{ "id": "...", "name": "..." }],
  "activeLaundryId": null }
```

### Seed
- Seed accounts keep one laundry each (`admin@` and `operator@` → the demo laundry), so the
  existing single-laundry e2e flows are unchanged; `tests/e2e/staff-roles.spec.ts` creates the
  multi-laundry setup itself.

---

## 6. Tests
- **Unit** (`tests/unit/permissions.test.ts`): `validateAssignment` rules; capability matrix unchanged.
- **E2E** (`tests/e2e/multi-laundry.spec.ts`, new cases):
  1. Super admin assigns a laundry admin to laundries A + B and an operator to A.
  2. Laundry admin sees A + B in the switcher, never C; orders list with "All my laundries" contains A+B orders only.
  3. Laundry admin opening a laundry C order URL → not found; forged switcher cookie for C → falls back.
  4. Operator sees only A, no switcher, no finance/catalogue.
  5. Removing B from the admin hides B's data on the next request.
  6. `GET /api/admin/me` returns the right shape for each role.
- `pnpm typecheck`, `pnpm lint`, `pnpm test`, Playwright suite.

## 7. Documentation (after implementation)
- New **`laundry_admin/docs/ROLES.md`**: roles table, capability matrix, laundry assignment rules,
  how scoping works (`Scope`, `vendorWhere`, `assertVendorAccess`), how to add a new capability or a
  new scoped query safely, `GET /api/admin/me`.
- `ARCHITECTURE.md` §6 (auth) and §7 (multi-vendor) updated to the multi-laundry model, linking ROLES.md.
- `PRODUCT.md` roles section updated with the new names.

---

## 8. Decisions (agreed)
1. Keep DB enum values `SUPER_ADMIN` / `ADMIN` / `USER`; relabel only.
2. A laundry admin **can manage operators** in their own laundries (create / edit / deactivate,
   assign to one of *their* laundries). They cannot see or edit other admins or super admins.
   Laundry admins themselves are assigned by the super admin only.
3. A multi-laundry admin starts on **"All my laundries"**.
