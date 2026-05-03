Landing Page Analyzing Agent

https://n8nworkflows.xyz/workflows/landing-page-analyzing-agent-7897


# Landing Page Analyzing Agent

### 1. Workflow Overview

The **Landing Page Analyzer** workflow is designed to analyze a user-submitted landing page URL and provide actionable, prioritized Conversion Rate Optimization (CRO) recommendations. It is intended for marketers, CRO specialists, and digital businesses aiming to improve landing page performance without requiring deep development effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the landing page URL via a web form submission.
- **1.2 Landing Page Content Retrieval:** Fetches the HTML content of the submitted landing page URL.
- **1.3 Content Preparation:** Converts raw HTML content into a Markdown format for downstream processing.
- **1.4 AI-Powered CRO Analysis:** Uses a Langchain AI agent with custom instructions to analyze the page content and generate a structured list of improvement tips.
- **1.5 Structured Extraction:** Extracts individual improvement tips from the AI output into separate labeled attributes.
- **1.6 User Feedback Form:** Presents the extracted improvement tips back to the user in a clean, formatted form.

Additional elements include sticky notes providing setup guidance and tutorial references, and an integration with Google Gemini (PaLM) API as the AI language model backend.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures user input — specifically, the URL of the landing page to analyze — via a webhook form trigger node.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger node (webhook)  
    - *Role:* Entry point that listens for HTTP POST requests containing form data.  
    - *Configuration:*  
      - Form Title: "Landing Page Optimizer"  
      - Form Description: "Make Your Page Convert Like a Top 1% Funnel"  
      - Single required field: "Your Landing Page Url" with placeholder "https://example.anything"  
    - *Input:* HTTP request from user submission  
    - *Output:* JSON object containing the submitted URL under key `Your Landing Page Url`  
    - *Error cases:*  
      - Missing or invalid URL input (no validation beyond required field)  
      - Unauthorized or malformed webhook calls  
    - *Version:* 2.2

---

#### 2.2 Landing Page Content Retrieval

- **Overview:**  
  Fetches the HTML content of the landing page URL provided by the user to enable analysis.

- **Nodes Involved:**  
  - HTTP Request  
  - Markdown

- **Node Details:**

  - **HTTP Request**  
    - *Type:* HTTP Request node  
    - *Role:* Performs a GET request to the submitted URL to retrieve raw HTML content.  
    - *Configuration:*  
      - URL: Dynamically set via expression from the form submission input (`={{ $json['Your Landing Page Url'] }}`)  
      - No authentication or additional headers configured (assumes public access)  
    - *Input:* JSON with `Your Landing Page Url`  
    - *Output:* JSON with `data` containing raw HTML response body  
    - *Error cases:*  
      - URL unreachable or returns error codes (404, 500, etc.)  
      - Timeout or network errors  
      - Non-HTML responses or redirects causing unexpected data  
    - *Version:* 4.2

  - **Markdown**  
    - *Type:* Markdown node  
    - *Role:* Converts HTML content retrieved into Markdown format for better AI processing and readability.  
    - *Configuration:*  
      - Input HTML via expression `={{ $json.data }}` from HTTP Request output  
    - *Input:* JSON with `data` field (HTML)  
    - *Output:* Markdown formatted text in JSON  
    - *Error cases:*  
      - Malformed HTML leading to incorrect Markdown conversion  
    - *Version:* 1

---

#### 2.3 AI-Powered CRO Analysis

- **Overview:**  
  Processes the Markdown content using an AI agent that is instructed as a Conversion Rate Optimization expert to generate prioritized and actionable improvement tips.

- **Nodes Involved:**  
  - AI Agent  
  - Gemini 2.5 Flash (AI language model backend)

- **Node Details:**

  - **AI Agent**  
    - *Type:* Langchain Agent node  
    - *Role:* Runs a prompt-based AI analysis on the landing page content to generate CRO recommendations.  
    - *Configuration:*  
      - Text input: Markdown content from previous node (`={{ $json.data }}`)  
      - System prompt: Detailed instructions directing the AI to:  
        - Act as a world-class CRO expert  
        - Provide action-oriented, specific, non-generic advice  
        - Prioritize modern 2024 CRO techniques with minimal implementation effort  
        - Focus on key conversion levers (clarity, relevance, trust, desire, urgency, friction reduction)  
      - Prompt Type: "define" (custom defined prompt)  
    - *Input:* Markdown text  
    - *Output:* Text containing structured numbered improvement tips  
    - *Error cases:*  
      - AI service downtime or quota exceeded  
      - Prompt parsing or generation errors  
      - Unexpected or incomplete AI output  
    - *Version:* 2

  - **Gemini 2.5 Flash**  
    - *Type:* Langchain Google Gemini Chat Model node  
    - *Role:* AI language model backend providing chat completions for the AI Agent node.  
    - *Configuration:*  
      - Credentials: Google Gemini (PaLM) API credentials configured via OAuth/API key  
      - No additional options specified  
    - *Input:* Receives queries from AI Agent node  
    - *Output:* Chat completion responses routed back to AI Agent  
    - *Error cases:*  
      - API authentication failure  
      - Rate limiting or quota issues  
      - Network failures  
    - *Version:* 1

---

#### 2.4 Structured Extraction

- **Overview:**  
  Parses the AI-generated text response to extract five separate labeled improvement tips with relevant emojis, making the data usable for user-friendly presentation.

- **Nodes Involved:**  
  - Information Extractor

- **Node Details:**

  - **Information Extractor**  
    - *Type:* Langchain Information Extractor node  
    - *Role:* Extracts specific structured attributes (Improvement Tips 1-5) from the AI text output.  
    - *Configuration:*  
      - Input text: AI Agent output (`={{ $json.output }}`)  
      - Attributes defined:  
        - Improvement Tip 1 (description: first most important improvement tip without any symbol, add relevant emoji)  
        - Improvement Tip 2  
        - Improvement Tip 3  
        - Improvement Tip 4  
        - Improvement Tip 5  
      - No additional options  
    - *Input:* Raw text from AI Agent  
    - *Output:* JSON object with keys `Improvement Tip 1` through `Improvement Tip 5` containing extracted text  
    - *Error cases:*  
      - AI output format changes causing extraction failures  
      - Missing or incomplete tips in AI response  
    - *Version:* 1.2

---

#### 2.5 User Feedback Form

- **Overview:**  
  Presents the extracted improvement tips back to the user in a formatted completion message within a form interface.

- **Nodes Involved:**  
  - Form

- **Node Details:**

  - **Form**  
    - *Type:* Form node  
    - *Role:* Displays the final results as a user-readable list of 5 improvement tips.  
    - *Configuration:*  
      - Operation: completion  
      - Completion Title: "Here are 5 Things You Should Improve 🧐"  
      - Completion Message:  
        ```
        1. {{ $json.output['Improvement Tip 1'] }}

        2. {{ $json.output['Improvement Tip 2'] }}

        3. {{ $json.output['Improvement Tip 3'] }}

        4. {{ $json.output['Improvement Tip 4'] }}

        5. {{ $json.output['Improvement Tip 5'] }}
        ```  
    - *Input:* Extracted improvement tips JSON from Information Extractor  
    - *Output:* Form response sent to front-end or user interface  
    - *Error cases:*  
      - Missing attributes causing incomplete display  
      - Frontend rendering issues  
    - *Version:* 1

---

#### 2.6 Supportive Nodes - Sticky Notes

- **Sticky Note** (Setup Guide)  
  - Provides detailed setup instructions including Google Gemini API credentials setup, webhook testing, and AI agent prompt customization.  
  - Contains links to Google AI Studio API key acquisition.  
  - Positioned prominently for user reference.

- **Sticky Note1** (Workflow Title)  
  - Simple title card: "Landing Page Analyzer"

- **Sticky Note3** (Tutorial Link)  
  - Embeds a YouTube tutorial thumbnail linking to a step-by-step video on building a similar AI agent workflow.

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                     | Input Node(s)          | Output Node(s)        | Sticky Note                                                     |
|---------------------|-----------------------------------|-----------------------------------|-----------------------|-----------------------|----------------------------------------------------------------|
| On form submission   | Form Trigger                      | Capture landing page URL           | -                     | HTTP Request          |                                                                |
| HTTP Request        | HTTP Request                     | Fetch landing page HTML            | On form submission    | Markdown              |                                                                |
| Markdown            | Markdown                         | Convert HTML to Markdown           | HTTP Request          | AI Agent              |                                                                |
| AI Agent            | Langchain Agent                  | Generate CRO improvement tips      | Markdown              | Information Extractor  |                                                                |
| Gemini 2.5 Flash    | Langchain LM Chat Google Gemini  | AI language model backend          | AI Agent (ai_languageModel) | AI Agent, Information Extractor (ai_languageModel) |                                                                |
| Information Extractor | Langchain Information Extractor | Extract individual tips from AI output | AI Agent              | Form                  |                                                                |
| Form                | Form                            | Display improvement tips to user   | Information Extractor | -                     |                                                                |
| Sticky Note         | Sticky Note                     | Setup guide and instructions       | -                     | -                     | # 🛠 Setup Guide ... [Google AI Studio](https://aistudio.google.com/apikey) |
| Sticky Note1        | Sticky Note                     | Workflow title card                 | -                     | -                     | ## Landing Page Analyzer                                       |
| Sticky Note3        | Sticky Note                     | Tutorial video link                 | -                     | -                     | [I Built an Auto Lead Finder AI Agent](https://youtu.be/gsFl25jw-7I) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and set its name to "Landing Page Analyzer".**

2. **Add a Form Trigger node:**  
   - Name: "On form submission"  
   - Set Form Title: "Landing Page Optimizer"  
   - Add a single form field:  
     - Label: "Your Landing Page Url"  
     - Placeholder: "https://example.anything"  
     - Mark as required  
   - Ensure webhook URL is accessible (note the webhook ID will be auto-generated).

3. **Add an HTTP Request node:**  
   - Name: "HTTP Request"  
   - Method: GET (default)  
   - URL: Set via expression referencing form input: `={{ $json['Your Landing Page Url'] }}`  
   - No authentication or headers needed.  
   - Connect "On form submission" output to "HTTP Request" input.

4. **Add a Markdown node:**  
   - Name: "Markdown"  
   - Set HTML input via expression: `={{ $json.data }}` to convert the fetched HTML to Markdown.  
   - Connect "HTTP Request" output to "Markdown" input.

5. **Add an AI Agent node (Langchain Agent):**  
   - Name: "AI Agent"  
   - Text input: `={{ $json.data }}` (Markdown content)  
   - Set prompt type to "define".  
   - Enter the system message instructing the AI as a CRO expert with the detailed prompt:  

     ```
     You are a world-class Conversion Rate Optimization (CRO) expert.

     Your mission is to analyze my landing page and return with the most important actions the business should take to improve conversions. No fluff, no filler — just what matters most.

     🎯 YOUR RESPONSE MUST FOLLOW THIS STRUCTURE:

     1) Each item should start with a strong action verb (e.g., Replace, Add, Remove, Rewrite, Rearrange, Clarify, Simplify, Make visible, etc.)

     2) Be specific. Tell me exactly what to do — what headline to change, what section to remove, what to add and where.

     3) Every recommendation must be based on the actual landing page content. No generic advice.

     4) Prioritize modern, high-impact CRO ideas relevant to 2024 digital audiences.

     5) Keep it easy to understand and implement. No heavy dev work required.

     6) Focus on conversion levers: clarity, relevance, trust, desire, urgency, and friction reduction.
     ```
   - Connect "Markdown" output to "AI Agent" input.

6. **Add a Google Gemini Chat Model node:**  
   - Name: "Gemini 2.5 Flash"  
   - Configure Google Gemini (PaLM) API credentials in n8n credentials manager.  
   - No additional options needed.  
   - Connect this node as the AI language model backend for the "AI Agent" node (via the AI Agent’s `ai_languageModel` input).

7. **Add an Information Extractor node (Langchain):**  
   - Name: "Information Extractor"  
   - Input text: `={{ $json.output }}` from "AI Agent" output.  
   - Define five attributes to extract:  
     - Improvement Tip 1: Description with relevant emoji, no symbols  
     - Improvement Tip 2  
     - Improvement Tip 3  
     - Improvement Tip 4  
     - Improvement Tip 5  
   - Connect "AI Agent" output to "Information Extractor" input.

8. **Add a Form node:**  
   - Name: "Form"  
   - Operation: completion  
   - Completion Title: "Here are 5 Things You Should Improve 🧐"  
   - Completion Message:  
     ```
     1. {{ $json.output['Improvement Tip 1'] }}

     2. {{ $json.output['Improvement Tip 2'] }}

     3. {{ $json.output['Improvement Tip 3'] }}

     4. {{ $json.output['Improvement Tip 4'] }}

     5. {{ $json.output['Improvement Tip 5'] }}
     ```  
   - Connect "Information Extractor" output to "Form" input.

9. **Add Sticky Notes for documentation (optional):**  
   - Add one with setup instructions, including:  
     - Google Gemini API credential setup (link: https://aistudio.google.com/apikey)  
     - Webhook testing notes  
     - Prompt customization advice  
   - Add a sticky note with the workflow title "Landing Page Analyzer".  
   - Add a sticky note linking to the YouTube tutorial: https://youtu.be/gsFl25jw-7I

10. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Setup Guide includes detailed instructions for configuring Google Gemini API credentials in n8n.                | See sticky note content and https://aistudio.google.com/apikey |
| Workflow is based on a step-by-step YouTube tutorial by Rakin Jakaria.                                          | https://youtu.be/gsFl25jw-7I                              |
| The AI prompt is carefully designed to prioritize actionable, specific, and modern CRO recommendations.        | Embedded in AI Agent node parameters                       |
| Google Gemini (PaLM) API integration is key for AI-powered analysis; ensure credentials and quota are valid.    | Credential setup required in n8n                           |
| The workflow assumes submitted URLs are publicly accessible without authentication.                              | HTTP Request node has no authentication configured         |

---

**Disclaimer:** The provided content is derived exclusively from an automated workflow created with n8n, adhering strictly to applicable content policies and containing no illegal, offensive, or protected elements. All processed data is legal and public.