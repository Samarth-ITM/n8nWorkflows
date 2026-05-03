Track competitor YouTube videos using WayinVideo summarization and Gmail

https://n8nworkflows.xyz/workflows/track-competitor-youtube-videos-using-wayinvideo-summarization-and-gmail-14408


# Track competitor YouTube videos using WayinVideo summarization and Gmail

# 1. Workflow Overview

This workflow lets a user submit a competitor’s YouTube video URL through an n8n chat interface, sends that video to the WayinVideo API for AI summarization, waits and polls until the summary is ready, formats the returned analysis into a styled HTML email, and sends the report through Gmail.

Typical use cases:
- Monitoring competitor marketing content
- Quickly extracting insights from YouTube videos without watching them fully
- Sending internal summary reports by email for research or sales intelligence

The workflow has one primary entry point and one retry loop:
- **Entry point:** chat message from the user
- **Loop:** repeated polling every 30 seconds until WayinVideo reports the summary status as `SUCCEEDED`

## 1.1 Input Reception

The workflow starts when a user opens the chat trigger and submits a YouTube URL. The chat trigger provides the URL as `chatInput`.

## 1.2 Video Submission to WayinVideo

The submitted URL is sent to the WayinVideo summarization API using an authenticated HTTP POST request. This creates a summary job and returns a job ID.

## 1.3 Wait and Poll Loop

After the submission, the workflow waits 30 seconds, then queries the WayinVideo results endpoint using the returned job ID. An IF node checks whether the summary job is complete. If not, the workflow loops back to the wait node and retries.

## 1.4 Email Composition

Once the summary is ready, a Code node extracts fields such as title, summary, highlights, events, timestamps, and tags, then generates a branded HTML email body and subject line.

## 1.5 Email Delivery

The HTML report is sent to a fixed destination email address using Gmail OAuth2 credentials.

## 1.6 Documentation / In-Canvas Guidance

Several sticky notes document setup, placeholders, customization points, and the role of each block. These notes are useful when reproducing or maintaining the workflow.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block exposes a public chat endpoint where a user can paste a competitor YouTube video URL. It is the only workflow trigger and supplies the initial `chatInput` consumed by downstream nodes.

### Nodes Involved
- `Chat Message`

### Node Details

#### Chat Message
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Public chat-based trigger that starts the workflow when the user sends a message.
- **Configuration choices:**
  - Public chat enabled
  - Initial prompt explains what the workflow expects and gives an example YouTube URL
- **Key expressions or variables used:**
  - Outputs the user message as `{{$json.chatInput}}`
- **Input and output connections:**
  - **Input:** none, this is the trigger
  - **Output:** `🎬 Submit Video for Summary`
- **Version-specific requirements:**
  - Uses node type version `1.1`
  - Requires an n8n version that supports the LangChain chat trigger node
- **Edge cases or potential failure types:**
  - User sends text that is not a valid YouTube URL
  - User sends multiple URLs or unrelated text
  - Public exposure may lead to spam unless the instance is protected
- **Sub-workflow reference:** none

---

## 2.2 Video Submission to WayinVideo

### Overview
This block takes the URL received from chat and creates a WayinVideo summarization job. It is responsible for sending the initial API request and returning the job identifier used later for polling.

### Nodes Involved
- `🎬 Submit Video for Summary`

### Node Details

#### 🎬 Submit Video for Summary
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to WayinVideo’s summaries endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/summaries`
  - Sends JSON body
  - Custom headers for bearer authentication and API versioning
  - Body fields:
    - `video_url` = chat input
    - `target_lang` = `en`
- **Key expressions or variables used:**
  - `={{ $json.chatInput }}`
- **Input and output connections:**
  - **Input:** `Chat Message`
  - **Output:** `⏳ Wait 30 Seconds`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Invalid or missing API key
  - Invalid YouTube URL
  - WayinVideo API schema changes
  - Rate limiting or timeout
  - Non-JSON API response
  - Unsupported target language code
- **Sub-workflow reference:** none

---

## 2.3 Wait and Poll Loop

### Overview
This block waits for WayinVideo to process the video and repeatedly checks the result endpoint until the job status becomes `SUCCEEDED`. It is the core asynchronous control flow of the workflow.

### Nodes Involved
- `⏳ Wait 30 Seconds`
- `🔄 Poll for Summary Results`
- `✅ Summary Ready?`

### Node Details

#### ⏳ Wait 30 Seconds
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses the workflow for a fixed delay before polling.
- **Configuration choices:**
  - Wait duration: 30 seconds
- **Key expressions or variables used:** none
- **Input and output connections:**
  - **Input:** from `🎬 Submit Video for Summary`, and from the false branch of `✅ Summary Ready?`
  - **Output:** `🔄 Poll for Summary Results`
- **Version-specific requirements:**
  - Uses Wait node version `1`
- **Edge cases or potential failure types:**
  - Very long jobs may cause many loop iterations
  - Instance restarts or execution retention settings may affect long waits
- **Sub-workflow reference:** none

#### 🔄 Poll for Summary Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries the WayinVideo results endpoint using the summary job ID returned by the submit node.
- **Configuration choices:**
  - Method defaults to GET
  - URL built dynamically with the job ID
  - Sends auth and API version headers
- **Key expressions or variables used:**
  - `=https://wayinvideo-api.wayin.ai/api/v2/summaries/results/{{ $('🎬 Submit Video for Summary').item.json.data.id }}`
- **Input and output connections:**
  - **Input:** `⏳ Wait 30 Seconds`
  - **Output:** `✅ Summary Ready?`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - The submit node did not return `data.id`
  - Expression resolution fails if upstream structure differs
  - API returns job-not-found, unauthorized, or transient errors
  - Polling too frequently may hit API rate limits if customized
- **Sub-workflow reference:** none

#### ✅ Summary Ready?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the summary status equals `SUCCEEDED`.
- **Configuration choices:**
  - Strict string equality check
  - Left value: `{{$json.data.status}}`
  - Right value: `SUCCEEDED`
- **Key expressions or variables used:**
  - `={{ $json.data.status }}`
- **Input and output connections:**
  - **Input:** `🔄 Poll for Summary Results`
  - **True output:** `📧 Build Competitor Analysis Email`
  - **False output:** `⏳ Wait 30 Seconds`
- **Version-specific requirements:**
  - Uses IF node version `2.3`
  - Condition engine configured with version `3`
- **Edge cases or potential failure types:**
  - API may return statuses other than `SUCCEEDED`, such as pending, failed, or processing
  - Failed jobs are not handled separately; they will currently keep looping forever unless the API changes or execution is stopped
  - Missing `data.status` may evaluate unexpectedly
- **Sub-workflow reference:** none

**Important design note:**  
This workflow only checks for `SUCCEEDED`. It does **not** explicitly handle terminal failure states such as `FAILED`. If WayinVideo returns a failed status permanently, the workflow will continue to loop every 30 seconds.

---

## 2.4 Email Composition

### Overview
This block transforms the successful WayinVideo response into a human-readable HTML email report. It formats title, summary, hashtags, highlight segments, and nested timestamped events.

### Nodes Involved
- `📧 Build Competitor Analysis Email`

### Node Details

#### 📧 Build Competitor Analysis Email
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes JavaScript to generate the final email subject and HTML message body.
- **Configuration choices:**
  - Reads `data` from the poll response
  - Uses default fallback values when title or summary is missing
  - Builds highlights dynamically
  - Converts millisecond timestamps to `mm:ss`
  - Attempts to recover the source video URL from:
    1. `$('🎬 Submit Video for Summary').item.json.body?.video_url`
    2. `$('Chat Message').item.json.chatInput`
- **Key expressions or variables used:**
  - `const data = $json.data;`
  - `$('🎬 Submit Video for Summary').item.json.body?.video_url`
  - `$('Chat Message').item.json.chatInput`
  - Output:
    - `subject`
    - `body`
- **Input and output connections:**
  - **Input:** true branch of `✅ Summary Ready?`
  - **Output:** `📨 Send Analysis via Gmail`
- **Version-specific requirements:**
  - Uses Code node version `2`
  - Requires JavaScript execution support in the n8n environment
- **Edge cases or potential failure types:**
  - If `data` is absent, the node can fail
  - If WayinVideo changes field names such as `highlights`, `events`, `tags`, `summary`, or `title`, the output may be incomplete
  - HTML is built directly from returned text; if untrusted content is returned, formatting could break
  - The expression trying to read `body.video_url` from the submit node may return nothing, depending on how the HTTP Request node stores request metadata in the execution data
- **Sub-workflow reference:** none

**Output produced by this node:**
- `subject`: `Competitor Analysis: <video title>`
- `body`: complete HTML email document fragment

---

## 2.5 Email Delivery

### Overview
This block sends the generated HTML report to a predefined recipient using Gmail. It is the final delivery step of the workflow.

### Nodes Involved
- `📨 Send Analysis via Gmail`

### Node Details

#### 📨 Send Analysis via Gmail
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email message via Gmail using OAuth2 credentials.
- **Configuration choices:**
  - Fixed recipient address placeholder
  - Subject comes from the Code node output
  - Message body comes from the Code node output
  - Attribution footer disabled
- **Key expressions or variables used:**
  - `={{ $json.subject }}`
  - `={{ $json.body }}`
- **Input and output connections:**
  - **Input:** `📧 Build Competitor Analysis Email`
  - **Output:** none
- **Version-specific requirements:**
  - Uses Gmail node version `2.1`
  - Requires Gmail OAuth2 credentials configured in n8n
- **Edge cases or potential failure types:**
  - Missing or expired Gmail OAuth token
  - Recipient placeholder not replaced
  - Gmail sending limits or account restrictions
  - Some Gmail node configurations may require explicit HTML-capable formatting depending on n8n/Gmail behavior
- **Sub-workflow reference:** none

---

## 2.6 In-Canvas Documentation and Setup Notes

### Overview
These sticky notes do not execute logic, but they explain how the workflow works, where to add credentials, and what each block does. They are important for maintainability and should be preserved when recreating the workflow.

### Nodes Involved
- `📋 Setup Guide`
- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`
- `Sticky Note4`

### Node Details

#### 📋 Setup Guide
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Large setup and usage note for the full workflow.
- **Configuration choices:**
  - Documents purpose, sequence, required credentials, placeholders, customization points, and a processing-time warning
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** sticky note version `1`
- **Edge cases or potential failure types:** none operational
- **Sub-workflow reference:** none

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains the submission block
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains the wait and fetch block
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains the readiness check and retry loop
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains the email building block
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains the Gmail sending block
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Chat Message | `@n8n/n8n-nodes-langchain.chatTrigger` | Public chat entry point that receives the YouTube URL |  | 🎬 Submit Video for Summary | ## 🔍 Competitor Video Analyzer → Email Report<br>**What it does:** User pastes a competitor's YouTube URL in chat. WayinVideo AI generates a full summary with key highlights & tags. The workflow builds a styled HTML report and emails it automatically.<br>**How it works:** 1. User sends a competitor video URL via chat 2. Video is submitted to WayinVideo for AI summarization 3. Waits 30s, then polls until summary is ready 4. If not ready → waits 30s again (auto-retry loop) 5. Builds a styled HTML email with title, summary, highlights & hashtags 6. Sends the report to your email via Gmail<br>**Credentials:** Gmail OAuth2<br>**Placeholders:** `YOUR_WAYINVIDEO_API_KEY`, `YOUR_REPORT_EMAIL@domain.com`<br>API key link: [wayinvideo.com/dashboard/api](https://wayinvideo.com/dashboard/api)<br>Long videos may take more than 30 seconds; workflow auto-retries. |
| 🎬 Submit Video for Summary | `n8n-nodes-base.httpRequest` | Creates the WayinVideo summary job | Chat Message | ⏳ Wait 30 Seconds | ## 🎬 Submit Video for AI Analysis<br>• Takes video URL from user input<br>• Sends it to WayinVideo API<br>• Starts AI summary generation<br>👉 This is where analysis begins |
| ⏳ Wait 30 Seconds | `n8n-nodes-base.wait` | Delays execution before polling and acts as retry target | 🎬 Submit Video for Summary; ✅ Summary Ready? (false) | 🔄 Poll for Summary Results | ## ⏳ Wait & Fetch Results<br>• Waits 30 seconds for processing<br>• Checks API for summary results<br>• Uses job ID to track progress<br>👉 Gives time for AI to process video |
| 🔄 Poll for Summary Results | `n8n-nodes-base.httpRequest` | Retrieves current summary job status and output | ⏳ Wait 30 Seconds | ✅ Summary Ready? | ## ⏳ Wait & Fetch Results<br>• Waits 30 seconds for processing<br>• Checks API for summary results<br>• Uses job ID to track progress<br>👉 Gives time for AI to process video |
| ✅ Summary Ready? | `n8n-nodes-base.if` | Checks whether WayinVideo returned `SUCCEEDED` | 🔄 Poll for Summary Results | 📧 Build Competitor Analysis Email; ⏳ Wait 30 Seconds | ## 🔁 Check if Summary is Ready<br>• Verifies if status = SUCCEEDED<br>• If NOT ready → waits again (loop)<br>• If ready → moves forward<br>👉 Smart retry system until result is ready |
| 📧 Build Competitor Analysis Email | `n8n-nodes-base.code` | Converts summary response into styled HTML email content | ✅ Summary Ready? (true) | 📨 Send Analysis via Gmail | ## 📧 Build Email Report<br>• Extracts title, summary, tags<br>• Creates styled HTML email<br>• Adds highlights with timestamps<br>👉 Converts data into readable report |
| 📨 Send Analysis via Gmail | `n8n-nodes-base.gmail` | Sends the final report email | 📧 Build Competitor Analysis Email |  | ## 📨 Send Email Report<br>• Sends report via Gmail<br>• Uses subject & HTML content<br>• Delivers final analysis to inbox<br>👉 Final output to user |
| 📋 Setup Guide | `n8n-nodes-base.stickyNote` | Global setup and operating instructions |  |  |  |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documents submission block |  |  |  |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documents wait/poll block |  |  |  |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documents decision loop |  |  |  |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documents email composition block |  |  |  |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documents email sending block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## 4.1 Create the trigger

1. **Create a new workflow** in n8n.
2. Add a **Chat Trigger** node.
3. Set the node name to **`Chat Message`**.
4. Enable **Public** chat access.
5. In the initial message, paste:

   `Hi! Send me a competitor YouTube video URL and I will summarize it and email you the key insights.`

   `Example: https://www.youtube.com/watch?v=XXXXXXXXX`

6. Keep default options unless you need access control or custom chat behavior.

## 4.2 Add the submission request

7. Add an **HTTP Request** node after `Chat Message`.
8. Name it **`🎬 Submit Video for Summary`**.
9. Configure:
   - **Method:** `POST`
   - **URL:** `https://wayinvideo-api.wayin.ai/api/v2/summaries`
10. Enable **Send Headers**.
11. Add headers:
   - `Authorization` = `Bearer YOUR_WAYINVIDEO_API_KEY`
   - `Content-Type` = `application/json`
   - `x-wayinvideo-api-version` = `v2`
12. Enable **Send Body**.
13. Add body parameters:
   - `video_url` = `{{ $json.chatInput }}`
   - `target_lang` = `en`
14. Connect `Chat Message` → `🎬 Submit Video for Summary`.

## 4.3 Add the wait node

15. Add a **Wait** node.
16. Name it **`⏳ Wait 30 Seconds`**.
17. Configure:
   - **Unit:** `Seconds`
   - **Amount:** `30`
18. Connect `🎬 Submit Video for Summary` → `⏳ Wait 30 Seconds`.

## 4.4 Add the polling request

19. Add another **HTTP Request** node.
20. Name it **`🔄 Poll for Summary Results`**.
21. Configure:
   - **Method:** GET
   - **URL:**
     `https://wayinvideo-api.wayin.ai/api/v2/summaries/results/{{ $('🎬 Submit Video for Summary').item.json.data.id }}`
22. Enable **Send Headers**.
23. Add the same headers as the submit node:
   - `Authorization` = `Bearer YOUR_WAYINVIDEO_API_KEY`
   - `Content-Type` = `application/json`
   - `x-wayinvideo-api-version` = `v2`
24. Connect `⏳ Wait 30 Seconds` → `🔄 Poll for Summary Results`.

## 4.5 Add the readiness check

25. Add an **IF** node.
26. Name it **`✅ Summary Ready?`**.
27. Configure one condition:
   - Left value: `{{ $json.data.status }}`
   - Operator: **equals**
   - Right value: `SUCCEEDED`
28. Use strict string comparison if your n8n version exposes that option.
29. Connect `🔄 Poll for Summary Results` → `✅ Summary Ready?`.

## 4.6 Create the retry loop

30. Connect the **false** output of `✅ Summary Ready?` back to **`⏳ Wait 30 Seconds`**.
31. This creates the polling loop:
   - wait
   - poll
   - test status
   - repeat if not complete

**Important:**  
This rebuild matches the original design, but you should strongly consider adding another branch for failure statuses such as `FAILED` to avoid endless loops.

## 4.7 Add the email-building code

32. Add a **Code** node.
33. Name it **`📧 Build Competitor Analysis Email`**.
34. Connect the **true** output of `✅ Summary Ready?` to this node.
35. Paste the following logic conceptually:
   - Read `data` from the WayinVideo result
   - Extract:
     - `title`
     - `summary`
     - `tags`
     - `highlights`
   - Build HTML sections:
     - title
     - source URL
     - summary
     - highlight cards
     - hashtags
   - Convert highlight and event timestamps from milliseconds to `mm:ss`
   - Output:
     - `subject`
     - `body`

36. Use these field expectations from the API response:
   - `data.title`
   - `data.summary`
   - `data.tags`
   - `data.highlights`
   - `data.highlights[].start`
   - `data.highlights[].end`
   - `data.highlights[].desc`
   - `data.highlights[].events[]`
   - `data.highlights[].events[].timestamp`
   - `data.highlights[].events[].desc`

37. Include fallback values:
   - title fallback: `Untitled Video`
   - summary fallback: `No summary available.`

38. Preserve or recreate the original source URL lookup logic:
   - first try submit-node request body if available
   - otherwise use `$('Chat Message').item.json.chatInput`

39. Output a single item shaped like:
   - `subject`: `Competitor Analysis: <title>`
   - `body`: `<html-like string>`

## 4.8 Add Gmail delivery

40. Add a **Gmail** node.
41. Name it **`📨 Send Analysis via Gmail`**.
42. Connect `📧 Build Competitor Analysis Email` → `📨 Send Analysis via Gmail`.
43. Configure Gmail credentials using **OAuth2**.
44. Set:
   - **To:** `YOUR_REPORT_EMAIL@domain.com`
   - **Subject:** `{{ $json.subject }}`
   - **Message:** `{{ $json.body }}`
45. Disable attribution append if your node exposes that option:
   - `appendAttribution = false`

## 4.9 Add the sticky notes

46. Add a large sticky note named **`📋 Setup Guide`** and include:
   - workflow purpose
   - how it works
   - credential requirement
   - placeholders:
     - `YOUR_WAYINVIDEO_API_KEY`
     - `YOUR_REPORT_EMAIL@domain.com`
   - API key link: [wayinvideo.com/dashboard/api](https://wayinvideo.com/dashboard/api)
   - customization notes for `target_lang` and email HTML
   - warning that long videos may take more than 30 seconds

47. Add a sticky note near the submit node:
   - Title: **🎬 Submit Video for AI Analysis**
   - Explain that it takes the user URL and starts AI summary generation

48. Add a sticky note near the wait/poll nodes:
   - Title: **⏳ Wait & Fetch Results**
   - Explain 30-second delay and job ID tracking

49. Add a sticky note near the IF node:
   - Title: **🔁 Check if Summary is Ready**
   - Explain `SUCCEEDED` check and retry loop

50. Add a sticky note near the Code node:
   - Title: **📧 Build Email Report**
   - Explain extraction of title, summary, tags, highlights

51. Add a sticky note near the Gmail node:
   - Title: **📨 Send Email Report**
   - Explain email delivery

## 4.10 Configure credentials and placeholders

52. Replace `YOUR_WAYINVIDEO_API_KEY` in both HTTP Request nodes.
53. Replace `YOUR_REPORT_EMAIL@domain.com` in the Gmail node.
54. Connect Gmail OAuth2 credentials to the Gmail node.
55. Test with a valid public YouTube URL.

## 4.11 Recommended hardening improvements

56. Add input validation before submission:
   - confirm message contains a YouTube URL
   - reject empty or malformed input
57. Add a second IF or Switch node for terminal statuses such as:
   - `FAILED`
   - `ERROR`
58. Add a maximum retry counter to prevent infinite polling.
59. Move the WayinVideo API key into n8n credentials or environment variables instead of hardcoding it in headers.
60. Consider making the recipient email dynamic if multiple users will use the workflow.

## 4.12 Sub-workflow setup

This workflow does **not** invoke any sub-workflow and is not itself documented as a callable sub-workflow. No Execute Workflow node is present.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| WayinVideo API key retrieval location | [https://wayinvideo.com/dashboard/api](https://wayinvideo.com/dashboard/api) |
| Gmail credential required | OAuth2 credential in n8n Gmail node |
| Customizable summary language | `target_lang` in `🎬 Submit Video for Summary` |
| Customizable report branding | HTML template inside `📧 Build Competitor Analysis Email` |
| Operational warning | Long videos may require more than 30 seconds; the workflow retries automatically |
| Main limitation | No explicit handling for failed WayinVideo jobs; current logic can loop indefinitely unless modified |

