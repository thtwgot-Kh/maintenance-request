# ระบบแจ้งซ่อมเครื่องจักร (Maintenance Work Order System)

Single-page Thai-language web app for submitting and tracking machine repair
requests through a multi-step approval workflow (factory manager → general
manager → technician assignment → repair).

## Live URLs

- **App:** https://thtwgot-kh.github.io/maintenance-request/ (GitHub Pages,
  deploys automatically from `main`)
- **Database (Google Sheet):**
  https://docs.google.com/spreadsheets/d/1er3WPCHJmV8_lioD-9gShr5mWDaajvXEYrEaaV-_p-k/edit
- **Repo:** `thtwgot-Kh/maintenance-request` on GitHub

## Architecture

Everything the browser needs is one file: **`index.html`** (vanilla JS, no
build step, no framework — a hand-rolled `state` object + `render()` that
does `app.innerHTML = ...` on every change). There is no backend server;
persistence and notifications are handled entirely by **two separate Google
Apps Script Web Apps**, both callable from the browser via `fetch`.

```
index.html (GitHub Pages, static)
   │
   ├─ storeGet/storeSet ──► Apps Script "คำร้องแจ้งซ่อม" (bound to the Sheet)
   │                        reads/writes the Sheet, mirrors readable tables
   │
   └─ tryLineWebhook ──────► Apps Script "LineOA MT" (standalone project)
                              pushes/broadcasts LINE messages
                                    ▲
                                    │
                       LINE ───────┘ (Messaging API webhook events:
                                      follow/message → logs userId to Sheet)
```

Both Apps Script projects live in Google's account, **not in this git repo**.
Their source is only ever pasted by hand into the Apps Script editor — see
"Apps Script projects" below for the current full source of each, and always
paste the **entire file**, replacing everything (partial pastes have broken
things before by deleting helper functions).

### 1. `index.html` — the client

- Constants near the top: `API_URL` (main data API) and `SHEET_URL` (link
  shown in Settings). Update these here and push to `main` if either Apps
  Script gets redeployed to a new URL.
- `state` is a single global object; `render()` rebuilds `#app` via
  `innerHTML` on every change (no virtual DOM).
- **Login & access control — no LINE login at all.** The app used to gate
  entry behind LINE LIFF (`liff.init`/`liff.login`). That was removed: the
  LINE Login channel sits in **"Developing"** status, which silently limits
  successful logins to the channel's own admin/tester accounts, so *every*
  other employee got either a native `เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ` alert
  (Android), a 404 (iOS), or the `access.line.me` email/password form —
  a password almost nobody has ever set. Publishing the channel was judged
  more trouble than it was worth. **Do not reintroduce a LIFF gate** unless
  that channel is actually published first.
  - Identity is now a **device id**: `D-<24 random base36 chars>` generated
    on first visit and persisted in `localStorage` under `mtDeviceId`
    (`getDeviceId()`). It is a bearer credential — whoever holds the string
    holds the access.
  - A user's roles are the **union of two sources** (`resolveAccess()`):
    the `สิทธิ์ผู้ใช้ LINE` sheet tab (server-side exact-match lookup via
    the unchanged `doGet('userRoles', <deviceId>)`), and their own approved
    row in the `accessRequests` KV key. Unknown role strings are filtered
    against `ROLES` so a typo in the sheet can't white-screen the app.
  - Unknown visitors are not blocked, they get a **"ขอสิทธิ์เข้าใช้งาน"**
    form (name / department / requested role / note; picking `tech` forces
    a technician-name choice so it matches the **ช่าง** tab exactly). That
    writes a `pending` entry to `accessRequests` and fires
    `tryLineWebhook('admin', ...)`. They then sit on a waiting screen that
    re-checks every 7s and `location.reload()`s itself the moment an admin
    approves — no second login, no re-scan, nothing to re-enter.
  - `admin` approves/rejects in **ตั้งค่าข้อมูลหลัก → คำขอสิทธิ์เข้าใช้งาน**,
    choosing the role to grant (which can differ from the one requested).
    A count badge sits on the settings tab while requests are pending.
  - **Only a `didHash` (sha256 of the device id) is ever stored server-side,
    never the raw id** — the whole `accessRequests` list is world-readable
    through the open Apps Script endpoint, so storing raw ids would let any
    user copy another's credential into `localStorage` and impersonate them.
    Keep it that way: the raw device id must stay client-side only.
  - Bootstrapping the first `admin` is the one manual step: the guest and
    waiting screens both expose that browser's own device id with a copy
    button, to be pasted by hand into `สิทธิ์ผู้ใช้ LINE` as
    `D-... | admin | <name>`.
- **Focus preservation:** because `render()` destroys/recreates every DOM
  node, any input that re-renders on `oninput` would normally lose focus
  after one keystroke. Fixed via a `data-fid="..."` attribute on such
  inputs/textareas; `render()` captures `document.activeElement`'s `data-fid`
  + cursor position before rebuilding and restores both after. **Any new
  text input whose `oninput` calls `render()` (directly or via polling) must
  get a `data-fid`,** or typing into it will break.
  - Known trap: an `onfocus` handler that unconditionally calls `render()`
    creates infinite recursion once focus-restore calls `.focus()` on it
    (this happened with the machine-search box — fixed by only calling
    `render()` from `onfocus` when the dropdown isn't already showing).
- **Polling:** `pollForUpdates()` runs every 7s via `setInterval`, re-fetches
  `config`/`orders`/`notifications`, and only calls `render()` if something
  actually changed (JSON-string diff) — this is how edits made directly in
  the Sheet (departments/machines/technicians) show up in the browser
  without a manual reload.
- **CORS gotcha:** all `fetch` calls to Apps Script use
  `Content-Type: text/plain;charset=utf-8`, never `application/json`.
  Apps Script Web Apps don't implement `doOptions`, so a JSON content-type
  triggers a CORS preflight that silently fails. The receiving script still
  `JSON.parse()`s the body regardless of the declared content type.

### 2. Apps Script "คำร้องแจ้งซ่อม" (bound to the Sheet) — main data API

Container-bound to the Database spreadsheet (open the Sheet → Extensions →
Apps Script to edit it). Deployed as a Web App (**Execute as: Me, Who has
access: Anyone**) at the URL in `API_URL`.

Two-tier storage model:

- **KV sheet** (`KV` tab): generic key→JSON-string store. Keys in use:
  `orders`, `counters`, `notifications`, `photos:<id>`, `lineWebhooks`,
  `accessRequests`, `lineUsers`, `lineUserRoles`. This
  is the literal backing store for `storeGet`/`storeSet` in the client for
  everything except `config`.
- **`config` is special and bidirectional:** departments/machines/
  technicians are **not** stored as JSON in KV. `doGet('config')` builds the
  config object live by reading three separate sheet tabs (**แผนก**,
  **เครื่องจักร**, **ช่าง**) directly, and `doPost('config', ...)` writes
  those same three tabs. This means **those tabs are the real source of
  truth** — a manager can add/edit/delete rows directly in the Sheet and the
  app picks it up within one poll cycle (~7s), no code involved.
- **Orders/notifications remain app-owned, one-way mirrors:** the
  **ใบแจ้งซ่อม** (orders) and **การแจ้งเตือน** (notifications) tabs are
  fully rewritten (`clearContents()` + re-append) every time the app saves
  `orders`/`notifications`. This was a deliberate choice — the approval
  workflow (who can approve, status transitions, timeline entries) lives in
  client JS logic, so letting someone hand-edit a status cell in the Sheet
  would desync it from reality. **Do not make these bidirectional** without
  also moving the workflow logic server-side.
  - The **ใบแจ้งซ่อม** tab's "รหัสเครื่องจักร" column is a `HYPERLINK()`
    formula jumping to the matching row in the **เครื่องจักร** tab.
  - Status/approval columns (สถานะ, อนุมัติผจก.โรงงาน, อนุมัติผจก.ทั่วไป)
    get a data-validation dropdown + conditional-formatting colors matching
    the app's status-pill palette, reapplied on every sync.
  - Dates are reformatted from raw ISO UTC to Thai Buddhist-calendar
    `dd/MM/yy[ HH:mm]` (matching the client's `fmtDate`/`fmtDateTime`) before
    being written — raw ISO strings in the Sheet are a past bug, not a
    format choice.
- **`สิทธิ์ผู้ใช้ LINE` tab (columns: `userId | บทบาท | ชื่อ`)** — controls
  who can log into the app and which role(s) they see. One row per
  (userId, role) pair; a person with multiple roles gets multiple rows.
  Despite the tab name it is **no longer LINE-specific**: the `userId`
  column now holds whatever identity string the client presents, which
  since the LIFF removal (see "Login & access control" above) is a
  **device id** of the form `D-<random>`. Hand-typing a row here is still
  the only way to create the *first* `admin`, and remains a valid way to
  grant any role permanently. Old `U...` LINE rows are inert — nothing
  presents a LINE userId to `doGet('userRoles')` any more.
  The 3rd column (`ชื่อ`) is free-text on every row (handy for the admin to
  see who's who at a glance) but the app only *reads* it on rows where
  บทบาท = `tech` — there it must exactly match a name in the **ช่าง** tab,
  since that's what lets the app know *which* technician a given
  `tech`-role LINE login is (used to restrict start/finish-repair actions
  to only that person's own assigned orders — see
  `assignedTechs(o).includes(state.myTechName)` in `index.html`). On any
  other role row the column is purely documentation and can hold whatever
  name the admin wants — the app never looks at it. `doGet('userRoles',
  ...)` returns `{ roles: [...], techName: '...' }` (not a bare array) —
  `readUserRoles_()` in the Apps Script reads all 3 columns and only
  populates `techName` from a `tech`-role row.
- `resyncAll()` — run manually from the Apps Script editor (function
  dropdown → `resyncAll` → Run) any time the mirror-writing code changes, to
  re-apply formatting/dropdowns/colors to already-saved data. Editing the
  code alone never retroactively touches existing sheet rows.
- **After any code change:** Deploy → Manage deployments → edit (pencil) →
  Version: **New version** → Deploy. This keeps the same `/exec` URL. Using
  "New deployment" instead mints a new URL, which then has to be updated in
  `index.html`'s `API_URL` and pushed to `main`.
- Full current source of this script is not stored in this repo (Apps
  Script has its own version history — File → see version history in the
  Apps Script editor). If picking this up fresh, open the Apps Script editor
  from the Sheet to read the live code directly.

### 3. Apps Script "LineOA MT" (standalone) — LINE relay

Separate, non-Sheet-bound Apps Script project. Deployed as its own Web App
(Execute as: Me, Who has access: Anyone). Its URL gets pasted into **two
places**:
1. `developers.line.biz` → the Channel → Messaging API tab → **Webhook URL**
   field (with "Use webhook" switched on) — so LINE calls it on
   follow/message events.
2. The app's own Settings page (`ตั้งค่าข้อมูลหลัก` → LINE Webhook table) —
   once per role (fm/gm/maint/requester) — so the browser calls it to send
   notifications.

It holds two secrets hardcoded as constants: `LINE_CHANNEL_ACCESS_TOKEN` and
`LINE_CHANNEL_SECRET`. **This script must never be committed to this git
repo or shared outside the team** — unlike the Sheet-connector script, it
contains live credentials. (If picking this up in a new session: the actual
token/secret values are not recorded here on purpose; ask the user or check
the LINE Developers Console → Messaging API tab if they need to be
rotated/reissued.)

Behavior:
- Incoming LINE events (`body.events` present) → logs each sender's
  `userId` + display name (fetched via the LINE profile API) into a
  **`LINE Users`** tab in the *same* Database spreadsheet (reuses
  `SPREADSHEET_ID`), so admins can look up who's who.
- Outgoing notification requests from the app (`body.role`/`body.message`
  present) → looks up recipients for that role in a **`แจ้งเตือน LINE -
  ผู้รับ`** tab (columns: บทบาท, userId) and pushes individually via LINE's
  push API. **Falls back to broadcasting to every OA friend if no recipient
  is mapped for that role yet** — this is the current state for roles that
  haven't had a real person's `userId` assigned.
  - `body.role` is not limited to the app's named roles (fm/gm/maint/
    tech/requester) — `assignTechnician()` in `index.html` sends each
    assigned technician's exact name (from the **ช่าง** tab /
    config.technicians) as `role`, so a per-technician personal LINE ping
    goes out to every technician on the order the moment it's assigned. To
    route that to the right person, add a row per technician to
    `แจ้งเตือน LINE - ผู้รับ` with บทบาท = their exact name string and
    userId = their LINE userId — same mechanism as the other roles, no
    Apps Script changes needed. Until a technician has a row there, their
    assignment ping falls back to the broadcast-to-everyone behavior above.
    (This per-name routing is separate from the `tech` app role below —
    `tech` is the role a technician's own LINE account holds to log into
    the app; the per-name row is what makes the *assignment* ping personal.)
  - `tryLineWebhook(role, message)` in `index.html` no longer requires a
    webhook URL configured under that exact `role` key in Settings — it
    falls back to whichever webhook URL *is* configured (they're always the
    same "LineOA MT" `/exec` URL in practice), so technician names never
    need their own row in the Settings → LINE Webhook table.
  - `tryLineWebhook` now also posts a **`userIds` array** alongside
    `role`/`message`, resolved client-side from the `lineUserRoles` KV map
    that `admin` fills in via **ตั้งค่าข้อมูลหลัก → กำหนดบทบาทผู้ใช้ LINE
    ที่ทักเข้ามา** (that screen lists the `lineUsers` KV entries and lets
    admin attach a role — and for `tech`, a technician name — to each LINE
    userId, replacing hand-editing the `แจ้งเตือน LINE - ผู้รับ` tab).
    **The relay script does not read `userIds` yet** — it still resolves
    recipients from that sheet tab and broadcasts when a role has no row,
    so this field is currently inert and harmless. To make routing precise,
    the relay's `doPost` needs a one-time hand edit: when `body.userIds` is
    a non-empty array, push to exactly those ids and skip both the sheet
    lookup and the broadcast fallback.
  - Nothing populates the `lineUsers` KV key automatically yet either — the
    relay logs followers to the **`LINE Users`** sheet tab only. Until it
    also mirrors them into KV (or the main script exposes that tab via a
    `doGet('lineUsers')` branch), admin adds people on that screen by
    pasting a userId + name copied out of the `LINE Users` tab.

## Git workflow used throughout this project

Every change so far: branch off `main` → edit `index.html` → commit → push →
`mcp__github__create_pull_request` → `mcp__github__merge_pull_request`
(squash). No long-lived feature branches; each PR is small and merged
immediately. Apps Script changes are never part of a PR — they're pasted by
hand into the Apps Script editor by the user, since that code doesn't live
in git.

## Known gotchas (things that already went wrong once)

- Apps Script's "Run" button in the editor calling `doGet`/`doPost` directly
  will always throw `Cannot read properties of undefined (reading
  'parameter')` — there's no real HTTP request behind it, so `e` is
  `undefined`. This is expected and harmless; test via a real URL instead
  (`.../exec?key=test` in a browser tab), not the Run button. (Both
  `doGet`/`doPost` now guard against `e` being undefined anyway.)
- A fresh Apps Script deployment with default settings often has **"Who has
  access"** set to something other than "Anyone", which makes anonymous
  `fetch` calls get redirected to a Google sign-in page — this shows up in
  the browser console as a CORS error ("No Access-Control-Allow-Origin
  header"), not as an auth error. Fix: Manage deployments → edit → confirm
  "Who has access: Anyone".
- Google Sheets' native "dropdown chip" colors (the colored circles you see
  when editing a Data Validation rule with Dropdown criteria) **cannot be
  set via Apps Script** — that's a manual, per-option UI action only. The
  automatic coloring implemented here uses Conditional Formatting instead
  (whole-cell background/text color keyed to exact text match), which *is*
  scriptable and is what actually colors the "ใบแจ้งซ่อม"/"การแจ้งเตือน"
  status columns.
- The KV sheet lost all its rows once during development (cause unknown —
  likely a manual edit while exploring the Sheet UI, not a script bug).
  Recovered via File → Version history → Restore. Consider protecting the
  `KV` tab (Data → Protect sheets and ranges) if this becomes a recurring
  risk.

## Where things stood at hand-off

- **Access control was rebuilt off LINE entirely** — see "Login & access
  control" under `index.html` above for the whole design and the reasons.
  Short version: anyone with the app link can open it in any browser, ask
  for access, and an `admin` grants a role in-app; LIFF is gone. The only
  remaining manual steps are seeding the first `admin` row in the sheet,
  and the two relay-script edits noted in the "LineOA MT" section (honour
  `userIds`, mirror followers into the `lineUsers` KV key) — both optional,
  both purely about making LINE notifications land on the right person
  instead of broadcasting.

- Core workflow (submit → FM approve → GM approve → assign → in-progress →
  done), Sheets persistence, near-real-time polling, and the simplified
  Settings page are all done and merged to `main`.
- LINE integration is mid-setup: OA created, Messaging API enabled, both
  Apps Script projects built and deployed, CORS bug fixed. Still open:
  finish wiring the LINE Developers Console Webhook URL field, have each
  real recipient message the OA once to get logged into `LINE Users`, then
  fill in the `แจ้งเตือน LINE - ผู้รับ` tab mapping each role to a real
  `userId`. Until that mapping exists per role, notifications for that role
  broadcast to every OA friend instead of the intended person.
- Technician assignment now also fires a personal LINE push (see
  `assignTechnician()` in `index.html`, keyed by the technician's exact name
  as `role`) — needs the same `แจ้งเตือน LINE - ผู้รับ` mapping filled in
  per technician once each has messaged the OA and shown up in `LINE
  Users`.
- **A job can now be assigned to multiple technicians at once.** The assign
  screen (maint role, `PENDING_ASSIGN` status) shows one technician picker
  per row with a "+ เพิ่มช่าง" button to add more (unlimited, via
  `state.ui.assignTechs`/`addAssignTechRow()`/`removeAssignTechRow()`); each
  selected technician gets their own personal LINE ping via the same
  per-name routing described above. Order data shape changed:
  `order.assignment.technician` (single string) → `order.assignment.technicians`
  (array of strings). `assignedTechs(o)` in `index.html` reads either shape
  so orders assigned before this change still display correctly.
  **The Apps Script "คำร้องแจ้งซ่อม" mirror code (not in this repo) still
  reads the old singular `technician` field for the "ช่างผู้รับผิดชอบ"
  column in the ใบแจ้งซ่อม sheet tab — it needs to be updated by hand to
  read `technicians` (joined with a comma), with a fallback to the old
  `technician` field for historical rows.**
- Added a dedicated `tech` app role (label "ช่าง") to `ROLES` in
  `index.html` — this is the role a technician's own LINE account would
  hold in `สิทธิ์ผู้ใช้ LINE` to log into the app themselves (distinct from
  the per-name LINE push routing above, which is what makes an
  *assignment* notification personal). `tech` was also added to the
  Settings → LINE Webhook role list.
- **Assigning a job now requires a planned start date and end date**
  (`assignment.startDate`/`.endDate`, two `<input type="date">` fields on
  the `PENDING_ASSIGN` assign screen) — `assignTechnician()` in
  `index.html` refuses to save (`toast(..., true)`) if either is blank, or
  if the end date is before the start date. Both dates are shown on the
  order detail page once assigned, and included in the per-technician
  personal LINE ping.
- **Starting/finishing a repair (`ASSIGNED`→`IN_PROGRESS`→`DONE`) is now
  gated on `role==='tech'` AND being one of the order's own assigned
  technicians** — `maint` only approves and assigns now; the technician
  side does the rest, and only for orders assigned to them personally
  (`assignedTechs(o).includes(state.myTechName)` in `index.html`).
  `state.myTechName` comes from the `ชื่อ` column of `สิทธิ์ผู้ใช้ LINE`
  (see above) via `doGet('userRoles', ...)`'s `techName` field — a `tech`
  account with no matching row there (or a name that doesn't exactly match
  the **ช่าง** tab) won't see the start/finish buttons on any order.
- **The "new repair request" page/nav tab is now titled "ใบคำร้อง /
  ใบแจ้งซ่อม"**, and the form has a new required "ประเภทเอกสาร" listbox
  (1. ใบคำร้อง / 2. ใบแจ้งซ่อม) shown before the machine-code field, saved
  as `order.docType` (`'request'` or `'repair'`) and shown as a small label
  on the order list card and detail page (`DOC_TYPES` in `index.html`).
- **`maint` (ผจก.แผนกซ่อมบำรุง) now records more of the paper work-order
  form on assignment**: a required receiving-manager name
  (`assignment.receivedByName`) and a required "การดำเนินงาน" execution
  plan (`assignment.executionPlan`: `immediate`/`need_purchase`/
  `use_existing`, see `EXECUTION_PLAN_META`). There's also a separate,
  editable-any-time-after-assignment **"บันทึกข้อมูลซ่อมบำรุง"** record
  (`order.maintRecord`: `causeAnalysis`, `inspectorOpinion` — see
  `INSPECTOR_OPINION_META` — and a `parts[]` table of
  name/qty/unit/price/shop/note) that `maint` fills in and every role can
  view read-only once a job is assigned.
- **Finishing a repair no longer closes the order directly.** `tech`
  pressing "ซ่อมเสร็จสิ้น" now moves the order to a new status
  `PENDING_VERIFY` ("รอผู้แจ้งตรวจสอบผลการซ่อม") instead of `DONE`. The
  `requester` then sees a "ตรวจสอบผลการซ่อม" action box with two choices:
  **ผ่าน** (moves to `DONE`, final) or **ไม่ผ่าน** (requires a reason —
  "ระบุสาเหตุที่ต้องซ่อมเพิ่มเติม" — moves the order back to `ASSIGNED` so
  the same technician(s) can re-open it via the normal start/finish flow).
  Each fail round is recorded in `order.verificationHistory[]` and the
  latest attempt in `order.verification` (`verifyRepair()` in
  `index.html`); `maintRecord` stays editable across repeat rounds.
  **This adds a new order status (`PENDING_VERIFY`) to the set the Apps
  Script "คำร้องแจ้งซ่อม" mirror code (not in this repo) writes to the
  ใบแจ้งซ่อม sheet tab's status data-validation dropdown + conditional
  formatting — that dropdown/color list needs to be updated by hand to
  include it, the same way the `technicians` field needed a manual mirror
  update.**
