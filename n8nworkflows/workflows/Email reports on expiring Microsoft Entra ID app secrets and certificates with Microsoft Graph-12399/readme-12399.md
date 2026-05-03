Email reports on expiring Microsoft Entra ID app secrets and certificates with Microsoft Graph

https://n8nworkflows.xyz/workflows/email-reports-on-expiring-microsoft-entra-id-app-secrets-and-certificates-with-microsoft-graph-12399


# Email reports on expiring Microsoft Entra ID app secrets and certificates with Microsoft Graph

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Detect Microsoft Entra ID (Azure AD) application **client secrets** and **certificates** nearing expiration (within a configurable number of days), then send a **single HTML email report** listing them. If nothing is expiring, the workflow ends without sending any email.

**Typical use cases:**
- Prevent outages caused by expired app credentials.
- Scheduled compliance/operations reporting (daily/weekly).
- Centralized notification to a team mailbox.

### 1.1 Logical Blocks
1. **Trigger & Configuration**
   - Manual start (can be replaced with schedule).
   - Set the notification target and expiry window.
2. **Graph Data Retrieval**
   - Pull Entra ID applications with `passwordCredentials` and `keyCredentials` via Microsoft Graph.
3. **Credential Expansion & Filtering**
   - Split applications into items; split secrets/certs into individual credential items.
   - Filter credentials expiring within the next **N days**.
4. **Normalization & Merge**
   - Normalize secret/cert entries into a consistent schema.
   - Merge both credential types into one list.
5. **Report & Notification**
   - Aggregate into an array, check if non-empty.
   - Render HTML table and email it; otherwise do nothing.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & Variables
**Overview:** Starts the workflow and defines runtime parameters used across filtering and notification steps.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set Variables

#### Node: When clicking ‘Execute workflow’
- **Type / role:** `Manual Trigger` — entry point for manual execution.
- **Configuration:** No parameters.
- **Connections:** Outputs to **Set Variables**.
- **Edge cases:** None. For automation, replace with Schedule Trigger.

#### Node: Set Variables
- **Type / role:** `Set` — defines configuration constants.
- **Key fields set:**
  - `notificationEmail` (string): default `user@example.com`
  - `daysBeforeExpiry` (number): default `14`
- **Expressions/variables used elsewhere:**
  - `$('Set Variables').first().json.daysBeforeExpiry` (used in date filters)
  - `$('Set Variables').item.json.notificationEmail` (used in email “to”)
- **Connections:** Outputs to **Get EntraID Applications and Secrets**.
- **Edge cases / failures:**
  - If `daysBeforeExpiry` is not a valid number, date expressions can behave unexpectedly.
  - If `notificationEmail` is empty/invalid, email node will fail.

---

### Block 2.2 — Fetch Entra ID Applications (Microsoft Graph)
**Overview:** Queries Microsoft Graph for all Entra ID application registrations including credential metadata (but not secret values).

**Nodes involved:**
- Get EntraID Applications and Secrets
- Split Out Applications

#### Node: Get EntraID Applications and Secrets
- **Type / role:** `HTTP Request` — calls Microsoft Graph.
- **Authentication:** OAuth2 via **genericCredentialType** (`oAuth2Api`).
  - Credential referenced: **Microsoft Graph (n8n)** (placeholder id `credential-id`).
  - Recommended flow (per notes): **Client Credentials**.
- **Request:**
  - **GET** `https://graph.microsoft.com/v1.0/applications?$select=id,appId,displayName,passwordCredentials,keyCredentials`
- **Output expectation:**
  - Graph returns an object with a `value` array of applications.
- **Connections:** Outputs to **Split Out Applications**.
- **Edge cases / failures:**
  - **401/403** if permissions missing (requires at least `Application.Read.All` as Application permission).
  - **Pagination risk:** Graph may paginate results with `@odata.nextLink`. This node as configured does not show pagination handling; large tenants may return only the first page unless n8n options are configured to follow next links.
  - Network timeouts / throttling (429).

#### Node: Split Out Applications
- **Type / role:** `Split Out` — turns the `value` array into individual items (one per application).
- **Configuration:** `fieldToSplitOut = "value"`.
- **Connections:** Outputs in parallel to:
  - **Split Out Client Secrets**
  - **Split Out Certificates**
- **Edge cases:**
  - If Graph response has no `value` or it’s not an array, node will output nothing or error.

---

### Block 2.3 — Extract & Filter Credentials (Secrets and Certificates)
**Overview:** For each application item, split the credential arrays into individual entries and filter those expiring within the next **N** days.

**Nodes involved:**
- Split Out Client Secrets
- Filter Client Secrets
- Split Out Certificates
- Filter Client Certificates

#### Node: Split Out Client Secrets
- **Type / role:** `Split Out` — expands `passwordCredentials` into one item per secret.
- **Configuration:** `fieldToSplitOut = "passwordCredentials"`.
- **Note (from node):** “Password Credentials are Client secrets in the EntraID UI”
- **Connections:** Outputs to **Filter Client Secrets**.
- **Edge cases:**
  - Apps without secrets: `passwordCredentials` may be empty/undefined → no output items.
  - Credential entries missing `endDateTime` can break downstream filter.

#### Node: Filter Client Secrets
- **Type / role:** `Filter` — keep only secrets expiring within the configured window.
- **Condition logic:** `endDateTime <= now + daysBeforeExpiry`
  - **Left:** `={{ $json.endDateTime }}`
  - **Right:** `={{ new Date(Date.now() + $('Set Variables').first().json.daysBeforeExpiry * 24 * 60 * 60 * 1000) }}`
  - Operator: `dateTime` “beforeOrEquals”
- **Connections:** Outputs to **Build Client Secrets Report**.
- **Edge cases / failures:**
  - If `endDateTime` is null/invalid, date comparison may fail.
  - Timezone nuance: Graph returns ISO timestamps; comparison uses JS `Date` parsing rules.

#### Node: Split Out Certificates
- **Type / role:** `Split Out` — expands `keyCredentials` into one item per certificate.
- **Configuration:** `fieldToSplitOut = "keyCredentials"`.
- **Note (from node):** “Key Credentials are Certificates in the EntraID UI”
- **Connections:** Outputs to **Filter Client Certificates**.
- **Edge cases:** Same pattern as secrets; empty/undefined `keyCredentials` yields no items.

#### Node: Filter Client Certificates
- **Type / role:** `Filter` — keep only certificates expiring within the configured window.
- **Condition logic:** identical to secrets:
  - `$json.endDateTime <= now + daysBeforeExpiry`
- **Connections:** Outputs to **Build Certificates Report**.
- **Edge cases:** Same as secrets.

---

### Block 2.4 — Normalize Records & Merge
**Overview:** Converts filtered secrets/certs into a consistent reporting schema and merges them into a single stream.

**Nodes involved:**
- Build Client Secrets Report
- Build Certificates Report
- Merge

#### Node: Build Client Secrets Report
- **Type / role:** `Set` — maps secret items to a normalized structure.
- **Key mappings:**
  - `applicationName` = `$('Split Out Applications').item.json.displayName`
  - `appId` = `$('Split Out Applications').item.json.appId`
  - `type` = `"Client Secret"`
  - `clientSecretName` = `$json.displayName`
  - `clientSecretId` = `$json.keyId`
  - `expiresInDays` = `Math.max(0, floor((endDateTime - now)/days))`
- **Connections:** Outputs to **Merge** (input index 0).
- **Important implementation detail:** Uses `$('Split Out Applications').item...` to reference the parent application while currently iterating credentials. This depends on n8n’s item linking behavior and may be sensitive to changes in execution/item pairing.
- **Edge cases:**
  - Missing `displayName`, `keyId`, or `endDateTime` leads to blanks/NaN.
  - `expiresInDays` floors and clamps to `>= 0`; already-expired credentials show as 0 days.

#### Node: Build Certificates Report
- **Type / role:** `Set` — maps certificate items to normalized structure.
- **Key mappings:**
  - `applicationName`, `appId` (same method as above)
  - `type` = `"Certificate"`
  - `certificateName` = `$json.displayName`
  - `certificateId` = `$json.keyId`
  - `expiresInDays` = same calculation
- **Connections:** Outputs to **Merge** (input index 1).
- **Edge cases:** Same as secrets mapping.

#### Node: Merge
- **Type / role:** `Merge` — combines the two streams (secrets + certificates).
- **Configuration:** Default merge behavior (as set, no explicit mode shown). In this design, it effectively unions both inputs into one output stream for aggregation.
- **Connections:** Outputs to **Aggregate**.
- **Edge cases / failures:**
  - If one branch produces no items, merge behavior depends on node defaults; typically it can still pass through the other input, but verify in your n8n version.
  - Misconfigured merge mode could block output waiting for paired items.

---

### Block 2.5 — Aggregate, Decide, Render HTML, Send Email
**Overview:** Collects merged items into an array, checks if it’s non-empty, generates an HTML table, and emails it.

**Nodes involved:**
- Aggregate
- If Expiring Secrets not empty
- HTML Table with Expiring Secrets
- Send email
- No Operation, do nothing

#### Node: Aggregate
- **Type / role:** `Aggregate` — gathers all item data into a single field.
- **Configuration:**
  - Mode: `aggregateAllItemData`
  - Destination field: `expiringSecrets`
- **Output:** A single item with `expiringSecrets: [ ...normalized entries... ]`.
- **Connections:** Outputs to **If Expiring Secrets not empty**.
- **Edge cases:**
  - If merge outputs zero items, aggregate may output an empty array or potentially no item (version-dependent). This workflow assumes `$json.expiringSecrets` exists for the IF node.

#### Node: If Expiring Secrets not empty
- **Type / role:** `If` — controls whether to send an email.
- **Condition:** Array `notEmpty`
  - Left: `={{ $json.expiringSecrets }}`
- **Connections:**
  - **True** → HTML Table with Expiring Secrets
  - **False** → No Operation, do nothing
- **Edge cases:**
  - If `expiringSecrets` is undefined (unexpected aggregate output), condition evaluation may fail.

#### Node: HTML Table with Expiring Secrets
- **Type / role:** `HTML` — renders the report body.
- **Behavior:**
  - Uses `($json.expiringSecrets || [])`
  - Sorts by `applicationName` (case-insensitive-ish via `sensitivity: "base"`)
  - Builds rows with alternating background
  - Highlights `expiresInDays`:
    - `<= 30` days: red/bold
    - `<= 90` days: orange/bold
- **Output:** `$json.html` containing the generated HTML.
- **Connections:** Outputs to **Send email**.
- **Edge cases:**
  - If any field is missing, the template uses nullish coalescing to avoid crashes.
  - If `expiresInDays` is non-numeric, styling may not apply correctly.

#### Node: Send email
- **Type / role:** `Email Send` — sends the HTML report.
- **Key configuration:**
  - `toEmail` = `={{ $('Set Variables').item.json.notificationEmail }}`
  - `subject` = `Applications with expiring EntraID secrets`
  - `html` = `={{ $json.html }}`
- **Credentials:** Not included in JSON; must be configured in n8n (SMTP or provider-backed email node settings depending on your instance).
- **Connections:** End of success path.
- **Edge cases / failures:**
  - Missing/invalid email credentials or SMTP connectivity issues.
  - Recipient policy restrictions, spam filtering, HTML stripping in some clients.

#### Node: No Operation, do nothing
- **Type / role:** `NoOp` — explicit end when no expiring credentials found.
- **Connections:** End of “false” path.
- **Edge cases:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set Variables | ## How it works / ## Setup steps (permissions, credentials, adjust variables, configure email) |
| Set Variables | Set | Configure `notificationEmail`, `daysBeforeExpiry` | When clicking ‘Execute workflow’ | Get EntraID Applications and Secrets | ## Variables |
| Get EntraID Applications and Secrets | HTTP Request | Query Graph applications + credentials metadata | Set Variables | Split Out Applications | ## Get EntraID Applications |
| Split Out Applications | Split Out | Expand Graph `value` array into per-app items | Get EntraID Applications and Secrets | Split Out Client Secrets; Split Out Certificates | ## Filter expiring secrets |
| Split Out Client Secrets | Split Out | Expand `passwordCredentials` per app | Split Out Applications | Filter Client Secrets | ## Filter expiring secrets |
| Filter Client Secrets | Filter | Keep secrets expiring within N days | Split Out Client Secrets | Build Client Secrets Report | ## Filter expiring secrets |
| Build Client Secrets Report | Set | Normalize secret record for reporting | Filter Client Secrets | Merge | ## Filter expiring secrets |
| Split Out Certificates | Split Out | Expand `keyCredentials` per app | Split Out Applications | Filter Client Certificates | ## Filter expiring secrets |
| Filter Client Certificates | Filter | Keep certificates expiring within N days | Split Out Certificates | Build Certificates Report | ## Filter expiring secrets |
| Build Certificates Report | Set | Normalize certificate record for reporting | Filter Client Certificates | Merge | ## Filter expiring secrets |
| Merge | Merge | Combine normalized secrets + certs streams | Build Client Secrets Report; Build Certificates Report | Aggregate | ## Report / Build the report as an HTML table and send it via mail |
| Aggregate | Aggregate | Collect all items into `expiringSecrets` array | Merge | If Expiring Secrets not empty | ## Report / Build the report as an HTML table and send it via mail |
| If Expiring Secrets not empty | If | Only proceed if results exist | Aggregate | HTML Table with Expiring Secrets (true); No Operation, do nothing (false) | ## Report / Build the report as an HTML table and send it via mail |
| HTML Table with Expiring Secrets | HTML | Render sorted HTML table report | If Expiring Secrets not empty (true) | Send email | ## Report / Build the report as an HTML table and send it via mail |
| Send email | Email Send | Send report email | HTML Table with Expiring Secrets | — | ## Report / Build the report as an HTML table and send it via mail |
| No Operation, do nothing | NoOp | End workflow when no expiring items | If Expiring Secrets not empty (false) | — | ## Report / Build the report as an HTML table and send it via mail |
| Sticky Note1 | Sticky Note | Documentation (how it works + setup steps) | — | — | ## How it works … / ## Setup steps … |
| Sticky Note | Sticky Note | Documentation label for filtering section | — | — | ## Filter expiring secrets |
| Sticky Note2 | Sticky Note | Documentation label for report section | — | — | ## Report / Build the report as an HTML table and send it via mail |
| Sticky Note3 | Sticky Note | Documentation label for variables | — | — | ## Variables |
| Sticky Note4 | Sticky Note | Documentation label for Graph fetch | — | — | ## Get EntraID Applications |
| Sticky Note6 | Sticky Note | Notes (schedule, client credentials, contact) | — | — | ### Notes - Requires EntraID Application - Use Client Credentials … - Use a schedule trigger … - Need help? ✉️ **office@sus-tech.at** |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (inactive by default).

2. **Add Trigger**
   1. Add node: **Manual Trigger**
   2. Name it: `When clicking ‘Execute workflow’`

3. **Add configuration variables**
   1. Add node: **Set** (v3+)
   2. Name: `Set Variables`
   3. Add fields:
      - `notificationEmail` (String) → e.g. `security-ops@example.com`
      - `daysBeforeExpiry` (Number) → e.g. `14`
   4. Connect: Manual Trigger → Set Variables

4. **Create Microsoft Graph OAuth2 credentials (Client Credentials recommended)**
   1. In Entra ID: create an **App Registration**
   2. Grant **Application permission**: `Application.Read.All`
   3. Admin-consent the permission
   4. In n8n Credentials:
      - Create **OAuth2 API** credentials for Microsoft Graph
      - Use tenant/client id/secret and token URL appropriate for Entra ID
      - Ensure scopes/permissions align with application permissions usage in your setup

5. **Add Graph HTTP Request**
   1. Add node: **HTTP Request**
   2. Name: `Get EntraID Applications and Secrets`
   3. Set:
      - Method: **GET**
      - URL: `https://graph.microsoft.com/v1.0/applications?$select=id,appId,displayName,passwordCredentials,keyCredentials`
      - Authentication: **OAuth2** (generic `oAuth2Api`)
      - Select your Microsoft Graph OAuth2 credentials
   4. Connect: Set Variables → HTTP Request
   5. (Recommended if many apps) Enable pagination / “follow nextLink” behavior if your n8n HTTP node supports it, or implement a loop based on `@odata.nextLink`.

6. **Split applications**
   1. Add node: **Split Out**
   2. Name: `Split Out Applications`
   3. Field to split out: `value`
   4. Connect: HTTP Request → Split Out Applications

7. **Secrets branch**
   1. Add node: **Split Out**
      - Name: `Split Out Client Secrets`
      - Field to split out: `passwordCredentials`
   2. Add node: **Filter**
      - Name: `Filter Client Secrets`
      - Condition: Date/Time **before or equals**
        - Left: `{{$json.endDateTime}}`
        - Right: `{{ new Date(Date.now() + $('Set Variables').first().json.daysBeforeExpiry * 24 * 60 * 60 * 1000) }}`
   3. Add node: **Set**
      - Name: `Build Client Secrets Report`
      - Set fields:
        - `applicationName` = `{{ $('Split Out Applications').item.json.displayName }}`
        - `appId` = `{{ $('Split Out Applications').item.json.appId }}`
        - `type` = `Client Secret`
        - `clientSecretName` = `{{ $json.displayName }}`
        - `clientSecretId` = `{{ $json.keyId }}`
        - `expiresInDays` = `{{ Math.max(0, Math.floor((Date.parse($json.endDateTime) - Date.now()) / (1000*60*60*24))) }}`
   4. Connect:
      - Split Out Applications → Split Out Client Secrets → Filter Client Secrets → Build Client Secrets Report

8. **Certificates branch**
   1. Add node: **Split Out**
      - Name: `Split Out Certificates`
      - Field to split out: `keyCredentials`
   2. Add node: **Filter**
      - Name: `Filter Client Certificates`
      - Use the same date comparison expression as secrets
   3. Add node: **Set**
      - Name: `Build Certificates Report`
      - Set fields:
        - `applicationName` = `{{ $('Split Out Applications').item.json.displayName }}`
        - `appId` = `{{ $('Split Out Applications').item.json.appId }}`
        - `type` = `Certificate`
        - `certificateName` = `{{ $json.displayName }}`
        - `certificateId` = `{{ $json.keyId }}`
        - `expiresInDays` = same formula as above
   4. Connect:
      - Split Out Applications → Split Out Certificates → Filter Client Certificates → Build Certificates Report

9. **Merge both streams**
   1. Add node: **Merge**
   2. Name: `Merge`
   3. Connect:
      - Build Client Secrets Report → Merge (Input 1)
      - Build Certificates Report → Merge (Input 2)
   4. Ensure the merge mode is appropriate to **combine/append** both streams (verify output when one branch is empty).

10. **Aggregate into an array**
   1. Add node: **Aggregate**
   2. Name: `Aggregate`
   3. Set:
      - Aggregate: **All item data**
      - Destination field: `expiringSecrets`
   4. Connect: Merge → Aggregate

11. **Conditional send**
   1. Add node: **If**
   2. Name: `If Expiring Secrets not empty`
   3. Condition:
      - Type: Array → **notEmpty**
      - Value: `{{ $json.expiringSecrets }}`
   4. Connect: Aggregate → If

12. **Build HTML**
   1. Add node: **HTML**
   2. Name: `HTML Table with Expiring Secrets`
   3. Paste the HTML template that:
      - Iterates `($json.expiringSecrets || [])`
      - Sorts by `applicationName`
      - Generates rows and styling based on `expiresInDays`
   4. Connect: If (true) → HTML

13. **Send email**
   1. Add node: **Email Send**
   2. Name: `Send email`
   3. Configure credentials (SMTP or your n8n email provider integration).
   4. Set:
      - To: `{{ $('Set Variables').item.json.notificationEmail }}`
      - Subject: `Applications with expiring EntraID secrets`
      - HTML: `{{ $json.html }}`
   5. Connect: HTML → Send email

14. **False path**
   1. Add node: **No Operation**
   2. Name: `No Operation, do nothing`
   3. Connect: If (false) → No Operation

15. **(Optional) Automation**
   - Replace Manual Trigger with **Schedule Trigger** (daily/weekly) and keep the rest identical.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Requires EntraID Application permission `Application.Read.All` and Microsoft Graph OAuth2 credentials configured in n8n; assign credential to the Graph HTTP Request node; adjust Set Variables; configure email node | From workflow notes |
| Use Client Credentials when adding OAuth2 credentials in n8n | From workflow notes |
| Use a schedule trigger to automatically run this | From workflow notes |
| Need help? ✉️ **office@sus-tech.at** | From workflow notes |