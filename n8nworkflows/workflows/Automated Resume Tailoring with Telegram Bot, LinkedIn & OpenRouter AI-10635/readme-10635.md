Automated Resume Tailoring with Telegram Bot, LinkedIn & OpenRouter AI

https://n8nworkflows.xyz/workflows/automated-resume-tailoring-with-telegram-bot--linkedin---openrouter-ai-10635


# Automated Resume Tailoring with Telegram Bot, LinkedIn & OpenRouter AI

### 1. Workflow Overview

This workflow automates the tailoring of a resume based on a job description or a LinkedIn job URL, using a Telegram bot as the entry point. It targets job seekers who want to customize their resume precisely for specific roles, leveraging AI to optimize relevance and format. The workflow consists of the following logical blocks:

- **1.1 Telegram Input Reception**: Receives user messages via Telegram, detects if input is a LinkedIn job URL or plain job description text.
- **1.2 LinkedIn Job Data Extraction**: If a LinkedIn URL is provided, extracts the job ID and scrapes the public job listing page for detailed job information.
- **1.3 Resume and Schema Loading**: Loads the candidate's base resume in JSON Resume format from a public URL and prepares the JSON schema for validation.
- **1.4 AI-Powered Resume Tailoring**: Uses OpenRouter’s GPT-4.1 model with a strict system prompt to tailor the resume JSON to the job description, enforcing factual accuracy and schema compliance.
- **1.5 Resume Generation & Delivery**: Sends the tailored resume JSON to an external backend to generate PDF and HTML versions, then delivers the PDF file and download links back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

- **Overview:**  
  Captures incoming Telegram messages that contain either a job description text or a LinkedIn job URL.

- **Nodes Involved:**  
  - OnTelegramMessage  
  - IsLinkedinUrl

- **Node Details:**

  - **OnTelegramMessage**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point to receive messages from users on Telegram.  
    - *Config:* Listens for 'message' updates only, uses Telegram API credential for the bot.  
    - *Connections:* Outputs to IsLinkedinUrl node.  
    - *Edge Cases:* No messages or unsupported message types can halt workflow. Telegram API rate limits or connectivity issues possible.

  - **IsLinkedinUrl**  
    - *Type:* If Node  
    - *Role:* Checks if the incoming message contains a LinkedIn job URL by detecting if message entities include a 'url' type.  
    - *Config:* Condition triggers main output if a URL entity exists; fallback output if not.  
    - *Connections:*  
      - True branch → SetJobLink (to extract URL)  
      - False branch → GetJsonResume (to proceed with direct job description text)  
    - *Edge Cases:* Messages without URLs fall through to false branch. URL detection may fail if message format changes or no entities are detected.

#### 1.2 LinkedIn Job Data Extraction

- **Overview:**  
  Extracts the job ID from the LinkedIn job URL and scrapes the public LinkedIn job listing page to collect job title, location, and description.

- **Nodes Involved:**  
  - SetJobLink  
  - MapJob  
  - GetJobHTML  
  - ParseJobHTML

- **Node Details:**

  - **SetJobLink**  
    - *Type:* Set Node  
    - *Role:* Saves the LinkedIn job URL extracted from the Telegram message for downstream processing.  
    - *Config:* Sets "job_link" field from the Telegram message's link preview URL.  
    - *Connections:* Outputs to MapJob.  
    - *Edge Cases:* If link_preview_options or URL missing, may cause errors.

  - **MapJob**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Parses the job_link string to extract a LinkedIn job ID. Supports query parameter extraction or URL path extraction.  
    - *Config:* Custom JavaScript function parsing "?currentJobId=" or "/jobs/view/{id}" pattern.  
    - *Connections:* Outputs to GetJobHTML.  
    - *Edge Cases:* If URL format changes or job ID missing, returns null; downstream scraping may fail.

  - **GetJobHTML**  
    - *Type:* HTTP Request  
    - *Role:* Fetches the LinkedIn job listing HTML page using the extracted job ID.  
    - *Config:* GET request to https://www.linkedin.com/jobs/view/{{job_id}}  
    - *Connections:* Outputs to ParseJobHTML.  
    - *Edge Cases:* Potential LinkedIn rate limiting, HTTP errors, or blocked requests. Proxy usage is suggested in sticky notes to mitigate.

  - **ParseJobHTML**  
    - *Type:* HTML Extract  
    - *Role:* Extracts job description text, job title, and location from the LinkedIn job page HTML using CSS selectors.  
    - *Config:* Extracts keys "description", "title", and "location" with respective CSS selectors.  
    - *Connections:* Outputs to GetJsonResume node.  
    - *Edge Cases:* If LinkedIn page structure changes, extraction may fail or return empty results.

#### 1.3 Resume and Schema Loading

- **Overview:**  
  Loads the candidate's base resume JSON from a public URL and sets up the JSON schema for validation and AI output enforcement.

- **Nodes Involved:**  
  - GetJsonResume  
  - SetAllFields

- **Node Details:**

  - **GetJsonResume**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the candidate’s resume JSON from a public URL (hosted e.g., as a GitHub Gist).  
    - *Config:* GET request to a fixed URL serving JSON Resume formatted resume.  
    - *Connections:* Outputs to SetAllFields.  
    - *Edge Cases:* URL may become unavailable or return invalid data; network errors possible.

  - **SetAllFields**  
    - *Type:* Set Node  
    - *Role:* Prepares all variables for the AI prompt, including concatenated job description and loaded resume JSON/schema.  
    - *Config:*  
      - "jd": Combines job title, location, and description if LinkedIn URL was detected, else uses plain text message.  
      - "json_resume": Parsed JSON from GetJsonResume's binary data.  
      - "resume_schema": Inline JSON schema defining the allowed resume structure.  
    - *Connections:* Outputs to GetTailoredJsonResume.  
    - *Edge Cases:* Parsing failures if resume JSON is malformed.

#### 1.4 AI-Powered Resume Tailoring

- **Overview:**  
  Applies a strict system prompt via OpenRouter GPT-4.1 model to optimize the resume JSON tailored specifically for the provided job description, ensuring schema compliance and factual accuracy.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - GetTailoredJsonResume  
  - EnsureJsonResumeSchema  
  - SetJsonResume

- **Node Details:**

  - **OpenRouter Chat Model**  
    - *Type:* LangChain OpenRouter LLM Chat Node  
    - *Role:* Invokes GPT-4.1 with a system prompt designed to tailor resumes without introducing fabricated content, outputting strict JSON.  
    - *Config:* Model set to "openai/gpt-4.1", response format "json_object". Uses OpenRouter API credentials.  
    - *Connections:* Outputs to GetTailoredJsonResume.  
    - *Edge Cases:* API key limits, model timeouts, or unexpected AI output formats.

  - **GetTailoredJsonResume**  
    - *Type:* LangChain Chain LLM Node  
    - *Role:* Sends the prompt with inputs (job description, resume JSON, resume schema) to the LLM for resume tailoring.  
    - *Config:* Contains an extensive system prompt detailing immutable behavior, strict JSON output only, security guarantees, and objective to optimize resume for ATS and human recruiters.  
    - *Connections:* Outputs to EnsureJsonResumeSchema.  
    - *Edge Cases:* LLM may produce invalid JSON or fail validation.

  - **EnsureJsonResumeSchema**  
    - *Type:* LangChain Output Parser Structured  
    - *Role:* Validates and auto-fixes the LLM output against the JSON resume schema to guarantee structural compliance.  
    - *Config:* Manual schema definition provided inline, based on JSON Resume specification. Auto-fix enabled.  
    - *Connections:* Outputs back to GetTailoredJsonResume node (feedback loop in LangChain).  
    - *Edge Cases:* If output cannot be fixed, workflow expects a JSON error object as fallback.

  - **SetJsonResume**  
    - *Type:* Set Node  
    - *Role:* Extracts the valid tailored resume JSON from the AI output for use in resume generation.  
    - *Config:* Sets node output to the AI output JSON.  
    - *Connections:* Outputs to GenerateResume node.  
    - *Edge Cases:* Empty or error JSON from AI output will propagate downstream.

#### 1.5 Resume Generation & Delivery

- **Overview:**  
  Sends tailored resume JSON to a backend service to generate PDF and HTML resume versions, then delivers the PDF file and download links to the user via Telegram.

- **Nodes Involved:**  
  - GenerateResume  
  - GetResumePdf  
  - SetFinalFields  
  - Send a document

- **Node Details:**

  - **GenerateResume**  
    - *Type:* HTTP Request  
    - *Role:* Posts the tailored resume JSON to an external backend API endpoint that generates resume views and file URLs.  
    - *Config:* POST to https://thebackend.rocket-champ.pw/resume with JSON body from SetJsonResume.  
    - *Connections:* Outputs to GetResumePdf.  
    - *Edge Cases:* Backend API downtime or unexpected response formats.

  - **GetResumePdf**  
    - *Type:* HTTP Request  
    - *Role:* Fetches the generated PDF file binary from the backend using the URL returned in the previous step.  
    - *Config:* POST request to backend PDF URL with resume JSON in body, expects binary response.  
    - *Connections:* Outputs to SetFinalFields.  
    - *Edge Cases:* Network errors, invalid URLs, or missing PDF data.

  - **SetFinalFields**  
    - *Type:* Set Node  
    - *Role:* Sets final parameters for Telegram message delivery, including chat ID, PDF binary data, and URLs for HTML and PDF downloads.  
    - *Config:*  
      - chat_id from Telegram message sender ID  
      - data (binary) from GetResumePdf response  
      - linkedin job URL for reference  
      - html and pdf URLs from backend generate resume response  
    - *Connections:* Outputs to Send a document node.  
    - *Edge Cases:* Missing fields or binary data can cause Telegram send failure.

  - **Send a document**  
    - *Type:* Telegram Node  
    - *Role:* Sends the generated PDF document to the user via Telegram, with inline keyboard buttons linking to HTML and PDF versions.  
    - *Config:*  
      - chatId dynamically set from SetFinalFields  
      - Operation: sendDocument with binary data  
      - Inline keyboard with buttons "HTML" and "PDF" linking to respective URLs  
      - Caption: "Your resume sir."  
      - Uses Telegram API credential for bot.  
    - *Edge Cases:* Telegram API limits, invalid binary data, or message size limits.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                                | Input Node(s)          | Output Node(s)           | Sticky Note                                                                              |
|-----------------------|----------------------------------|-----------------------------------------------|-----------------------|--------------------------|------------------------------------------------------------------------------------------|
| OnTelegramMessage      | Telegram Trigger                 | Entry point receiving Telegram messages       | -                     | IsLinkedinUrl             | Create a telegram bot using BotFather bot and add credentials for it.                    |
| IsLinkedinUrl          | If Node                         | Checks if input message contains LinkedIn URL | OnTelegramMessage      | SetJobLink / GetJsonResume|                                                                                          |
| SetJobLink             | Set Node                        | Stores LinkedIn job URL from Telegram message | IsLinkedinUrl          | MapJob                   |                                                                                          |
| MapJob                 | Code Node                      | Extracts job ID from LinkedIn URL              | SetJobLink             | GetJobHTML               |                                                                                          |
| GetJobHTML             | HTTP Request                   | Fetches LinkedIn job listing HTML              | MapJob                 | ParseJobHTML             | You Can Use a Proxy - This node may sometimes not successfully retrieve the page due to rate-limiting. Proxy recommended. |
| ParseJobHTML           | HTML Extract                   | Extracts job title, location, description      | GetJobHTML             | GetJsonResume            |                                                                                          |
| GetJsonResume          | HTTP Request                   | Loads candidate's base JSON resume              | IsLinkedinUrl / ParseJobHTML | SetAllFields        | Get The Resume From Where You Host It - Set URL to your JSON Resume formatted resume.    |
| SetAllFields           | Set Node                      | Prepares combined input fields for AI prompt   | GetJsonResume          | GetTailoredJsonResume    |                                                                                          |
| OpenRouter Chat Model  | LangChain LLM Chat Node        | Invokes GPT-4.1 model for resume tailoring     | (internal AI chain)    | GetTailoredJsonResume    | Setup OpenRouter - Requires OpenRouter API token and credential setup.                   |
| GetTailoredJsonResume  | LangChain Chain LLM Node       | Sends system prompt with inputs to LLM         | OpenRouter Chat Model  | EnsureJsonResumeSchema    |                                                                                          |
| EnsureJsonResumeSchema | LangChain Output Parser Node   | Validates AI output against resume schema      | GetTailoredJsonResume  | GetTailoredJsonResume    |                                                                                          |
| SetJsonResume          | Set Node                      | Extracts tailored resume JSON from AI output    | GetTailoredJsonResume  | GenerateResume           |                                                                                          |
| GenerateResume         | HTTP Request                   | Sends tailored resume JSON to backend for PDF/HTML generation | SetJsonResume          | GetResumePdf             |                                                                                          |
| GetResumePdf           | HTTP Request                   | Downloads generated PDF binary from backend    | GenerateResume         | SetFinalFields           |                                                                                          |
| SetFinalFields         | Set Node                      | Sets Telegram message parameters for delivery  | GetResumePdf           | Send a document          |                                                                                          |
| Send a document        | Telegram Node                  | Sends PDF document and links to user on Telegram | SetFinalFields         | -                        | Get Back A Tailored Resume - Sends PDF with links to HTML and PDF hosted on backend.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot Credentials**  
   - Use BotFather in Telegram to create a bot and obtain API token.  
   - Add the Telegram API credential in n8n.

2. **Create Telegram Trigger**  
   - Node: OnTelegramMessage (Telegram Trigger)  
   - Configure to listen for 'message' updates.  
   - Select the Telegram API credential.

3. **Add If Node to Detect LinkedIn URL**  
   - Node: IsLinkedinUrl (If)  
   - Condition: Check if message entities contain a URL type.  
   - True output: Proceed to extract LinkedIn job link.  
   - False output: Proceed to load resume and use plain job description text.

4. **Set LinkedIn Job Link**  
   - Node: SetJobLink (Set)  
   - Set field "job_link" to `={{ $json.message.link_preview_options.url }}`.

5. **Extract Job ID from URL**  
   - Node: MapJob (Code)  
   - JavaScript code to extract job ID from LinkedIn URL either via query param or path regex.

6. **Download LinkedIn Job HTML Page**  
   - Node: GetJobHTML (HTTP Request)  
   - GET request to `https://www.linkedin.com/jobs/view/{{ $json.job_id }}`.

   - Optional: Configure proxy for this node if LinkedIn rate limits.

7. **Parse Job HTML Content**  
   - Node: ParseJobHTML (HTML Extract)  
   - Extract keys:  
     - description: CSS selector `div.description__text.description__text--rich`  
     - title: CSS selector `head title`  
     - location: CSS selector `.sub-nav-cta__meta-text`

8. **Retrieve Candidate Resume JSON**  
   - Node: GetJsonResume (HTTP Request)  
   - GET request to your publicly hosted JSON Resume URL (e.g., GitHub Gist).

9. **Set All Variables for AI Input**  
   - Node: SetAllFields (Set)  
   - Assign:  
     - jd: Combine LinkedIn job title, location, description if LinkedIn URL used, else use plain Telegram message text.  
     - json_resume: Parse the JSON resume from GetJsonResume.  
     - resume_schema: Paste the JSON Resume schema inline.

10. **Configure OpenRouter API Credentials**  
    - Obtain OpenRouter API token and add credential in n8n.

11. **Set up OpenRouter Chat Model Node**  
    - Node: OpenRouter Chat Model (LangChain LLM Chat)  
    - Model: openai/gpt-4.1  
    - Options: responseFormat = json_object  
    - Connect to GetTailoredJsonResume node.

12. **Create GetTailoredJsonResume Node**  
    - Node: GetTailoredJsonResume (LangChain Chain LLM)  
    - Use the provided fixed system prompt enforcing immutable resume tailoring rules and outputting strict JSON.  
    - Inputs: JOB_DESCRIPTION, RESUME_JSON, RESUME_SCHEMA.  
    - Connect output to EnsureJsonResumeSchema.

13. **Add EnsureJsonResumeSchema Node**  
    - Node: EnsureJsonResumeSchema (LangChain Output Parser Structured)  
    - Paste JSON Resume schema inline as manual schema.  
    - Enable autoFix.  
    - Connect output back to GetTailoredJsonResume for validation.

14. **Extract Tailored Resume JSON**  
    - Node: SetJsonResume (Set)  
    - Set output json to `={{ $json.output }}` from AI node.

15. **Send Tailored Resume to Backend**  
    - Node: GenerateResume (HTTP Request)  
    - POST to `https://thebackend.rocket-champ.pw/resume`  
    - JSON body: `={{ $node["SetJsonResume"].json }}`.

16. **Download Generated PDF**  
    - Node: GetResumePdf (HTTP Request)  
    - POST to `https://thebackend.rocket-champ.pw{{ $json.pdfUrl }}` with resume JSON in body.  
    - Expect binary PDF response.

17. **Prepare Telegram Output Fields**  
    - Node: SetFinalFields (Set)  
    - Assign:  
      - chat_id: `={{ $node["OnTelegramMessage"].json.message.from.id }}`  
      - data: binary from GetResumePdf  
      - linkedin: `https://linkedin.com/jobs/view/{{ $node["SetAllFields"].json.job_id }}`  
      - html: `https://thebackend.rocket-champ.pw{{ $node["GenerateResume"].json.viewUrl }}`  
      - pdf: `https://thebackend.rocket-champ.pw{{ $node["GenerateResume"].json.pdfUrl }}`

18. **Send PDF Document via Telegram**  
    - Node: Send a document (Telegram)  
    - chatId: from SetFinalFields.chat_id  
    - Operation: sendDocument  
    - Binary property: data (PDF binary)  
    - Inline keyboard with buttons:  
      - Text: "HTML", URL: from SetFinalFields.html  
      - Text: "PDF", URL: from SetFinalFields.pdf  
    - Caption: "Your resume sir."  
    - Use Telegram API credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Setup Telegram Bot by creating it with BotFather and configuring API credentials in n8n.                                                                                                                                         | Telegram Bot setup instruction                                                                           |
| Proxy usage recommended for LinkedIn scraping to avoid IP rate-limiting; OxyLabs offers free proxies.                                                                                                                            | Proxy recommendation for HTTP Request node                                                               |
| Host your candidate resume JSON in JSON Resume format publicly, e.g., via a GitHub Gist for easy access.                                                                                                                         | https://jsonresume.org/schema                                                                             |
| OpenRouter API token required; create credential in n8n to connect to GPT-4.1 model.                                                                                                                                             | https://openrouter.ai/                                                                                     |
| Backend service for resume generation is open-source and self-hostable: https://github.com/daniel-iliesh/nest-thebackend                                                                                                          | The Backend project for resume generation                                                                 |
| Workflow demonstrates immutable LLM system prompt design to guarantee factual, schema-valid, and concise resume tailoring without hallucinations or prompt injection vulnerabilities.                                              | System prompt embedded in GetTailoredJsonResume node                                                      |
| Generated resume PDF and HTML are hosted on an external backend; Telegram user receives direct PDF file plus inline keyboard links for sharing or downloading.                                                                    | Delivery design                                                                                           |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, respecting all applicable content policies and containing no illegal or offensive material. All processed data is legal and publicly available.