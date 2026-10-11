# n8n Learning Project: Guide

**Goal:** build integrations that connect **Bizzabo** (events) with **CRMs** (HubSpot first, then Salesforce and Marketo), using n8n running locally in Docker and versioned with Git.

**Status:** Lessons 1–3 done (last updated 2026-10-11). Next: Lesson 4.

---

## 1. Project structure

```
n8n/
├── .git/                 # git history (hidden)
├── .gitattributes        # forces LF line endings on JSON, so diffs stay clean
├── docker-compose.yml    # how Docker runs n8n
├── GUIDE.md              # this file
├── notes.md              # personal notes
└── workflows/            # exported workflows (one JSON file per workflow)
    ├── .gitkeep          # placeholder so git keeps the folder even if it's empty
    ├── P7DiJt9I9WhCvJdK.json   # Hello-Git (Lesson 1)
    └── Ef4J2o9SkIaOnORr.json   # Mock Registrants to HubSpot (Lessons 2–3)
```

**Where things live:**

| Thing | Location | In git? |
|---|---|---|
| Workflows (while you edit them) | n8n's database, inside the Docker volume `n8n_data` | No |
| Workflows (exported) | `workflows/*.json` | **Yes** |
| Credentials (API keys) | n8n's database, encrypted | **Never** |
| Credentials' encryption key | Docker volume `n8n_data` | No |
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

## 3. Git Bash vs PowerShell

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

- **To-do before Lesson 4:** in Edit Fields, rename `firstName` → `firstname`, `lastName` → `lastname`, `jobTitle` → `jobtitle` (the workflow still uses the camelCase names). Then save, export, and commit.
- **Lesson 4:** create/update HubSpot contacts from the 5 mock registrants, with no duplicates (match on email).
- **Lesson 5:** custom properties (`event_status`, etc.) plus the registered → attended / cancelled flow (HTTP Request node practice).
- **Lesson 6+:** pagination (`total: 208` → loop through pages), error handling and retries, triggers (Schedule vs Webhook).
- **Later:** HubSpot OAuth2 credential; Marketing Events in a sandbox or developer test account; FastAPI service in the same `docker-compose.yml`; a script that runs export + diff in one step.

**Rebuilding this at work:** clone or copy the repo → `docker compose up -d` → create the owner account → import workflows → recreate credentials (`Bizzabo - Work`, `HubSpot - Sandbox`) → re-select credentials in each node.
