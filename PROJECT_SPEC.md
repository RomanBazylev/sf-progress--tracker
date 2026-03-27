# SF Pipeline Tracker — Project Specification

> **Single-file web app** (`index.html`) for Gearset-style Salesforce deployment pipeline visualization.  
> Repo: `RomanBazylev/sf-progress--tracker` · Branch: `main` · Hosted on GitHub Pages.

---

## 1. Architecture

| Aspect | Detail |
|--------|--------|
| **Format** | All HTML + CSS + JS in one file (`index.html`, ~2900 lines) |
| **Storage** | Projects stored in `localStorage`, encrypted with AES-256-GCM + PBKDF2 |
| **APIs** | GitHub REST API v3, Salesforce Tooling API v59.0 |
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

### 2.2 Settings (4 sections)
1. **GitHub Connection** (1/4) — PAT + repo name → "Test Connection" validates & loads branches
2. **Salesforce Cases** (2/4) — Production org URL for linking Case numbers from PR descriptions
3. **Pipeline Environments** (3/4) — Ordered list of environment branches (e.g., dev → staging → uat → main); presets: Standard, Extended, Auto-detect from Gearset `gs-pipeline/*` branches
4. **Security** (4/4) — Change password, export project JSON, delete project

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
  - Post-deploy checklist (merged PRs) — custom checkbox items

### 2.5 Batch Operations
- **Select multiple open PRs** via checkboxes on PR cards
- **Batch Promote** — merge all selected PRs sequentially
- Progress toast showing success/failure count

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
- **Progress indicators** — step-by-step: creating branch → merging 1/N → creating PR → done
- **Error handling** — branch already exists (422), merge conflicts (409 per-PR), protected branch check
- **Post-release** — clears selection, refreshes data so released PRs no longer appear in Release tab

### 2.7 Metadata Tab (Salesforce Integration)
- **Connect screen** — Instance URL, Consumer Key, CORS Proxy fields
- **OAuth Implicit Flow** — redirects to SF login, captures token from URL hash
- **Smart redirect recovery** — `sf_oauth_pending` flag in `sessionStorage` auto-navigates back to Metadata tab after OAuth redirect
- **Fetch Config** (3 steps):
  1. **Select Metadata Types** — checkboxes for 6 types (ApexClass, ApexTrigger, ApexPage, ApexComponent, AuraBundle, LWC) + Select All / Deselect All
  2. **Date Range Filter** — presets (All time, Today, 2 days, Week, Month, Custom) with From/To date inputs
  3. **Compare with Git Branch** — dropdown of all repo branches, auto-loaded
- **Metadata Browser** — grid of components with columns:
  - Checkbox (for commit selection), Name, Type, Last Modified, Change Status, Actions
  - Change statuses: New in SF, Modified, In Sync
  - Actions: View (source code), Diff (side-by-side comparison)
  - Sticky column headers
  - Empty state with "Modify Filters" button when 0 results
- **Fetch Progress** — live indicator showing "Fetching ApexClass 2/6..."
- **Diff Modal** — shows code differences between SF and git versions
- **Commit to Git** — select changed components, choose existing or create new branch, commit message
  - Base path auto-detected from existing repo structure via `detectBasePath()`
  - Expandable file preview showing full paths before commit
  - Commit progress steps: resolving branch → uploading files → creating tree → committing → updating ref
  - Creates blobs, tree, commit via GitHub Git Data API

### 2.8 UX System
- **Toast notifications** — all user feedback via animated toasts (ok/err/warn/info), no browser alerts
- **Help modal** (press `?`) — keyboard shortcuts table + quick tips
- **Keyboard shortcuts**: R=refresh, T=theme, S=settings, M=metadata, P=pipeline, /=search, ?=help, Esc=close
- **Project name badge** in header when unlocked
- **⬅ Switch project button** — returns to project selector from any view
- **Nav icons** — 📦 Pipeline, ☁️ Metadata, ⚙️ Settings
- **Step numbers** on Settings sections (1/4 → 4/4) and Fetch Config (1/3 → 3/3)
- **Required field indicators** — red `*` on mandatory fields
- **Outside-click-to-close** on Create/Unlock/Help modals
- **Dark/light theme** toggle with persistence

---

## 3. Data Flow

```
GitHub API ──┐
             ├── refreshData() ──→ pipelineCache ──→ renderPipeline()
             │     ├─ open PRs (ghFetchAll)
             │     ├─ merged PRs (ghFetch)
             │     └─ drift comparisons (ghFetch per env pair)
             │
             └── buildFeatures() ──→ feature grouping by head branch

Salesforce ──┐
             ├── sfOAuthLogin() → OAuth implicit → token in sessionStorage
             ├── fetchMetadata() → SOQL per type via Tooling API → metaCache
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
| `metaCache` | object | `{ components[], gitTree{}, gitBranch, hasFetched }` |
| `metaFetchConfig` | object | `{ types[], datePreset, dateFrom, dateTo, compareBranch }` |
| `repoBranches` | array | All branch names from repo (for metadata comparison) |
| `activeEnvIndex` | number | Selected pipeline stage (-1 = overview) |
| `activeEnvTab` | string | 'open' / 'merged' / 'sync' / 'release' |
| `batchSelected` | Set | PR numbers selected for batch promote |
| `releaseSelected` | Set | PR numbers selected for release to next env |
| `isCreatingRelease` | boolean | Lock flag during release PR creation |

---

## 5. Key Functions

### Navigation & UI
| Function | Purpose |
|----------|---------|
| `showView(name)` | Switch between pipeline/metadata/settings views |
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
| `renderReleaseForm(envIdx)` | Release form: target env, branch name, title, included PRs, create button |
| `createReleasePr(envIdx)` | Create branch from target → merge feature SHAs → create PR via GitHub API |

### Metadata
| Function | Purpose |
|----------|---------|
| `sfOAuthLogin()` | Redirect to SF OAuth page |
| `handleSfOAuthCallback()` | Capture token from URL hash |
| `restoreSfSession()` | Restore token from sessionStorage |
| `renderMetadataView()` | State machine: no token → connect, no data → config, data → browser |
| `buildFetchConfigHtml()` | Type grid + date presets + branch selector |
| `fetchMetadata()` | SOQL queries for selected types + date range via Tooling API |
| `fetchGitTreeForDiff(components)` | Load git tree for selected branch, compare with SF |
| `showDiff(idx)` | Show side-by-side diff modal |
| `detectBasePath()` | Scan git tree for existing SFDX paths (classes/triggers/pages) → auto-detect base path (fallback: `force-app/main/default`) |
| `showCommitForm()` | Commit selected components to git; auto-populated base path, expandable file preview |
| `doCommit()` | Create blobs → tree → commit → update ref via GitHub API |

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

---

## 8. File Structure

```
sf-progress-tracker/
├── index.html          # The entire application (~2900 lines)
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
