Notion AI Summary & Tags

https://n8nworkflows.xyz/workflows/notion-ai-summary---tags-4431


# Notion AI Summary & Tags

---

### 1. Workflow Overview

This workflow automates the process of enriching Notion database pages with AI-generated summaries and tags. It is designed for users who maintain content collections in Notion and want to leverage AI to create concise summaries and relevant tags automatically.

The workflow is logically divided into these blocks:

- **1.1 Input Reception (Notion Trigger):** Watches for new or updated pages in a specified Notion database.
- **1.2 Content Retrieval (HTTP Request):** Fetches the raw content from a URL stored in the database page.
- **1.3 AI Processing (AI Agent + OpenAI Chat Model):** Uses OpenAI's GPT-4o-mini model via Langchain agent to generate a summary and keyword tags from the content.
- **1.4 Parsing AI Output (Code):** Extracts structured summary and tags from the AI agent’s markdown response.
- **1.5 Notion Update (Notion node):** Updates the original Notion page with the AI-generated summary and tags.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception (Notion Trigger)

- **Overview:**  
  Watches a Notion database continuously (every minute) for new or updated pages, triggering the workflow when changes occur.

- **Nodes Involved:**  
  - Notion Trigger

- **Node Details:**

  | Property | Description |
  |---|---|
  | **Type** | n8n native Notion Trigger node |
  | **Configuration** | Polls every minute; monitors a specific Notion database identified by `YOUR_DATABASE_ID`. The database ID is stored securely and referenced dynamically. |
  | **Expressions / Variables** | Database ID is referenced securely via credential binding (`__rl` flag). |
  | **Input/Output** | No input nodes; outputs page data containing metadata such as page ID and URL. |
  | **Version** | v1 |
  | **Potential Failures** | - Authentication failure if Notion credentials expire or are invalid.  
  - Rate limiting by Notion API if polling too frequently.  
  - Database ID misconfigured or missing, causing trigger failure. |
  | **Sub-workflow** | None |

---

#### 1.2 Content Retrieval (HTTP Request)

- **Overview:**  
  Retrieves the content of the page or URL stored in the Notion database page to supply raw text for AI summarization.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  | Property | Description |
  |---|---|
  | **Type** | HTTP Request (n8n native) |
  | **Configuration** | Sends a GET request to the URL extracted dynamically from the Notion Trigger node’s JSON (`{{$json.URL}}`). Custom headers set for User-Agent, Accept, and Accept-Language to mimic typical browser behavior and avoid blocking. |
  | **Expressions / Variables** | URL dynamically from input JSON. |
  | **Input/Output** | Input: Notion Trigger output.  
  Output: Raw HTML/text content of the URL. |
  | **Version** | v4.2 |
  | **Potential Failures** | - Invalid or missing URL in data.  
  - Network errors, timeouts, or 4xx/5xx HTTP responses.  
  - Content blocked or altered by website (e.g., bot detection). |
  | **Sub-workflow** | None |

---

#### 1.3 AI Processing (AI Agent + OpenAI Chat Model)

- **Overview:**  
  Uses an AI agent to analyze the fetched content, creating a concise summary and generating relevant tags. The AI Agent node defines the prompt and orchestrates the language model call, which is handled by the OpenAI Chat Model node.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**

  - **AI Agent**

    | Property | Description |
    |---|---|
    | **Type** | Langchain AI Agent node (third-party n8n node) |
    | **Configuration** | Custom prompt defines the task: summarize input content in 3-5 sentences, then provide 3–6 concise tags as markdown with specific formatting (`**Summary**:` and `**Tags**:`). Input text is dynamically injected from the HTTP Request output (`{{ $json.data }}`). |
    | **Expressions / Variables** | `{{ $json.data }}` injects fetched content. |
    | **Input/Output** | Input: HTTP Request output content.  
    Output: AI-generated markdown text containing summary and tags. |
    | **Version** | v1.9 |
    | **Potential Failures** | - AI model quota exceeded or authentication errors.  
    - Unexpected AI output format breaking parsing.  
    - Timeout or latency issues communicating with OpenAI. |
    | **Sub-workflow** | None |

  - **OpenAI Chat Model**

    | Property | Description |
    |---|---|
    | **Type** | Langchain OpenAI Chat Model node |
    | **Configuration** | Uses `gpt-4o-mini` model, a smaller GPT-4 variant optimized for chat completions. No additional options specified. |
    | **Expressions / Variables** | None beyond default. |
    | **Input/Output** | Input: AI Agent language model call.  
    Output: AI text completion passed back to AI Agent. |
    | **Version** | v1.2 |
    | **Potential Failures** | - Credential misconfiguration or API key issues.  
    - Model unavailability or deprecation. |
    | **Sub-workflow** | None |

---

#### 1.4 Parsing AI Output (Code)

- **Overview:**  
  Parses the AI-generated markdown text to extract the summary and tags into structured JSON properties for Notion update.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  | Property | Description |
  |---|---|
  | **Type** | n8n native Code node (JavaScript) |
  | **Configuration** | Custom JavaScript extracts:  
  - Summary: Matches text following `**Summary**:` up to `**Tags**:`  
  - Tags: Matches text after `**Tags**:` and splits by commas into an array |
  | **Expressions / Variables** | Uses regex on `item.json.output` (AI text) from AI Agent output. |
  | **Input/Output** | Input: AI Agent output markdown text.  
  Output: JSON with keys `aiSummary` (string) and `tags` (array of strings). |
  | **Version** | v2 |
  | **Potential Failures** | - AI output missing expected markdown structure causes regex failure and error throw.  
  - Tags or summary empty or malformed. |
  | **Sub-workflow** | None |

---

#### 1.5 Notion Update (Notion)

- **Overview:**  
  Updates the original Notion page that triggered the workflow by writing back the AI-generated summary and tags into designated properties.

- **Nodes Involved:**  
  - Notion

- **Node Details:**

  | Property | Description |
  |---|---|
  | **Type** | n8n native Notion node |
  | **Configuration** | Updates a database page by:  
  - Page ID dynamically taken from Notion Trigger node (`={{ $('Notion Trigger').item.json.id }}`)  
  - Updates two fields:  
    - `AI summary` (rich text) with AI summary string  
    - `Tags` (multi-select) with array of tags |
  | **Expressions / Variables** | Uses expressions to map input JSON fields `aiSummary` and `tags` from Code node. |
  | **Input/Output** | Input: Parsed summary and tags JSON from Code node.  
  Output: Confirmation of Notion page update. |
  | **Version** | v2.2 |
  | **Potential Failures** | - Permission errors if API token lacks write access.  
  - Page ID missing or invalid.  
  - Property keys misconfigured or changed in Notion database schema. |
  | **Sub-workflow** | None |

---

### 3. Summary Table

| Node Name       | Node Type                            | Functional Role                           | Input Node(s)         | Output Node(s)      | Sticky Note                                                                                       |
|-----------------|------------------------------------|-----------------------------------------|-----------------------|---------------------|-------------------------------------------------------------------------------------------------|
| Notion Trigger  | n8n-nodes-base.notionTrigger       | Watches Notion database for changes     | —                     | HTTP Request        |                                                                                                 |
| HTTP Request    | n8n-nodes-base.httpRequest          | Retrieves raw content from URL           | Notion Trigger        | AI Agent            |                                                                                                 |
| AI Agent        | @n8n/n8n-nodes-langchain.agent     | Generates summary and tags via AI        | HTTP Request          | Code                |                                                                                                 |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model called by AI Agent        | AI Agent (ai_languageModel) | AI Agent            |                                                                                                 |
| Code            | n8n-nodes-base.code                 | Parses AI output markdown into JSON     | AI Agent              | Notion              | Throws error if summary or tags cannot be extracted from AI text output                         |
| Notion          | n8n-nodes-base.notion              | Updates Notion page with AI summary/tags | Code                  | —                   | Ensure Notion database schema has properties `AI summary|rich_text` and `Tags|multi_select` configured |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Notion Trigger Node**  
   - Type: Notion Trigger  
   - Set Poll Times: Every Minute  
   - Set Database ID: Input your Notion database ID (the content database you want to watch)  
   - Configure Notion credentials with appropriate permissions (read access to the database)

2. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Connect input from Notion Trigger main output  
   - Method: GET  
   - URL: Use expression `{{$json.URL}}` to fetch the URL field from the Notion page data  
   - Set Headers:  
     - User-Agent: `Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)`  
     - Accept: `text/html`  
     - Accept-Language: `en-US,en;q=0.9`  
   - Leave other options default

3. **Create AI Agent Node**  
   - Type: Langchain AI Agent  
   - Connect input from HTTP Request main output  
   - Prompt Type: Define  
   - Prompt Text:  
     ```
     You are a helpful assistant.
     Summarize the following content in 3-5 sentences.
     Then suggest 3–6 concise tags (single-word or short phrases).
     {{ $json.data }}

     Respond as in valid markdown:
     **Summary**: <summary>
     **Tags**: tag1, tag2, ...
     ```
   - Leave Options default

4. **Create OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Connect input from AI Agent node’s `ai_languageModel` output  
   - Model: Select `gpt-4o-mini`  
   - Configure OpenAI credentials with valid API Key

5. **Connect OpenAI Chat Model node output back to AI Agent node input** (as `ai_languageModel`)

6. **Create Code Node**  
   - Type: Code  
   - Connect input from AI Agent main output  
   - Use the following JavaScript code:  
     ```javascript
     const items = $input.all();
     const result = [];

     for (const item of items) {
       const text = item.json.output;

       const summaryMatch = text.match(/\*\*Summary\*\*:\s*([\s\S]*?)\s*\*\*Tags\*\*:/i);
       const tagsMatch = text.match(/\*\*Tags\*\*:\s*([^\n\r]*)/i);

       if (!summaryMatch || !tagsMatch) {
         throw new Error("Couldn't extract summary or tags from AI output");
       }

       const summary = summaryMatch[1].trim();
       const tagsStr = tagsMatch[1].trim();
       const tags = tagsStr
         .split(",")
         .map(tag => tag.trim())
         .filter(tag => tag.length > 0);

       result.push({
         json: {
           aiSummary: summary,
           tags: tags
         }
       });
     }

     return result;
     ```

7. **Create Notion Node**  
   - Type: Notion  
   - Connect input from Code node main output  
   - Operation: Update Database Page  
   - Page ID: Expression `={{ $('Notion Trigger').item.json.id }}`  
   - Properties to update:  
     - `AI summary` (rich text): Expression `={{ $json.aiSummary }}`  
     - `Tags` (multi-select): Expression `={{ $json.tags }}`  
   - Configure Notion credentials with write permission

8. **Activate the Workflow**  
   - Save and activate the workflow to start polling and processing Notion database changes.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                   |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| The AI prompt formatting requires the AI response to strictly follow markdown with "**Summary**:" and "**Tags**:" labels for parsing. | Critical for Code node regex parsing to function correctly.      |
| Ensure your Notion database schema includes the properties exactly named `AI summary` (rich text) and `Tags` (multi-select) to avoid update errors. | Schema setup in Notion UI required before use.                   |
| OpenAI API key must have access to GPT-4o-mini or equivalent; verify model availability in your account.      | OpenAI account and API documentation                              |
| The workflow relies on polling every minute, which may cause rate limits—adjust as appropriate for your use case. | Consider Notion API quotas and adjust polling frequency if needed. |
| For more on Langchain AI Agent node usage in n8n, visit: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain.agent | Official n8n documentation                                        |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---