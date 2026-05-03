Triage video bug support tickets using WayinVideo and GPT-4o-mini to Google Sheets

https://n8nworkflows.xyz/workflows/triage-video-bug-support-tickets-using-wayinvideo-and-gpt-4o-mini-to-google-sheets-14553


# Triage video bug support tickets using WayinVideo and GPT-4o-mini to Google Sheets

# 1. Workflow Overview

This workflow automatically converts customer bug-report screen recordings into structured support tickets. A support agent submits a recording URL plus basic metadata through an n8n form, WayinVideo analyzes the recording to locate likely bug moments, GPT-4o-mini turns those moments into a formatted bug ticket, and the final result is appended to Google Sheets.

Typical use cases:
- Support teams triaging incoming bug reports faster
- QA teams turning unstructured recordings into standardized tickets
- Product or engineering teams centralizing bug intake in a spreadsheet

## 1.1 Input Reception

The workflow starts with a web form where a support agent provides:
- Screen Recording URL
- Customer Name
- Bug Type

This data becomes the canonical input used by all later nodes.

## 1.2 WayinVideo Bug Moment Submission

The submitted recording is sent to WayinVideo’s `find-moments` API endpoint. The request asks WayinVideo to detect moments relevant to the reported bug type.

## 1.3 Polling for Analysis Results

Because WayinVideo analysis is asynchronous, the workflow waits 45 seconds and then polls the results endpoint using the job ID returned by the first API call.

## 1.4 Conditional Branching

The workflow checks whether WayinVideo has returned any clips in `data.clips`. If clips exist, processing continues to AI ticket generation. If not, the workflow loops back to the 45-second wait and polls again.

## 1.5 AI Ticket Generation

An AI Agent node uses GPT-4o-mini to transform the WayinVideo clip data into a structured support ticket with a fixed format, including summary, priority, category, steps to reproduce, expected/actual behavior, assignee, and suggested fix.

## 1.6 Ticket Storage

The generated ticket and source metadata are appended to a Google Sheet in a tab named `Bug Tickets`.

## 1.7 Embedded Documentation / Warnings

Several sticky notes document setup and important operational risks:
- overall setup instructions
- workflow block descriptions
- infinite loop warning
- hardcoded API key warning

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block provides the workflow entry point. A support agent fills in a hosted n8n form with the recording URL, customer name, and bug type, which become the inputs for the downstream WayinVideo and AI steps.

### Nodes Involved
- `1. Form — Bug Recording + Details`

### Node Details

#### 1. Form — Bug Recording + Details
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that exposes a web form and starts the workflow on submission.
- **Configuration choices:**
  - Form title: `🐛 Support Ticket — Video Bug Triage`
  - Description explains that AI will find the bug moment and create a support ticket automatically
  - Three required fields:
    - `Screen Recording URL`
    - `Customer Name`
    - `Bug Type`
- **Key expressions or variables used:**
  - Downstream nodes reference:
    - `$json['Screen Recording URL']`
    - `$json['Customer Name']`
    - `$json['Bug Type']`
- **Input and output connections:**
  - Input: none, this is the trigger
  - Output: `2. WayinVideo — Find Bug Moments`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.2`, so the instance should support the Form Trigger node in that version range
- **Edge cases or potential failure types:**
  - Form submission may succeed with a syntactically valid but unusable URL
  - Recording links may require authentication and be inaccessible to WayinVideo
  - Users may enter vague `Bug Type` values, reducing detection accuracy
- **Sub-workflow reference:** none

---

## Block 2 — WayinVideo Bug Moment Submission

### Overview
This block sends the submitted screen recording to WayinVideo for AI-based detection of bug-related moments. It creates the asynchronous analysis job and returns an ID used later for polling.

### Nodes Involved
- `2. WayinVideo — Find Bug Moments`

### Node Details

#### 2. WayinVideo — Find Bug Moments
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends an authenticated POST request to WayinVideo’s `find-moments` API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments`
  - JSON request body includes:
    - `video_url` from the form
    - `query` built from the bug type plus `"error or problem moment"`
    - `project_name` as `Bug - <Customer Name>`
    - `limit: 3`
    - `enable_export: false`
    - `target_lang: "en"`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{ $json['Screen Recording URL'] }}`
  - `{{ $json['Bug Type'] }}`
  - `{{ $json['Customer Name'] }}`
- **Input and output connections:**
  - Input: `1. Form — Bug Recording + Details`
  - Output: `3. Wait — 45 Seconds`
- **Version-specific requirements:**
  - Uses `typeVersion: 4.2` of HTTP Request
- **Edge cases or potential failure types:**
  - Invalid or expired WayinVideo token
  - URL unreachable by WayinVideo
  - Unsupported video format
  - Rate limits or upstream API downtime
  - Expression failures if form fields are renamed
- **Sub-workflow reference:** none

---

## Block 3 — Polling for WayinVideo Results

### Overview
This block handles the asynchronous nature of the WayinVideo job. It pauses execution for 45 seconds, then requests the result using the analysis job ID from the previous step.

### Nodes Involved
- `3. Wait — 45 Seconds`
- `4. WayinVideo — Get Bug Moments`

### Node Details

#### 3. Wait — 45 Seconds
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays execution before polling the result endpoint.
- **Configuration choices:**
  - Wait amount: `45` seconds
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input:
    - `2. WayinVideo — Find Bug Moments`
    - false branch from `5. If — Bug Moments Found?`
  - Output: `4. WayinVideo — Get Bug Moments`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Long-running workflow executions consume execution history and system resources
  - Wait-node resume behavior depends on n8n execution mode/persistence configuration
- **Sub-workflow reference:** none

#### 4. WayinVideo — Get Bug Moments
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Polls the WayinVideo results endpoint using the job ID from node 2.
- **Configuration choices:**
  - Method defaults to GET
  - URL is dynamically constructed:
    - `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments/results/<job_id>`
  - Uses same headers as node 2
- **Key expressions or variables used:**
  - `$('2. WayinVideo — Find Bug Moments').item.json.data.id`
- **Input and output connections:**
  - Input: `3. Wait — 45 Seconds`
  - Output: `5. If — Bug Moments Found?`
- **Version-specific requirements:**
  - Uses `typeVersion: 4.2`
- **Edge cases or potential failure types:**
  - If node 2 does not return `data.id`, the URL expression fails
  - Job may still be pending even after 45 seconds
  - Invalid token or version header mismatch
  - API may return a response structure different from expected
- **Sub-workflow reference:** none

---

## Block 4 — Conditional Branching and Retry Loop

### Overview
This block determines whether WayinVideo has found bug moments. If `data.clips` is non-empty, the workflow proceeds; otherwise, it loops back to wait and poll again.

### Nodes Involved
- `5. If — Bug Moments Found?`

### Node Details

#### 5. If — Bug Moments Found?
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether the array of clips returned by WayinVideo is non-empty.
- **Configuration choices:**
  - Condition type: array
  - Operation: `notEmpty`
  - Left value: `{{ $json.data.clips }}`
  - Strict validation enabled in the condition options
- **Key expressions or variables used:**
  - `$json.data.clips`
- **Input and output connections:**
  - Input: `4. WayinVideo — Get Bug Moments`
  - True output: `6. AI Agent — Generate Bug Ticket`
  - False output: `3. Wait — 45 Seconds`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - If the API returns `data` but omits `clips`, strict evaluation may behave unexpectedly depending on payload shape
  - The false branch creates an unbounded loop
  - If the API reports an error state rather than an empty clips array, this node may not distinguish it cleanly
- **Sub-workflow reference:** none

---

## Block 5 — AI Ticket Generation

### Overview
This block transforms the bug moments into a structured support ticket. It uses an AI Agent node with an OpenAI chat model connected as the language model backend.

### Nodes Involved
- `6. AI Agent — Generate Bug Ticket`
- `6a. OpenAI — Chat Model (GPT-4o-mini)`

### Node Details

#### 6. AI Agent — Generate Bug Ticket
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based agent node that sends a prompt to the attached chat model and returns generated text.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Instructional prompt frames the model as a technical support engineer
  - Pulls customer metadata from the form node
  - Pulls clip data from the WayinVideo results node
  - Enforces a strict ticket format with sections:
    - BUG SUMMARY
    - PRIORITY
    - BUG CATEGORY
    - EXACT LOCATION IN VIDEO
    - STEPS TO REPRODUCE
    - EXPECTED BEHAVIOR
    - ACTUAL BEHAVIOR
    - ASSIGN TO
    - SUGGESTED FIX
    - STATUS
- **Key expressions or variables used:**
  - `$('1. Form — Bug Recording + Details').item.json['Customer Name']`
  - `$('1. Form — Bug Recording + Details').item.json['Bug Type']`
  - `$('1. Form — Bug Recording + Details').item.json['Screen Recording URL']`
  - `$('4. WayinVideo — Get Bug Moments').item.json.data.clips.map(...).join('\n')`
  - Inside mapping:
    - `c.idx`
    - `c.title`
    - `c.begin_ms`
    - `c.end_ms`
    - `c.desc`
    - `c.score`
- **Input and output connections:**
  - Main input: true branch from `5. If — Bug Moments Found?`
  - AI language model input: `6a. OpenAI — Chat Model (GPT-4o-mini)`
  - Main output: `7. Google Sheets — Save Bug Ticket`
- **Version-specific requirements:**
  - Uses `typeVersion: 3.1`
  - Requires n8n installation with LangChain/AI nodes enabled
- **Edge cases or potential failure types:**
  - If clips array items lack expected fields, the mapping expression can fail or produce malformed text
  - Model output may not perfectly obey the requested format
  - OpenAI credential or model-access issues can stop execution
  - Token limits may become relevant if clip descriptions are unusually long
- **Sub-workflow reference:** none

#### 6a. OpenAI — Chat Model (GPT-4o-mini)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model used by the AI Agent.
- **Configuration choices:**
  - Model selected: `gpt-4o-mini`
  - No custom built-in tools configured
- **Key expressions or variables used:** none
- **Input and output connections:**
  - No main-data input/output
  - Connected to `6. AI Agent — Generate Bug Ticket` via `ai_languageModel`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.3`
  - Requires valid OpenAI credentials configured in n8n
- **Edge cases or potential failure types:**
  - Invalid API key
  - Missing billing or model access
  - Temporary OpenAI API outages or rate limits
- **Sub-workflow reference:** none

---

## Block 6 — Ticket Storage in Google Sheets

### Overview
This block persists the generated ticket into Google Sheets for downstream support-team use. It stores both the AI-generated ticket and important source metadata from the form and WayinVideo analysis.

### Nodes Involved
- `7. Google Sheets — Save Bug Ticket`

### Node Details

#### 7. Google Sheets — Save Bug Ticket
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends one row to a specified Google Sheet tab.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet document ID: `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Bug Tickets`
  - Mapping mode: manual mapping (`defineBelow`)
  - Columns populated:
    - `Status`: `Open`
    - `Bug Type`: from form
    - `Bug Ticket`: AI output
    - `Customer Name`: from form
    - `Recording URL`: from form
    - `Date Submitted`: `$now`
    - `Bug Moments Found`: number of clips
    - `Top Bug Timestamp`: first clip begin/end converted to seconds
- **Key expressions or variables used:**
  - `$('1. Form — Bug Recording + Details').item.json['Bug Type']`
  - `$json.output`
  - `$('1. Form — Bug Recording + Details').item.json['Customer Name']`
  - `$('1. Form — Bug Recording + Details').item.json['Screen Recording URL']`
  - `$now`
  - `$('4. WayinVideo — Get Bug Moments').item.json.data.clips.length`
  - `$('4. WayinVideo — Get Bug Moments').item.json.data.clips[0].begin_ms`
  - `$('4. WayinVideo — Get Bug Moments').item.json.data.clips[0].end_ms`
- **Input and output connections:**
  - Input: `6. AI Agent — Generate Bug Ticket`
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion: 4.5`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Spreadsheet ID invalid
  - Sheet tab `Bug Tickets` missing
  - OAuth2 token expired or insufficient permissions
  - Column names in the sheet not matching the mapping expectations
  - `clips[0]` access would fail if the prior If condition were bypassed or the payload were malformed
- **Sub-workflow reference:** none

---

## Block 7 — Documentation and Operational Warnings

### Overview
These sticky notes do not process data, but they are important for understanding setup, maintenance, and risks. They explain the workflow purpose, section grouping, and known weaknesses.

### Nodes Involved
- `Main — Overview & Setup`
- `Section — Input`
- `Section — WayinVideo Bug Detection`
- `Section — AI Ticket + Save`
- `Warning — Infinite Loop Risk`
- `Warning — Hardcoded API Key`

### Node Details

#### Main — Overview & Setup
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation node summarizing purpose, setup, and customization options.
- **Configuration choices:**
  - Includes setup instructions for:
    - WayinVideo API key replacement in nodes 2 and 4
    - OpenAI credential setup in node 6a
    - Google Sheet ID replacement in node 7
    - Google OAuth2 setup in node 7
  - Includes customization ideas:
    - Increase WayinVideo limit from 3 to 5
    - Upgrade model from `gpt-4o-mini` to `gpt-4o`
    - Add Slack notification after Google Sheets
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none directly
- **Sub-workflow reference:** none

#### Section — Input
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** labels the input area and explains the form purpose
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Section — WayinVideo Bug Detection
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** labels the WayinVideo submission/polling block
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Section — AI Ticket + Save
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** labels the AI-generation and save block, including note about looping back when results are not ready
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Warning — Infinite Loop Risk
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** warns that the false branch can loop forever if WayinVideo never returns clips
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** highlights runaway executions
- **Sub-workflow reference:** none

#### Warning — Hardcoded API Key
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** warns that the WayinVideo token is hardcoded in plain text and should be moved to credentials
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** secret leakage
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main — Overview & Setup | Sticky Note | Visual documentation and setup guidance |  |  | ## AI Bug Ticket Generator — WayinVideo + GPT-4o-mini + Google Sheets Automatically triage customer bug reports from screen recordings. WayinVideo AI finds the exact bug moment in the video, GPT-4o-mini generates a structured support ticket, and it is saved to Google Sheets.  \n### How it works  \n1. A support agent submits a screen recording URL, customer name, and bug type via the web form.  \n2. WayinVideo scans the recording to find the exact bug moment using AI.  \n3. GPT-4o-mini analyzes the moment and generates a structured ticket (priority, steps to reproduce, fix suggestion).  \n4. The completed ticket is appended to a Google Sheet for the support team.  \n### Setup  \n1. Replace YOUR_WAYINVIDEO_API_KEY in nodes "2. WayinVideo — Find Bug Moments" and "4. WayinVideo — Get Bug Moments".  \n2. Add your OpenAI API key to the credential in node "6a. OpenAI — Chat Model".  \n3. Replace YOUR_GOOGLE_SHEET_ID in node "7. Google Sheets — Save Bug Ticket".  \n4. Connect your Google account in node "7. Google Sheets — Save Bug Ticket" via OAuth2.  \n### Customization tips  \n- Increase limit from 3 to 5 in node 2 to capture more bug moments.  \n- Swap gpt-4o-mini for gpt-4o for higher-quality ticket generation.  \n- Add a Slack node after step 7 to notify the dev team instantly. |
| Section — Input | Sticky Note | Visual label for input section |  |  | ## 1. Input Support agent submits a screen recording URL, customer name, and bug type via a web form. |
| Section — WayinVideo Bug Detection | Sticky Note | Visual label for WayinVideo section |  |  | ## 2. WayinVideo Bug Detection Submits the recording to WayinVideo to find the exact bug moment. Waits 45s then polls for results. |
| Section — AI Ticket + Save | Sticky Note | Visual label for AI and persistence section |  |  | ## 3. AI Ticket Generation + Save If bug moments found, GPT-4o-mini generates a structured ticket and saves it to Google Sheets. If not yet ready, loops back to wait 45s and poll again. |
| Warning — Infinite Loop Risk | Sticky Note | Operational warning about retry loop |  |  | ## ⚠️ WARNING — Infinite Loop Risk If WayinVideo never returns bug moments (invalid URL or unsupported format), this workflow loops forever. Add a max-retry counter to the If node to prevent runaway executions. |
| Warning — Hardcoded API Key | Sticky Note | Security warning about embedded token |  |  | ## ⚠️ WARNING — Hardcoded API Key The WayinVideo API key is stored in plain text inside node parameters. Move it to an n8n Credential (Header Auth) for security before sharing this workflow. |
| 1. Form — Bug Recording + Details | Form Trigger | Collects recording URL, customer name, and bug type |  | 2. WayinVideo — Find Bug Moments | ## 1. Input Support agent submits a screen recording URL, customer name, and bug type via a web form. |
| 2. WayinVideo — Find Bug Moments | HTTP Request | Creates WayinVideo bug-moment analysis job | 1. Form — Bug Recording + Details | 3. Wait — 45 Seconds | ## 2. WayinVideo Bug Detection Submits the recording to WayinVideo to find the exact bug moment. Waits 45s then polls for results.  \n## ⚠️ WARNING — Hardcoded API Key The WayinVideo API key is stored in plain text inside node parameters. Move it to an n8n Credential (Header Auth) for security before sharing this workflow. |
| 3. Wait — 45 Seconds | Wait | Delays before polling WayinVideo result | 2. WayinVideo — Find Bug Moments; 5. If — Bug Moments Found? (false branch) | 4. WayinVideo — Get Bug Moments | ## 2. WayinVideo Bug Detection Submits the recording to WayinVideo to find the exact bug moment. Waits 45s then polls for results. |
| 4. WayinVideo — Get Bug Moments | HTTP Request | Polls WayinVideo for detected clips | 3. Wait — 45 Seconds | 5. If — Bug Moments Found? | ## 2. WayinVideo Bug Detection Submits the recording to WayinVideo to find the exact bug moment. Waits 45s then polls for results.  \n## ⚠️ WARNING — Hardcoded API Key The WayinVideo API key is stored in plain text inside node parameters. Move it to an n8n Credential (Header Auth) for security before sharing this workflow. |
| 5. If — Bug Moments Found? | If | Checks whether WayinVideo returned clips | 4. WayinVideo — Get Bug Moments | 6. AI Agent — Generate Bug Ticket; 3. Wait — 45 Seconds | ## 3. AI Ticket Generation + Save If bug moments found, GPT-4o-mini generates a structured ticket and saves it to Google Sheets. If not yet ready, loops back to wait 45s and poll again.  \n## ⚠️ WARNING — Infinite Loop Risk If WayinVideo never returns bug moments (invalid URL or unsupported format), this workflow loops forever. Add a max-retry counter to the If node to prevent runaway executions. |
| 6. AI Agent — Generate Bug Ticket | AI Agent | Generates structured bug ticket from clips and metadata | 5. If — Bug Moments Found? (true branch); 6a. OpenAI — Chat Model (GPT-4o-mini) via AI model port | 7. Google Sheets — Save Bug Ticket | ## 3. AI Ticket Generation + Save If bug moments found, GPT-4o-mini generates a structured ticket and saves it to Google Sheets. If not yet ready, loops back to wait 45s and poll again. |
| 6a. OpenAI — Chat Model (GPT-4o-mini) | OpenAI Chat Model | Provides LLM backend for AI Agent |  | 6. AI Agent — Generate Bug Ticket | ## 3. AI Ticket Generation + Save If bug moments found, GPT-4o-mini generates a structured ticket and saves it to Google Sheets. If not yet ready, loops back to wait 45s and poll again. |
| 7. Google Sheets — Save Bug Ticket | Google Sheets | Appends generated ticket and metadata to spreadsheet | 6. AI Agent — Generate Bug Ticket |  | ## 3. AI Ticket Generation + Save If bug moments found, GPT-4o-mini generates a structured ticket and saves it to Google Sheets. If not yet ready, loops back to wait 45s and poll again. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node** named `1. Form — Bug Recording + Details`.
   - Set the title to: `🐛 Support Ticket — Video Bug Triage`
   - Set the description to: `Submit a customer bug screen recording URL — AI will find the exact bug moment and create a support ticket automatically.`
   - Add 3 required form fields:
     1. `Screen Recording URL`
        - Placeholder: `https://zoom.us/rec/xxxxxxx or Loom/Drive link`
     2. `Customer Name`
        - Placeholder: `e.g. Rahul Sharma / Acme Corp`
     3. `Bug Type`
        - Placeholder: `e.g. Login Error, Payment Failure, UI Bug, App Crash`

3. **Add an HTTP Request node** named `2. WayinVideo — Find Bug Moments`.
   - Connect it after the form node.
   - Configure:
     - Method: `POST`
     - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments`
     - Body content type: JSON
     - Enable sending headers
   - Add headers:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`
   - Set JSON body to:
     - `video_url` = form field `Screen Recording URL`
     - `query` = form field `Bug Type` plus ` error or problem moment`
     - `project_name` = `Bug - ` plus `Customer Name`
     - `limit` = `3`
     - `enable_export` = `false`
     - `target_lang` = `en`
   - Recommended improvement:
     - Replace the hardcoded bearer token with an n8n credential such as Header Auth or generic HTTP auth storage.

4. **Add a Wait node** named `3. Wait — 45 Seconds`.
   - Connect it after node 2.
   - Set the wait duration to `45` seconds.

5. **Add another HTTP Request node** named `4. WayinVideo — Get Bug Moments`.
   - Connect it after the Wait node.
   - Configure:
     - Method: `GET`
     - URL: dynamic expression pointing to the results endpoint using the ID from node 2:
       - `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments/results/{{ job_id }}`
       - Where `job_id` is taken from `2. WayinVideo — Find Bug Moments` → `data.id`
   - Add the same headers as node 2:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`

6. **Add an If node** named `5. If — Bug Moments Found?`.
   - Connect it after node 4.
   - Configure a condition checking whether the clips array is not empty:
     - Left value: `data.clips`
     - Operator: `Array` → `is not empty`
   - True branch should mean “clips found”.
   - False branch should mean “not ready / no clips yet”.

7. **Connect the false branch of the If node back to the Wait node.**
   - This creates the polling loop:
     - `5. If — Bug Moments Found?` false → `3. Wait — 45 Seconds`

8. **Add an OpenAI Chat Model node** named `6a. OpenAI — Chat Model (GPT-4o-mini)`.
   - This is the model provider for the AI Agent.
   - Select model: `gpt-4o-mini`
   - Attach valid OpenAI credentials in n8n.
   - If your account lacks access to this model, choose another supported OpenAI chat model.

9. **Add an AI Agent node** named `6. AI Agent — Generate Bug Ticket`.
   - Connect the true branch of the If node to this AI Agent.
   - Connect the OpenAI Chat Model node to the AI Agent’s `ai_languageModel` port.
   - Set prompt mode to define the prompt directly in the node.
   - Use a prompt that:
     - frames the AI as a technical support engineer
     - includes:
       - customer name
       - bug type
       - recording URL
       - all bug moments returned by WayinVideo
     - instructs the model to return this exact structure:
       - BUG SUMMARY
       - PRIORITY
       - BUG CATEGORY
       - EXACT LOCATION IN VIDEO
       - STEPS TO REPRODUCE
       - EXPECTED BEHAVIOR
       - ACTUAL BEHAVIOR
       - ASSIGN TO
       - SUGGESTED FIX
       - STATUS
   - Build the WayinVideo moments list by iterating over `data.clips` and formatting:
     - clip index
     - title
     - begin/end in seconds
     - description
     - relevance score

10. **Add a Google Sheets node** named `7. Google Sheets — Save Bug Ticket`.
    - Connect it after the AI Agent.
    - Choose operation: `Append`
    - Authenticate with Google Sheets OAuth2 credentials.
    - Enter your spreadsheet ID in the document field.
    - Set the sheet/tab name to `Bug Tickets`.

11. **Map the Google Sheets columns manually.**
    Create these output columns in the target sheet and map them as follows:
    - `Status` → `Open`
    - `Bug Type` → form field `Bug Type`
    - `Bug Ticket` → AI Agent output text
    - `Customer Name` → form field `Customer Name`
    - `Recording URL` → form field `Screen Recording URL`
    - `Date Submitted` → current date/time
    - `Bug Moments Found` → number of clips returned
    - `Top Bug Timestamp` → first clip start and end in seconds

12. **Prepare the destination Google Sheet.**
    - Create a Google Spreadsheet.
    - Add a worksheet/tab named `Bug Tickets`.
    - Ensure the header row contains the exact column names:
      - `Status`
      - `Bug Type`
      - `Bug Ticket`
      - `Customer Name`
      - `Recording URL`
      - `Date Submitted`
      - `Bug Moments Found`
      - `Top Bug Timestamp`

13. **Add sticky notes for documentation if desired.**
    - One overview note with setup and customization tips
    - One section note for Input
    - One section note for WayinVideo Bug Detection
    - One section note for AI Ticket Generation + Save
    - One warning note for infinite loop risk
    - One warning note for hardcoded API key risk

14. **Configure credentials securely.**
    - **OpenAI:** create or select an OpenAI API credential in n8n for node 6a
    - **Google Sheets:** connect Google OAuth2 in node 7
    - **WayinVideo:** ideally create a reusable credential pattern instead of embedding the bearer token in raw headers

15. **Test the workflow with a reachable video URL.**
    - Submit the form with:
      - a public or otherwise accessible recording URL
      - a customer name
      - a specific bug type
    - Confirm:
      - node 2 returns a job ID in `data.id`
      - node 4 eventually returns clips in `data.clips`
      - node 6 produces formatted ticket text
      - node 7 appends a row to the sheet

16. **Strongly recommended hardening improvements before production use.**
    - Add a retry counter to stop the polling loop after a fixed number of attempts
    - Add explicit error handling for:
      - invalid WayinVideo responses
      - inaccessible URLs
      - empty or malformed clips arrays
      - Google Sheets append failures
    - Optionally add notifications such as Slack or email after successful row creation

### Expected Inputs
- `Screen Recording URL`: publicly accessible or accessible to WayinVideo
- `Customer Name`: free text
- `Bug Type`: short issue category or description

### Expected Outputs
- One generated support ticket in structured plain text
- One appended row in the `Bug Tickets` Google Sheet

### No Sub-Workflow Setup Required
This workflow does **not** invoke any sub-workflows or Execute Workflow nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace the placeholder WayinVideo bearer token in both WayinVideo HTTP nodes before running. | Nodes `2. WayinVideo — Find Bug Moments` and `4. WayinVideo — Get Bug Moments` |
| Replace `YOUR_GOOGLE_SHEET_ID` with the real spreadsheet ID. | Node `7. Google Sheets — Save Bug Ticket` |
| Add OpenAI credentials to the chat model node before testing. | Node `6a. OpenAI — Chat Model (GPT-4o-mini)` |
| The workflow can loop indefinitely if WayinVideo never returns clips. Add a retry limit or timeout guard. | Polling branch around nodes `3`, `4`, and `5` |
| The WayinVideo token is hardcoded in node parameters and should be moved to n8n credentials for safer sharing and maintenance. | Security note |
| Suggested customizations from the embedded notes: increase clip limit from 3 to 5, upgrade to `gpt-4o`, or add Slack after Google Sheets. | Workflow enhancement ideas |