Convert Notion to Markdown and Back to Notion

https://n8nworkflows.xyz/workflows/convert-notion-to-markdown-and-back-to-notion-2901


# Convert Notion to Markdown and Back to Notion

### 1. Workflow Overview

This workflow automates the conversion of Notion page content into Markdown format and then back into Notion blocks, effectively tripling the content of the most recently updated page in a specified Notion database. It is designed primarily as a demonstration and template for users who want to manipulate Notion content programmatically using n8n.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Input Retrieval:** Detects the most recently updated page in a Notion database and retrieves its child blocks with rich text data.
- **1.2 Conversion to Markdown:** Converts the retrieved Notion blocks into Markdown format, preserving formatting such as headings, lists, and text annotations.
- **1.3 Conversion from Markdown to Notion Blocks:** Parses the Markdown back into Notion block objects, reconstructing rich text annotations.
- **1.4 Append Converted Blocks to Notion Page:** Appends the newly created Notion blocks back to the original page, effectively duplicating (tripling) the content.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Retrieval

**Overview:**  
This block listens for updates to pages within a specified Notion database and retrieves the full rich text child blocks of the updated page using a custom HTTP request (to overcome limitations of the default Notion node).

**Nodes Involved:**  
- Notion Trigger  
- Notion  
- Get Child blocks  
- Split Out  
- Sticky Note (configuration guidance)

**Node Details:**

- **Notion Trigger**  
  - *Type:* Notion Trigger  
  - *Role:* Watches for page updates in a specific Notion database (polls every minute).  
  - *Configuration:* Event set to "pagedUpdatedInDatabase", database ID specified, Notion API credentials used.  
  - *Input/Output:* No input; outputs updated page metadata including page ID.  
  - *Edge Cases:* Missed triggers if polling interval is too long; API rate limits; database ID must be correct and accessible.  
  - *Sticky Note:* Advises configuring a Notion connection.

- **Notion**  
  - *Type:* Notion node (built-in)  
  - *Role:* Retrieves all blocks of the updated page using the page ID from the trigger.  
  - *Configuration:* Operation "getAll" on resource "block", returns all blocks for the page ID.  
  - *Input:* Page ID from Notion Trigger.  
  - *Output:* List of blocks with limited formatting (plain text only).  
  - *Edge Cases:* Limited formatting support; may miss rich text annotations.  
  - *Sticky Note:* Notes that this node removes formatting like bold and links.

- **Get Child blocks**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves child blocks of the page with full rich text data, including formatting and links, via Notion API.  
  - *Configuration:* GET request to `https://api.notion.com/v1/blocks/{{ $json.id }}/children`, uses Notion API credentials.  
  - *Input:* Page ID from Notion Trigger.  
  - *Output:* Full block data with rich text annotations.  
  - *Edge Cases:* API rate limits, authentication errors, page ID must be valid and shared with the integration.  
  - *Sticky Note:* Indicates this method is used to get rich text data.

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the array of blocks from the HTTP response into individual items for processing.  
  - *Configuration:* Splits on the "results" field of the HTTP response.  
  - *Input:* HTTP response from Get Child blocks.  
  - *Output:* Individual block items for further processing.  
  - *Edge Cases:* Empty or malformed "results" array.

- **Sticky Notes**  
  - Provide user guidance on configuration and node choice.

---

#### 2.2 Conversion to Markdown

**Overview:**  
Converts the retrieved Notion blocks into Markdown text, preserving headings, lists, to-dos, paragraphs, and basic formatting.

**Nodes Involved:**  
- Notion Node Blocks to Md  
- Full Notion Blocks to Md  
- Sticky Note (explaining differences)

**Node Details:**

- **Notion Node Blocks to Md**  
  - *Type:* Code node  
  - *Role:* Converts simplified Notion blocks (from the built-in Notion node) to Markdown.  
  - *Configuration:* JavaScript function mapping block types to Markdown syntax (e.g., headings to `#`, lists to `-`). Skips unsupported types.  
  - *Input:* Blocks from the Notion node.  
  - *Output:* Single Markdown string combining all blocks.  
  - *Edge Cases:* Ignores blocks without content; limited formatting support; may produce empty lines.

- **Full Notion Blocks to Md**  
  - *Type:* Code node  
  - *Role:* Converts rich Notion blocks (from HTTP request) to Markdown, including rich text annotations like bold, italic, strikethrough, underline, code, and links.  
  - *Configuration:* Parses rich text arrays and applies Markdown syntax accordingly; supports multiple block types including quotes and code blocks.  
  - *Input:* Individual blocks from Split Out node.  
  - *Output:* Markdown string with rich formatting preserved.  
  - *Edge Cases:* Unsupported block types are skipped; complex nested blocks not handled; malformed rich text could cause parsing errors.  
  - *Sticky Note:* Explains this node is used to get rich text data.

---

#### 2.3 Conversion from Markdown to Notion Blocks

**Overview:**  
Parses the Markdown string back into Notion block objects with rich text annotations, reconstructing the original formatting.

**Nodes Involved:**  
- Md to Notion Blocks v3  
- Sticky Note (explaining demo purpose)

**Node Details:**

- **Md to Notion Blocks v3**  
  - *Type:* Code node  
  - *Role:* Parses Markdown line-by-line and converts it into an array of Notion block objects with appropriate types and rich text annotations.  
  - *Configuration:* JavaScript function that handles headings, bulleted lists, quotes, code blocks, and paragraphs; uses regex to detect bold, italic, code, and links in text.  
  - *Input:* Markdown string from previous code nodes.  
  - *Output:* JSON array of Notion blocks ready for API submission.  
  - *Edge Cases:* Complex Markdown structures (nested lists, tables) not supported; code block parsing assumes triple backticks; regex may miss edge cases in Markdown syntax.  
  - *Sticky Note:* Notes this will triple the content as a demo.

---

#### 2.4 Append Converted Blocks to Notion Page

**Overview:**  
Appends the newly created Notion blocks as children to the original Notion page, effectively duplicating the content.

**Nodes Involved:**  
- Add blocks as Children

**Node Details:**

- **Add blocks as Children**  
  - *Type:* HTTP Request  
  - *Role:* Sends a PATCH request to the Notion API to append the new blocks as children of the original page block.  
  - *Configuration:* URL constructed dynamically using the page ID from the Notion Trigger node; JSON body contains the array of blocks from the previous node; uses Notion API credentials.  
  - *Input:* Notion blocks array from Md to Notion Blocks v3.  
  - *Output:* API response confirming the append operation.  
  - *Edge Cases:* API rate limits, authentication errors, invalid block structure, page permissions.  
  - *Sticky Note:* None.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                  |
|---------------------------|---------------------|----------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Notion Trigger            | Notion Trigger      | Detects updated pages in Notion DB     | —                       | Notion, Get Child blocks  | ## Configure a notion connection.                                                           |
| Notion                   | Notion              | Retrieves blocks (plain text only)     | Notion Trigger          | Notion Node Blocks to Md  | ## Either use the official Notion getAll: Blocks node This removes formatting like bold and links. |
| Get Child blocks          | HTTP Request        | Retrieves rich text child blocks via API | Notion Trigger          | Split Out                 | ## ... or get block rich text data with custom HTTP request.                                |
| Split Out                 | Split Out           | Splits array of blocks into items      | Get Child blocks        | Full Notion Blocks to Md  | ## ... or get block rich text data with custom HTTP request.                                |
| Notion Node Blocks to Md  | Code                | Converts plain blocks to Markdown       | Notion                  | Md to Notion Blocks v3    | ## Either use the official Notion getAll: Blocks node This removes formatting like bold and links. |
| Full Notion Blocks to Md  | Code                | Converts rich blocks to Markdown        | Split Out               | Md to Notion Blocks v3    | ## ... or get block rich text data with custom HTTP request.                                |
| Md to Notion Blocks v3    | Code                | Converts Markdown back to Notion blocks | Notion Node Blocks to Md, Full Notion Blocks to Md | Add blocks as Children | ## This will triple the content by way of demo.                                             |
| Add blocks as Children    | HTTP Request        | Appends new blocks to original page    | Md to Notion Blocks v3  | —                        |                                                                                              |
| Sticky Note               | Sticky Note         | Guidance on Notion node limitations    | —                       | —                        | ## Either use the official Notion getAll: Blocks node This removes formatting like bold and links. |
| Sticky Note1              | Sticky Note         | Guidance on HTTP request for rich text | —                       | —                        | ## ... or get block rich text data                                                          |
| Sticky Note2              | Sticky Note         | Guidance on Notion credential setup    | —                       | —                        | ## Configure a notion connection.                                                           |
| Sticky Note3              | Sticky Note         | Demo purpose note on content tripling  | —                       | —                        | ## This will triple the content by way of demo.                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Notion Credentials**  
   - In n8n, create a new Notion API credential following https://docs.n8n.io/integrations/builtin/credentials/notion/  
   - Share the relevant Notion pages or databases with the integration.

2. **Add Notion Trigger Node**  
   - Type: Notion Trigger  
   - Event: "pagedUpdatedInDatabase"  
   - Polling: Every minute  
   - Database ID: Set to your target Notion database ID  
   - Credentials: Select your Notion credential

3. **Add Notion Node**  
   - Type: Notion  
   - Resource: Block  
   - Operation: Get All  
   - Block ID: Set to `={{ $json.id }}` from Notion Trigger output  
   - Credentials: Same Notion credential

4. **Add HTTP Request Node (Get Child blocks)**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.notion.com/v1/blocks/{{ $json.id }}/children` (use expression to insert page ID)  
   - Authentication: Use Notion API credential  
   - Purpose: Retrieve rich text child blocks with formatting

5. **Add Split Out Node**  
   - Type: Split Out  
   - Field to split out: `results` (from HTTP response)  
   - Purpose: Process each block individually

6. **Add Code Node (Full Notion Blocks to Md)**  
   - Type: Code  
   - JavaScript: Use the provided function to convert rich Notion blocks to Markdown, preserving formatting (headings, bold, italic, links, etc.)  
   - Input: Output from Split Out node

7. **Add Code Node (Notion Node Blocks to Md)**  
   - Optional alternative to step 6 for plain text blocks from the Notion node  
   - Converts simpler blocks to Markdown without rich formatting

8. **Add Code Node (Md to Notion Blocks v3)**  
   - Type: Code  
   - JavaScript: Use the provided function to parse Markdown back into Notion block objects with rich text annotations  
   - Input: Markdown string from either Markdown conversion node

9. **Add HTTP Request Node (Add blocks as Children)**  
   - Type: HTTP Request  
   - Method: PATCH  
   - URL: `https://api.notion.com/v1/blocks/{{ $('Notion Trigger').first().json.id }}/children`  
   - Body Content-Type: JSON  
   - Body: `{ "children": {{ $json.blocks.toJsonString() }} }`  
   - Authentication: Notion API credential  
   - Purpose: Append the new blocks to the original Notion page

10. **Connect Nodes in Order:**  
    - Notion Trigger → Notion  
    - Notion Trigger → Get Child blocks  
    - Get Child blocks → Split Out  
    - Split Out → Full Notion Blocks to Md  
    - Notion → Notion Node Blocks to Md  
    - Notion Node Blocks to Md → Md to Notion Blocks v3  
    - Full Notion Blocks to Md → Md to Notion Blocks v3  
    - Md to Notion Blocks v3 → Add blocks as Children

11. **Add Sticky Notes (Optional)**  
    - Add sticky notes near nodes to provide user guidance as in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Create a Notion credential and share pages as described in the official n8n docs.               | https://docs.n8n.io/integrations/builtin/credentials/notion/                                   |
| The built-in Notion node "Get All Blocks" removes formatting like bold and links.                | Sticky Note near Notion node                                                                    |
| Using a custom HTTP request to Notion API allows retrieval of rich text data including links.   | Sticky Note near Get Child blocks node                                                          |
| This workflow is a demo that triples the content of the last updated page to illustrate conversion. | Sticky Note near Md to Notion Blocks v3 node                                                    |
| Future improvements may include official n8n nodes supporting Markdown extraction and writing.  | Workflow description and sticky notes                                                          |
| Community blocks exist for Markdown-Notion conversion but this workflow is simpler to adapt.    | Workflow description                                                                            |

---

This documentation fully describes the workflow’s structure, node configurations, and logic, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.