# n8n Learning Project: Guide

**Goal:** build integrations that connect **Bizzabo** (events) with **CRMs** (HubSpot first, then Salesforce and Marketo), using n8n running locally in Docker and versioned with Git.

**Status:** Lessons 1–6 done, plus pagination and incremental sync (last updated 2026-10-23). Next: error handling (see section 11 and the known issues in section 9).

**Picking this up?** Read section 11 → "Setup on a new machine" first. Credentials and data tables are **not** in git, so you have to recreate them before the workflow runs.

---

## 1. Project structure

```
n8n-HubSpot-integration/
├── .git/                 # git history (hidden)
├── .gitattributes        # forces LF line endings on JSON, so diffs stay clean
├── docker-compose.yml    # how Docker runs n8n
├── GUIDE.md              # this file
├── notes.md              # personal notes
└── workflows/            # exported workflows (one JSON file per workflow)
    ├── .gitkeep          # placeholder so git keeps the folder even if it's empty
    ├── P7DiJt9I9WhCvJdK.json   # Hello-Git (Lesson 1)
    └── Ef4J2o9SkIaOnORr.json   # Mock Registrants to HubSpot (Lessons 2–6: Bizzabo → HubSpot sync)
```

**Where things live:**

| Thing | Location | In git? |
|---|---|---|
| Workflows (while you edit them) | n8n's database, inside the Docker volume `n8n_data` | No |
| Workflows (exported) | `workflows/*.json` | **Yes** |
| Credentials (API keys) | n8n's database, encrypted | **Never** |
| Credentials' encryption key | Docker volume `n8n_data` | No |
| Data tables (`bizzabo_token`, `sync_state`) and their rows | n8n's database, inside `n8n_data` | No (recreate by hand) |
| n8n owner account | Docker volume `n8n_data` | No |

---

## 2. Daily routine

**Start**
1. Open **Docker Desktop** and wait until the whale icon stops animating.
2. In the project folder:
   ```bash
   docker compose up -d
   ```
3. Open http://localhost:5678

**Stop**
```bash
docker compose down
```
Then quit Docker Desktop (tray icon → Quit) to free RAM.

**Useful**
```bash
docker compose ps         # is n8n running? (STATUS = Up)
docker compose logs -f    # watch n8n's output (Ctrl+C to exit)
```

> ⚠️ **Never run `docker compose down -v`.** The `-v` deletes the `n8n_data` volume, which holds your workflows, account, credentials, **and the key that decrypts them**.

---

## 3. Git Bash vs PowerShell (Windows only)

**On macOS/Linux:** use the PowerShell column as-is in Terminal (no prefix needed). Project folder: wherever you cloned the repo.

Git Bash rewrites anything that looks like a Linux path (`/workflows/`) into a Windows path **before** Docker sees it. Exports then end up inside the container at `/home/node/C:/Program Files/Git/workflows/` instead of in your project.

| Command | PowerShell | Git Bash |
|---|---|---|
| Export | `docker compose exec n8n n8n export:workflow --backup --output=/workflows/` | `MSYS_NO_PATHCONV=1 docker compose exec n8n n8n export:workflow --backup --output=/workflows/` |
| Import | `docker compose exec n8n n8n import:workflow --separate --input=/workflows/` | `MSYS_NO_PATHCONV=1 docker compose exec n8n n8n import:workflow --separate --input=/workflows/` |
| List workflows | `docker compose exec n8n n8n list:workflow` | same (no path involved) |
| Go to project | `cd $HOME\Desktop\n8n` | `cd ~/Desktop/n8n` |

`docker compose up/down/ps/logs` and all `git` commands are the same in both.

**Tip:** pick one terminal for n8n work. PowerShell is simpler because it never needs the prefix.

---

## 4. Save a version (n8n → git)

```
Ctrl+S in n8n  →  export  →  git diff  →  git add  →  git commit  →  git status
```

```powershell
# 1. Export (after saving in n8n!)
docker compose exec n8n n8n export:workflow --backup --output=/workflows/

# 2. Review what changed
git status
git diff

# 3. Commit
git add .
git commit -m "Describe what changed"

# 4. Confirm: should say "nothing to commit, working tree clean"
git status
```

**Checks:**
- After exporting, the JSON file's timestamp (`ls -la workflows`) should be **now**. If not, the export went somewhere else (see section 3) or you didn't save in n8n first.
- In `git diff`, ignore the noise (`updatedAt`, `versionId`, `versionCounter`) and look for the real changes.
- `git status` column meaning: `M ` (first column) = staged · ` M` (second column) = modified, not staged · `??` = new, untracked.

---

## 5. Restore a version (git → n8n)

```
git checkout <commit> -- <file>  →  import  →  refresh browser (F5)
```

```powershell
# See history
git log --oneline

# Restore one workflow file from a commit (HEAD = latest commit)
git checkout <commit-id> -- workflows/<workflow-id>.json

# Load it into n8n (overwrites the workflow with the same ID)
docker compose exec n8n n8n import:workflow --separate --input=/workflows/

# Then press F5 in the browser
```

To return to the latest version: `git checkout HEAD -- workflows/<workflow-id>.json`, then import again.

`git checkout` only changes the **file**. n8n doesn't notice until you **import**.

---

## 6. What we built

### Lesson 1: Setup and version control
- Docker Compose runs n8n; the `./workflows` folder is shared with the container as `/workflows`.
- Workflow **Hello-Git**: Manual Trigger → Edit Fields (`message = Hello from n8n!`).
- Practiced the save loop (section 4) and the restore loop (section 5).

### Lesson 2: HTTP Request + expressions
Workflow **Mock Registrants to HubSpot**:

```
Manual Trigger → HTTP Request → Split Out → Edit Fields → HubSpot (Get many contacts)
```

- **HTTP Request:** `GET https://dummyjson.com/users?limit=5` (free mock API, no key; data resembles event registrants).
- **Split Out:** field `users` turns 1 item into 5 items.
- **Key concept:** n8n works on **items**, and each node runs **once per item**.
- **Edit Fields:** maps the data to HubSpot-style fields:

  | Field | Expression |
  |---|---|
  | `email` | `{{ $json.email }}` |
  | `firstname` | `{{ $json.firstName }}` |
  | `lastname` | `{{ $json.lastName }}` |
  | `fullName` | `{{ $json.firstName }} {{ $json.lastName }}` |
  | `company` | `{{ $json.company.name }}` |
  | `jobtitle` | `{{ $json.company.title }}` |

- **Expressions:** `$json` = the current item; dots go deeper (`company.name`); text outside `{{ }}` is copied literally (including spaces).
- **Careful:** a dot in a field **name** (`company.name`) creates a nested object. HubSpot needs flat, **lowercase** names (`firstname`, `jobtitle`), and names are case-sensitive.

### Lesson 3: Credentials (HubSpot)
- Free personal HubSpot account (not the company's).
- **Auth: HubSpot Service Key.** New accounts can no longer create legacy private apps (cutoff 2026-09-28 for new accounts, 2026-10-26 for existing ones). Create one under Settings → Integrations → Service Keys.
- Scopes: `crm.objects.contacts.read`, `crm.objects.contacts.write`.
- n8n credential type **HubSpot Service Key**, named `HubSpot - Personal Sandbox`.
- **Allowed HTTP Request Domains:** Specific Domains → `api.hubapi.com`.
- Verified: the exported workflow contains only the credential's `id` + `name`, never the key.

### Lesson 4: Bizzabo token (auto-refresh)
Bizzabo API tokens expire after 24h, so a second chain in **Mock Registrants to HubSpot** refreshes the token and stores it in a Data Table:

```
Manual Trigger → Get Bizzabo Token Row → HTTP Request → Split Out → Edit Fields → HubSpot

Schedule Trigger (23h) → Get Bizzabo Token → Save Bizzabo Token
```

- **Credential** `Bizzabo - Token Request` (type **Custom Auth**). Secrets live here, never in the node:
  ```json
  {
    "headers": { "User-Agent": "n8n-bizzabo-integration" },
    "body": { "client_id": "<CLIENT_ID>", "client_secret": "<CLIENT_SECRET>", "account_id": 123456 }
  }
  ```
  `account_id` must be **snake_case and a number** (`accountId` → `401 "not installed on account '0'"`). Keep the `User-Agent`: Bizzabo's Cloudflare blocks some default ones with a 403. Allowed domain: `auth.bizzabo.com`.
- **Get Bizzabo Token** (HTTP Request): `POST https://auth.bizzabo.com/oauth/token`, Generic Credential Type → Custom Auth, JSON body fields `grant_type = client_credentials` and `audience = https://api.bizzabo.com/api`. The credential adds the rest of the body. Returns `access_token`, `expires_in` (86400).
- **Data table `bizzabo_token`** (project Personal): columns `name` (String), `access_token` (String), `expires_at` (Date). It always holds **one row** (`name = bizzabo`).
- **Save Bizzabo Token** (Data Table, **Upsert**): condition `name = bizzabo`; values `name = bizzabo`, `access_token = {{ $json.access_token }}`, `expires_at = {{ $now.plus($json.expires_in, 'seconds').toISO() }}`.
- **Get Bizzabo Token Row** (Data Table, **Get row(s)**): condition `name = bizzabo`. Later Bizzabo HTTP Requests use the header `Authorization: Bearer {{ $('Get Bizzabo Token Row').first().json.access_token }}` (plus the same `User-Agent`).
- **Two triggers, one workflow:** use the orange **Execute workflow** button next to the Manual Trigger to test the HubSpot chain. **Publishing** only activates the Schedule Trigger. The two chains must **not** be connected.
- The schedule only runs while the workflow is **published**.

### Lessons 5–6: Real Bizzabo registrations → HubSpot contacts
The dummyjson call is gone. The Manual Trigger chain now pulls real registrations and upserts them into HubSpot:

```
Manual Trigger → Get Bizzabo Token Row → Get Sync State → HTTP Request → Split Out → Edit Fields → Create or update a contact → Save Sync State
```

- **HTTP Request:** `GET https://api.bizzabo.com/v2/events/985638/registrations`, header `Authorization: Bearer {{ $('Get Bizzabo Token Row').first().json.access_token }}`.
  - Use `.first()`, not `.item`: `.item` follows item pairing and breaks once the chain has several items.
- **Split Out:** field `content` (Bizzabo puts the registrations list in `content`).
- **Edit Fields:** Bizzabo keeps attendee data under `properties`:

  | Field | Expression |
  |---|---|
  | `email` | `{{ $json.properties.email }}` |
  | `firstname` | `{{ $json.properties.firstName }}` |
  | `lastname` | `{{ $json.properties.lastName }}` |
  | `company` | `{{ $json.properties.company }}` |
  | `jobtitle` | `{{ $json.properties.title }}` |

- **Create or update a contact** (HubSpot node, Contact → Create or Update): matches on **email**, so running it twice doesn't create duplicates. Credential: `HubSpot Service Key account`.

### Pagination + incremental sync
**Pagination** (HTTP Request → Options → Pagination):
- Query parameter `size = 200` (records per page).
- Pagination mode "Update a Parameter in Each Request": `page = {{ $pageCount }}` (0-based, matches Bizzabo).
- Complete when "Other": `{{ $response.body.page.number + 1 >= $response.body.page.totalPages }}`.

**Incremental sync:** each run only fetches registrations modified since the last run.
- **Data table `sync_state`**: columns `name` (String), `last_modified` (String). One row: `name = registrations`.
- **Get Sync State** (Data Table, Get row(s)): condition `name = registrations`.
- HTTP Request query parameter `filter = modified>{{ $('Get Sync State').first().json.last_modified }}`.
- **Save Sync State** (Data Table, Upsert, **Settings → Execute Once = on**). It stores the newest `modified` value it saw, minus 30 minutes as a safety overlap:
  ```
  {{ DateTime.fromISO($('Split Out').all().map(i => i.json.modified).sort().pop()).minus({ minutes: 30 }).toUTC().toFormat("yyyy-MM-dd'T'HH:mm:ss'Z'") }}
  ```
- **Seed the row by hand** before the first run: `name = registrations`, `last_modified = 2000-01-01T00:00:00Z` (= "fetch everything"). If the row is missing, Get Sync State returns 0 items and the workflow stops silently.
- **Force a full re-sync:** set `last_modified` back to `2000-01-01T00:00:00Z`.

---

## 7. HubSpot notes for the Bizzabo integration

| Need | How | Works on free plan? |
|---|---|---|
| Create/update contacts | HubSpot node (Contact) | ✅ |
| Event status (registered / attended / cancelled) | **Custom contact properties** (e.g. `event_status`, `event_name`, `registration_date`) | ✅ |
| Event history with attendance states | **Marketing Events API** (`REGISTERED`, `ATTENDED`, `CANCELLED`, `NO_SHOW`); scopes `crm.objects.marketing_events.read/write` | ❌ (scopes not available on free plan) |
| Custom timeline activities (what Bizzabo's official app shows) | Timeline Events API: only for **approved public apps using OAuth** | ❌ Not possible for custom integrations |
| Notes / tasks / calls / meetings / emails | `crm.objects.<type>.read/write`; emails also need `sales-email-read` | ✅ |

**Background:**
- Bizzabo's official integration connects with a HubSpot **pop-up = OAuth**. It's a Marketplace public app, which is why it can write timeline events.
- **Service Key** = your own integration in one HubSpot account. **OAuth** = an app installed into many accounts.
- Marketing Events: attendance updates are idempotent (same contact + timestamp = no duplicate). Marking an event **complete can't be undone**, so only do it after all attendance changes are in.
- Existing legacy private apps (for example, at work) keep working; new integrations need Service Keys or OAuth.

---

## 8. Security rules

1. **Never put keys or tokens in any file in this repo:** not in workflows, `notes.md`, or this guide. Only in n8n → Credentials.
2. **Least privilege:** give each key only the scopes it needs, and restrict **Allowed HTTP Request Domains** to the service's API domain.
3. **Practice on personal or sandbox accounts**, not company production data.
4. **Name credentials clearly:** `<Service> - <Account/Environment>` (e.g. `HubSpot - Personal Sandbox`, `Bizzabo - Work`).
5. If you ever push this repo to GitHub, make it **private**, and for work integrations use the company's GitHub/GitLab.

---

## 9. Troubleshooting

| Problem | Cause → fix |
|---|---|
| `failed to connect to the docker API ... dockerDesktopLinuxEngine` | Docker Desktop isn't running → start it and wait |
| `archive/tar: invalid tar header` when pulling the image | Docker version too old → update Docker Desktop |
| Exported file didn't change | Git Bash without `MSYS_NO_PATHCONV=1`, or forgot Ctrl+S in n8n first |
| `No such file or directory: C:/Program Files/Git/...` | Same Git Bash path rewriting → add the prefix or use PowerShell |
| `LF will be replaced by CRLF` warning | Covered by `.gitattributes` (`*.json text eol=lf`) |
| A field in Edit Fields is saved empty | Type the value, then press Tab or click elsewhere before closing the panel |
| HubSpot node returns the same result 5 times | Normal: it runs once per input item |
| PC very slow | 8 GB RAM is tight for Docker + VS Code + browser → close apps, quit Docker when done; upgrade planned (2nd DDR4-3200 SO-DIMM for dual-channel) |
| Run stops at **Split Out** with no error | The Bizzabo call didn't return `content`: usually an expired token (Bizzabo returns 401). Check the HTTP Request output, and in its **Settings → On Error** make sure it's **Stop Workflow** (not Continue). Run **Get Bizzabo Token → Save Bizzabo Token** manually to get a fresh token. |
| Run stops at **Get Sync State** / nothing happens | The `sync_state` table has no `registrations` row → seed it (Lessons 5–6) |
| Split Out finds nothing, but the token is fresh | No registrations changed since `last_modified` → normal. Reset `last_modified` to force a full sync. |

**Known issues (to fix next):**
1. **Save Bizzabo Token** writes `expires_at = {{ $json.expires_in }}` (a number of seconds, e.g. `86400`), not a date. It should be `{{ $now.plus($json.expires_in, 'seconds').toISO() }}` as described in Lesson 4.
2. The manual chain doesn't check whether the token has expired, so if the workflow isn't published (schedule off), the token goes stale after 24h. Fix: compare `expires_at` with `$now` and refresh first if needed.
3. No error handling yet: add an **If** after HTTP Request (`{{ Array.isArray($json.content) }}`) → false branch → **Stop and Error** with a clear message, plus an Error Workflow (Error Trigger → Slack/email) for scheduled runs.
4. The `filter` query value ends with a stray newline (`...last_modified }}\n`). It works today but should be removed.
5. The Bizzabo HTTP Request doesn't send the `User-Agent` header mentioned in Lesson 4; add it if you hit a Cloudflare 403.

---

## 10. Git cheat sheet

```bash
git status                       # what changed? staged?
git diff                         # changes not yet staged
git diff --cached                # changes staged for commit
git add .                        # stage everything
git commit -m "message"          # save a version
git log --oneline                # history
git show <commit>                # what one commit changed
git checkout <commit> -- <file>  # restore a file from a commit
```

---

## 11. Next steps

- **Setup on a new machine:**
  1. `git clone https://github.com/giovanig-cmyk/n8n-HubSpot-integration.git`, `cd` into it, `docker compose up -d`, create the n8n owner account at http://localhost:5678.
  2. Import workflows (section 5).
  3. Create the data tables (they point to the original machine's IDs, so re-select them in each node):
     - `bizzabo_token` (Lesson 4) → re-select in **Save Bizzabo Token** and **Get Bizzabo Token Row**.
     - `sync_state` (Lessons 5–6) → re-select in **Get Sync State** and **Save Sync State**, and seed the `registrations` row.
  4. Create your own credentials and re-select them: `Bizzabo - Token Request` in **Get Bizzabo Token**, a HubSpot Service Key credential in **Create or update a contact**.
  5. Run **Get Bizzabo Token** → **Save Bizzabo Token** once to fill the token table, then **Publish** so the 23h refresh runs.
  6. Test with **Execute workflow** on the Manual Trigger. The event ID (`985638`) is hard-coded in the HTTP Request URL; change it if you use a different event.
- **Next: error handling.** Fix known issues 1–3 in section 9 (token expiry date, refresh-before-call, visible errors + Error Workflow).
- **Then:** custom properties (`event_status`, etc.) plus the registered → attended / cancelled flow; retries; triggers (Schedule vs Webhook) so the HubSpot sync runs on its own.
- **Later:** HubSpot OAuth2 credential; Marketing Events in a sandbox or developer test account; FastAPI service in the same `docker-compose.yml`; a script that runs export + diff in one step.

**Rebuilding this at work:** clone or copy the repo → `docker compose up -d` → create the owner account → import workflows → recreate credentials (`Bizzabo - Token Request`, `HubSpot - Sandbox`) and the `bizzabo_token` + `sync_state` data tables → re-select credentials and data tables in each node.
