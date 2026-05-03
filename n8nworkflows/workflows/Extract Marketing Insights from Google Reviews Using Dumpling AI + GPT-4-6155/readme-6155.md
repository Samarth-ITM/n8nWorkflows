Extract Marketing Insights from Google Reviews Using Dumpling AI + GPT-4

https://n8nworkflows.xyz/workflows/extract-marketing-insights-from-google-reviews-using-dumpling-ai---gpt-4-6155


# Extract Marketing Insights from Google Reviews Using Dumpling AI + GPT-4

### 1. Workflow Overview

This workflow automates the extraction of marketing insights from Google reviews of a specified business using Dumpling AI and GPT-4 via LangChain agents. It is designed for marketers, product teams, and brand strategists who want to conduct voice-of-customer (VOC) analysis and generate actionable intelligence from customer feedback.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Collects business name and Google Place ID from a user-submitted form.
- **1.2 Review Fetching:** Uses Dumpling AI API to fetch up to 30 Google reviews for the specified place.
- **1.3 Data Preparation:** Splits the fetched review data and aggregates review texts for processing.
- **1.4 AI Analysis:** Applies a GPT-4 powered LangChain Agent to analyze reviews and extract structured marketing insights.
- **1.5 Output Parsing and Storage:** Parses the AI output into structured fields and saves the insights into a Google Sheet for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Collects key input parameters (Business Name and Place ID) from a user via a web form to initiate the workflow.

- **Nodes Involved:**  
  - Submit Business Name + Place ID

- **Node Details:**

  - **Submit Business Name + Place ID**  
    - Type: Form Trigger (n8n-nodes-base.formTrigger)  
    - Role: Entry point node that exposes a web form with two fields: Business name and Place ID.  
    - Configuration:  
      - Form title: "Google Review"  
      - Fields: "Business name" (string), "Place ID" (string)  
      - Webhook ID assigned for external access.  
    - Inputs: None (trigger node)  
    - Outputs: Emits form data as JSON on submission.  
    - Edge Cases:  
      - Missing or invalid Place ID or Business Name inputs may result in empty downstream data or API errors. Input validation is not explicitly configured here.  
      - Network or webhook unavailability may block form submission.  

#### 2.2 Review Fetching

- **Overview:**  
  Uses Dumpling AI’s Google Reviews API to retrieve the latest 30 reviews for the specified Place ID.

- **Nodes Involved:**  
  - 🔎 Dumpling AI: Fetch Google Reviews  
  - 🧮 Split: Each Review

- **Node Details:**

  - **🔎 Dumpling AI: Fetch Google Reviews**  
    - Type: HTTP Request (n8n-nodes-base.httpRequest)  
    - Role: Calls Dumpling AI API’s endpoint to fetch Google reviews for the given Place ID.  
    - Configuration:  
      - Method: POST  
      - URL: https://app.dumplingai.com/api/v1/get-google-reviews  
      - Body (JSON):  
        ```json
        {
          "placeId": "{{ $json['Place ID'] }}",
          "reviews": "30"
        }
        ```  
      - Authentication: HTTP Header Auth using stored Dumpling AI credentials.  
    - Inputs: JSON containing Place ID from form trigger.  
    - Outputs: JSON response including an array of reviews under `items`.  
    - Edge Cases:  
      - API rate limiting or auth failures may cause errors.  
      - Invalid Place ID may return empty or error responses.  
      - Network timeouts.  

  - **🧮 Split: Each Review**  
    - Type: Split Out (n8n-nodes-base.splitOut)  
    - Role: Splits the array of reviews in the `items` field into individual review items for downstream processing.  
    - Configuration: Splits on `items` field from Dumpling AI response.  
    - Inputs: JSON with array of reviews.  
    - Outputs: Multiple items, each representing a single review.  
    - Edge Cases:  
      - Empty `items` array leads to no outputs.  

#### 2.3 Data Preparation

- **Overview:**  
  Aggregates all individual review texts back into one consolidated dataset for AI analysis.

- **Nodes Involved:**  
  - 📦 Aggregate: Merge All Review Texts

- **Node Details:**

  - **📦 Aggregate: Merge All Review Texts**  
    - Type: Aggregate (n8n-nodes-base.aggregate)  
    - Role: Combines all split review items into a single array containing only the `review_text` field.  
    - Configuration:  
      - Aggregate mode: Aggregate all item data  
      - Fields to include: `review_text` only  
    - Inputs: Individual review items from split node.  
    - Outputs: Single item with aggregated review_text array.  
    - Edge Cases:  
      - If any review lacks `review_text`, that entry may be missing or null in aggregation.  
      - Empty input results in empty aggregated list.  

#### 2.4 AI Analysis

- **Overview:**  
  Uses GPT-4 via a LangChain Agent to analyze all aggregated reviews and extract actionable marketing insights under defined categories.

- **Nodes Involved:**  
  - 🧠 LangChain Tools (for Agents)  
  - 🔌 GPT-4 Model (used in Agent)  
  - 🤖 GPT-4: Extract Marketing Insights

- **Node Details:**

  - **🧠 LangChain Tools (for Agents)**  
    - Type: LangChain ToolThink Node (@n8n/n8n-nodes-langchain.toolThink)  
    - Role: Provides toolset support for LangChain agents; linked as AI tool input to the LangChain Agent.  
    - Inputs: None directly; connected as tool to Agent node.  
    - Outputs: Passes tool data to Agent.  
    - Edge Cases: Minimal; depends on LangChain runtime.  

  - **🔌 GPT-4 Model (used in Agent)**  
    - Type: LangChain LM Chat OpenAI (@n8n/n8n-nodes-langchain.lmChatOpenAi)  
    - Role: Provides GPT-4 language model to the LangChain Agent for generating insights.  
    - Configuration:  
      - Model: GPT-4 (gpt-4o)  
      - Credentials: OpenAI API key configured with OAuth or API key.  
    - Inputs: None directly; connected as language model to Agent node.  
    - Outputs: Model responses to Agent.  
    - Edge Cases:  
      - API quota or rate limit errors possible.  
      - Network or auth failures.  
      - Latency/timeouts on large inputs.  

  - **🤖 GPT-4: Extract Marketing Insights**  
    - Type: LangChain Agent (@n8n/n8n-nodes-langchain.agent)  
    - Role: Core AI analysis node that sends aggregated reviews to GPT-4 with a detailed prompt requesting structured marketing insights.  
    - Configuration:  
      - Prompt instructs AI to analyze reviews and return:  
        - Marketing Angles  
        - Customer Motivations  
        - Frictions & Barriers  
        - Product Opportunities  
        - Voice of Customer (VOC) Snippets (3-5 quotes)  
      - System message sets role: "Senior Marketing Strategist AI" with emphasis on actionable insights from customer reviews.  
      - Input text includes JSON-stringified review data.  
      - Uses AI tool and AI language model inputs.  
      - Output parser enabled for structured output.  
    - Inputs: Aggregated review texts and LangChain tool/model nodes.  
    - Outputs: Structured JSON insights.  
    - Edge Cases:  
      - Large input size may hit token limits; truncation or batching might be necessary for extensive reviews.  
      - Potential unexpected AI output format; however, output parser mitigates this.  
      - API failures or timeouts.  

#### 2.5 Output Parsing and Storage

- **Overview:**  
  Parses the structured AI output into clearly defined fields and appends or updates a Google Sheet with the extracted marketing insights for record-keeping and further analysis.

- **Nodes Involved:**  
  - 📊 Parse: Format Insights for Output  
  - 📄 Google Sheets: Save Insights

- **Node Details:**

  - **📊 Parse: Format Insights for Output**  
    - Type: LangChain Output Parser Structured (@n8n/n8n-nodes-langchain.outputParserStructured)  
    - Role: Parses AI output JSON into a defined schema with arrays of strings for each insight category.  
    - Configuration:  
      - Manual schema specifying arrays of strings for fields: marketingAngles, customerMotivations, frictionsAndBarriers, productOpportunities, voiceOfCustomerSnippets.  
    - Inputs: AI agent output JSON.  
    - Outputs: Parsed JSON ready for mapping to Google Sheets.  
    - Edge Cases:  
      - If AI output deviates from schema, parsing may fail or produce incomplete data.  

  - **📄 Google Sheets: Save Insights**  
    - Type: Google Sheets (n8n-nodes-base.googleSheets)  
    - Role: Appends or updates a Google Sheet with the marketing insights linked to the business name and Place ID.  
    - Configuration:  
      - Operation: Append or Update (matching on Place ID)  
      - Mapping:  
        - Place ID from form input  
        - Business Name from form input  
        - Marketing Angles, Customer Motivations, Product Opportunities, Frictions and Barriers, VOC Snippets from parsed AI output (joined as text blocks separated by double newlines)  
      - Credentials: Google OAuth2 API configured.  
      - Sheet: Specified by document ID and sheet name (gid=0)  
    - Inputs: Parsed insight JSON and form data for business info.  
    - Outputs: Confirmation of sheet update.  
    - Edge Cases:  
      - Google API auth failure or quota limits.  
      - Data type mismatches or empty fields.  
      - Sheet schema mismatches breaking column mapping.

---

### 3. Summary Table

| Node Name                        | Node Type                             | Functional Role                     | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                      |
|---------------------------------|-------------------------------------|-----------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Submit Business Name + Place ID  | Form Trigger                        | Collect business name and Place ID | None                             | 🔎 Dumpling AI: Fetch Google Reviews |                                                                                               |
| 🔎 Dumpling AI: Fetch Google Reviews | HTTP Request                       | Fetch Google reviews from API      | Submit Business Name + Place ID  | 🧮 Split: Each Review             |                                                                                               |
| 🧮 Split: Each Review            | Split Out                          | Split reviews array into individual reviews | 🔎 Dumpling AI: Fetch Google Reviews | 📦 Aggregate: Merge All Review Texts |                                                                                               |
| 📦 Aggregate: Merge All Review Texts | Aggregate                         | Aggregate all review texts         | 🧮 Split: Each Review            | 🤖 GPT-4: Extract Marketing Insights |                                                                                               |
| 🧠 LangChain Tools (for Agents)  | LangChain ToolThink                | Provides AI tools for analysis     | None                             | 🤖 GPT-4: Extract Marketing Insights |                                                                                               |
| 🔌 GPT-4 Model (used in Agent)    | LangChain LM Chat OpenAI           | GPT-4 language model               | None                             | 🤖 GPT-4: Extract Marketing Insights |                                                                                               |
| 🤖 GPT-4: Extract Marketing Insights | LangChain Agent                   | Analyze reviews and extract insights | 📦 Aggregate: Merge All Review Texts, 🧠 LangChain Tools, 🔌 GPT-4 Model (used in Agent) | 📊 Parse: Format Insights for Output |                                                                                               |
| 📊 Parse: Format Insights for Output | LangChain Output Parser Structured | Parse AI output into structured JSON | 🤖 GPT-4: Extract Marketing Insights | 📄 Google Sheets: Save Insights |                                                                                               |
| 📄 Google Sheets: Save Insights   | Google Sheets                     | Save marketing insights to sheet  | 📊 Parse: Format Insights for Output, Submit Business Name + Place ID | None                            |                                                                                               |
| Sticky Note                     | Sticky Note                       | Documentation note                | None                             | None                            | **📊 Google Review → Marketing Insight Extractor** Workflow inputs business name & Place ID, fetches 30 reviews, analyzes with GPT-4 + LangChain Agent, extracts marketing insights, and logs to Google Sheets. Tools: Dumpling AI, GPT-4, Google Sheets. Good for VOC analysis by marketing/product teams. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: Submit Business Name + Place ID  
   - Type: Form Trigger  
   - Configure form with two fields:  
     - Business name (string)  
     - Place ID (string)  
   - Assign webhook ID for external trigger.  

2. **Add an HTTP Request Node for Dumpling AI**  
   - Name: 🔎 Dumpling AI: Fetch Google Reviews  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/get-google-reviews`  
   - Body Content-Type: JSON  
   - Body JSON:  
     ```json
     {
       "placeId": "{{ $json['Place ID'] }}",
       "reviews": "30"
     }
     ```  
   - Authentication: HTTP Header Auth (configure credentials with Dumpling AI API key)  
   - Connect input from Submit Business Name + Place ID node.  

3. **Add a Split Out Node**  
   - Name: 🧮 Split: Each Review  
   - Type: Split Out  
   - Split field: `items` (array of reviews from Dumpling AI response)  
   - Connect input from Dumpling AI HTTP Request node output.  

4. **Add an Aggregate Node**  
   - Name: 📦 Aggregate: Merge All Review Texts  
   - Type: Aggregate  
   - Aggregate mode: Aggregate all item data  
   - Fields to include: `review_text` only  
   - Connect input from Split Out node output.  

5. **Add LangChain ToolThink Node**  
   - Name: 🧠 LangChain Tools (for Agents)  
   - Type: LangChain ToolThink  
   - No special config required  
   - This node will be connected as AI tool input to the LangChain Agent node.  

6. **Add LangChain LM Chat OpenAI Node**  
   - Name: 🔌 GPT-4 Model (used in Agent)  
   - Type: LangChain LM Chat OpenAI  
   - Model: gpt-4o (GPT-4)  
   - Credentials: Configure with valid OpenAI API key or OAuth2  
   - Connect output to LangChain Agent as AI language model.  

7. **Add LangChain Agent Node**  
   - Name: 🤖 GPT-4: Extract Marketing Insights  
   - Type: LangChain Agent  
   - Configure prompt (define prompt type):  
     - Text prompt includes instructions to analyze Google reviews for marketing insights, specifying categories: marketing angles, customer motivations, frictions/barriers, product opportunities, VOC snippets.  
     - Embed aggregated review data as JSON string: `{{ JSON.stringify($json.data) }}` or equivalent expression.  
   - Configure system message to define AI role as Senior Marketing Strategist.  
   - Enable output parser (for structured output).  
   - Connect AI tool input from LangChain Tools node.  
   - Connect AI language model input from GPT-4 Model node.  
   - Connect main input from Aggregate node output.  

8. **Add LangChain Output Parser Structured Node**  
   - Name: 📊 Parse: Format Insights for Output  
   - Type: LangChain Output Parser Structured  
   - Define manual schema with fields:  
     - marketingAngles (array of strings)  
     - customerMotivations (array of strings)  
     - frictionsAndBarriers (array of strings)  
     - productOpportunities (array of strings)  
     - voiceOfCustomerSnippets (array of strings)  
   - Connect input from LangChain Agent node output.  

9. **Add Google Sheets Node**  
   - Name: 📄 Google Sheets: Save Insights  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Configure Google Sheets OAuth2 credentials.  
   - Document ID: Set to your target Google Sheet document ID.  
   - Sheet Name: Use sheet ID or name (e.g., "gid=0")  
   - Map columns:  
     - Place ID: `={{ $('Submit Business Name + Place ID').item.json['Place ID'] }}`  
     - Business Name: `={{ $('Submit Business Name + Place ID').item.json['Business name'] }}`  
     - marketing Angles: Join array from parsed output with `\n\n` separator  
     - customer Motivations: Join array similarly  
     - frictions And Barriers: Join array similarly  
     - product Opportunities: Join array similarly  
     - voice Of Customer Snippets: Join array similarly  
   - Connect input from Output Parser node.  

10. **Connect workflow nodes in the following order:**  
    Submit Business Name + Place ID → 🔎 Dumpling AI: Fetch Google Reviews → 🧮 Split: Each Review → 📦 Aggregate: Merge All Review Texts → 🤖 GPT-4: Extract Marketing Insights → 📊 Parse: Format Insights for Output → 📄 Google Sheets: Save Insights  

11. **Ensure credentials are properly configured for:**  
    - Dumpling AI API (HTTP Header Auth)  
    - OpenAI API (GPT-4 model)  
    - Google Sheets (OAuth2)  

12. **Test the workflow by submitting a form with a valid Place ID and Business name.**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Workflow designed for marketing and product teams to extract voice-of-customer insights from Google reviews automatically.                                                                                                                                  | -                                                                                                                            |
| Uses Dumpling AI for fetching Google reviews efficiently via API.                                                                                                                                                                                         | Dumpling AI: https://app.dumplingai.com                                                                                      |
| GPT-4 via LangChain Agent is leveraged to analyze unstructured reviews into structured marketing intelligence.                                                                                                                                             | OpenAI GPT-4: https://openai.com                                                                                             |
| Outputs are saved to a Google Sheet for easy access, reporting, and integration with other tools.                                                                                                                                                              | Google Sheets API: https://developers.google.com/sheets/api                                                                    |
| The Sticky Note summarizes the workflow as a Google Review → Marketing Insight Extractor with tools and use case.                                                                                                                                             | See sticky note content in section 3 above                                                                                     |
| To avoid token limit issues with GPT-4, consider truncating or batching reviews if scaling beyond 30 reviews.                                                                                                                                                  | Practical consideration                                                                                                       |
| Ensure API keys and OAuth credentials have appropriate permissions and are securely stored in n8n credentials manager.                                                                                                                                          | Security best practice                                                                                                         |

---

**Disclaimer:** The text provided is extracted exclusively from a workflow created with n8n automation tool. It fully complies with applicable content policies and contains no illegal, offensive, or protected content. All processed data are legal and public.

---

This documentation enables advanced users and automation agents to fully understand, replicate, and modify the workflow for extracting marketing insights from Google reviews using Dumpling AI and GPT-4.