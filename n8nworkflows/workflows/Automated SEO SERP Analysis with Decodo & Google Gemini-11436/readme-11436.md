Automated SEO SERP Analysis with Decodo & Google Gemini

https://n8nworkflows.xyz/workflows/automated-seo-serp-analysis-with-decodo---google-gemini-11436


# Automated SEO SERP Analysis with Decodo & Google Gemini

### 1. Workflow Overview

This workflow automates SEO SERP (Search Engine Results Page) analysis by integrating Decodo’s Google search scraping API with the Google Gemini AI model. It is designed to receive a list of keywords, perform Google search queries via Decodo, extract and normalize organic search results, and then use AI to generate concise SEO reports. Finally, the results are emailed automatically.

The workflow consists of three logical blocks:

- **1.1 Input Initialization and Keyword Preparation:** Trigger and prepare a structured list of keywords for processing.
- **1.2 Decodo Google Search and Organic Results Extraction:** Loop over keywords, query Decodo’s API for SERP data, and extract normalized organic results.
- **1.3 AI Analysis and Reporting:** Aggregate all extracted data, analyze with Google Gemini AI agent to produce a compact SEO analysis, and email the report.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Keyword Preparation

**Overview:**  
This block initializes the workflow when manually triggered, sets up a raw list of keywords, and formats them into individual items for further processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Edit Fields (Set node)  
- Code in JavaScript (Code node)  
- Loop Over Items (SplitInBatches node)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Config: No parameters; triggers workflow execution on demand.  
  - Inputs: None  
  - Outputs: To "Edit Fields" node  
  - Edge cases: None specific; manual trigger relies on user interaction.

- **Edit Fields**  
  - Type: Set  
  - Role: Defines the keywords array to analyze.  
  - Config: Raw JSON output specifying keywords: ["keyword_1", "keyword_2", "keyword_3"] (placeholder keywords).  
  - Inputs: From Manual Trigger  
  - Outputs: To "Code in JavaScript"  
  - Edge cases: Keywords must be valid strings; empty or malformed list would cause no searches.

- **Code in JavaScript**  
  - Type: Code  
  - Role: Transforms the keywords array into separate workflow items for parallel processing.  
  - Config: JavaScript code extracting keywords from JSON input and returning a mapped array of individual keyword objects.  
  - Key expression: `const { keywords } = $json; return keywords.map(k => ({ json: { keyword: k } }));`  
  - Inputs: From "Edit Fields"  
  - Outputs: To "Loop Over Items"  
  - Edge cases: If `keywords` is missing or not an array, node might fail or produce empty output.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes keywords one-by-one (batch size default) for controlled API querying and rate management.  
  - Config: Default batch options (no special batch size configured).  
  - Inputs: From "Code in JavaScript"  
  - Outputs: To "Code in JavaScript2" and "Decodo" nodes (two branches for parallel processing).  
  - Edge cases: Large keyword lists could cause longer execution time; batch size tuning may be needed.

---

#### 2.2 Decodo Google Search and Organic Results Extraction

**Overview:**  
This block performs the Google search queries using Decodo’s API for each keyword, extracts organic search results safely regardless of JSON nesting, and normalizes the data for downstream analysis.

**Nodes Involved:**  
- Decodo (Decodo API node)  
- Code in JavaScript1 (Code node for organic results extraction)

**Node Details:**

- **Decodo**  
  - Type: Decodo API node (Custom Decodo Google Search)  
  - Role: Queries Google search for the current keyword using Decodo’s scraping API.  
  - Config: Operation = "google_search", Query = expression `{{$json.keyword}}` passing current keyword.  
  - Inputs: From "Loop Over Items"  
  - Outputs: To "Code in JavaScript1"  
  - Credential: Requires Decodo API credential with Web Scraping API Advanced plan token.  
  - Edge cases: API rate limits, invalid token, network errors, or Decodo downtime may cause failures.

- **Code in JavaScript1**  
  - Type: Code  
  - Role: Recursively searches the Decodo API response to find all organic results arrays, flattening and normalizing them.  
  - Config: Custom JavaScript with recursive function `findOrganic` to handle nested JSON and extract organic search results safely.  
  - Key logic includes:  
    - Recursive search for arrays named "organic"  
    - Mapping each organic item to normalized JSON with fields: keyword, title, url, snippet, position  
  - Inputs: From "Decodo"  
  - Outputs: To "Loop Over Items" (continuation branch)  
  - Edge cases: Unexpected JSON structures, missing fields, or empty organic arrays handled gracefully by recursive function.

---

#### 2.3 AI Analysis and Reporting

**Overview:**  
This block aggregates all extracted organic search data, formats it into a single JSON object, passes it to Google Gemini AI for SEO analysis, and sends the resulting report via Gmail.

**Nodes Involved:**  
- Code in JavaScript2 (Aggregation)  
- Google Gemini Chat Model (AI Language Model)  
- AI Agent (SEO Analysis agent)  
- Send a message (Gmail node)  

**Node Details:**

- **Code in JavaScript2**  
  - Type: Code  
  - Role: Aggregates all incoming items into a single JSON object under `chatInput` key to feed the AI agent.  
  - Config: Combines all items using `$input.all().map(item => item.json)` and returns one item with combined data.  
  - Inputs: From "Loop Over Items" (batch processing output)  
  - Outputs: To "AI Agent"  
  - Edge cases: Large data sets might cause memory or processing overhead.

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini Chat Model  
  - Role: Provides AI language model capabilities for the AI Agent node.  
  - Config: No special parameters; acts as language model endpoint for AI analysis.  
  - Inputs: Connected as AI language model for "AI Agent".  
  - Outputs: None (used as resource node)  
  - Credential: Requires Google Gemini API credentials (not explicitly shown).  
  - Edge cases: API quota limits, network issues, or model unavailability.

- **AI Agent**  
  - Type: Langchain AI Agent  
  - Role: Receives combined search data and produces a concise SEO analysis report.  
  - Config: System message instructs the agent to produce a very short, compact SEO report with specific fields (Intent, Content Type, Strengths, Weaknesses, Opportunities), max 6 lines per result, no long paragraphs.  
  - Inputs: From "Code in JavaScript2" and AI model connection to "Google Gemini Chat Model".  
  - Outputs: To "Send a message" node.  
  - Edge cases: Model interpretation errors, malformed input data, or output truncation.

- **Send a message**  
  - Type: Gmail node  
  - Role: Sends the SEO analysis report via email.  
  - Config:  
    - Recipient: user@example.com (placeholder email)  
    - Subject: "Weekly SEO Watchlist — Automated Report"  
    - Message body: uses expression `{{$json.output}}` from AI Agent output.  
    - Email type: Plain text  
  - Inputs: From "AI Agent"  
  - Credential: Requires Gmail OAuth2 credentials configured in n8n.  
  - Edge cases: Authentication failures, quota limits, invalid recipient address.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                               | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                         |
|---------------------------|-------------------------------|-----------------------------------------------|-------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Entry point to start workflow execution       | None                          | Edit Fields                  | Group A: This section initializes the workflow and prepares the keyword list for the search loop.                    |
| Edit Fields               | Set                           | Defines keywords array                         | When clicking ‘Execute workflow’ | Code in JavaScript           | Group A: This section initializes the workflow and prepares the keyword list for the search loop.                    |
| Code in JavaScript        | Code                          | Converts keywords array into individual items | Edit Fields                   | Loop Over Items              | Group A: This section initializes the workflow and prepares the keyword list for the search loop.                    |
| Loop Over Items           | SplitInBatches                | Processes keywords in batches                  | Code in JavaScript            | Code in JavaScript2, Decodo  | Group B : Using Decodo API it search Google based on data from previous group to extract and normalize the organic search results. |
| Decodo                   | Decodo API                    | Queries Google search via Decodo               | Loop Over Items               | Code in JavaScript1          | Group B : Using Decodo API it search Google based on data from previous group to extract and normalize the organic search results. |
| Code in JavaScript1       | Code                          | Extracts and normalizes organic search results | Decodo                       | Loop Over Items (continuation) | Group B : Using Decodo API it search Google based on data from previous group to extract and normalize the organic search results. |
| Code in JavaScript2       | Code                          | Aggregates all search results into one object | Loop Over Items               | AI Agent                    | Group C: Aggregate the collected search data and pass it to the Google Gemini AI Agent for analysis and the AI prepares a report to be emailed. |
| Google Gemini Chat Model  | AI Language Model (Langchain) | Provides language model backend for AI Agent  | None (resource node)          | AI Agent                    | Group C: Aggregate the collected search data and pass it to the Google Gemini AI Agent for analysis and the AI prepares a report to be emailed. |
| AI Agent                 | Langchain AI Agent            | Analyzes data and generates SEO report         | Code in JavaScript2, Google Gemini Chat Model | Send a message              | Group C: Aggregate the collected search data and pass it to the Google Gemini AI Agent for analysis and the AI prepares a report to be emailed. |
| Send a message           | Gmail                         | Sends the final SEO report via email           | AI Agent                     | None                        | Group C: Aggregate the collected search data and pass it to the Google Gemini AI Agent for analysis and the AI prepares a report to be emailed. |
| Sticky Note              | Sticky Note                   | Comments for Group A                            | None                        | None                        | Group A: This section initializes the workflow and prepares the keyword list for the search loop.                    |
| Sticky Note1             | Sticky Note                   | Comments for Group B                            | None                        | None                        | Group B : Using Decodo API it search Google based on data from previous group to extract and normalize the organic search results. |
| Sticky Note2             | Sticky Note                   | Comments for Group C                            | None                        | None                        | Group C: Aggregate the collected search data and pass it to the Google Gemini AI Agent for analysis and the AI prepares a report to be emailed. |
| Sticky Note15            | Sticky Note                   | Instructions for Decodo credential setup       | None                        | None                        | ## How to Set Up **Decodo** Credentials in n8n: Activate plan, get token, add in n8n credentials. Details: github.com/Decodo/n8n-nodes-decodo/tree/main |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: "When clicking ‘Execute workflow’"  
   - No parameters; triggers workflow on manual execution.

2. **Create Set node**  
   - Name: "Edit Fields"  
   - Mode: Raw JSON  
   - JSON Output:  
     ```json
     {
       "keywords": [
         "keyword_1",
         "keyword_2",
         "keyword_3"
       ]
     }
     ```  
   - Connect output from Manual Trigger.

3. **Create Code node**  
   - Name: "Code in JavaScript"  
   - Language: JavaScript  
   - Code:
     ```javascript
     const { keywords } = $json;
     return keywords.map(k => ({ json: { keyword: k } }));
     ```  
   - Connect input from "Edit Fields".

4. **Create SplitInBatches node**  
   - Name: "Loop Over Items"  
   - Default batch size (or tune as needed)  
   - Connect input from "Code in JavaScript".

5. **Create Decodo API node**  
   - Name: "Decodo"  
   - Operation: "google_search"  
   - Query: Expression `{{$json.keyword}}`  
   - Credentials: Setup Decodo API credential (see step 13)  
   - Connect input from "Loop Over Items".

6. **Create Code node**  
   - Name: "Code in JavaScript1"  
   - Language: JavaScript  
   - Paste the recursive extractor code:
     ```javascript
     // Recursive search for all arrays named "organic"
     function findOrganic(node, results = []) {
       if (!node || typeof node !== "object") return results;
       if (Array.isArray(node)) {
         for (const item of node) findOrganic(item, results);
       } else {
         for (const key of Object.keys(node)) {
           if (key === "organic" && Array.isArray(node[key])) {
             results.push(...node[key]);
           } else {
             findOrganic(node[key], results);
           }
         }
       }
       return results;
     }
     const organicItems = findOrganic($json);
     const cleaned = organicItems.map(r => ({
       keyword: $json.keyword || "",
       title: r.title || "",
       url: r.url || "",
       snippet: r.desc || r.snippet || "",
       position: r.pos || null
     }));
     return cleaned.map(item => ({ json: item }));
     ```  
   - Connect input from "Decodo".

7. **Connect output of "Code in JavaScript1" back to "Loop Over Items" continuation branch**  
   - This allows batch continuation and aggregation.

8. **Create Code node**  
   - Name: "Code in JavaScript2"  
   - Language: JavaScript  
   - Code:
     ```javascript
     const combined = $input.all().map(item => item.json);
     return [{ json: { chatInput: combined } }];
     ```  
   - Connect input from "Loop Over Items" (the batch output branch that collects processed items).

9. **Create Google Gemini Chat Model node**  
   - Name: "Google Gemini Chat Model"  
   - No special parameters; serves as AI model resource for agent.

10. **Create AI Agent node**  
    - Name: "AI Agent"  
    - System Message:
      ```
      You are an SEO analyst.  
      You will receive JSON “data” containing several Google SERP results.

      Your job is to produce a **very short, compact SEO analysis**, max 6 lines per result:

      1. Intent (short)
      2. Content Type (short)
      3. Strengths (bullet)
      4. Weaknesses (bullet)
      5. Opportunities (bullet)

      No long paragraphs. No storytelling. Only compressed SEO insights.
      ```  
    - Connect AI Language Model input to "Google Gemini Chat Model" node.  
    - Connect input from "Code in JavaScript2".

11. **Create Gmail node**  
    - Name: "Send a message"  
    - Email To: `user@example.com` (replace with real recipient)  
    - Subject: "Weekly SEO Watchlist — Automated Report"  
    - Message: Expression `{{$json.output}}` from AI Agent node output  
    - Email Type: Text  
    - Credentials: Configure Gmail OAuth2 credentials in n8n  
    - Connect input from "AI Agent".

12. **Connect workflow nodes in order:**  
    - Manual Trigger → Edit Fields → Code in JavaScript → Loop Over Items  
    - Loop Over Items branches:  
      - → Decodo → Code in JavaScript1 → Loop Over Items (continuation)  
      - → Code in JavaScript2 → AI Agent → Send a message  
    - AI Agent also connects to Google Gemini Chat Model as AI model resource.

13. **Credential Setup:**  
    - **Decodo Credential Setup:**  
      - Activate Web Scraping API Advanced plan on Decodo dashboard (trial available).  
      - Copy API token from Decodo dashboard Web Scraping API page.  
      - In n8n, create new credential of type "Decodo Credentials API".  
      - Paste token and save.  
    - **Gmail Credential Setup:**  
      - Create OAuth2 credentials for Gmail in n8n.  
      - Ensure proper scopes to send emails.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Decodo API credentials require Web Scraping API Advanced plan. Start with free trial at Decodo dashboard. | See sticky note content and https://github.com/Decodo/n8n-nodes-decodo/tree/main |
| The AI Agent uses Google Gemini Chat Model for SEO analysis using Langchain integration. | Requires Google Gemini API credentials and proper quota. |
| The workflow sends SEO reports automatically by email using Gmail OAuth2 credentials. | Gmail API setup documentation recommended. |
| Keywords in "Edit Fields" are placeholders; replace with actual target SEO keywords. | User customization point. |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow built using n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.