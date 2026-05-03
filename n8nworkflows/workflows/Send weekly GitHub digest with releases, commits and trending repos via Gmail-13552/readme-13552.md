Send weekly GitHub digest with releases, commits and trending repos via Gmail

https://n8nworkflows.xyz/workflows/send-weekly-github-digest-with-releases--commits-and-trending-repos-via-gmail-13552


# Send weekly GitHub digest with releases, commits and trending repos via Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send weekly GitHub digest with releases, commits and trending repos via Gmail

**Purpose:**  
Runs every Monday at 9am, gathers three types of GitHub activity (recent releases, recent commits/activity, and today’s trending repositories), merges them into a single HTML email, and sends it via Gmail **only if at least one section contains content**.

**Target use cases:**
- Personal or team weekly GitHub digest
- Tracking releases for a set of repositories
- Tracking development activity for selected GitHub users *and/or* repositories
- Including a “Trending today” snapshot in the same email

### 1.1 Scheduling & Configuration
- Triggered weekly (Monday 9am).
- Central configuration via a Set node: recipient email, lookback window, watched repos, tracked entities.

### 1.2 Releases Branch (watched repos → latest release → filter by lookback)
- Splits watched repos array into items.
- Calls GitHub Releases API per repo (most recent release).
- Aggregates and filters releases newer than `days_back`.

### 1.3 Commits Branch (tracked entities → user events or repo commits → normalize)
- Splits tracked entities array into items.
- If entity is a repo path (`owner/repo`), fetch commits.
- If entity is a username/org, fetch public events and extract push commits.
- Normalizes and de-duplicates commits.

### 1.4 Trending Branch (scrape GitHub trending HTML → parse top repos)
- Downloads GitHub trending page (daily).
- Parses HTML to extract top ~10 trending repos.

### 1.5 Merge, Gate, Compose & Send
- Merges the three branches’ outputs.
- IF node checks whether any section count is > 0.
- Builds a combined HTML email (with subject).
- Sends via Gmail; otherwise “Do Nothing”.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Variables

**Overview:** Triggers the workflow weekly and defines user-editable settings used by all branches.

**Nodes involved:**
- Every Monday at 9am
- Set Variables

#### Node: Every Monday at 9am
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):** Runs weekly on **Monday** at **09:00**.
- **Connections:**
  - Output → **Set Variables**
- **Potential failures / edge cases:**
  - Timezone depends on n8n instance settings; 9am is evaluated in that timezone.
- **Version notes:** typeVersion `1.2`.

#### Node: Set Variables
- **Type / role:** `Set` — provides configuration constants for the run.
- **Key fields set:**
  - `recipient_email` (string): target email address.
  - `days_back` (number): lookback window in days (default 7).
  - `release_repos` (array): watched repos (e.g., `["n8n-io/n8n"]`).
  - `tracked_entities` (array): usernames and/or repos (e.g., `["anthropics/claude", "huggingface/transformers"]`).
- **Key expressions:**
  - Arrays are defined via expressions like `{{ ['n8n-io/n8n'] }}`.
- **Connections:**
  - Output fan-out → **Split Watched Repos**, **Split Tracked Entities**, **Fetch GitHub Trending Page**
- **Potential failures / edge cases:**
  - Invalid arrays (e.g., string instead of array) will cause Split Out nodes to error or produce no items.
  - `days_back` non-numeric can break date math downstream.
- **Version notes:** typeVersion `3.4`.

---

### Block 2 — Releases Branch

**Overview:** For each watched repo, fetches the most recent release and keeps it only if published within the lookback window.

**Nodes involved:**
- Releases Section (sticky note)
- Split Watched Repos
- Fetch Latest Release
- Aggregate Releases
- Sticky Note (Important note about prereleases)

#### Node: Releases Section (Sticky Note)
- **Type / role:** `Sticky Note` — documentation only.
- **Content:** Explains that this section fetches the latest release per watched repo and filters by lookback window.
- **Connections:** none.

#### Node: Split Watched Repos
- **Type / role:** `Split Out` — expands `release_repos` array into one item per repo.
- **Configuration:**
  - `fieldToSplitOut`: `release_repos`
- **Input expectations:** The incoming item must include `release_repos` as an array (from **Set Variables**).
- **Connections:**
  - Input ← **Set Variables**
  - Output → **Fetch Latest Release**
- **Potential failures / edge cases:**
  - If `release_repos` is empty → produces 0 items → downstream yields empty release aggregation.
  - If `release_repos` contains non-string values → GitHub URL construction may fail.
- **Version notes:** typeVersion `1`.

#### Node: Fetch Latest Release
- **Type / role:** `HTTP Request` — calls GitHub REST API for releases.
- **Configuration (interpreted):**
  - URL template: `https://api.github.com/repos/{{ $json.release_repos }}/releases?per_page=1`
    - Note: after Split Out, `$json.release_repos` is the current repo string (e.g., `n8n-io/n8n`).
  - Headers:
    - `Accept: application/vnd.github+json`
    - `X-GitHub-Api-Version: 2022-11-28`
  - **Error handling:** `onError: continueRegularOutput` (workflow continues even if GitHub returns errors).
- **Connections:**
  - Input ← **Split Watched Repos**
  - Output → **Aggregate Releases**
- **Potential failures / edge cases:**
  - GitHub rate limits (especially without token).
  - 404 if repo doesn’t exist or is private.
  - Response is an array; downstream handles array vs object.
  - Because errors continue, downstream may receive error-shaped JSON (varies by n8n), potentially missing expected fields.
- **Version notes:** typeVersion `4.4`.

#### Node: Sticky Note (Important)
- **Type / role:** `Sticky Note` — documentation only.
- **Content (important behavior note):**
  - GitHub `/releases/latest` skips prereleases.
  - Current workflow uses `/releases?per_page=1` to include prereleases.
  - Suggests using `https://api.github.com/repos/repo-name/releases/latest` if you want only “usual” releases.
- **Connections:** none.

#### Node: Aggregate Releases
- **Type / role:** `Code` — consolidates all repo results into a single `{ releases, release_count }`.
- **Core logic (interpreted):**
  - Reads `days_back` from **Set Variables** and computes a cutoff timestamp.
  - For each input item:
    - Ensures release list is treated as an array.
    - Skips if `published_at` or `html_url` missing.
    - Skips drafts; includes prereleases.
    - Skips releases older than cutoff.
    - Extracts repo name from `html_url`.
    - Sanitizes body: removes markdown headings and HTML comments, trims, truncates to ~280 chars.
    - Only keeps the most recent release per repo (break after first acceptable release).
  - Outputs a single item: `{ releases: [...], release_count: N }`.
- **Key variables / references:**
  - `$('Set Variables').first().json.days_back`
- **Connections:**
  - Input ← **Fetch Latest Release**
  - Output → **Merge Releases + Commits** (input 0)
- **Potential failures / edge cases:**
  - If GitHub returns error objects or unexpected shapes, code may push nothing (safe) due to guards.
  - Release body truncation always adds `...` when body exists (even if already short).
- **Version notes:** typeVersion `2`.

---

### Block 3 — Commits Branch

**Overview:** For each tracked entity, fetches either user public events (for usernames) or recent repo commits (for repo paths), then normalizes to a unified “commit” format.

**Nodes involved:**
- Commits Section (sticky note)
- Split Tracked Entities
- Fetch Entity Data
- Extract Commits

#### Node: Commits Section (Sticky Note)
- **Type / role:** `Sticky Note` — documentation only.
- **Content:** Supports both GitHub users (`torvalds`) and repos (`huggingface/transformers`).
- **Connections:** none.

#### Node: Split Tracked Entities
- **Type / role:** `Split Out` — expands `tracked_entities` array into one item per entity.
- **Configuration:**
  - `fieldToSplitOut`: `tracked_entities`
- **Connections:**
  - Input ← **Set Variables**
  - Output → **Fetch Entity Data**
- **Potential failures / edge cases:**
  - Empty array → branch yields no commits.
  - Non-string entities → URL building expression may fail.
- **Version notes:** typeVersion `1`.

#### Node: Fetch Entity Data
- **Type / role:** `HTTP Request` — calls GitHub API, choosing endpoint based on whether entity contains `/`.
- **Configuration (interpreted):**
  - URL expression:
    - If `tracked_entities` includes `/` (repo path):
      - `https://api.github.com/repos/<entity>/commits?per_page=20&since=<ISO date>`
      - `<ISO date>` computed as: `$now.minus({days: $('Set Variables').first().json.days_back}).toISO()`
    - Else (username/org):
      - `https://api.github.com/users/<entity>/events/public?per_page=30`
  - Headers:
    - `Accept: application/vnd.github+json`
    - `X-GitHub-Api-Version: 2022-11-28`
  - **Error handling:** `onError: continueRegularOutput`
- **Connections:**
  - Input ← **Split Tracked Entities**
  - Output → **Extract Commits**
- **Potential failures / edge cases:**
  - Rate limits; unauthenticated requests are low.
  - User events endpoint does not accept `since`; filtering happens in code node.
  - For repo commits, GitHub returns an array; for user events, also an array. Downstream code expects either event objects or commit objects; mixed arrays can cause partial extraction.
  - Private repos will 404 unless authenticated.
- **Version notes:** typeVersion `4.4`.

#### Node: Extract Commits
- **Type / role:** `Code` — normalizes events/commits into `{ commits, commit_count }`.
- **Core logic (interpreted):**
  - Computes cutoff timestamp based on `days_back`.
  - Iterates through all incoming items:
    - Skips items with `message === 'Not Found'`.
    - **If `data.type` exists** (treat as a single user event object):
      - Keeps only `PushEvent` events newer than cutoff.
      - Extracts up to 3 commits from the push payload.
      - Builds commit entries with repo, author, sha, URL, date.
    - **Else if `data.sha && data.commit` exists** (treat as a single commit object):
      - Filters by commit date newer than cutoff.
      - Extracts repo path from `html_url`.
      - Builds commit entry.
  - De-duplicates by `sha`, keeps up to 40 unique commits.
  - Outputs single item: `{ commits: [...], commit_count: N }`.
- **Key variables / references:**
  - `$('Set Variables').first().json.days_back`
- **Connections:**
  - Input ← **Fetch Entity Data**
  - Output → **Merge Releases + Commits** (input 1)
- **Potential failures / edge cases (important):**
  - **Array handling risk:** GitHub endpoints usually return arrays; this code treats `item.json` as a single object, not an array. If n8n returns the whole array in `item.json`, `data.type` is undefined and `data.sha` is undefined, resulting in **zero commits extracted**.  
    - In many n8n setups, HTTP Request returns the parsed JSON as the item’s JSON (which could indeed be an array). If so, you’d need to loop through the array elements.
  - PushEvent commit URLs are synthesized; if repo name is missing, URLs may be malformed.
- **Version notes:** typeVersion `2`.

---

### Block 4 — Trending Branch

**Overview:** Downloads GitHub Trending HTML and parses it into a list of trending repositories.

**Nodes involved:**
- Trending Section (sticky note)
- Fetch GitHub Trending Page
- Parse Trending HTML

#### Node: Trending Section (Sticky Note)
- **Type / role:** `Sticky Note` — documentation only.
- **Content:** Scrapes the trending page for the day the workflow runs.
- **Connections:** none.

#### Node: Fetch GitHub Trending Page
- **Type / role:** `HTTP Request` — fetches HTML from GitHub Trending.
- **Configuration (interpreted):**
  - URL: `https://github.com/trending?since=daily`
  - Headers:
    - `User-Agent`: browser-like UA string
    - `Accept`: `text/html,application/xhtml+xml`
  - **Error handling:** `onError: continueRegularOutput`
- **Connections:**
  - Input ← **Set Variables**
  - Output → **Parse Trending HTML**
- **Potential failures / edge cases:**
  - GitHub may block/alter content for bots; UA header helps but is not guaranteed.
  - If GitHub changes markup, parsing logic may fail and return 0 repos.
  - Depending on n8n HTTP node settings, HTML might appear in `json.body` or `json.data`; code checks both.
- **Version notes:** typeVersion `4.2`.

#### Node: Parse Trending HTML
- **Type / role:** `Code` — extracts repo entries from HTML.
- **Core logic (interpreted):**
  - Reads HTML from `$input.first().json.data || .body`.
  - Iteratively finds up to 10 “Box-row” blocks.
  - Extracts:
    - `name` (repo path `owner/repo`)
    - `url`
    - `description` (trimmed, whitespace-normalized, truncated)
    - `language`
    - `stars_today`
  - Outputs: `{ trending_repos: [...], trending_count: N }`.
- **Connections:**
  - Input ← **Fetch GitHub Trending Page**
  - Output → **Merge All Data** (input 1)
- **Potential failures / edge cases:**
  - Regex-based parsing is brittle; minor HTML changes can break extraction.
  - “descMatch” selector is heuristic; may return empty description.
- **Version notes:** typeVersion `2`.

---

### Block 5 — Merge, Gate, Compose & Send

**Overview:** Merges the three branch outputs, checks if there is anything to send, builds a combined HTML email, and sends it via Gmail.

**Nodes involved:**
- Merge Releases + Commits
- Merge All Data
- Has Any Content?
- Build Combined Email
- Send Weekly Digest
- Do Nothing
- Commits Section1 (sticky note / email template note)

#### Node: Merge Releases + Commits
- **Type / role:** `Merge` — combines the Releases aggregation and Commits aggregation.
- **Configuration:** Default merge behavior (no explicit mode shown).
- **Connections:**
  - Input 0 ← **Aggregate Releases**
  - Input 1 ← **Extract Commits**
  - Output → **Merge All Data** (input 0)
- **Potential failures / edge cases:**
  - With default merge settings, behavior depends on n8n merge mode defaults; typically it pairs items. Here each side returns a single item, which is compatible.
  - If either side outputs 0 items, merge may output 0 items (depending on mode). That would prevent email sending even if the other branch had content.
- **Version notes:** typeVersion `3.2`.

#### Node: Merge All Data
- **Type / role:** `Merge` — merges (Releases+Commits) with Trending output.
- **Configuration:** Default merge behavior.
- **Connections:**
  - Input 0 ← **Merge Releases + Commits**
  - Input 1 ← **Parse Trending HTML**
  - Output → **Has Any Content?**
- **Potential failures / edge cases:**
  - Same merge-mode dependency risk as above: if Trending parsing returns 0 items, merge could output 0 items depending on configuration defaults.
- **Version notes:** typeVersion `3.2`.

#### Node: Has Any Content?
- **Type / role:** `IF` — gates sending based on section counts.
- **Condition logic (OR):**
  - `release_count > 0` OR `commit_count > 0` OR `trending_count > 0`
- **Connections:**
  - True → **Build Combined Email**
  - False → **Do Nothing**
- **Potential failures / edge cases:**
  - If upstream merge produced no item, this node won’t run.
  - If counts are missing/non-numeric, strict type validation could fail or evaluate unexpectedly.
- **Version notes:** typeVersion `2.2`.

#### Node: Build Combined Email
- **Type / role:** `Code` — creates subject + HTML with three sections.
- **Core logic (interpreted):**
  - Searches all incoming items for:
    - object with `releases`
    - object with `commits`
    - object with `trending_repos`
    - Falls back to empty lists if not found.
  - Formats:
    - Subject: “🤖 Weekly GitHub Digest — <Month Day, Year>”
    - HTML sections with inline styling:
      - Releases (shows “No new releases…” if empty)
      - Commits (shows “No recent commits…” if empty; includes up to 15)
      - Trending (shows “Could not load…” if empty; includes up to 10)
  - Outputs: `{ html, subject, release_count, commit_count, trending_count }`
- **Connections:**
  - Input ← **Has Any Content?** (true branch)
  - Output → **Send Weekly Digest**
- **Potential failures / edge cases:**
  - If upstream merge delivers a single combined item rather than multiple items, “find” still works as long as the properties exist on that item; otherwise sections may appear empty.
  - HTML is not sanitized for injection; safe for trusted GitHub content but still HTML.
- **Version notes:** typeVersion `2`.

#### Node: Send Weekly Digest
- **Type / role:** `Gmail` — sends the email.
- **Configuration (interpreted):**
  - To: `$('Set Variables').first().json.recipient_email`
  - Subject: `$json.subject`
  - Message body: `$json.html` (HTML content)
  - Options: `appendAttribution: false`
  - Credentials: Gmail OAuth2 (“Gmail (Dummy Account)” in template)
- **Connections:**
  - Input ← **Build Combined Email**
  - Output: none (end)
- **Potential failures / edge cases:**
  - OAuth2 expired / missing scopes.
  - Gmail API quotas.
  - If sending to multiple recipients is desired, you must format `sendTo` accordingly.
- **Version notes:** typeVersion `2.2`.

#### Node: Do Nothing
- **Type / role:** `NoOp` — terminal path when there’s nothing to send.
- **Connections:**
  - Input ← **Has Any Content?** (false branch)
- **Version notes:** typeVersion `1`.

#### Node: Commits Section1 (Sticky Note)
- **Type / role:** `Sticky Note` — documents the email template block.
- **Content:** Describes that it aggregates all three branches into HTML and outputs `html`, `subject`, and counts; invites modifications.
- **Connections:** none.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Describes workflow purpose & setup |  |  | ## 🤖 Weekly GitHub Digest; Sends one combined weekly email with Releases/Commits/Trending; Setup steps and tips (GitHub token, tracked_entities accepts usernames and repos) |
| Every Monday at 9am | Schedule Trigger | Weekly trigger (Mon 9am) |  | Set Variables |  |
| Set Variables | Set | Stores recipient/lookback/repos/entities | Every Monday at 9am | Split Watched Repos; Split Tracked Entities; Fetch GitHub Trending Page | ## 🤖 Weekly GitHub Digest; Sends one combined weekly email with Releases/Commits/Trending; Setup steps and tips (GitHub token, tracked_entities accepts usernames and repos) |
| Releases Section | Sticky Note | Documents releases branch |  |  | ## Releases; Fetches latest release per watched repo within lookback window. |
| Split Watched Repos | Split Out | Iterates through `release_repos` array | Set Variables | Fetch Latest Release | ## Releases; Fetches latest release per watched repo within lookback window. |
| Fetch Latest Release | HTTP Request | Calls GitHub releases API per repo | Split Watched Repos | Aggregate Releases | ## Releases; Fetches latest release per watched repo within lookback window.; #### Important GitHub’s /releases/latest skips prereleases; use `/releases/latest` if you only want usual releases |
| Sticky Note | Sticky Note | Notes about `/releases/latest` vs prereleases |  |  | #### Important GitHub’s /releases/latest endpoint skips anything flagged as prerelease; use `https://api.github.com/repos/repo-name/releases/latest` if desired |
| Aggregate Releases | Code | Filters/aggregates releases into single list + count | Fetch Latest Release | Merge Releases + Commits | ## Releases; Fetches latest release per watched repo within lookback window. |
| Commits Section | Sticky Note | Documents commits branch |  |  | ## Commits; Supports users (`torvalds`) and repos (`huggingface/transformers`). |
| Split Tracked Entities | Split Out | Iterates through `tracked_entities` array | Set Variables | Fetch Entity Data | ## Commits; Supports users (`torvalds`) and repos (`huggingface/transformers`). |
| Fetch Entity Data | HTTP Request | Calls GitHub user events or repo commits | Split Tracked Entities | Extract Commits | ## Commits; Supports users (`torvalds`) and repos (`huggingface/transformers`). |
| Extract Commits | Code | Normalizes commits/events into list + count | Fetch Entity Data | Merge Releases + Commits | ## Commits; Supports users (`torvalds`) and repos (`huggingface/transformers`). |
| Trending Section | Sticky Note | Documents trending branch |  |  | ## GitHub Trending Today; Scrapes GitHub trending page for the day. |
| Fetch GitHub Trending Page | HTTP Request | Downloads trending HTML | Set Variables | Parse Trending HTML | ## GitHub Trending Today; Scrapes GitHub trending page for the day. |
| Parse Trending HTML | Code | Parses HTML to trending repo list + count | Fetch GitHub Trending Page | Merge All Data | ## GitHub Trending Today; Scrapes GitHub trending page for the day. |
| Merge Releases + Commits | Merge | Combines releases + commits payloads | Aggregate Releases; Extract Commits | Merge All Data |  |
| Merge All Data | Merge | Combines (releases+commits) with trending | Merge Releases + Commits; Parse Trending HTML | Has Any Content? |  |
| Has Any Content? | IF | Sends only if any count > 0 | Merge All Data | Build Combined Email (true); Do Nothing (false) |  |
| Build Combined Email | Code | Builds final HTML email + subject | Has Any Content? (true) | Send Weekly Digest | ## Email Template; Aggregates output into HTML. Output fields: `html`, `subject`, counts. Feel free to modify. |
| Send Weekly Digest | Gmail | Sends email via Gmail OAuth2 | Build Combined Email |  | ## Email Template; Aggregates output into HTML. Output fields: `html`, `subject`, counts. Feel free to modify. |
| Do Nothing | NoOp | Ends run if no content | Has Any Content? (false) |  |  |
| Commits Section1 | Sticky Note | Documents email template outputs |  |  | ## Email Template; Aggregates output into HTML. Output fields: `html`, `subject`, `release_count`,`commit_count`, `trending_count`. Feel free to modify this template. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Schedule Trigger**
   2. Set to: Weekly → **Monday** at **09:00**
   3. Name it: **Every Monday at 9am**

2) **Add configuration node**
   1. Add node: **Set**
   2. Name: **Set Variables**
   3. Add fields (keep types):
      - `recipient_email` (String) = `user@example.com`
      - `days_back` (Number) = `7`
      - `release_repos` (Array) = expression like `{{ ['n8n-io/n8n'] }}`
      - `tracked_entities` (Array) = expression like `{{ ['anthropics/claude','huggingface/transformers'] }}`
   4. Connect: **Every Monday at 9am → Set Variables**

3) **Releases branch**
   1. Add node: **Split Out** → name **Split Watched Repos**
      - Field to split out: `release_repos`
   2. Add node: **HTTP Request** → name **Fetch Latest Release**
      - Method: GET
      - URL: `https://api.github.com/repos/{{ $json.release_repos }}/releases?per_page=1`
      - Send Headers: true
      - Headers:
        - `Accept: application/vnd.github+json`
        - `X-GitHub-Api-Version: 2022-11-28`
      - On Error: **Continue (regular output)**
   3. Add node: **Code** → name **Aggregate Releases**
      - Paste logic equivalent to: compute cutoff from `days_back`, pick most recent non-draft release newer than cutoff, output `{releases, release_count}`.
      - Ensure it references config via `$('Set Variables').first().json`.
   4. Connect:
      - **Set Variables → Split Watched Repos → Fetch Latest Release → Aggregate Releases**

4) **Commits branch**
   1. Add node: **Split Out** → name **Split Tracked Entities**
      - Field to split out: `tracked_entities`
   2. Add node: **HTTP Request** → name **Fetch Entity Data**
      - Method: GET
      - URL (expression) selecting endpoint:
        - Repo entity (`owner/repo`): `/repos/<entity>/commits?per_page=20&since=<ISO>`
        - User entity: `/users/<entity>/events/public?per_page=30`
      - Reuse headers:
        - `Accept: application/vnd.github+json`
        - `X-GitHub-Api-Version: 2022-11-28`
      - On Error: **Continue (regular output)**
   3. Add node: **Code** → name **Extract Commits**
      - Implement: cutoff by `days_back`, extract commits from PushEvents and/or commit objects, de-dup by sha, output `{commits, commit_count}`.
      - (Recommended when recreating: explicitly handle API responses that are arrays.)
   4. Connect:
      - **Set Variables → Split Tracked Entities → Fetch Entity Data → Extract Commits**

5) **Trending branch**
   1. Add node: **HTTP Request** → name **Fetch GitHub Trending Page**
      - URL: `https://github.com/trending?since=daily`
      - Send headers: true
      - Headers:
        - `User-Agent`: a modern browser UA
        - `Accept`: `text/html,application/xhtml+xml`
      - On Error: **Continue (regular output)**
   2. Add node: **Code** → name **Parse Trending HTML**
      - Parse the HTML from `json.data` or `json.body`
      - Extract up to 10 repos into `{trending_repos, trending_count}`
   3. Connect:
      - **Set Variables → Fetch GitHub Trending Page → Parse Trending HTML**

6) **Merge branches**
   1. Add node: **Merge** → name **Merge Releases + Commits**
      - Connect:
        - Input 1: **Aggregate Releases**
        - Input 2: **Extract Commits**
   2. Add node: **Merge** → name **Merge All Data**
      - Connect:
        - Input 1: **Merge Releases + Commits**
        - Input 2: **Parse Trending HTML**

7) **Gate sending**
   1. Add node: **IF** → name **Has Any Content?**
   2. Add OR conditions:
      - `{{$json.release_count}} > 0`
      - `{{$json.commit_count}} > 0`
      - `{{$json.trending_count}} > 0`
   3. Connect: **Merge All Data → Has Any Content?**

8) **Build email**
   1. Add node: **Code** → name **Build Combined Email**
   2. Generate:
      - `subject` (string)
      - `html` (string)
      - counts
   3. Connect: **Has Any Content? (true) → Build Combined Email**

9) **Send email**
   1. Add node: **Gmail** → name **Send Weekly Digest**
   2. Configure:
      - To: `{{$('Set Variables').first().json.recipient_email}}`
      - Subject: `{{$json.subject}}`
      - Message: `{{$json.html}}`
      - Disable attribution if desired
   3. **Credentials:** create/select **Gmail OAuth2** credential and authorize.
   4. Connect: **Build Combined Email → Send Weekly Digest**

10) **No-content path**
   1. Add node: **No Operation (NoOp)** → name **Do Nothing**
   2. Connect: **Has Any Content? (false) → Do Nothing**

11) **Optional: GitHub authentication improvement**
   - In both GitHub API HTTP Request nodes, add header:
     - `Authorization: Bearer <YOUR_GITHUB_TOKEN>`
   - This significantly increases rate limits and improves reliability for private repos (if token has access).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Only sends if at least one section has content.” | Workflow design (IF gate before email send) |
| Tip: “Add a GitHub token header for 5000 req/hr limit” | Recommended to avoid rate limiting on GitHub API |
| `tracked_entities` accepts usernames AND repo paths | Used in commits branch URL selection logic |
| GitHub `/releases/latest` skips prereleases; workflow uses `/releases?per_page=1` to include prereleases | Sticky note in Releases block |
| Footer credit: “Weekly digest created by Dahiana Porto via N8N” | Embedded in the email HTML template |
| Trending is scraped from GitHub HTML and can break if markup changes | Trending parsing uses regex/heuristics |

