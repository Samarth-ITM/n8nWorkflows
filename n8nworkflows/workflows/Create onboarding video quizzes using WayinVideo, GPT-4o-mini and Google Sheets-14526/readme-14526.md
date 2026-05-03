Create onboarding video quizzes using WayinVideo, GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/create-onboarding-video-quizzes-using-wayinvideo--gpt-4o-mini-and-google-sheets-14526


# Create onboarding video quizzes using WayinVideo, GPT-4o-mini and Google Sheets

# 1. Workflow Overview

This workflow generates onboarding quiz questions from a video URL submitted through an n8n form. It sends the video to WayinVideo for transcription, waits and polls until the transcript is ready, then passes the transcript to an OpenAI chat model through an AI Agent node to generate multiple-choice questions, and finally stores the result in Google Sheets.

Typical use cases:
- HR onboarding knowledge checks
- Internal training quiz generation
- Department-specific compliance or process learning content
- Rapid conversion of recorded training videos into structured question banks

The workflow is organized into the following logical blocks.

## 1.1 Input Reception

The workflow starts with an n8n form that collects:
- the onboarding video URL
- the department/topic
- the requested number of questions

This block is the entry point and provides all user-defined variables used later.

## 1.2 Transcript Submission and Retrieval Loop

The video URL is submitted to the WayinVideo transcription API. The workflow then waits 45 seconds and requests the transcript result using the transcript job ID returned by the submission request.

This block handles asynchronous transcription, which is necessary because transcript generation is not immediate.

## 1.3 Transcript Status Check

An IF node is intended to verify whether the transcript is complete. If complete, the workflow proceeds to quiz generation. If not, it loops back to the wait node and polls again.

However, in the current JSON, this IF node is misconfigured with an empty condition. As a result, it will always follow the false branch and create an infinite polling loop unless fixed.

## 1.4 AI Quiz Generation

Once the transcript is ready, the AI Agent node formats the transcript and prompts GPT-4o-mini to generate multiple-choice questions.

The prompt enforces:
- transcript-only grounding
- a fixed MCQ structure
- a specific count of generated questions
- answer keys and references

## 1.5 Result Storage

The generated quiz text, together with metadata such as department, video URL, question count, and generation timestamp, is appended to a Google Sheet tab named `Quiz Bank`.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Input Reception

### Overview
This block collects the user input that drives the rest of the workflow. It acts as the only entry point and provides the video URL, department/topic, and requested number of questions.

### Nodes Involved
- `1. Form — Video URL + Topic`

### Node Details

#### 1. Form — Video URL + Topic
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Webhook-based form entry node that starts the workflow when a user submits the form.
- **Configuration choices:**
  - Form title: `🎓 Onboarding Video Quiz Builder`
  - Description is written in Hindi/Hinglish and explains that the user pastes a video URL and AI generates MCQ quiz questions saved to Google Sheets.
  - Three required fields:
    - `Onboarding Video URL`
    - `Department / Topic`
    - `Number of Questions`
- **Key expressions or variables used downstream:**
  - `{{$json['Onboarding Video URL']}}`
  - `{{$json['Department / Topic']}}`
  - `{{$json['Number of Questions']}}`
- **Input and output connections:**
  - No input node; this is the workflow trigger.
  - Output goes to `2. WayinVideo — Submit Transcript`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.2`
  - Requires n8n version supporting Form Trigger v2.x
- **Edge cases or potential failure types:**
  - Invalid or inaccessible video URL may later break transcription submission
  - `Number of Questions` is collected as free text, so non-numeric values are possible
  - No validation is applied for URL pattern or numeric range
- **Sub-workflow reference:** None

---

## 2.2 Block: Transcript Submission and Retrieval Loop

### Overview
This block submits the video to WayinVideo for transcription, waits for processing time, and then queries the transcription result endpoint using the returned transcript job ID.

### Nodes Involved
- `2. WayinVideo — Submit Transcript`
- `3. Wait — 45 Seconds`
- `4. WayinVideo — Get Transcript`

### Node Details

#### 2. WayinVideo — Submit Transcript
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to create a transcription job in WayinVideo.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/transcripts`
  - JSON body includes:
    - `video_url` from form input
    - `target_lang: "en"`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{$json['Onboarding Video URL']}}`
- **Input and output connections:**
  - Input from `1. Form — Video URL + Topic`
  - Output to `3. Wait — 45 Seconds`
- **Version-specific requirements:**
  - Uses `typeVersion: 4.2`
  - Requires HTTP Request node version supporting JSON body configuration as shown
- **Edge cases or potential failure types:**
  - Invalid API token returns auth failures
  - Unsupported or unreachable video URL may cause API validation errors
  - Rate limiting or server-side API errors
  - If response shape changes and `data.id` is absent, downstream transcript polling breaks
- **Sub-workflow reference:** None

#### 3. Wait — 45 Seconds
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution to give the asynchronous transcript job time to process.
- **Configuration choices:**
  - Wait amount: `45` seconds
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from:
    - `2. WayinVideo — Submit Transcript`
    - false branch of `5. If — Transcript Status Check`
  - Output to `4. WayinVideo — Get Transcript`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Wait may be too short for long videos, causing repeated polling
  - Excessive polling can increase execution duration
  - If IF logic is incorrect, this node becomes part of an infinite loop
- **Sub-workflow reference:** None

#### 4. WayinVideo — Get Transcript
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the transcription result using the transcript job ID returned by the submit step.
- **Configuration choices:**
  - Method defaults to GET
  - URL dynamically references the transcript ID:
    - `https://wayinvideo-api.wayin.ai/api/v2/transcripts/results/{{ $('2. WayinVideo — Submit Transcript').item.json.data.id }}`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{ $('2. WayinVideo — Submit Transcript').item.json.data.id }}`
- **Input and output connections:**
  - Input from `3. Wait — 45 Seconds`
  - Output to `5. If — Transcript Status Check`
- **Version-specific requirements:**
  - Uses `typeVersion: 4.2`
- **Edge cases or potential failure types:**
  - If submit step did not return `data.id`, this URL expression fails
  - Auth and API version header issues
  - Result may not yet contain transcript data if processing is incomplete
  - Response structure may differ depending on job status
- **Sub-workflow reference:** None

---

## 2.3 Block: Transcript Status Check

### Overview
This block is supposed to determine whether the transcription process has completed. If successful, execution continues to AI generation; otherwise it returns to the wait node and polls again.

### Nodes Involved
- `5. If — Transcript Status Check`

### Node Details

#### 5. If — Transcript Status Check
- **Type and technical role:** `n8n-nodes-base.if`  
  Branching node intended to test transcript job status.
- **Configuration choices:**
  - The node currently has an empty condition:
    - empty `leftValue`
    - equals operator
    - empty `rightValue`
  - A sticky note explicitly warns this is broken and should check `{{$json.data.status}} == SUCCEEDED`
- **Key expressions or variables used:**
  - Intended: `{{$json.data.status}}`
  - Intended comparison value: `SUCCEEDED`
- **Input and output connections:**
  - Input from `4. WayinVideo — Get Transcript`
  - True output goes to `6. AI — Generate MCQ Quiz`
  - False output goes to `3. Wait — 45 Seconds`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - **Current critical issue:** infinite loop risk because the condition is empty
  - If `data.status` is missing, strict validation may fail or branch unexpectedly
  - Depending on WayinVideo response values, `SUCCEEDED` may need exact case matching
  - Long-running transcripts may cause execution timeout depending on n8n environment
- **Sub-workflow reference:** None

**Required fix:**  
Configure the IF node to evaluate whether:
- left value: `{{$json.data.status}}`
- operator: equals
- right value: `SUCCEEDED`

---

## 2.4 Block: AI Quiz Generation

### Overview
This block takes the completed transcript and the user’s requested question count, then prompts an OpenAI chat model through the LangChain Agent node to generate MCQ quiz content.

### Nodes Involved
- `6. AI — Generate MCQ Quiz`
- `OpenAI — Chat Model`

### Node Details

#### 6. AI — Generate MCQ Quiz
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent node that constructs a prompt and sends it to the connected language model.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - The prompt instructs the model to:
    - generate exactly the requested number of MCQs
    - use only the transcript
    - produce 4 answer choices per question
    - include correct answer and transcript reference
  - The transcript is flattened into a readable text block using:
    - speaker label
    - approximate start time in seconds
    - utterance text
- **Key expressions or variables used:**
  - Requested count:
    - `{{ $('1. Form — Video URL + Topic').item.json['Number of Questions'] }}`
  - Department:
    - `{{ $('1. Form — Video URL + Topic').item.json['Department / Topic'] }}`
  - Transcript formatting:
    - `{{ $('4. WayinVideo — Get Transcript').item.json.data.transcript.map(s => '[' + s.speaker + ' | ' + Math.floor(s.start/1000) + 's] ' + s.text).join('\n') }}`
- **Input and output connections:**
  - Main input from true branch of `5. If — Transcript Status Check`
  - AI language model input from `OpenAI — Chat Model`
  - Main output to `7. Google Sheets — Save Quiz Bank`
- **Version-specific requirements:**
  - Uses `typeVersion: 3.1`
  - Requires n8n with LangChain AI nodes installed/enabled
- **Edge cases or potential failure types:**
  - Prompt contains contradictory instructions:
    - says “Create exactly {{Number of Questions}}”
    - but also says “No more, no less — always 10 questions”
    - and final line says `TOTAL QUESTIONS GENERATED: 10`
  - If user requests a number other than 10, the model may produce inconsistent output
  - If transcript array is missing or empty, the `.map(...)` expression can fail
  - Very large transcripts can exceed model context window or cause latency/cost increases
  - Since output is free-form text, formatting may drift
- **Sub-workflow reference:** None

**Prompt quality note:**  
The prompt should be corrected to remove hardcoded `10` if variable counts are intended.

#### OpenAI — Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the OpenAI chat model used by the AI Agent node.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - No additional options configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No main input
  - Connected to `6. AI — Generate MCQ Quiz` via `ai_languageModel`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.3`
  - Requires valid OpenAI credentials in n8n
- **Edge cases or potential failure types:**
  - Missing or invalid OpenAI credentials
  - Model availability differences across OpenAI accounts
  - Rate limiting or token quota exhaustion
  - Large transcript prompt may exceed token limits
- **Sub-workflow reference:** None

---

## 2.5 Block: Result Storage

### Overview
This block appends the generated quiz content and associated metadata to a Google Sheet. It acts as the final persistence layer for the workflow.

### Nodes Involved
- `7. Google Sheets — Save Quiz Bank`

### Node Details

#### 7. Google Sheets — Save Quiz Bank
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends one row per workflow run into a target Google Sheet tab.
- **Configuration choices:**
  - Operation: `append`
  - Document ID: placeholder `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Quiz Bank`
  - Column mapping:
    - `Video URL` from form input
    - `Department` from form input
    - `Generated On` from `$now`
    - `Quiz Questions` from AI output
    - `Total Questions` from form input
- **Key expressions or variables used:**
  - `{{ $('1. Form — Video URL + Topic').item.json['Onboarding Video URL'] }}`
  - `{{ $('1. Form — Video URL + Topic').item.json['Department / Topic'] }}`
  - `{{ $now }}`
  - `{{ $json.choices[0].message.content }}`
  - `{{ $('1. Form — Video URL + Topic').item.json['Number of Questions'] }}`
- **Input and output connections:**
  - Input from `6. AI — Generate MCQ Quiz`
  - No output node
- **Version-specific requirements:**
  - Uses `typeVersion: 4.5`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Invalid or missing Google credentials
  - Placeholder sheet ID must be replaced
  - Tab `Quiz Bank` must exist
  - Column names must match expected headers for reliable mapping
  - The expression `{{$json.choices[0].message.content}}` assumes the AI Agent output has this structure; depending on node version/output mode, this may not match actual output and could save blank data or fail
- **Sub-workflow reference:** None

**Compatibility note:**  
The AI Agent node often returns a direct text field rather than raw OpenAI `choices[0].message.content`. This mapping should be verified in execution data and adjusted if necessary.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 1. Form — Video URL + Topic | n8n-nodes-base.formTrigger | Collects video URL, department/topic, and requested question count |  | 2. WayinVideo — Submit Transcript | ## 📋 Step 1 — User Input<br>Form collects video URL, department topic, and desired question count. |
| 2. WayinVideo — Submit Transcript | n8n-nodes-base.httpRequest | Creates a WayinVideo transcription job | 1. Form — Video URL + Topic | 3. Wait — 45 Seconds | ## 🎬 Steps 2–4 — Transcription<br>Submits video to WayinVideo, waits 45s, then fetches the completed transcript.<br><br>## 🎓 Onboarding Video Quiz Builder<br><br>### How it works<br>User submits a training video URL via form. The workflow sends the video to WayinVideo for transcription, waits 45 seconds, then polls until the transcript is ready. Once ready, an AI agent (GPT) reads the full transcript and generates the exact number of MCQ questions requested. The final quiz is saved to a Google Sheet.<br><br>### Setup<br>1. Add your WayinVideo API key — replace `YOUR_WAYINVIDEO_API_KEY` in nodes 2 and 4<br>2. Add your OpenAI API key in the `OpenAI — Chat Model` node<br>3. Connect Google Sheets OAuth2 credentials in node 7<br>4. Replace `YOUR_GOOGLE_SHEET_ID` in node 7 with your actual Sheet ID<br>5. Ensure your Google Sheet has a tab named **Quiz Bank** with columns: Department, Video URL, Total Questions, Quiz Questions, Generated On<br><br>### Customization tips<br>- Swap `gpt-4o-mini` for a stronger model in the OpenAI node for higher quality questions<br>- Increase the Wait time (node 3) if transcription fails on longer videos<br>- Edit the AI prompt in node 6 to change question format, language, or difficulty level |
| 3. Wait — 45 Seconds | n8n-nodes-base.wait | Delays execution before polling or re-polling transcript status | 2. WayinVideo — Submit Transcript; 5. If — Transcript Status Check (false) | 4. WayinVideo — Get Transcript | ## 🎬 Steps 2–4 — Transcription<br>Submits video to WayinVideo, waits 45s, then fetches the completed transcript.<br><br>## ⚠️ WARNING — Empty IF Condition (Infinite Loop Risk)<br>The IF node has no condition configured. As-is, it will ALWAYS route to the false branch and loop back to the Wait node indefinitely.<br><br>**Fix:** Set the condition to check `{{ $json.data.status }}` equals `SUCCEEDED` so the true branch runs the AI agent only when transcription is complete. |
| 4. WayinVideo — Get Transcript | n8n-nodes-base.httpRequest | Polls WayinVideo for transcript result using transcript job ID | 3. Wait — 45 Seconds | 5. If — Transcript Status Check | ## 🎬 Steps 2–4 — Transcription<br>Submits video to WayinVideo, waits 45s, then fetches the completed transcript.<br><br>## ⚠️ WARNING — Empty IF Condition (Infinite Loop Risk)<br>The IF node has no condition configured. As-is, it will ALWAYS route to the false branch and loop back to the Wait node indefinitely.<br><br>**Fix:** Set the condition to check `{{ $json.data.status }}` equals `SUCCEEDED` so the true branch runs the AI agent only when transcription is complete. |
| 5. If — Transcript Status Check | n8n-nodes-base.if | Branches based on transcript readiness | 4. WayinVideo — Get Transcript | 6. AI — Generate MCQ Quiz (true); 3. Wait — 45 Seconds (false) | ## ✅ Step 5 — Status Check<br>Checks if transcript is ready. If not ready, loops back to wait.<br><br>## ⚠️ WARNING — Empty IF Condition (Infinite Loop Risk)<br>The IF node has no condition configured. As-is, it will ALWAYS route to the false branch and loop back to the Wait node indefinitely.<br><br>**Fix:** Set the condition to check `{{ $json.data.status }}` equals `SUCCEEDED` so the true branch runs the AI agent only when transcription is complete. |
| 6. AI — Generate MCQ Quiz | @n8n/n8n-nodes-langchain.agent | Generates MCQ quiz content from transcript | 5. If — Transcript Status Check (true); OpenAI — Chat Model (AI model input) | 7. Google Sheets — Save Quiz Bank | ## 🤖 Step 6 — AI Quiz Generation<br>GPT reads the full transcript and generates the exact number of MCQ questions with answers and references.<br><br>## 🎓 Onboarding Video Quiz Builder<br><br>### How it works<br>User submits a training video URL via form. The workflow sends the video to WayinVideo for transcription, waits 45 seconds, then polls until the transcript is ready. Once ready, an AI agent (GPT) reads the full transcript and generates the exact number of MCQ questions requested. The final quiz is saved to a Google Sheet.<br><br>### Setup<br>1. Add your WayinVideo API key — replace `YOUR_WAYINVIDEO_API_KEY` in nodes 2 and 4<br>2. Add your OpenAI API key in the `OpenAI — Chat Model` node<br>3. Connect Google Sheets OAuth2 credentials in node 7<br>4. Replace `YOUR_GOOGLE_SHEET_ID` in node 7 with your actual Sheet ID<br>5. Ensure your Google Sheet has a tab named **Quiz Bank** with columns: Department, Video URL, Total Questions, Quiz Questions, Generated On<br><br>### Customization tips<br>- Swap `gpt-4o-mini` for a stronger model in the OpenAI node for higher quality questions<br>- Increase the Wait time (node 3) if transcription fails on longer videos<br>- Edit the AI prompt in node 6 to change question format, language, or difficulty level |
| OpenAI — Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the OpenAI model used by the AI Agent |  | 6. AI — Generate MCQ Quiz | ## 🤖 Step 6 — AI Quiz Generation<br>GPT reads the full transcript and generates the exact number of MCQ questions with answers and references.<br><br>## 🎓 Onboarding Video Quiz Builder<br><br>### How it works<br>User submits a training video URL via form. The workflow sends the video to WayinVideo for transcription, waits 45 seconds, then polls until the transcript is ready. Once ready, an AI agent (GPT) reads the full transcript and generates the exact number of MCQ questions requested. The final quiz is saved to a Google Sheet.<br><br>### Setup<br>1. Add your WayinVideo API key — replace `YOUR_WAYINVIDEO_API_KEY` in nodes 2 and 4<br>2. Add your OpenAI API key in the `OpenAI — Chat Model` node<br>3. Connect Google Sheets OAuth2 credentials in node 7<br>4. Replace `YOUR_GOOGLE_SHEET_ID` in node 7 with your actual Sheet ID<br>5. Ensure your Google Sheet has a tab named **Quiz Bank** with columns: Department, Video URL, Total Questions, Quiz Questions, Generated On<br><br>### Customization tips<br>- Swap `gpt-4o-mini` for a stronger model in the OpenAI node for higher quality questions<br>- Increase the Wait time (node 3) if transcription fails on longer videos<br>- Edit the AI prompt in node 6 to change question format, language, or difficulty level |
| 7. Google Sheets — Save Quiz Bank | n8n-nodes-base.googleSheets | Appends quiz output and metadata to Google Sheets | 6. AI — Generate MCQ Quiz |  | ## 💾 Step 7 — Save Results<br>Appends quiz questions and metadata to the Google Sheet Quiz Bank tab.<br><br>## 🎓 Onboarding Video Quiz Builder<br><br>### How it works<br>User submits a training video URL via form. The workflow sends the video to WayinVideo for transcription, waits 45 seconds, then polls until the transcript is ready. Once ready, an AI agent (GPT) reads the full transcript and generates the exact number of MCQ questions requested. The final quiz is saved to a Google Sheet.<br><br>### Setup<br>1. Add your WayinVideo API key — replace `YOUR_WAYINVIDEO_API_KEY` in nodes 2 and 4<br>2. Add your OpenAI API key in the `OpenAI — Chat Model` node<br>3. Connect Google Sheets OAuth2 credentials in node 7<br>4. Replace `YOUR_GOOGLE_SHEET_ID` in node 7 with your actual Sheet ID<br>5. Ensure your Google Sheet has a tab named **Quiz Bank** with columns: Department, Video URL, Total Questions, Quiz Questions, Generated On<br><br>### Customization tips<br>- Swap `gpt-4o-mini` for a stronger model in the OpenAI node for higher quality questions<br>- Increase the Wait time (node 3) if transcription fails on longer videos<br>- Edit the AI prompt in node 6 to change question format, language, or difficulty level |

---

# 4. Reproducing the Workflow from Scratch

Below is the exact build order to recreate this workflow manually in n8n.

1. **Create a new workflow**
   - Name it: `Create onboarding video quizzes using WayinVideo, GPT-4o-mini and Google Sheets`

2. **Add a Form Trigger node**
   - Node type: `Form Trigger`
   - Name: `1. Form — Video URL + Topic`
   - Form title: `🎓 Onboarding Video Quiz Builder`
   - Form description:  
     `Onboarding video ka URL paste karo — AI automatically MCQ quiz questions banayega Google Sheet mein save karne ke liye.`
   - Add three required fields:
     1. `Onboarding Video URL`
        - placeholder: `https://www.youtube.com/watch?v=xxxxxxx`
     2. `Department / Topic`
        - placeholder: `e.g. HR Policy, IT Security, Sales Process`
     3. `Number of Questions`
        - placeholder: `e.g. 10`

3. **Add an HTTP Request node for transcript submission**
   - Node type: `HTTP Request`
   - Name: `2. WayinVideo — Submit Transcript`
   - Method: `POST`
   - URL: `https://wayinvideo-api.wayin.ai/api/v2/transcripts`
   - Enable sending headers
   - Enable sending body
   - Body type: JSON
   - JSON body:
     ```json
     {
       "video_url": "{{ $json['Onboarding Video URL'] }}",
       "target_lang": "en"
     }
     ```
   - Add headers:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`
   - Connect `1. Form — Video URL + Topic` → `2. WayinVideo — Submit Transcript`

4. **Add a Wait node**
   - Node type: `Wait`
   - Name: `3. Wait — 45 Seconds`
   - Set wait amount to `45` seconds
   - Connect `2. WayinVideo — Submit Transcript` → `3. Wait — 45 Seconds`

5. **Add an HTTP Request node for transcript retrieval**
   - Node type: `HTTP Request`
   - Name: `4. WayinVideo — Get Transcript`
   - Method: `GET`
   - URL expression:
     ```text
     =https://wayinvideo-api.wayin.ai/api/v2/transcripts/results/{{ $('2. WayinVideo — Submit Transcript').item.json.data.id }}
     ```
   - Add headers:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`
   - Connect `3. Wait — 45 Seconds` → `4. WayinVideo — Get Transcript`

6. **Add an IF node for transcript status**
   - Node type: `If`
   - Name: `5. If — Transcript Status Check`
   - Configure the condition correctly:
     - Left value: `{{ $json.data.status }}`
     - Operator: `equals`
     - Right value: `SUCCEEDED`
   - Important: this fix is required; the source workflow JSON leaves this blank
   - Connect `4. WayinVideo — Get Transcript` → `5. If — Transcript Status Check`

7. **Create the polling loop**
   - Connect the **false** output of `5. If — Transcript Status Check` → `3. Wait — 45 Seconds`
   - This causes the workflow to wait and poll again when the transcript is not ready

8. **Add an AI Agent node**
   - Node type: `AI Agent`
   - Name: `6. AI — Generate MCQ Quiz`
   - Set prompt type to define text directly
   - Paste a prompt equivalent to the workflow prompt, but ideally correct the hardcoded `10` references if you want dynamic counts
   - A faithful reconstruction of the current prompt is:

     ```text
     =You are a quiz creator. Create exactly {{ $('1. Form — Video URL + Topic').item.json['Number of Questions'] }} multiple choice questions (MCQs) only from the transcript provided. No more, no less — always 10 questions.

     RULES:
     - Use only information from the transcript. No general knowledge.
     - Each question must have 4 options: A, B, C, D
     - Only one option is correct
     - Wrong options should be close but clearly wrong based on the transcript
     - Questions should cover facts, numbers, rules, steps mentioned in the video
     - No repeated questions
     - Keep language simple and easy to understand

     OUTPUT FORMAT — use this exact format for every question:

     Q[number]: [Question]
     A) [Option]
     B) [Option]
     C) [Option]
     D) [Option]
     Correct Answer: [A / B / C / D]
     Reference: [Speaker name and approximate time]
     ---

     EXAMPLE:

     Q1: How many paid leaves does an employee get per year?
     A) 8 days
     B) 10 days
     C) 12 days
     D) 15 days
     Correct Answer: C
     Reference: Speaker 1 — around 2 min 15 sec
     ---

     IMPORTANT:
     - Always generate exactly {{ $('1. Form — Video URL + Topic').item.json['Number of Questions'] }} questions — this is mandatory
     - Output questions only — no intro text, no conclusion, no explanation
     - At the end write: TOTAL QUESTIONS GENERATED: 10

     Department: {{ $('1. Form — Video URL + Topic').item.json['Department / Topic'] }}

     Create exactly {{ $('1. Form — Video URL + Topic').item.json['Number of Questions'] }} questions from this transcript only.

     Video Transcript:
     {{ $('4. WayinVideo — Get Transcript').item.json.data.transcript.map(s => '[' + s.speaker + ' | ' + Math.floor(s.start/1000) + 's] ' + s.text).join('\n') }}
     ```

   - Recommended improvement:
     - Replace all hardcoded `10` values with the same `Number of Questions` expression

9. **Add an OpenAI Chat Model node**
   - Node type: `OpenAI Chat Model`
   - Name: `OpenAI — Chat Model`
   - Select model: `gpt-4o-mini`
   - Add valid OpenAI credentials
   - Connect this node to `6. AI — Generate MCQ Quiz` using the AI language model connector, not the normal main connector

10. **Connect the success branch to the AI node**
    - Connect the **true** output of `5. If — Transcript Status Check` → `6. AI — Generate MCQ Quiz`

11. **Prepare the Google Sheet**
    - Create or choose a Google Sheet
    - Copy its spreadsheet ID
    - Create a tab named `Quiz Bank`
    - Add the following column headers in the sheet:
      - `Department`
      - `Video URL`
      - `Total Questions`
      - `Quiz Questions`
      - `Generated On`

12. **Add a Google Sheets node**
    - Node type: `Google Sheets`
    - Name: `7. Google Sheets — Save Quiz Bank`
    - Operation: `Append`
    - Connect Google Sheets OAuth2 credentials
    - Set document ID to your real spreadsheet ID
    - Set sheet name to `Quiz Bank`
    - Use explicit column mapping:
      - `Video URL` = `{{ $('1. Form — Video URL + Topic').item.json['Onboarding Video URL'] }}`
      - `Department` = `{{ $('1. Form — Video URL + Topic').item.json['Department / Topic'] }}`
      - `Generated On` = `{{ $now }}`
      - `Quiz Questions` = `{{ $json.choices[0].message.content }}`
      - `Total Questions` = `{{ $('1. Form — Video URL + Topic').item.json['Number of Questions'] }}`
    - Connect `6. AI — Generate MCQ Quiz` → `7. Google Sheets — Save Quiz Bank`

13. **Verify AI output structure before finalizing Google Sheets mapping**
    - Run a test execution up to the AI node
    - Inspect the actual output field from `6. AI — Generate MCQ Quiz`
    - If `choices[0].message.content` does not exist, replace the `Quiz Questions` mapping with the correct text field returned by your n8n version

14. **Add credentials**
    - **WayinVideo**
      - This workflow uses direct bearer headers rather than a stored credential node
      - Replace `YOUR_TOKEN_HERE` in both WayinVideo HTTP Request nodes
    - **OpenAI**
      - Add valid OpenAI API credentials in `OpenAI — Chat Model`
    - **Google Sheets**
      - Connect Google OAuth2 credentials in `7. Google Sheets — Save Quiz Bank`

15. **Test the full workflow**
    - Submit a sample video URL
    - Confirm the transcription submit step returns `data.id`
    - Confirm the transcript polling step eventually returns `data.status = SUCCEEDED`
    - Confirm AI output is generated
    - Confirm a row is appended to `Quiz Bank`

16. **Recommended hardening improvements**
    - Add validation for numeric question count
    - Add a maximum retry or loop counter to avoid indefinite polling
    - Add error handling branch for failed transcript statuses such as `FAILED`
    - Normalize AI prompt so requested count is not contradictory
    - Validate transcript existence before calling `.map(...)`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| 🎓 Onboarding Video Quiz Builder: User submits a training video URL via form. The workflow sends the video to WayinVideo for transcription, waits 45 seconds, then polls until the transcript is ready. Once ready, an AI agent (GPT) reads the full transcript and generates the exact number of MCQ questions requested. The final quiz is saved to a Google Sheet. | Workflow overview note |
| Add your WayinVideo API key — replace the placeholder bearer token in nodes 2 and 4. | WayinVideo setup |
| Add your OpenAI API key in the `OpenAI — Chat Model` node. | OpenAI setup |
| Connect Google Sheets OAuth2 credentials in node 7. | Google Sheets setup |
| Replace `YOUR_GOOGLE_SHEET_ID` in node 7 with your actual Sheet ID. | Google Sheets setup |
| Ensure your Google Sheet has a tab named **Quiz Bank** with columns: Department, Video URL, Total Questions, Quiz Questions, Generated On. | Sheet structure requirement |
| Customization: swap `gpt-4o-mini` for a stronger model for higher quality questions. | Model tuning |
| Customization: increase the wait time if transcription fails on longer videos. | Polling/transcription tuning |
| Customization: edit the AI prompt to change question format, language, or difficulty level. | Prompt tuning |

## Additional implementation cautions
- The IF node is currently broken in the provided workflow and must be fixed before production use.
- The AI prompt is internally inconsistent because it asks for a variable number of questions but also repeatedly hardcodes `10`.
- The Google Sheets mapping for quiz content may need adjustment depending on the actual output shape of the AI Agent node in your n8n version.
- No sub-workflows are used in this workflow.
- There is only one entry point: the Form Trigger node.