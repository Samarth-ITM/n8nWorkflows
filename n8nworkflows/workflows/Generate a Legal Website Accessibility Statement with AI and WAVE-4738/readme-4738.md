Generate a Legal Website Accessibility Statement with AI and WAVE

https://n8nworkflows.xyz/workflows/generate-a-legal-website-accessibility-statement-with-ai-and-wave-4738


# Generate a Legal Website Accessibility Statement with AI and WAVE

---

### 1. Workflow Overview

This workflow automates the generation of a legally compliant website Accessibility Statement tailored to EU regulations, specifically the European Accessibility Act (EAA). It is designed for businesses needing to meet accessibility disclosure requirements by analyzing their website for accessibility issues and generating a formal, professional HTML statement.

The workflow logic is divided into these main blocks:

- **1.1 Input Configuration:** Setting up all necessary parameters including API keys, target website URL, company details, and output language.
- **1.2 Website Content & Accessibility Data Retrieval:** Fetching the raw HTML of the target website and requesting an accessibility report from the WAVE API.
- **1.3 Data Processing:** Parsing and correlating WAVE report items to actual website elements using Cheerio for context extraction.
- **1.4 Accessibility Statement Generation:** Feeding the processed accessibility data and company info into an AI language model (Google Gemini) with a specialized prompt to generate a formal HTML Accessibility Statement.
- **1.5 Output Handling & Distribution:** Parsing the AI output, converting it into a downloadable HTML file, and sending it by email to a specified recipient.
- **1.6 Workflow Control & User Guidance:** Manual trigger to start the process and sticky notes providing instructions and setup guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Configuration

- **Overview:**  
  This block centralizes all user-configurable inputs such as API keys, website URL, company info, email for report delivery, and desired output language. Acts as the primary parameter source for downstream nodes.

- **Nodes Involved:**  
  - CHANGE THESE: dependencies

- **Node Details:**

  - **CHANGE THESE: dependencies**  
    - Type: Set node  
    - Role: Holds all essential variables: WAVE API key, URL to analyze, company name, company country (used for legal context), email for summary, and desired output language.  
    - Configuration: User must fill in the values before execution. The node outputs a JSON object accessed by subsequent nodes via expressions.  
    - Inputs: None (starting point)  
    - Outputs: Connected to “Get Website HTML” node.  
    - Edge Cases: Missing or invalid API key or URL will cause HTTP request failures downstream. Incorrect email or language may affect delivery or statement language.  

#### 2.2 Website Content & Accessibility Data Retrieval

- **Overview:**  
  This block fetches the target website’s HTML content and retrieves its accessibility analysis report from the WAVE API.

- **Nodes Involved:**  
  - Get Website HTML  
  - Get WAVE Report

- **Node Details:**

  - **Get Website HTML**  
    - Type: HTTP Request  
    - Role: Downloads the full HTML content of the specified website URL.  
    - Configuration: URL is dynamically set to the user input `URL to analyze`.  
    - Inputs: From “CHANGE THESE: dependencies” node.  
    - Outputs: Passes HTML content to “Get WAVE Report” node.  
    - Edge Cases: Network errors, invalid URL, website blocks scraping or API calls. Timeouts or 4xx/5xx HTTP errors possible.

  - **Get WAVE Report**  
    - Type: HTTP Request  
    - Role: Calls WAVE API to get accessibility issues categorized by types for the specified URL.  
    - Configuration: URL dynamically constructed with WAVE API key and target URL from dependencies node. Uses reporttype=4 for detailed output.  
    - Inputs: Receives website HTML output (though only needs URL; chained for order).  
    - Outputs: JSON report forwarded to “Map WAVE Report Items to Website selectors.” node.  
    - Edge Cases: Invalid API key, exceeding rate limits, API downtime, malformed URL, or errors in response format.

#### 2.3 Data Processing

- **Overview:**  
  Maps the accessibility issues reported by WAVE to actual HTML elements on the website by using selectors and Cheerio HTML parsing, extracting context snippets for AI input.

- **Nodes Involved:**  
  - Map WAVE Report Items to Website selectors.

- **Node Details:**

  - **Map WAVE Report Items to Website selectors.**  
    - Type: Code (JavaScript)  
    - Role: Loads website HTML into Cheerio, iterates over WAVE accessibility categories and items, finds matching elements using provided selectors, and extracts contextual HTML snippets.  
    - Configuration: Uses input from “Get Website HTML” and “Get WAVE Report” nodes. Outputs an array of analysis items with issue type, description, selectors, and context HTML.  
    - Inputs: Receives website HTML and WAVE JSON report.  
    - Outputs: JSON array of structured issue data sent to “Accessibility Statement Generator”.  
    - Edge Cases: Missing elements for selectors, malformed selectors, large HTML snippets truncated for context, potential runtime errors in cheerio loading or iteration.

#### 2.4 Accessibility Statement Generation

- **Overview:**  
  Uses an AI language model (Google Gemini) to generate a fully formatted, legally compliant HTML Accessibility Statement based on the processed accessibility data and company metadata.

- **Nodes Involved:**  
  - Accessibility Statement Generator  
  - gemini 2.5 pro  
  - Structured Output Parser

- **Node Details:**

  - **Accessibility Statement Generator**  
    - Type: LangChain Agent  
    - Role: Formulates a detailed prompt incorporating company details, compliance requirements, and accessibility scan data to instruct the AI to generate the statement.  
    - Configuration: Text prompt dynamically built with expressions pulling in company name, country, URL, and JSON stringified accessibility issues. Instructions ensure the output is a complete HTML document in English adhering to EAA and WCAG 2.1 AA level. Includes placeholders for contact and enforcement details.  
    - Inputs: Receives structured accessibility data from “Map WAVE Report Items to Website selectors.”  
    - Outputs: Raw AI output JSON object with “Accessibility Statement” field.  
    - Edge Cases: AI model errors, prompt parsing failures, incomplete or malformed HTML output, language mismatches, API quota or authentication errors.

  - **gemini 2.5 pro**  
    - Type: AI Language Model (Google Gemini)  
    - Role: Provides the language generation engine for the “Accessibility Statement Generator”.  
    - Configuration: Uses Google Palm API credentials. Model specified as “models/gemini-2.5-pro-preview-06-05”.  
    - Inputs: Prompt text from “Accessibility Statement Generator”.  
    - Outputs: AI-generated text JSON passed back to “Accessibility Statement Generator”.  
    - Edge Cases: API authentication issues, network timeouts, rate limiting, model deprecation.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser (Structured)  
    - Role: Parses AI output strictly according to a manual JSON schema expecting an object with a string property “Accessibility Statement” containing the HTML code.  
    - Configuration: JSON schema enforces output format and extracts the HTML content.  
    - Inputs: AI agent output.  
    - Outputs: Clean parsed JSON object forwarded to “Parse output as html” node.  
    - Edge Cases: Schema mismatches, parsing errors if AI output is malformed or incomplete.

#### 2.5 Output Handling & Distribution

- **Overview:**  
  Converts the AI-generated HTML string into a binary file for download and sends the accessibility statement via email to the configured recipient.

- **Nodes Involved:**  
  - Parse output as html  
  - Create accesibility statement html file  
  - Send accessibility statement by email

- **Node Details:**

  - **Parse output as html**  
    - Type: HTML Node  
    - Role: Extracts the raw HTML string from the parsed JSON and prepares it for file creation.  
    - Configuration: The expression extracts `output['Accessibility Statement']` from incoming JSON.  
    - Inputs: From “Accessibility Statement Generator” via Structured Output Parser.  
    - Outputs: Passes HTML string to “Create accesibility statement html file”.  
    - Edge Cases: Empty or invalid HTML string.

  - **Create accesibility statement html file**  
    - Type: Move Binary Data  
    - Role: Converts the JSON HTML content into a binary file format with correct filename and MIME type for download or attachment.  
    - Configuration: Filename dynamically generated with company name in snake_case, MIME type set to “text/html”. Uses raw data for conversion.  
    - Inputs: Receives HTML string.  
    - Outputs: Binary file sent to email node.  
    - Edge Cases: Filename conflicts, encoding errors.

  - **Send accessibility statement by email**  
    - Type: Gmail Node  
    - Role: Sends the generated HTML file as an email attachment to the email address specified in the configuration node.  
    - Configuration: Uses Gmail OAuth2 credentials; subject dynamically includes company name; message body informs about the attachment.  
    - Inputs: Receives binary HTML file.  
    - Outputs: None (final step).  
    - Edge Cases: Email sending failures, invalid recipient address, authentication errors.

#### 2.6 Workflow Control & User Guidance

- **Overview:**  
  Provides a manual trigger for workflow execution and sticky notes for user instructions and workflow explanation.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters.  
    - Inputs: None  
    - Outputs: Starts the chain by triggering “CHANGE THESE: dependencies”.  
    - Edge Cases: None.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Detailed instructions on workflow purpose, setup steps, and execution advice.  
    - Content: Explains automated generation of EU Accessibility Statements, setup steps including API keys and credentials, and running instructions.  
    - Inputs/Outputs: None.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Short guidance emphasizing configuration step as the main starting point.  
    - Content: “This is the main configuration node for the workflow. Click on this node and fill in all the required fields before running.”  
    - Inputs/Outputs: None.

---

### 3. Summary Table

| Node Name                              | Node Type                                | Functional Role                         | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                                                        |
|--------------------------------------|-----------------------------------------|---------------------------------------|----------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’      | Manual Trigger                         | Workflow start trigger                 | None                             | CHANGE THESE: dependencies              |                                                                                                                                  |
| CHANGE THESE: dependencies            | Set                                    | Central configuration of variables    | When clicking ‘Execute workflow’ | Get Website HTML                       | # ⚙️ Step 1: Start Here! This is the main configuration node for the workflow. Click on this node and fill in all required fields. |
| Get Website HTML                     | HTTP Request                           | Fetch raw website HTML content         | CHANGE THESE: dependencies       | Get WAVE Report                       |                                                                                                                                  |
| Get WAVE Report                     | HTTP Request                           | Retrieve accessibility report from WAVE API | Get Website HTML                 | Map WAVE Report Items to Website selectors. |                                                                                                                                  |
| Map WAVE Report Items to Website selectors. | Code (JavaScript)                      | Map WAVE issues to HTML elements and extract context | Get WAVE Report                  | Accessibility Statement Generator       |                                                                                                                                  |
| Accessibility Statement Generator    | LangChain Agent                       | Generate legal Accessibility Statement HTML via AI | Map WAVE Report Items to Website selectors. | Parse output as html                    |                                                                                                                                  |
| gemini 2.5 pro                      | AI Language Model (Google Gemini)      | AI model engine for statement generation | Accessibility Statement Generator | Accessibility Statement Generator (via AI output) |                                                                                                                                  |
| Structured Output Parser             | LangChain Output Parser (Structured)  | Parse AI output to JSON with HTML code | Accessibility Statement Generator | Accessibility Statement Generator       |                                                                                                                                  |
| Parse output as html                | HTML Node                             | Extract HTML string from JSON output   | Accessibility Statement Generator | Create accesibility statement html file |                                                                                                                                  |
| Create accesibility statement html file | Move Binary Data                      | Convert HTML string to binary file     | Parse output as html             | Send accessibility statement by email  |                                                                                                                                  |
| Send accessibility statement by email | Gmail Node                           | Email the generated HTML Accessibility Statement | Create accesibility statement html file | None                                   |                                                                                                                                  |
| Sticky Note                        | Sticky Note                           | Workflow overview and setup instructions | None                             | None                                   | # 🚀 Automatically Generate Your EU Accessibility Statement - Explains workflow purpose and setup instructions including API keys and credentials |
| Sticky Note1                       | Sticky Note                           | Configuration step reminder            | None                             | None                                   | # ⚙️ Step 1: Start Here! This is the main configuration node for the workflow.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Node Type: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed.

2. **Create Set Node for Configuration:**  
   - Node Type: Set  
   - Name: `CHANGE THESE: dependencies`  
   - Add the following fields with default or placeholder values:  
     - `wave_api_key` (string) — your WAVE API key  
     - `URL to analyze` (string) — website URL to scan  
     - `Email for summary` (string) — recipient email address  
     - `Desired Output Language` (string) — e.g., "english"  
     - `Company Name` (string) — your company’s name  
     - `Country of the Company (used to apply local law)` (string) — e.g., "Germany"  
   - Connect Manual Trigger node output to this node.

3. **Create HTTP Request Node to Get Website HTML:**  
   - Node Type: HTTP Request  
   - Name: `Get Website HTML`  
   - URL: Use expression `={{ $json["URL to analyze"] }}` from the Set node  
   - Method: GET  
   - Connect from `CHANGE THESE: dependencies`.

4. **Create HTTP Request Node to Get WAVE Report:**  
   - Node Type: HTTP Request  
   - Name: `Get WAVE Report`  
   - URL: Expression:  
     ```
     =https://wave.webaim.org/api/request?key={{ $('CHANGE THESE: dependencies').item.json.wave_api_key }}&reporttype=4&url={{ $('CHANGE THESE: dependencies').item.json['URL to analyze'] }}
     ```  
   - Method: GET  
   - Connect from `Get Website HTML`.

5. **Create Code Node to Map WAVE Items:**  
   - Node Type: Code (JavaScript)  
   - Name: `Map WAVE Report Items to Website selectors.`  
   - Paste the provided JavaScript code that loads the HTML via Cheerio, iterates categories and items from WAVE report, finds selectors, extracts context HTML, and returns an array of issue objects.  
   - Connect from `Get WAVE Report`.

6. **Create LangChain Agent Node for Accessibility Statement Generation:**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `Accessibility Statement Generator`  
   - Prompt: Use the detailed legal prompt including:  
     - Company name, country, URL, and accessibility issues as JSON string  
     - Instructions to generate a clean, professional HTML statement in English compliant with EAA and WCAG 2.1 AA  
     - Placeholder text for contact and enforcement info  
   - Output parser: Enable and configure as Structured Output Parser with schema expecting an object with a string field `"Accessibility Statement"`.  
   - Connect from `Map WAVE Report Items to Website selectors.`.

7. **Create AI Language Model Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Name: `gemini 2.5 pro`  
   - Model: `models/gemini-2.5-pro-preview-06-05`  
   - Credentials: Connect Google Palm API credentials.  
   - Connect AI Language Model input to `Accessibility Statement Generator`.

8. **Create HTML Node to Extract HTML String:**  
   - Node Type: HTML  
   - Name: `Parse output as html`  
   - HTML parameter: Expression `{{ $json.output['Accessibility Statement'] }}`  
   - Connect from `Accessibility Statement Generator`.

9. **Create Move Binary Data Node to Create File:**  
   - Node Type: Move Binary Data  
   - Name: `Create accesibility statement html file`  
   - Mode: jsonToBinary  
   - File name: Expression  
     ```
     accessibility_statement_{{ $('CHANGE THESE: dependencies').item.json['Company Name'].toSnakeCase() }}.html
     ```  
   - MIME type: `text/html`  
   - Source key: `html`  
   - Connect from `Parse output as html`.

10. **Create Gmail Node to Send Email:**  
    - Node Type: Gmail  
    - Name: `Send accessibility statement by email`  
    - Recipient: Expression `={{ $('CHANGE THESE: dependencies').item.json['Email for summary'] }}`  
    - Subject: `Accessibility Statement for {{ $('CHANGE THESE: dependencies').item.json['Company Name'] }}`  
    - Message: Static text "This Email contains your Accessibility Statement. Check the attached files."  
    - Attachments: Use the binary data from previous node  
    - Credentials: Connect Gmail OAuth2 credentials  
    - Connect from `Create accesibility statement html file`.

11. **Add Sticky Notes for User Instructions:**  
    - Create two Sticky Note nodes with the exact content:  
      - One describing the overview and setup steps of the workflow.  
      - One reminding the user to fill in the configuration node before execution.  
    - Position them suitably for user visibility.

12. **Verify and Test:**  
    - Ensure all expressions reference correct nodes by name.  
    - Fill in API keys, URLs, emails, and company info in the Set node.  
    - Connect Google Gemini and Gmail OAuth2 credentials properly.  
    - Run the workflow manually via the trigger node and monitor execution logs for errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is designed to meet the EU’s European Accessibility Act (EAA) compliance deadline of June 28, 2025, automating the generation of legally required accessibility statements.                                        | EU EAA compliance context                                                                       |
| It uses the WAVE API (https://wave.webaim.org/api/) for automated accessibility scanning and Google Gemini AI for natural language generation of the statement.                                                                | WAVE API and Google Gemini references                                                           |
| The Accessibility Statement output includes placeholders for contact emails and national enforcement bodies, which must be manually filled by the company before publication.                                                  | Legal compliance requirement                                                                    |
| The workflow relies on n8n v0.210+ for compatibility with LangChain nodes and Google Gemini integration.                                                                                                                       | Version requirement                                                                             |
| For detailed setup, users should ensure valid API keys, OAuth2 credentials, and correct input parameters. Failure to do so will cause workflow errors or incomplete outputs.                                                    | Setup and troubleshooting advice                                                                |
| The workflow offers extensibility to swap the AI language model node with other providers if needed, by adjusting prompt and credentials accordingly.                                                                          | Flexibility in AI model integration                                                             |
| Workflow instructions and setup guidance are embedded as sticky notes for ease of user onboarding.                                                                                                                            | Embedded user documentation within the workflow                                                 |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---