Website Content Analysis with GPT-4 and Storage to Google Sheets

https://n8nworkflows.xyz/workflows/website-content-analysis-with-gpt-4-and-storage-to-google-sheets-4355


# Website Content Analysis with GPT-4 and Storage to Google Sheets

### 1. Workflow Overview

This workflow automates the process of analyzing website content using GPT-4 and storing the results in Google Sheets. It is designed for users who want to extract a concise summary and key structured data from any given website without incurring API scraping costs. The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger initiates the process; the target website URL is fetched via an HTTP request.
- **1.2 Content Conversion:** Converts the raw HTML content from the website into Markdown format for easier natural language processing.
- **1.3 AI Processing:** Utilizes OpenAI’s GPT-4.1-mini model to analyze the Markdown content and extract a summary plus key information.
- **1.4 Data Storage:** Saves the AI-generated structured content into a designated Google Sheets spreadsheet.
- **1.5 Documentation & Setup Notes:** Sticky note describing workflow purpose, setup instructions, and credential requirements.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow manually and retrieves the full HTML content of the specified website URL.

- **Nodes Involved:**  
  - Click to Start  
  - Input Your Website URL

- **Node Details:**

  - **Click to Start**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution on user command.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Input Your Website URL"  
    - Potential Failures: User forgets to trigger; no failure expected internally.

  - **Input Your Website URL**  
    - Type: HTTP Request  
    - Role: Fetches website HTML content from the URL.  
    - Configuration:  
      - URL: Set dynamically as "https://medium.com" (default example; user-adjustable).  
      - Response Format: String (raw HTML)  
      - No additional options set (e.g., no headers or authentication).  
    - Inputs: From "Click to Start"  
    - Outputs: Passes raw HTML in `data` field to next node.  
    - Potential Failures:  
      - Network errors (timeout, DNS failure)  
      - HTTP errors (404, 5xx)  
      - Incorrect or unreachable URL  
    - Edge Cases: Websites with JavaScript-rendered content may not be fully captured.

#### 2.2 Content Conversion

- **Overview:**  
  Converts the raw HTML content into Markdown to facilitate more effective AI text processing.

- **Nodes Involved:**  
  - Convert to Markdown Format

- **Node Details:**

  - **Convert to Markdown Format**  
    - Type: Markdown Conversion  
    - Role: Transforms HTML string into Markdown format.  
    - Configuration:  
      - Input HTML: Bound to `{{$json.data}}` from previous HTTP node.  
      - Destination Key: Stores output Markdown as `HTMLtoMarkDownConversion`.  
    - Inputs: HTML content from "Input Your Website URL"  
    - Outputs: Markdown text for AI processing.  
    - Potential Failures:  
      - Invalid or malformed HTML input causing conversion errors.  
      - Empty content passed forward if website response is empty.

#### 2.3 AI Processing

- **Overview:**  
  Uses OpenAI GPT-4.1-mini to analyze the Markdown content, producing a concise summary and a list of key website attributes.

- **Nodes Involved:**  
  - Process the Markdown to readable Contents

- **Node Details:**

  - **Process the Markdown to readable Contents**  
    - Type: OpenAI (LangChain integration)  
    - Role: Sends Markdown content as prompt to GPT-4.1-mini model for content analysis.  
    - Configuration:  
      - Model: GPT-4.1-mini (latest stable from LangChain presets).  
      - Messages:  
        - System prompt instructs the AI to act as a professional website analyst, extracting:  
          - A concise summary paragraph of the website’s purpose, target audience, and core offerings.  
          - A list of 10 key extracted items such as page title, meta description, services, CTAs, contact info, navigation, etc.  
        - The Markdown content is injected via expression `{{ $json.HTMLtoMarkDownConversion }}`.  
    - Credentials: OpenAI API key configured (credential name "NafiulHasanBD OpenAI").  
    - Inputs: Markdown content from "Convert to Markdown Format"  
    - Outputs: AI-generated text output for storage.  
    - Potential Failures:  
      - API key issues (invalid, expired, quota exceeded).  
      - Timeout or rate limiting from OpenAI.  
      - Errors in expression evaluation (e.g., missing Markdown content).  
    - Edge Cases: MarkDown exceeding token limits; ambiguous or empty input resulting in poor outputs.

#### 2.4 Data Storage

- **Overview:**  
  Updates or adds a row in a Google Sheets document with the AI-analyzed website content.

- **Nodes Involved:**  
  - Save the Website Scraping content to Google Sheet

- **Node Details:**

  - **Save the Website Scraping content to Google Sheet**  
    - Type: Google Sheets  
    - Role: Saves the processed information into a Google Sheet for persistent storage and review.  
    - Configuration:  
      - Operation: Update (could be changed to append depending on use case).  
      - Sheet Name: Not preset, configured dynamically or manually before execution.  
      - Document ID: Must be set to the target Google Sheets file ID.  
      - Data mapping: Not explicitly configured in the JSON, requires user setup to map AI output to columns.  
    - Credentials: OAuth2 Google Sheets API (credential name "nafiul.automation Google AC").  
    - Inputs: AI text output from "Process the Markdown to readable Contents"  
    - Outputs: None (terminal node)  
    - Potential Failures:  
      - Authentication errors (expired token, wrong scopes).  
      - Invalid document ID or sheet name.  
      - Permission denied on the Sheet.  
      - Mismatched data format causing update failures.

#### 2.5 Documentation & Setup Notes

- **Overview:**  
  Provides user instructions and workflow purpose description for clarity and onboarding.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note (UI-only)  
    - Content:  
      - Describes workflow purpose: scraping website content, summarizing with OpenAI, storing to Google Sheets.  
      - Setup instructions: specifying URL, adding OpenAI and Google Sheets credentials, configuring Google Sheets node.  
    - Position: Placed prominently in the editor for user reference.  
    - Inputs/Outputs: None  
    - No failure modes (informational only).

---

### 3. Summary Table

| Node Name                          | Node Type                 | Functional Role                          | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                   |
|-----------------------------------|---------------------------|----------------------------------------|-----------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Click to Start                    | Manual Trigger            | Start workflow manually                 | None                        | Input Your Website URL             |                                                                                                |
| Input Your Website URL            | HTTP Request              | Retrieve raw HTML content from website | Click to Start              | Convert to Markdown Format         |                                                                                                |
| Convert to Markdown Format        | Markdown Conversion       | Convert HTML to Markdown                | Input Your Website URL       | Process the Markdown to readable Contents |                                                                                                |
| Process the Markdown to readable Contents | OpenAI (LangChain)       | Analyze Markdown with GPT-4, extract summary and key info | Convert to Markdown Format   | Save the Website Scraping content to Google Sheet |                                                                                                |
| Save the Website Scraping content to Google Sheet | Google Sheets            | Store AI output into Google Sheets     | Process the Markdown to readable Contents | None                              |                                                                                                |
| Sticky Note                      | Sticky Note               | Workflow overview and setup instructions | None                        | None                              | ## 📌 What This Workflow Does\n\nThis template scrapes the content from a specified URL, uses OpenAI to summarize or extract key information from it, and then saves the structured output into a new row in Google Sheets.\n\n---\n\n## ⚙️ Setup Required\n\n### 1. **URL**\nSet the website you want to scrape in the **\"Start\"** node.\n\n### 2. **Credentials**\nAdd your API credentials for both the **OpenAI** and **Google Sheets** nodes.\n\n### 3. **Configuration**\nIn the **Google Sheets** node:\n- Select your target spreadsheet.\n- Map the columns to the data you want to save. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Add a "Manual Trigger" node named "Click to Start". This node will initiate the workflow on manual command.

2. **Add an HTTP Request Node**  
   - Name: "Input Your Website URL"  
   - Set the HTTP Method to `GET` (default).  
   - Set the URL parameter to the website you want to analyze, e.g., `https://medium.com`. Use an expression if you want this dynamic later.  
   - Set Response Format to `String` to get raw HTML content.  
   - Connect "Click to Start" node’s output to this node's input.

3. **Add a Markdown Conversion Node**  
   - Name: "Convert to Markdown Format"  
   - Configure input HTML parameter with expression referencing previous node's output: `{{$json.data}}`.  
   - Set the destination key as `HTMLtoMarkDownConversion` to store the Markdown output.  
   - Connect "Input Your Website URL" node output to this node.

4. **Add OpenAI Node (LangChain Integration)**  
   - Name: "Process the Markdown to readable Contents"  
   - Select the OpenAI GPT model `gpt-4.1-mini` or similar GPT-4 variant.  
   - In the messages parameter, add a system message with the prompt instructing the AI to:  
     - Summarize the website's purpose, audience, and offerings.  
     - Extract 10 key pieces of information (title, meta, services, CTAs, contacts, navigation, etc.)  
   - Use expression to inject the Markdown text: `{{ $json.HTMLtoMarkDownConversion }}`.  
   - Set up OpenAI API credentials (create or select existing).  
   - Connect "Convert to Markdown Format" node output to this node.

5. **Add Google Sheets Node**  
   - Name: "Save the Website Scraping content to Google Sheet"  
   - Set Operation to "Update" or "Append" depending on your use case.  
   - Select the target Google Spreadsheet by entering the Document ID.  
   - Select the Sheet Name within that document.  
   - Map the fields from the AI output to the relevant columns in the sheet (e.g., summary, key points).  
   - Set up Google Sheets OAuth2 credentials for authentication.  
   - Connect "Process the Markdown to readable Contents" node output to this node.

6. **Add Sticky Note (Optional but Recommended)**  
   - Add a Sticky Note to the canvas to describe the workflow’s purpose and setup instructions for future users.

7. **Verify Connections and Workflow Execution**  
   - Ensure connections follow: Manual Trigger → HTTP Request → Markdown Conversion → OpenAI → Google Sheets.  
   - Save and activate the workflow.  
   - Run manually to test functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| This workflow avoids API scraping costs by downloading raw HTML and processing with OpenAI locally.                                                        | Workflow description                                        |
| OpenAI GPT-4.1-mini model is used for balance between cost and performance.                                                                                   | AI Processing node configuration                             |
| Google Sheets node requires OAuth2 credentials with sufficient permissions to update the target spreadsheet.                                                | Credential setup instructions                                |
| For dynamic URL input, consider replacing the fixed URL in the HTTP Request node with an expression or external trigger input.                             | Enhancement suggestion                                      |
| Markdown conversion improves AI prompt clarity by stripping HTML tags and formatting.                                                                        | Conversion node explanation                                 |
| Official n8n OpenAI documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.openai/                                                   | Reference link                                              |
| Google Sheets API documentation: https://developers.google.com/sheets/api/guides/concepts                                                                | Reference link                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.