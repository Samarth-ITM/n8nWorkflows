Extract LinkedIn Profile Data & Generate Follow-ups with Vision AI and Google Forms

https://n8nworkflows.xyz/workflows/extract-linkedin-profile-data---generate-follow-ups-with-vision-ai-and-google-forms-9855


# Extract LinkedIn Profile Data & Generate Follow-ups with Vision AI and Google Forms

---

### 1. Workflow Overview

This workflow automates the extraction of structured professional profile data from LinkedIn header screenshots submitted via a Google Form, enriches this data with AI-generated personalized follow-up messages, and updates a Google Sheets CRM accordingly. It is designed for relationship-building use cases, such as sales, recruiting, or networking, where quick insights and follow-ups based on visual profile data and personal notes are essential.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Triggered by Google Form submissions linked to a Google Sheet.
- **1.2 Image Retrieval:** Downloads the uploaded LinkedIn profile screenshot from Google Drive.
- **1.3 AI Vision Processing:** Uses OpenAI’s Vision model to analyze the image and extract structured profile data.
- **1.4 Data Structuring:** Post-processes and normalizes the AI-extracted data for consistency.
- **1.5 CRM Update:** Appends or updates the CRM Google Sheet with the enriched profile data.
- **1.6 AI Message Generation:** Generates a personalized LinkedIn direct message (DM) based on extracted data and user notes.
- **1.7 CRM Message Update:** Updates the CRM Google Sheet with the generated follow-up message.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for new rows added to a Google Sheet, which correspond to Google Form submissions containing a LinkedIn screenshot URL and quick notes.

- **Nodes Involved:**  
  - `trigger: form submission`

- **Node Details:**  
  - **Type:** Google Sheets Trigger  
  - **Role:** Initiates workflow on new form submission (new row added).  
  - **Configuration:**  
    - Monitors the sheet named by environment variable `SHEETS_TAB` or defaults to `'crm'`.  
    - Uses document ID from environment variable `SHEETS_ID`.  
    - Polls every minute for new rows.  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Emits new row data JSON.  
  - **Edge cases:**  
    - Failure if Google Sheets credentials are invalid or revoked.  
    - Missed triggers if polling interval is too long or API rate-limited.  
  - **Sticky Note:**  
    - "Trigger: When a 2-field only Google Form is submitted with a LinkedIn header screenshot + quick personal notes."

#### 1.2 Image Retrieval

- **Overview:**  
  Downloads the uploaded screenshot image file from Google Drive using the URL submitted in the form.

- **Nodes Involved:**  
  - `get screenshot`

- **Node Details:**  
  - **Type:** Google Drive (download file)  
  - **Role:** Extracts file ID from form URL and downloads binary image data.  
  - **Configuration:**  
    - Dynamically extracts Google Drive file ID from the form column specified by environment variable `COL_DRIVE_URL` or default column `'Upload a screenshot or image of the person'`.  
    - Downloads the file as binary data under property `data`.  
  - **Inputs:** Output from `trigger: form submission` (form row JSON).  
  - **Outputs:** Binary image data for analysis.  
  - **Edge cases:**  
    - Invalid or malformed URL resulting in failed file ID extraction.  
    - File not found or access denied due to Google Drive permission issues.  
  - **Sticky Note:**  
    - "This step gets the new screenshot to parse information."

#### 1.3 AI Vision Processing

- **Overview:**  
  Uses OpenAI’s Vision-enabled model to analyze the screenshot image and extract structured profile information.

- **Nodes Involved:**  
  - `Analyze image`

- **Node Details:**  
  - **Type:** OpenAI Vision (LangChain OpenAI node)  
  - **Role:** Analyze binary image input and extract JSON with profile info.  
  - **Configuration:**  
    - Model ID from environment variable `OPENAI_VISION_MODEL` or defaults to `'chatgpt-4o-latest'`.  
    - Instructions enforce strict JSON output with exactly five keys: `name`, `job_title`, `company`, `university`, `description`.  
    - Input type set to `binary` using property `data`.  
  - **Inputs:** Binary image data from `get screenshot`.  
  - **Outputs:** Model-generated JSON text with extracted fields.  
  - **Edge cases:**  
    - Model returns invalid JSON or unexpected format requiring robust parsing downstream.  
    - API timeouts or quota limits.  
    - Image content unclear or containing multiple people; model instructed to pick primary person.  
  - **Sticky Note:**  
    - "Open AI's vision model analyses the screenshot to extract info."

#### 1.4 Data Structuring

- **Overview:**  
  Processes the raw model output to normalize fields, apply title casing to names, handle nulls, and trim descriptions.

- **Nodes Involved:**  
  - `structure info`

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:** Parse, clean, and normalize AI output into well-structured JSON.  
  - **Key Logic:**  
    - Parses raw output string, removing markdown code fences and smart quotes.  
    - Maps multiple possible alias keys to canonical field names.  
    - Applies title case to personal names but retains original casing for companies and universities.  
    - Trims description to max 300 characters.  
  - **Inputs:** AI JSON output from `Analyze image`.  
  - **Outputs:** Cleaned JSON with keys: `name`, `job_title`, `company`, `location`, `university`, `description`.  
  - **Edge cases:**  
    - Parsing failures on malformed JSON; uses fallback double-parse.  
    - Missing fields replaced with `null`.  
  - **Sticky Note:**  
    - "Extracted information is structured."

#### 1.5 CRM Update

- **Overview:**  
  Appends or updates the Google Sheet CRM with the structured profile data and the original screenshot URL.

- **Nodes Involved:**  
  - `update crm`

- **Node Details:**  
  - **Type:** Google Sheets (append or update)  
  - **Role:** Synchronizes enriched profile data into the CRM sheet row corresponding to the form submission.  
  - **Configuration:**  
    - Matches rows by the screenshot URL column (`Upload a screenshot or image of the person`).  
    - Updates or appends columns: `company`, `location`, `full name`, `job title`, `university`, `description`, and preserves the screenshot URL.  
    - Sheet and document IDs from environment variables.  
  - **Inputs:** Cleaned JSON from `structure info` and original form data via expression.  
  - **Outputs:** Updated sheet row confirmation.  
  - **Edge cases:**  
    - Google Sheets API failures or permission errors.  
    - Conflicting updates if multiple workflows write simultaneously.  
  - **Sticky Note:**  
    - "Google sheet with form submission info is updated with added fields."

#### 1.6 AI Message Generation

- **Overview:**  
  Generates a personalized LinkedIn direct message based on structured profile data and personal notes from the form.

- **Nodes Involved:**  
  - `Message a model`

- **Node Details:**  
  - **Type:** OpenAI Text Completion (LangChain)  
  - **Role:** Craft a short, professional follow-up message grounded strictly on notes and structured data.  
  - **Configuration:**  
    - Model ID from environment variable `OPENAI_TEXT_MODEL` or defaults to `'gpt-4o-mini'`.  
    - Temperature set low (0.2) for focused output.  
    - System prompt enforces style and safety rules: no invented facts, grounding on notes, length ≤450 chars, friendly and professional tone.  
    - User prompt includes direct injection of notes and extracted fields.  
  - **Inputs:** Structured profile data JSON from `update crm` and original notes from form submission.  
  - **Outputs:** Text message in plain text, no markdown or emojis.  
  - **Edge cases:**  
    - Model hallucination mitigated by strict prompts, but must monitor output quality.  
    - Potential API rate limits or timeouts.  
  - **Sticky Note:**  
    - "With all this info AND your original notes, Open AI generates a message."

#### 1.7 CRM Message Update

- **Overview:**  
  Updates the CRM Google Sheet with the generated personalized follow-up message.

- **Nodes Involved:**  
  - `update crm with message`

- **Node Details:**  
  - **Type:** Google Sheets (append or update)  
  - **Role:** Adds the AI-generated follow-up message to the CRM row identified by the screenshot URL.  
  - **Configuration:**  
    - Matches rows by screenshot URL column.  
    - Updates the `follow-up message` column.  
    - Sheet and document IDs from environment variables.  
  - **Inputs:** Generated message text from `Message a model` and screenshot URL from `get screenshot`.  
  - **Outputs:** Updated row confirmation.  
  - **Edge cases:**  
    - Google Sheets API failures or permission issues.  
    - Message content may be empty if AI fails; handle gracefully.  
  - **Sticky Note:**  
    - "crm updated with message."

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                            | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                       |
|------------------------|----------------------------------|--------------------------------------------|--------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| trigger: form submission| Google Sheets Trigger            | Initiates workflow on form submission      | -                        | get screenshot             | Trigger: When a 2-field only Google Form is submitted with a LinkedIn header screenshot + quick personal notes. |
| get screenshot         | Google Drive (download file)     | Downloads screenshot image from Drive      | trigger: form submission | Analyze image              | This step gets the new screenshot to parse information                                          |
| Analyze image          | OpenAI Vision (LangChain OpenAI) | Extracts structured data from the image    | get screenshot           | structure info             | Open AI's vision model analyses the screenshot to extract info                                  |
| structure info         | Code (JavaScript)                | Normalizes and cleans AI extracted data    | Analyze image            | update crm                 | Extracted information is structured                                                            |
| update crm             | Google Sheets (append/update)    | Updates CRM with structured profile data   | structure info           | Message a model            | Google sheet with form submission info is updated with added fields                             |
| Message a model        | OpenAI Text Completion (LangChain)| Generates personalized LinkedIn follow-up | update crm               | update crm with message    | With all this info AND your original notes, Open AI generates a message                        |
| update crm with message| Google Sheets (append/update)    | Updates CRM with AI-generated follow-up    | Message a model          | -                          | crm updated with message                                                                        |
| Sticky Note            | Sticky Note                     | Informational overlays                      | -                        | -                          | Trigger: When a 2-field only Google Form is submitted with a LinkedIn header screenshot + quick personal notes. |
| Sticky Note1           | Sticky Note                     | Informational overlay                       | -                        | -                          | This step gets the new screenshot to parse information                                         |
| Sticky Note2           | Sticky Note                     | Informational overlay                       | -                        | -                          | Open AI's vision model analyses the screenshot to extract info                                 |
| Sticky Note3           | Sticky Note                     | Informational overlay                       | -                        | -                          | Extracted information is structured                                                            |
| Sticky Note4           | Sticky Note                     | Informational overlay                       | -                        | -                          | Google sheet with form submission info is updated with added fields                            |
| Sticky Note5           | Sticky Note                     | Informational overlay                       | -                        | -                          | With all this info AND your original notes, Open AI generates a message                        |
| Sticky Note6           | Sticky Note                     | Informational overlay                       | -                        | -                          | crm updated with message                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger node:**
   - Type: Google Sheets Trigger  
   - Configure credentials with Google Sheets OAuth2.  
   - Set event to `Row Added`.  
   - Set document ID to environment variable `SHEETS_ID` or your Google Sheet ID.  
   - Set sheet name to environment variable `SHEETS_TAB` or `'crm'`.  
   - Set polling interval to every 1 minute.

2. **Add a Google Drive node (download file):**
   - Type: Google Drive  
   - Operation: Download  
   - Credentials: Google Drive OAuth2.  
   - Set `File ID` parameter using expression to extract the Drive file ID from the form column URL:  
     ```js
     (function () {
       const col = $env.COL_DRIVE_URL || 'Upload a screenshot or image of the person';
       const s = $json[col] || '';
       const m = s.match(/(?:\/file\/d\/|\/d\/|open\?id=|uc\?id=|thumbnail\?id=|id=)([-\w]{10,})/);
       return m ? m[1] : '';
     })()
     ```
   - Set `Binary Property` to `data`.

3. **Add an OpenAI Vision node (LangChain OpenAI):**
   - Type: OpenAI (LangChain)  
   - Resource: Image  
   - Operation: Analyze  
   - Input Type: Binary  
   - Binary Property Name: `data`  
   - Model ID: environment variable `OPENAI_VISION_MODEL` or `'chatgpt-4o-latest'`.  
   - Prompt:  
     ```
     You extract structured data from images of profiles, email signatures, business cards, event badges, resumes, or screenshots.

     Rules:
     - OUTPUT STRICT JSON ONLY. No markdown, no prose, no comments.
     - Return exactly these 5 keys; if a field isn’t clearly present, use null.
     - Prefer the primary/most prominent person if multiple appear.
     - Normalize whitespace and title-case personal names; keep original casing for company/university.
     - Keep description to 1–2 sentences summarizing what’s visible (role, focus, notable lines). Do not invent info.

     Return exactly:
     {
       "name": string|null,
       "job_title": string|null,
       "company": string|null,
       "university": string|null,
       "description": string|null
     }
     ```

4. **Add a Code node:**
   - Type: Code (JavaScript)  
   - Mode: "Run Once for Each Item"  
   - Paste the provided JS code that parses AI output, normalizes fields, applies title case to names, handles nulls, trims description length to 300 chars, and outputs structured JSON with keys: `name`, `job_title`, `company`, `location`, `university`, `description`.

5. **Add a Google Sheets node for CRM update:**
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Credentials: Google Sheets OAuth2.  
   - Document ID: environment variable `SHEETS_ID`.  
   - Sheet Name: environment variable `SHEETS_TAB` or `'crm'`.  
   - Matching Columns: `Upload a screenshot or image of the person` (used to find existing row).  
   - Mapping Columns:  
     - `company`, `location`, `full name`, `job title`, `university`, `description`, and the screenshot URL from the original form submission column.  

6. **Add an OpenAI Text Completion node (LangChain):**
   - Type: OpenAI (LangChain)  
   - Model: environment variable `OPENAI_TEXT_MODEL` or `'gpt-4o-mini'`.  
   - Temperature: 0.2  
   - Messages:  
     - System message setting style and grounding rules (no invented facts, plain text only, ≤450 characters, friendly and professional tone).  
     - User message template that injects:  
       - Notes from the form submission (column `Quick notes about the person` or env var `COL_NOTES`).  
       - Structured fields: `name`, `title/job title`, `company`, `location`, `university`, `description`.  
       - Task: Write one LinkedIn DM citing one concrete note detail with a light ask and a question.

7. **Add a Google Sheets node to update CRM with the generated message:**
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Credentials: Google Sheets OAuth2.  
   - Document ID and Sheet Name as before.  
   - Matching Columns: `Upload a screenshot or image of the person`.  
   - Mapping Columns: `follow-up message` mapped to the AI-generated message content, and the screenshot URL column.

8. **Connect nodes in order:**
   - `trigger: form submission` → `get screenshot` → `Analyze image` → `structure info` → `update crm` → `Message a model` → `update crm with message`

9. **Environment Variables and Credentials:**
   - Ensure environment variables are set:  
     - `SHEETS_ID` (Google Sheet ID)  
     - `SHEETS_TAB` (tab name in the sheet, default `'crm'`)  
     - `COL_DRIVE_URL` (name of the form column with screenshot URL, default `'Upload a screenshot or image of the person'`)  
     - `COL_NOTES` (name of the form column with quick notes, default `'Quick notes about the person'`)  
     - `OPENAI_VISION_MODEL` (OpenAI vision capable model, default `'chatgpt-4o-latest'`)  
     - `OPENAI_TEXT_MODEL` (OpenAI text completion model, default `'gpt-4o-mini'`)  
   - Configure Google OAuth2 credentials for Sheets and Drive nodes.  
   - Configure OpenAI credentials for LangChain OpenAI nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow is triggered by Google Form submissions containing only two fields: a screenshot URL and quick notes. | See Sticky Note on `trigger: form submission` node.                                                    |
| OpenAI Vision model is instructed to return strict JSON with five keys for reliable parsing.         | Prompts in `Analyze image` node; critical for downstream normalization.                                |
| The AI-generated LinkedIn DM strictly uses only the notes and extracted fields to avoid hallucination. | Prompt constraints in `Message a model` node ensure grounded and safe messaging.                       |
| Use environment variables to customize sheet names, columns, and model selection for flexibility.    | Allows adapting workflow easily to different Google Sheets or AI models without node reconfiguration. |
| Folder permissions and file access rights in Google Drive must allow the workflow’s service account to read uploaded screenshots. | Otherwise, `get screenshot` node will fail to download images.                                        |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---