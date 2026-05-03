Convert Notion Page to WordPress (Gutenberg) HTML

https://n8nworkflows.xyz/workflows/convert-notion-page-to-wordpress--gutenberg--html-9917


# Convert Notion Page to WordPress (Gutenberg) HTML

### 1. Workflow Overview

This workflow automates the conversion of a Notion page into WordPress Gutenberg-compatible HTML content. It is designed as a reusable sub-workflow that accepts a Notion page URL, fetches and processes the page content, and outputs a complete HTML string compatible with WordPress block editor.

The workflow logically divides into three main blocks:

**1.1 Input Reception and Notion Data Extraction**  
- Receives the Notion page URL either via manual trigger or as input from a parent workflow.  
- Fetches the main Notion database page metadata.  
- Retrieves all child blocks recursively, including nested blocks inside columns, toggles, etc.

**1.2 Content Transformation and HTML Conversion**  
- Parses and converts rich text annotations (bold, italic, underline, links) into HTML tags.  
- Maps each Notion block type (headings, paragraphs, images, videos, lists, dividers, etc.) to the corresponding WordPress Gutenberg block HTML structure.  
- Cleans up fields to retain only those necessary for further processing.  
- Rebuilds nested block structures such as columns and column lists, wrapping child blocks into proper HTML wrappers.

**1.3 Final Assembly of WordPress Content**  
- Filters to include only top-level blocks directly under the main page.  
- Creates a unified content field `wp` combining nested children or regular content.  
- Aggregates all blocks’ HTML into one collection.  
- Joins the aggregated HTML strings into a single final HTML output ready for WordPress.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Notion Data Extraction

**Overview:**  
This block initializes the workflow by receiving the Notion page URL and fetching all relevant Notion page data and child blocks needed for conversion.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- When Executed by Another Workflow  
- Edit Fields2  
- Get a database page  
- Get many child blocks

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution for testing purposes.  
  - Inputs: None  
  - Outputs: Connects to Edit Fields2 node.  
  - Failure Modes: None typical; manual trigger.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Sub-workflow trigger to receive `notion_url` from parent workflows.  
  - Inputs: Receives JSON input with key `notion_url`.  
  - Outputs: Connects to Get a database page node.  
  - Failure Modes: Missing or malformed input can cause failures.

- **Edit Fields2**  
  - Type: Set  
  - Role: Holds the `notion_url` string input for downstream nodes.  
  - Configuration: Sets `notion_url` field as string (empty by default for manual use).  
  - Inputs: From manual trigger.  
  - Outputs: Connects to Get a database page.  
  - Failure Modes: Incorrect or empty URL input will cause downstream Notion API call failure.

- **Get a database page**  
  - Type: Notion node (resource: databasePage, operation: get)  
  - Role: Fetches the main Notion page object metadata for the given page URL.  
  - Configuration: Uses `pageId` parameter extracted from `notion_url`.  
  - Credentials: Requires Notion API credentials.  
  - Inputs: Receives `notion_url` from Edit Fields2 or sub-workflow trigger.  
  - Outputs: Connects to Get many child blocks.  
  - Failure Modes: Invalid URL or permissions errors from Notion API.

- **Get many child blocks**  
  - Type: Notion node (resource: block, operation: getAll)  
  - Role: Recursively fetches all child blocks of the Notion page, including nested blocks.  
  - Configuration: Uses `blockId` from the page URL, with `fetchNestedBlocks` enabled for deep recursive fetch. Returns all results.  
  - Credentials: Notion API credentials.  
  - Inputs: From Get a database page node.  
  - Outputs: Connects to decode paragraphs node.  
  - Failure Modes: API rate limits, permission issues, or network errors.

---

#### 2.2 Content Transformation and HTML Conversion

**Overview:**  
This core block transforms raw Notion blocks into WordPress Gutenberg-compatible HTML blocks, handling text annotations and block-level HTML structure, including nested columns.

**Nodes Involved:**  
- decode paragraphs  
- decode blocks  
- drop unnecessary fields  
- nested blocks

**Node Details:**

- **decode paragraphs**  
  - Type: Code (JavaScript)  
  - Role: Converts rich text annotations (bold, italic, underline, strikethrough, links) into corresponding HTML markup for paragraph and list item blocks.  
  - Key Expressions: Iterates over text arrays, applies HTML tags based on annotation flags, preserves links.  
  - Inputs: From Get many child blocks node.  
  - Outputs: Connects to decode blocks.  
  - Failure Modes: Unexpected block types or missing text fields could cause errors.

- **decode blocks**  
  - Type: Code (JavaScript)  
  - Role: Maps each Notion block type (e.g., heading_1, code, video, image, audio, embed, divider, lists) to WordPress Gutenberg HTML block syntax.  
  - Key Expressions: Large switch statement on `$json.type` to assign `content` field with appropriate HTML.  
  - Special Handling:  
    - Videos embed YouTube or Vimeo links inside WordPress shortcode blocks.  
    - Images and audio blocks reference external or file URLs.  
    - Headings get wrapped in corresponding `<h1>`, `<h2>`, `<h3>` tags and Gutenberg comments.  
  - Inputs: From decode paragraphs.  
  - Outputs: Connects to drop unnecessary fields.  
  - Failure Modes: Unsupported block types default to raw HTML block, which might not render as expected.

- **drop unnecessary fields**  
  - Type: Set  
  - Role: Cleans block data by keeping only essential fields (`id`, `parent_id`, `type`, `content`, `has_children`) for further processing.  
  - Inputs: From decode blocks.  
  - Outputs: Connects to nested blocks.  
  - Failure Modes: None expected.

- **nested blocks**  
  - Type: Code (JavaScript)  
  - Role: Rebuilds the nested structure for `column` and `column_list` blocks by injecting HTML children inside appropriate Gutenberg column wrappers.  
  - Key Expressions:  
    - For each `column` block, gathers its child blocks’ HTML content and wraps in `<div class="wp-block-column">` tags with Gutenberg comments.  
    - For each `column_list`, gathers all child column HTML and wraps within `<div class="wp-block-columns">` tags.  
  - Inputs: From drop unnecessary fields.  
  - Outputs: Connects to only top level blocks filter.  
  - Failure Modes: Missing or malformed parent-child relationships could cause incomplete nesting.

---

#### 2.3 Final Assembly of WordPress Content

**Overview:**  
This block filters to top-level blocks, assembles the final HTML strings, and produces a single output suitable for WordPress import.

**Nodes Involved:**  
- only top level blocks  
- choose content field  
- Aggregate  
- join lines

**Node Details:**

- **only top level blocks**  
  - Type: Filter  
  - Role: Filters blocks to include only those whose `parent_id` matches the main Notion page ID (top-level blocks). Nested blocks are already injected in prior step.  
  - Inputs: From nested blocks.  
  - Outputs: Connects to choose content field.  
  - Failure Modes: Incorrect parent_id matching could exclude needed blocks.

- **choose content field**  
  - Type: Set  
  - Role: Creates a unified field `wp` which contains either nested children content or regular content HTML for each block.  
  - Configuration: Sets `wp = $json.children || $json.content`  
  - Inputs: From only top level blocks.  
  - Outputs: Connects to Aggregate.  
  - Failure Modes: Missing content fields could lead to empty results.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Merges all block items into a single item containing an array of all `wp` HTML strings.  
  - Configuration: Uses mergeLists option on `wp` field.  
  - Inputs: From choose content field.  
  - Outputs: Connects to join lines.  
  - Failure Modes: Large payloads might affect performance.

- **join lines**  
  - Type: Set  
  - Role: Joins the array of HTML strings from Aggregate into one single string separated by newlines, stored in the `wp` field.  
  - Inputs: From Aggregate.  
  - Outputs: Final output of the workflow.  
  - Failure Modes: None expected.

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                        | Input Node(s)                 | Output Node(s)            | Sticky Note                                                                                                                        |
|------------------------------|---------------------------|-------------------------------------|------------------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual start for testing             |                              | Edit Fields2               | See Sticky Note4 for how to trigger                                                                                              |
| When Executed by Another Workflow | Execute Workflow Trigger  | Sub-workflow trigger input           |                              | Get a database page        | See Sticky Note4 for how to trigger                                                                                              |
| Edit Fields2                 | Set                       | Holds input Notion page URL          | When clicking ‘Execute workflow’ | Get a database page        | See Sticky Note4 for manual input instructions                                                                                    |
| Get a database page          | Notion                    | Fetches main Notion page metadata    | Edit Fields2, When Executed by Another Workflow | Get many child blocks     | See Sticky Note1 for extraction details                                                                                          |
| Get many child blocks        | Notion                    | Fetches all child blocks recursively | Get a database page           | decode paragraphs          | See Sticky Note1 for extraction details                                                                                          |
| decode paragraphs            | Code                      | Converts rich text annotations to HTML | Get many child blocks         | decode blocks              | See Sticky Note2 for transformation logic                                                                                        |
| decode blocks               | Code                      | Maps Notion block types to Gutenberg HTML | decode paragraphs             | drop unnecessary fields    | See Sticky Note2 for transformation logic                                                                                        |
| drop unnecessary fields      | Set                       | Cleans and retains essential fields | decode blocks                 | nested blocks              | See Sticky Note2 for transformation logic                                                                                        |
| nested blocks               | Code                      | Rebuilds nested columns with child HTML | drop unnecessary fields       | only top level blocks      | See Sticky Note2 for transformation logic                                                                                        |
| only top level blocks        | Filter                    | Filters blocks to top-level only     | nested blocks                 | choose content field       | See Sticky Note3 for final assembly                                                                                              |
| choose content field         | Set                       | Creates unified `wp` content field   | only top level blocks          | Aggregate                  | See Sticky Note3 for final assembly                                                                                              |
| Aggregate                   | Aggregate                 | Aggregates all blocks’ HTML into array | choose content field           | join lines                 | See Sticky Note3 for final assembly                                                                                              |
| join lines                  | Set                       | Joins HTML array into single string  | Aggregate                    |                           | See Sticky Note3 for final assembly                                                                                              |
| Sticky Note1                | Sticky Note               | Extraction stage explanation          |                              |                           | Describes Get a database page and Get many child blocks                                                                          |
| Sticky Note2                | Sticky Note               | Transformation stage explanation      |                              |                           | Describes decode paragraphs, decode blocks, drop unnecessary fields, nested blocks                                                |
| Sticky Note3                | Sticky Note               | Final assembly stage explanation      |                              |                           | Describes only top level blocks, choose content field, Aggregate, join lines                                                    |
| Sticky Note4                | Sticky Note               | Trigger usage instructions            |                              |                           | Explains manual and sub-workflow triggers                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Manual start for testing.

2. **Create Execute Workflow Trigger node**  
   - Name: `When Executed by Another Workflow`  
   - Purpose: Sub-workflow entry point expecting an input parameter named `notion_url` (string).

3. **Create Set node**  
   - Name: `Edit Fields2`  
   - Purpose: Define string field `notion_url` (empty string by default).  
   - Connect `When clicking ‘Execute workflow’` → `Edit Fields2`.

4. **Create Notion node (Get a database page)**  
   - Name: `Get a database page`  
   - Credentials: Assign Notion API credentials.  
   - Parameters:  
     - Resource: `databasePage`  
     - Operation: `get`  
     - pageId: Set to expression extracting from `{{$json.notion_url}}` with mode URL.  
   - Connect both `Edit Fields2` and `When Executed by Another Workflow` main outputs to `Get a database page`.

5. **Create Notion node (Get many child blocks)**  
   - Name: `Get many child blocks`  
   - Credentials: Notion API credentials.  
   - Parameters:  
     - Resource: `block`  
     - Operation: `getAll`  
     - blockId: Expression from `{{$json.url}}` (URL mode).  
     - Return All: true  
     - Fetch Nested Blocks: true  
     - Simplify Output: false  
   - Connect `Get a database page` → `Get many child blocks`.

6. **Create Code node**  
   - Name: `decode paragraphs`  
   - Parameters: JavaScript code that converts Notion text annotations (bold, italic, underline, strikethrough, links) into HTML tags for paragraph and list item types.  
   - Execution Mode: Run once per item.  
   - Connect `Get many child blocks` → `decode paragraphs`.

7. **Create Code node**  
   - Name: `decode blocks`  
   - Parameters: JavaScript code with a switch-case on `$json.type`, mapping each Notion block type to WordPress Gutenberg HTML blocks (headings, paragraphs, images, videos, embed, dividers, lists, audio, code blocks).  
   - Execution Mode: Run once per item.  
   - Connect `decode paragraphs` → `decode blocks`.

8. **Create Set node**  
   - Name: `drop unnecessary fields`  
   - Parameters: Keep fields `id`, `parent_id`, `type`, `content`, `has_children`.  
   - Connect `decode blocks` → `drop unnecessary fields`.

9. **Create Code node**  
   - Name: `nested blocks`  
   - Parameters: JavaScript code that:  
     - For each `column` block, collect its child blocks’ `content` and wrap inside `<div class="wp-block-column">` with Gutenberg comments.  
     - For each `column_list` block, collect all child columns’ children HTML and wrap inside `<div class="wp-block-columns">` with Gutenberg comments.  
   - Execution Mode: Run once on all items.  
   - Connect `drop unnecessary fields` → `nested blocks`.

10. **Create Filter node**  
    - Name: `only top level blocks`  
    - Parameters: Filter blocks where `parent_id` (normalized by removing dashes) equals the main page ID (also normalized).  
    - Connect `nested blocks` → `only top level blocks`.

11. **Create Set node**  
    - Name: `choose content field`  
    - Parameters: Set field `wp` as `$json.children || $json.content`.  
    - Connect `only top level blocks` → `choose content field`.

12. **Create Aggregate node**  
    - Name: `Aggregate`  
    - Parameters:  
      - Merge Lists: true  
      - Field to Aggregate: `wp`  
    - Connect `choose content field` → `Aggregate`.

13. **Create Set node**  
    - Name: `join lines`  
    - Parameters: Set field `wp` as `{{$json.wp.join("\n")}}`, joining all HTML lines into one string.  
    - Connect `Aggregate` → `join lines`.

14. **Final Output:** The `join lines` node’s output contains the full WordPress-ready HTML string of the converted Notion page.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed exclusively as a sub-workflow and requires a valid Notion page URL input under the key `notion_url`. For manual testing, insert the URL in the `Edit Fields2` node before executing.                                                                                                                                                                                                   | See Sticky Note4                                                                                                              |
| The core transformation logic is implemented in two custom JavaScript code nodes (`decode paragraphs` and `decode blocks`), which handle rich text annotations and the mapping of Notion block types to WordPress Gutenberg HTML blocks.                                                                                                                                                                          | See Sticky Note2                                                                                                              |
| Nested blocks such as columns and column lists are reconstructed in the `nested blocks` code node to preserve layout structure in WordPress. The approach uses Gutenberg-specific HTML comments and div wrappers to ensure compatibility with the block editor.                                                                                                                                                 | See Sticky Note2                                                                                                              |
| The final assembly filters top-level blocks only, merges their HTML content, and joins into a single string, providing a clean, ready-to-publish WordPress post body in Gutenberg format.                                                                                                                                                                                                                       | See Sticky Note3                                                                                                              |
| Notion API credentials must be configured and authorized with access to the relevant Notion workspace and pages. API rate limits and permission errors are possible failure points and should be monitored.                                                                                                                                                                                                    | n8n Notion integration documentation                                                                                            |
| WordPress Gutenberg HTML blocks rely on specific HTML comments wrapping block content. This workflow emits those comments explicitly to ensure correct block recognition by WordPress.                                                                                                                                                                                                                          | WordPress Block Editor documentation                                                                                           |
| Videos from YouTube and Vimeo are embedded using WordPress shortcodes `[embed]URL[/embed]` inside Gutenberg shortcode blocks.                                                                                                                                                                                                                                                                                  | WordPress embed shortcodes documentation                                                                                      |

---

**Disclaimer:** The provided content originates exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All manipulated data are legal and public.