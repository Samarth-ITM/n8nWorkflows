Weekly Meal Planner: AI-Generated Grocery Lists with Price Comparison to WhatsApp

https://n8nworkflows.xyz/workflows/weekly-meal-planner--ai-generated-grocery-lists-with-price-comparison-to-whatsapp-8241


# Weekly Meal Planner: AI-Generated Grocery Lists with Price Comparison to WhatsApp

### 1. Workflow Overview

This workflow automates the generation of a weekly meal plan and grocery list, leveraging AI to create structured meal plans from user input and converting these plans into a PDF document for distribution. The PDF is intended to be shared via messaging platforms such as Telegram (though the sending node is not present in the current JSON but implied by the workflow name). The workflow primarily serves users who want to streamline meal planning by utilizing AI-generated suggestions and formatted outputs.

Logical blocks:

- **1.1 Input Reception:** Triggering the workflow and fetching user-submitted data via an HTTP request.
- **1.2 AI Processing:** Preparing a prompt based on the input, sending it to OpenAI's Chat API, and receiving a meal plan.
- **1.3 Data Validation & Shaping:** Parsing and validating the AI response to ensure it conforms to the desired JSON schema.
- **1.4 PDF Generation:** Transforming the validated plan into an HTML format, then converting this HTML into a PDF document via an external PDF service.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initiates the workflow manually and retrieves the user-submitted meal planning details via an HTTP request.
- **Nodes Involved:** Trigger: Weekly Meal Workflow, Fetch Fillout Submission (HTTP)
  
**Node Details:**

- **Trigger: Weekly Meal Workflow**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual activation; no parameters required.  
  - Inputs: None  
  - Outputs: Triggers the HTTP request node.  
  - Edge Cases: None specific; manual trigger requires user action.  
   
- **Fetch Fillout Submission (HTTP)**  
  - Type: HTTP Request  
  - Role: Retrieves form submission data from a Fillout form or similar source (details not specified).  
  - Configuration: Likely a GET request with authorization headers (credentials referenced in metadata).  
  - Inputs: Trigger node output  
  - Outputs: Raw submission data for prompt preparation.  
  - Edge Cases: Could fail due to network errors, invalid credentials, or malformed HTTP response.

#### 2.2 AI Processing

- **Overview:** Prepares the prompt for OpenAI, sends it via HTTP request, and obtains an AI-generated weekly meal plan response.
- **Nodes Involved:** Prep Prompt from Fillout, OpenAI Chat (HTTP)
  
**Node Details:**

- **Prep Prompt from Fillout**  
  - Type: Code (JavaScript)  
  - Role: Extracts relevant fields from the Fillout submission and formats a prompt tailored for OpenAI Chat completion.  
  - Key Logic: Dynamically builds the prompt text using submission data, ensuring it meets OpenAI input requirements.  
  - Inputs: HTTP request output with form data  
  - Outputs: Structured prompt text for OpenAI  
  - Edge Cases: Expression errors if expected fields are missing or malformed input data.  

- **OpenAI Chat (HTTP)**  
  - Type: HTTP Request  
  - Role: Sends prepared prompt to OpenAI Chat Completion API using HTTP, retrieving AI-generated meal plan.  
  - Configuration: POST request with OpenAI API key (credential: openAiApi), prompt in request body, model selection likely present.  
  - Inputs: Prompt text from code node  
  - Outputs: Raw AI response JSON  
  - Edge Cases: API errors (rate limits, auth errors), network failures, invalid prompt causing malformed responses.

#### 2.3 Data Validation & Shaping

- **Overview:** Parses and verifies the AI response to ensure the meal plan adheres to the expected JSON format, reshaping if necessary for downstream processing.
- **Nodes Involved:** Validate & Shape Plan
  
**Node Details:**

- **Validate & Shape Plan**  
  - Type: Code (JavaScript)  
  - Role: Validates JSON structure of AI output, extracts relevant fields, and ensures compatibility with PDF generation requirements.  
  - Key Logic: JSON parsing, error handling for invalid JSON, data normalization.  
  - Inputs: AI response JSON  
  - Outputs: Clean, validated structured plan data  
  - Edge Cases: Invalid JSON, missing or unexpected fields, runtime exceptions in code.

#### 2.4 PDF Generation

- **Overview:** Converts the validated meal plan data into an HTML format suitable for PDF conversion, then calls an external API to generate a PDF document.
- **Nodes Involved:** Build HTML for PDF, PDF4me: HTML to PDF
  
**Node Details:**

- **Build HTML for PDF**  
  - Type: Code (JavaScript)  
  - Role: Constructs an HTML document from the structured meal plan data, formatting it for readability and print layout.  
  - Inputs: Validated plan data  
  - Outputs: HTML string representing the weekly meal plan  
  - Edge Cases: HTML injection issues if data is not sanitized, malformed HTML templates.  

- **PDF4me: HTML to PDF**  
  - Type: HTTP Request  
  - Role: Calls PDF4me API to convert the HTML string into a PDF document.  
  - Configuration: POST request with HTML content, API key likely configured in credentials (httpHeaderAuth).  
  - Inputs: HTML content from previous node  
  - Outputs: PDF file binary or link (exact usage downstream implied but not shown)  
  - Edge Cases: API errors, HTML too large or complex, network failures.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                      | Input Node(s)                  | Output Node(s)              | Sticky Note |
|-----------------------------|---------------------|------------------------------------|-------------------------------|-----------------------------|-------------|
| Trigger: Weekly Meal Workflow | Manual Trigger      | Starts workflow manually            | None                          | Fetch Fillout Submission (HTTP) |             |
| Fetch Fillout Submission (HTTP) | HTTP Request       | Retrieves user form submission      | Trigger: Weekly Meal Workflow  | Prep Prompt from Fillout     |             |
| Prep Prompt from Fillout      | Code                | Prepares AI prompt from form data  | Fetch Fillout Submission (HTTP) | OpenAI Chat (HTTP)          |             |
| OpenAI Chat (HTTP)            | HTTP Request        | Sends prompt to OpenAI, gets response | Prep Prompt from Fillout      | Validate & Shape Plan        |             |
| Validate & Shape Plan         | Code                | Parses and validates AI response   | OpenAI Chat (HTTP)             | Build HTML for PDF           |             |
| Build HTML for PDF            | Code                | Generates HTML from validated plan | Validate & Shape Plan          | PDF4me: HTML to PDF          |             |
| PDF4me: HTML to PDF           | HTTP Request        | Converts HTML to PDF document      | Build HTML for PDF             | None                        |             |
| Sticky Note                  | Sticky Note          | (Empty content)                    | None                          | None                        |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "Trigger: Weekly Meal Workflow"  
   - Type: Manual Trigger  
   - No parameters needed.  

2. **Create HTTP Request Node to Fetch Submission**  
   - Name: "Fetch Fillout Submission (HTTP)"  
   - Type: HTTP Request  
   - HTTP Method: GET (assumed)  
   - URL: Set to the Fillout form submission API endpoint  
   - Authentication: Use credential "httpHeaderAuth" (configured with API key or token)  
   - Connect output of Manual Trigger to this node.  

3. **Create Code Node to Prepare OpenAI Prompt**  
   - Name: "Prep Prompt from Fillout"  
   - Type: Code  
   - Language: JavaScript  
   - Input: Data from HTTP Request node  
   - Logic: Extract necessary form fields, build a prompt string that clearly specifies the requirements for the weekly meal plan.  
   - Connect output of HTTP Request node to this node.  

4. **Create HTTP Request Node for OpenAI Chat API**  
   - Name: "OpenAI Chat (HTTP)"  
   - Type: HTTP Request  
   - HTTP Method: POST  
   - URL: OpenAI Chat Completion endpoint (e.g., https://api.openai.com/v1/chat/completions)  
   - Authentication: Use "openAiApi" credential (API key)  
   - Request Body: Include model, temperature, and the prompt from the previous node in the required JSON structure.  
   - Connect output of Code node to this node.  

5. **Create Code Node to Validate and Shape AI Response**  
   - Name: "Validate & Shape Plan"  
   - Type: Code  
   - Language: JavaScript  
   - Input: AI response JSON from OpenAI node  
   - Logic: Parse the response, check for valid JSON, extract meal plan details, handle exceptions.  
   - Connect output of OpenAI HTTP node to this node.  

6. **Create Code Node to Build HTML**  
   - Name: "Build HTML for PDF"  
   - Type: Code  
   - Language: JavaScript  
   - Input: Validated plan data  
   - Logic: Construct a clean, styled HTML string representing the weekly meal plan layout for PDF conversion.  
   - Connect output of Validation node to this node.  

7. **Create HTTP Request Node for PDF4me Conversion**  
   - Name: "PDF4me: HTML to PDF"  
   - Type: HTTP Request  
   - HTTP Method: POST  
   - URL: PDF4me API endpoint for HTML to PDF conversion  
   - Authentication: Use "httpHeaderAuth" credential (API key for PDF4me)  
   - Request Body: Include the HTML content from previous node.  
   - Connect output of HTML build node to this node.  

8. **(Optional) Add Output or Messaging Node**  
   - Not present in current workflow JSON but implied by workflow name (Telegram).  
   - Add Telegram or other messaging node to send the generated PDF.  
   - Use appropriate credentials (telegramApi).  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                     |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------|
| Workflow created by n8n Template Creator                                                      | Metadata author information                         |
| Credentials used: httpHeaderAuth (for Fillout & PDF4me), openAiApi (OpenAI), telegramApi (implied) | Credential references in workflow metadata         |
| Workflow name suggests integration with Telegram for PDF delivery, but node not present here. | Possible extension to add in production             |
| PDF generation uses PDF4me API, a third-party service for HTML to PDF conversion.             | https://www.pdf4me.com/en/                          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.