# SF Pipeline Tracker — Project Specification

> **Single-file web app** (`index.html`) for Gearset-style Salesforce deployment pipeline visualization.  
> Repo: `RomanBazylev/sf-progress--tracker` · Branch: `main` · Hosted on GitHub Pages.

---

## 1. Architecture

| Aspect | Detail |
|--------|--------|
| **Format** | All HTML + CSS + JS in one file (`index.html`, ~4200 lines) |
| **Storage** | Projects stored in `localStorage`, encrypted with AES-256-GCM + PBKDF2 |
| **APIs** | GitHub REST API v3, Salesforce Tooling API v65.0, Salesforce SOAP Metadata API v65.0 |
| **Auth – GitHub** | Personal Access Token (PAT) with `repo` scope |
| **Auth – Salesforce** | OAuth 2.0 Implicit Flow (`response_type=token`) |
| **CORS** | Cloudflare Worker proxy for Salesforce API calls (required) |
| **No backend** | Everything runs client-side in the browser |

---

## 2. Features

### 2.1 Project Management
- **Multi-project support** — create, unlock, lock, delete, export/import projects
- **Password-protected** — each project encrypted with its own password (AES-256-GCM)
- **Auto-persist** — settings and data saved to `localStorage` after each change

### 2.2 Settings (5 sections)
1. **GitHub Connection** (1/5) — PAT + repo name → "Test Connection" validates & loads branches
2. **Salesforce Cases** (2/5) — Production org URL for linking Case numbers from PR descriptions
3. **Pipeline Environments** (3/5) — Ordered list of environment branches (e.g., dev → staging → uat → main); presets: Standard, Extended, Auto-detect from Gearset `gs-pipeline/*` branches
4. **Merge Options** (4/5) — Default merge method (merge/squash/rebase) for promoting PRs
5. **Security** (5/5) — Change password, export project JSON, delete project

### 2.3 Pipeline View
- **Gearset-style horizontal flow** — environments shown as connected stage cards with colored accent bars and arrow connectors
- **Stage cards** show: display name, branch, open PR count, merged PR count, org label/link
- **Drift badges** between stages showing ahead/behind commit count (overflow-visible for proper positioning)
- **Click to expand** — clicking a stage shows an env panel below with tabs:
  - **Open** tab: PR cards (clickable → side panel)
  - **Merged** tab: recently merged PRs
  - **Sync** tab: sync PRs between environments
- **Overview mode** (no stage selected): recent activity feed, reviewer heatmap, sync PRs
- **Summary bar**: features in flight, open PRs, avg PR age, bottleneck
- **Filter bar**: search PRs, filter by author, last-updated timer, refresh button

### 2.4 PR Side Panel
- Opens on PR card click; shows full details:
  - Title, number, state (open/draft/merged)
  - Author with avatar and creation time
  - Branches (head → base)
  - Merge status (✓ Mergeable / ✕ Conflicts / ⏳ Checking...)
  - Reviewers with approval status
  - Status checks (CI/CD)
  - Labels
  - Files changed (+additions/-deletions)
  - Description (truncated with "Show full" button)
  - Feature journey (environment progression dots)
  - **Linked Issues** — extracts URLs from PR body:
    - Salesforce URLs (Case links, etc.) auto-detected
    - Jira/Atlassian/Azure/ServiceNow URLs auto-detected
    - Fallback: Case number pattern matching → constructs URL from base URL
    - GitHub issue references (#123)
  - Promote button (open PRs) — merge via GitHub API
  - Post-deploy checklist (merged PRs, and open PRs with existing items) — custom checkbox items

### 2.5 Batch Operations
- **Select multiple open PRs** via checkboxes on PR cards
- **Batch Promote** — merge all selected PRs sequentially
- **Logging** — `logMsg()` on start, per-PR success/failure, and final summary
- **`try/finally` safety** — `isPromoting` flag always reset even on unexpected errors
- Progress toast showing success/failure count
- Browser notification via `notifyUser()` on completion

### 2.6 Release Tab (Cherry-pick to Next Environment)
- **Release tab** appears on env panel for non-last environments with releasable merged PRs
- **Releasable PR detection** — merged PRs in env X whose feature has no open/merged PR targeting env X+1
- **Selectable PR cards** — checkboxes to pick specific PRs to include; if none selected, all releasable are included
- **Release form** with:
  - **Target environment** (read-only) — next env in pipeline
  - **Branch name** (editable) — auto-generated: `release/<head-ref>` for single PR, `release/YYYY-MM-DD` for multiple
  - **PR title** (editable) — auto-generated: `Release: <title> → <env>` or `Release: N changes → <env>`
  - **Included PRs summary** — list of PRs that will be cherry-picked
- **Create Release PR** workflow:
  1. Get target branch HEAD SHA
  2. Create release branch from target (e.g., master)
  3. Merge each selected feature's commit SHA into release branch via GitHub Merges API
  4. Create PR from release branch to target env branch
- **Progress trail** — state-backed `releaseProgressSteps[]` array renders full step-by-step trail (not just current step):
  - Each completed step stays visible (✓ Resolved branch, ✓ Branch created, ✓ #42 — 5 files applied...)
  - Active step updates in-place with file-level progress (🍒 Cherry-picking 2/3: #45... (12/18 files))
  - Failed steps shown inline (✕ #50 failed: merge conflict)
  - Trail survives DOM re-renders — `renderReleaseForm()` rebuilds from `releaseProgressSteps` state
- **Auto-refresh guard** — `refreshData()` skips when `isCreatingRelease || isPromoting` to prevent DOM invalidation
- **Resilient DOM references** — all button/progress updates use live `getElementById()` queries, not cached references
- **Comprehensive logging** — `logMsg()` at every step: operation start, branch resolve, branch create, each cherry-pick result, file-level skips, PR creation, final summary, errors
- **Error handling** — branch already exists (422), merge conflicts (409 per-PR), protected branch check; `try/finally` guarantees `isCreatingRelease` flag reset
- **Completion notifications** — `notifyUser()` on both success and failure; toast with 15s duration
- **Post-deploy checklist aggregation** — after promote PR is created, all post-deploy checklist items from source PRs are copied into the new PR's checklist (prefixed with `[#sourceNum]`, reset to unchecked); visible immediately on the open promote PR
- **Post-release** — clears selection, refreshes data so released PRs no longer appear in Release tab

### 2.7 Metadata Tab (Salesforce Integration)
- **Per-environment SF connections** — each pipeline environment can have its own Org URL, Consumer Key, and CORS Proxy configured in Settings → Environments
- **Per-environment OAuth App Install (Gearset-style)** — each environment row in Settings shows an install badge:
  - `✓ App` (green) if `env.sfClientId` is set and `env.connectedAppStatus === 'installed'`
  - `? App` (grey) if `env.sfClientId` is set but status unverified
  - `⚡ Install App` (yellow) if no Consumer Key — opens the install modal
  - Install modal: Org URL (pre-filled from env), Username, Password, Security Token → "Check & Install" button
  - **Check flow**: SOAP login → Tooling API query for `ExternalClientAppId` (ECA) → fallback query for `ConnectedApplication` (legacy CA)
  - **Install flow** (if not found): tries External Client App deployment first (Spring '26+), falls back to Connected App deployment (legacy)
  - ECA metadata types: `ExternalClientApplication`, `ExtlClntAppOauthSettings`, `ExtlClntAppGlobalOauthSettings` — deployed as a zip via SOAP Metadata API `deploy()`
  - After deploy, retrieves Consumer Key and auto-fills `env.sfClientId`; updates `env.connectedAppStatus`
  - Functions: `openEnvInstallModal()`, `closeEnvInstallModal()`, `envCheckAndInstall()`, `_eiBuildEcaZip()`, `_eiMetaDeploy()`, `_eiPollDeploy()`, `_eiRetrieveEcaConsumerKey()`
  - Reuses: `bsSoapLogin()`, `bsMetaSoapFetch()`, `bsApiFetch()`, `bsBuildConnectedAppZip()` (for legacy fallback)
- **Quick Connect cards** (Metadata view) — environments without a Consumer Key show an inline `⚡ Install App` button instead of being disabled
- **Connect screen** — Instance URL, Consumer Key, CORS Proxy fields; env-specific OAuth with `sfOAuthLogin(envIdx)`
- **OAuth Implicit Flow** — redirects to SF login, captures token from URL hash
- **Bootstrap: Auto-Deploy Connected App (legacy)** — collapsible "Quick Setup" section in Metadata view:
  - SOAP Login via Partner API (`/services/Soap/u/`) with username + password + security token (no Connected App needed)
  - Builds and deploys a `ConnectedApp` metadata package (`SFPipelineTracker`) via SOAP Metadata API `deploy()`
  - OAuth scopes: Api + RefreshToken; `isConsumerSecretOptional=true` (public client); `ipRelaxation=BYPASS`
  - Polls `checkDeployStatus` with exponential backoff; shows step-by-step progress trail
  - After successful deploy, queries Tooling API for the auto-generated Consumer Key and auto-fills it
  - Handles Spring '26 Connected App creation restriction: surfaces a clear error message if blocked
  - Functions: `bsSoapLogin()`, `bsMetaSoapFetch()`, `bsApiFetch()`, `bsBuildConnectedAppZip()`, `bsDeployConnectedApp()`
- **Smart redirect recovery** — `sf_oauth_pending` flag in `sessionStorage` auto-navigates back to Metadata tab after OAuth redirect
- **SOAP describeMetadata** — `sfSoapFetch()` calls the Metadata API `describeMetadata()` to dynamically discover:
  - Directory names (`describeTypeDir`) and file suffixes (`describeTypeSuffix`) for each type
  - Child type relationships (`describeChildTypes`) — only from CustomObject parent
  - InFolder/MetaFile flags (`describeInFolder`, `describeMetaFile`)
  - Results cached in `sessionStorage` (cache key `sf_describe_v2_` + instance)
- **Fetch Config** (3 steps):
  1. **Select Metadata Types** — compact 3-column grid with 53+ types organized by category (Code, UI Components, Objects & Fields, Security, Automation, Configuration, Other); group-level checkboxes via `toggleMetaGroup()`; tooltips with type descriptions
  2. **Date Range Filter** — presets (All time, Today, 2 days, Week, Month, Custom) with From/To date inputs
  3. **Compare with Git Branch** — dropdown of all repo branches, auto-loaded
- **API Info Bar** — badges showing: API version, SOAP/hardcoded describe status, component count, query success/failure stats
- **401 Session Detection** — early exit on first 401 response; shows "Session Expired" card with Reconnect button
- **Metadata Browser** — grid of components with columns:
  - Checkbox (for commit selection), Name, Type, Last Modified, Change Status, Actions
  - Change statuses: New in SF, Modified, In Sync
  - Actions: View (source code), Diff (side-by-side comparison)
  - Sticky column headers
  - Empty state with "Modify Filters" button when 0 results
- **Fetch Progress** — live indicator showing "Fetching ApexClass 2/N..."
- **SFDX-compatible paths** — `buildSfdxPath()` routes files to correct directory structure:
  - Dynamic maps from SOAP `describeMetadata()` with hardcoded fallback (`OBJECT_CHILD_FOLDERS`)
  - Object-child types (CustomField, RecordType, ValidationRule, etc.) → `objects/{ObjectName}/{childFolder}/`
  - Bundle types (LWC, Aura) → deep file structure
  - Robust object name extraction via `||` chain: `EntityDefinition.DeveloperName || TableEnumOrId || SobjectType || PageOrSobjectType`
- **Diff Modal** — shows code differences between SF and git versions
- **Commit to Git** — select changed components, choose existing or create new branch, commit message
  - Base path auto-detected from existing repo structure via `detectBasePath()`
  - Expandable file preview showing full paths before commit
  - Commit progress steps: resolving branch → uploading files → creating tree → committing → updating ref
  - Creates blobs, tree, commit via GitHub Git Data API

### 2.8 Compare & Deploy Tab (Gearset-style)
- **Purpose** — compare metadata between any combination of SF orgs and Git branches, then deploy selected changes
- **Comparison modes**: Org→Org, Org→Git, Git→Org, Git→Git
- **Multi-org OAuth** — `cdConnections` map stores per-environment tokens independently from the global `sfState`; `cdOAuthLogin(envIdx)` triggers OAuth with return context saved in `sessionStorage`
- **Parameterized API** — `cdApiFetch(conn, path, opts)` and `cdSoapFetch(conn, soapBody)` accept explicit connection objects (instance URL + token) instead of relying on global state, enabling simultaneous connections to multiple orgs
- **Source / Target panels** — each side selectable as "Org" (with environment dropdown from `projectData.sfEnvs`) or "Git" (with branch dropdown from GitHub API)
- **Metadata type filter** — compact grid of 60+ types (same `META_QUERIES` definitions used by Metadata tab), organized by category with group-level toggles; "All / None" shortcuts
- **Date range filter** — presets (All time, Today, 2 days, Week, Month, Custom) with From/To date inputs
- **Comparison engine** (`cdRunComparison`):
  - Fetches selected metadata types from both sides in parallel
  - Org fetch: SOQL via Tooling API (reuses `META_QUERIES[].soql`)
  - Git fetch: GitHub Contents API per type directory, then individual file fetch
  - Compares by component name; classifies as Added, Removed, Modified, or Unchanged
  - Progress indicator: "Fetching ApexClass from source… (2/N)"
- **Results grid** (`renderCdResults`):
  - Tabs: All, Added, Removed, Modified, Unchanged — with counts
  - Left sidebar: type filter list with per-type counts
  - Search box for component name filtering
  - Selectable rows with checkbox (for deploy selection)
  - Select All / Deselect All
- **Side-by-side diff viewer** (`cdShowItemDiff`) — modal overlay showing source vs target code in scrollable panels
- **Deploy to SF Org** (`cdDeployToOrg`):
  - Builds `package.xml` from selected components
  - Creates ZIP file (via JSZip) with component source files laid out in MDAPI structure
  - SOAP `deploy()` call with `checkOnly` option for validation-only runs
  - Polls `checkDeployStatus()` until completion
  - Progress steps: Building package → Uploading → Deploying → Complete/Failed
- **Deploy to Git** (`cdDeployToGit`):
  - Creates blobs for each selected component via GitHub Git Database API
  - Builds tree, creates commit, updates branch ref
  - Progress steps: Creating blobs → Building tree → Committing → Updating ref
- **Validate button** — runs check-only deployment to verify changes without actually deploying
- **Quick Deploy** (`cdQuickDeploy`) — after successful validation (`checkOnly=true`), the validated deploy ID is cached (`cdState._lastValidation`); the ⚡ Quick Deploy button calls SOAP `deployRecentValidation()` to deploy without re-running tests (valid for 10 days per Salesforce)
- **Test Results** — deploy status polling now parses `runTestResult` from SOAP `checkDeployStatus` response; displays test summary (passed/failed/total) inline; expandable table with class, method, outcome, time, and failure message for each test
- **Deployment History** — persistent log of all deployments stored in `projectData.deployHistory[]` (max 50, FIFO); each entry records: timestamp, type (validate/deploy/quickDeploy), target env, component count, status, test results; displayed as a collapsible table at the bottom of Compare & Deploy config view
- **Org Health** (`fetchEnvHealth`) — button on connected org panels fetches `/services/data/v65.0/limits/` and displays a modal with grid of limit cards showing used/max, remaining, and color-coded progress bars (green <70%, yellow 70-90%, red >90%); prioritizes important limits (DailyApiRequests, DataStorageMB, etc.)

### 2.9 Metadata Type Presets & package.xml
- **Save Preset** — saves current type selection as a named preset in `projectData.metaPresets`; chips displayed above type grid in both Metadata and Compare & Deploy views
- **Load Preset** — clicking a preset chip restores that type selection instantly
- **Delete Preset** — × button on each chip removes it from saved presets
- **Export package.xml** — generates standard Salesforce `package.xml` from selected types (with `<members>*</members>` and API version); downloads as file
- **Import package.xml** — file picker accepts `.xml` file; parses `<types><name>` elements and maps them back to known `META_QUERIES` types; updates type selection
- Functions: `metaSavePreset()`, `metaLoadPreset()`, `metaDeletePreset()`, `cdLoadPreset()`, `_buildPackageXml()`, `_parsePackageXml()`, `metaExportPackageXml()`, `cdExportPackageXml()`, `metaImportPackageXml()`, `cdImportPackageXml()`

### 2.10 Logs Tab (In-App Diagnostics)
- **Application log buffer** — `appLogs[]` array (max 500 entries, FIFO)
- **`logMsg(level, source, message, detail)`** — pushes to buffer + mirrors to browser console
- **Log levels**: info (ℹ️), success (✅), warn (⚠️), error (❌) with color-coded rows
- **Filter buttons** — ALL / INFO / SUCCESS / WARN / ERROR
- **Newest-first** display in monospace font with timestamps (HH:MM:SS.mmm)
- **Actions**: Clear logs, Copy All to clipboard, Download as `.txt` file
- **Sources**: describeMetadata, buildSfdxPath, ensureFullMetadata, fetchMetadata, gitTree, and more
- All 17+ former `console.log`/`console.warn` calls rewired to `logMsg()` for in-app visibility

### 2.11 UX System
- **Toast notifications** — all user feedback via animated toasts (ok/err/warn/info), no browser alerts
- **Help modal** (press `?`) — keyboard shortcuts table + quick tips
- **Keyboard shortcuts**: R=refresh, T=theme, S=settings, M=metadata, P=pipeline, D=compare&deploy, L=logs, /=search, ?=help, Esc=close
- **Project name badge** in header when unlocked
- **⬅ Switch project button** — returns to project selector from any view
- **Nav icons** — 📦 Pipeline, ☁️ Metadata, 🔀 Compare, ⚙️ Settings
- **Step numbers** on Settings sections (1/5 → 5/5) and Fetch Config (1/3 → 3/3)
- **Required field indicators** — red `*` on mandatory fields
- **Outside-click-to-close** on Create/Unlock/Help modals
- **Dark/light theme** toggle with persistence
- **Auto-refresh timer** — configurable interval (1/2/5/10 min or disabled) in Settings → Merge Options; footer shows live countdown with pulsing dot; skips refresh during long operations; `startAutoRefresh()` / `stopAutoRefresh()` lifecycle

---

## 3. Data Flow

```
GitHub API ──┐
             ├── refreshData() ──→ pipelineCache ──→ renderPipeline()
             │     ├─ open PRs (ghFetchAll)
             │     ├─ merged PRs (ghFetch)
             │     └─ drift comparisons (ghFetch per env pair)
             │
             └── buildFeatures() ──→ feature grouping by head branch (supports multiple PRs per env)

Salesforce ──┐
             ├── sfOAuthLogin(envIdx) → OAuth implicit → token in sessionStorage
             ├── fetchDescribeMetadata() → SOAP describeMetadata() → dynamic type maps (cached)
             ├── fetchMetadata() → enriches paths from describe → SOQL per type via Tooling API → metaCache
             └── fetchGitTreeForDiff() → GitHub tree API → diff comparison
```

---

## 4. State Variables

| Variable | Type | Purpose |
|----------|------|---------|
| `currentProjectIndex` | number | Index into localStorage projects array |
| `currentPassword` | string | Password for active project (in memory only) |
| `projectData` | object | Decrypted project: `{ pat, repo, envs[], caseBaseUrl, sfInstance, sfClientId, sfCorsProxy, prefs, checklists }` |
| `pipelineCache` | object | `{ prs[], mergedPrs[], branches[], drifts{}, features[], syncPrs[], releasablePrs{}, lastFetch }` |
| `detailCache` | object | PR detail/review cache by PR number |
| `sfState` | object | `{ token, instance, user }` (session-scoped via sessionStorage) |
| `metaCache` | object | `{ components[], gitTree{}, gitBranch, hasFetched, failedTypes[], queriedCount }` |
| `metaFetchConfig` | object | `{ types[], datePreset, dateFrom, dateTo, compareBranch }` |
| `repoBranches` | array | All branch names from repo (for metadata comparison) |
| `SF_API_VERSION` | string | `'65.0'` — centralized constant used by all API call sites |
| `describeTypeDir` | object | `{ typeName → directoryName }` — from SOAP describeMetadata |
| `describeTypeSuffix` | object | `{ typeName → suffix }` — from SOAP describeMetadata |
| `describeChildTypes` | Set | Child type names (only from CustomObject parent) |
| `describeInFolder` | object | `{ typeName → boolean }` — folder-based types |
| `describeMetaFile` | object | `{ typeName → boolean }` — types needing -meta.xml |
| `appLogs` | array | In-app log buffer (max 500 entries) with `{ ts, level, source, message, detail }` |
| `activeEnvIndex` | number | Selected pipeline stage (-1 = overview) |
| `activeEnvTab` | string | 'open' / 'merged' / 'sync' / 'release' |
| `batchSelected` | Set | PR numbers selected for batch promote |
| `releaseSelected` | Set | PR numbers selected for release to next env |
| `isCreatingRelease` | boolean | Lock flag during release PR creation |
| `_autoRefreshTimer` | number | setInterval handle for auto-refresh |
| `_arNextRefresh` | number | Timestamp of next scheduled auto-refresh |
| `releaseProgressSteps` | array | State-backed progress trail: `[{ text, cls }]` — survives DOM re-renders |

---

## 5. Key Functions

### Navigation & UI
| Function | Purpose |
|----------|---------|
| `showView(name)` | Switch between pipeline/metadata/logs/settings views; auto-calls `renderLogsView()` for logs |
| `enterProject()` | After unlock: show nav, populate settings, load data |
| `lockProject()` | Return to project selector; fully resets sfState, metaCache, metaFetchConfig, repoBranches & sessionStorage to prevent cross-project bleed |
| `toast(msg, type, ms)` | Show notification (types: ok, err, warn, info) |
| `showHelpModal()` / `closeHelpModal()` | Help modal with shortcuts |

### Pipeline
| Function | Purpose |
|----------|---------|
| `refreshData()` | Parallel fetch of PRs + drift → cache → render |
| `buildFeatures()` | Group PRs by feature (head branch) |
| `renderPipeline()` | Build Gearset-style stage cards + connectors |
| `renderEnvPanel(idx, tab)` | Tabbed panel below selected stage |
| `renderOverview()` | Activity feed + heatmap when no stage selected |
| `renderSummary()` | Stats bar (features, PRs, age, bottleneck) |
| `openSidePanel(prNum)` | Fetch PR detail + reviews + checks → panel |
| `promotePr(prNum)` / `batchPromote()` | Merge PRs via GitHub API |
| `extractIssues(pr)` | Extract SF URLs + case patterns + GH issues from body |

### Release
| Function | Purpose |
|----------|---------|
| `renderReleasePrCard(pr)` | PR card with checkbox for release selection |
| `toggleReleasePr(prNum)` / `releaseClear()` | Manage release selection set |
| `getReleaseBranchSuggestion()` | Auto-generate branch name: `release/<head-ref>` or `release/YYYY-MM-DD` |
| `getReleaseTitleSuggestion(envIdx)` | Auto-generate PR title based on selection |
| `getReleaseBodySuggestion(envIdx)` | Auto-generate PR body with included changes list |
| `renderReleaseForm(envIdx)` | Release form: target env, branch name, title, included PRs, create button; rebuilds progress trail from `releaseProgressSteps` state |
| `renderReleaseTrail()` | Render full progress trail from `releaseProgressSteps[]` into `#releaseProgress` DOM |
| `trailStep(text, cls)` | Append a new step to the trail and render |
| `trailUpdateLast(text, cls)` | Update the last trail step in-place (e.g., file progress counter) |
| `createReleasePr(envIdx)` | Create branch from target → cherry-pick PRs → create PR; state-backed progress trail + comprehensive logging |

### Metadata
| Function | Purpose |
|----------|---------|
| `sfOAuthLogin(envIdx)` | Redirect to SF OAuth page (per-env or manual fields) |
| `handleSfOAuthCallback()` | Capture token from URL hash |
| `restoreSfSession()` | Restore token from sessionStorage |
| `renderMetadataView()` | State machine: no token → connect, no data → config, data → browser |
| `buildFetchConfigHtml()` | Compact 3-column type grid with group checkboxes + date presets + branch selector |
| `toggleMetaGroup(groupLabel, on)` | Check/uncheck all types in a category |
| `sfSoapFetch(soapBody)` | POST SOAP envelope to Metadata API via CORS proxy, returns parsed XML |
| `fetchDescribeMetadata()` | SOAP describeMetadata() → builds 5 dynamic maps, sessionStorage cached |
| `fetchMetadata()` | Enriches types from describe → SOQL queries via Tooling API → metaCache |
| `buildSfdxPath(typeName, displayName, qPath, ext)` | Compute SFDX-compatible file path using dynamic maps + hardcoded fallback |
| `fetchGitTreeForDiff(components)` | Load git tree for selected branch, compare with SF |
| `showDiff(idx)` | Show side-by-side diff modal |
| `detectBasePath()` | Scan git tree for existing SFDX paths → auto-detect base path |
| `showCommitForm()` | Commit selected components to git |
| `doCommit()` | Create blobs → tree → commit → update ref via GitHub API |

### Logs
| Function | Purpose |
|----------|--------|
| `logMsg(level, source, message, detail)` | Push log entry to `appLogs` + mirror to console |
| `renderLogsView()` | Render log entries with filters, badges, actions |
| `copyLogsToClipboard()` | Copy all logs as plain text |
| `downloadLogs()` | Download logs as `.txt` file |

### Crypto
| Function | Purpose |
|----------|---------|
| `Crypto.encrypt(password, data)` | AES-256-GCM encryption with PBKDF2 key derivation |
| `Crypto.decrypt(password, encrypted)` | Decryption; returns original data object |

---

## 6. CSS Theme System

Two themes: `dark` (default) and `light`, toggled via `data-theme` attribute on `<html>`.

### CSS Variables (defined on `:root`)
```
--bg, --surface, --surface2, --panel-bg
--text, --muted
--border, --border-bright
--accent, --accent-dim
--green, --green-dim, --red, --red-dim
--yellow, --yellow-dim, --blue, --blue-dim
--purple, --purple-dim
--overlay
```

---

## 7. Security Model

- All project data encrypted at rest with AES-256-GCM
- PBKDF2 key derivation (100k iterations, SHA-256)
- PAT and secrets never stored in plaintext
- SF tokens stored in `sessionStorage` (cleared on tab close)
- No external dependencies or CDNs — zero supply-chain risk
- CORS proxy required for SF API calls (browser same-origin policy)
- `localStorage` writes wrapped in try/catch for `QuotaExceededError` — user warned when storage is full

---

## 8. Robustness & Error Handling

- **Retry with backoff** — `sfApiFetch()` retries on 429/503 (up to 2 retries with exponential backoff); `ghFetch()` retries on 429 and 403+Retry-After
- **`isFetchingMeta` guard** — prevents concurrent metadata fetches from double-click
- **401 session detection** — early exit on first 401, shows "Session Expired" card with Reconnect button
- **Debounced search** — pipeline search and metadata filter inputs use 150ms debounce to avoid excessive re-renders
- **Persistent type selection** — `metaFetchConfig.types` saved to encrypted project data via `projectData.metaFetchTypes`, restored on project unlock
- **`isPromoting` / `isCreatingRelease`** — lock flags prevent concurrent batch/release operations; reset via `try/finally` to prevent stuck states on unexpected errors
- **Auto-refresh guard** — `refreshData()` returns early when `isCreatingRelease || isPromoting` to prevent DOM invalidation during long-running operations
- **State-backed progress trail** — `releaseProgressSteps[]` array survives DOM re-renders; `renderReleaseForm()` rebuilds trail from state; live `getElementById()` queries instead of cached references
- **Comprehensive logging** — `createReleasePr()` and `batchPromote()` log every step (start, per-PR result, file skips, completion) via `logMsg()` for in-app Logs tab visibility
- **`_origPath` / `_origExt`** — prevents stale META_QUERIES mutation when switching orgs
- **sessionStorage cache** for describeMetadata — avoids redundant SOAP calls within a session

---

## 8. File Structure

```
sf-progress-tracker/
├── index.html          # The entire application (~4300 lines)
├── README.md           # Quick readme
└── PROJECT_SPEC.md     # This specification (keep up to date!)
```

---

## 9. Commit History

| Commit | Description |
|--------|-------------|
| `3bb55ab` | Initial: org URL links to pipeline environments |
| `581b0d4` | Environment-centric pipeline view |
| `78be960` | Refined env node cards |
| `9b6d107` | Pipeline rail line connector |
| `442bf20` | Promote, batch select, linked issues, checklist |
| `1d725f6` | Salesforce Cases replacing Jira |
| `73e7bf4` | Metadata tab: SF OAuth, Tooling API, diff, commit |
| `50394ec` | Pre-fetch type & date filters |
| `54b4160` | OAuth redirect auto-navigate fix |
| `4582545` | Better status labels, View button, tooltips |
| `e1125c8` | Sticky metadata grid headers |
| `1521d8c` | Branch selector + create-new-branch on commit |
| `450d9b7` | UX overhaul: toasts, help modal, step numbers, progress |
| `6f516f1` | Fix duplicate skeleton CSS |
| `3b3302c` | Fix merge status, case link extraction, metadata fetch loop, Gearset pipeline redesign |
| *(pending)* | Fix cross-project bleed, project switcher, drift overflow, commit form UX, auto base path |

---

*Last updated: 2025-07-24*
