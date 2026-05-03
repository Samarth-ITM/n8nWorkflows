Enrich Pipedrive's Organization Data with OpenAI GPT-4o & Notify it in Slack

https://n8nworkflows.xyz/workflows/enrich-pipedrive-s-organization-data-with-openai-gpt-4o---notify-it-in-slack-2318


# Enrich Pipedrive's Organization Data with OpenAI GPT-4o & Notify it in Slack

### 1. Workflow Overview

This workflow automates enriching newly created organization records in Pipedrive by leveraging website data scraped from a custom "website" field of the organization. The scraped HTML content is processed by the OpenAI GPT-4o model to extract structured company insights, which are then added back as a detailed note to the organization in Pipedrive. Finally, a Slack notification is sent to inform a designated channel about the enriched organization note.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and Input Reception:** Detects creation of a new organization in Pipedrive and extracts the associated website URL.
- **1.2 Data Scraping:** Retrieves the homepage content from the website URL using the ScrapingBee API.
- **1.3 AI Processing:** Sends scraped HTML content to OpenAI GPT-4o with a system prompt to generate a formatted company summary.
- **1.4 Organization Note Creation:** Adds the AI-generated summary as a note to the Pipedrive organization.
- **1.5 Formatting for Slack:** Converts the HTML note into Slack-compatible markdown formatting through two sequential nodes.
- **1.6 Slack Notification:** Posts the formatted company note to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Reception

- **Overview:**  
  This block listens for new organization creations in Pipedrive. When an organization is created, it retrieves the organization's data, including a custom field for the website URL (stored under a random API ID).

- **Nodes Involved:**  
  - `Pipedrive Trigger - An Organization is created`

- **Node Details:**  
  - **Node:** `Pipedrive Trigger - An Organization is created`  
    - Type: Pipedrive Trigger  
    - Role: Initiates workflow on creation of a Pipedrive organization object.  
    - Configuration: Trigger set to action "added" on object "organization". Requires Pipedrive API credentials.  
    - Key Variables: Outputs organization's JSON including custom website field under a random ID (e.g., `current.<random_api_id_custom_website_field>`).  
    - Input: None (trigger node)  
    - Output: Organization data JSON  
    - Possible Failures: API authentication errors, webhook misconfiguration, missing or malformed website URL field.  
    - Version: 1

---

#### 2.2 Data Scraping

- **Overview:**  
  Uses the ScrapingBee API to retrieve the HTML content of the organization's website homepage from the URL obtained in the previous step.

- **Nodes Involved:**  
  - `Scrapingbee - Get Organization's URL content`

- **Node Details:**  
  - **Node:** `Scrapingbee - Get Organization's URL content`  
    - Type: HTTP Request  
    - Role: Calls external scraping API to fetch website content.  
    - Configuration:  
      - Method: GET  
      - URL: `https://app.scrapingbee.com/api/v1`  
      - Query Parameters:  
        - `api_key`: User must insert ScrapingBee API key.  
        - `url`: Dynamically set to the website URL extracted from the Pipedrive trigger node.  
        - `render_js`: false (no JavaScript rendering).  
    - Input: Organization JSON with website URL.  
    - Output: JSON containing the HTML content of the website page under `.content`.  
    - Possible Failures: Invalid API key, scraping API limits or downtime, invalid or unreachable website URL, network timeouts, rate limits.  
    - Version: 4.2

---

#### 2.3 AI Processing

- **Overview:**  
  Sends the scraped HTML content to OpenAI GPT-4o with a system prompt instructing it to summarize company information, including products/services, target market, unique selling propositions, and potential competitors. The AI returns a richly formatted HTML summary optimized for Pipedrive notes.

- **Nodes Involved:**  
  - `OpenAI - Message GPT-4o with Scraped Data`

- **Node Details:**  
  - **Node:** `OpenAI - Message GPT-4o with Scraped Data`  
    - Type: OpenAI Node (LangChain variant)  
    - Role: Generate company summary from HTML content using GPT-4o.  
    - Configuration:  
      - Model: GPT-4o (selected for large token capacity).  
      - Messages:  
        - User content: scraped HTML content.  
        - System prompt: detailed instructions for summarization and HTML formatting (includes an example response).  
    - Input: Scraped HTML content JSON from previous node.  
    - Output: JSON with GPT-generated HTML summary under `message.content`.  
    - Credentials: OpenAI API key required.  
    - Possible Failures: API quota exceeded, invalid API key, prompt or input too large, timeouts, malformed input causing generation errors.  
    - Version: 1.3

---

#### 2.4 Organization Note Creation

- **Overview:**  
  Creates a note attached to the Pipedrive organization with the HTML summary generated by OpenAI.

- **Nodes Involved:**  
  - `Pipedrive - Create a Note with OpenAI output`

- **Node Details:**  
  - **Node:** `Pipedrive - Create a Note with OpenAI output`  
    - Type: Pipedrive Node  
    - Role: Creates a note record linked to the organization using AI-generated content.  
    - Configuration:  
      - Resource: `note`  
      - Additional Field: `org_id` set dynamically to the organization ID from the trigger node.  
      - Content field: populated with the HTML content from OpenAI output (`{{$json.message.content}}`).  
    - Input: AI-generated HTML summary.  
    - Output: Confirmation of note creation.  
    - Credentials: Pipedrive API key required.  
    - Possible Failures: API authentication errors, invalid organization ID, rate limits, malformed note content.  
    - Version: 1

---

#### 2.5 Formatting for Slack

- **Overview:**  
  Converts the HTML note into Slack-friendly markdown through two nodes: first from HTML to standard markdown, then from markdown to Slack markdown syntax.

- **Nodes Involved:**  
  - `HTML To Markdown`  
  - `Code - Markdown to Slack Markdown`

- **Node Details:**  

  - **Node:** `HTML To Markdown`  
    - Type: Markdown Node  
    - Role: Converts HTML to Markdown syntax.  
    - Configuration: Input HTML from Pipedrive note creation output field `.content`.  
    - Input: HTML note content.  
    - Output: Standard markdown text.  
    - Possible Failures: Malformed HTML, unexpected tags causing conversion errors.  
    - Version: 1

  - **Node:** `Code - Markdown to Slack Markdown`  
    - Type: Code (JavaScript) Node  
    - Role: Custom script to convert generic markdown to Slack-specific markdown formatting.  
    - Configuration: Inline JavaScript code performing regex replacements:  
      - Converts headers (#, ##) to bold text.  
      - Converts unordered lists (`*`) to arrow bullets (`➡️`).  
      - Converts simple markdown tables into Slack-friendly text bullet lists.  
    - Input: Markdown from previous node.  
    - Output: Slack markdown formatted text under `slackFormattedMarkdown`.  
    - Possible Failures: Unexpected markdown structures, regex mismatches, empty input.  
    - Version: 2

---

#### 2.6 Slack Notification

- **Overview:**  
  Sends a Slack message to a predefined channel with the Slack-formatted summary note, notifying users of the new enriched Pipedrive organization.

- **Nodes Involved:**  
  - `Slack - Notify`

- **Node Details:**  
  - **Node:** `Slack - Notify`  
    - Type: Slack Node  
    - Role: Posts a message to Slack channel to notify about the new organization and its enriched note.  
    - Configuration:  
      - Text field: includes dynamic insertion of organization name and markdown summary.  
      - Channel: selected from a cached list named `pipedrive-notification`.  
      - Authentication: OAuth2 with Slack credentials.  
    - Input: Slack markdown formatted summary from previous node.  
    - Output: Confirmation of message sent.  
    - Possible Failures: Slack API rate limits, invalid channel, OAuth token expiry, message formatting errors.  
    - Version: 2.2

---

### 3. Summary Table

| Node Name                                    | Node Type                       | Functional Role                                   | Input Node(s)                                | Output Node(s)                             | Sticky Note                                                                                               |
|----------------------------------------------|--------------------------------|--------------------------------------------------|----------------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Pipedrive Trigger - An Organization is created | Pipedrive Trigger              | Trigger workflow on new organization creation    | None                                         | Scrapingbee - Get Organization's URL content | Contains full workflow description, disclaimers, notes on OpenAI model, and workflow overview            |
| Scrapingbee - Get Organization's URL content  | HTTP Request                   | Scrapes website content from organization URL    | Pipedrive Trigger - An Organization is created | OpenAI - Message GPT-4o with Scraped Data  | See above                                                                                                 |
| OpenAI - Message GPT-4o with Scraped Data     | OpenAI (LangChain)             | Generates company summary from scraped HTML      | Scrapingbee - Get Organization's URL content | Pipedrive - Create a Note with OpenAI output | See above                                                                                                 |
| Pipedrive - Create a Note with OpenAI output  | Pipedrive                      | Creates note on Pipedrive organization            | OpenAI - Message GPT-4o with Scraped Data    | HTML To Markdown                           | See above                                                                                                 |
| HTML To Markdown                               | Markdown                      | Converts HTML note to markdown                     | Pipedrive - Create a Note with OpenAI output | Code - Markdown to Slack Markdown          | See above                                                                                                 |
| Code - Markdown to Slack Markdown              | Code (JavaScript)              | Converts markdown to Slack-specific markdown      | HTML To Markdown                             | Slack - Notify                             | See above                                                                                                 |
| Slack - Notify                                 | Slack                         | Sends Slack notification about enriched organization | Code - Markdown to Slack Markdown             | None                                       | See above                                                                                                 |
| Sticky Note                                    | StickyNote                    | Documentation and instructions                     | None                                         | None                                       | Full workflow description, legal disclaimers, and operational notes                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a **Pipedrive Trigger** node named `Pipedrive Trigger - An Organization is created`.  
   - Configure it to trigger on action `added` and object `organization`.  
   - Set up and attach your **Pipedrive API credentials**.  
   - Confirm the organization object includes a custom "website" field (note: in n8n this will appear as a random ID).

2. **Add Scraping Node**  
   - Add an **HTTP Request** node named `Scrapingbee - Get Organization's URL content`.  
   - Set Method to `GET`.  
   - Set URL to `https://app.scrapingbee.com/api/v1`.  
   - Add Query Parameters:  
     - `api_key`: Your ScrapingBee API key.  
     - `url`: Set expression to the custom website field from the Pipedrive trigger node, e.g., `{{$json.current.<random_api_id_custom_website_field>}}`.  
     - `render_js`: `false`.  
   - Connect the Pipedrive trigger node output to this node’s input.

3. **Add OpenAI Node**  
   - Add an **OpenAI (LangChain)** node named `OpenAI - Message GPT-4o with Scraped Data`.  
   - Select model `gpt-4o`.  
   - Configure the messages:  
     - User message: content set to the scraped HTML (`{{$json.data}}`).  
     - System message: Use the detailed prompt instructing the model to extract company info and format as HTML (copy from the provided example).  
   - Attach your **OpenAI API credentials**.  
   - Connect the scraping node output to this node.

4. **Add Pipedrive Note Creation Node**  
   - Add a **Pipedrive** node named `Pipedrive - Create a Note with OpenAI output`.  
   - Set resource to `note`.  
   - In Additional Fields, set `org_id` to the organization ID from the trigger node: `{{$('Pipedrive Trigger - An Organization is created').item.json.meta.id}}`.  
   - Set `content` field to the OpenAI node output: `{{$json.message.content}}`.  
   - Connect the OpenAI node output to this node.  
   - Use the same Pipedrive API credentials as before.

5. **Add HTML to Markdown Node**  
   - Add a **Markdown** node named `HTML To Markdown`.  
   - Set the HTML input to the content from the Pipedrive note creation node: `{{$json.content}}`.  
   - Connect the Pipedrive note node output here.

6. **Add Markdown to Slack Markdown Code Node**  
   - Add a **Code** node named `Code - Markdown to Slack Markdown`.  
   - Use the provided JavaScript code to convert standard markdown to Slack markdown:  
     ```js
     const inputMarkdown = items[0].json.data;

     function convertMarkdownToSlackFormat(markdown) {
         let slackFormatted = markdown;
         
         // Convert headers
         slackFormatted = slackFormatted.replace(/^# (.*$)/gim, '*$1*');
         slackFormatted = slackFormatted.replace(/^## (.*$)/gim, '*$1*');
         
         // Convert unordered lists
         slackFormatted = slackFormatted.replace(/^\* (.*$)/gim, '➡️ $1');
         
         // Convert tables
         const tableRegex = /\n\|.*\|\n\|.*\|\n((\|.*\|\n)+)/;
         const tableMatch = slackFormatted.match(tableRegex);
         if (tableMatch) {
             const table = tableMatch[0];
             const rows = table.split('\n').slice(3, -1);
             const formattedRows = rows.map(row => {
                 const columns = row.split('|').slice(1, -1).map(col => col.trim());
                 return `*${columns[0]}*: ${columns[1]}`;
             }).join('\n');
             slackFormatted = slackFormatted.replace(table, formattedRows);
         }
         
         return slackFormatted;
     }

     const slackMarkdown = convertMarkdownToSlackFormat(inputMarkdown);
     console.log(slackMarkdown);

     return [{ slackFormattedMarkdown: slackMarkdown }];
     ```  
   - Connect the HTML to Markdown node output here.

7. **Add Slack Notification Node**  
   - Add a **Slack** node named `Slack - Notify`.  
   - Set authentication to OAuth2 and attach your Slack credentials.  
   - Choose the target Slack channel (from cached list or manually enter channel ID).  
   - Set the message text to:  
     ```
     =*New Organization {{$('Pipedrive Trigger - An Organization is created').item.json.current.name}} created on Pipedrive* :

     {{$json.slackFormattedMarkdown}}
     ```  
   - Connect the Code node output to this Slack node.

8. **Finalize Workflow**  
   - Connect all nodes in the order described:  
     `Pipedrive Trigger` → `ScrapingBee HTTP Request` → `OpenAI` → `Pipedrive Note Creation` → `HTML To Markdown` → `Code` → `Slack Notify`.  
   - Test the workflow with a new organization creation in Pipedrive with a valid website URL.  
   - Adjust OpenAI system prompt or Slack formatting code as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses a scraping API; ensure compliance with your local web scraping regulations before use.                                | Disclaimer in workflow description                                                                 |
| OpenAI GPT-4o is selected for its large input capacity but is also the most expensive option; consider cost vs. benefit.                | Important note in workflow description                                                             |
| System prompt for OpenAI is customizable to tailor the company summary output format and content.                                        | OpenAI Node configuration                                                                           |
| ScrapingBee is the default scraping service used, but any HTTP request node or alternative scraping API can be substituted.             | Node 2 description / https://www.scrapingbee.com/                                                  |
| Slack markdown formatting is customized via JavaScript code to convert HTML-derived markdown into Slack-readable format.                | Node 6 code node                                                                                     |
| The custom "website" field in Pipedrive must exist and be correctly configured; n8n only sees this field as a random ID in the JSON data. | Important for the trigger node data extraction                                                     |
| Workflow is designed for Pipedrive organizations but can be adapted for other CRM objects with website URLs and note capabilities.       | General adaptability note                                                                           |

---

This reference document fully describes the workflow structure, function, and configuration, enabling advanced users and automation agents to understand, reproduce, and safely maintain or enhance the process.