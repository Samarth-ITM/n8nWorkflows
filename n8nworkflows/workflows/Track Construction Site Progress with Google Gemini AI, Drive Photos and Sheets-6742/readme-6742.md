Track Construction Site Progress with Google Gemini AI, Drive Photos and Sheets

https://n8nworkflows.xyz/workflows/track-construction-site-progress-with-google-gemini-ai--drive-photos-and-sheets-6742


# Track Construction Site Progress with Google Gemini AI, Drive Photos and Sheets

### 1. Workflow Overview

The **AI-Powered Construction Site Progress Tracker** workflow automates daily monitoring and reporting of construction site progress by analyzing site photos stored in Google Drive, summarizing progress via Google Gemini AI, logging results in Google Sheets, and notifying stakeholders via email. It is designed for construction managers, contractors, and project stakeholders who need concise, automated daily updates on site status.

**Logical Blocks:**

- **1.1 Scheduled Trigger and Photo Retrieval:**  
  Automatically triggers daily and fetches the latest site photos from a designated Google Drive folder.

- **1.2 Image Processing Loop:**  
  Splits the list of photos into batches and iteratively sends each image for AI analysis.

- **1.3 AI Analysis and Summarization:**  
  Passes images through Google Gemini AI and a LangChain-based AI node to produce a structured, bullet-point progress summary and next step suggestion.

- **1.4 Data Logging:**  
  Appends the AI-generated summaries to a Google Sheet for record-keeping and tracking.

- **1.5 Notification:**  
  Sends an email to stakeholders containing a notification that the daily progress report is ready.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Photo Retrieval

- **Overview:**  
  This block initiates the workflow on a daily schedule and retrieves all recent construction site photos from a specified Google Drive folder.

- **Nodes Involved:**  
  - Schedule Trigger  
  - List a file  
  - Sticky Note3 (contextual comment)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule trigger  
    - Role: Starts the workflow automatically every day.  
    - Configuration: Runs on a daily interval (default daily schedule).  
    - Inputs: None (trigger node).  
    - Outputs: Connected to “List a file”.  
    - Edge Cases: Misconfiguration may cause missed runs; time zone considerations may affect trigger timing.

  - **List a file**  
    - Type: Google Drive node  
    - Role: Lists files (photos) in a specified Google Drive folder.  
    - Configuration: Operation = “list” with default options; uses Google Sheets test credentials for Drive access.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Sends list of files to the batch splitter node.  
    - Edge Cases: Invalid or missing folder permissions, empty folder, API rate limits, auth token expiration.

  - **Sticky Note3**  
    - Content: "Fetches the latest uploaded site photos from the Drive folder."

---

#### 2.2 Image Processing Loop

- **Overview:**  
  This block splits the list of retrieved images into batches and processes each image individually through AI analysis.

- **Nodes Involved:**  
  - Tack All Images (SplitInBatches)  
  - Sticky Note2 (contextual comment)

- **Node Details:**

  - **Tack All Images**  
    - Type: SplitInBatches  
    - Role: Processes each image one-by-one by batching the input list from Google Drive.  
    - Configuration: Default batch size (1 per batch, implied).  
    - Inputs: List of files from Google Drive node.  
    - Outputs: Each image passed individually to AI Analysis and loops back to self for next batch.  
    - Edge Cases: Empty input list, batch size misconfiguration, infinite loop if batch termination condition fails.

  - **Sticky Note2**  
    - Content: "Loops through each image to send them individually for AI analysis."

---

#### 2.3 AI Analysis and Summarization

- **Overview:**  
  Processes each image through Google Gemini AI, then a LangChain AI node to produce a concise construction progress summary and suggested next step.

- **Nodes Involved:**  
  - Google Gemini  
  - AI Analysis (LangChain chainLlm)  
  - Sticky Note4 (contextual comment)

- **Node Details:**

  - **Google Gemini**  
    - Type: LangChain Google Gemini AI node  
    - Role: Sends image data to Google Gemini AI model for initial construction progress detection.  
    - Configuration: Model = "models/gemini-1.0-pro".  
    - Credentials: Google Palm API credentials configured.  
    - Inputs: Single images from batch splitter.  
    - Outputs: Sends AI response to the LangChain AI Analysis node.  
    - Edge Cases: API quota exceeded, auth failure, model unavailability, response latency.

  - **AI Analysis**  
    - Type: LangChain chainLlm node  
    - Role: Processes Gemini AI output with a custom prompt to generate a structured summary and next step.  
    - Configuration: Custom prompt instructs to summarize progress in 2-3 bullet points focusing on structural elements, machinery, materials, and changes, plus a concise next step suggestion.  
    - Inputs: AI response from Google Gemini node.  
    - Outputs: Sends summarized text to Google Sheets appending node.  
    - Edge Cases: Prompt failure, incomplete data, API errors, formatting errors.

  - **Sticky Note4**  
    - Content: "Passes each image to Gemini AI for construction progress detection."

---

#### 2.4 Data Logging

- **Overview:**  
  Appends the AI-generated construction progress summaries and next step suggestions to a Google Sheet for daily tracking.

- **Nodes Involved:**  
  - Append data to a sheet  
  - Sticky Note1 (contextual comment)

- **Node Details:**

  - **Append data to a sheet**  
    - Type: Google Sheets node  
    - Role: Appends rows containing AI-generated progress summaries to "Sheet1!A:B" of a specified sheet.  
    - Configuration: Operation = “append”, Sheet ID provided (masked in the JSON).  
    - Credentials: Google Sheets API credentials configured.  
    - Inputs: Summarized text from AI Analysis node.  
    - Outputs: Sends success output to Email notification node.  
    - Edge Cases: Sheet ID invalid or missing, permission denied, API quota exceeded, data format mismatch.

  - **Sticky Note1**  
    - Content: "Logs the AI-generated summary into a Google Sheet for tracking."

---

#### 2.5 Notification

- **Overview:**  
  Sends an email notification to stakeholders indicating the daily construction site progress report is ready.

- **Nodes Involved:**  
  - Send email  
  - Sticky Note5 (contextual comment)

- **Node Details:**

  - **Send email**  
    - Type: Email Send node  
    - Role: Sends a plain-text email notification to a fixed recipient email address.  
    - Configuration:  
      - To: contractor@gmail.com  
      - From: abc@gmail.com  
      - Subject: "Construction Site Progress Report"  
      - Text: "Construction site progress report is ready"  
    - Credentials: SMTP credentials configured for sending.  
    - Inputs: Triggered after successful append to Google Sheets.  
    - Outputs: None (end node).  
    - Edge Cases: SMTP auth failure, invalid email addresses, network issues.

  - **Sticky Note5**  
    - Content: "Sends the daily site progress update to stakeholders."

---

### 3. Summary Table

| Node Name          | Node Type                    | Functional Role                            | Input Node(s)      | Output Node(s)        | Sticky Note                                                                                 |
|--------------------|------------------------------|--------------------------------------------|--------------------|-----------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger   | Schedule Trigger              | Initiates workflow daily                    | None               | List a file           | Runs the workflow daily to check for new site photos.                                      |
| List a file        | Google Drive                 | Retrieves latest site photos from Drive    | Schedule Trigger   | Tack All Images       | Fetches the latest uploaded site photos from the Drive folder.                             |
| Tack All Images    | SplitInBatches               | Iterates through each photo individually   | List a file        | AI Analysis, Tack All Images | Loops through each image to send them individually for AI analysis.                     |
| Google Gemini      | LangChain Google Gemini AI  | Runs image through Google Gemini AI model  | Tack All Images     | AI Analysis (LangChain) | Passes each image to Gemini AI for construction progress detection.                         |
| AI Analysis        | LangChain chainLlm           | Creates progress summary & next step       | Google Gemini      | Append data to a sheet |                                                                                             |
| Append data to a sheet | Google Sheets              | Logs AI summary into Google Sheet           | AI Analysis        | Send email             | Logs the AI-generated summary into a Google Sheet for tracking.                            |
| Send email         | Email Send                   | Sends notification email to stakeholders   | Append data to a sheet | None                | Sends the daily site progress update to stakeholders.                                      |
| Sticky Note        | Sticky Note                  | Comment                                    | None               | None                  | Runs the workflow daily to check for new site photos.                                      |
| Sticky Note1       | Sticky Note                  | Comment                                    | None               | None                  | Logs the AI-generated summary into a Google Sheet for tracking.                            |
| Sticky Note2       | Sticky Note                  | Comment                                    | None               | None                  | Loops through each image to send them individually for AI analysis.                        |
| Sticky Note3       | Sticky Note                  | Comment                                    | None               | None                  | Fetches the latest uploaded site photos from the Drive folder.                             |
| Sticky Note4       | Sticky Note                  | Comment                                    | None               | None                  | Passes each image to Gemini AI for construction progress detection.                        |
| Sticky Note5       | Sticky Note                  | Comment                                    | None               | None                  | Sends the daily site progress update to stakeholders.                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger Node:**  
   - Type: Schedule Trigger  
   - Configure interval: Daily (default every 24 hours)  
   - Position: Starting node of the workflow

2. **Add Google Drive Node to List Files:**  
   - Type: Google Drive  
   - Operation: List  
   - Configure folder ID (target folder containing site photos)  
   - Set Google API credentials (OAuth2 with appropriate Drive access)  
   - Connect output of Schedule Trigger to this node

3. **Add SplitInBatches Node to Process Images Individually:**  
   - Type: SplitInBatches  
   - Use default batch size of 1 (process one image per batch)  
   - Connect output of Google Drive node to this node

4. **Add Google Gemini AI Node for Image Analysis:**  
   - Type: LangChain Google Gemini AI node  
   - Model name: "models/gemini-1.0-pro"  
   - Set Google Palm API credentials (API key with access to Gemini model)  
   - Connect output of SplitInBatches node to this node

5. **Add LangChain Chain LLM Node for Summarization:**  
   - Type: LangChain chainLlm node  
   - Configure prompt with instructions:  
     "You are an AI construction progress analyst. Analyze the provided daily site photo and summarize the visible construction progress. Focus on structural elements (foundation, walls, roofing), machinery, materials, and noticeable changes. Return in 2-3 bullet points. Then suggest the next step of construction. Format: Progress Summary: • Point 1 • Point 2 Next Expected Step: <one-line suggestion>"  
   - Connect output of Google Gemini node (ai_languageModel output) to this node

6. **Add Google Sheets Node to Append Data:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Sheet ID: Set your target Google Sheet ID  
   - Range: "Sheet1!A:B" (or adjust to your sheet layout)  
   - Use Google Sheets credentials with write access  
   - Connect output of AI Analysis node to this node

7. **Add Email Send Node for Notification:**  
   - Type: Email Send  
   - Configure SMTP credentials (username, password, host, port, SSL/TLS)  
   - Set email parameters:  
     - To: contractor@gmail.com  
     - From: abc@gmail.com  
     - Subject: "Construction Site Progress Report"  
     - Text: "Construction site progress report is ready"  
   - Connect output of Append data to a sheet node to this node

8. **Verify connections:**  
   - Schedule Trigger → List a file → Tack All Images (SplitInBatches) → Google Gemini → AI Analysis → Append data to a sheet → Send email  
   - Tack All Images node loops back to itself for batch processing until all images processed

9. **Set up Credentials:**  
   - Google API OAuth2 credentials for Drive and Sheets  
   - Google Palm API credentials for Google Gemini AI  
   - SMTP credentials for sending emails

10. **Optional: Add Sticky Notes for Documentation:**  
    - Add sticky notes near respective nodes with explanatory comments as per block analysis for internal documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow runs daily to automate construction progress reporting using AI and Google ecosystem tools. | Workflow overview                                                                                   |
| Uses Google Gemini AI (PaLM) and LangChain integration for natural language summarization.          | Node "Google Gemini" and "AI Analysis"                                                             |
| Google Drive folder must be correctly shared and accessible by the Google API credentials used.     | Google Drive node requirements                                                                      |
| Google Sheets must have appropriate permissions set to allow appending data.                        | Google Sheets node requirements                                                                     |
| SMTP email credentials must be valid to send notifications successfully.                            | Email Send node requirements                                                                        |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.