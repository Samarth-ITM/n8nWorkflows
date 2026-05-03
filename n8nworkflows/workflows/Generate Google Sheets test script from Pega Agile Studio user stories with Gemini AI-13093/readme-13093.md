Generate Google Sheets test script from Pega Agile Studio user stories with Gemini AI

https://n8nworkflows.xyz/workflows/generate-google-sheets-test-script-from-pega-agile-studio-user-stories-with-gemini-ai-13093


# Generate Google Sheets test script from Pega Agile Studio user stories with Gemini AI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow generates a Google Spreadsheet “test script” from a Pega Agile Studio user story (entered as `US-1234` in chat). It retrieves the user story via API, extracts and numbers acceptance criteria, asks Gemini to generate step-level test cases as JSON, and writes both acceptance criteria and test steps into separate Google Sheets tabs. It then performs cleanup to remove repeated values in consecutive rows (e.g., repeated TestID/Scenario).

**Target use cases:**
- Functional/software testers working with **Pega Platform + Pega Agile Studio** who need fast, structured test scripts and traceability to acceptance criteria.

**Logical blocks (node-to-node dependency groups):**
1.1 **Chat input & user story retrieval** → Trigger + HTTP call to Agile Studio  
1.2 **Spreadsheet creation & header provisioning** → Create spreadsheet with 2 tabs, write headers via Google Sheets API  
1.3 **Acceptance criteria normalization & insertion** → Split acceptance criteria list, add numbering, merge with spreadsheetId, append to `Acc_Matrix`  
1.4 **AI test case generation & JSON shaping** → Gemini agent produces JSON array, Code node parses, Split Out turns array into items  
1.5 **Test script insertion & cleanup** → Merge spreadsheetId with test rows, append/update rows, remove redundant repeated fields, clear and re-insert cleaned rows

---

## 2. Block-by-Block Analysis

### 2.1 Chat input & user story retrieval

**Overview:**  
Receives a chat message containing a user story ID (e.g., `US-1234`) and fetches the corresponding user story details from Pega Agile Studio using OAuth2.

**Nodes involved:**
- When chat message received
- Retrieve the US from Pega Agile Studio

#### Node: When chat message received
- **Type / role:** LangChain Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`) – entry point.
- **Configuration:** Default options; provides `chatInput` in the incoming JSON.
- **Key data:** `$json.chatInput` (expected format: `US-1234`).
- **Outputs:** Connected to **Retrieve the US from Pega Agile Studio**.
- **Edge cases / failures:**
  - Empty or malformed `chatInput` → downstream API call likely 404/400.
  - Trigger availability depends on n8n chat feature being enabled/configured.

#### Node: Retrieve the US from Pega Agile Studio
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) – pulls user story data.
- **Configuration choices:**
  - **URL:** `https://dmgbv-agilestudio.pegacloud.io:443/prweb/PRRestService/api/v2/userstories/{{ $json.chatInput }}`
  - **Auth:** Generic credential type using **OAuth2 API** (configured as `oAuth2Api`).
  - Also has an `httpBasicAuth` credential attached, but the node is configured to use OAuth2 (`genericAuthType: oAuth2Api`).
- **Outputs:** Fans out to:
  - IterateOverAccCrit (acceptance criteria processing)
  - AI: Create testcases (AI generation)
  - Create spreadsheet for the testscript (sheet creation)
- **Edge cases / failures:**
  - OAuth token expiration / insufficient scopes → 401/403.
  - User story not found → 404.
  - Pega environment latency/timeouts.
  - Response schema differences: workflow expects `details.*` fields (e.g., `details.acceptanceCriteria`, `details.description`, `details.ID`, `details.name`).

---

### 2.2 Spreadsheet creation & header provisioning

**Overview:**  
Creates a spreadsheet with two sheets (`Acc_Matrix`, `Testscript`) and manually inserts column headers into row 1 of each sheet using Google Sheets API.

**Nodes involved:**
- Create spreadsheet for the testscript
- Add column headers for the first sheet
- Add column headers for the second sheet
- Merge2

#### Node: Create spreadsheet for the testscript
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) – creates a new spreadsheet.
- **Configuration choices:**
  - **Resource:** Spreadsheet
  - **Operation:** Create (implicit via resource + title + sheetsUi)
  - **Title expression:**  
    `{{ $('Retrieve the US from Pega Agile Studio').item.json.details.ID }} - Testscript AI Enhanced - {{ $('Retrieve the US from Pega Agile Studio').item.json.details.name }}`
  - **Sheets created:** `Acc_Matrix`, `Testscript`
- **Credentials:** Google Sheets OAuth2.
- **Outputs:** Connected to:
  - Add column headers for the first sheet
  - Add column headers for the second sheet
  - Merge: Get SpreadsheetID (to carry spreadsheetId to later steps)
- **Edge cases / failures:**
  - Google OAuth scope missing (needs Sheets create/edit).
  - Title expression fails if `details.ID`/`details.name` missing.
  - Google API quotas.

#### Node: Add column headers for the first sheet
- **Type / role:** HTTP Request – direct call to Google Sheets v4 `values.update` endpoint.
- **Configuration choices:**
  - **Method:** PUT  
  - **URL:** `https://sheets.googleapis.com/v4/spreadsheets/{{ $json.spreadsheetId }}/values/Acc_Matrix!A1:Z1?valueInputOption=RAW`
  - **JSON body:** `["Acc. Number", "Acc. Description"]` as first row.
  - **Auth:** predefined credential `googleSheetsOAuth2Api`.
- **Outputs:** To **Merge2** (input 1).
- **Edge cases / failures:**
  - Wrong sheet name (must match exactly `Acc_Matrix`).
  - Missing `spreadsheetId` (must come from Create spreadsheet output).

#### Node: Add column headers for the second sheet
- **Type / role:** HTTP Request – sets headers for `Testscript`.
- **Configuration choices:**
  - **URL:** `.../values/Testscript!A1:Z1?valueInputOption=RAW`
  - **Headers row:**  
    `["TestID","Scenario","Acceptance criteria","Preconditions","Step no.","Steps","Expected result","Status"]`
- **Outputs:** To **Merge2** (input 2).
- **Edge cases / failures:** same as above; requires `Testscript` to exist.

#### Node: Merge2
- **Type / role:** Merge (`n8n-nodes-base.merge`) – synchronization/join point.
- **Configuration choices:**
  - **Mode:** `combineBySql` (no explicit query provided in JSON snippet; default SQL combine behavior for this node mode).
  - Receives one input from each header-writing request.
- **Outputs:** To **Merge** (as input index 1).
- **Functional intent:** Ensure both header operations have completed and provide combined data forward.
- **Edge cases / failures:**
  - If one header call fails, merge won’t produce combined output as expected.
  - `combineBySql` without an explicit query can be fragile depending on item structure; this node is used primarily as a “join” barrier.

---

### 2.3 Acceptance criteria normalization & insertion

**Overview:**  
Takes `details.acceptanceCriteria` (list) from Pega, splits it into individual criteria, assigns sequential numbers, merges with the spreadsheetId, and appends rows into `Acc_Matrix`.

**Nodes involved:**
- IterateOverAccCrit
- AddNumbersToAccCrit
- Merge
- Add acceptance criteria to the Google Spreadsheet

#### Node: IterateOverAccCrit
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) – iterates over acceptance criteria array.
- **Configuration:** `fieldToSplitOut: details.acceptanceCriteria`
- **Outputs:** To **AddNumbersToAccCrit**
- **Edge cases / failures:**
  - If `details.acceptanceCriteria` is not an array (null/string/object), Split Out fails or produces unexpected items.

#### Node: AddNumbersToAccCrit
- **Type / role:** Code (`n8n-nodes-base.code`) – formats each acceptance criterion and adds numbering.
- **Logic (interpreted):**
  - For each incoming item, builds:
    ```js
    {
      Acc: {
        Description: item.json["details.acceptanceCriteria"],
        Number: newItems.length + 1
      }
    }
    ```
  - Produces a clean list of items with `Acc.Number` and `Acc.Description`.
- **Outputs:** To **Merge** (input 0).
- **Edge cases / failures:**
  - This numbering resets per execution, not per user story segment (fine here).
  - It uses `item.json["details.acceptanceCriteria"]` after Split Out; if Split Out produces a different shape, numbering may break.

#### Node: Merge
- **Type / role:** Merge – combines acceptance criteria items with spreadsheet context (spreadsheetId + other data).
- **Configuration choices:**
  - **Mode:** combineBySql
  - **Query:** `SELECT * FROM input1 LEFT JOIN input2 ON input1.id = input2.name`
  - **Inputs:**
    - Input 1: from AddNumbersToAccCrit (acceptance criteria items)
    - Input 2: from Merge2 (result of header creation)
- **Outputs:** To **Add acceptance criteria to the Google Spreadsheet**
- **Edge cases / failures:**
  - The SQL join keys (`input1.id` and `input2.name`) may not exist in the actual items produced. If so, the join may yield nulls or an empty combination depending on node behavior/version.
  - Practical intent appears to be: “carry spreadsheetId forward”; a simpler merge mode (e.g., “merge by position”) might be more robust.

#### Node: Add acceptance criteria to the Google Spreadsheet
- **Type / role:** Google Sheets – append rows to `Acc_Matrix`.
- **Configuration choices:**
  - **Operation:** Append
  - **Sheet:** `Acc_Matrix`
  - **DocumentId:** `{{ $json.spreadsheetId }}`
  - **Columns mapped:**
    - `Acc. Number` ← `{{ $json.Acc.Number }}`
    - `Acc. Description` ← `{{ $json.Acc.Description }}`
  - `retryOnFail: true`
- **Outputs:** none (end of acceptance criteria branch).
- **Edge cases / failures:**
  - Missing `spreadsheetId` from merge result.
  - Column header mismatch (must match exact header strings created earlier).
  - Google API quota/timeout; retry mitigates transient failures.

---

### 2.4 AI test case generation & JSON shaping

**Overview:**  
Uses Gemini (Google Gemini Chat Model) via a LangChain Agent to generate step-level test cases from the user story description and acceptance criteria. Parses the AI output into structured JSON and splits each test-step row into separate items.

**Nodes involved:**
- Google Gemini Chat Model
- AI: Create testcases
- Code
- Split Out

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain LLM connector (`lmChatGoogleGemini`) – provides the language model to the Agent.
- **Configuration choices:**
  - **Model:** `models/gemini-2.5-pro`
- **Credentials:** Google Gemini(PaLM) API account.
- **Connections:** Supplies the **AI language model** connection to **AI: Create testcases**.
- **Edge cases / failures:**
  - API key invalid/quota exceeded.
  - Model name availability depends on Google account/region and node version.

#### Node: AI: Create testcases
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) – prompts the LLM.
- **Configuration choices:**
  - **Prompt text:** Uses Pega response fields:
    - `details.acceptanceCriteria`
    - `details.description`
  - Requires output as a **single raw JSON array**, each object containing:
    `testId, scenario, acceptanceCriteria, preconditions, stepNumber, steps, expectedResult`
  - Instructs carry-over for blank `testId`/`scenario`.
  - System message includes additional constraints (notably: “first step named 'Fold open for the steps' in cursive but not counted as first step”).
- **Outputs:** To **Code** node.
- **Edge cases / failures:**
  - LLM may still return markdown fences or non-JSON despite instruction.
  - Inconsistencies between system message and requested output fields can cause format drift.
  - Large user stories may hit token limits → truncated JSON.

#### Node: Code
- **Type / role:** Code – parses the agent output into JSON.
- **Logic (interpreted):**
  - Reads: `$('AI: Create testcases').first().json.output`
  - Removes markdown fences by:
    - `.replace('```json\n', '')`
    - `.replace('\n```', '')`
  - `JSON.parse` into `jsonData`
  - Returns `{ json: { result: jsonData } }`
- **Outputs:** To **Split Out**
- **Edge cases / failures:**
  - If AI returns raw JSON without fences, this still works.
  - If AI returns ``` (without `json`) or additional whitespace/text, parsing fails.
  - If AI returns an object not an array, later Split Out won’t behave as intended.

#### Node: Split Out
- **Type / role:** Split Out – converts `result` array into one item per row.
- **Configuration choices:**
  - `fieldToSplitOut: result`
  - `include: allOtherFields` (keeps other top-level fields if present)
- **Outputs:** To **Merge: Get SpreadsheetID** (input index 1)
- **Edge cases / failures:**
  - `result` missing/not an array.
  - Very large arrays may slow execution or hit platform limits.

---

### 2.5 Test script insertion & cleanup

**Overview:**  
Combines spreadsheetId with each AI-generated test-step row, appends/updates rows into `Testscript`, then removes redundant repeated columns for consecutive steps of the same test case. It clears rows and re-inserts cleaned data.

**Nodes involved:**
- Merge: Get SpreadsheetID
- Add testcases to the correct columns
- Code to remove redundant data
- Clear Google Spreadsheet
- Add the cleaned testcases back in the Google Spreadsheet

#### Node: Merge: Get SpreadsheetID
- **Type / role:** Merge – combines spreadsheet creation output (spreadsheetId) with split AI rows.
- **Configuration choices:**
  - **Mode:** combineBySql (no explicit query shown)
  - **Inputs:**
    - Input 0: from Create spreadsheet for the testscript
    - Input 1: from Split Out (AI rows)
- **Outputs:** To **Add testcases to the correct columns**
- **Edge cases / failures:**
  - If merge doesn’t properly replicate spreadsheetId across all AI rows, Google Sheets node will fail due to missing `spreadsheetId`.
  - `combineBySql` behavior depends on item fields; if it doesn’t produce a cartesian/positional merge as expected, only one row may get the ID.

#### Node: Add testcases to the correct columns
- **Type / role:** Google Sheets – append/update test rows into `Testscript`.
- **Configuration choices:**
  - **Operation:** `appendOrUpdate`
  - **Sheet:** `Testscript`
  - **DocumentId:** `{{ $json.spreadsheetId }}`
  - **Matching column:** `TestID`
  - **Mapping:** pulls from nested `result.*`:
    - `TestID` ← `result.testId`
    - `Scenario` ← `result.scenario`
    - `Acceptance criteria` ← `result.acceptanceCriteria`
    - `Preconditions` ← `result.preconditions`
    - `Step no.` ← `result.stepNumber`
    - `Steps` ← `result.steps`
    - `Expected result` ← `result.expectedResult`
- **Outputs:** To **Code to remove redundant data**
- **Edge cases / failures:**
  - If `TestID` repeats across multiple steps (intended), `appendOrUpdate` with matching on TestID can overwrite instead of append, depending on how n8n maps updates. This is a structural risk: step-level rows are not uniquely keyed.
  - Header mismatch will cause incorrect column mapping.
  - If `result` is not present (merge changed structure), expressions fail.

#### Node: Code to remove redundant data
- **Type / role:** Code – clears repeated values for consecutive rows with same TestID.
- **Logic (interpreted):**
  - Iterates all rows in current input.
  - Tracks `previousTestID`.
  - If `currentTestID` equals `previousTestID` and not empty, clears:
    - `TestID`, `Scenario`, `Preconditions`, `Acceptance criteria`
  - Keeps `Step no.`, `Steps`, `Expected result` intact.
- **Outputs:** To **Clear Google Spreadsheet**
- **Edge cases / failures:**
  - Assumes incoming items have `item.json.TestID` etc. This depends on what the Google Sheets node outputs (it may output written row data, or metadata, depending on node behavior/version).
  - If rows are not sorted by TestID/step order, cleanup may remove wrong values.

#### Node: Clear Google Spreadsheet
- **Type / role:** Google Sheets – deletes a fixed range of rows to “reset” the sheet before re-inserting cleaned data.
- **Configuration choices:**
  - **Operation:** Clear
  - **Clear:** specificRows
  - **Sheet:** `Testscript`
  - **DocumentId:** `{{ $('Merge: Get SpreadsheetID').item.json.spreadsheetId }}`
  - **StartIndex:** 2 (keeps headers in row 1)
  - **RowsToDelete:** 500
- **Outputs:** To **Add the cleaned testcases back in the Google Spreadsheet**
- **Edge cases / failures:**
  - Hard-coded delete size: if more than 500 rows are generated, leftovers remain.
  - If less than 500 rows exist, Google API typically tolerates it, but behavior can vary.
  - Uses spreadsheetId from Merge node by cross-node reference; if execution branching changes, that reference can break.

#### Node: Add the cleaned testcases back in the Google Spreadsheet
- **Type / role:** Google Sheets – re-inserts cleaned rows.
- **Configuration choices:**
  - **Operation:** `appendOrUpdate`
  - **Sheet:** `Testscript`
  - **DocumentId:** `{{ $('Merge: Get SpreadsheetID').item.json.spreadsheetId }}`
  - **Mapping mode:** autoMapInputData
  - **Matching column:** `TestID`
- **Outputs:** none (final node).
- **Edge cases / failures:**
  - Same “matching by TestID” risk: multiple step rows share a TestID, so updates can overwrite.
  - If cleaned rows intentionally blank TestID for subsequent steps, matching may fail or cause unintended appends.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point (chat) | — | Retrieve the US from Pega Agile Studio |  |
| Retrieve the US from Pega Agile Studio | n8n-nodes-base.httpRequest | Fetch Pega user story via REST API | When chat message received | IterateOverAccCrit; AI: Create testcases; Create spreadsheet for the testscript |  |
| Create spreadsheet for the testscript | n8n-nodes-base.googleSheets | Create Google Spreadsheet + 2 tabs | Retrieve the US from Pega Agile Studio | Add column headers for the first sheet; Add column headers for the second sheet; Merge: Get SpreadsheetID | ## 2. Add column headers for the sheets<br>The sheets do not have column headers, so we will have to add them manually so the AI will know where to insert them later. |
| Add column headers for the first sheet | n8n-nodes-base.httpRequest | Write header row to Acc_Matrix | Create spreadsheet for the testscript | Merge2 | ## 2. Add column headers for the sheets<br>The sheets do not have column headers, so we will have to add them manually so the AI will know where to insert them later. |
| Add column headers for the second sheet | n8n-nodes-base.httpRequest | Write header row to Testscript | Create spreadsheet for the testscript | Merge2 | ## 2. Add column headers for the sheets<br>The sheets do not have column headers, so we will have to add them manually so the AI will know where to insert them later. |
| Merge2 | n8n-nodes-base.merge | Join barrier for both header writes | Add column headers for the first sheet; Add column headers for the second sheet | Merge | ## 2. Add column headers for the sheets<br>The sheets do not have column headers, so we will have to add them manually so the AI will know where to insert them later. |
| IterateOverAccCrit | n8n-nodes-base.splitOut | Split acceptanceCriteria array into items | Retrieve the US from Pega Agile Studio | AddNumbersToAccCrit | ## 1. Add numbers to acc crit<br>Because the acceptance criteria do not have the numbers attached in the data, we have to iterate over them and add them ourselves. |
| AddNumbersToAccCrit | n8n-nodes-base.code | Add sequential AC numbering + reshape | IterateOverAccCrit | Merge | ## 1. Add numbers to acc crit<br>Because the acceptance criteria do not have the numbers attached in the data, we have to iterate over them and add them ourselves. |
| Merge | n8n-nodes-base.merge | Combine acceptance criteria items with spreadsheet context | AddNumbersToAccCrit; Merge2 | Add acceptance criteria to the Google Spreadsheet | ## 1. Add numbers to acc crit<br>Because the acceptance criteria do not have the numbers attached in the data, we have to iterate over them and add them ourselves. |
| Add acceptance criteria to the Google Spreadsheet | n8n-nodes-base.googleSheets | Append AC rows to Acc_Matrix | Merge | — | ## 1. Add numbers to acc crit<br>Because the acceptance criteria do not have the numbers attached in the data, we have to iterate over them and add them ourselves. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider (Gemini) | — | AI: Create testcases (ai_languageModel) |  |
| AI: Create testcases | @n8n/n8n-nodes-langchain.agent | Generate test-step JSON from US + AC | Retrieve the US from Pega Agile Studio; Google Gemini Chat Model | Code |  |
| Code | n8n-nodes-base.code | Parse AI output string into JSON array | AI: Create testcases | Split Out | ## 3. Convert AI testcases to Google Spreadsheets<br>Process the raw AI data into JSON and iterate over it so each testcase is mapped in Google Spreadsheets. |
| Split Out | n8n-nodes-base.splitOut | Split parsed JSON array into step rows | Code | Merge: Get SpreadsheetID | ## 3. Convert AI testcases to Google Spreadsheets<br>Process the raw AI data into JSON and iterate over it so each testcase is mapped in Google Spreadsheets. |
| Merge: Get SpreadsheetID | n8n-nodes-base.merge | Attach spreadsheetId to each AI test-step row | Create spreadsheet for the testscript; Split Out | Add testcases to the correct columns | ## 4. Add testcases and remove duplicate values in rows<br>In these steps we will add the testcases, check which rows have duplicate values, and remove those values. Then we will append the data again in the testsheet. |
| Add testcases to the correct columns | n8n-nodes-base.googleSheets | Append/update AI rows into Testscript | Merge: Get SpreadsheetID | Code to remove redundant data | ## 4. Add testcases and remove duplicate values in rows<br>In these steps we will add the testcases, check which rows have duplicate values, and remove those values. Then we will append the data again in the testsheet. |
| Code to remove redundant data | n8n-nodes-base.code | Blank repeated TestID/Scenario/Preconditions/AC in consecutive rows | Add testcases to the correct columns | Clear Google Spreadsheet | ## 4. Add testcases and remove duplicate values in rows<br>In these steps we will add the testcases, check which rows have duplicate values, and remove those values. Then we will append the data again in the testsheet. |
| Clear Google Spreadsheet | n8n-nodes-base.googleSheets | Delete rows 2..(2+500) from Testscript | Code to remove redundant data | Add the cleaned testcases back in the Google Spreadsheet | ## 4. Add testcases and remove duplicate values in rows<br>In these steps we will add the testcases, check which rows have duplicate values, and remove those values. Then we will append the data again in the testsheet. |
| Add the cleaned testcases back in the Google Spreadsheet | n8n-nodes-base.googleSheets | Re-insert cleaned rows into Testscript | Clear Google Spreadsheet | — | ## 4. Add testcases and remove duplicate values in rows<br>In these steps we will add the testcases, check which rows have duplicate values, and remove those values. Then we will append the data again in the testsheet. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | ## 1. Add numbers to acc crit<br>Because the acceptance criteria do not have the numbers attached in the data, we have to iterate over them and add them ourselves. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | ## Generate Google Spreadsheets Testscript with AI using Pega Agile Studio<br>When working as a functional Pega Software tester, this workflow will create a Google Spreadsheet with acceptance criteria and testcases based on the Pega Agile Studio userstory provided. This improves speed and efficiency while working in sprints on new functionalities.<br><br>**Who's it for**<br>* If you are working as a software tester using the Pega Platform including Pega Agile Studio.<br><br>**How it works**<br>* When the user chats an userstory in the format "US-1234", a HTTP Request will be made to Pega Agile Studio to retrieve the Userstory and commence creating a Google Spreadsheet.<br>* It will add the acceptance criteria on a seperate sheet for traceability.<br>* Next the AI will create testscases based on the Userstory provided.<br>* In the end, a small cleanup will be performed to remove duplicate rows/data created by the AI.<br>* You will have a Google Spreadsheet file in your My Drive containing your testcases!<br><br>**How to set up**<br>* In the Chat, provide the userstory where you want to create a testscript for, in the format "US-1234".<br>* Add you OAuth2 Api for Agile Studio, so you can access the Pega Agile Studio through API calls.<br><br>**Requirements**<br>* Access to Pega Agile Studio OAuth2 Api.<br>* AI API.<br>* Access to Google Cloud for the Google API's |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | ## 2. Add column headers for the sheets<br>The sheets do not have column headers, so we will have to add them manually so the AI will know where to insert them later. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — | ## 4. Add testcases and remove duplicate values in rows<br>In these steps we will add the testcases, check which rows have duplicate values, and remove those values. Then we will append the data again in the testsheet. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment | — | — | ## 3. Convert AI testcases to Google Spreadsheets<br>Process the raw AI data into JSON and iterate over it so each testcase is mapped in Google Spreadsheets. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
1. Add node: **When chat message received** (Chat Trigger / LangChain).
2. Keep default options.
3. This node must output a JSON containing `chatInput`.

2) **Retrieve user story from Pega Agile Studio**
4. Add node: **HTTP Request** named “Retrieve the US from Pega Agile Studio”.
5. Set:
   - Method: `GET` (default)
   - URL: `https://<your-agilestudio-host>/prweb/PRRestService/api/v2/userstories/{{ $json.chatInput }}`
   - Authentication: **OAuth2** (generic credential type `oAuth2Api` in n8n)
6. Create/assign credentials:
   - **OAuth2 API** credential with the token URL, auth URL, client id/secret, and scopes required by Pega Agile Studio REST.
7. Connect: **Chat Trigger → Retrieve the US**.

3) **Create Google Spreadsheet with two sheets**
8. Add node: **Google Sheets** named “Create spreadsheet for the testscript”.
9. Resource: **Spreadsheet**.
10. Operation: **Create** (set title and initial sheets).
11. Title expression example:
    - `{{ $json.details.ID }} - Testscript AI Enhanced - {{ $json.details.name }}`
    (Use the HTTP response path that matches your Pega payload; in the provided workflow it is `details.ID` and `details.name`.)
12. In “Sheets UI” (or equivalent in your n8n version), create two sheets:
    - `Acc_Matrix`
    - `Testscript`
13. Configure Google Sheets OAuth2 credential with Drive/Sheets access.
14. Connect: **Retrieve the US → Create spreadsheet**.

4) **Add headers to both sheets (row 1)**
15. Add node: **HTTP Request** “Add column headers for the first sheet”.
   - Method: `PUT`
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $json.spreadsheetId }}/values/Acc_Matrix!A1:Z1?valueInputOption=RAW`
   - Body type: JSON
   - Body:
     - `{"values":[["Acc. Number","Acc. Description"]]}`
   - Auth: predefined **googleSheetsOAuth2Api**
16. Add node: **HTTP Request** “Add column headers for the second sheet”.
   - Method: `PUT`
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $json.spreadsheetId }}/values/Testscript!A1:Z1?valueInputOption=RAW`
   - Body:
     - `{"values":[["TestID","Scenario","Acceptance criteria","Preconditions","Step no.","Steps","Expected result","Status"]]}`
   - Auth: same Google credential
17. Add node: **Merge** “Merge2” to join both header calls (mode: combineBySql or a deterministic join).
18. Connect:
   - **Create spreadsheet → Header 1**
   - **Create spreadsheet → Header 2**
   - **Header 1 → Merge2 (input 1)**
   - **Header 2 → Merge2 (input 2)**

5) **Process acceptance criteria and append to Acc_Matrix**
19. Add node: **Split Out** “IterateOverAccCrit”
   - Field to split out: `details.acceptanceCriteria`
20. Add node: **Code** “AddNumbersToAccCrit” with logic to output:
   - `Acc.Number` sequentially
   - `Acc.Description` from each acceptance criterion entry
21. Add node: **Merge** “Merge” to combine the numbered AC items with spreadsheet context (so each item has `spreadsheetId`).
22. Add node: **Google Sheets** “Add acceptance criteria to the Google Spreadsheet”
   - Operation: **Append**
   - Sheet: `Acc_Matrix`
   - DocumentId: `{{ $json.spreadsheetId }}`
   - Map columns:
     - `Acc. Number` = `{{ $json.Acc.Number }}`
     - `Acc. Description` = `{{ $json.Acc.Description }}`
23. Connect:
   - **Retrieve the US → IterateOverAccCrit → AddNumbersToAccCrit → Merge (input 1)**
   - **Merge2 → Merge (input 2)**
   - **Merge → Add acceptance criteria**

6) **Configure Gemini model + agent to generate testcases**
24. Add node: **Google Gemini Chat Model**
   - Model: `models/gemini-2.5-pro` (or the closest available)
   - Credential: Google Gemini/PaLM API key
25. Add node: **AI Agent** “AI: Create testcases”
   - Connect the Gemini model to the agent via the AI language model connection.
   - Prompt should embed:
     - `details.acceptanceCriteria`
     - `details.description`
   - Require: output **raw JSON array only** with keys:
     `testId, scenario, acceptanceCriteria, preconditions, stepNumber, steps, expectedResult`
26. Connect: **Retrieve the US → AI: Create testcases**

7) **Parse AI output into JSON + split into rows**
27. Add node: **Code** “Code” to parse the agent output:
   - Read the agent output field (commonly `json.output`)
   - Strip markdown fences if present
   - `JSON.parse` into array
   - Return `{ result: parsedArray }`
28. Add node: **Split Out** “Split Out”
   - Field to split: `result`
29. Connect: **AI Agent → Code → Split Out**

8) **Merge spreadsheetId into each test row and write to Testscript**
30. Add node: **Merge** “Merge: Get SpreadsheetID”
   - Input 1: from **Create spreadsheet** (contains `spreadsheetId`)
   - Input 2: from **Split Out** (AI test rows)
31. Add node: **Google Sheets** “Add testcases to the correct columns”
   - Sheet: `Testscript`
   - DocumentId: `{{ $json.spreadsheetId }}`
   - Operation: append/appendOrUpdate (match your desired behavior)
   - Map columns from `result.*`
32. Connect:
   - **Create spreadsheet → Merge: Get SpreadsheetID (input 1)**
   - **Split Out → Merge: Get SpreadsheetID (input 2)**
   - **Merge: Get SpreadsheetID → Add testcases…**

9) **Cleanup: remove redundant repeated fields, clear old rows, reinsert**
33. Add node: **Code** “Code to remove redundant data” to blank repeated TestID/Scenario/Preconditions/Acceptance criteria for consecutive rows.
34. Add node: **Google Sheets** “Clear Google Spreadsheet”
   - Operation: Clear specific rows on `Testscript`
   - Start at row 2, delete e.g. 500 rows
   - DocumentId from the spreadsheetId reference
35. Add node: **Google Sheets** “Add the cleaned testcases back in the Google Spreadsheet”
   - Operation: append/appendOrUpdate
   - Auto-map cleaned fields back into the sheet
36. Connect:
   - **Add testcases… → Code to remove redundant data → Clear Google Spreadsheet → Add the cleaned testcases back…**

**Credential checklist:**
- Pega Agile Studio: **OAuth2 API** credential (plus base URL reachability).
- Google Sheets: **Google Sheets OAuth2** credential with edit permissions.
- Gemini: **Google Gemini/PaLM API** credential (API key or OAuth depending on your n8n node variant).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Generate Google Spreadsheets Testscript with AI using Pega Agile Studio (purpose, who it’s for, how it works, setup, requirements) | Sticky note in workflow canvas (overview block) |
| “Add numbers to acc crit … iterate over them and add them ourselves.” | Sticky note for acceptance criteria numbering block |
| “Add column headers for the sheets … add them manually so the AI will know where to insert them later.” | Sticky note for sheet header provisioning block |
| “Convert AI testcases to Google Spreadsheets … process raw AI data into JSON and iterate over it …” | Sticky note for AI JSON parsing + iteration block |
| “Add testcases and remove duplicate values in rows … remove those values … append the data again …” | Sticky note for insertion + cleanup block |