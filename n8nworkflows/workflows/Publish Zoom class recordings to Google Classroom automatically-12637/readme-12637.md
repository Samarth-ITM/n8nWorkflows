Publish Zoom class recordings to Google Classroom automatically

https://n8nworkflows.xyz/workflows/publish-zoom-class-recordings-to-google-classroom-automatically-12637


# Publish Zoom class recordings to Google Classroom automatically

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Publish Zoom class recordings to Google Classroom automatically

**Purpose:**  
Automatically publishes a Zoom cloud recording (via Zoom `recording.completed` webhook) into Google Classroom as a **CourseWorkMaterial link** under a specific **topic** (e.g., “2. Class Materials”). The workflow also supports a **manual form webhook** for fallback uploads and includes Zoom **endpoint URL validation** (handshake) logic.

**Target use cases:**
- Online schools/teachers using **Zoom** for classes and **Google Classroom** to distribute recording links.
- Environments where meeting names are expected to match Classroom course names exactly.
- Need to filter out short meetings and alert when mapping (class/topic) fails.

### Logical Blocks
1.1 **Zoom Webhook Reception & Validation Handshake**  
Handles Zoom URL validation challenge and routes real events into processing.

1.2 **Recording Duration Filter**  
Rejects recordings shorter than a configurable threshold.

1.3 **Input Normalization (Zoom vs Manual Form) + Merge**  
Creates a unified `{ topic, shareLink }` payload from either Zoom event or manual form.

1.4 **Classroom Course Discovery + AI-based Name Validation**  
Fetches Classroom courses, extracts active class names, then uses OpenAI to validate whether the Zoom meeting title corresponds to a real active class.

1.5 **Material Title Generation + Course Lookup**  
Generates a material title (date) and finds the target ACTIVE course by exact name match.

1.6 **Topic Lookup + Publish Material**  
Fetches topics, finds the configured topic, publishes a CourseWorkMaterial link. Sends Gmail alerts if class/topic not found.

---

## 2. Block-by-Block Analysis

### 2.1 Zoom Webhook Reception & Validation Handshake

**Overview:**  
Receives Zoom webhook requests. If the event is `endpoint.url_validation`, it performs the HMAC challenge response (Zoom handshake) and responds immediately. Otherwise, it proceeds to normal recording processing.

**Nodes involved:**
- Zoom Webhook
- Webhook Validation?
- Encryption
- Set Data
- Respond to Webhook

#### Node: **Zoom Webhook**
- **Type / Role:** `Webhook` (trigger) — entry point for Zoom events.
- **Key configuration:**
  - Method: `POST`
  - Response mode: `responseNode` (workflow must use “Respond to Webhook” node for responses)
  - Path: `7b08a076-7beb-4eec-959a-3e1b2051db79`
- **Outputs:** To `Webhook Validation?`
- **Edge cases / failures:**
  - Zoom not reaching endpoint (network, TLS, firewall).
  - If the workflow is not active/published, Zoom validation fails.
  - Missing or unexpected payload structure will break expressions downstream (e.g., `body.payload.object.topic`).

#### Node: **Webhook Validation?**
- **Type / Role:** `IF` — routes Zoom URL validation vs normal event.
- **Condition:** `$json.body.event === "endpoint.url_validation"`
- **True output:** to `Encryption` (handshake path)  
- **False output:** to `Set Parameters` (normal processing path)
- **Edge cases:**
  - If `body.event` missing, condition evaluates false and flow goes to normal path.

#### Node: **Encryption**
- **Type / Role:** `Crypto` — computes Zoom HMAC for URL validation.
- **Configuration choices:**
  - Action: `hmac`
  - Hash: `SHA256`
  - Value: `{{ $json.body.payload.plainToken }}`
  - Secret: `YOUR_CREDENTIAL_HERE` (must match Zoom “Secret Token” / validation secret configured in Zoom webhook app)
- **Output:** to `Set Data`
- **Edge cases:**
  - Wrong secret ⇒ Zoom validation fails.
  - Missing `payload.plainToken` ⇒ node errors or returns unusable hash.

#### Node: **Set Data**
- **Type / Role:** `Set` — prepares the handshake response body.
- **Key fields:**
  - `plainToken = {{ $('Zoom Webhook').item.json.body.payload.plainToken }}`
  - `encryptedToken = {{ $json.data }}` (crypto output is in `data`)
- **Output:** to `Respond to Webhook`
- **Edge cases:**
  - If execution context changes and `Zoom Webhook` item is not available, expression fails.

#### Node: **Respond to Webhook**
- **Type / Role:** `RespondToWebhook` — returns JSON to Zoom for URL validation.
- **Respond with:** JSON
- **Response body:**
  - `plainToken` and `encryptedToken` echoed back from the current item
- **Edge cases:**
  - If used on non-validation events, Zoom may not expect this response; however it is only connected on validation branch.

**Sticky note (applies to this block):**
- **“Zoom Webhook Validation — It's a Zoom's initial handshake.”**

---

### 2.2 Recording Duration Filter

**Overview:**  
For real recording events, computes the actual duration from Zoom recording file timestamps and filters out recordings shorter than a threshold (default 30 minutes).

**Nodes involved:**
- Set Parameters
- Duration Filtering
- Relevant Duration?

#### Node: **Set Parameters**
- **Type / Role:** `Set` — central configuration for thresholds and target topic name.
- **Key fields:**
  - `durationThreshold = 30`
  - `desiredTopic = "2. Class Materials"`
- **Output:** to `Duration Filtering`
- **Edge cases:**
  - Setting `desiredTopic` incorrectly will later trigger topic-not-found email path.

#### Node: **Duration Filtering**
- **Type / Role:** `Code` — validates recording data and computes duration.
- **Logic summary:**
  - Reads Zoom payload: `$('Zoom Webhook').item.json.body.payload.object`
  - Reads threshold: `$input.first().json.durationThreshold`
  - Uses first `recording_files[0]` and requires `recording_start` and `recording_end`
  - Calculates minutes; rejects if `< desiredThreshold`
- **Output:** boolean-like `json.status` with reason/duration details.
- **Edge cases / failures:**
  - Zoom payload may not include `recording_files` or timestamps ⇒ returns `{status:false, reason:"missing_timestamps"}`.
  - Uses only the first recording file; if that file lacks timestamps but another has them, it will incorrectly fail.

#### Node: **Relevant Duration?**
- **Type / Role:** `IF` — continues only if duration check passed.
- **Condition:** `{{ $json.status }}` is boolean true.
- **True output:** to `Set From Zoom`
- **False output:** no further connection (recording is silently ignored)
- **Edge cases:**
  - If `Duration Filtering` returns a non-boolean `status`, strict validation may cause issues (it returns boolean, so OK).

**Sticky note (applies to this block):**
- **“Check Call Duration — Specify the threshold in the previous node and filter out calls that do not match.”**

---

### 2.3 Input Normalization (Zoom vs Manual Form) + Merge

**Overview:**  
Normalizes input into a common shape `{ topic, shareLink }` either from the Zoom webhook payload or from a manual form webhook, then merges both paths into one stream used by downstream Classroom logic.

**Nodes involved:**
- Form Webhook
- Set From Form
- Set From Zoom
- Merge

#### Node: **Form Webhook**
- **Type / Role:** `Webhook` — manual/fallback entry point.
- **Key configuration:**
  - Method: `POST`
  - Path: `0ab18fbb-1c85-42da-ad80-d5e663aa0668`
- **Output:** to `Set From Form`
- **Edge cases:**
  - Payload format assumed: `$json.body["Classroom Name"]` and `$json.body["Zoom Share Link"]`. If form provider sends different structure, mapping breaks.

#### Node: **Set From Form**
- **Type / Role:** `Set` — maps form fields to normalized fields.
- **Key fields:**
  - `topic = {{ $json.body["Classroom Name"] }}`
  - `shareLink = {{ $json.body["Zoom Share Link"] }}`
- **Output:** to `Merge` (input index 1)
- **Edge cases:**
  - If form fields missing, `topic/shareLink` become empty and downstream matching fails.

#### Node: **Set From Zoom**
- **Type / Role:** `Set` — maps Zoom data to normalized fields.
- **Key fields (note the node references the non-validation branch context):**
  - `topic = {{ $('Webhook Validation?').item.json.body.payload.object.topic }}`
  - `shareLink = {{ $('Webhook Validation?').item.json.body.payload.object.share_url }}`
- **Output:** to `Merge` (input index 0)
- **Edge cases:**
  - Relies on `Webhook Validation?` node context rather than directly `Zoom Webhook`; should work but is somewhat brittle if branches/items change.

#### Node: **Merge**
- **Type / Role:** `Merge` — joins normalized payload streams.
- **Configuration:** default merge behavior (no explicit mode set in parameters).
- **Output:** to `Get All Classrooms`
- **Edge cases:**
  - Merge behavior depends on n8n defaults (often “append” or “merge by index”). If both triggers run separately, you generally expect single input at a time. If both inputs arrive in one execution (rare), you may get unexpected pairing.

**Sticky note (applies to this block):**
- **“Form Input — Create a separate form for manual uploads if something fails…”**

---

### 2.4 Classroom Course Discovery + AI-based Name Validation

**Overview:**  
Fetches all Classroom courses, extracts ACTIVE course names, and asks an OpenAI model to decide whether the Zoom meeting name corresponds to an actual class recording and whether the class exists.

**Nodes involved:**
- Get All Classrooms
- Return Active Classes' Names
- Check Classes
- Switch
- Issue with Classes

#### Node: **Get All Classrooms**
- **Type / Role:** `HTTP Request` — calls Google Classroom API.
- **Request:**
  - `GET https://classroom.googleapis.com/v1/courses`
  - Auth: Google OAuth2 credential (`Classroom Access`)
- **Output:** to `Return Active Classes' Names`
- **Edge cases / failures:**
  - 401/403 due to OAuth issues or missing scopes.
  - Pagination: Classroom API may paginate; this node does not handle `nextPageToken`, so large course lists may be incomplete.

#### Node: **Return Active Classes' Names**
- **Type / Role:** `Code` — extracts active course names only.
- **Logic:**
  - Filters `payload.courses` by `courseState === "ACTIVE"`
  - Returns `{ courses: [name1, name2, ...] }`
- **Output:** to `Check Classes`
- **Edge cases:**
  - If `payload.courses` is missing/undefined, code will throw (no guard).

#### Node: **Check Classes**
- **Type / Role:** `OpenAI (LangChain)` — validates Zoom meeting name vs active classes list.
- **Model:** `gpt-4.1-mini`
- **Response format:** JSON object (enforced via `textFormat: json_object`); `simplify: false` (raw structured output kept).
- **Prompt contract:** Must return:
  - `{"status": boolean, "message": "<one of 3 exact messages>" }`
  - Messages allowed:
    1) `"zoom call is not class recording."` (status false)  
    2) `"zoom call is class recording but class not found."` (status false)  
    3) `"zoom is class recording and class is found."` (status true)
- **Inputs used:**
  - Zoom name: `{{ $('Zoom Webhook').item.json.body.payload.object.topic }}`
  - Active classes: `{{ $json.courses }}`
- **Output:** to `Switch`
- **Edge cases / failures:**
  - Model may output JSON not matching the required schema; downstream `Switch` expects deep fields like `$json.output[0].content[0].text.status`.
  - If Zoom trigger wasn’t used (manual form path), this node still references `Zoom Webhook` topic, which may be absent and break the validation logic.

#### Node: **Switch**
- **Type / Role:** `Switch` — routes based on the LLM output.
- **Rules / outputs:**
  - Output “True”: checks `{{ $json.output[0].content[0].text.status }}` is boolean true
    - Goes to `Generate Material Title`
  - Output “False”: checks message equals `"zoom call is class recording but class not found."`
    - Goes to `Issue with Classes`
- **Edge cases:**
  - If model output changes shape, the expressions will fail or not match.
  - Other “false” message (`zoom call is not class recording.`) is not routed; it will drop (no connection).

#### Node: **Issue with Classes**
- **Type / Role:** `Gmail` — alerts when class recording is detected but class not found.
- **To:** `user@example.com`
- **Subject:** `Course not found`
- **Body:** Uses model message and instructs manual upload:
  - `{{ $('Check Classes').item.json.output[0].content[0].text.message }}. Upload video manually using this form: {INSERT MANUAL FORM LINK}`
- **Edge cases:**
  - Requires Gmail OAuth2 credential.
  - Placeholder `{INSERT MANUAL FORM LINK}` must be replaced.
  - If LLM output schema mismatch, message expression fails.

**Sticky notes (apply to this block and downstream class lookup):**
- **“Validate calls by name — Here we check if call name has a match with a Class”**
- **“Find Class & Upload Materials — Here we get the course ID…search for specified topic and actually upload the call to Google Class”**

---

### 2.5 Material Title Generation + Course Lookup

**Overview:**  
Generates a title from the Zoom meeting start date and finds the matching ACTIVE Google Classroom course ID by exact case-insensitive name match to the normalized `topic` field.

**Nodes involved:**
- Generate Material Title
- Find Active Course by Name

#### Node: **Generate Material Title**
- **Type / Role:** `Code` — creates a human-friendly title.
- **Logic:**
  - Reads `start_time` from Zoom payload.
  - Builds `D.M.YYYY` (no leading zeros), e.g., `7.1.2026`.
- **Output:** `{ title }`
- **Edge cases:**
  - If `start_time` missing/invalid, date parsing yields `Invalid Date`.

#### Node: **Find Active Course by Name**
- **Type / Role:** `Code` — maps the merged `topic` to an ACTIVE course.
- **Inputs:**
  - Target course name: `$('Merge').first().json.topic` (trimmed)
  - Courses payload: `$('Get All Classrooms').first().json.courses`
- **Logic:**
  - Filters courses to `courseState === 'ACTIVE'`
  - Exact match (case-insensitive): `course.name.trim().toLowerCase() === target`
- **Outputs:**
  - If found: `{ status:"Found", courseId, courseName }`
  - Else: `{ status:"Not Found", searchedName, activeCourseCount }`
- **Edge cases:**
  - If `Get All Classrooms` returns no `courses`, it uses empty array (guard exists).
  - If the course name differs slightly (extra spaces, separators), it won’t match.

---

### 2.6 Topic Lookup + Publish Material

**Overview:**  
Fetches topics for the identified course, finds the configured topic name, and publishes the Zoom share link as a Classroom material. Sends email if topic not found.

**Nodes involved:**
- Get All Topics in Class
- Find Topic
- If2
- Publish Materials to Topic
- Topic Not Found

#### Node: **Get All Topics in Class**
- **Type / Role:** `HTTP Request` — lists topics in a course.
- **Request:**
  - `GET https://classroom.googleapis.com/v1/courses/{{ $json.courseId }}/topics`
  - Auth: Google OAuth2 (`Classroom Access`)
- **alwaysOutputData:** true (ensures output item exists even on some failures)
- **Output:** to `Find Topic`
- **Edge cases:**
  - If `courseId` missing (course not found), request URL becomes invalid ⇒ 404/400.
  - Missing scopes: needs topic read scope.

#### Node: **Find Topic**
- **Type / Role:** `Code` — finds topic by exact name.
- **Inputs:**
  - Topics array: `$input.item.json.topic` (note: API usually returns `topic`, but may differ; must match actual response)
  - Desired topic: `$('Set Parameters').first().json.desiredTopic`
- **Output (non-standard shape):**
  - On found: `{ status:"found", topicId }`
  - Else: `{ status:"not found" }`
- **Edge cases / failures:**
  - If the Classroom API returns `topic` vs `topics` (API commonly returns `topic: []` or `topic` field naming can vary by wrapper), mismatch breaks lookup.
  - The node returns a plain object rather than `{ json: {...} }` array item format typical in n8n code nodes; depending on n8n version, this may still work or may require wrapping in `return [{json: {...}}]`.

#### Node: **If2**
- **Type / Role:** `IF` — routes found vs not found topic.
- **Condition:** `{{ $json.status }} === "found"`
- **True output:** to `Publish Materials to Topic`
- **False output:** to `Topic Not Found`
- **Edge cases:**
  - If `Find Topic` output format is wrong, `status` may not exist and routing fails.

#### Node: **Publish Materials to Topic**
- **Type / Role:** `HTTP Request` — creates a CourseWorkMaterial in Classroom.
- **Request:**
  - `POST https://classroom.googleapis.com/v1/courses/{{ courseId }}/courseWorkMaterials`
  - Uses `courseId` from `Find Active Course by Name`
  - JSON body includes:
    - `title` from `Generate Material Title`
    - `state: "PUBLISHED"`
    - `topicId` from `Find Topic`
    - `materials: [{ link: { url: <Zoom share_url>, title: "Zoom Recording" } }]`
- **Important detail:** The link URL is taken from **Zoom Webhook**:
  - `{{ $('Zoom Webhook').item.json.body.payload.object.share_url }}`
  - This ignores the merged `shareLink` (manual form path won’t work unless Zoom data exists).
- **Edge cases / failures:**
  - Missing Classroom scopes for creating materials ⇒ 403.
  - If topicId invalid ⇒ 400.
  - Trailing whitespace in URL expression (`"{{ ... }} "` includes a space) could create an invalid URL in some cases.

#### Node: **Topic Not Found**
- **Type / Role:** `Gmail` — alerts when topic is missing.
- **To:** `user@example.com`
- **Subject:** `Course not found` (subject text is misleading; it’s topic-not-found)
- **Body:** `Topic "2. Class Materials" is not found for <courseName>`
- **Edge cases:**
  - Hard-coded topic name in message; if `desiredTopic` changes, message should be updated to reference it dynamically.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Zoom Webhook | Webhook | Receives Zoom events (recording + URL validation) | — | Webhook Validation? | Zoom Webhook Validation — It's a Zoom's initial handshake. |
| Webhook Validation? | IF | Routes URL validation vs normal processing | Zoom Webhook | Encryption; Set Parameters | Zoom Webhook Validation — It's a Zoom's initial handshake. |
| Encryption | Crypto | Computes HMAC SHA256 for Zoom URL validation | Webhook Validation? (true) | Set Data | Zoom Webhook Validation — It's a Zoom's initial handshake. |
| Set Data | Set | Builds `plainToken/encryptedToken` response | Encryption | Respond to Webhook | Zoom Webhook Validation — It's a Zoom's initial handshake. |
| Respond to Webhook | RespondToWebhook | Sends validation response back to Zoom | Set Data | — | Zoom Webhook Validation — It's a Zoom's initial handshake. |
| Set Parameters | Set | Stores duration threshold + desired Classroom topic | Webhook Validation? (false) | Duration Filtering | Check Call Duration — Specify the threshold in the previous node and filter out calls that do not match. |
| Duration Filtering | Code | Calculates duration from recording timestamps | Set Parameters | Relevant Duration? | Check Call Duration — Specify the threshold in the previous node and filter out calls that do not match. |
| Relevant Duration? | IF | Continues only for long-enough recordings | Duration Filtering | Set From Zoom | Check Call Duration — Specify the threshold in the previous node and filter out calls that do not match. |
| Set From Zoom | Set | Normalizes Zoom payload into `{topic, shareLink}` | Relevant Duration? | Merge | Validate calls by name — Here we check if call name has a match with a Class |
| Form Webhook | Webhook | Manual fallback entry point | — | Set From Form | Form Input — Create a separate form for manual uploads if something fails… |
| Set From Form | Set | Normalizes form fields into `{topic, shareLink}` | Form Webhook | Merge | Form Input — Create a separate form for manual uploads if something fails… |
| Merge | Merge | Unifies Zoom + Form normalized inputs | Set From Zoom; Set From Form | Get All Classrooms | Validate calls by name — Here we check if call name has a match with a Class |
| Get All Classrooms | HTTP Request | Lists Google Classroom courses | Merge | Return Active Classes' Names | Find Class & Upload Materials — Here we get the course ID…upload the call to Google Class |
| Return Active Classes' Names | Code | Extracts names of ACTIVE courses | Get All Classrooms | Check Classes | Validate calls by name — Here we check if call name has a match with a Class |
| Check Classes | OpenAI (LangChain) | Validates meeting name vs active classes | Return Active Classes' Names | Switch | Validate calls by name — Here we check if call name has a match with a Class |
| Switch | Switch | Routes based on LLM result | Check Classes | Generate Material Title; Issue with Classes | Validate calls by name — Here we check if call name has a match with a Class |
| Issue with Classes | Gmail | Alerts when class not found | Switch (False) | — | Form Input — Create a separate form for manual uploads if something fails… |
| Generate Material Title | Code | Generates material title from meeting date | Switch (True) | Find Active Course by Name | Find Class & Upload Materials — Here we get the course ID…upload the call to Google Class |
| Find Active Course by Name | Code | Finds ACTIVE course by exact name match | Generate Material Title | Get All Topics in Class | Find Class & Upload Materials — Here we get the course ID…upload the call to Google Class |
| Get All Topics in Class | HTTP Request | Lists topics in the matched course | Find Active Course by Name | Find Topic | Find Class & Upload Materials — Here we get the course ID…upload the call to Google Class |
| Find Topic | Code | Finds the configured topic by name | Get All Topics in Class | If2 | Find Class & Upload Materials — Here we get the course ID…upload the call to Google Class |
| If2 | IF | Routes topic found vs not found | Find Topic | Publish Materials to Topic; Topic Not Found | Find Class & Upload Materials — Here we get the course ID…upload the call to Google Class |
| Publish Materials to Topic | HTTP Request | Creates Classroom material with Zoom link | If2 (true) | — | Find Class & Upload Materials — Here we get the course ID…upload the call to Google Class |
| Topic Not Found | Gmail | Alerts when topic missing | If2 (false) | — | Find Class & Upload Materials — Here we get the course ID…upload the call to Google Class |
| Sticky Note | StickyNote | Comment | — | — | ## Validate calls by name… |
| Sticky Note1 | StickyNote | Comment | — | — | ## Zoom Webhook Validation… |
| Sticky Note2 | StickyNote | Comment | — | — | ## Form Input… |
| Sticky Note3 | StickyNote | Comment | — | — | ## Check Call Duration… |
| Sticky Note4 | StickyNote | Comment | — | — | ## Find Class & Upload Materials… |
| Sticky Note5 | StickyNote | Comment / setup notes | — | — | ## Auto-publish Zoom recordings to Google Classroom… (includes Google scopes list) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook trigger (Zoom)**
   1. Add node: **Webhook**
   2. Method: `POST`
   3. Response mode: `Using Respond to Webhook node`
   4. Set a unique **Path** (this becomes your Zoom webhook URL)
   5. Name it: `Zoom Webhook`

2) **Add URL validation routing**
   1. Add node: **IF** named `Webhook Validation?`
   2. Condition: `{{$json.body.event}}` equals `endpoint.url_validation`
   3. Connect: `Zoom Webhook` → `Webhook Validation?`

3) **Implement Zoom URL validation response**
   1. Add node: **Crypto** named `Encryption`
      - Action: `HMAC`
      - Algorithm: `SHA256`
      - Value: `={{ $json.body.payload.plainToken }}`
      - Secret: set to your Zoom webhook secret (store in n8n credentials/variables if possible)
   2. Add node: **Set** named `Set Data`
      - Set `plainToken` to `={{ $('Zoom Webhook').item.json.body.payload.plainToken }}`
      - Set `encryptedToken` to `={{ $json.data }}`
   3. Add node: **Respond to Webhook** named `Respond to Webhook`
      - Respond with: `JSON`
      - Body:
        - `plainToken: {{ $json.plainToken }}`
        - `encryptedToken: {{ $json.encryptedToken }}`
   4. Connect: `Webhook Validation? (true)` → `Encryption` → `Set Data` → `Respond to Webhook`

4) **Add parameter node (threshold + desired topic)**
   1. Add node: **Set** named `Set Parameters`
   2. Add fields:
      - `durationThreshold` (number), e.g. `30`
      - `desiredTopic` (string), e.g. `2. Class Materials`
   3. Connect: `Webhook Validation? (false)` → `Set Parameters`

5) **Add duration filtering**
   1. Add node: **Code** named `Duration Filtering`
      - Implement logic to:
        - Read Zoom payload object
        - Compute duration from `recording_start`/`recording_end`
        - Compare vs `durationThreshold`
        - Output `{status:true/false}`
   2. Add node: **IF** named `Relevant Duration?`
      - Condition: `{{ $json.status }}` is `true`
   3. Connect: `Set Parameters` → `Duration Filtering` → `Relevant Duration?`

6) **Normalize Zoom data**
   1. Add node: **Set** named `Set From Zoom`
      - `topic = {{ $('Webhook Validation?').item.json.body.payload.object.topic }}`
      - `shareLink = {{ $('Webhook Validation?').item.json.body.payload.object.share_url }}`
   2. Connect: `Relevant Duration? (true)` → `Set From Zoom`

7) **(Optional) Add manual form webhook**
   1. Add node: **Webhook** named `Form Webhook` (POST, unique path)
   2. Add node: **Set** named `Set From Form`
      - `topic = {{ $json.body["Classroom Name"] }}`
      - `shareLink = {{ $json.body["Zoom Share Link"] }}`
   3. Connect: `Form Webhook` → `Set From Form`

8) **Merge normalized inputs**
   1. Add node: **Merge** named `Merge` (default config)
   2. Connect:
      - `Set From Zoom` → `Merge` (Input 1)
      - `Set From Form` → `Merge` (Input 2)

9) **Configure Google Classroom credential**
   1. Create **Google OAuth2 API** credential in n8n (name e.g. “Classroom Access”)
   2. Scopes (as indicated in the note):
      - `https://www.googleapis.com/auth/classroom.courseworkmaterials`
      - `https://www.googleapis.com/auth/classroom.topics.readonly`
      - `https://www.googleapis.com/auth/classroom.courses.readonly`
      - `https://www.googleapis.com/auth/classroom.announcements`
      - `https://www.googleapis.com/auth/classroom.coursework.students`

10) **List all courses**
   1. Add node: **HTTP Request** named `Get All Classrooms`
      - GET `https://classroom.googleapis.com/v1/courses`
      - Authentication: predefined credential → `googleOAuth2Api` (“Classroom Access”)
   2. Connect: `Merge` → `Get All Classrooms`

11) **Extract active class names**
   1. Add node: **Code** named `Return Active Classes' Names`
      - Filter courses to ACTIVE, map to `name`
      - Output `{ courses: [...] }`
   2. Connect: `Get All Classrooms` → `Return Active Classes' Names`

12) **Configure OpenAI credential + LLM validation**
   1. Create **OpenAI API** credential in n8n.
   2. Add node: **OpenAI (LangChain)** named `Check Classes`
      - Model: `gpt-4.1-mini`
      - Output format: JSON object
      - Provide system+user messages enforcing schema `{status:boolean,message:"..."}` and allowed messages
      - Feed in Zoom topic and active classes list
   3. Connect: `Return Active Classes' Names` → `Check Classes`

13) **Route based on LLM output**
   1. Add node: **Switch** named `Switch`
      - Rule 1 (“True”): LLM `status` is true → proceed
      - Rule 2 (“False”): LLM `message` equals `zoom call is class recording but class not found.` → alert
   2. Connect: `Check Classes` → `Switch`

14) **Email on class-not-found**
   1. Create **Gmail OAuth2** credential in n8n.
   2. Add node: **Gmail** named `Issue with Classes`
      - To: your email
      - Subject: e.g. `Course not found`
      - Body: include LLM message + manual form link
   3. Connect: `Switch (False)` → `Issue with Classes`

15) **Generate material title**
   1. Add node: **Code** named `Generate Material Title`
      - Convert Zoom `start_time` into `D.M.YYYY`
      - Output `{ title }`
   2. Connect: `Switch (True)` → `Generate Material Title`

16) **Find the ACTIVE course by name**
   1. Add node: **Code** named `Find Active Course by Name`
      - Read `Merge.topic`
      - Compare to Classroom course names (ACTIVE only), exact case-insensitive match
      - Output `{courseId, courseName}` on match
   2. Connect: `Generate Material Title` → `Find Active Course by Name`

17) **List topics in the course**
   1. Add node: **HTTP Request** named `Get All Topics in Class`
      - GET `https://classroom.googleapis.com/v1/courses/{{ $json.courseId }}/topics`
      - Auth: Google OAuth2 credential
   2. Connect: `Find Active Course by Name` → `Get All Topics in Class`

18) **Find desired topic**
   1. Add node: **Code** named `Find Topic`
      - Find topic whose `name` equals `Set Parameters.desiredTopic`
      - Output `{ status:"found", topicId }` or `{ status:"not found" }`
   2. Connect: `Get All Topics in Class` → `Find Topic`

19) **Route topic found vs not found**
   1. Add node: **IF** named `If2`
      - Condition: `{{ $json.status }}` equals `found`
   2. Connect: `Find Topic` → `If2`

20) **Publish material (topic found)**
   1. Add node: **HTTP Request** named `Publish Materials to Topic`
      - POST `https://classroom.googleapis.com/v1/courses/{{ <courseId> }}/courseWorkMaterials`
      - JSON body includes:
        - Title from `Generate Material Title`
        - `topicId` from `Find Topic`
        - `materials.link.url` = Zoom share URL
      - Auth: Google OAuth2
   2. Connect: `If2 (true)` → `Publish Materials to Topic`

21) **Email if topic not found**
   1. Add node: **Gmail** named `Topic Not Found`
      - To: your email
      - Subject/Body: explain missing topic + course name
   2. Connect: `If2 (false)` → `Topic Not Found`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Auto-publish Zoom recordings to Google Classroom… listens for Zoom webhooks… filters out calls that last less than 30 mins… call must be named exactly as the Google Class… looks for a specific topic… If topic is not found, you'll get an email.” | Sticky note: “Auto-publish Zoom recordings to Google Classroom” |
| “Register the webhook URL in Zoom and enable recording.completed notifications… Validate the URL in Zoom (make sure to publish the workflow)” | Zoom webhook setup guidance (sticky note) |
| “Set the HMAC signing secret in the Crypto node to match Zoom.” | Zoom URL validation requirement (sticky note) |
| Google Classroom OAuth scopes list: `https://www.googleapis.com/auth/classroom.courseworkmaterials`, `https://www.googleapis.com/auth/classroom.topics.readonly`, `https://www.googleapis.com/auth/classroom.courses.readonly`, `https://www.googleapis.com/auth/classroom.announcements`, `https://www.googleapis.com/auth/classroom.coursework.students` | Credential setup guidance (sticky note) |
| “Connect a Gmail account for alert emails… Add OpenAI (LangChain) API credentials for the model used to validate names.” | Credential prerequisites (sticky note) |
| “(Optional) Configure the form webhook for manual upload entries… add a form link to the Gmail node.” | Manual fallback path (sticky note) |
| “The last node in the system returns a direct link to the newly created material…” | Improvement suggestion (sticky note) |

