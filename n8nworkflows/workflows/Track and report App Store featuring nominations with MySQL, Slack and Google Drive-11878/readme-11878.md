Track and report App Store featuring nominations with MySQL, Slack and Google Drive

https://n8nworkflows.xyz/workflows/track-and-report-app-store-featuring-nominations-with-mysql--slack-and-google-drive-11878


# Track and report App Store featuring nominations with MySQL, Slack and Google Drive

## 1. Workflow Overview

**Title:** Track and report App Store featuring nominations with MySQL, Slack and Google Drive

**Purpose:**  
This workflow periodically pulls **apps** and **App Store featuring nominations** from the **Apple App Store Connect API**, stores/updates them in **MySQL**, then generates a **CSV report** and shares it via **Slack** and **Google Drive**.

**Target use cases:**
- Teams tracking App Store featuring submission history (status, dates, related app, in-app event).
- Automated weekly reporting to stakeholders via Slack and Drive.
- Building a historical database to query nominations over time.

### 1.1 Scheduling & Parallel Execution
A weekly schedule triggers two parallel branches:
- Branch A: Generate report (based on existing MySQL data) → export CSV → upload to Slack/Drive
- Branch B: Authenticate to App Store Connect → fetch apps & nominations → upsert into MySQL

### 1.2 App Store Connect Authentication (JWT → Bearer)
Creates a JWT token (Apple ASC requirement) used by HTTP Bearer auth for API calls.

### 1.3 Data Ingestion (Apps + Nominations)
Fetches:
- `/v1/apps` (paginated)
- `/v1/nominations?filter[state]=SUBMITTED&include=inAppEvents,relatedApps`

Splits the response arrays into items and upserts into MySQL tables.

### 1.4 Reporting & Distribution
Runs a SQL query joining nominations + applications, converts the result into a CSV binary, then uploads to:
- Slack (file upload)
- Google Drive (service account upload)

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduling & Entry Point
**Overview:** Runs weekly and starts both data refresh and report generation.  
**Nodes involved:** `Weekly Trigger`

#### Node: Weekly Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Config choices:**
  - Runs every **7 days** at **10:00** (server timezone).
- **Connections:**
  - Output → `JWT Auth`
  - Output → `Generate the report with a query`
- **Edge cases / failures:**
  - Timezone mismatch (n8n instance timezone vs expected business timezone).
  - If the DB is empty on first run, report may be blank (still uploaded).
- **Version notes:** typeVersion **1.2**.

---

### Block 2.2 — App Store Connect API Authentication
**Overview:** Generates a short-lived JWT for App Store Connect API access.  
**Nodes involved:** `JWT Auth`

#### Node: JWT Auth
- **Type / role:** `JWT` — creates a signed JWT for Apple App Store Connect.
- **Config choices (interpreted):**
  - JWT includes header option `kid` (Key ID from Apple).
  - Uses JSON claims via `claimsJson`.
  - Claims include:
    - `iss`: Issuer ID (from App Store Connect API keys)
    - `aud`: `appstoreconnect-v1`
    - `bid`: App Bundle ID (note: usage depends on Apple endpoint; may not be required for all calls)
    - `exp`: now + 10 minutes (computed in seconds)
  - `useJson: true` means output is JSON (token available for subsequent auth usage depending on credential wiring).
- **Credentials:** `jwtAuth Credential` (contains the private key material / signing settings).
- **Connections:**
  - Output → `Fetch submitted nominations from App Store Connect`
  - Output → `Fetch apps from App Store Connect`
- **Key expressions:**
  - `exp`: `{{ Math.floor((new Date().getTime() + 10 * 60 * 1000) / 1000) }}`
- **Edge cases / failures:**
  - Invalid `iss`, `kid`, or private key → Apple returns **401 Unauthorized**.
  - Clock skew can invalidate `exp`.
  - If `bid` is wrong/unneeded, may cause unexpected authorization issues depending on Apple validation.
- **Version notes:** typeVersion **1**.

**Important integration note:** The HTTP nodes use **genericCredentialType = httpBearerAuth**. In practice, you must ensure the bearer token used by that credential is the JWT generated here (dynamic token), or update the HTTP Request nodes to explicitly set `Authorization: Bearer {{$json.token}}` (or equivalent output field). As provided, the workflow implies credentials are configured to supply the token, but the JSON alone cannot confirm dynamic token mapping.

---

### Block 2.3 — Fetch & Store Applications (Apps)
**Overview:** Fetches apps from Apple and upserts each app into MySQL (`applications` table).  
**Nodes involved:** `Fetch apps from App Store Connect`, `Split Out1`, `Store applications in MySQL`

#### Node: Fetch apps from App Store Connect
- **Type / role:** `HTTP Request` — calls Apple ASC API `/v1/apps`.
- **Config choices:**
  - URL: `https://api.appstoreconnect.apple.com/v1/apps`
  - Authentication: Bearer (`genericAuthType: httpBearerAuth`)
  - Pagination:
    - Mode: “response contains next URL”
    - Next URL expression: `{{$response.body.links["next"]}}`
    - `maxRequests: 3` (hard cap)
    - `limitPagesFetched: true`
- **Connections:**
  - Output → `Split Out1`
- **Edge cases / failures:**
  - More than 3 pages of apps → data truncated (only first pages fetched).
  - Apple rate limiting (429) or transient 5xx.
  - Pagination breaks if `links.next` is absent or schema differs.
- **Version notes:** typeVersion **4.2**.

#### Node: Split Out1
- **Type / role:** `Split Out` — converts an array into one item per element.
- **Config choices:**
  - Splits field: `data` (Apple JSON: `{ data: [...] }`)
- **Connections:**
  - Output → `Store applications in MySQL`
- **Edge cases / failures:**
  - If API error response doesn’t contain `data`, node errors.
- **Version notes:** typeVersion **1**.

#### Node: Store applications in MySQL
- **Type / role:** `MySQL` — upsert into `applications`.
- **Config choices:**
  - Operation: `upsert`
  - Table: `applications`
  - Match column: `apple_app_id`
  - Match value: `{{$json.id.toNumber()}}` (casts Apple string ID to number)
  - Values sent:
    - `name` = `{{$json.attributes.name}}`
    - `bundle_id` = `{{$json.attributes.bundleId}}`
- **Connections:** terminal for this branch.
- **Edge cases / failures:**
  - `.toNumber()` can fail if `id` is not numeric or missing.
  - MySQL constraint failures if schema differs (e.g., `apple_app_id` not unique).
  - Character set issues if app names contain unusual characters (table uses `utf8mb4` per sticky note, which is correct).
- **Version notes:** typeVersion **2.5**.

**Related sticky note (table schema):**
- `CREATE TABLE applications (...)` (see Summary Table for full content).

---

### Block 2.4 — Fetch & Store Submitted Nominations
**Overview:** Fetches submitted nominations and upserts each nomination into MySQL (`nominations` table).  
**Nodes involved:** `Fetch submitted nominations from App Store Connect`, `Split Out`, `Store nominations in MySQL`

#### Node: Fetch submitted nominations from App Store Connect
- **Type / role:** `HTTP Request` — calls Apple ASC API `/v1/nominations`.
- **Config choices:**
  - URL: `https://api.appstoreconnect.apple.com/v1/nominations`
  - Authentication: Bearer (`httpBearerAuth`)
  - Query parameters:
    - `filter[state] = SUBMITTED`
    - `include = inAppEvents,relatedApps`
- **Connections:**
  - Output → `Split Out`
- **Edge cases / failures:**
  - If `include` relationships are empty, later expressions referencing `[0]` will fail.
  - Apple may paginate nominations; **no pagination configured** here (potentially incomplete dataset if many nominations).
- **Version notes:** typeVersion **4.2**.

#### Node: Split Out
- **Type / role:** `Split Out` — one nomination per item.
- **Config choices:**
  - Field: `data`
- **Connections:**
  - Output → `Store nominations in MySQL`
- **Edge cases / failures:**
  - Missing `data` on error responses.
- **Version notes:** typeVersion **1**.

#### Node: Store nominations in MySQL
- **Type / role:** `MySQL` — upsert into `nominations`.
- **Config choices:**
  - Operation: `upsert`
  - Table: `nominations`
  - Match column: `nomination_id`
  - Match value: `{{$json.id}}`
  - Values sent (mapping Apple nomination fields → DB columns):
    - `type` = `{{$json.type}}`
    - `name` = `{{$json.attributes.name}}`
    - `sub_type` = `{{$json.attributes.type}}`
    - `description` = `{{$json.attributes.description}}`
    - `created_at` / `last_modified_at` / `submitted_at` / `publish_start_at` / `publish_end_at`:
      - Each uses `$if(...)` + `DateTime.fromISO(..., { zone: 'utc' }).toFormat('yyyy-MM-dd HH:mm:ss')`
    - `state` = `{{$json.attributes.state}}`
    - `notes` = `{{$json.attributes.notes}}`
    - `link` = `{{$json.links.self}}`
    - `app_id` = `{{$json.relationships.relatedApps.data[0].id.toNumber()}}`
    - `in_app_event` = `{{$json.relationships.inAppEvents.data[0].id}}`
- **Connections:** terminal for this branch.
- **Key expressions / variables:**
  - Heavy use of Luxon `DateTime.fromISO` (available in n8n expressions).
  - `$if(condition, valueIfTrue, valueIfFalse)` to avoid parsing null timestamps.
- **Edge cases / failures:**
  - If `relationships.relatedApps.data` is empty or undefined → `[0]` access throws.
  - If `relationships.inAppEvents.data` is empty → `[0]` throws.
  - `.toNumber()` can fail if `id` missing/non-numeric.
  - MySQL column type mismatch (`in_app_event` is `bigint` in sticky note; node sends string IDs unless converted).
- **Version notes:** typeVersion **2.5**.

**Related sticky note (table schema):**
- `CREATE TABLE nominations (...)` (see Summary Table for full content).

---

### Block 2.5 — Reporting, CSV Export, Slack + Google Drive Sharing
**Overview:** Queries MySQL to build a nominations report, converts results to a CSV file, then uploads the CSV to Slack and Google Drive.  
**Nodes involved:** `Generate the report with a query`, `Convert to File`, `Upload a file`, `Upload file`

#### Node: Generate the report with a query
- **Type / role:** `MySQL` — executes a SELECT query to produce report rows.
- **Config choices:**
  - Operation: `executeQuery`
  - Query joins nominations to applications by `n.app_id = a.apple_app_id`
  - Orders by `submitted_at desc`
  - Selected fields: `app_name, nomination_name, nomination_id, app_id, bundle_id, sub_type, submitted_at, publish_start_at, state`
- **Connections:**
  - Output → `Convert to File`
- **Edge cases / failures:**
  - If schema/database name differs (`app_store_featurings` hardcoded), query fails.
  - If `submitted_at` is null for many rows, ordering may not reflect reality.
- **Version notes:** typeVersion **2.5**.

#### Node: Convert to File
- **Type / role:** `Convert to File` — turns JSON rows into a binary file (CSV).
- **Config choices:**
  - Output binary property: `nominations`
  - Filename: `nominations_{{ $now.toString() }}.csv`
- **Connections:**
  - Output → `Upload file` (Google Drive)
  - Output → `Upload a file` (Slack)
- **Edge cases / failures:**
  - `$now.toString()` may include characters not ideal for filenames (spaces, colons) depending on environment.
  - CSV column order depends on incoming JSON keys and node behavior.
- **Version notes:** typeVersion **1.1**.

#### Node: Upload a file (Slack)
- **Type / role:** `Slack` — uploads the CSV to a Slack channel.
- **Config choices:**
  - Resource: `file` (file upload)
  - Binary property: `nominations`
  - Channel: `C09L0022N68` (hardcoded via expression)
  - Initial comment randomly chosen between two messages:
    - `"Hi! here's the list of nominations :tada:"`
    - `"Hello!:smiling_face_with_3_hearts: list of nominations below, enjoy!  "`
- **Credentials:** `slackApi Credential`
- **Connections:** terminal.
- **Edge cases / failures:**
  - Missing `files:write` permission, wrong channel ID, or Slack app not in channel.
  - Slack file upload limits (size) unlikely for small CSVs but possible if large dataset.
- **Version notes:** typeVersion **2.3**.

#### Node: Upload file (Google Drive)
- **Type / role:** `Google Drive` — uploads the CSV into a specific folder.
- **Config choices:**
  - Authentication: `serviceAccount`
  - Drive: “N8N drive” (ID: `0AMSgS_z8k1A-Uk9PVA`)
  - Folder: “App Store Featuring Submissions” (ID: `1QaoP_--EGbSNMqFfe3WcUff7TugNj3En`)
  - Input binary field: `nominations`
  - File name: `nominations_{{$now.toString()}}.csv`
- **Credentials:** `googleApi Credential` (service account must have access to the target folder/drive)
- **Connections:** terminal.
- **Edge cases / failures:**
  - Service account not shared onto the folder/shared drive → 403.
  - Shared drive permissions differ from “My Drive”; needs correct Drive API scopes and sharing.
- **Version notes:** typeVersion **3**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | Schedule Trigger | Weekly entry point; starts ingestion and reporting | — | JWT Auth; Generate the report with a query | ### Apple App Store Connect: Featuring Nominations Report\n\nThis workflow automates the process of tracking and reporting app nominations submitted to Apple for App Store featuring consideration. It connects to the App Store Connect API to fetch your list of apps and submitted nominations, stores the data in a MySQL database, and generates a report of all nominations. The report is then exported as a CSV file and can be automatically shared via Google Drive and Slack.\n\n#### Key features\n* Authenticates with App Store Connect using JWT.\n* Fetches all apps and submitted nominations, including details and related in-app events (API documentation: https://developer.apple.com/documentation/appstoreconnectapi/featuring-nominations)\n* Stores and updates app and nomination data in MySQL tables.\n* Generates a comprehensive nominations report with app and nomination details.\n* Exports the report as a CSV file.\n* Shares the report automatically to Google Drive and Slack.\n* Runs on a weekly schedule, but can be triggered manually as well.\n\n#### Setup Instructions\n* Obtain your App Store Connect API credentials (Issuer ID, Key ID, and private key) from your Apple Developer account.\n* Set up a MySQL database and configure the connection details in the workflow’s MySQL node(s).\n* (Optional) Connect your Google Drive and Slack accounts using the respective n8n nodes if you want to share the report automatically.\n* Update any credentials in the workflow to match your setup.\n* Activate the workflow and set the schedule as needed.\n\n\nThis template is ideal for teams who regularly submit apps or updates for featuring on the App Store and want to keep track of their nomination history and status in a structured, automated way. |
| JWT Auth | JWT | Generate Apple ASC JWT token | Weekly Trigger | Fetch submitted nominations from App Store Connect; Fetch apps from App Store Connect | App Store Connect API authentication |
| Fetch apps from App Store Connect | HTTP Request | Get apps list from ASC API (paginated) | JWT Auth | Split Out1 | Fetch applications from App Store Connect API and store them in MySQL |
| Split Out1 | Split Out | Split `data` array (apps) into items | Fetch apps from App Store Connect | Store applications in MySQL | Fetch applications from App Store Connect API and store them in MySQL |
| Store applications in MySQL | MySQL | Upsert apps into `applications` | Split Out1 | — | CREATE TABLE `applications` (\n  `id` int NOT NULL AUTO_INCREMENT,\n  `apple_app_id` bigint NOT NULL,\n  `name` text NOT NULL,\n  `bundle_id` text NOT NULL,\n  PRIMARY KEY (`id`),\n  UNIQUE KEY `apple_app_id_UNIQUE` (`apple_app_id`)\n) ENGINE=InnoDB AUTO_INCREMENT=6378 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
| Fetch submitted nominations from App Store Connect | HTTP Request | Get submitted nominations from ASC API | JWT Auth | Split Out | Fetch nominations from App Store Connect API and store them in MySQL |
| Split Out | Split Out | Split `data` array (nominations) into items | Fetch submitted nominations from App Store Connect | Store nominations in MySQL | Fetch nominations from App Store Connect API and store them in MySQL |
| Store nominations in MySQL | MySQL | Upsert nominations into `nominations` | Split Out | — | CREATE TABLE `nominations` (\n  `id` int NOT NULL AUTO_INCREMENT,\n  `nomination_id` varchar(45) NOT NULL,\n  `app_id` bigint DEFAULT NULL,\n  `type` varchar(45) DEFAULT NULL,\n  `name` text,\n  `sub_type` varchar(45) DEFAULT NULL,\n  `description` text,\n  `created_at` datetime DEFAULT NULL,\n  `last_modified_at` datetime DEFAULT NULL,\n  `submitted_at` datetime DEFAULT NULL,\n  `state` varchar(45) DEFAULT NULL,\n  `publish_start_at` datetime DEFAULT NULL,\n  `publish_end_at` datetime DEFAULT NULL,\n  `notes` text,\n  `link` text,\n  `in_app_event` bigint DEFAULT NULL,\n  `store_account` varchar(45) DEFAULT NULL,\n  PRIMARY KEY (`id`),\n  UNIQUE KEY `nomination_id_UNIQUE` (`nomination_id`)\n) ENGINE=InnoDB AUTO_INCREMENT=911 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
| Generate the report with a query | MySQL | Build report dataset from DB join | Weekly Trigger | Convert to File | Run a SQL query to generate the report and upload it to Slack and Google Drive |
| Convert to File | Convert to File | Convert report rows to CSV binary | Generate the report with a query | Upload file; Upload a file | Run a SQL query to generate the report and upload it to Slack and Google Drive |
| Upload a file | Slack | Upload CSV to Slack channel | Convert to File | — | Run a SQL query to generate the report and upload it to Slack and Google Drive |
| Upload file | Google Drive | Upload CSV to Drive folder | Convert to File | — | Run a SQL query to generate the report and upload it to Slack and Google Drive |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add “Weekly Trigger” (Schedule Trigger)**
   - Interval: every **7 days**
   - Time: **10:00**
3. **Add “JWT Auth” (JWT node)**
   - Set header option `kid` to your **App Store Connect Key ID**
   - Enable **Use JSON**
   - Claims JSON:
     - `iss`: your **Issuer ID**
     - `aud`: `appstoreconnect-v1`
     - `bid`: your app bundle id (optional depending on your implementation)
     - `exp`: now + 10 minutes (seconds)
   - Configure **JWT credentials** with Apple private key (from `.p8`) and algorithm (Apple uses ES256).
4. **Add “Fetch apps from App Store Connect” (HTTP Request)**
   - Method: GET
   - URL: `https://api.appstoreconnect.apple.com/v1/apps`
   - Auth: **Bearer**
     - Configure an **HTTP Bearer Auth credential** that provides `Authorization: Bearer <JWT>`.
     - If you need the token to be dynamic per run, set the node to send the header using an expression referencing the JWT node output (implementation varies by n8n version/credential capabilities).
   - Pagination:
     - Enable pagination “next URL in response”
     - Next URL: `{{$response.body.links["next"]}}`
     - Max requests: `3`
5. **Add “Split Out1” (Split Out)**
   - Field to split out: `data`
6. **Add “Store applications in MySQL” (MySQL)**
   - Configure MySQL credentials (host, user, password, database).
   - Operation: **Upsert**
   - Table: `applications`
   - Match:
     - Column: `apple_app_id`
     - Value: `{{$json.id.toNumber()}}`
   - Values:
     - `name` = `{{$json.attributes.name}}`
     - `bundle_id` = `{{$json.attributes.bundleId}}`
   - Ensure the `applications` table exists (use schema from sticky note).
7. **Add “Fetch submitted nominations from App Store Connect” (HTTP Request)**
   - Method: GET
   - URL: `https://api.appstoreconnect.apple.com/v1/nominations`
   - Auth: same Bearer approach as apps node
   - Enable query parameters:
     - `filter[state]` = `SUBMITTED`
     - `include` = `inAppEvents,relatedApps`
   - (Recommended) Add pagination if you expect many nominations.
8. **Add “Split Out” (Split Out)**
   - Field: `data`
9. **Add “Store nominations in MySQL” (MySQL)**
   - Operation: **Upsert**
   - Table: `nominations`
   - Match:
     - Column: `nomination_id`
     - Value: `{{$json.id}}`
   - Map values:
     - `type`, `name`, `sub_type`, `description`, `state`, `notes`, `link`
     - Datetimes: parse ISO → UTC → `yyyy-MM-dd HH:mm:ss` using Luxon
     - `app_id` from `relationships.relatedApps.data[0].id.toNumber()`
     - `in_app_event` from `relationships.inAppEvents.data[0].id`
   - Ensure the `nominations` table exists (use schema from sticky note).
10. **Add “Generate the report with a query” (MySQL)**
    - Operation: `Execute Query`
    - Paste the SELECT statement used to build the report (adjust database/schema name if needed).
11. **Add “Convert to File” (Convert to File)**
    - Output binary property: `nominations`
    - File name: `nominations_{{ $now.toString() }}.csv`
    - Ensure output format is CSV (node default behavior for tabular JSON).
12. **Add “Upload a file” (Slack)**
    - Resource: `File`
    - Binary property: `nominations`
    - Channel ID: `C09L0022N68` (or your channel)
    - Initial comment: use the provided random message expression (optional)
    - Configure Slack API credentials with permission to upload files.
13. **Add “Upload file” (Google Drive)**
    - Operation: upload (file)
    - Authentication: **Service Account**
    - Drive ID and Folder ID: select your target location
    - Input binary field: `nominations`
    - Configure Google service account credentials and share the folder/drive with the service account email.
14. **Connect nodes exactly as follows:**
    - Weekly Trigger → JWT Auth
    - Weekly Trigger → Generate the report with a query
    - JWT Auth → Fetch apps from App Store Connect → Split Out1 → Store applications in MySQL
    - JWT Auth → Fetch submitted nominations from App Store Connect → Split Out → Store nominations in MySQL
    - Generate the report with a query → Convert to File → (Slack Upload) and → (Google Drive Upload)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Apple Featuring Nominations API documentation | https://developer.apple.com/documentation/appstoreconnectapi/featuring-nominations |
| MySQL table schema for `applications` | Included in sticky note content in the workflow (see Summary Table). |
| MySQL table schema for `nominations` | Included in sticky note content in the workflow (see Summary Table). |
| Workflow purpose and setup checklist (Apple credentials, MySQL, optional Slack/Drive) | Included in the large sticky note describing the workflow (see Summary Table). |
| Disclaimer (provided by user) | Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. |