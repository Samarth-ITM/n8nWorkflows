Convert Markdown to Formatted Notion Pages with Mark2Notion

https://n8nworkflows.xyz/workflows/convert-markdown-to-formatted-notion-pages-with-mark2notion-8331


# Convert Markdown to Formatted Notion Pages with Mark2Notion

### 1. Workflow Overview

This workflow, titled **"Convert Markdown to Formatted Notion Pages with Mark2Notion"**, automates the transformation of Markdown text into richly formatted Notion pages by leveraging the Mark2Notion API. It is designed to handle complex Markdown structures including nested lists, tables, and long content, while addressing Notion API constraints such as block limits and text length restrictions.

**Target Use Cases:**  
- Automating content publishing from AI-generated Markdown (e.g., ChatGPT outputs) into Notion  
- Converting form inputs or GitHub markdown content directly into Notion documentation  
- Creating structured meeting notes or reports from raw Markdown data  

**Logical Blocks:**

- **1.1 Input Trigger & Markdown Setup:** Manual start of the workflow and embedding sample Markdown content.  
- **1.2 Notion Page Creation:** Creating a new Notion page that will act as the parent for appended content.  
- **1.3 Mark2Notion API Call:** Sending the Markdown and necessary credentials to Mark2Notion API to append the content to the created Notion page.  
- **1.4 Documentation & Configuration Notes:** Several sticky notes provide essential instructions and credentials setup guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger & Markdown Setup

**Overview:**  
This block triggers the workflow manually and sets a sample Markdown text that will be converted and appended to a Notion page.

**Nodes Involved:**  
- When you click 'Execute Workflow' (Manual Trigger)  
- Set Markdown

**Node Details:**

- **When you click 'Execute Workflow'**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow execution.  
  - *Configuration:* No parameters; simply triggers downstream nodes.  
  - *Input:* None  
  - *Output:* Activates the "Set Markdown" node.  
  - *Edge Cases:* None specific; may fail if manual trigger is not activated.  

- **Set Markdown**  
  - *Type:* Set  
  - *Role:* Defines the Markdown content to be processed.  
  - *Configuration:* Raw JSON output with a `markdown` field containing a multiline string of Markdown text. The sample includes headers, paragraphs, nested lists, and a table.  
  - *Key Expressions:* The entire Markdown text is hardcoded within this node's `markdown` property.  
  - *Input:* Trigger from Manual Trigger node  
  - *Output:* Passes JSON with `markdown` property to "Create a page".  
  - *Edge Cases:* Hardcoded content means no dynamic input; modifying the Markdown requires editing this node or connecting a dynamic Markdown source.

---

#### 1.2 Notion Page Creation

**Overview:**  
Creates a new Notion page that will serve as the container for the appended Markdown content.

**Nodes Involved:**  
- Create a page

**Node Details:**

- **Create a page**  
  - *Type:* Notion (Built-in n8n node)  
  - *Role:* Creates a new Notion page with a specified title, acting as the parent page for appending content.  
  - *Configuration:*  
    - Title set statically as "Mark2Notion Test Page"  
    - `pageId` parameter is set to accept an ID mode (dynamic), but in this workflow the input is empty (likely to create a top-level page or subpage under default workspace).  
  - *Input:* Receives JSON with `markdown` (though not directly used here) from "Set Markdown"  
  - *Output:* Passes created page details, including its ID, to "HTTP Request - Mark2Notion Append"  
  - *Version Requirements:* Requires Notion API credentials configured in n8n.  
  - *Edge Cases:*  
    - Authentication failures if Notion credentials are invalid or missing  
    - API rate limits or quota exceeded errors  
    - Invalid pageId or permissions errors if the integration lacks access to the parent page  

---

#### 1.3 Mark2Notion API Call

**Overview:**  
Sends the Markdown content along with necessary authentication to the Mark2Notion API to convert and append the content to the newly created Notion page.

**Nodes Involved:**  
- HTTP Request - Mark2Notion Append

**Node Details:**

- **HTTP Request - Mark2Notion Append**  
  - *Type:* HTTP Request  
  - *Role:* Posts Markdown data to Mark2Notion API endpoint `/api/append` for processing.  
  - *Configuration:*  
    - Method: POST  
    - URL: https://api.mark2notion.com/api/append  
    - Authentication: HTTP Header Auth with generic credentials (Mark2Notion API key)  
    - Body parameters:  
      - `markdown`: dynamically set from the `markdown` field of "Set Markdown" node  
      - `notionToken`: token for Notion API integration (passed as body parameter)  
      - `pageId`: ID of the created Notion page (dynamically from "Create a page")  
  - *Expressions Used:*  
    - `={{ $('Set Markdown').item.json.markdown }}` for Markdown content  
    - `={{ $json.id }}` for page ID from "Create a page"  
  - *Input:* Receives page ID from "Create a page"  
  - *Output:* Outputs response from Mark2Notion API (typically confirmation of append success)  
  - *Edge Cases:*  
    - HTTP errors due to invalid API keys, expired tokens, or network issues  
    - Payload too large if Markdown content exceeds limits (though Mark2Notion handles chunking internally)  
    - Rate limiting by Mark2Notion API  
    - Invalid or missing page ID or Notion token leading to failed append operations  

---

#### 1.4 Documentation & Configuration Notes

**Overview:**  
Sticky notes provide detailed guidance for users on how to configure credentials, use the workflow, and understand its capabilities.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3

**Node Details:**

- **Sticky Note (at position 0,0)**  
  - *Purpose:* Explains the workflow’s function, capabilities, use cases, and usage instructions.  
  - *Content Highlights:*  
    - Describes the complexity handled by Mark2Notion (tables, nested lists, rate limits)  
    - Lists use cases like AI content publishing and meeting notes automation  
    - Step-by-step usage instructions including API key and integration setup  

- **Sticky Note1 (near "Set Markdown")**  
  - *Purpose:* Suggests that any Markdown output node can be inserted here, such as AI model replies or GitHub issues.  

- **Sticky Note2 (near HTTP Request node)**  
  - *Purpose:* Credential setup instructions for Mark2Notion API:  
    - Use HTTP Header Auth with header `x-api-key`  
    - Pass `notionToken` as a body parameter  

- **Sticky Note3 (near Create a page node)**  
  - *Purpose:* Instructions for Notion credentials and page ID:  
    - Create Notion integration and obtain token  
    - Add integration to parent page  
    - Extract page ID from URL and set as parameter  

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                          | Input Node(s)                 | Output Node(s)                       | Sticky Note                                                                                                         |
|--------------------------------|--------------------|----------------------------------------|------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When you click 'Execute Workflow' | Manual Trigger     | Entry point to start the workflow      | None                         | Set Markdown                       |                                                                                                                     |
| Set Markdown                   | Set                | Defines sample Markdown content         | When you click 'Execute Workflow' | Create a page                     | ## Insert any Markdown output node here Use any node that outputs Markdown, like an LLM reply or GitHub issues       |
| Create a page                 | Notion             | Creates a new Notion page                | Set Markdown                 | HTTP Request - Mark2Notion Append | ## Set Notion Credentials 1. Create a Notion credential with the token from your Notion integration 2. Get the parent page ID — a new page will be created as a subpage 3. You can get the page ID from the URL: https://www.notion.so/Your-Page-Title-[PAGE_ID_HERE] 4. Set the page ID as the Parent Page (By ID) parameter |
| HTTP Request - Mark2Notion Append | HTTP Request       | Sends Markdown and tokens to Mark2Notion API | Create a page               | None                             | ## Set Mark2Notion and Notion credentials 1. Set Mark2Notion API key as Header Auth 2. Use `x-api-key` as the auth header name 3. Set `notionToken` as a body parameter |
| Sticky Note                   | Sticky Note        | Documentation and usage instructions    | None                         | None                              | Transform Markdown text into beautifully formatted Notion pages using the Mark2Notion API. This workflow handles all the complexity of Notion's block structure, including tables, nested lists, code blocks, and special formatting. ... (full content in node analysis) |
| Sticky Note1                  | Sticky Note        | Guidance on inserting Markdown outputs  | None                         | None                              | ## Insert any Markdown output node here Use any node that outputs Markdown, like an LLM reply or GitHub issues       |
| Sticky Note2                  | Sticky Note        | Credential setup instructions for Mark2Notion API | None                         | None                              | ## Set Mark2Notion and Notion credentials 1. Set Mark2Notion API key as Header Auth 2. Use `x-api-key` as the auth header name 3. Set `notionToken` as a body parameter |
| Sticky Note3                  | Sticky Note        | Credential setup instructions for Notion | None                         | None                              | ## Set Notion Credentials 1. Create a Notion credential with the token from your Notion integration 2. Get the parent page ID — a new page will be created as a subpage 3. You can get the page ID from the URL: https://www.notion.so/Your-Page-Title-[PAGE_ID_HERE] 4. Set the page ID as the Parent Page (By ID) parameter |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When you click 'Execute Workflow'`  
   - No parameters needed. This node starts the workflow on manual execution.

2. **Add a Set node**  
   - Name: `Set Markdown`  
   - Mode: Raw  
   - JSON output: Create a JSON object with a single field `markdown` containing your Markdown text as a multiline string (example provided in the original node).  
   - Connect output of `When you click 'Execute Workflow'` to this node.

3. **Add a Notion node**  
   - Name: `Create a page`  
   - Operation: Create a page  
   - Title: Set statically to `"Mark2Notion Test Page"` or your desired page title.  
   - Parent Page: Set by ID (paste your Notion parent page ID here if available)  
   - Credentials: Configure Notion credentials by creating a Notion integration (https://notion.so/my-integrations), obtain the token, and set it in n8n credentials.  
   - Connect output of `Set Markdown` to this node.

4. **Add an HTTP Request node**  
   - Name: `HTTP Request - Mark2Notion Append`  
   - Method: POST  
   - URL: `https://api.mark2notion.com/api/append`  
   - Authentication: HTTP Header Auth  
     - Create credentials with header name `x-api-key` and your Mark2Notion API key as the value (https://mark2notion.com)  
   - Body parameters (JSON):  
     - `markdown`: Expression referencing `Set Markdown` node's markdown field: `={{ $('Set Markdown').item.json.markdown }}`  
     - `notionToken`: Your Notion integration token (pass as a static or dynamic value)  
     - `pageId`: Expression referencing the created page ID: `={{ $json.id }}` from `Create a page` node  
   - Connect output of `Create a page` node to this HTTP Request node.

5. **(Optional) Add Sticky Note nodes** for documentation and guidance on credentials and usage.

6. **Configure credentials:**  
   - Mark2Notion API key as an HTTP Header Auth credential with header name `x-api-key`.  
   - Notion integration token as a Notion credential in n8n.

7. **Test the workflow:**  
   - Execute manually using the trigger node.  
   - Verify a new Notion page is created, and the Markdown content is appended and properly formatted by Mark2Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Transform Markdown text into beautifully formatted Notion pages using the Mark2Notion API. Handles complex Notion block structures including tables, nested lists, code blocks, and special formatting. Includes chunking, splitting, rate limiting. | Overview sticky note at (0,0) position              |
| Mark2Notion API key can be obtained for free with 100 requests/month at https://mark2notion.com                                                                                                                                                      | Usage instructions in sticky notes                  |
| Create a Notion integration at https://notion.so/my-integrations and add it to your parent page for proper permissions.                                                                                                                             | Credential setup notes in sticky notes               |
| The page ID can be extracted from the Notion page URL: https://www.notion.so/Your-Page-Title-[PAGE_ID_HERE]                                                                                                                                          | Sticky note near "Create a page" node               |
| Use any Markdown output node, such as AI language model replies or GitHub issues, by replacing the "Set Markdown" node with your dynamic source.                                                                                                    | Sticky note near "Set Markdown" node                 |

---

**Disclaimer:** The text provided is extracted solely from an automated n8n workflow. It complies fully with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.