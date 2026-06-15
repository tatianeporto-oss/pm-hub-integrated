# PM Hub — Shopee BR Product

A single-file product-operations hub for Shopee Brazil's PM team: Today briefing,
Projects (List / Table / Board / Timeline), Changelog, Insights, Team, and Tasks —
with multi-user login (L1 / L2 / L3), live dashboards, and Google Calendar + Drive
integrations.

> Built for the **PM Hub Challenge** (GPM: Matheus Mendes). The graded deliverable is
> a single `.html` file with no build step and no external runtime dependencies beyond
> optional Google integrations.

---

## 1. How it's built (dev vs. delivery)

The app is developed as **multiple files under `app/`** for maintainability, then
**bundled into one self-contained `.html`** for delivery.

```
app/
  PM Hub.html        ← entry point (loads everything below)
  styles.css, team-v3.css
  data.js, team-data.js     ← seed data
  helpers.js                ← shared logic (dates, movement, ETA, icons)
  app.js                    ← state, persistence, navigation
  render-today.js / render-projects.js / render-panel.js /
  render-changelog.js / render-insights.js / render-team.js / render-tasks.js
  calendar.js, select.js    ← reusable Notion/Coda-style pickers
  drive-api.js              ← real Google Drive Picker integration
  prefs.js, tweaks.js       ← theme + preferences
```

To produce the graded single file, the contents of every CSS/JS file are inlined into
`PM Hub.html` (one `<style>` / `<script>` block each, in the same order they load).
The result opens by double-click and works offline.

### Data & persistence

State lives in `localStorage` under the **original challenge keys** (kept for
backward-compatibility, as required by the rules):

| Key | Contents |
|-----|----------|
| `sph_v3` | projects |
| `sph_team3` | team members |
| `sph_tasks3` | tasks |

The "today" reference date is re-anchored to the **real day the tool is loaded**, so
all date-relative metrics (days quiet, overdue, "moved this week") stay live.

---

## 2. Publishing the tool (Mission C)

The app is a static file, so any static host works. **Recommended: GitHub Pages (free).**

### GitHub Pages — step by step

1. Create a free GitHub account and a new **public repository** (e.g. `pm-hub`).
2. Rename the delivered file to **`index.html`** and upload it (drag-and-drop in the
   repo's *Add file → Upload files*, then *Commit*).
3. In the repo, go to **Settings → Pages**.
4. Under *Build and deployment → Source*, pick **Deploy from a branch**, select the
   `main` branch and `/ (root)` folder, then **Save**.
5. Wait ~1 minute. Your public URL appears at the top of the Pages settings:
   `https://<your-username>.github.io/pm-hub/`

That URL is shareable immediately, served over HTTPS, with a global CDN. Cost: **R$ 0**.

### Optional integrations to configure at publish time

The tool is **fully usable with zero configuration** — every integration has a built-in
fallback. To switch any of them from demo/simulated mode to the real service, paste the
relevant credentials into the config blocks at the top of `PM Hub.html` (`<head>`). All
three are independent — set up any, all, or none.

| Integration | Config block | What it unlocks | Detailed steps |
|-------------|-------------|-----------------|----------------|
| **Google Sign-In** (Mission B) | `AUTH_CONFIG.clientId` | Real "Continue with Google" instead of simulated email login | §4 |
| **Google Drive Picker** (Files) | `DRIVE_CONFIG` (clientId, apiKey, appId) | "Browse Drive" reads the user's real Drive | §3 |
| **Claude AI insights** (Mission C) | `AI_CONFIG.endpoint` (Cloudflare Worker) | Live Claude-written Product pulse | §2b |

> ⚠️ The Google integrations (Sign-In + Drive Picker) **only work on a hosted `https://`
> origin** with that exact origin whitelisted in Google Cloud — not via `file://`. GitHub
> Pages satisfies this.

### Other free options

| Platform | How | Cost |
|----------|-----|------|
| **Netlify Drop** | Drag the HTML onto [app.netlify.com/drop](https://app.netlify.com/drop) | Free |
| **Vercel** | Import the repo or drag the file | Free |
| **Cloudflare Pages** | Upload or connect Git; unlimited bandwidth | Free |
| **Custom domain** | Buy a domain (Registro.br / Cloudflare) and point it at any of the above | ~R$ 40/year (hosting stays free) |

---

## 2b. The AI insights — two ways to publish (Mission C)

The "Product pulse" / executive-summary insights can run **two ways**. Both are valid
submissions; pick per how much you want to wire up.

### Option A — Deterministic mode (zero setup)

Publish the single HTML to GitHub Pages exactly as in §2. With `AI_CONFIG.endpoint`
left **blank**, the insights are generated **locally from the live portfolio data** — a
faithful, data-driven "pulse" that needs no API key, no backend, and no cost. The badge
reads *"Product pulse · generated"*. This is the safe default and works offline.

### Option B — Real Claude via a Cloudflare Worker (live AI)

To make the insights genuinely Claude-written, add a tiny **serverless proxy** that holds
your Anthropic API key. The browser calls the proxy, never Anthropic directly — so the
key is never exposed. The badge then reads *"Product pulse · Claude"* and the **Refresh
insight** button asks Claude live, summarizing only the real portfolio snapshot.

**Why a proxy at all?** GitHub Pages is static — it can't keep a secret, and putting an
API key in the HTML would let anyone read and abuse it. The Worker is that secret-keeper.

The Worker code ships in **`cloudflare-worker/`** (`worker.js` + `wrangler.toml`).

#### Step by step

1. **Get an Anthropic API key** — sign in at [console.anthropic.com](https://console.anthropic.com),
   go to **API Keys → Create Key**, copy it (starts with `sk-ant-…`). Add a little credit
   under **Billing** (these summaries cost fractions of a cent each with Haiku).
2. **Install the Cloudflare CLI** — install [Node.js](https://nodejs.org) first, then in a
   terminal: `npm install -g wrangler` (or use `npx wrangler …` without installing).
3. **Get the Worker files** — download this repo; the `cloudflare-worker/` folder has
   `worker.js` and `wrangler.toml`. In a terminal, `cd` into that folder.
4. **Log in to Cloudflare** — run `wrangler login` (opens the browser; create a free
   Cloudflare account if you don't have one).
5. **Store the key as a secret** (never in any file):
   ```
   wrangler secret put ANTHROPIC_API_KEY
   ```
   Paste the `sk-ant-…` key when prompted.
6. **Deploy** — run `wrangler deploy`. It prints your Worker URL, e.g.
   `https://pm-hub-ai.YOURNAME.workers.dev`. Copy it.
7. **Point the app at it** — open `index.html`, find the `AI_CONFIG` block near the top,
   and paste the URL:
   ```js
   window.AI_CONFIG = { endpoint: 'https://pm-hub-ai.YOURNAME.workers.dev' };
   ```
   Re-upload `index.html` to GitHub (commit). Done — the insights are now live Claude.
8. **(Recommended) Lock CORS** — in `wrangler.toml`, uncomment `ALLOWED_ORIGIN` and set it
   to your Pages URL (`https://YOURNAME.github.io`), then `wrangler deploy` again. This
   stops other sites from using your Worker.

**Cost & safety:** Cloudflare Workers' free tier covers 100,000 requests/day. Anthropic
charges per token — the pulse prompts are tiny; Haiku keeps it to fractions of a cent.
The key lives only in the Worker secret store. If anything fails (no key, rate limit,
network), the app **automatically falls back to deterministic mode** — it never breaks.

To change the model, set `MODEL` in `wrangler.toml` (default `claude-3-5-haiku-latest`;
use `claude-3-5-sonnet-latest` for richer prose at higher cost).

---

## 3. Google Drive integration (real, optional)

"Browse Drive" in a project's **Relevant Files** opens a Google Drive file picker.

- **Out of the box** it runs a built-in **demo picker** (a representative file list) so
  the tool is fully usable with zero setup.
- **When you supply credentials** (below), it switches automatically to the **real
  Google Drive Picker**, reading the signed-in user's actual Drive and attaching the
  chosen file's real name + URL. No backend required — auth happens client-side via
  Google Identity Services.

### Why it needs setup

Google requires every app that reads a user's Drive to identify itself with credentials
from **your own Google Cloud project**, and to whitelist the exact origin it runs on.
These can't be pre-baked — each deployment uses its own.

### One-time setup

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create (or
   pick) a project.
2. **APIs & Services → Library →** enable the **Google Picker API**.
3. **APIs & Services → Credentials → Create credentials → OAuth client ID**
   - Application type: **Web application**
   - Under **Authorized JavaScript origins**, add your hosting origin exactly, e.g.
     `https://<your-username>.github.io`
   - Copy the **Client ID**.
4. **Create credentials → API key** → copy the **API key**.
5. **IAM & Admin → Settings** → copy the **Project number** (this is the App ID).
6. Open `PM Hub.html` and fill in the `DRIVE_CONFIG` block near the top of `<head>`:

   ```js
   window.DRIVE_CONFIG = {
     clientId: '1234567890-abc123.apps.googleusercontent.com',
     apiKey:   'AIzaSy...',
     appId:    '1234567890'   // Google Cloud project number
   };
   ```

7. Deploy and open the page on the **authorized origin**. "Browse Drive" now opens the
   genuine Picker. (If a Google sign-in is needed, the user is prompted once.)

Leave `DRIVE_CONFIG` blank to keep the demo picker.

### Google Calendar

Reminders attach to Google Calendar with **no setup** — the "Add & Open Calendar"
button builds a pre-filled event deep-link (title = `PM Hub · <project>`, with the
chosen date/time). This works for any signed-in Google account.

---

## 4. Multi-user model — Authentication, Identity & Role-Based Views (Mission B)

### How sign-in works

PM Hub opens on a **"Continue with Google"** screen restricted to the **@shopee.com**
domain. Authentication only establishes *who you are* — your **permissions are never
configured by hand**. After sign-in, the email is matched against the **Team Directory**,
and that record alone determines your level, your squad and your reporting line. The
directory is the single source of truth.

```
Continue with Google  →  validate @shopee.com  →  match Team Directory  →  level + squad resolved
```

**Out of the box** sign-in runs in **simulated mode** (zero setup): the screen accepts any
`@shopee.com` email (e.g. `erika.izawa@shopee.com`), validates the domain and resolves the
person from the directory. It works on GitHub Pages with no backend, and a discreet
**"Open the demo"** link offers one-click login as L1 / L2 / L3 for graders. **When you
supply an OAuth Client ID** (below) it switches automatically to the **real Google
Sign-In** button; the returned account's email runs through the exact same directory
match, and non-`@shopee.com` accounts are rejected.

The email → person mapping uses the canonical `firstname.lastname@shopee.com` convention
(stored emails win when present), so `tatiane.porto@shopee.com` resolves to **Tatiane
Porto · L3 · reports to Matheus Torresi**.

### Why it needs setup

Google requires every app that signs users in to identify itself with an **OAuth Client
ID** from **your own Google Cloud project**, and to whitelist the exact origin it runs on.
This can't be pre-baked — each deployment uses its own. (Identity only — no API key or
backend is needed for Sign-In.)

### One-time setup (real Google Sign-In)

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create (or
   pick) a project.
2. **APIs & Services → OAuth consent screen** → configure it once:
   - User type **Internal** (restricts to your Google Workspace org — ideal for Shopee),
     or **External** if you can't use Internal.
   - Fill app name + support email and save.
3. **APIs & Services → Credentials → Create credentials → OAuth client ID**
   - Application type: **Web application**
   - Under **Authorized JavaScript origins**, add your hosting origin exactly, e.g.
     `https://<your-username>.github.io`
   - Copy the **Client ID**.
4. Open `PM Hub.html` and paste it into the `AUTH_CONFIG` block near the top of `<head>`:

   ```js
   window.AUTH_CONFIG = {
     clientId: '1234567890-abc123.apps.googleusercontent.com'
   };
   ```

5. Deploy and open the page on the **authorized origin**. The login screen now renders the
   genuine **Continue with Google** button. Sign-in is restricted to `@shopee.com`, and
   the account is matched against the Team Directory to resolve level/squad/reporting line.

Leave `AUTH_CONFIG.clientId` blank to keep the simulated sign-in (full demo, zero setup).

> Tip: the same Google Cloud project can hold the **Sign-In Client ID** (this section) and
> the **Drive Picker** Client ID + API key (§3). They're independent — set up either,
> both, or neither.

### The levels (derived from the directory)

| Level | Who | Dashboard & Tasks | Can edit | Leadership analytics |
|-------|-----|-------------------|----------|----------------------|
| **L1 — GPM** | Matheus Mendes (no manager) | **All teams** | **Everything** | Visible |
| **L2 — Squad lead** | The 4 squad mentors — Leandra Mieko (Seller FBS), Lucas Menegaldo (Taxes FBS), Matheus Torresi (New Services), Danilo Leite (Monetization) | **Their squad** | Their squad's projects | Visible |
| **L3 — PM** | Every other PM | **Today: their squad** · Tasks: own | **Only projects they own** | Hidden |
| **Guest** | A Shopee employee not in the directory | All (read-only spectator) | **Nothing** | Hidden |

Level is computed from the org structure: **Matheus Mendes = L1**, the **squad mentors =
L2** (report to the GPM), **everyone else = L3** (reports to their squad's mentor). No
`level` is set by hand in the UI.

### What each level experiences

- **Filtered Dashboard & Tasks.** *Today* gives L3 their **squad's** picture — the same
  team-wide read their L2 lead sees (not just their own initiatives), so a PM has the
  context of everything their team is shipping. *Tasks* stays personal (what that PM
  owns). L1 sees the whole portfolio; L2 sees their squad. The sidebar's open-task count
  follows the Tasks scope.
- **Edit gating.** Inline editing, status changes, weekly updates, files, reminders,
  archive and delete are enabled only where you have rights. Everywhere else the project
  opens with a **"View only"** banner and disabled controls. (An L3 owns and edits the
  initiatives they create.)
- **Leadership views disappear for L3.** The Team **Workload** tab and the Insights
  **Team Comparison** / per-team health sections are simply **not rendered** for L3 or
  guests — not greyed out, gone. L1 and L2 see them.
- **Identity chip (top-right).** Shows your avatar, name, level badge and reporting line
  (e.g. *"Tatiane P. · L3 · Reports to Matheus T."*). Clicking it opens a menu to **sign
  out** and — for testing the build — a **"Preview a level"** switcher (L1/L2/L3) that
  re-renders the whole app as that level without signing out. The preview respects the
  same read-only rules.

### Implementation notes

- All of the above lives in **`app/auth.js`** (`window.AUTH`): org derivation, session,
  scoping (`scopedProjects` / `scopedTasks`), permissions (`canEdit` / `canSeeLeadership`),
  the login overlay and the identity chip. The session persists under `pmh_session`.
- Per-L2-team Dashboard health metrics (Mission D) — **load per PM**, **projects with no
  update in >10 days**, **% on-track vs. overdue** — surface in the Insights *Team
  Comparison* cards, gated to L1/L2.

---

## 4b. Notifications & Preferences (support surfaces, not modules)

Both are intentionally lightweight — workspace support, never dashboards, and they never
duplicate Today / Insights.

### Notifications — "what requires my attention right now?"

A **bell beside the identity chip** opens a right-side panel (Linear / Notion-Inbox feel).
The badge counts unseen *Action Required* items; opening the panel marks them seen
(`pmh_notif_seen`). Two categories, prioritised by relevance over volume:

- **Action Required** (all levels) — only the viewer's **own** work: a project with no
  update in >7 days, a project past or approaching (≤7d) its live date, a task due today,
  an overdue task. Always on top.
- **Recent Team Activity** — role-aware awareness, **never** a company-wide feed:
  - **L3** → no team activity at all (only Action Required).
  - **L2** → movement from **direct reports** only (e.g. *"Ricardo moved Samsung New
    Logistic Channel to Live Testing."*).
  - **L1** → organizational awareness, **aggregated by squad** (e.g. *"Seller FBS moved 2
    initiatives this week."*) — awareness, not per-initiative tracking.

Health scores, overdue counts and workload deliberately **stay out** of notifications —
they live in Today and Insights. Implemented in `render-notifications.js` (`window.NOTIF`).

### Preferences — minimal workspace settings

Opened from the sidebar (identity/profile stays in Team). Three controls only:

- **Theme** — just two: **Dark** (default, primary) and **Accent** (warm, Coda-inspired).
- **Opens on** — the default landing module (Today / Projects / Changelog / Insights /
  Team / Tasks); read on boot from `pmh_landing`.
- **Notify me about** — per-type notification toggles (projects without updates, tasks due
  today, overdue tasks, upcoming live dates, ETA changes; *team activity* for L1/L2 only).
  Stored in `pmh_notif_prefs`; the panel respects them live.

---

## 5. Bugs fixed & prompts (Missions A · documentation)

> This section is the running log of bugs found, the prompt used, and the result.
> _(To be compiled — ask Claude to assemble it from the change history.)_

| # | Area | Bug | Prompt summary | Result |
|---|------|-----|----------------|--------|
| | | | | |

---

## 6. Metric definitions (standardized across all modules)

Every module — Today, Changelog, Projects, Insights — derives its signals from the
**same shared helpers** in `helpers.js`, so a number means the same thing everywhere.
All windows are relative to the **real day the tool is loaded** (`daysQuiet` is the
number of days since a project's last *movement*).

| Term | Definition | Helper |
|------|-----------|--------|
| **Movement** | A **status change** OR a logged **weekly update**. Plain body edits (objective, PICs, ETAs, files) do **not** count. | (stamps `lastActivity`) |
| **Moved this week** | Had movement in the last **7 calendar days** (`daysQuiet ≤ 7`). | `movedThisWeek(p)` |
| **Quiet / Gone quiet** | In active flow **and** no movement for **more than 10 days** (`daysQuiet > 10`). | `isStale(p)` |
| **Overdue / Past live date** | Past its Live ETA, not yet Live, not frozen. | `p.overdue` |
| **Needs attention** (umbrella) | **Overdue** OR **quiet** (stale). Frozen/paused is a deliberate state, **not** a risk. The single risk signal. | `needsAttention(p)` |
| **Launching soon (ETA highlight)** | Overdue OR Live ETA within the next **15 days**. | `etaImminent(p)` |

### How the Changelog uses them

The Changelog groups each project's latest movement into four editorial buckets:

- **Launches** — status reached Live / A/B / Live Testing, or the update text announces a release.
- **Progress** — moving forward (default for active movement).
- **Decisions** — the update text records an approval, sign-off, scope change, or kickoff.
- **Risks & delays** — the project **needs attention** (`needsAttention(p)` = overdue or quiet)
  **or** the PM flagged a blocker in their own update text (keywords like *blocked, waiting, on hold, slip*). Frozen/paused projects are a deliberate state and are **not** auto-flagged as risk.

The Changelog digest reports `N need(s) attention` and `N gone quiet (>10 days)` using
those exact helpers — the same thresholds the Today briefing uses ("haven't been touched
in over ten days") and the Projects quick-filters ("Gone quiet", "Past live date").

### How the Insights module uses them

Insights is read-first: each section opens with a plain-language sentence, then the evidence.

- **Portfolio health score** = average of two halves: **freshness** (active initiatives not
  gone quiet, i.e. `daysQuiet ≤ 10`) and **on-time delivery** (active initiatives not
  overdue). The "Fresh 18/94" and "On time 49/94" factors are clickable — they open the
  Projects list filtered to exactly those initiatives (`fresh` / `ontime` quick filters).
- **Is the work moving?** — "This week" = `movedThisWeek` (`daysQuiet ≤ 7`); "Prev week" =
  moved 8–14 days ago. Both are clickable into the filtered Projects list. Upcoming-launch
  months open the Projects list filtered to that month's Live ETAs.
- **What is at risk?** — two columns: **Past live date** (overdue) and **Gone quiet · 10d+**
  (`isStale`). **Highlight ranking:** overdue is sorted by priority (P0 first), then by
  longest-overdue; quiet is sorted by longest-quiet. The **top 5** of each are surfaced as
  the most urgent; "See all" opens the full filtered list. Every row opens the initiative.
- **How are priorities distributed?** — active initiatives bucketed P0/P1/P2/P3/To define;
  each bar opens the Projects list filtered to that priority.
- **Which teams are healthiest?** (Mission D) — gated to L1/L2; per-team health, on-track %,
  load per PM, and quiet count.

---

## 7. Naming & format conventions

- **PM names** display as `First L.` (first name + last-initial).
- **Dates** display as `1 Oct '26` (day · month · year).
- The pipeline field is labelled **Status** throughout.
- Live ETA is highlighted in the accent color only when **overdue or launching within
  15 days**; all other dates are muted.

---

*Shopee Brasil · Group Product Management · PM Hub Challenge 2026*
