Create Weekly Space Newsletter with NASA APIs, GPT-4 & Google Workspace

https://n8nworkflows.xyz/workflows/create-weekly-space-newsletter-with-nasa-apis--gpt-4---google-workspace-10911


# Create Weekly Space Newsletter with NASA APIs, GPT-4 & Google Workspace

---

### 1. Workflow Overview

This workflow automates the creation and distribution of a weekly space-themed newsletter using multiple NASA APIs and AI-generated content. It targets space enthusiasts or organizations wishing to deliver curated, engaging space news and imagery every Monday at 9 AM.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & NASA API Preparation**: Initiates the workflow weekly and constructs API query URLs for data retrieval.
- **1.2 NASA Data Fetching & Aggregation**: Retrieves space-related data from NASA APIs and merges them into a single dataset.
- **1.3 AI Content Generation**: Uses OpenAI GPT-4 to generate the newsletter text based on the aggregated data.
- **1.4 Newsletter Document Creation**: Creates and updates a Google Document with the AI-generated content.
- **1.5 PDF Conversion & Distribution**: Converts the Google Doc to PDF, emails the newsletter, archives it to Notion (optional), and sends a Slack notification upon success.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & NASA API Preparation

**Overview:**  
This block triggers the workflow weekly on Monday at 9 AM and prepares the URLs required to fetch data from various NASA APIs for the last 7 days.

**Nodes Involved:**  
- Weekly Monday Trigger  
- Prepare NASA API URLs  

**Node Details:**  

- **Weekly Monday Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow execution once a week on Monday at 9:00 AM.  
  - Configuration: Interval set to weekly, trigger day Monday, hour 9.  
  - Connections: Output to "Prepare NASA API URLs".  
  - Edge Cases: Misconfigured time zone or daylight savings can affect trigger timing.

- **Prepare NASA API URLs**  
  - Type: Code (JavaScript)  
  - Role: Calculates the date range for the past 7 days and constructs URLs for NASA APIs: APOD, DONKI, NeoWS, and EPIC.  
  - Configuration:  
    - Uses current date to define start and end dates.  
    - NASA API key hardcoded as 'DEMO_KEY' (must be replaced with a valid key).  
  - Key expressions: Template literals dynamically build API URLs with date parameters and API key.  
  - Inputs: From Schedule Trigger.  
  - Outputs: JSON containing URLs and date info.  
  - Edge Cases: If API key is invalid or missing, API calls will fail. Date calculations assume system clock accuracy.  
  - Notes: Requires user to replace 'DEMO_KEY' with a valid key as per Sticky Note 2.

---

#### 2.2 NASA Data Fetching & Aggregation

**Overview:**  
Fetches data asynchronously from NASA’s Astronomy Picture of the Day (APOD) and Space Weather (DONKI) APIs, merges the results, and limits the dataset size before passing it to AI generation.

**Nodes Involved:**  
- Fetch Astronomy Picture  
- Fetch Space Weather  
- Combine NASA Data  
- Limit API Results  

**Node Details:**  

- **Fetch Astronomy Picture**  
  - Type: HTTP Request  
  - Role: Fetches APOD data using the URL built in the previous node.  
  - Configuration: URL is dynamically set from prior node output.  
  - Inputs: From "Prepare NASA API URLs".  
  - Outputs: APOD JSON data.  
  - Edge Cases: API rate limits, network errors, or malformed URLs can cause failures.

- **Fetch Space Weather**  
  - Type: HTTP Request  
  - Role: Fetches DONKI space weather notifications.  
  - Configuration: URL dynamically from input JSON.  
  - Inputs: From "Prepare NASA API URLs".  
  - Outputs: DONKI JSON data.  
  - Edge Cases: Similar to APOD node, subject to API limits and network issues.

- **Combine NASA Data**  
  - Type: Merge  
  - Role: Combines data from APOD and DONKI requests into a single data set.  
  - Configuration: Mode set to "combineAll" to merge all inputs.  
  - Inputs: Outputs from both Fetch nodes.  
  - Outputs: Combined JSON array.  
  - Edge Cases: If one input is empty or fails, merged data may be incomplete.

- **Limit API Results**  
  - Type: Limit  
  - Role: Limits the number of combined data items passed forward to avoid excessive AI input size.  
  - Configuration: Default limit (unspecified in JSON, assumed reasonable).  
  - Inputs: From "Combine NASA Data".  
  - Outputs: Limited combined data.  
  - Edge Cases: Overly restrictive limits may omit relevant data; no limit risks large input size.

---

#### 2.3 AI Content Generation

**Overview:**  
Uses GPT-4 via the LangChain OpenAI node to generate engaging newsletter content from the aggregated NASA data.

**Nodes Involved:**  
- Generate Newsletter with AI  

**Node Details:**  

- **Generate Newsletter with AI**  
  - Type: OpenAI (LangChain)  
  - Role: Processes NASA data to create a human-readable newsletter using GPT-4.  
  - Configuration:  
    - Model set to GPT-4.  
    - Input: Limited NASA data from previous node.  
    - Prompt customization is possible (per Sticky Note 3).  
  - Inputs: From "Limit API Results".  
  - Outputs: JSON containing generated newsletter text.  
  - Edge Cases:  
    - API quota exhaustion or auth failure.  
    - Unexpected data format causing prompt or response errors.  
    - Latency or timeouts on OpenAI side.

---

#### 2.4 Newsletter Document Creation

**Overview:**  
Creates a new Google Document named with the current date, then inserts the AI-generated newsletter content into the document.

**Nodes Involved:**  
- Create Google Doc  
- Insert Newsletter Content  

**Node Details:**  

- **Create Google Doc**  
  - Type: Google Docs  
  - Role: Creates a new Google Document in the root folder with a date-stamped title.  
  - Configuration: Title template "SpaceWeekly_YYYY-MM-DD".  
  - Inputs: From AI generation node.  
  - Outputs: Document metadata including document ID.  
  - Credentials: Requires Google Workspace OAuth2 credentials.  
  - Edge Cases: Permission issues or quota limits in Google Docs API.

- **Insert Newsletter Content**  
  - Type: Google Docs (Update)  
  - Role: Inserts the AI-generated newsletter text into the created document.  
  - Configuration: Inserts text at the start of the document using dynamic content from AI node.  
  - Inputs: From "Create Google Doc".  
  - Outputs: Updated document info including document ID.  
  - Edge Cases: Document locking, API limits, or invalid document ID.

---

#### 2.5 PDF Conversion & Distribution

**Overview:**  
Converts the Google Doc into a PDF, emails the newsletter to recipients, optionally archives it to Notion, and sends a Slack notification to confirm success.

**Nodes Involved:**  
- Convert to PDF  
- Send Newsletter Email  
- Archive to Notion (disabled by default)  
- Wait for Distribution  
- Notify Success on Slack  

**Node Details:**  

- **Convert to PDF**  
  - Type: Google Drive  
  - Role: Downloads the Google Doc as a PDF file.  
  - Configuration: Uses document ID from prior node, converts to PDF format, names file with date.  
  - Inputs: From "Insert Newsletter Content".  
  - Outputs: Binary PDF data.  
  - Credentials: Google Workspace OAuth2.  
  - Edge Cases: API quota limits, conversion failures.

- **Send Newsletter Email**  
  - Type: Gmail  
  - Role: Sends the newsletter email with AI text and attaches the PDF.  
  - Configuration:  
    - Recipient email hardcoded as "your-email@example.com" (to be customized).  
    - Subject includes current date.  
    - Body contains AI-generated newsletter text.  
    - PDF attached as binary data.  
  - Inputs: From "Convert to PDF" and "Generate Newsletter with AI".  
  - Credentials: Gmail OAuth2 credentials.  
  - Edge Cases: Email sending limits, invalid email addresses.

- **Archive to Notion** (disabled)  
  - Type: Notion  
  - Role: Optionally archives newsletter content into a Notion database.  
  - Configuration: Disabled by default; requires Notion database ID and API token.  
  - Inputs: From "Send Newsletter Email".  
  - Edge Cases: Permission issues, invalid database ID.

- **Wait for Distribution**  
  - Type: Merge  
  - Role: Waits for completion of both email sending and Notion archiving before continuing.  
  - Inputs: From "Send Newsletter Email" and "Archive to Notion".  
  - Outputs: Signals completion of distribution.

- **Notify Success on Slack**  
  - Type: Slack  
  - Role: Sends a confirmation message to a Slack channel announcing successful newsletter distribution.  
  - Configuration:  
    - Message text fixed.  
    - Channel selected from list (default "general").  
    - Uses OAuth2 authentication.  
  - Inputs: From "Wait for Distribution".  
  - Edge Cases: Slack API rate limits, webhook misconfiguration.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                       | Input Node(s)                    | Output Node(s)                     | Sticky Note                                              |
|---------------------------|---------------------------------|------------------------------------|---------------------------------|----------------------------------|----------------------------------------------------------|
| Weekly Monday Trigger      | Schedule Trigger                | Triggers workflow weekly            | -                               | Prepare NASA API URLs             |                                                          |
| Prepare NASA API URLs      | Code                            | Builds NASA API URLs for data fetch| Weekly Monday Trigger            | Fetch Astronomy Picture, Fetch Space Weather | Step 1: API Configuration (replace 'DEMO_KEY')           |
| Fetch Astronomy Picture    | HTTP Request                   | Fetches APOD data                   | Prepare NASA API URLs            | Combine NASA Data                | Step 1: API Configuration                                 |
| Fetch Space Weather        | HTTP Request                   | Fetches DONKI space weather data   | Prepare NASA API URLs            | Combine NASA Data                | Step 1: API Configuration                                 |
| Combine NASA Data          | Merge                          | Merges APOD and DONKI data         | Fetch Astronomy Picture, Fetch Space Weather | Limit API Results               | Step 1: API Configuration                                 |
| Limit API Results          | Limit                         | Limits combined data size           | Combine NASA Data                | Generate Newsletter with AI      |                                                          |
| Generate Newsletter with AI| OpenAI (LangChain)             | Generates newsletter content        | Limit API Results               | Create Google Doc               | Step 2: AI Content Generation (customize prompt)          |
| Create Google Doc          | Google Docs                    | Creates new Google Doc              | Generate Newsletter with AI      | Insert Newsletter Content        |                                                          |
| Insert Newsletter Content  | Google Docs                    | Inserts AI content into document    | Create Google Doc               | Convert to PDF                  |                                                          |
| Convert to PDF             | Google Drive                   | Converts Google Doc to PDF          | Insert Newsletter Content       | Send Newsletter Email, Archive to Notion |                                                          |
| Send Newsletter Email      | Gmail                         | Sends newsletter email              | Convert to PDF, Generate Newsletter with AI | Wait for Distribution           | Step 3: Distribution (configure recipients)               |
| Archive to Notion (disabled) | Notion                       | Archives newsletter in Notion       | Send Newsletter Email           | Wait for Distribution           | Step 3: Distribution (enable and configure if needed)     |
| Wait for Distribution      | Merge                         | Waits for email & archive completion| Send Newsletter Email, Archive to Notion | Notify Success on Slack         |                                                          |
| Notify Success on Slack    | Slack                         | Sends success notification          | Wait for Distribution           | -                              | Step 3: Distribution (configure Slack webhook & channel)  |
| Sticky Note                | Sticky Note                   | Documentation and instructions      | -                               | -                              | Weekly Space Newsletter Generator overview and setup      |
| Sticky Note 2              | Sticky Note                   | API key and API endpoint notes      | -                               | -                              | Step 1: API Configuration details                         |
| Sticky Note 3              | Sticky Note                   | AI prompt customization notes       | -                               | -                              | Step 2: AI Content Generation customization guide          |
| Sticky Note 4              | Sticky Note                   | Distribution configuration notes    | -                               | -                              | Step 3: Distribution guidance and Notion node disabled by default |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Monday at 9:00 AM.

2. **Add a Code Node "Prepare NASA API URLs"**  
   - Type: Code (JavaScript)  
   - Paste code that:  
     - Calculates today’s date and date 7 days ago.  
     - Sets NASA API key variable (replace `'DEMO_KEY'` with your actual key).  
     - Constructs URLs for APOD, DONKI, NeoWS, and EPIC APIs with date parameters.  
   - Connect output from Schedule Trigger.

3. **Add HTTP Request Node "Fetch Astronomy Picture"**  
   - Type: HTTP Request  
   - URL: Use expression to pull `apodUrl` from previous Code node output.  
   - Connect from "Prepare NASA API URLs".

4. **Add HTTP Request Node "Fetch Space Weather"**  
   - Type: HTTP Request  
   - URL: Use expression to pull `donkiUrl` from previous Code node output.  
   - Connect from "Prepare NASA API URLs".

5. **Add Merge Node "Combine NASA Data"**  
   - Type: Merge  
   - Mode: combineAll  
   - Inputs: Connect outputs from both "Fetch Astronomy Picture" and "Fetch Space Weather".

6. **Add Limit Node "Limit API Results"**  
   - Type: Limit  
   - Default limit or set a reasonable maximum number of items to pass.  
   - Connect from "Combine NASA Data".

7. **Add OpenAI Node "Generate Newsletter with AI"**  
   - Type: OpenAI (LangChain)  
   - Credentials: Connect your OpenAI GPT-4 API credentials.  
   - Model: GPT-4  
   - Input: Pass data from "Limit API Results".  
   - Customize prompt as desired for newsletter style and sections.

8. **Add Google Docs Node "Create Google Doc"**  
   - Type: Google Docs  
   - Credentials: Connect Google Workspace OAuth2.  
   - Title: Use expression to name document "SpaceWeekly_YYYY-MM-DD" based on current date.  
   - Connect from "Generate Newsletter with AI".

9. **Add Google Docs Node "Insert Newsletter Content"**  
   - Type: Google Docs (Update)  
   - Action: Insert text using the AI-generated newsletter content from previous node.  
   - Connect from "Create Google Doc".

10. **Add Google Drive Node "Convert to PDF"**  
    - Type: Google Drive  
    - Credentials: Google Workspace OAuth2.  
    - Operation: Download with conversion from Google Docs to PDF.  
    - File ID: Use document ID from "Insert Newsletter Content".  
    - File name: Use date-stamped naming convention with `.pdf` extension.  
    - Connect from "Insert Newsletter Content".

11. **Add Gmail Node "Send Newsletter Email"**  
    - Type: Gmail  
    - Credentials: Connect Gmail OAuth2.  
    - To: Configure recipient email(s).  
    - Subject: Include current date.  
    - Message body: Include AI-generated newsletter text.  
    - Attachments: Attach PDF binary data from "Convert to PDF".  
    - Connect from "Convert to PDF" and "Generate Newsletter with AI".

12. **(Optional) Add Notion Node "Archive to Notion"**  
    - Type: Notion  
    - Credentials: Connect Notion API token.  
    - Configure database ID and page title/content with newsletter data.  
    - Default: Disabled; enable and configure if archiving is desired.  
    - Connect from "Send Newsletter Email".

13. **Add Merge Node "Wait for Distribution"**  
    - Type: Merge  
    - Inputs: Connect from "Send Newsletter Email" and "Archive to Notion" (or only email if Notion disabled).  
    - Purpose: Waits for all distribution tasks to complete.

14. **Add Slack Node "Notify Success on Slack"**  
    - Type: Slack  
    - Credentials: Connect Slack OAuth2.  
    - Channel: Select channel (e.g., "general").  
    - Message: Fixed success notification text.  
    - Connect from "Wait for Distribution".

15. **Add Sticky Notes**  
    - Add multiple Sticky Note nodes positioned as per logical blocks, containing:  
      - Workflow overview and setup instructions.  
      - API key replacement and NASA API details.  
      - AI prompt customization notes.  
      - Distribution configuration and Notion node guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| NASA API key must be acquired at https://api.nasa.gov and replaced in the Code node before running workflow.  | NASA API documentation and signup page          |
| Customize AI prompt to adapt newsletter tone, content sections, and language preferences as needed.            | Sticky Note 3 in workflow                         |
| Email recipients and Slack channel must be configured with valid addresses and OAuth2 credentials.             | Sticky Note 4 and Gmail/Slack node configuration |
| Notion archiving is optional and disabled by default; requires database ID and API token to enable.             | Notion integration documentation                  |
| Workflow runs every Monday at 9 AM; adjust Schedule Trigger for different scheduling needs.                      | Schedule Trigger configuration                    |
| OpenAI GPT-4 usage requires valid API keys and is subject to usage quotas.                                      | OpenAI API documentation                          |
| Google Workspace credentials require OAuth2 setup with appropriate scopes for Docs, Drive, and Gmail APIs.      | Google Cloud Platform Console                      |
| Slack notifications require OAuth2 credentials and channel selection; Webhook setup recommended for reliability.| Slack API and OAuth2 app setup guides             |

---

**Disclaimer:**  
The provided text originates solely from an automated n8n workflow. It complies strictly with content and usage policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.

---