Create AI Meeting Summaries with Fireflies Transcripts & Dart Tasks using Gemini

https://n8nworkflows.xyz/workflows/create-ai-meeting-summaries-with-fireflies-transcripts---dart-tasks-using-gemini-10851


# Create AI Meeting Summaries with Fireflies Transcripts & Dart Tasks using Gemini

---

### 1. Workflow Overview

This workflow automates the generation of meeting summaries by processing transcripts from Fireflies.ai and integrating the results into the Dart workspace. Its primary use case is for teams or individuals who want automatic meeting notes, centralized documentation, and task creation for follow-up actions, leveraging AI summarization models.

**Logical Blocks:**

- **1.1 Input Reception:**  
  Receives meeting completion events from Fireflies via webhook and fetches the related transcript.

- **1.2 AI Processing and Parsing:**  
  Uses Google Gemini AI to generate a structured JSON summary from the transcript and parses that output.

- **1.3 Workspace Resource Retrieval:**  
  Retrieves the target Folder and Dartboard from Dart where documents and tasks will be created.

- **1.4 Document Creation:**  
  Creates a new Dart document populated with the AI-generated meeting summary.

- **1.5 Task Creation:**  
  Retrieves the newly created document and creates a related review task in Dart linking both the document and Fireflies recording.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving a webhook POST from Fireflies signaling that a meeting transcription has completed. It then fetches the transcript details using the meeting ID.

- **Nodes Involved:**  
  - Webhook  
  - Get a transcript

- **Node Details:**  

  - **Webhook**  
    - Type: Webhook node (HTTP POST listener)  
    - Configuration: Listens for POST requests at a unique path (UUID-based). No authentication specified (assumes secure or trusted environment).  
    - Inputs: External HTTP POST from Fireflies API webhook (event: "Transcription completed").  
    - Outputs: JSON containing meetingId used for further API calls.  
    - Edge Cases: Missing or malformed webhook data; unauthorized access if deployed publicly without security.

  - **Get a transcript**  
    - Type: Fireflies AI node (custom integration for Fireflies API)  
    - Configuration: Uses meetingId from webhook JSON body to fetch transcript details.  
    - Credentials: Fireflies API key required.  
    - Inputs: JSON with meetingId from Webhook node.  
    - Outputs: Transcript JSON object including meeting title, transcript URL, and summary fields.  
    - Edge Cases: Invalid meetingId, API rate limits, expired or revoked API keys.

---

#### 1.2 AI Processing and Parsing

- **Overview:**  
  This block uses the Google Gemini AI chat model to generate a structured JSON summary of the meeting transcript, then parses the AI output into usable fields for subsequent nodes.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Fireflies auto-summary (LangChain agent)  
  - Parsing stage for logic result (Code node)

- **Node Details:**  

  - **Google Gemini Chat Model**  
    - Type: AI Language Model node (Google PaLM Gemini 2.5 Pro)  
    - Configuration: Uses the "models/gemini-2.5-pro" model with default options.  
    - Credentials: Google PaLM API key required.  
    - Inputs: Not connected directly in trigger flow; used as AI model resource by downstream node.  
    - Edge Cases: API quota exhaustion, network latency, or invalid credentials.

  - **Fireflies auto-summary**  
    - Type: LangChain AI agent node  
    - Configuration: Prompt designed to output a strict JSON object summarizing the meeting. The prompt enforces JSON-only output with four keys: overview, summary, key_items, action_items. Input is concatenated summary fields from the Fireflies transcript.  
    - Inputs: Transcript summary text from "Get a transcript" node.  
    - Outputs: Raw JSON string with meeting summary in predefined schema.  
    - Edge Cases: AI output malformed/not valid JSON, model response delays, or unexpected content.

  - **Parsing stage for logic result**  
    - Type: Code node (JavaScript)  
    - Configuration: Cleans and parses the raw AI output to ensure valid JSON object. Throws error if parsing fails.  
    - Inputs: Raw AI JSON string from "Fireflies auto-summary".  
    - Outputs: Parsed JSON object with meeting summary fields (overview, summary, key_items, action_items).  
    - Edge Cases: Parsing failures if AI output is corrupted or improperly formatted.

---

#### 1.3 Workspace Resource Retrieval

- **Overview:**  
  Retrieves the target Folder and Dartboard resources from Dart workspace where documents and tasks will be created, ensuring all creations are properly organized.

- **Nodes Involved:**  
  - Retrieve an existing folder  
  - Retrieve an existing dartboard

- **Node Details:**  

  - **Retrieve an existing folder**  
    - Type: Dart API node  
    - Configuration: Fetches folder by fixed ID ("u4FhgTk3stn7") where meeting documents will be saved.  
    - Credentials: Dart API with OAuth2 credentials.  
    - Inputs: Triggered after parsing AI output.  
    - Outputs: Folder metadata confirming target location.  
    - Edge Cases: Invalid or expired folder ID, permission errors.

  - **Retrieve an existing dartboard**  
    - Type: Dart API node  
    - Configuration: Fetches Dartboard by fixed ID ("K7jRC0JC2Wxz") where tasks will be created.  
    - Credentials: Dart API with OAuth2 credentials.  
    - Inputs: Triggered after creating the document (to ensure document exists first).  
    - Outputs: Dartboard metadata for task creation.  
    - Edge Cases: Invalid dartboard ID, permission errors.

---

#### 1.4 Document Creation

- **Overview:**  
  Creates a new document in the retrieved Folder containing the AI-generated meeting summary, formatted with clear sections for overview, summary, key items, and action items.

- **Nodes Involved:**  
  - Create a new doc

- **Node Details:**  

  - **Create a new doc**  
    - Type: Dart API node  
    - Configuration: Creates a Dart document with title from transcript data and formatted text including overview, summary, key items, and action items extracted from parsed JSON.  
    - Inputs: Folder info (ensures correct placement) and parsed summary JSON.  
    - Outputs: Document metadata including document ID and URL.  
    - Edge Cases: Document creation failures due to API errors, invalid folder, or malformed content.

---

#### 1.5 Task Creation

- **Overview:**  
  Retrieves the newly created document to confirm details, then creates a review task in Dart linking the document and the Fireflies meeting recording for easy follow-up.

- **Nodes Involved:**  
  - Retrieve an existing doc  
  - Create a new task

- **Node Details:**  

  - **Retrieve an existing doc**  
    - Type: Dart API node  
    - Configuration: Fetches the newly created document by ID to obtain metadata such as URL for task description.  
    - Inputs: Document ID from "Create a new doc" node output.  
    - Outputs: Document details including public URL.  
    - Edge Cases: Document not found if creation failed or delayed.

  - **Create a new task**  
    - Type: Dart API node  
    - Configuration: Creates a task titled "Review recent meeting: [meeting title]" with description linking to the Dart document and the Fireflies meeting transcript URL.  
    - Inputs: Document URL from "Retrieve an existing doc" and transcript meeting details.  
    - Outputs: Task metadata confirming creation.  
    - Edge Cases: Task creation failure, invalid Dartboard reference, permission issues.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                     | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
|---------------------------|-------------------------------|-----------------------------------|----------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                   | Webhook                       | Receive Fireflies webhook trigger | -                          | Get a transcript                 | ## 1. Workflow launch through Fireflies AI webhook trigger<br>Triggered by an online meeting completion connected to Fireflies<br><br>**Summary categorization**<br>AI generates summary from fireflies transcript node with these categories in mind:<br>- Overview<br>- Summary<br>- Key items <br>- Action items<br><br>**Model selection**<br>Test and use any AI chat model. (Results could vary by model quality)                                                                                                        |
| Get a transcript          | Fireflies API node            | Fetch meeting transcript data     | Webhook                    | Fireflies auto-summary           |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Fireflies auto-summary    | LangChain AI agent            | Generate AI JSON summary          | Get a transcript           | Parsing stage for logic result   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Parsing stage for logic result | Code (JavaScript)             | Parse AI JSON output               | Fireflies auto-summary     | Retrieve an existing folder      | ## 2. Parsing the AI output<br>Parse output into a usable format for the next nodes.<br><br>### JSON formatting<br>- **Parse:** AI output into structured JSON  <br>- **Extract:** Key takeaways, Topics covered, Next items, Action items<br>- **Ensure:** Clean, consistent, taggable data                                                                                                                                                                                                        |
| Retrieve an existing folder | Dart API node                | Get target folder for docs        | Parsing stage for logic result | Create a new doc               | ## 3. Retrieve target folder<br>Select the Folder in your workspace where new meeting documents will be created.<br><br>### Folder targeting<br>- Locate your Folder ID in the workspace  <br>- Replace the placeholder ID in this node<br>- Confirms all generated documents route to the correct folder                                                                                                                                                                                          |
| Create a new doc          | Dart API node                | Create meeting summary document   | Retrieve an existing folder | Retrieve an existing dartboard   | ## 4. Create new document<br>This step creates a new document in your selected folder containing the generated meeting summary.<br><br>### Document creation includes<br>- Document title<br>- Key takeaways<br>- Topics covered<br>- Next items<br>- Action items <br>                                                                                                                                                                                                                         |
| Retrieve an existing dartboard | Dart API node             | Get target dartboard for tasks    | Create a new doc           | Retrieve an existing doc         | ## 5. Retrieve target dartboard<br>Select the Dartboard in your workspace where new tasks will be created.<br><br>### Dartboard targeting<br>- Locate your Dartboard ID in the workspace  <br>- Replace the placeholder ID in this node<br>- Confirms all generated tasks route to the correct board                                                                                                                                                                                         |
| Retrieve an existing doc  | Dart API node                | Get newly created document details| Retrieve an existing dartboard | Create a new task              |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Create a new task         | Dart API node                | Create review task with links     | Retrieve an existing doc    | -                               | ## 6. Create a review task in Dart  <br>Create a task in Dart to review the summary of your recent online meeting through your newly linked document.  <br><br>### Task creation includes  <br>- **CTA to view meeting summary** + linked newly generated document  <br>- **CTA to view Fireflies meeting recording** + Fathom recording link  <br><br>### Outcome  <br>A review task is automatically generated in Dart, connecting both your meeting summary and Fireflies recording for seamless follow-up.  |
| Sticky Note               | Sticky Note                  | Project overview and instructions | -                          | -                               | ## Meeting Summary Generator (Fireflies.AI → Dart)  \nAutomatically generate a meeting summary from your meetings through Fathom, save it to a Dart document, and create a review task with the Fathom link attached.  \n\n### Who’s It For  \n- Teams or individuals needing automatic meeting notes  \n- Project managers tracking reviews and actions  \n- Users of Fireflies + Dart for streamlined documentation  \n\n### How to Set Up  \n- Connect your Dart account (workspace + folder access)  \n- Add your PROD webhook link in the webhook node to Fireflies API settings ([Docs](https://docs.fireflies.ai/graphql-api/webhooks)) \n- Add your [API key](https://docs.fireflies.ai/fundamentals/authorization) on the Fireflies Transcript node\n- Replace dummy Folder ID and Dartboard ID with your targets  \n- Choose your preferred AI model for summaries  \n\n### Customizing the Workflow  \n- Edit the AI prompt to adjust tone or style  \n- Add or change summary sections (e.g., Key takeaways, Action Items, Summary)  \n |
| Sticky Note1              | Sticky Note                  | Block 1 explanation               | -                          | -                               | See Webhook sticky note above (covers 1.1 block)                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sticky Note2              | Sticky Note                  | Block 2 explanation               | -                          | -                               | See Parsing stage sticky note above (covers 1.2 block)                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Sticky Note5              | Sticky Note                  | Block 3 explanation               | -                          | -                               | See Retrieve folder sticky note above (covers 1.3 folder retrieval)                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Sticky Note6              | Sticky Note                  | Block 4 explanation               | -                          | -                               | See Create doc sticky note above (covers 1.4 document creation)                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sticky Note7              | Sticky Note                  | Block 5 explanation               | -                          | -                               | See Retrieve dartboard sticky note above (covers 1.3 dartboard retrieval)                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Sticky Note4              | Sticky Note                  | Block 6 explanation               | -                          | -                               | See Task creation sticky note above (covers 1.5 task creation)                                                                                                                                                                                                                                                                                                                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use a unique UUID path (e.g., "437a1a60-35af-4c81-9058-176d825b7173")  
   - Purpose: Receive Fireflies webhook event for "Transcription completed"  
   - No authentication configured (consider securing if exposed to public)  

2. **Add Fireflies Get a transcript Node**  
   - Type: Fireflies AI node (custom integration)  
   - Credentials: Configure Fireflies API key credential  
   - Parameter: Transcript ID set to `={{ $json.body.meetingId }}` (extracted from webhook payload)  
   - Connect Webhook output to this node's input  

3. **Add Google Gemini Chat Model Node**  
   - Type: LangChain AI Language Model (Google PaLM Gemini 2.5 Pro)  
   - Credentials: Configure Google PaLM API key credential  
   - Model Name: "models/gemini-2.5-pro"  
   - No special options needed  

4. **Add Fireflies auto-summary Node (LangChain Agent)**  
   - Type: LangChain AI Agent  
   - Prompt: Use the provided JSON-enforcing prompt that instructs AI to return a strict JSON summary object with keys: overview, summary, key_items, action_items  
   - Input: Pass the transcript summary fields (overview, short_summary, bullet_gist, action_items) from "Get a transcript" node  
   - AI Model: Connect to the Google Gemini Chat Model node  
   - Connect "Get a transcript" output to this node's input  

5. **Add Parsing stage for logic result Node**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     let rawString = $input.first().json.output;
     rawString = rawString.trim().replace(/^```(json)?/i, '').replace(/```$/i, '').trim();
     try {
       return JSON.parse(rawString);
     } catch (error) {
       throw new Error(`Failed to parse JSON. Cleaned input was:\n${rawString}\n\nError: ${error.message}`);
     }
     ```  
   - Connect output of "Fireflies auto-summary" to this node's input  

6. **Add Retrieve an existing folder Node**  
   - Type: Dart API node  
   - Operation: Get Folder  
   - Folder ID: Replace placeholder `"u4FhgTk3stn7"` with your actual Dart folder ID  
   - Credentials: Configure Dart API OAuth2 credentials  
   - Connect output of parsing node here  

7. **Add Create a new doc Node**  
   - Type: Dart API node  
   - Operation: Create Doc  
   - Parameters:  
     - Title: `={{ $("Get a transcript").item.json.data.title }}`  
     - Text: A formatted string combining overview, summary, key_items, and action_items from parsing node JSON, e.g.:  
       ```
       Overview:

       {{ $('Parsing stage for logic result').item.json.summary.overview }}

       Summary:

       {{ $('Parsing stage for logic result').item.json.summary.summary }}

       Key items:

       {{ $('Parsing stage for logic result').item.json.summary.key_items }}

       Action Items:

       {{ $('Parsing stage for logic result').item.json.summary.action_items }}
       ```  
   - Connect output of folder retrieval node here  

8. **Add Retrieve an existing dartboard Node**  
   - Type: Dart API node  
   - Operation: Get Dartboard  
   - Dartboard ID: Replace placeholder `"K7jRC0JC2Wxz"` with your actual Dartboard ID  
   - Credentials: Dart API OAuth2  
   - Connect output of "Create a new doc" node here  

9. **Add Retrieve an existing doc Node**  
   - Type: Dart API node  
   - Operation: Get Doc  
   - Document ID: `={{ $("Create a new doc").item.json.item.id }}`  
   - Credentials: Dart API OAuth2  
   - Connect output of dartboard retrieval node here  

10. **Add Create a new task Node**  
    - Type: Dart API node  
    - Operation: Create Task  
    - Parameters:  
      - Title: `Review recent meeting: {{ $("Get a transcript").item.json.data.title }}`  
      - Description:  
        ```
        View meeting summary here: {{ $json.item.htmlUrl }}

        [Fireflies meeting link]({{ $("Get a transcript").item.json.data.transcript_url }})
        ```  
    - Credentials: Dart API OAuth2  
    - Connect output of "Retrieve an existing doc" node here  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                     | Context or Link                                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Connect your Dart account fully to enable workspace and folder access.                                                                                                                                                                                          | Dart API setup                                                                                                                   |
| Add your PROD webhook URL from the Webhook node into Fireflies API webhook settings to receive live meeting completion events.                                                                                                                                | Fireflies webhook docs: https://docs.fireflies.ai/graphql-api/webhooks                                                          |
| You need to add your Fireflies API key on the Fireflies Transcript node for authentication.                                                                                                                                                                     | Fireflies API key docs: https://docs.fireflies.ai/fundamentals/authorization                                                     |
| Replace dummy Folder ID and Dartboard ID placeholders in Dart nodes with your actual workspace resource IDs to target where documents and tasks are created.                                                                                                  | Dart workspace resource management                                                                                                |
| The AI prompt is strictly designed to output JSON only, no markdown or other text to ensure parsing reliability. Adjust tone or sections by modifying the prompt in the "Fireflies auto-summary" node.                                                         | AI prompt customization                                                                                                          |
| This workflow is ideal for teams wanting automated meeting summaries linked directly to task and document management for seamless project tracking and review.                                                                                                 | Workflow purpose                                                                                                                 |
| Fireflies transcripts can include multiple summary fields; this workflow concatenates them for AI input to generate a structured summary.                                                                                                                    | Fireflies transcript data structure                                                                                              |
| Google Gemini model ("models/gemini-2.5-pro") is selected here for AI summarization, but any compatible model can be tested and swapped, affecting summary quality and response times.                                                                          | Google PaLM API: https://developers.generativeai.google/api/guides/models                                                      |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.