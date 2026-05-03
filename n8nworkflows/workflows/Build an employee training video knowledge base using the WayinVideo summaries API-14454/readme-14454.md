Build an employee training video knowledge base using the WayinVideo summaries API

https://n8nworkflows.xyz/workflows/build-an-employee-training-video-knowledge-base-using-the-wayinvideo-summaries-api-14454


# Build an employee training video knowledge base using the WayinVideo summaries API

# 1. Workflow Overview

This workflow builds a lightweight internal knowledge base from training videos. A user submits a YouTube or other supported training video URL plus a topic/department through an n8n form. The workflow sends that URL to the WayinVideo summaries API, waits for processing, polls for the generated summary and highlights, and finally appends the extracted content into Google Sheets.

Typical use cases:
- Building a searchable spreadsheet of onboarding or training videos
- Standardizing internal learning content summaries
- Reducing manual note-taking for HR, sales, support, or product teams

## 1.1 Input Reception
A Form Trigger node exposes a simple web form where team members provide:
- Training Video URL
- Topic / Department

This is the workflow’s only entry point.

## 1.2 AI Video Submission and Processing
The submitted video URL is posted to the WayinVideo summaries API. The API returns a summary job identifier. The workflow then waits 30 seconds before attempting to fetch the processing result.

## 1.3 Polling and Retry Logic
The workflow checks whether the API response contains non-empty highlights. If highlights exist, processing is considered complete. If not, the workflow loops back to the Wait node and retries indefinitely.

## 1.4 Knowledge Base Storage
Once highlights are available, the workflow formats:
- title
- summary
- highlights
- tags
- original URL
- topic/department

It then appends a new row to a Google Sheet used as the training video knowledge base.

---

# 2. Block-by-Block Analysis

## Block 1 — User Input

### Overview
This block collects the user’s input through a hosted n8n form. It starts the full automation and provides the two core fields used downstream: the video URL and the topic/department classification.

### Nodes Involved
- `1. Form — Video URL + Topic1`

### Node Details

#### 1. Form — Video URL + Topic1
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Public form entry point that starts the workflow when submitted.
- **Configuration choices:**
  - Form title: `🎓 Training Video Knowledge Base`
  - Description explains that AI will extract key points and save them to Google Sheets
  - Two required fields:
    - `Training Video URL`
    - `Topic / Department`
- **Key expressions or variables used:**
  - Downstream nodes reference:
    - `$json['Training Video URL']`
    - `$json['Topic / Department']`
- **Input and output connections:**
  - No input; this is the trigger
  - Output goes to `2. WayinVideo — Submit Summary Request`
- **Version-specific requirements:**
  - Uses Form Trigger node version `2.2`
  - Requires n8n setup that supports hosted forms
- **Edge cases or potential failure types:**
  - Invalid or unsupported video URL entered by the user
  - Form accessible publicly, so spam or malformed submissions are possible unless additional controls are added
  - If the field labels are changed later, downstream expressions will break unless updated
- **Sub-workflow reference:** None

---

## Block 2 — AI Video Submission and Initial Wait

### Overview
This block sends the submitted training video URL to WayinVideo for summarization, then pauses for 30 seconds to allow processing time before polling for results.

### Nodes Involved
- `2. WayinVideo — Submit Summary Request`
- `3. Wait — 30 Seconds`

### Node Details

#### 2. WayinVideo — Submit Summary Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the WayinVideo summaries endpoint to create a summary job.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/summaries`
  - Sends JSON body
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
  - JSON payload:
    - `video_url` from the submitted form
    - `target_lang` set to `en`
- **Key expressions or variables used:**
  - `{{ $json['Training Video URL'] }}`
- **Input and output connections:**
  - Input from `1. Form — Video URL + Topic1`
  - Output to `3. Wait — 30 Seconds`
- **Version-specific requirements:**
  - HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Invalid bearer token or expired API access
  - Unsupported video platform or inaccessible video URL
  - Rate limiting or quota issues from WayinVideo
  - Request body failure if the form field name changes
  - Network timeout or remote API 4xx/5xx responses
- **Sub-workflow reference:** None

#### 3. Wait — 30 Seconds
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays execution before polling the API result.
- **Configuration choices:**
  - Wait duration: `30` seconds
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Input from:
    - `2. WayinVideo — Submit Summary Request`
    - `5. Check — Highlights Ready?` false branch
  - Output to `4. WayinVideo — Fetch Summary Result`
- **Version-specific requirements:**
  - Wait node version `1.1`
- **Edge cases or potential failure types:**
  - Long-running workflows may accumulate executions if used heavily
  - For long videos, 30 seconds may be insufficient and cause repeated polling cycles
  - Wait behavior depends on n8n execution mode and persistence configuration
- **Sub-workflow reference:** None

---

## Block 3 — Result Fetching and Retry Decision

### Overview
This block polls WayinVideo for the summary result and tests whether highlights are available. If the highlights array is not empty, the workflow proceeds to storage. Otherwise, it loops and checks again after another wait.

### Nodes Involved
- `4. WayinVideo — Fetch Summary Result`
- `5. Check — Highlights Ready?`

### Node Details

#### 4. WayinVideo — Fetch Summary Result
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves the generated summary result using the summary job ID returned by the submit request.
- **Configuration choices:**
  - Default method: `GET`
  - URL built dynamically using the submission response job ID:
    - `https://wayinvideo-api.wayin.ai/api/v2/summaries/results/{id}`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{ $('2. WayinVideo — Submit Summary Request').item.json.data.id }}`
- **Input and output connections:**
  - Input from `3. Wait — 30 Seconds`
  - Output to `5. Check — Highlights Ready?`
- **Version-specific requirements:**
  - HTTP Request node version `4.2`
  - The expression depends on item linkage to the original submit node remaining intact
- **Edge cases or potential failure types:**
  - If the submit node did not return `data.id`, this URL expression fails
  - Authentication and version header issues as above
  - Result may not yet exist or may return incomplete data
  - API schema changes could break downstream field extraction
- **Sub-workflow reference:** None

#### 5. Check — Highlights Ready?
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the returned `highlights` array exists and is non-empty.
- **Configuration choices:**
  - Single condition
  - Checks `={{ $json.data.highlights }}`
  - Operator: array `notEmpty`
- **Key expressions or variables used:**
  - `{{ $json.data.highlights }}`
- **Input and output connections:**
  - Input from `4. WayinVideo — Fetch Summary Result`
  - True output to `6. Save — Append Row to Google Sheet`
  - False output back to `3. Wait — 30 Seconds`
- **Version-specific requirements:**
  - IF node version `2.3`
  - Uses conditions format version `3`
- **Edge cases or potential failure types:**
  - If `data` is missing entirely, condition evaluation may not behave as expected
  - Highlights could exist structurally but contain empty or malformed `events`
  - False branch currently retries forever
- **Sub-workflow reference:** None

---

## Block 4 — Save to Google Sheets

### Overview
This block formats the WayinVideo response into spreadsheet-friendly text and appends one new row to the target Google Sheet. It combines data from both the API result and the original form submission.

### Nodes Involved
- `6. Save — Append Row to Google Sheet`

### Node Details

#### 6. Save — Append Row to Google Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row to a Google Sheets document serving as the knowledge base.
- **Configuration choices:**
  - Operation: `append`
  - Sheet selected by URL / document ID placeholder: `YOUR_GOOGLE_SHEET_URL`
  - Sheet/tab reference: `gid=0` with cached name `Sheet1`
  - Manual field mapping using six columns:
    - `Topic/Department`
    - `Video URL`
    - `Video Title`
    - `Key Summary`
    - `Highlights`
    - `Tags`
- **Key expressions or variables used:**
  - Topic/Department:
    - `{{ $('1. Form — Video URL + Topic1').item.json['Topic / Department'] }}`
  - Video URL:
    - `{{ $('1. Form — Video URL + Topic1').item.json['Training Video URL'] }}`
  - Video Title:
    - `{{ $json.data.title }}`
  - Key Summary:
    - `{{ $json.data.summary }}`
  - Highlights:
    - Flattens nested highlight events and formats them as a numbered list
  - Tags:
    - Formats each tag as a bullet line
- **Input and output connections:**
  - Input from true branch of `5. Check — Highlights Ready?`
  - No further downstream node
- **Version-specific requirements:**
  - Google Sheets node version `4.5`
  - Requires valid Google credentials configured in n8n
- **Edge cases or potential failure types:**
  - Spreadsheet URL placeholder not replaced
  - OAuth/credential failure for Google
  - Sheet tab mismatch if `gid=0` is not the intended sheet
  - If `tags` is not an array, `.map()` fails
  - If `highlights` items lack `events` or `desc`, the formatting expression can fail
  - Column names in the target sheet must match the configured schema exactly for predictable mapping
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Sticky — Overview + Setup | Sticky Note | Workspace documentation and setup guidance |  |  | ## 📋 Training Video Knowledge Base Builder  \n### How it works  \nThis workflow turns any YouTube training video into a structured knowledge base entry — automatically. A team member fills out a simple form with the video URL and department or topic name. The workflow sends the video to the WayinVideo API, waits for AI to generate a summary, then checks if highlights are ready. If not ready, it waits and retries. Once confirmed, it saves the title, summary, highlights, and tags to a shared Google Sheet.  \n### Setup  \n1. Replace **YOUR_WAYINVIDEO_API_KEY** in both HTTP Request nodes.  \n2. Connect your Google account in the Save to Google Sheet node.  \n3. Update the Google Sheets URL to your own spreadsheet.  \n4. Your sheet must have columns: Topic/Department, Video URL, Video Title, Key Summary, Highlights, Tags.  \n5. Activate and share the form URL with your team.  \n### Customization tips  \n- Increase wait time (default 30s) for longer videos.  \n- Add a Slack or email node to notify your team when a new entry is saved.  \n- Map {{ $now }} to a Date Added column to auto-log entry dates. |
| Section — User Input | Sticky Note | Visual section label for input block |  |  | ## 📥 Section 1 — User Input  \nTeam member submits the training video URL and department name via a simple web form. This triggers the entire workflow automatically. |
| Section — AI Video Processing | Sticky Note | Visual section label for WayinVideo processing block |  |  | ## 🤖 Section 2 — AI Video Processing  \nVideo URL is sent to WayinVideo API to generate a summary. Workflow waits 30 seconds, then polls the API for the completed result. |
| Section — Retry Check | Sticky Note | Visual section label for polling decision block |  |  | ## 🔁 Section 3 — Retry Check  \nChecks if highlights are ready. If yes, saves to Sheet. If not, loops back and retries automatically. |
| Warning — Infinite Retry Risk | Sticky Note | Operational warning for retry loop |  |  | ## ⚠️ WARNING — Infinite Retry Risk  \nThis node retries indefinitely if WayinVideo never returns highlights. For unsupported or very long videos, the loop may consume many executions. Add a counter or max retry limit before using in production. |
| Section — Save to Google Sheet | Sticky Note | Visual section label for storage block |  |  | ## 💾 Section 4 — Save to Knowledge Base  \nAppends a new row with video title, summary, highlights, tags, topic/department, and original URL into the shared Google Sheets knowledge base. |
| 1. Form — Video URL + Topic1 | Form Trigger | Collects video URL and topic/department from users |  | 2. WayinVideo — Submit Summary Request | ## 📥 Section 1 — User Input  \nTeam member submits the training video URL and department name via a simple web form. This triggers the entire workflow automatically. |
| 2. WayinVideo — Submit Summary Request | HTTP Request | Creates WayinVideo summary job | 1. Form — Video URL + Topic1 | 3. Wait — 30 Seconds | ## 🤖 Section 2 — AI Video Processing  \nVideo URL is sent to WayinVideo API to generate a summary. Workflow waits 30 seconds, then polls the API for the completed result.  \n## 📋 Training Video Knowledge Base Builder  \n### How it works  \nThis workflow turns any YouTube training video into a structured knowledge base entry — automatically. A team member fills out a simple form with the video URL and department or topic name. The workflow sends the video to the WayinVideo API, waits for AI to generate a summary, then checks if highlights are ready. If not ready, it waits and retries. Once confirmed, it saves the title, summary, highlights, and tags to a shared Google Sheet.  \n### Setup  \n1. Replace **YOUR_WAYINVIDEO_API_KEY** in both HTTP Request nodes.  \n2. Connect your Google account in the Save to Google Sheet node.  \n3. Update the Google Sheets URL to your own spreadsheet.  \n4. Your sheet must have columns: Topic/Department, Video URL, Video Title, Key Summary, Highlights, Tags.  \n5. Activate and share the form URL with your team.  \n### Customization tips  \n- Increase wait time (default 30s) for longer videos.  \n- Add a Slack or email node to notify your team when a new entry is saved.  \n- Map {{ $now }} to a Date Added column to auto-log entry dates. |
| 3. Wait — 30 Seconds | Wait | Delays before polling or repolling results | 2. WayinVideo — Submit Summary Request; 5. Check — Highlights Ready? (false) | 4. WayinVideo — Fetch Summary Result | ## 🤖 Section 2 — AI Video Processing  \nVideo URL is sent to WayinVideo API to generate a summary. Workflow waits 30 seconds, then polls the API for the completed result.  \n## ⚠️ WARNING — Infinite Retry Risk  \nThis node retries indefinitely if WayinVideo never returns highlights. For unsupported or very long videos, the loop may consume many executions. Add a counter or max retry limit before using in production. |
| 4. WayinVideo — Fetch Summary Result | HTTP Request | Polls WayinVideo for completed summary content | 3. Wait — 30 Seconds | 5. Check — Highlights Ready? | ## 🤖 Section 2 — AI Video Processing  \nVideo URL is sent to WayinVideo API to generate a summary. Workflow waits 30 seconds, then polls the API for the completed result. |
| 5. Check — Highlights Ready? | IF | Tests whether summary highlights are available yet | 4. WayinVideo — Fetch Summary Result | 6. Save — Append Row to Google Sheet; 3. Wait — 30 Seconds | ## 🔁 Section 3 — Retry Check  \nChecks if highlights are ready. If yes, saves to Sheet. If not, loops back and retries automatically.  \n## ⚠️ WARNING — Infinite Retry Risk  \nThis node retries indefinitely if WayinVideo never returns highlights. For unsupported or very long videos, the loop may consume many executions. Add a counter or max retry limit before using in production. |
| 6. Save — Append Row to Google Sheet | Google Sheets | Stores completed knowledge-base entry in spreadsheet | 5. Check — Highlights Ready? (true) |  | ## 💾 Section 4 — Save to Knowledge Base  \nAppends a new row with video title, summary, highlights, tags, topic/department, and original URL into the shared Google Sheets knowledge base.  \n## 📋 Training Video Knowledge Base Builder  \n### How it works  \nThis workflow turns any YouTube training video into a structured knowledge base entry — automatically. A team member fills out a simple form with the video URL and department or topic name. The workflow sends the video to the WayinVideo API, waits for AI to generate a summary, then checks if highlights are ready. If not ready, it waits and retries. Once confirmed, it saves the title, summary, highlights, and tags to a shared Google Sheet.  \n### Setup  \n1. Replace **YOUR_WAYINVIDEO_API_KEY** in both HTTP Request nodes.  \n2. Connect your Google account in the Save to Google Sheet node.  \n3. Update the Google Sheets URL to your own spreadsheet.  \n4. Your sheet must have columns: Topic/Department, Video URL, Video Title, Key Summary, Highlights, Tags.  \n5. Activate and share the form URL with your team.  \n### Customization tips  \n- Increase wait time (default 30s) for longer videos.  \n- Add a Slack or email node to notify your team when a new entry is saved.  \n- Map {{ $now }} to a Date Added column to auto-log entry dates. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Give it the title: `Build an employee training video knowledge base using the WayinVideo summaries API`.

2. **Add a Form Trigger node**
   - Node type: `Form Trigger`
   - Name it: `1. Form — Video URL + Topic1`
   - Set:
     - **Form Title**: `🎓 Training Video Knowledge Base`
     - **Form Description**: `Submit a training video URL — AI will extract key points and save them automatically to Google Sheets.`
   - Add two required fields:
     1. `Training Video URL`
        - Placeholder: `https://www.youtube.com/watch?v=xxxxxxx`
     2. `Topic / Department`
        - Placeholder: `e.g. HR Onboarding, Sales Training, Product Demo`
   - Save the node.

3. **Add the first HTTP Request node for WayinVideo submission**
   - Node type: `HTTP Request`
   - Name it: `2. WayinVideo — Submit Summary Request`
   - Configure:
     - **Method**: `POST`
     - **URL**: `https://wayinvideo-api.wayin.ai/api/v2/summaries`
     - Enable **Send Headers**
     - Enable **Send Body**
     - **Specify Body**: `JSON`
   - Add headers:
     - `Authorization` → `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` → `v2`
     - `Content-Type` → `application/json`
   - Set JSON body to:
     ```json
     {
       "video_url": "{{ $json['Training Video URL'] }}",
       "target_lang": "en"
     }
     ```
     In n8n this must be entered as an expression-enabled JSON body.
   - Connect `1. Form — Video URL + Topic1` → `2. WayinVideo — Submit Summary Request`.

4. **Add a Wait node**
   - Node type: `Wait`
   - Name it: `3. Wait — 30 Seconds`
   - Set wait amount to `30` seconds.
   - Connect `2. WayinVideo — Submit Summary Request` → `3. Wait — 30 Seconds`.

5. **Add the second HTTP Request node for polling**
   - Node type: `HTTP Request`
   - Name it: `4. WayinVideo — Fetch Summary Result`
   - Configure:
     - **Method**: `GET`
     - **URL** as expression:
       ```text
       https://wayinvideo-api.wayin.ai/api/v2/summaries/results/{{ $('2. WayinVideo — Submit Summary Request').item.json.data.id }}
       ```
     - Enable **Send Headers**
   - Add the same headers:
     - `Authorization` → `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` → `v2`
     - `Content-Type` → `application/json`
   - Connect `3. Wait — 30 Seconds` → `4. WayinVideo — Fetch Summary Result`.

6. **Add an IF node to detect completion**
   - Node type: `IF`
   - Name it: `5. Check — Highlights Ready?`
   - Create one condition:
     - Left value: `{{ $json.data.highlights }}`
     - Operator: `Array` → `is not empty`
   - Connect `4. WayinVideo — Fetch Summary Result` → `5. Check — Highlights Ready?`

7. **Create the retry loop**
   - Connect the **false** output of `5. Check — Highlights Ready?` back to `3. Wait — 30 Seconds`
   - This creates a polling cycle:
     - wait
     - fetch result
     - check highlights
     - if not ready, wait again

8. **Add the Google Sheets node**
   - Node type: `Google Sheets`
   - Name it: `6. Save — Append Row to Google Sheet`
   - Operation: `Append`
   - Connect the **true** output of `5. Check — Highlights Ready?` to this node.

9. **Connect Google credentials**
   - In the Google Sheets node, create or select Google Sheets OAuth2 credentials.
   - Authorize against the Google account that owns or can edit the destination spreadsheet.

10. **Set the destination spreadsheet**
    - In the Google Sheets node:
      - **Document ID**: use your spreadsheet URL
      - **Sheet Name**: select the target sheet/tab, in the original workflow it is `gid=0` / `Sheet1`
    - Ensure the sheet contains these exact columns:
      - `Topic/Department`
      - `Video URL`
      - `Video Title`
      - `Key Summary`
      - `Highlights`
      - `Tags`

11. **Map the Google Sheets columns**
    - Use manual mapping / define-below mode.
    - Configure:
      - **Topic/Department**
        ```text
        {{ $('1. Form — Video URL + Topic1').item.json['Topic / Department'] }}
        ```
      - **Video URL**
        ```text
        {{ $('1. Form — Video URL + Topic1').item.json['Training Video URL'] }}
        ```
      - **Video Title**
        ```text
        {{ $json.data.title }}
        ```
      - **Key Summary**
        ```text
        {{ $json.data.summary }}
        ```
      - **Highlights**
        ```text
        {{ $json.data.highlights
          .flatMap(item => item.events.map(e => e.desc))
          .map((desc, i) => `${i + 1}. ${desc}`)
          .join('\n\n')
        }}
        ```
      - **Tags**
        ```text
        {{ $json.data.tags.map(tag => `• ${tag}`).join('\n') }}
        ```

12. **Replace placeholders**
    - In both HTTP Request nodes, replace `Bearer YOUR_TOKEN_HERE` with your real WayinVideo API bearer token.
    - In the Google Sheets node, replace the placeholder spreadsheet URL with your actual sheet.

13. **Test with a sample video**
    - Submit the form with:
      - a valid supported training video URL
      - a sample topic, such as `HR Onboarding`
    - Confirm:
      - submit request returns a `data.id`
      - fetch request eventually returns `data.summary`, `data.title`, `data.highlights`, and `data.tags`
      - the final row appears in Google Sheets

14. **Activate the workflow**
    - Once testing is successful, activate the workflow.
    - Share the generated form URL with your team.

15. **Recommended production hardening**
    - Add a retry counter before the loop to avoid infinite polling
    - Add error handling branches for HTTP failures
    - Optionally add:
      - Slack notification
      - Email confirmation
      - Date Added column using `{{ $now }}`

### Sub-workflow setup
- This workflow does **not** use any sub-workflow nodes.
- There are no child workflows to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace the WayinVideo API token in both HTTP Request nodes before use. | WayinVideo API authentication |
| Connect a Google account in the Google Sheets node and point it to your own spreadsheet. | Google Sheets integration |
| Required spreadsheet columns: Topic/Department, Video URL, Video Title, Key Summary, Highlights, Tags. | Spreadsheet schema requirement |
| Increase the wait time above 30 seconds for longer videos if results are not ready on the first poll. | Performance tuning |
| The retry loop is currently unbounded and can continue indefinitely if highlights never become available. Add a max retry counter for production. | Reliability / cost control |
| You can add a Slack or email node after the Google Sheets append step to notify the team when a new knowledge base entry is created. | Optional extension |
| You can add a Date Added column and map `{{ $now }}` to log when each row was created. | Optional extension |