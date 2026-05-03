Generate your GitLab year-in-review wrapped report automatically

https://n8nworkflows.xyz/workflows/generate-your-gitlab-year-in-review-wrapped-report-automatically-12394


# Generate your GitLab year-in-review wrapped report automatically

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow generates a personalized “GitLab Wrapped” (year-in-review) report for a given GitLab username by forking (or reusing) the upstream project, injecting CI/CD variables, triggering a pipeline, and polling until the pipeline succeeds. The resulting report is published via GitLab Pages.

**Primary use cases**
- Automate building a yearly GitLab activity recap for a user.
- Self-service generation via an n8n Form Trigger.
- Run on GitLab.com or a self-hosted GitLab instance (URL input exists, though some calls are hardcoded to gitlab.com—see edge cases).

**Logical blocks**
1.1 **Form Input & Normalization**  
Collects username/token/URL/year and normalizes them into consistent fields.

1.2 **Security Validation (PAT ownership check)**  
Calls `/user` with the provided PAT and prevents continuing if the token appears to belong to the same username entered (intended to avoid leaking private projects).

1.3 **Fork Discovery/Creation**  
Searches for an existing fork owned by the PAT user and attempts to fork if needed; merges paths.

1.4 **CI/CD Variable Configuration**  
Sets GitLab CI/CD project variables used by the fork’s pipeline to generate the wrapped.

1.5 **Pipeline Execution & Monitoring**  
Triggers pipeline on `main`, polls every 2 minutes until status is `success`, then ends with a “wrapped available” message.

---

## 2. Block-by-Block Analysis

### 1.1 Form Input & Normalization

**Overview:** Receives user inputs via an n8n form and maps them into canonical fields used by later nodes.

**Nodes involved**
- GitLab Wrapped Form
- Workflow Configuration

#### Node: **GitLab Wrapped Form**
- **Type / role:** `Form Trigger` — entry point; collects user parameters.
- **Key configuration (interpreted):**
  - Form title: “GitLab Wrapped 2024”
  - Description: “Generate your personalized GitLab Wrapped for 2024”
  - Fields:
    - `user` (required)
    - `token` (required)
    - `url` (default `https://gitlab.com`)
    - `Year` (number, default `2025`)
  - Attribution disabled (`appendAttribution: false`)
- **Inputs/Outputs:** No inputs (trigger). Outputs one item containing submitted fields.
- **Version notes:** typeVersion `2.3` (Form Trigger behavior and field schema depend on n8n versions supporting forms).
- **Potential failures / edge cases:**
  - Users may enter a self-hosted GitLab URL, but later nodes sometimes call `https://gitlab.com` directly (inconsistent).
  - “Year” is collected but later overwritten/ignored in places (see CI/CD variable block).

#### Node: **Workflow Configuration**
- **Type / role:** `Set` — normalizes and renames fields for downstream use.
- **Key configuration:**
  - Keeps all original fields (`includeOtherFields: true`)
  - Creates:
    - `name` = `{{$json.user}}`
    - `apiToken` = `{{$json.token}}`
    - `gitlabUrl` = `{{$json.url}}`
    - `year` = `"2024"` (hardcoded)
- **Inputs/Outputs:** Input from Form Trigger; output feeds into PAT validation.
- **Version notes:** typeVersion `3.4`.
- **Potential failures / edge cases:**
  - Year is hardcoded to `2024` even if the form “Year” is different.
  - Later nodes reference both `apiToken` and `token` inconsistently.

---

### 1.2 Security Validation (PAT ownership check)

**Overview:** Validates the PAT by querying the authenticated user. Intended behavior is to stop if the PAT belongs to the same username entered (to avoid accessing private projects), but the IF logic is contradictory/misconfigured.

**Nodes involved**
- Validate PAT Owner
- Is PAT Owner invalid?
- No Operation, to not leak private Projects

#### Node: **Validate PAT Owner**
- **Type / role:** `HTTP Request` — calls GitLab API to identify the PAT owner.
- **Key configuration:**
  - `GET {{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/user/`
  - Header: `PRIVATE-TOKEN: {{$('Workflow Configuration').first().json.apiToken}}`
  - `onError: continueRegularOutput` (continues even if request fails)
- **Inputs/Outputs:** Input from Workflow Configuration; output to IF node.
- **Version notes:** typeVersion `4.3`.
- **Potential failures / edge cases:**
  - Invalid/expired token → GitLab returns 401/403; because of `continueRegularOutput`, downstream may run with error payload or missing `username`.
  - Self-hosted GitLab URL must be correct; otherwise DNS/SSL failures propagate.

#### Node: **Is PAT Owner invalid?**
- **Type / role:** `IF` — decides whether to stop or proceed.
- **Key configuration (interpreted):**
  - Condition 1: `{{$json.username}} equals {{$('Workflow Configuration').item.json.user}}`
  - Condition 2: an additional “equals” condition with **empty leftValue and empty rightValue** (always true)
  - Combinator: `and`
- **Inputs/Outputs:**
  - **True branch** → “No Operation, to not leak private Projects”
  - **False branch** → “Search Existing Fork” and “Try to fork” (both)
- **Version notes:** typeVersion `2.3`.
- **Potential failures / edge cases (important):**
  - The node name suggests it checks “invalid”, but the first condition is “PAT owner == entered user”. That is treated as **True**.  
  - The extra empty شرط is effectively always true, so the IF result depends almost entirely on condition 1.
  - If `username` is missing (API error), condition 1 is false → workflow proceeds, which may be unsafe.
  - The sticky note describes the opposite (“Checks that the PAT does NOT belong…”). The implemented logic appears inverted relative to the description.

#### Node: **No Operation, to not leak private Projects**
- **Type / role:** `NoOp` — terminates the “blocked” path without doing anything.
- **Inputs/Outputs:** Triggered from IF true branch; no further connections.
- **Potential failures / edge cases:** None (acts as a stop).

---

### 1.3 Fork Discovery/Creation

**Overview:** Attempts to find an existing “gitlab-wrapped” project owned by the PAT user; also attempts to fork the upstream project. Both paths are merged to get a project id.

**Nodes involved**
- Search Existing Fork
- Try to fork
- Merge
- set ProjectId

#### Node: **Search Existing Fork**
- **Type / role:** `HTTP Request` — lists owned projects matching search term.
- **Key configuration:**
  - URL: `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/projects?owned=true&search=gitlab-wrapped`
  - Headers: `PRIVATE-TOKEN: {{$('Workflow Configuration').first().json.apiToken}}`
  - `onError: continueRegularOutput`
  - **Note:** Node is configured with a JSON body despite being a GET; typically ignored by GitLab but may be confusing.
- **Outputs:** To Merge (input 2).
- **Potential failures / edge cases:**
  - If multiple matches returned, `.first().json.id` may select an unintended project.
  - If no results, `first()` may be undefined later (breaks `set ProjectId`).

#### Node: **Try to fork**
- **Type / role:** `HTTP Request` — forks upstream repository to the authenticated user.
- **Key configuration:**
  - URL (hardcoded gitlab.com): `https://gitlab.com/api/v4/projects/michaelangelorivera%2Fgitlab-wrapped/fork`
  - Method: POST
  - Header: `PRIVATE-TOKEN: {{$('Workflow Configuration').first().json.token}}`
  - `onError: continueRegularOutput`
- **Outputs:** To Merge (input 1).
- **Potential failures / edge cases:**
  - **Bug:** references `$('Workflow Configuration').first().json.token` but the Set node defines `apiToken` (token may still exist due to `includeOtherFields: true`, but this is inconsistent and fragile).
  - Fork may already exist → GitLab returns error (often 409). With `continueRegularOutput`, workflow continues but must still find the project via search.
  - Hardcoded to gitlab.com; does not respect self-hosted `gitlabUrl`.

#### Node: **Merge**
- **Type / role:** `Merge` — combines results of “Try to fork” and “Search Existing Fork”.
- **Key configuration:** Default merge behavior (not explicitly configured in JSON).
- **Inputs/Outputs:** Receives both branches; outputs to `set ProjectId`.
- **Potential failures / edge cases:**
  - Depending on merge mode defaults, it may wait for both inputs; if one branch errors unexpectedly, could stall or merge partial data (implementation-specific).
  - Output structure depends on merge strategy; downstream assumes “Search Existing Fork” is available.

#### Node: **set ProjectId**
- **Type / role:** `Code` — derives `projectId` for subsequent API calls.
- **Key configuration:**
  - JS code: `return [{projectId: $('Search Existing Fork').first().json.id}]`
- **Inputs/Outputs:** Input from Merge; output to “Set token env var”.
- **Potential failures / edge cases:**
  - If `Search Existing Fork` returned empty array or failed, `first().json.id` throws or yields undefined → later URLs break.
  - Ignores fork response entirely; even if “Try to fork” returns the new project id, it is not used.

---

### 1.4 CI/CD Variable Configuration

**Overview:** Creates GitLab CI/CD variables in the forked project that the pipeline uses: token, username, and year.

**Nodes involved**
- Set token env var
- Set Username env var
- Set Year env var

#### Node: **Set token env var**
- **Type / role:** `HTTP Request` — creates variable `GITLAB_TOKEN`.
- **Key configuration:**
  - POST `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/projects/{{$json.projectId}}/variables`
  - Header: `PRIVATE-TOKEN: {{$('Workflow Configuration').first().json.apiToken}}`
  - JSON body:
    - key: `GITLAB_TOKEN`
    - value: `{{$('Workflow Configuration').first().json.apiToken}}`
    - masked: true
    - protected: true
    - environment_scope: `*`
  - `onError: continueRegularOutput`
- **Inputs/Outputs:** Uses `projectId` from `set ProjectId`; outputs to “Set Username env var”.
- **Potential failures / edge cases:**
  - Variable may already exist → GitLab returns 400 (key taken). With `continueRegularOutput`, the workflow continues but variable might not be updated. Consider PUT to update.
  - `protected: true` might prevent pipeline usage on unprotected branches depending on project settings.

#### Node: **Set Username env var**
- **Type / role:** `HTTP Request` — creates variable `GITLAB_USERNAME`.
- **Key configuration:**
  - POST `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/projects/{{ $('set ProjectId').item.json.projectId }}/variables`
  - Body: key `GITLAB_USERNAME`, value `{{$('Workflow Configuration').first().json.user}}`, masked false, protected false
  - Header uses `apiToken`
  - `onError: continueRegularOutput`
- **Inputs/Outputs:** From previous variable node; outputs to “Set Year env var”.
- **Potential failures / edge cases:** Same “already exists” problem; value updates are not guaranteed.

#### Node: **Set Year env var**
- **Type / role:** `HTTP Request` — creates variable `GITLAB_YEAR`.
- **Key configuration:**
  - POST `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/projects/{{ $('set ProjectId').item.json.projectId }}/variables`
  - Body: key `GITLAB_YEAR`, value `"2025"` (hardcoded here), masked false, protected false
  - Header uses `apiToken`
  - `onError: continueRegularOutput`
- **Inputs/Outputs:** Outputs to “Trigger Pipeline”.
- **Potential failures / edge cases:**
  - **Year mismatch:** workflow configuration sets `year` to 2024; this node sets `GITLAB_YEAR` to 2025; the form default is 2025. The effective year is whatever this node sends (2025).
  - If variable exists, won’t update.

---

### 1.5 Pipeline Execution & Monitoring

**Overview:** Triggers the pipeline for the fork on `main`, then polls the latest pipeline status every 2 minutes until it reports `success`.

**Nodes involved**
- Trigger Pipeline
- Wait for CI/CD
- Check Pipeline Status
- Pipeline Complete?
- Wrapped available at https://YOUR_USERNAME.gitlab.io/gitlab-wrapped

#### Node: **Trigger Pipeline**
- **Type / role:** `HTTP Request` — triggers a pipeline on the project’s `main` branch.
- **Key configuration:**
  - POST `https://gitlab.com/api/v4/projects/{{ $('set ProjectId').item.json.projectId }}/pipeline?ref=main`
  - Header: `PRIVATE-TOKEN: {{$('Workflow Configuration').first().json.apiToken}}`
  - Body: empty JSON
- **Inputs/Outputs:** From Set Year env var; output to Wait.
- **Potential failures / edge cases:**
  - Hardcoded to gitlab.com (ignores `gitlabUrl`).
  - If default branch is not `main`, trigger fails.
  - If pipeline permissions restricted, 403.

#### Node: **Wait for CI/CD**
- **Type / role:** `Wait` — polling delay.
- **Key configuration:** Wait 2 minutes.
- **Inputs/Outputs:** From Trigger Pipeline and from Pipeline Complete? (false branch); outputs to Check Pipeline Status.
- **Potential failures / edge cases:** Slows execution; may hit workflow execution time limits depending on n8n hosting constraints.

#### Node: **Check Pipeline Status**
- **Type / role:** `HTTP Request` — fetches latest pipeline for `main`.
- **Key configuration:**
  - GET `https://gitlab.com/api/v4/projects/{{ $('set ProjectId').item.json.projectId }}/pipelines?ref=main&per_page=1`
  - Header: `PRIVATE-TOKEN: {{$('Workflow Configuration').first().json.apiToken}}`
- **Outputs:** To IF node “Pipeline Complete?”
- **Potential failures / edge cases:**
  - GitLab returns an array of pipelines; the workflow checks `$json.status` directly, which may not exist unless n8n auto-splits items or returns first element as item. If it remains an array, `$json.status` is undefined and the IF never succeeds.
  - Hardcoded gitlab.com again.

#### Node: **Pipeline Complete?**
- **Type / role:** `IF` — loops until pipeline succeeds.
- **Key configuration:**
  - Condition: `{{$json.status}} equals "success"`
  - True → finish node
  - False → back to Wait (loop)
- **Potential failures / edge cases:**
  - If pipeline fails (`failed`, `canceled`) this loops forever (no max retries / no failure exit).
  - If status field missing (see previous node), loops forever.

#### Node: **Wrapped available at https://YOUR_USERNAME.gitlab.io/gitlab-wrapped**
- **Type / role:** `NoOp` — end marker indicating where output will be available.
- **Inputs:** From IF true branch.
- **Outputs:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| GitLab Wrapped Form | formTrigger | Collect user inputs (username/token/url/year) | — | Workflow Configuration | ## 📋 Form Input\nCollects user credentials and preferences to generate the wrapped report. |
| Workflow Configuration | set | Normalize fields (`apiToken`, `gitlabUrl`, etc.) | GitLab Wrapped Form | Validate PAT Owner | ## 📋 Form Input\nCollects user credentials and preferences to generate the wrapped report. |
| Validate PAT Owner | httpRequest | Validate PAT and fetch token owner | Workflow Configuration | Is PAT Owner invalid? | ## 🔐 Security Validation\nEnsures the PAT token does not belong to the specified user to prevent unauthorized access to private projects. |
| Is PAT Owner invalid? | if | Gate execution based on PAT owner vs entered username | Validate PAT Owner | No Operation, to not leak private Projects; Search Existing Fork; Try to fork | ## 🔐 Security Validation\nEnsures the PAT token does not belong to the specified user to prevent unauthorized access to private projects. |
| No Operation, to not leak private Projects | noOp | Stop path to avoid leakage | Is PAT Owner invalid? (true) | — | ## 🔐 Security Validation\nEnsures the PAT token does not belong to the specified user to prevent unauthorized access to private projects. |
| Try to fork | httpRequest | Fork upstream gitlab-wrapped project | Is PAT Owner invalid? (false) | Merge | ## 🍴 Fork Setup\nCreates or finds your fork of the gitlab-wrapped project to run the generation pipeline. |
| Search Existing Fork | httpRequest | Search owned projects for existing fork | Is PAT Owner invalid? (false) | Merge | ## 🍴 Fork Setup\nCreates or finds your fork of the gitlab-wrapped project to run the generation pipeline. |
| Merge | merge | Combine fork/search paths | Try to fork; Search Existing Fork | set ProjectId | ## 🍴 Fork Setup\nCreates or finds your fork of the gitlab-wrapped project to run the generation pipeline. |
| set ProjectId | code | Extract `projectId` (from Search result) | Merge | Set token env var | ## 🍴 Fork Setup\nCreates or finds your fork of the gitlab-wrapped project to run the generation pipeline. |
| Set token env var | httpRequest | Create CI/CD variable `GITLAB_TOKEN` | set ProjectId | Set Username env var | ## ⚙️ CI/CD Configuration\nConfigures environment variables in your fork to pass settings to the pipeline. |
| Set Username env var | httpRequest | Create CI/CD variable `GITLAB_USERNAME` | Set token env var | Set Year env var | ## ⚙️ CI/CD Configuration\nConfigures environment variables in your fork to pass settings to the pipeline. |
| Set Year env var | httpRequest | Create CI/CD variable `GITLAB_YEAR` | Set Username env var | Trigger Pipeline | ## ⚙️ CI/CD Configuration\nConfigures environment variables in your fork to pass settings to the pipeline. |
| Trigger Pipeline | httpRequest | Trigger pipeline on `main` | Set Year env var | Wait for CI/CD | ## 🚀 Pipeline Execution\nTriggers the wrapped generation pipeline and monitors it until completion (polls every 2 minutes). |
| Wait for CI/CD | wait | Sleep 2 minutes between polls | Trigger Pipeline; Pipeline Complete? (false) | Check Pipeline Status | ## 🚀 Pipeline Execution\nTriggers the wrapped generation pipeline and monitors it until completion (polls every 2 minutes). |
| Check Pipeline Status | httpRequest | Fetch latest pipeline status | Wait for CI/CD | Pipeline Complete? | ## 🚀 Pipeline Execution\nTriggers the wrapped generation pipeline and monitors it until completion (polls every 2 minutes). |
| Pipeline Complete? | if | Loop until pipeline status is `success` | Check Pipeline Status | Wrapped available…; Wait for CI/CD | ## 🚀 Pipeline Execution\nTriggers the wrapped generation pipeline and monitors it until completion (polls every 2 minutes). |
| Wrapped available at https://YOUR_USERNAME.gitlab.io/gitlab-wrapped | noOp | End marker / output location | Pipeline Complete? (true) | — | ## 🚀 Pipeline Execution\nTriggers the wrapped generation pipeline and monitors it until completion (polls every 2 minutes). |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n named **“GitLab Wrapped Workflow”**.

2) **Add “Form Trigger” node** named **“GitLab Wrapped Form”**  
   - Form title: `GitLab Wrapped 2024`  
   - Description: `Generate your personalized GitLab Wrapped for 2024`  
   - Fields:
     - `user` (required, text)
     - `token` (required, text/password)
     - `url` (text, default `https://gitlab.com`)
     - `Year` (number, default `2025`)
   - Disable attribution if desired.

3) **Add “Set” node** named **“Workflow Configuration”**, connect from the form trigger.  
   - Enable “Keep Only Set” = **false** (i.e., include other fields).  
   - Add fields:
     - `name` (string) = `{{$json.user}}`
     - `apiToken` (string) = `{{$json.token}}`
     - `gitlabUrl` (string) = `{{$json.url}}`
     - `year` (string) = `2024` (to match workflow; consider switching to `{{$json.Year}}` if you want it dynamic)

4) **Add “HTTP Request” node** named **“Validate PAT Owner”**, connect from Workflow Configuration.  
   - Method: GET  
   - URL: `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/user/`  
   - Send headers: on  
   - Header: `PRIVATE-TOKEN` = `{{$('Workflow Configuration').first().json.apiToken}}`  
   - Error handling: set **On Error = Continue (continueRegularOutput)**.

5) **Add “IF” node** named **“Is PAT Owner invalid?”**, connect from Validate PAT Owner.  
   - Condition (String equals):  
     - Left: `{{$json.username}}`  
     - Right: `{{$('Workflow Configuration').item.json.user}}`  
   - (To mirror the provided workflow exactly, add the second empty equals condition; practically, remove it.)
   - True output → add a **NoOp** node named **“No Operation, to not leak private Projects”** and connect it.

6) From the **False** output of the IF node, create **two parallel HTTP Request nodes**:

   6.1) **“Search Existing Fork”**  
   - Method: GET  
   - URL: `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/projects?owned=true&search=gitlab-wrapped`  
   - Header: `PRIVATE-TOKEN` = `{{$('Workflow Configuration').first().json.apiToken}}`  
   - On Error: Continue.

   6.2) **“Try to fork”**  
   - Method: POST  
   - URL: `https://gitlab.com/api/v4/projects/michaelangelorivera%2Fgitlab-wrapped/fork`  
   - Header: `PRIVATE-TOKEN` = `{{$('Workflow Configuration').first().json.token}}` (as in workflow; more robust: use `apiToken`)  
   - On Error: Continue.

7) **Add “Merge” node** named **“Merge”**  
   - Connect “Try to fork” → Merge input 1  
   - Connect “Search Existing Fork” → Merge input 2  
   - Keep default merge settings (to match the workflow).

8) **Add “Code” node** named **“set ProjectId”**, connect from Merge.  
   - Code:
     ```javascript
     return [{ projectId: $('Search Existing Fork').first().json.id }];
     ```
   - (This matches the workflow; more robust would fall back to fork response if search is empty.)

9) **Add HTTP Request node** “Set token env var”, connect from set ProjectId.  
   - Method: POST  
   - URL: `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/projects/{{$json.projectId}}/variables`  
   - JSON body:
     - key: `GITLAB_TOKEN`
     - value: `{{$('Workflow Configuration').first().json.apiToken}}`
     - masked: `true`
     - protected: `true`
     - environment_scope: `*`
   - Header: `PRIVATE-TOKEN` = `{{$('Workflow Configuration').first().json.apiToken}}`  
   - On Error: Continue.

10) **Add HTTP Request node** “Set Username env var”, connect from Set token env var.  
   - Method: POST  
   - URL: `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/projects/{{$('set ProjectId').item.json.projectId}}/variables`  
   - JSON body: key `GITLAB_USERNAME`, value `{{$('Workflow Configuration').first().json.user}}`, masked false, protected false, environment_scope `*`  
   - Header uses `apiToken`  
   - On Error: Continue.

11) **Add HTTP Request node** “Set Year env var”, connect from Set Username env var.  
   - Method: POST  
   - URL: `{{$('Workflow Configuration').first().json.gitlabUrl}}/api/v4/projects/{{$('set ProjectId').item.json.projectId}}/variables`  
   - JSON body: key `GITLAB_YEAR`, value `2025` (hardcoded in provided workflow; you can replace with `{{$('Workflow Configuration').first().json.year}}` or the form year)  
   - Header uses `apiToken`  
   - On Error: Continue.

12) **Add HTTP Request node** “Trigger Pipeline”, connect from Set Year env var.  
   - Method: POST  
   - URL: `https://gitlab.com/api/v4/projects/{{$('set ProjectId').item.json.projectId}}/pipeline?ref=main`  
   - Header: `PRIVATE-TOKEN` = `{{$('Workflow Configuration').first().json.apiToken}}`  
   - Body: `{}` (empty JSON)

13) **Add “Wait” node** “Wait for CI/CD”, connect from Trigger Pipeline.  
   - Wait: 2 minutes.

14) **Add HTTP Request node** “Check Pipeline Status”, connect from Wait.  
   - Method: GET  
   - URL: `https://gitlab.com/api/v4/projects/{{$('set ProjectId').item.json.projectId}}/pipelines?ref=main&per_page=1`  
   - Header: `PRIVATE-TOKEN` = `{{$('Workflow Configuration').first().json.apiToken}}`

15) **Add IF node** “Pipeline Complete?”, connect from Check Pipeline Status.  
   - Condition: `{{$json.status}} equals success`  
   - True → connect to a **NoOp** node named **“Wrapped available at https://YOUR_USERNAME.gitlab.io/gitlab-wrapped”**  
   - False → connect back to **Wait for CI/CD** (loop)

**Credentials:** This workflow uses GitLab PATs passed via the form and inserted as HTTP header `PRIVATE-TOKEN`. No n8n credential object is required, but you should ensure:
- The PAT has scopes: `api`, `read_repository`, `write_repository` (per sticky note).
- Your n8n instance is secured, since users submit tokens via a form.

**Sub-workflows:** None (no Execute Workflow node present).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Powered by `https://gitlab.com/michaelangelorivera/gitlab-wrapped` by `https://gitlab.com/michaelangelorivera` | From sticky note (project credit and source) |
| Wrapped output location: `https://YOUR-USERNAME.gitlab.io/gitlab-wrapped` | From sticky note and final NoOp node |
| Setup: Create a GitLab PAT with `api`, `read_repository`, `write_repository` scopes | From sticky note “Setup steps” |
| Workflow intent: stops if PAT belongs to the entered username to prevent unauthorized access to private projects (as described) | Sticky note describes this; implementation may be inverted/misaligned with the description |

