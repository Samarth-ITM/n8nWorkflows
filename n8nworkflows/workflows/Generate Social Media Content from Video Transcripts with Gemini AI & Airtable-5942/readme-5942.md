Generate Social Media Content from Video Transcripts with Gemini AI & Airtable

https://n8nworkflows.xyz/workflows/generate-social-media-content-from-video-transcripts-with-gemini-ai---airtable-5942


# Generate Social Media Content from Video Transcripts with Gemini AI & Airtable

---
### 1. Workflow Overview

This workflow automates the generation of rich social media content from video transcripts stored in Airtable, leveraging Google Gemini AI models and integrating with Google Drive for file management. It is designed for content creators or social media managers who want to streamline multi-platform content creation without manual rewriting or formatting.

The workflow is composed of the following logical blocks:

- **1.1 Trigger & Input Reception**  
  Listens for webhook events from Airtable to initiate the process, capturing the record ID and action type.

- **1.2 Data Retrieval**  
  Fetches the full Airtable record, including the video transcript and metadata.

- **1.3 Project Folder Management**  
  Creates a dedicated Google Drive folder for the project under a specific parent directory and links this folder back to Airtable.

- **1.4 AI-Powered Content Generation**  
  Uses Google Gemini AI (flash model variant) to analyze the transcript and generate a structured JSON output with tailored content for multiple social media platforms.

- **1.5 Saving Artifacts**  
  Saves the original transcript as a text file into the Google Drive folder.

- **1.6 Update Airtable with Results**  
  Stores all AI-generated social media content back to the Airtable record, formatted for ease of publishing.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

- **Overview:**  
  This block waits for an Airtable webhook trigger to start the workflow, capturing essential parameters like RecordId.

- **Nodes Involved:**  
  - 🎯 Webhook Trigger  
  - When clicking ‘Execute workflow’ (manual trigger for testing)  
  - Edit Fields1 (manual input simulation for RecordId)

- **Node Details:**

  - **🎯 Webhook Trigger**  
    - Type: Webhook node  
    - Role: Receives HTTP POST requests from Airtable automation.  
    - Config: Path set uniquely to "2d9f3a0a-e2e4-4005-8ac9-f192349a59fd". No authentication or additional options configured.  
    - Inputs: External HTTP webhook call  
    - Outputs: Provides JSON containing query parameters, including `RecordId`.  
    - Potential Failures: Network issues, missing or malformed webhook requests, incorrect path.

  - **When clicking ‘Execute workflow’**  
    - Type: Manual trigger  
    - Role: Allows manual execution of the workflow for testing.  
    - No inputs, outputs trigger the next node.

  - **Edit Fields1**  
    - Type: Set node  
    - Role: Sets a manual test value for `query.RecordId` to simulate webhook input (hardcoded RecordId).  
    - Outputs: JSON with `query.RecordId` = "recA37lWBsz2Y233D".  
    - Potential Failure: None; manual input.

#### 2.2 Data Retrieval

- **Overview:**  
  Fetches the full Airtable record using the RecordId from the trigger, extracting fields like transcript and name.

- **Nodes Involved:**  
  - 1. Get Record Data

- **Node Details:**

  - **1. Get Record Data**  
    - Type: Airtable node (Get Record operation)  
    - Role: Retrieves the record from the configured Airtable base and table using RecordId from webhook input.  
    - Config:  
      - Base: "Netkreatives.com" Airtable app  
      - Table: "Youtube tool"  
      - Record ID: Expression `={{ $json.query.RecordId }}`  
    - Credentials: Airtable Personal Access Token  
    - Inputs: JSON with `query.RecordId`  
    - Outputs: Full record fields including Name, transcript, and others.  
    - Potential Failures: Invalid RecordId, authentication errors, API rate limits, record not found.

#### 2.3 Project Folder Management

- **Overview:**  
  Creates a Google Drive folder named after the Airtable record’s Name field inside a predefined "tutorials" directory, then updates the Airtable record with the folder ID.

- **Nodes Involved:**  
  - 2. Create Project Folder  
  - 5. Link Folder to Record

- **Node Details:**

  - **2. Create Project Folder**  
    - Type: Google Drive node (Create Folder operation)  
    - Role: Creates a new folder in Google Drive under the folder ID "tutorials" (ID: 10BP0rscwtRsG1tYr7N2xfxVRLch_1-MO).  
    - Config:  
      - Folder name set dynamically as `={{ $json.Name }}` from Airtable record  
      - Folder color set to blue (#0E66E9)  
      - Drive: "My Drive"  
    - Credentials: Google Drive OAuth2 account  
    - Inputs: Record fields from Airtable  
    - Outputs: Folder metadata with folder ID  
    - Potential Failures: Auth errors, quota limits, invalid folder ID, Google API errors.

  - **5. Link Folder to Record**  
    - Type: Airtable node (Update Record operation)  
    - Role: Updates the Airtable record with the newly created Google Drive folder ID in the "google drive" field.  
    - Config:  
      - Record ID from webhook trigger `={{ $('🎯 Webhook Trigger').item.json.query.RecordId }}`  
      - Field "google drive" set to folder ID `={{ $json.id }}` from previous node  
      - Base and Table same as previous Airtable node  
    - Credentials: Airtable Personal Access Token  
    - Inputs: Folder metadata  
    - Outputs: Success/failure of update  
    - Potential Failures: Record lock, invalid IDs, API errors.

#### 2.4 AI-Powered Content Generation

- **Overview:**  
  Uses Google Gemini AI Flash model to generate structured multi-platform social media content from the video transcript.

- **Nodes Involved:**  
  - 🤖 AI Content Generator  
  - ⚡ Gemini Flash Model  
  - 📋 JSON Output Parser

- **Node Details:**

  - **🤖 AI Content Generator**  
    - Type: LangChain Agent node  
    - Role: Coordinates AI model interaction and output parsing.  
    - Config:  
      - Prompt includes detailed instructions to create engaging, value-driven content per platform (YouTube, Twitter, LinkedIn, Facebook, Instagram, TikTok, Shorts).  
      - Output expected as structured JSON.  
      - Uses AI language model and output parser subnodes.  
    - Inputs: Transcript and record data from Airtable  
    - Outputs: Parsed JSON with fields like youtube_title, twitter_thread, instagram_post, etc.  
    - Dependencies: Uses "⚡ Gemini Flash Model" for generation and "📋 JSON Output Parser" for response structure.  
    - Potential Failures: AI model timeouts, malformed JSON output, prompt errors, API quota exceeded.

  - **⚡ Gemini Flash Model**  
    - Type: LangChain LM Chat Google Gemini node  
    - Role: Provides the underlying AI model (Google Gemini 2.5 Flash) for text generation.  
    - Config: Model name "models/gemini-2.5-flash"  
    - Credentials: Google Gemini API  
    - Inputs: Text prompt from AI Content Generator  
    - Outputs: Raw AI response  
    - Potential Failures: Authentication, network, rate limiting.

  - **📋 JSON Output Parser**  
    - Type: LangChain Output Parser node  
    - Role: Parses the JSON response from Gemini model and auto-fixes minor errors if needed.  
    - Config: Uses a schema example defining expected fields (youtube_title, twitter_thread array, etc.)  
    - Inputs: Raw AI response  
    - Outputs: Structured JSON for downstream use  
    - Potential Failures: Parsing errors, schema mismatch.

#### 2.5 Saving Artifacts

- **Overview:**  
  Saves the original video transcript text as a file inside the created Google Drive folder.

- **Nodes Involved:**  
  - 6. Save Transcript File

- **Node Details:**

  - **6. Save Transcript File**  
    - Type: Google Drive node (Create from Text operation)  
    - Role: Creates a text file named after the record Name, containing the transcript text.  
    - Config:  
      - Filename: `={{ $json.fields.Name }}`  
      - Content: `={{ $json.fields.transcript }}`  
      - Folder ID: dynamically set from "2. Create Project Folder" node output  
      - Drive: "My Drive"  
    - Credentials: Google Drive OAuth2  
    - Inputs: Airtable record fields and folder ID  
    - Outputs: Created file metadata  
    - Potential Failures: Auth errors, quota exceeded, missing transcript data.

#### 2.6 Update Airtable with Results

- **Overview:**  
  Updates the original Airtable record with all AI-generated social media content fields, including joining arrays like Twitter threads into newline-separated strings.

- **Nodes Involved:**  
  - 4. Save Social Media Content

- **Node Details:**

  - **4. Save Social Media Content**  
    - Type: Airtable node (Update Record operation)  
    - Role: Writes back to Airtable the complete set of generated social content fields.  
    - Config:  
      - Record ID from "1. Get Record Data" node  
      - Fields mapped with expressions from AI Content Generator output JSON  
      - Twitter thread array joined with double newlines for readability  
      - Supports multiple social fields: youtube_title, linkedin_post, instagram_post, tiktok_caption, etc.  
      - Base and Table consistent with earlier Airtable nodes  
    - Credentials: Airtable Personal Access Token  
    - Inputs: AI content JSON  
    - Outputs: Confirmation of update  
    - Potential Failures: Airtable API rate limits, invalid field names, record lock.

---

### 3. Summary Table

| Node Name               | Node Type                                   | Functional Role                              | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                              |
|-------------------------|---------------------------------------------|----------------------------------------------|-------------------------|---------------------------|---------------------------------------------------------------------------------------------------------|
| 🎯 Webhook Trigger      | Webhook                                     | Entry trigger from Airtable webhook          | —                       | 1. Get Record Data         |                                                                                                         |
| When clicking ‘Execute workflow’ | Manual Trigger                         | Manual test trigger                           | —                       | Edit Fields1               |                                                                                                         |
| Edit Fields1             | Set                                         | Sets test RecordId for manual trigger        | When clicking ‘Execute workflow’ | —                   |                                                                                                         |
| 1. Get Record Data       | Airtable (Get Record)                        | Retrieves full Airtable record by RecordId   | 🎯 Webhook Trigger       | 2. Create Project Folder, 🤖 AI Content Generator |                                                                                                         |
| 2. Create Project Folder | Google Drive (Create Folder)                  | Creates project folder in Google Drive       | 1. Get Record Data       | 5. Link Folder to Record   |                                                                                                         |
| 5. Link Folder to Record | Airtable (Update Record)                      | Updates Airtable record with folder ID       | 2. Create Project Folder | 6. Save Transcript File    |                                                                                                         |
| 6. Save Transcript File  | Google Drive (Create from Text)                | Saves transcript text file in Drive folder   | 5. Link Folder to Record | —                         |                                                                                                         |
| 🤖 AI Content Generator  | LangChain Agent                              | Generates multi-platform social content JSON | 1. Get Record Data       | 4. Save Social Media Content |                                                                                                         |
| ⚡ Gemini Flash Model    | LangChain LM Chat Google Gemini              | AI model providing text generation            | 🤖 AI Content Generator  | 📋 JSON Output Parser      |                                                                                                         |
| 📋 JSON Output Parser    | LangChain Output Parser Structured            | Parses AI JSON output into structured data    | ⚡ Gemini Flash Model     | 🤖 AI Content Generator    |                                                                                                         |
| 4. Save Social Media Content | Airtable (Update Record)                   | Saves generated content back to Airtable      | 🤖 AI Content Generator  | —                         | Saves all AI-generated social media content to Airtable record                                          |
| Workflow Documentation   | Sticky Note                                  | Describes workflow overview and logic         | —                       | —                         | # 🎬 Social Media Content Generator ... (full documentation text)                                       |
| Sticky Note              | Sticky Note                                  | Branding and social media links                | —                       | —                         | ## Netkreatives : Ai powered growth and productivity \n*Follow me on :\n🐦 Twitter/X: https://x.com/netkreatives\n💼 LinkedIn: https://www.linkedin.com/company/netkreatives\n📺 YouTube: https://www.youtube.com/@netkreatives\n📸 Instagram: https://www.instagram.com/netkreatives/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger**  
   - Node Type: Webhook  
   - Configuration:  
     - Set unique path, e.g., `2d9f3a0a-e2e4-4005-8ac9-f192349a59fd`  
     - No authentication required  
   - Purpose: Receive Airtable webhook calls with RecordId query parameter.  

2. **Create Manual Trigger for Testing** (optional)  
   - Node Type: Manual Trigger  
   - Connect to next node to simulate webhook.

3. **Create Set Node for Manual RecordId Input** (optional)  
   - Node Type: Set  
   - Add field: `query.RecordId` with a test RecordId string value.  
   - Connect Manual Trigger to this node.

4. **Create Airtable Get Record Node**  
   - Node Type: Airtable  
   - Operation: Get Record  
   - Credentials: Airtable Personal Access Token (set up in n8n credentials)  
   - Base: Select your Airtable base (e.g., "Netkreatives.com")  
   - Table: Select your table (e.g., "Youtube tool")  
   - Record ID: Use expression `={{ $json.query.RecordId }}`  
   - Connect Webhook Trigger (or Set node) output to this node.

5. **Create Google Drive Create Folder Node**  
   - Node Type: Google Drive  
   - Operation: Create Folder  
   - Credentials: Google Drive OAuth2 account  
   - Drive ID: "My Drive"  
   - Folder ID: Parent folder ID for "tutorials" (e.g., `10BP0rscwtRsG1tYr7N2xfxVRLch_1-MO`)  
   - Folder Name: Expression `={{ $json.Name }}` from Airtable record  
   - Folder Color: Set to blue (#0E66E9) for visual identification  
   - Connect output of Airtable Get Record to this node.

6. **Create Airtable Update Record Node to Link Folder**  
   - Node Type: Airtable  
   - Operation: Update Record  
   - Credentials: Same Airtable Personal Access Token  
   - Base/Table: Same as Get Record node  
   - Record ID: Expression `={{ $('🎯 Webhook Trigger').item.json.query.RecordId }}` (or from trigger node)  
   - Field to update: "google drive" set to folder ID from Google Drive node (`={{ $json.id }}`)  
   - Connect Google Drive Create Folder node output to this node.

7. **Create Google Drive Create from Text Node for Transcript**  
   - Node Type: Google Drive  
   - Operation: Create from Text  
   - Credentials: Google Drive OAuth2  
   - Drive: "My Drive"  
   - Folder ID: Expression `={{ $('2. Create Project Folder').item.json.id }}` (folder created in previous step)  
   - File Name: Expression `={{ $json.fields.Name }}`  
   - Content: Expression `={{ $json.fields.transcript }}`  
   - Connect previous Airtable Update Record node to this node.

8. **Create LangChain AI Content Generator Node**  
   - Node Type: LangChain Agent  
   - Prompt: Use detailed prompt instructing AI to produce JSON structured multi-platform content from transcript, focusing on engagement and platform-specific optimization.  
   - Output Parser: Set to JSON Output Parser node.  
   - Connect output of Airtable Get Record node to this node (to supply transcript and metadata).

9. **Set up Gemini Flash Model Node**  
   - Node Type: LangChain LM Chat Google Gemini  
   - Model Name: `models/gemini-2.5-flash`  
   - Credentials: Google Gemini API credentials configured in n8n  
   - Connect this node as AI language model for the AI Content Generator node.

10. **Create JSON Output Parser Node**  
    - Node Type: LangChain Output Parser Structured  
    - Schema: Provide JSON schema example matching expected AI output (youtube_title, twitter_thread array, etc.)  
    - Enable Auto Fix to handle minor JSON formatting errors  
    - Connect Gemini Flash Model node output to this node, then connect parser output back to AI Content Generator node.

11. **Create Airtable Update Record Node to Save AI Content**  
    - Node Type: Airtable  
    - Operation: Update Record  
    - Credentials: Airtable Personal Access Token  
    - Base/Table: Same as previous Airtable nodes  
    - Record ID: Expression `={{ $('1. Get Record Data').item.json.id }}`  
    - Map all fields from AI Content Generator output JSON to corresponding Airtable fields (e.g., youtube_title, linkedin_post, instagram_post, etc.)  
    - For fields that are arrays (e.g., twitter_thread), join with newline characters (`.join('\n\n')`) for readability  
    - Connect AI Content Generator node output to this node.

12. **Add Sticky Notes** (optional)  
    - Add sticky notes with workflow description and branding links to improve maintainability and team documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow automates social media content generation from video transcripts, integrating Airtable, Google Drive, and Google Gemini AI. | Main workflow purpose and architecture.                                                         |
| Netkreatives Social Profiles: Twitter/X, LinkedIn, YouTube, Instagram for updates and support.                | https://x.com/netkreatives, https://www.linkedin.com/company/netkreatives, https://www.youtube.com/@netkreatives, https://www.instagram.com/netkreatives/ |
| Google Gemini 2.5 Flash model used for fast AI content generation with structured JSON output parsing.        | Requires valid Google Gemini API credentials configured in n8n.                                  |
| Airtable API requires Personal Access Token with appropriate read/write permissions on target base and table. | Ensure token validity and permission scopes before running.                                     |
| Google Drive OAuth2 credentials must allow folder and file creation in the specified Drive and folder.         | Recommended to create dedicated service account or OAuth2 app for secure access.                 |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow built with n8n, an integration and automation tool. The process adheres strictly to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.