Generate AEO snippets from Google PAA with SerpApi and Gemini

https://n8nworkflows.xyz/workflows/generate-aeo-snippets-from-google-paa-with-serpapi-and-gemini-13867


# Generate AEO snippets from Google PAA with SerpApi and Gemini

## 1. Workflow Overview

This workflow, titled **AEO Assistant**, collects a keyword through an n8n form, performs a Google search through **SerpApi**, iterates over the returned search results, asks **Google Gemini** to generate short answer-engine-optimized snippets in a Speakable Schema style, and stores the outputs in **Google Sheets**.

Its practical use case is SEO/AEO research: a user submits a seed keyword, and the workflow generates concise AI answers based on discovered result titles, then logs them in a spreadsheet for editorial or optimization review.

### 1.1 Input Reception
The workflow begins with a form submission where a user provides a keyword.

### 1.2 Search Retrieval
The submitted keyword is sent to SerpApi to fetch Google search results.

### 1.3 Result Expansion
The workflow splits the returned `organic_results` array into individual items so each result can be processed independently.

### 1.4 AI Snippet Generation
Each result title is passed to Gemini with a fixed prompt instructing it to generate a 40–50 word factual answer formatted as a Speakable Schema snippet.

### 1.5 Logging and Storage
Each generated answer is appended as a new row in a Google Sheet along with timestamp, seed keyword, and the source title used as the “question”.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block provides the manual entry point of the workflow. A user submits a single keyword via an n8n-hosted form, which becomes the seed input for the entire downstream process.

**Nodes Involved:**  
- On form submission

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Acts as the workflow trigger and creates the initial execution payload from a submitted web form.
- **Configuration choices:**  
  - Form title is set to **“AEO assistant”**
  - One form field is defined: **Keyword**
- **Key expressions or variables used:**  
  - Outputs `{{$json.Keyword}}`
  - Also provides submission metadata such as `submittedAt`, later used downstream
- **Input and output connections:**  
  - No incoming connection; this is the trigger
  - Outgoing: `Google search`
- **Version-specific requirements:**  
  - Uses **typeVersion 2.5**
  - Requires n8n version supporting Form Trigger v2.x
- **Edge cases or potential failure types:**  
  - Empty keyword submission
  - Form endpoint unavailable if workflow is inactive
  - Manual test vs production webhook URL confusion
- **Sub-workflow reference:**  
  - None

---

## 2.2 Search Retrieval

**Overview:**  
This block sends the submitted keyword to SerpApi to obtain Google search data. Despite the description referencing People Also Ask, the implemented workflow actually uses the `organic_results` array rather than `related_questions`.

**Nodes Involved:**  
- Google search

### Node Details

#### Google search
- **Type and technical role:** `n8n-nodes-serpapi.serpApi`  
  Queries SerpApi for Google Search results.
- **Configuration choices:**  
  - Search query `q` is dynamically set from the form input: `{{$json.Keyword}}`
  - No extra request options or additional fields are configured
  - Defaults imply a generic Google search request
- **Key expressions or variables used:**  
  - `={{ $json.Keyword }}`
- **Input and output connections:**  
  - Input: `On form submission`
  - Output: `Split Out`
- **Version-specific requirements:**  
  - Uses **typeVersion 1**
  - Requires the SerpApi community node/package to be installed in n8n
- **Edge cases or potential failure types:**  
  - Missing or invalid SerpApi credentials
  - Quota exhaustion / rate limiting
  - Search returning no `organic_results`
  - Regional differences in SERP structure
  - Mismatch between expected PAA data and actual configured output
- **Sub-workflow reference:**  
  - None

**Important implementation note:**  
The sticky note says the workflow targets **related_questions (PAA)**, but the actual node chain processes **`organic_results`**. Therefore, the workflow as provided does **not** currently extract Google PAA questions.

---

## 2.3 Result Expansion

**Overview:**  
This block converts the array of organic search results into separate items, allowing downstream AI processing to run once per search result.

**Nodes Involved:**  
- Split Out

### Node Details

#### Split Out
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits one array field into multiple workflow items.
- **Configuration choices:**  
  - Splits the field **`organic_results`**
  - No additional options are configured
- **Key expressions or variables used:**  
  - Field to split: `organic_results`
- **Input and output connections:**  
  - Input: `Google search`
  - Output: `Message a model`
- **Version-specific requirements:**  
  - Uses **typeVersion 1**
- **Edge cases or potential failure types:**  
  - `organic_results` missing or not an array
  - Empty search result set causing no downstream executions
  - If SerpApi schema changes, the split field may fail
- **Sub-workflow reference:**  
  - None

**Implementation discrepancy:**  
The sticky note labels this as “Process relevant results” and mentions cleaning/filtering, but no filtering logic exists. The node only splits the array.

---

## 2.4 AI Snippet Generation

**Overview:**  
This block sends each split result title to Gemini. The model is instructed to act as an AEO specialist and produce a short direct answer formatted as a Speakable Schema snippet.

**Nodes Involved:**  
- Message a model

### Node Details

#### Message a model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Calls Google Gemini in chat/message mode for text generation.
- **Configuration choices:**  
  - Model: **`models/gemini-3-flash-preview`**
  - Messages:
    - A model-role instruction:  
      “You are an AEO Specialist. For the given question, provide a 40-50 word answer that is factual and direct. Format as a Speakable Schema snippet.”
    - User content is populated from the split item title: `{{$json.title}}`
  - No built-in tools enabled
  - No advanced options configured
- **Key expressions or variables used:**  
  - `={{ $json.title }}`
  - Output later consumed via `{{$json.content.parts[0].text}}`
- **Input and output connections:**  
  - Input: `Split Out`
  - Output: `Append row in sheet`
- **Version-specific requirements:**  
  - Uses **typeVersion 1.1**
  - Requires the LangChain-enabled Gemini node available in the current n8n build
  - Requires valid Google AI / Gemini credentials
- **Edge cases or potential failure types:**  
  - Invalid or missing Gemini credentials
  - Model availability issues for preview model IDs
  - Rate limits or token limits
  - Unexpected output format; the node assumes `content.parts[0].text` exists
  - Prompt drift: model may return code blocks, HTML, or schema variants rather than a plain 40–50 word answer
- **Sub-workflow reference:**  
  - None

**Observed behavior from pinned data:**  
The model often returns full JSON-LD blocks, sometimes plus HTML, not just a plain short answer. This means the prompt yields structured schema-like output, and the sheet stores that full generated text.

---

## 2.5 Logging and Storage

**Overview:**  
This block appends each AI-generated output to a Google Sheet. It combines values from the original form submission and the current split/model item.

**Nodes Involved:**  
- Append row in sheet

### Node Details

#### Append row in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends one row per processed item into a selected Google Sheet tab.
- **Configuration choices:**  
  - Operation: **append**
  - Spreadsheet: **AEO Master Log**
  - Sheet/tab: **Client 1** (`gid=0`)
  - Explicit column mapping is used
  - Executes for every incoming item, not once
- **Key expressions or variables used:**  
  - Timestamp: `{{ $('On form submission').item.json.submittedAt }}`
  - Seed Keyword: `{{ $('On form submission').item.json.Keyword }}`
  - PAA Questions: `{{ $('Split Out').item.json.title }}`
  - AI-Generated Answer: `{{ $json.content.parts[0].text }}`
- **Input and output connections:**  
  - Input: `Message a model`
  - No outgoing node
- **Version-specific requirements:**  
  - Uses **typeVersion 4.7**
  - Requires Google Sheets OAuth2 credentials
  - Spreadsheet and target tab must already exist
- **Edge cases or potential failure types:**  
  - Invalid Google credentials
  - Spreadsheet/tab deleted or renamed
  - Column mismatch if sheet schema changes
  - Expression lookup failures if upstream nodes are renamed
  - Large AI output may be truncated by spreadsheet cell or API constraints
- **Sub-workflow reference:**  
  - None

**Important naming note:**  
The column is labeled **“PAA Questions”**, but the actual value written comes from `Split Out.title`, which is the title of an organic search result, not a PAA question.

---

## 2.6 Documentation / Visual Annotation Nodes

**Overview:**  
These nodes do not affect execution. They provide visual guidance, usage notes, requirements, and customization ideas directly on the n8n canvas.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation block for the workflow.
- **Configuration choices:**  
  Contains a long explanatory note covering:
  - Workflow purpose
  - How it works
  - How to use
  - Requirements
  - Customization ideas
- **Input and output connections:**  
  - None
- **Version-specific requirements:**  
  - typeVersion 1
- **Edge cases or potential failure types:**  
  - None at runtime
- **Sub-workflow reference:**  
  - None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: **“## 1. Get the seed keyword/s”**
- **Connections:** None
- **Runtime risks:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: **“## 2. Scrape Google Search results”**
- **Connections:** None
- **Runtime risks:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: **“## 3. Process relevant results”**
- **Connections:** None
- **Runtime risks:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: **“## 4. Get an optimised snippet”**
- **Connections:** None
- **Runtime risks:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: **“## 5. Add to your AEO master log”**
- **Connections:** None
- **Runtime risks:** None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Entry trigger that captures the seed keyword from a form |  | Google search | ## 1. Get the seed keyword/s |
| Google search | n8n-nodes-serpapi.serpApi | Runs a Google search via SerpApi using the submitted keyword | On form submission | Split Out | ## 2. Scrape Google Search results |
| Split Out | n8n-nodes-base.splitOut | Splits the `organic_results` array into individual items | Google search | Message a model | ## 3. Process relevant results |
| Message a model | @n8n/n8n-nodes-langchain.googleGemini | Generates AEO-style snippet text from each result title | Split Out | Append row in sheet | ## 4. Get an optimised snippet |
| Append row in sheet | n8n-nodes-base.googleSheets | Appends final output rows to Google Sheets | Message a model |  | ## 5. Add to your AEO master log |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation and workflow guidance |  |  | # Generate AEO snippets for Google PAA results using SerpApi and AI<br>## How it works<br>- The workflow initiates via a **Form Submission** with a target keyword.<br>- It uses the **SerpApi node** to scrape real-time Google Search results, specifically targeting the **related_questions (PAA)** array.<br>- An **Item Lists node** cleans the data to ensure only high-relevance questions are processed.<br>- The questions are sent to a **Gemini node** with a specialized prompt to draft a 40-50 word answer optimized for "Speakable" Schema and featured snippet logic.<br>- The finalised Q&A pairs are then appended to a **Google Sheet**, creating a ready-to-publish content gap report for the agency's editorial team.<br>## How to use<br>- Replace the keyword placeholder in the HTTP/SerpApi node with your client’s target search term.<br>- Connect your Notion or Google Sheets account to the final destination node to house the results.<br>- Adjust the Schedule Trigger frequency based on how often you perform keyword research for your clients.<br>## Requirements<br>- **SerpApi Account:** To programmatically access Google’s "People Also Ask" data.<br>- **AI Credentials:** An API key for Gemini or OpenAI to generate the response text.<br>- **Destination App:** A Notion workspace, Google Sheet, or Airtable base to store the output.<br>## Customisation<br>- **Competitor Tracking:** Modify the SerpApi parameters to include a specific domain to see what questions your competitors are currently ranking for.<br>- **Schema Generation:** Add a second AI node to automatically wrap the answers in JSON-LD Question and AcceptedAnswer schema code for immediate dev-handoff.<br>- **Multi-lingual AEO:** Add a translation node before the final output to localise answer snippets for international SEO campaigns.<br>- **Slack Notification:** Add a Slack node at the end to notify the account manager to review the new optimisation opportunities.<br>- **Client Organisation:** create a new tab for each client to organise |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual section label |  |  | ## 1. Get the seed keyword/s |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual section label |  |  | ## 2. Scrape Google Search results |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual section label |  |  | ## 3. Process relevant results |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual section label |  |  | ## 4. Get an optimised snippet |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual section label |  |  | ## 5. Add to your AEO master log |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it something like **AEO Assistant**.

2. **Add the trigger node**
   - Add a **Form Trigger** node.
   - Set the node name to **On form submission**.
   - Set the form title to **AEO assistant**.
   - Add one field:
     - Label: **Keyword**
   - Save the form configuration.

3. **Add the SerpApi node**
   - Add a **SerpApi** node.
   - Name it **Google search**.
   - In the query field `q`, use:
     - `{{$json.Keyword}}`
   - Leave request options and additional fields at default unless you need localization/device tuning.
   - Add your **SerpApi credentials**.
   - Connect:
     - `On form submission` → `Google search`

4. **Add the split node**
   - Add a **Split Out** node.
   - Name it **Split Out**.
   - Set **Field To Split Out** to:
     - `organic_results`
   - Connect:
     - `Google search` → `Split Out`

5. **Add the Gemini node**
   - Add a **Google Gemini / Message Model** node from the LangChain-enabled nodes.
   - Name it **Message a model**.
   - Select model:
     - `models/gemini-3-flash-preview`
   - Add messages:
     - First message:
       - Role: model
       - Content:  
         `You are an AEO Specialist. For the given question, provide a 40-50 word answer that is factual and direct. Format as a Speakable Schema snippet.`
     - Second message:
       - Role: user
       - Content:
         `{{$json.title}}`
   - Leave built-in tools disabled.
   - Add your **Gemini / Google AI credentials**.
   - Connect:
     - `Split Out` → `Message a model`

6. **Prepare the destination spreadsheet**
   - In Google Sheets, create a spreadsheet such as **AEO Master Log**.
   - Create a tab such as **Client 1**.
   - Add these columns in the first row:
     - `Timestamp`
     - `Seed Keyword`
     - `PAA Questions`
     - `AI-Generated Answer`

7. **Add the Google Sheets node**
   - Add a **Google Sheets** node.
   - Name it **Append row in sheet**.
   - Set:
     - Operation: **Append**
     - Spreadsheet/document: your target sheet
     - Sheet/tab: **Client 1**
   - Use explicit column mapping:
     - **Timestamp** → `{{ $('On form submission').item.json.submittedAt }}`
     - **Seed Keyword** → `{{ $('On form submission').item.json.Keyword }}`
     - **PAA Questions** → `{{ $('Split Out').item.json.title }}`
     - **AI-Generated Answer** → `{{ $json.content.parts[0].text }}`
   - Add your **Google Sheets OAuth2 credentials**.
   - Connect:
     - `Message a model` → `Append row in sheet`

8. **Add visual sticky notes if desired**
   - Add sticky notes with these contents:
     - `## 1. Get the seed keyword/s`
     - `## 2. Scrape Google Search results`
     - `## 3. Process relevant results`
     - `## 4. Get an optimised snippet`
     - `## 5. Add to your AEO master log`
   - Optionally add the large documentation sticky note with workflow description and customization ideas.

9. **Test the workflow**
   - Activate test mode on the form trigger.
   - Submit a sample keyword such as:
     - `Best CRM for small business 2026`
   - Confirm:
     - SerpApi returns `organic_results`
     - Split Out emits one item per organic result
     - Gemini returns content in `content.parts[0].text`
     - Google Sheets receives one appended row per result

10. **Activate the workflow**
    - Publish/activate the workflow so the form webhook remains live.

### Required credentials
- **SerpApi**
  - Needed by `Google search`
  - Must have API access and available request quota
- **Google Gemini / Google AI**
  - Needed by `Message a model`
  - Must support the chosen model ID
- **Google Sheets OAuth2**
  - Needed by `Append row in sheet`
  - The connected account must have edit access to the spreadsheet

### Important reproduction caveat
If you want the workflow to truly generate snippets from **Google PAA**, change the split node from:
- `organic_results`

to the actual PAA field returned by your SerpApi response, typically something like:
- `related_questions`

You would also need to update the mapped “question” expression accordingly, depending on the exact SerpApi output structure.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow description on the canvas says it targets Google PAA / `related_questions`, but the actual implementation uses `organic_results`. | Implementation discrepancy to review before production use |
| The canvas note mentions an “Item Lists node” for cleaning/filtering, but no such node exists in the workflow. | Documentation discrepancy |
| The workflow stores result titles in a column called “PAA Questions”. | Naming mismatch in Google Sheets mapping |
| The Gemini prompt asks for a 40–50 word answer, but observed outputs often include JSON-LD code blocks and HTML wrappers. | Prompt/output consistency issue |
| The large sticky note suggests possible extensions: competitor tracking, schema generation, multilingual output, Slack notifications, and client-specific tabs. | Built-in customization ideas from the author |