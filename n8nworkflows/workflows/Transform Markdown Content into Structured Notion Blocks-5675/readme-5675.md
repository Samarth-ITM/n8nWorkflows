Transform Markdown Content into Structured Notion Blocks

https://n8nworkflows.xyz/workflows/transform-markdown-content-into-structured-notion-blocks-5675


# Transform Markdown Content into Structured Notion Blocks

### 1. Workflow Overview

This workflow is designed to transform Markdown-formatted text content into structured Notion blocks, then update a specified Notion page with this content. It is suitable for users who want to automate the process of importing rich Markdown content into Notion while preserving formatting such as headings, lists, quotes, code blocks, and inline text styles.

The workflow is logically segmented into these blocks:

- **1.1 Input Reception:** Receives or generates Markdown content to process.
- **1.2 Markdown to Notion Blocks Conversion:** Parses the Markdown text and converts it into Notion's block format using a custom JavaScript function.
- **1.3 Notion Page Update:** Sends the generated Notion blocks to the Notion API to update the content of a specified Notion page.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  The workflow starts by manually triggering the execution and then providing mock Markdown data for testing or demonstration purposes.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Mock data (Set)  
  - Sticky Note (Mock data info)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to initiate the workflow manually.  
    - Configuration: No parameters; triggers workflow execution on user command.  
    - Inputs: None  
    - Outputs: Triggers the next node "Mock data".  
    - Failure Modes: None expected in normal use.  
    - Notes: None.

  - **Mock data**  
    - Type: Set  
    - Role: Provides dummy Markdown text in the `output` field to simulate real input data.  
    - Configuration: Assigns a Markdown-formatted string to the `output` variable.  
    - Inputs: Trigger from manual node.  
    - Outputs: JSON containing the Markdown text (key: `output`).  
    - Failure Modes: None if data is correctly assigned; no external dependencies.  
    - Notes:  
      > This node only serves the purpose of returning dummy Markdown-formatted text in the `output` field. Replace this with your actual data source.  
      (Sticky Note attached to this node.)

  - **Sticky Note (Mock data info)**  
    - Type: Sticky Note  
    - Role: Documentation comment explaining the Mock data node’s purpose.  
    - No inputs or outputs.

#### 1.2 Markdown to Notion Blocks Conversion

- **Overview:**  
  This block contains a single Code node that parses the Markdown text and converts it into an array of Notion block objects, which represent the content in Notion's API format.

- **Nodes Involved:**  
  - Convert into Notion blocks (Code)  
  - Sticky Note1 (Conversion explanation)

- **Node Details:**

  - **Convert into Notion blocks**  
    - Type: Code (JavaScript)  
    - Role: Converts Markdown input text into Notion block objects compatible with Notion API.  
    - Configuration:  
      - Reads Markdown text from incoming JSON field `output`.  
      - Optionally reads `page_id` from input JSON to include in output.  
      - Defines functions to parse Markdown line-by-line, detect block types such as headers, lists, quotes, code blocks, dividers, and paragraphs.  
      - Supports inline formatting: bold, italic, and links inside paragraphs or headings.  
      - Generates a plain text excerpt (max 300 chars) by stripping Markdown syntax for summary or preview use.  
      - Returns an object with keys: `blocks` (array of Notion blocks), `page_id` (if available), `original_output` (raw Markdown), `plain_text_excerpt`, and `block_count`.  
    - Key Expressions:  
      - Input: `$input.item.json.output` (Markdown text)  
      - Output: JSON object with Notion blocks and metadata.  
    - Inputs: From "Mock data" node.  
    - Outputs: To "Insert Notion page content" node.  
    - Failure Modes / Edge Cases:  
      - Invalid or empty Markdown input results in empty blocks array.  
      - Unclosed code blocks are handled gracefully by closing at end of text.  
      - Complex nested Markdown features (tables, nested lists, etc.) are not supported by this parser.  
      - Any JS runtime errors (e.g., unexpected input types) could cause node failure.  
    - Notes:  
      > This code converts the markdown string into the block format required by Notion.  
      (Sticky Note attached here.)

  - **Sticky Note1 (Conversion explanation)**  
    - Type: Sticky Note  
    - Role: Describes the purpose of the code node.  
    - No inputs or outputs.

#### 1.3 Notion Page Update

- **Overview:**  
  This HTTP Request node sends the generated Notion blocks to the Notion API, updating the content of a Notion page by patching its children blocks.

- **Nodes Involved:**  
  - Insert Notion page content (HTTP Request)  
  - Sticky Note2 (Update explanation)

- **Node Details:**

  - **Insert Notion page content**  
    - Type: HTTP Request  
    - Role: Sends a PATCH request to the Notion API to replace the children blocks of a given Notion page or block.  
    - Configuration:  
      - URL: `https://api.notion.com/v1/blocks/2255b6e0c94f80e7951df98d53ce2c39/children` (hardcoded block/page ID to update)  
      - Method: PATCH  
      - Body: JSON body with `children` parameter set to the array of Notion blocks from previous node (`={{ $json.blocks }}`)  
      - Headers: Includes `Notion-Version` header set to `2022-06-28`  
      - Authentication: Uses predefined Notion API credential configured in n8n (`notionApi`)  
      - Options: Batching enabled with batch size 100 to handle large block arrays in chunks  
    - Inputs: Receives blocks array from "Convert into Notion blocks" node.  
    - Outputs: Notion API response (not further used downstream).  
    - Failure Modes / Edge Cases:  
      - Invalid or expired Notion API credentials cause auth errors.  
      - Incorrect or missing page/block ID in URL leads to 404 or bad request errors.  
      - Large payloads might trigger rate limits or timeout.  
      - API changes or version mismatches could cause incompatibility.  
    - Notes:  
      > This node updates a Notion page with the blocks generated in the previous step. Replace the page ID in the URL field with your actual Notion page ID or an expression that returns the page ID.  
      (Sticky Note attached.)

  - **Sticky Note2 (Update explanation)**  
    - Type: Sticky Note  
    - Role: Provides instructions about the necessity to replace the hardcoded page ID with a real one or dynamic expression.  
    - No inputs or outputs.

---

### 3. Summary Table

| Node Name                  | Node Type         | Functional Role                                  | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                                        |
|----------------------------|-------------------|-------------------------------------------------|---------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger  | Starts workflow execution manually               | None                      | Mock data                   |                                                                                                                    |
| Mock data                  | Set               | Provides dummy Markdown-formatted text           | When clicking ‘Execute workflow’ | Convert into Notion blocks | This node only serves the purpose of returning dummy Markdown-formatted text in the `output` field. Replace this with your actual data source. |
| Convert into Notion blocks | Code              | Converts Markdown text into Notion block objects | Mock data                 | Insert Notion page content   | This code converts the markdown string into the block format required by Notion.                                  |
| Insert Notion page content | HTTP Request      | Updates Notion page content with generated blocks | Convert into Notion blocks | None                        | This node updates a Notion page with the blocks generated in the previous step. Replace the page ID in the URL field with your actual Notion page ID or an expression that returns the page ID. |
| Sticky Note                | Sticky Note       | Explains purpose of Mock data node                | None                      | None                        | This node only serves the purpose of returning dummy Markdown-formatted text in the `output` field. Replace this with your actual data source. |
| Sticky Note1               | Sticky Note       | Explains purpose of conversion code node          | None                      | None                        | This code converts the markdown string into the block format required by Notion.                                  |
| Sticky Note2               | Sticky Note       | Explains purpose of Notion update HTTP node       | None                      | None                        | This node updates a Notion page with the blocks generated in the previous step. Replace the page ID in the URL field with your actual Notion page ID or an expression that returns the page ID. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No configuration needed.

2. **Create Set Node for Mock Data**  
   - Name: `Mock data`  
   - Type: Set  
   - Add a string field `output` with sample Markdown text to simulate input.  
   - Example value:  
     ```
     # Doloria Vensat Ipsum
     
     Lorem versatio clarnum est vendora siltrix...
     ```
   - Connect output of Manual Trigger node to this node.

3. **Create Code Node to Convert Markdown to Notion Blocks**  
   - Name: `Convert into Notion blocks`  
   - Type: Code  
   - Language: JavaScript (Node.js)  
   - Paste the provided JavaScript code that:  
     - Reads input Markdown from `$input.item.json.output`.  
     - Parses Markdown line-by-line detecting headings, lists, quotes, code blocks, dividers, and paragraphs.  
     - Parses inline formatting (bold, italic, links).  
     - Builds an array of Notion block objects according to Notion API schema.  
     - Generates a plain text excerpt of first 300 characters without Markdown syntax.  
     - Returns JSON including `blocks`, `page_id` (if available), `original_output`, `plain_text_excerpt`, and `block_count`.  
   - Connect output of Mock data node to this node.

4. **Create HTTP Request Node to Update Notion Page**  
   - Name: `Insert Notion page content`  
   - Type: HTTP Request  
   - Method: PATCH  
   - URL: `https://api.notion.com/v1/blocks/<your-page-or-block-id>/children`  
     - Replace `<your-page-or-block-id>` with your actual Notion page or block ID (or use an expression if dynamic).  
   - Authentication: Use Notion API credentials configured in n8n (OAuth2 or Integration Token).  
   - Headers: Add header `Notion-Version` with value `2022-06-28`.  
   - Body Parameters:  
     - JSON mode with parameter `children` set to `={{ $json.blocks }}` to send converted blocks.  
   - Options: Enable batching with batch size 100 to handle large payloads efficiently.  
   - Connect output of Code node to this HTTP Request node.

5. **(Optional) Add Sticky Notes**  
   - Add sticky notes near each major node to document their purpose and instructions, for workflow clarity.

6. **Credentials Setup**  
   - Configure Notion API credentials in n8n credentials manager.  
   - Ensure the integration has permission to update blocks on the intended Notion page.

7. **Testing**  
   - Execute the workflow manually.  
   - Verify that the dummy Markdown content is converted and sent to the correct Notion page.  
   - Adjust the page/block ID as needed for your target environment.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The Markdown to Notion conversion script handles basic Markdown features including headings, lists, quotes, code blocks, and inline bold/italic/links formatting. | Conversion node JavaScript code.                                                                        |
| Replace the hardcoded Notion block/page ID in the HTTP Request node with your actual Notion page ID or use expressions to dynamically assign it.                   | Sticky Note attached to the HTTP Request node.                                                         |
| Notion API version header is set to `2022-06-28` to ensure compatibility with stable API features.              | HTTP Request node configuration.                                                                        |
| Mock data node is for demonstration only; replace with actual data source or trigger node for production use.   | Sticky Note attached to Mock data node.                                                                 |
| Batch size in HTTP Request node may be adjusted depending on the size of content and Notion API rate limits.    | HTTP Request node options.                                                                               |

---

**Disclaimer:**  
The text provided here is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.