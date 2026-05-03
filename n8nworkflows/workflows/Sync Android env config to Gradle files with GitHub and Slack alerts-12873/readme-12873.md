Sync Android env config to Gradle files with GitHub and Slack alerts

https://n8nworkflows.xyz/workflows/sync-android-env-config-to-gradle-files-with-github-and-slack-alerts-12873


# Sync Android env config to Gradle files with GitHub and Slack alerts

## 1. Workflow Overview

**Title (given):** Sync Android env config to Gradle files with GitHub and Slack alerts  
**Workflow name (JSON):** Environment Config Diff & Propagate for Android Builds

**Purpose:**  
Automatically detects when `.env.staging` changes in a GitHub repository, then propagates matching environment variable values into Android build configuration files (`app/build.gradle` and `gradle.properties`) on a newly created branch. It commits the updates, opens a pull request to `main`, and notifies a Slack channel with the PR link.

**Target use cases:**
- Keep Android Gradle configuration in sync with a source-of-truth `.env.staging`.
- Prevent manual drift between `.env` values and Gradle config.
- Enforce review via PR + team notification.

### Logical Blocks
**1.1 Webhook intake & change gating**  
Receives GitHub push events and continues only if `.env.staging` was modified.

**1.2 Fetch + parse source-of-truth and current Gradle props**  
Downloads `.env.staging` and `gradle.properties` from `main`, parses both into key/value maps.

**1.3 Compare variables + prepare branch**  
Computes differences, generates a new branch name, reads latest `main` SHA, creates a branch.

**1.4 Fetch & decode files on the new branch for targeted diff**  
Fetches `.env.staging`, `app/build.gradle`, and `gradle.properties` (branch context), decodes them to text and parses `.env`.

**1.5 Apply updates + commit to branch**  
Rewrites `app/build.gradle` buildConfigField values and updates/creates keys in `gradle.properties`, then commits both files to the new branch.

**1.6 Pull request + Slack notification**  
Creates a PR and posts details to Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Webhook intake & change gating

**Overview:**  
This block starts the workflow from a GitHub push webhook and determines whether `.env.staging` changed. If not, the workflow stops early.

**Nodes involved:**
- Receive GitHub Webhook
- Check if .env.staging Changed
- Proceed Only if ENV Changed

#### Node: Receive GitHub Webhook
- **Type / role:** Webhook Trigger (`n8n-nodes-base.webhook`) — entry point.
- **Config (interpreted):**
  - HTTP Method: `POST`
  - Path: `env-config-diff`
  - Produces a webhook URL like `/webhook/env-config-diff` (and `/webhook-test/...` for test).
- **Inputs / outputs:** No inputs; outputs the HTTP request payload.
- **Edge cases / failures:**
  - GitHub webhook secret not validated (none configured) → consider adding signature validation.
  - Payload format differences: n8n test mode wraps payload under `body`, which is handled downstream.
  - If GitHub sends `ping` event or non-push event, downstream logic may misbehave.
- **Version notes:** Node typeVersion `2.1`.

#### Node: Check if .env.staging Changed
- **Type / role:** Code node — inspects commits for modified files.
- **Key logic:**
  - Uses: `const payload = $input.first().json.body ?? $input.first().json;`
  - Aggregates `commit.added`, `commit.modified`, `commit.removed`.
  - Sets `envFileChanged = changedFiles.includes('.env.staging')`
- **Outputs:**  
  `repository`, `branch` (ref), `envFileChanged` (boolean), `changedFiles` array.
- **Edge cases / failures:**
  - Large pushes: GitHub push payload may truncate commit list depending on event size; consider using GitHub Compare API if accuracy matters.
  - `.env.staging` path differences (e.g., `config/.env.staging`) won’t match; this is strict equality.

#### Node: Proceed Only if ENV Changed
- **Type / role:** IF node — gate.
- **Condition:** `{{ $json.envFileChanged }}` is `true`.
- **Outputs:**  
  - **True** path continues to fetch files; **False** path ends (no nodes connected).
- **Edge cases / failures:**
  - If `envFileChanged` is missing/not boolean, strict validation may fail (typeValidation is strict).

---

### 2.2 Fetch + parse source-of-truth and current Gradle props

**Overview:**  
Downloads `.env.staging` and `gradle.properties` from `main` and converts them into key/value objects for comparison.

**Nodes involved:**
- Fetch .env.staging from Repo
- Parse .env.staging into Key-Value
- Fetch gradle.properties from Repo
- Parse gradle.properties into Key-Value
- Combine Env & Gradle

#### Node: Fetch .env.staging from Repo
- **Type / role:** GitHub node (File → Get) — reads file from repository.
- **Config:**
  - Owner: `tanvidg2025`
  - Repository: `android-config-sync-n8nDemo` (note: expression `=android-config-sync-n8nDemo` in JSON; effectively constant)
  - File path: `.env.staging`
  - Reference: `main`
  - Returns Base64 content in `json.content`
- **Credentials:** `GitHub account 8` (GitHub API)
- **Edge cases / failures:**
  - 404 if file does not exist on `main`.
  - Permission/auth errors if token lacks `repo` scope for private repos.
  - Default branch mismatch (if default isn’t `main`, reference may fail).

#### Node: Parse .env.staging into Key-Value
- **Type / role:** Code node — Base64 decode + parse `KEY=VALUE`.
- **Logic details:**
  - Decodes `json.content` from Base64 into `decodedText`.
  - Ignores empty lines and `#` comments.
  - Splits on first `=` (supports values containing `=` by joining remainder).
- **Output:**
  - `rawEnv`: full decoded text
  - `envVars`: object map `{ KEY: VALUE }`
- **Edge cases / failures:**
  - Lines like `export KEY=VALUE` will parse with key `export KEY` (not desired).
  - Quoted values not unquoted; whitespace not trimmed around key.
  - Duplicate keys: last one wins silently.

#### Node: Fetch gradle.properties from Repo
- **Type / role:** GitHub node (File → Get)
- **Config:** same repo/owner, file path `gradle.properties`, reference `main`.
- **Output:** Base64 content in `json.content`.
- **Edge cases:** same as above.

#### Node: Parse gradle.properties into Key-Value
- **Type / role:** Code node — decode + parse properties format.
- **Logic details:**
  - Similar to `.env` parsing; ignores comments/empty lines; splits on `=`.
- **Output:**
  - `rawGradle`: decoded text
  - `gradleVars`: object map
- **Edge cases / failures:**
  - `gradle.properties` commonly supports `:` separators and escaped characters; this simplistic parser won’t fully match Gradle property parsing rules.
  - Keys with dots (`android.useAndroidX`) work fine; whitespace around `=` not trimmed from key.

#### Node: Combine Env & Gradle
- **Type / role:** Merge node — combines both parsed outputs into one stream.
- **Config:** default merge behavior (no explicit mode set in JSON).
- **Inputs / outputs:**
  - Input 1: from “Parse .env.staging into Key-Value”
  - Input 2: from “Parse gradle.properties into Key-Value”
  - Output: combined items to comparison node
- **Edge cases / failures:**
  - If merge mode is not set appropriately, item alignment can be problematic. The downstream comparison assumes `$input.all()` returns two items with env first and gradle second.

---

### 2.3 Compare variables + prepare branch

**Overview:**  
Determines which env keys differ from `gradle.properties`, then creates a new branch off `main` for safe updates.

**Nodes involved:**
- Compare ENV and Gradle Variables
- Generate New Branch Name
- Get Latest SHA of main Branch
- Prepare SHA for Branch Creation
- Create New Git Branch

#### Node: Compare ENV and Gradle Variables
- **Type / role:** Code node — computes differences.
- **Inputs:** expects two items from merge:
  - `inputArray[0].envVars`
  - `inputArray[1].gradleVars`
- **Output fields:**
  - `changes`: array of `{ key, envValue, gradleValue, reason }`
  - `hasChanges`: boolean
  - `needsCacheInvalidation`: always includes keys in `['API_KEY','BUILD_VERSION','ENVIRONMENT']` if they exist in env (note: it does not check whether they changed)
  - `envVars`, `gradleVars`
- **Important note:**  
  The workflow does **not** currently gate on `hasChanges`. Even if `changes` is empty, it will still create a branch, attempt edits, and open a PR (unless later steps end up making no edits).
- **Edge cases / failures:**
  - If merge returns items in opposite order, comparison breaks silently.
  - Missing `envVars` or `gradleVars` → defaults to `{}` and may produce no changes.

#### Node: Generate New Branch Name
- **Type / role:** Code node — creates unique branch name.
- **Logic:** `config-sync/<ISO timestamp with : and . replaced>`
- **Output:** passes through previous data + `branchName`.
- **Edge cases:**  
  - Branch names are safe; ISO time includes `Z` but that’s allowed.
  - If multiple runs in the same millisecond, collision is still possible but unlikely.

#### Node: Get Latest SHA of main Branch
- **Type / role:** HTTP Request — fetches the SHA of `main` to branch from.
- **Config concerns (as provided):**
  - URL is placeholder: `{{Your_Github_URL_Here}}`
  - Sends `Authorization: Bearar ghp_YOUR_GITHUB_TOKEN_HERE` (typo: should be **Bearer**, and token should not be embedded in node parameters when using credentials)
  - Authentication set to `predefinedCredentialType` with `nodeCredentialType: githubApi`
- **Expected intent:** call GitHub REST API to get ref for `main`, e.g.:  
  `GET https://api.github.com/repos/<owner>/<repo>/git/ref/heads/main`
- **Edge cases / failures:**
  - 401 due to `Bearar` typo if header is used.
  - 404 if URL is wrong or branch not `main`.
  - Rate limiting / API errors.
- **Version notes:** HTTP Request node typeVersion `4.3`.

#### Node: Prepare SHA for Branch Creation
- **Type / role:** Set node — extracts SHA.
- **Config:** sets `commitSHA = {{$json["object"]["sha"]}}`
- **Dependencies:** requires the previous HTTP response shape to include `object.sha` (as GitHub refs do).
- **Edge cases:**  
  If the API response shape differs (e.g., using a commits API), expression fails or produces undefined.

#### Node: Create New Git Branch
- **Type / role:** HTTP Request — creates a new ref.
- **Config:**
  - URL placeholder `{{Your_Github_URL_Here}}` (intended: `POST /repos/<owner>/<repo>/git/refs`)
  - JSON body:
    - `ref`: `refs/heads/<branchName>`
    - `sha`: from `commitSHA`
  - Uses GitHub credential type.
- **Output expectation:** GitHub returns created ref (`ref`, `object.sha`, etc.).
- **Downstream dependency:** Next GitHub “Fetch .env.staging from New Branch” uses `reference: {{ $json.ref }}`. GitHub returns `ref` like `refs/heads/config-sync/...`.
- **Edge cases / failures:**
  - 422 if branch already exists (name collision).
  - 403 if token lacks permission.
  - Wrong base SHA → branch from unexpected commit.

---

### 2.4 Fetch & decode files on the new branch for targeted diff

**Overview:**  
Once the branch exists, fetches the files to be updated and determines which keys should be applied to `build.gradle` and `gradle.properties`.

**Nodes involved:**
- Fetch .env.staging from New Branch
- Decode .env.staging
- Fetch build.gradle File
- Decode build.gradle
- Fetch gradle.properties File
- Decode gradle.properties
- Combine Decoded Files for Diff
- Identify Config Changes

#### Node: Fetch .env.staging from New Branch
- **Type / role:** GitHub node (File → Get) — reads `.env.staging` from the newly created branch.
- **Config:** `reference = {{ $json.ref }}` from “Create New Git Branch”.
- **Edge cases / failures:**
  - If `ref` is not a valid reference string for this node (some APIs expect a branch name, not `refs/heads/...`), it may fail. Many GitHub endpoints accept either, but this is worth validating.

#### Node: Decode .env.staging
- **Type / role:** Code node — Base64 decode + parse into object.
- **Output:** `raw` and `parsed`.
- **Note:** Output keys differ from earlier parse node (`rawEnv/envVars` vs `raw/parsed`). Downstream “Identify Config Changes” uses `items[0].json.parsed`.

#### Node: Fetch build.gradle File
- **Type / role:** GitHub node (File → Get)
- **Config:** file path `app/build.gradle`
- **Important:** No explicit branch/reference configured here, so it will fetch the repository’s default branch (often `main`), not the newly created branch.
  - This may still work for editing because commits are made explicitly to the new branch later, but it means the “diff/identify changes” step is not guaranteed to use the branch’s file versions.
- **Edge cases:** default branch not `main`, or mismatched state.

#### Node: Decode build.gradle
- **Type / role:** Code node — decodes content, but parses it as if it were `KEY=VALUE`.
- **Reality check:** `app/build.gradle` is not a properties file; the parsing result is not meaningful. However, downstream uses **`raw` text**, which is correct.
- **Output:** `raw` and `parsed` (parsed is mostly irrelevant).

#### Node: Fetch gradle.properties File
- **Type / role:** GitHub node (File → Get)
- **Config:** file path `gradle.properties`
- **Note:** also lacks explicit branch reference, same concern as build.gradle fetch.

#### Node: Decode gradle.properties
- **Type / role:** Code node — decodes and parses `KEY=VALUE`.
- **Output:** `raw` and `parsed`.
- **Downstream usage:** “Identify Config Changes” uses `raw` text only.

#### Node: Combine Decoded Files for Diff
- **Type / role:** Merge node — combines three decoded outputs.
- **Config:** `numberInputs: 3` (explicit).
- **Order expected by downstream:**
  1. Decoded `.env.staging` (items[0])
  2. Decoded `build.gradle` (items[1])
  3. Decoded `gradle.properties` (items[2])
- **Edge cases:** If merge order differs, “Identify Config Changes” will misinterpret inputs and may throw or apply wrong logic.

#### Node: Identify Config Changes
- **Type / role:** Code node — decides which env keys apply to which target file.
- **Logic:**
  - Validates exactly 3 inputs; otherwise throws.
  - For each env key:
    - If `buildGradleText.includes("\"<KEY>\"")`, it marks it for build.gradle update.
    - If `gradlePropsText.includes("<KEY>=")`, it marks it for gradle.properties update.
- **Output:** `{ updates: { buildGradle: {...}, gradleProperties: {...} }, hasChanges }`
- **Edge cases / failures:**
  - `includes("\"KEY\"")` is a blunt heuristic. If the key appears elsewhere (comments/other strings), it may incorrectly update.
  - If the key exists in `.env.staging` but not present in `gradle.properties`, it will **not** be added here (because it only updates if it already includes `${key}=`). However, the later “Apply ENV Changes to gradle.properties” *does* add missing keys—but it only receives keys included in `updates.gradleProperties`, which currently only includes existing keys. So new keys in env will **not** be appended to gradle.properties under this logic.
  - Similar for build.gradle: only updates keys already referenced as `"KEY"` somewhere.

---

### 2.5 Apply updates + commit to branch

**Overview:**  
Edits file text using regex replacements, then commits both files to the newly created branch.

**Nodes involved:**
- Apply ENV Changes to build.gradle
- Apply ENV Changes to gradle.properties
- Commit build.gradle Updates to Branch
- Commit gradle.properties Updates to Branch

#### Node: Apply ENV Changes to build.gradle
- **Type / role:** Code node — regex replace `buildConfigField` values.
- **Inputs referenced via `$items()`:**
  - Updates from `Identify Config Changes` → `updates.buildGradle`
  - Raw build.gradle from `Decode build.gradle` → `raw`
- **Replacement logic:**
  - Regex: `(buildConfigField\s+"String",\s+"KEY",\s+"\\")(.*?)(\")`
  - Replaces captured value with env value.
- **Output:** object with:
  - `path: "app/build.gradle"`
  - `content: <base64 of updated text>`
  - `message: "Sync env vars to build.gradle"`
- **Edge cases / failures:**
  - Only updates buildConfigField with `"String"` type; other types ignored.
  - If Gradle formatting differs (single quotes, spacing, multiline), regex may not match.
  - If value contains characters needing escaping in Gradle string literals, this does not escape them.

#### Node: Apply ENV Changes to gradle.properties
- **Type / role:** Code node — replace or append `KEY=value` lines.
- **Inputs referenced via `$items()`:**
  - Updates from `Identify Config Changes` → `updates.gradleProperties`
  - Raw props from `Decode gradle.properties` → `raw`
- **Logic:**
  - For each key, replace `^KEY=.*$` (multiline) or append `\nKEY=value`.
- **Output:** `{ path: "gradle.properties", content: base64, message }`
- **Edge cases / failures:**
  - As noted earlier, keys missing from file may never be in `updates.gradleProperties` due to the “Identify Config Changes” includes-check, so the append path might never run for new keys.
  - Does not preserve ordering or comments near the key.

#### Node: Commit build.gradle Updates to Branch
- **Type / role:** GitHub node (File → Edit) — commits file change.
- **Config:**
  - File path and content from previous node (`{{$json.path}}`, `{{$json.content}}`)
  - Commit message `{{$json.message}}`
  - Branch: `{{ $('Generate New Branch Name').first().json.branchName }}`
- **Edge cases / failures:**
  - GitHub “edit file” often requires the file SHA of the existing blob. n8n’s GitHub node typically handles this internally, but if not, it can fail.
  - If branch doesn’t exist or name mismatch → 404.

#### Node: Commit gradle.properties Updates to Branch
- **Type / role:** GitHub node (File → Edit)
- **Config:** analogous to build.gradle commit.
- **Edge cases:** same.

---

### 2.6 Pull request + Slack notification

**Overview:**  
Creates a PR from the new branch into `main` and posts a Slack message to alert the team.

**Nodes involved:**
- Prepare Pull Request Data
- Create Pull Request
- Notify Team on Slack

#### Node: Prepare Pull Request Data
- **Type / role:** Set node — produces `branchName` for PR creation.
- **Value:** `{{ $('Generate New Branch Name').first().json.branchName }}`
- **Edge cases:**  
  If earlier node didn’t run or data was not available, expression fails.

#### Node: Create Pull Request
- **Type / role:** HTTP Request — calls GitHub API to open PR.
- **Config concerns (as provided):**
  - URL placeholder: `{{Your_Github_URL_Here}}` (intended: `POST /repos/<owner>/<repo>/pulls`)
  - Body:
    - title: “Sync Android config files”
    - head: `{{ $json.branchName }}`
    - base: `main`
  - Header: `Accept: application/vnd.github+json`
- **Output expectation:** PR object with `html_url`.
- **Edge cases / failures:**
  - 422 if PR already exists for branch, or head/base invalid.
  - 401/403 if token permissions insufficient.

#### Node: Notify Team on Slack
- **Type / role:** Slack node — posts message to a channel.
- **Config:**
  - Channel: `n8n` (ID `C09S57E2JQ2`)
  - Text template uses:
    - `{{ $json.branchName }}`
    - `{{ $json.pullRequest.html_url }}`
- **Potential mismatch:**  
  The HTTP Request response from “Create Pull Request” will likely have `html_url` at the top level (GitHub returns `{ html_url: ... }`), not nested under `pullRequest`. Unless n8n wraps it, `{{ $json.pullRequest.html_url }}` may be undefined. Expect to change to `{{ $json.html_url }}` (or map response in a Set node).
- **Edge cases / failures:**
  - Slack auth/token scopes missing (`chat:write`).
  - Channel ID invalid or bot not in channel.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive GitHub Webhook | Webhook | Entry point: receive GitHub push event | — | Check if .env.staging Changed | Listens for changes in the repository, like when someone updates files. This starts the workflow automatically. |
| Check if .env.staging Changed | Code | Parse webhook payload; detect `.env.staging` changes | Receive GitHub Webhook | Proceed Only if ENV Changed | ## Check if .env.staging Changed + Proceed Only if ENV Changed  \nChecks whether the .env.staging file was modified in the recent commit. The workflow continues only if this file has changed. |
| Proceed Only if ENV Changed | IF | Gate workflow execution | Check if .env.staging Changed | Fetch .env.staging from Repo; Fetch gradle.properties from Repo | ## Check if .env.staging Changed + Proceed Only if ENV Changed  \nChecks whether the .env.staging file was modified in the recent commit. The workflow continues only if this file has changed. |
| Fetch .env.staging from Repo | GitHub | Get `.env.staging` from main | Proceed Only if ENV Changed | Parse .env.staging into Key-Value | ## Fetch & Parse Files \nFetches the .env.staging and gradle.properties files from the repository and converts them into a simple key-value format so we can easily compare and use the values. |
| Parse .env.staging into Key-Value | Code | Decode + parse env vars | Fetch .env.staging from Repo | Combine Env & Gradle | ## Fetch & Parse Files \nFetches the .env.staging and gradle.properties files from the repository and converts them into a simple key-value format so we can easily compare and use the values. |
| Fetch gradle.properties from Repo | GitHub | Get `gradle.properties` from main | Proceed Only if ENV Changed | Parse gradle.properties into Key-Value | ## Fetch & Parse Files \nFetches the .env.staging and gradle.properties files from the repository and converts them into a simple key-value format so we can easily compare and use the values. |
| Parse gradle.properties into Key-Value | Code | Decode + parse gradle properties | Fetch gradle.properties from Repo | Combine Env & Gradle | ## Fetch & Parse Files \nFetches the .env.staging and gradle.properties files from the repository and converts them into a simple key-value format so we can easily compare and use the values. |
| Combine Env & Gradle | Merge | Combine parsed env + gradle vars | Parse .env.staging into Key-Value; Parse gradle.properties into Key-Value | Compare ENV and Gradle Variables | ## Combine & Prepare Branch\nCompares the values in .env.staging and gradle.properties to see what needs updating. Then, creates a new branch from the main branch to apply these updates safely. |
| Compare ENV and Gradle Variables | Code | Compute differences; flag change set | Combine Env & Gradle | Generate New Branch Name | ## Combine & Prepare Branch\nCompares the values in .env.staging and gradle.properties to see what needs updating. Then, creates a new branch from the main branch to apply these updates safely. |
| Generate New Branch Name | Code | Create unique branch name | Compare ENV and Gradle Variables | Get Latest SHA of main Branch | ## Combine & Prepare Branch\nCompares the values in .env.staging and gradle.properties to see what needs updating. Then, creates a new branch from the main branch to apply these updates safely. |
| Get Latest SHA of main Branch | HTTP Request | Get base SHA for branching | Generate New Branch Name | Prepare SHA for Branch Creation | ## Combine & Prepare Branch\nCompares the values in .env.staging and gradle.properties to see what needs updating. Then, creates a new branch from the main branch to apply these updates safely. |
| Prepare SHA for Branch Creation | Set | Extract `object.sha` as commitSHA | Get Latest SHA of main Branch | Create New Git Branch | ## Combine & Prepare Branch\nCompares the values in .env.staging and gradle.properties to see what needs updating. Then, creates a new branch from the main branch to apply these updates safely. |
| Create New Git Branch | HTTP Request | Create new branch ref | Prepare SHA for Branch Creation | Fetch .env.staging from New Branch; Fetch build.gradle File; Fetch gradle.properties File | ## Combine & Prepare Branch\nCompares the values in .env.staging and gradle.properties to see what needs updating. Then, creates a new branch from the main branch to apply these updates safely. |
| Fetch .env.staging from New Branch | GitHub | Get env file from new branch | Create New Git Branch | Decode .env.staging | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Decode .env.staging | Code | Decode + parse env file content | Fetch .env.staging from New Branch | Combine Decoded Files for Diff | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Fetch build.gradle File | GitHub | Get `app/build.gradle` | Create New Git Branch | Decode build.gradle | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Decode build.gradle | Code | Decode build.gradle content to raw text | Fetch build.gradle File | Combine Decoded Files for Diff | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Fetch gradle.properties File | GitHub | Get `gradle.properties` | Create New Git Branch | Decode gradle.properties | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Decode gradle.properties | Code | Decode gradle.properties to raw text | Fetch gradle.properties File | Combine Decoded Files for Diff | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Combine Decoded Files for Diff | Merge | Merge decoded env/build.gradle/props | Decode .env.staging; Decode build.gradle; Decode gradle.properties | Identify Config Changes | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Identify Config Changes | Code | Determine which keys affect which target file | Combine Decoded Files for Diff | Apply ENV Changes to build.gradle; Apply ENV Changes to gradle.properties | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Apply ENV Changes to build.gradle | Code | Regex-update buildConfigField values | Identify Config Changes (via `$items()`); Decode build.gradle (via `$items()`) | Commit build.gradle Updates to Branch | ## Update Files\n\nUpdates the build.gradle and gradle.properties files with the new values from .env.staging and commits these changes to the new branch.\n |
| Apply ENV Changes to gradle.properties | Code | Replace/append properties lines | Identify Config Changes (via `$items()`); Decode gradle.properties (via `$items()`) | Commit gradle.properties Updates to Branch | ## Update Files\n\nUpdates the build.gradle and gradle.properties files with the new values from .env.staging and commits these changes to the new branch.\n |
| Commit build.gradle Updates to Branch | GitHub | Commit edited build.gradle to branch | Apply ENV Changes to build.gradle | Prepare Pull Request Data | ## Update Files\n\nUpdates the build.gradle and gradle.properties files with the new values from .env.staging and commits these changes to the new branch.\n |
| Commit gradle.properties Updates to Branch | GitHub | Commit edited gradle.properties to branch | Apply ENV Changes to gradle.properties | Prepare Pull Request Data | ## Update Files\n\nUpdates the build.gradle and gradle.properties files with the new values from .env.staging and commits these changes to the new branch.\n |
| Prepare Pull Request Data | Set | Provide branchName for PR request | Commit build.gradle Updates to Branch; Commit gradle.properties Updates to Branch | Create Pull Request | ## Create Pull Request & Notify\n\nPrepares and creates a pull request for the updated files and sends a message to the team on Slack to let them know a new PR is ready for review.\n |
| Create Pull Request | HTTP Request | Open PR to main | Prepare Pull Request Data | Notify Team on Slack | ## Create Pull Request & Notify\n\nPrepares and creates a pull request for the updated files and sends a message to the team on Slack to let them know a new PR is ready for review.\n |
| Notify Team on Slack | Slack | Notify channel with PR link | Create Pull Request | — | ## Create Pull Request & Notify\n\nPrepares and creates a pull request for the updated files and sends a message to the team on Slack to let them know a new PR is ready for review.\n |
| Sticky Note | Sticky Note | Comment | — | — | Listens for changes in the repository, like when someone updates files. This starts the workflow automatically. |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Check if .env.staging Changed + Proceed Only if ENV Changed \nChecks whether the .env.staging file was modified in the recent commit. The workflow continues only if this file has changed. |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Fetch & Parse Files \nFetches the .env.staging and gradle.properties files from the repository and converts them into a simple key-value format so we can easily compare and use the values. |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Combine & Prepare Branch\nCompares the values in .env.staging and gradle.properties to see what needs updating. Then, creates a new branch from the main branch to apply these updates safely. |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Fetch & Decode Files for Diff\nFetches the latest versions of config files from the new branch, decodes them into readable text and prepares a comparison to see exactly which values need to be updated. |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Update Files\n\nUpdates the build.gradle and gradle.properties files with the new values from .env.staging and commits these changes to the new branch.\n |
| Sticky Note6 | Sticky Note | Comment | — | — | ## Create Pull Request & Notify\n\nPrepares and creates a pull request for the updated files and sends a message to the team on Slack to let them know a new PR is ready for review.\n |
| Sticky Note7 | Sticky Note | Comment | — | — | ## How it works\n\nThis workflow starts automatically when a change is pushed to GitHub. It first checks whether the .env.staging file was modified and if not, the workflow stops immediately. When .env.staging has changed, the workflow reads the environment values and compares them with the existing values in build.gradle and gradle.properties. If differences are found, it creates a new branch from the main branch, applies only the required updates to the Android configuration files and commits those changes to the new branch. After committing, it creates a pull request so the changes can be reviewed safely before merging. Finally, a Slack notification is sent to inform the team that a configuration update pull request is ready.\n\n\n## Setup steps\n\n**1.** Import this workflow into n8n.\n\n**2.** Connect your GitHub credentials.\n\n**3.** Connect your Slack credentials and select a channel.\n\n**4.** Make sure .env.staging, build.gradle and gradle.properties exist in the repo.\n\n**5.** Activate the workflow.\n\n**6.** Push a change to .env.staging to test. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name: `Environment Config Diff & Propagate for Android Builds` (or your preferred name)

2) **Add trigger: “Webhook”**
   - Node: **Webhook**
   - HTTP Method: `POST`
   - Path: `env-config-diff`
   - Save node, copy the **Production** webhook URL.

3) **Configure GitHub webhook**
   - In GitHub repo settings → Webhooks → Add webhook
   - Payload URL: the n8n Production webhook URL
   - Content type: `application/json`
   - Events: “Just the push event”
   - (Optional but recommended) Set a secret; implement signature validation in n8n.

4) **Add Code node: “Check if .env.staging Changed”**
   - Place after Webhook
   - Paste logic that:
     - Reads payload from `{{$json.body ?? $json}}`
     - Collects changed files from commits
     - Sets `envFileChanged` boolean if `.env.staging` is included

5) **Add IF node: “Proceed Only if ENV Changed”**
   - Condition: Boolean → `{{ $json.envFileChanged }}` is `true`
   - Connect from the Code node to this IF node.

6) **Add GitHub credentials**
   - Create a **GitHub API** credential in n8n (OAuth or PAT).
   - Ensure scopes:
     - Private repo: `repo`
     - Public: usually enough with appropriate access; PR creation still requires permissions.

7) **Add GitHub node: “Fetch .env.staging from Repo”**
   - Resource: **File**
   - Operation: **Get**
   - Owner: your org/user
   - Repository: your repo
   - File path: `.env.staging`
   - Reference: `main` (or your base branch)
   - Connect from IF (true path) to this node.

8) **Add Code node: “Parse .env.staging into Key-Value”**
   - Decode Base64 `content`
   - Parse `KEY=VALUE` lines to an object `envVars`
   - Output both raw text and `envVars`

9) **Add GitHub node: “Fetch gradle.properties from Repo”**
   - File path: `gradle.properties`
   - Reference: `main`
   - Connect from IF (true path) to this node too (parallel branch).

10) **Add Code node: “Parse gradle.properties into Key-Value”**
   - Decode and parse to `gradleVars`.

11) **Add Merge node: “Combine Env & Gradle”**
   - Connect:
     - Parse env → Merge input 1
     - Parse gradle.properties → Merge input 2

12) **Add Code node: “Compare ENV and Gradle Variables”**
   - Read both merged items via `$input.all()`
   - Build `changes[]`, `hasChanges`, etc.
   - (Recommended improvement) Add an IF node after this to stop if `hasChanges` is false.

13) **Add Code node: “Generate New Branch Name”**
   - Create `branchName = config-sync/<timestamp>`

14) **Add HTTP Request node: “Get Latest SHA of main Branch”**
   - Method: `GET`
   - URL (example):
     - `https://api.github.com/repos/<owner>/<repo>/git/ref/heads/main`
   - Authentication: use **GitHub API credential** (avoid hardcoding tokens)
   - Headers:
     - `Accept: application/vnd.github+json`

15) **Add Set node: “Prepare SHA for Branch Creation”**
   - Set field `commitSHA = {{ $json.object.sha }}`

16) **Add HTTP Request node: “Create New Git Branch”**
   - Method: `POST`
   - URL (example):
     - `https://api.github.com/repos/<owner>/<repo>/git/refs`
   - Body (JSON):
     - `ref`: `refs/heads/{{ $('Generate New Branch Name').item.json.branchName }}`
     - `sha`: `{{ $('Prepare SHA for Branch Creation').item.json.commitSHA }}`
   - Auth: GitHub credential

17) **Fetch files for editing (ideally from the new branch)**
   - Add GitHub node: “Fetch .env.staging from New Branch”
     - Reference: use the created branch name (recommended: `{{ $('Generate New Branch Name').first().json.branchName }}`).
   - Add GitHub node: “Fetch build.gradle File”
     - File path: `app/build.gradle`
     - Reference: same branch name (recommended).
   - Add GitHub node: “Fetch gradle.properties File”
     - File path: `gradle.properties`
     - Reference: same branch name (recommended).

18) **Add Code nodes to decode**
   - “Decode .env.staging” → output `{ raw, parsed }`
   - “Decode build.gradle” → output `{ raw }` (parsing not required)
   - “Decode gradle.properties” → output `{ raw }` (optional parsed)

19) **Add Merge node: “Combine Decoded Files for Diff”**
   - Set **Number of Inputs = 3**
   - Connect decoded env/build.gradle/gradle.properties.

20) **Add Code node: “Identify Config Changes”**
   - Confirm 3 inputs
   - For each env key, decide updates for:
     - build.gradle (if the key is referenced)
     - gradle.properties
   - (Recommended improvement) include new keys rather than only existing matches.

21) **Add Code node: “Apply ENV Changes to build.gradle”**
   - Replace `buildConfigField "String", "KEY", "\"value\""` values via regex
   - Output `path`, Base64 `content`, `message`

22) **Add Code node: “Apply ENV Changes to gradle.properties”**
   - Replace `KEY=` lines or append missing keys (ensure missing keys are included in updates)
   - Output `path`, Base64 `content`, `message`

23) **Add GitHub node: “Commit build.gradle Updates to Branch”**
   - Resource: File
   - Operation: Edit
   - File path: `{{ $json.path }}`
   - Content: `{{ $json.content }}`
   - Commit message: `{{ $json.message }}`
   - Branch: `{{ $('Generate New Branch Name').first().json.branchName }}`

24) **Add GitHub node: “Commit gradle.properties Updates to Branch”**
   - Same config as above, with the second file.

25) **Add Set node: “Prepare Pull Request Data”**
   - Set `branchName = {{ $('Generate New Branch Name').first().json.branchName }}`

26) **Add HTTP Request node: “Create Pull Request”**
   - Method: `POST`
   - URL (example):
     - `https://api.github.com/repos/<owner>/<repo>/pulls`
   - JSON body:
     - title/body/head/base (head = branchName, base = main)
   - Auth: GitHub credential

27) **Add Slack credentials + node**
   - Create Slack credential (OAuth or bot token) with `chat:write`
   - Node: “Notify Team on Slack”
   - Channel: select your channel
   - Message includes PR link (likely `{{ $json.html_url }}` depending on response shape)

28) **Activate workflow**
   - Push a change modifying `.env.staging` to verify end-to-end behavior.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow description and setup steps are embedded as a large sticky note in the canvas. | (Sticky Note7 content) |
| HTTP Request nodes use placeholder URLs (`{{Your_Github_URL_Here}}`) and must be replaced with actual GitHub API endpoints. | Nodes: Get Latest SHA of main Branch, Create New Git Branch, Create Pull Request |
| The Authorization header contains a typo `Bearar` and should be `Bearer` if manually used. Prefer using n8n GitHub credentials instead of raw headers. | Node: Get Latest SHA of main Branch |
| Slack message template likely references a non-existent field `pullRequest.html_url`. Typically GitHub returns `html_url` at the top level. | Node: Notify Team on Slack |

Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.