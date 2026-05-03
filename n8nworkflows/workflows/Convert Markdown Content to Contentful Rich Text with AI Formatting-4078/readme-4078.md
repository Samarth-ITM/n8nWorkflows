Convert Markdown Content to Contentful Rich Text with AI Formatting

https://n8nworkflows.xyz/workflows/convert-markdown-content-to-contentful-rich-text-with-ai-formatting-4078


# Convert Markdown Content to Contentful Rich Text with AI Formatting

---

### 1. Workflow Overview

This workflow automates the conversion of Markdown content into Contentful-compatible Rich Text format, leveraging AI for precise formatting. Its primary use case is for content creators or CMS managers who want to upload richly formatted articles into Contentful while ensuring compliance with Contentful’s strict Rich Text JSON schema. The workflow receives structured article data, splits the Markdown content into manageable chunks, converts each chunk into validated Contentful Rich Text JSON using an AI agent, merges the chunks, formats the final entry, and publishes it to Contentful via API.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives article data as JSON input from another workflow or trigger.
- **1.2 Content Splitting:** Divides Markdown content into smaller chunks based on heading levels for manageable AI processing.
- **1.3 AI Formatting:** Uses an AI language model to convert each Markdown chunk into fully compliant Contentful Rich Text JSON.
- **1.4 Content Merging:** Aggregates all chunk outputs into a single Contentful Rich Text document.
- **1.5 Final Formatting:** Integrates metadata and references into the final Contentful entry structure.
- **1.6 Contentful API Publishing:** Sends the fully prepared entry to Contentful’s Management API for creation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow by receiving a fully structured article JSON object. It acts as the entry point for external workflows or manual executions.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Sticky Note2 (informational)

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Receives input JSON representing article data with fields like title, slug, category, description, keywords, content (Markdown), meta data, readingTime, difficulty.  
  - Configuration: Uses a JSON example to define expected input schema.  
  - Inputs: Triggered externally or manually with JSON data.  
  - Outputs: Passes article data downstream.  
  - Edge cases: Missing or malformed input fields could cause downstream errors.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Content: Describes availability of free workflows from Varritech and usage instructions.  
  - Purpose: Provides contextual information and branding, no execution role.

---

#### 2.2 Content Splitting

**Overview:**  
Splits the full Markdown content into chunks based on level 3 headings (`###`) to enable focused AI processing on smaller content blocks.

**Nodes Involved:**  
- Split by Headings

**Node Details:**

- **Split by Headings**  
  - Type: Code  
  - Role: Uses JavaScript regex to split the Markdown content string into an array of chunks at each `###` heading.  
  - Configuration: Splits on heading level 3 markers, trims whitespace, and creates output items each containing an index, slug, title, and one content chunk.  
  - Inputs: Receives full article JSON from previous node.  
  - Outputs: Multiple items, each with a Markdown chunk to convert.  
  - Edge cases: Content without level 3 headings returns a single chunk; malformed Markdown might affect splitting accuracy.

---

#### 2.3 AI Formatting

**Overview:**  
Transforms each Markdown chunk into a Contentful Rich Text JSON object using an AI language model with strict formatting rules.

**Nodes Involved:**  
- OpenAI Chat Model2  
- Markdown to Contentful format  
- Sticky Note (e1bbe1ba) (informational)

**Node Details:**

- **OpenAI Chat Model2**  
  - Type: AI Language Model (OpenAI GPT-4.1)  
  - Role: Processes each Markdown chunk text input to generate Contentful Rich Text JSON output.  
  - Configuration: Uses GPT-4.1 model with no special options.  
  - Credentials: Requires OpenAI API key credential configured.  
  - Inputs: Receives Markdown chunk text via expression referencing `contentChunk`.  
  - Outputs: AI-generated JSON string of Contentful Rich Text compliant formatting.  
  - Edge cases: API timeouts, rate limits, malformed prompt input could cause failures.

- **Markdown to Contentful format**  
  - Type: Langchain Agent (AI agent wrapper)  
  - Role: Defines detailed system message with formatting guidelines and examples to instruct AI on producing valid Contentful Rich Text JSON.  
  - Key Configuration:  
    - System message includes strict rules for node structure, data inclusion, marks, content arrays, node types, and examples for paragraphs, headings, lists, links, images, and more.  
    - Input text is injected from Markdown chunk.  
  - Inputs: Text from Split by Headings node, AI model from OpenAI Chat Model2.  
  - Outputs: JSON string representing Contentful Rich Text format for the chunk.  
  - Edge cases: AI output might not be parseable JSON; strict prompt design mitigates this, but parse errors are handled downstream.  
  - Sticky Note (e1bbe1ba) explains the node’s purpose and references Contentful documentation.

---

#### 2.4 Content Merging

**Overview:**  
Combines all individual Rich Text JSON chunks from AI into a single Contentful Rich Text document with a unified `document` root node.

**Nodes Involved:**  
- Combine Rich Text Objects

**Node Details:**

- **Combine Rich Text Objects**  
  - Type: Code  
  - Role: Iterates over AI output items, parses JSON strings if necessary, extracts each chunk’s `content` array, and concatenates them into the `content` array of a single Contentful `document` node.  
  - Configuration:  
    - Handles cases where AI output is nested inside an `output` key or is raw stringified JSON.  
    - Throws errors on JSON parsing failures for robust error detection.  
  - Inputs: Receives multiple AI-generated Rich Text JSON chunks.  
  - Outputs: Single item with full combined Rich Text JSON document.  
  - Edge cases: JSON parsing errors, missing content arrays, empty chunks.

---

#### 2.5 Final Formatting

**Overview:**  
Prepares the final Contentful entry payload by integrating metadata fields and the combined Rich Text content into the expected Contentful Entry JSON structure.

**Nodes Involved:**  
- Merge1  
- Format1

**Node Details:**

- **Merge1**  
  - Type: Merge  
  - Role: Joins two streams: original article metadata and combined Rich Text content.  
  - Configuration: Defaults to merge on index or first item; typically joins metadata with formatted content.  
  - Inputs:  
    - From Split by Headings (metadata)  
    - From Combine Rich Text Objects (content)  
  - Outputs: Merged JSON containing metadata and content for formatting.

- **Format1**  
  - Type: Code  
  - Role: Constructs the Contentful entry JSON, embedding all fields with English locale keys (`en-US`).  
  - Configuration:  
    - Overwrites `content` field with the combined Rich Text content.  
    - Wraps category as a Contentful reference link object.  
    - Structures fields exactly as required by Contentful Management API including title, slug, category, description, keywords, meta fields, readingTime, difficulty, and content.  
  - Inputs: Merged metadata and combined content.  
  - Outputs: JSON ready for Contentful API submission.  
  - Edge cases: Missing fields in input may cause incomplete payloads.

---

#### 2.6 Contentful API Publishing

**Overview:**  
Sends the fully formatted Contentful entry JSON to Contentful Management API to create a new entry in the specified space and environment.

**Nodes Involved:**  
- Create newly formatted Contentful Entry  
- Sticky Note1 (informational)  
- Sticky Note3 (informational)

**Node Details:**

- **Create newly formatted Contentful Entry**  
  - Type: HTTP Request  
  - Role: POSTs the formatted JSON entry to Contentful API endpoint for entry creation.  
  - Configuration:  
    - URL template requires user to replace `{INSERT_YOUR_SPACE}` with actual Contentful space ID.  
    - HTTP Headers include Authorization Bearer token (Management API token must be inserted), Content-Type for Contentful management, X-Contentful-Version, and Content-Type ID for the entry (`knowledgeBaseArticle`).  
    - Sends JSON body from previous node’s output.  
  - Inputs: Fully formatted Contentful entry JSON.  
  - Outputs: API response (success or error).  
  - Edge cases: Authentication failure, invalid token, missing space ID, API rate limits, or payload validation errors.  
  - Sticky Notes provide detailed field descriptions and requirements for this node.

- **Sticky Note1**  
  - Content: Detailed publishing instructions, required Contentful API parameters, and field descriptions for the entry.  
  - Purpose: Guidance for user configuration of API node.

- **Sticky Note3**  
  - Content: Offers help and consulting services related to Contentful implementations and automation.  
  - Purpose: Informational for users needing advanced support.

---

### 3. Summary Table

| Node Name                        | Node Type                           | Functional Role                      | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                               |
|---------------------------------|-----------------------------------|------------------------------------|----------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger           | Entry point; receives article JSON | None                             | Split by Headings, Merge1          |                                                                                                                          |
| Split by Headings               | Code                              | Splits Markdown into chunks         | When Executed by Another Workflow | Markdown to Contentful format       |                                                                                                                          |
| OpenAI Chat Model2              | AI Language Model (OpenAI GPT-4.1) | Converts Markdown chunk to Rich Text JSON | Markdown to Contentful format (ai_languageModel) | Markdown to Contentful format       |                                                                                                                          |
| Markdown to Contentful format    | Langchain Agent (AI agent)         | Defines AI prompt & converts chunk  | Split by Headings, OpenAI Chat Model2 (ai_languageModel) | Combine Rich Text Objects           | Converts to Proprietary Format for Contentful; references Contentful docs                                                |
| Combine Rich Text Objects        | Code                              | Merges AI chunk outputs into one    | Markdown to Contentful format     | Merge1                            |                                                                                                                          |
| Merge1                         | Merge                             | Combines metadata & combined content | When Executed by Another Workflow, Combine Rich Text Objects | Format1                          |                                                                                                                          |
| Format1                        | Code                              | Prepares final Contentful entry JSON | Merge1                          | Create newly formatted Contentful Entry |                                                                                                                          |
| Create newly formatted Contentful Entry | HTTP Request                     | Publishes entry to Contentful API   | Format1                         | None                             | Publishes to Contentful API; requires space ID & token; detailed field info provided                                     |
| Sticky Note                    | Sticky Note                       | Information on AI formatting         | None                            | None                             | Converts to Proprietary Format for Contentful; references contentful docs                                                |
| Sticky Note1                   | Sticky Note                       | Info on Contentful API publishing    | None                            | None                             | Publishes to Contentful API; requirements and field descriptions                                                        |
| Sticky Note2                   | Sticky Note                       | Branding and free workflows info     | None                            | None                             | Varritech Free Workflows for n8n; link to varritech.com                                                                  |
| Sticky Note3                   | Sticky Note                       | Support and consulting info          | None                            | None                             | Need Additional Help With Contentful? Contact varritech.com                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure with JSON example input containing fields: `title`, `slug`, `category` (with `id`), `description`, `keywords` (array), `content` (Markdown string), `metaTitle`, `metaDescription`, `readingTime`, `difficulty`.

2. **Add Content Splitting Node**  
   - Add a **Code** node named `Split by Headings`.  
   - Paste JavaScript code to split the Markdown content by level 3 headings (`###`).  
   - Input is the JSON from the trigger node.  
   - Output multiple items each containing a chunk of Markdown as `contentChunk`.

3. **Add OpenAI Chat Model Node**  
   - Add **AI Language Model (OpenAI GPT)** node named `OpenAI Chat Model2`.  
   - Select model `gpt-4.1`.  
   - Configure with your OpenAI API credentials.  
   - Connect it as AI language model input for next node.

4. **Add AI Formatting Node**  
   - Add **Langchain Agent** node named `Markdown to Contentful format`.  
   - Configure its system message with detailed instructions for converting Markdown to Contentful Rich Text JSON (use provided prompt content).  
   - Set prompt input as expression referencing `contentChunk` from `Split by Headings`.  
   - Connect AI model node as its language model source.  
   - This node produces JSON-convertible Contentful Rich Text for each chunk.

5. **Add Content Merging Node**  
   - Add **Code** node named `Combine Rich Text Objects`.  
   - Paste code that iterates all chunk outputs, parses JSON safely, and merges `content` arrays into a single Contentful Rich Text `document` object.

6. **Add Merge Node**  
   - Add **Merge** node named `Merge1`.  
   - Connect original metadata stream from `When Executed by Another Workflow` into one input.  
   - Connect merged content stream from `Combine Rich Text Objects` into the second input.  
   - Configure default merge strategy (e.g., merge by index).

7. **Add Final Formatting Node**  
   - Add **Code** node named `Format1`.  
   - Paste code that builds Contentful entry JSON structure, embedding metadata and combined Rich Text content.  
   - Use English locale keys (`en-US`).  
   - Wrap category as Contentful reference link object.

8. **Add HTTP Request Node for Publishing**  
   - Add **HTTP Request** node named `Create newly formatted Contentful Entry`.  
   - Configure method: POST.  
   - URL: `https://api.contentful.com/spaces/{INSERT_YOUR_SPACE}/environments/master/entries` (replace `{INSERT_YOUR_SPACE}` with your Contentful space ID).  
   - Headers:  
     - Authorization: `Bearer {INSERT TOKEN HERE}` (Contentful Management API token)  
     - Content-Type: `application/vnd.contentful.management.v1+json`  
     - X-Contentful-Version: `2`  
     - X-Contentful-Content-Type: `knowledgeBaseArticle` (or your content type ID)  
   - Body: JSON from `Format1` node.  
   - Send headers and body as JSON.

9. **Connect the Nodes**  
   - Connect `When Executed by Another Workflow` → `Split by Headings` → `Markdown to Contentful format` → `Combine Rich Text Objects` → `Merge1` → `Format1` → `Create newly formatted Contentful Entry`.  
   - Connect `Combine Rich Text Objects` second input of `Merge1`.  
   - Connect `When Executed by Another Workflow` first input of `Merge1`.

10. **Add Sticky Notes for Documentation**  
    - Add sticky notes as per original workflow for user guidance on AI formatting, Contentful API publishing, and Varritech branding/support info.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------|
| Converts to Proprietary Format for Contentful - Uses example output format to generate content for specific Contentful Rich Text formatting. References Contentful documentation for Rich Text JSON structure and validation rules.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note attached to AI formatting nodes |
| Publishes to Contentful API - Requires user to insert their Contentful space ID and Management API token. Detailed field descriptions for the article entry including title, slug, category reference, description, keywords, meta title, meta description, difficulty level, related articles, content (Rich Text), and reading time.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note attached to HTTP request node    |
| Varritech Free Workflows for n8n - Collection of free, ready-to-use n8n workflows with pre-built data integrations, automation templates, and implementation guides. Visit [varritech.com](https://varritech.com) for downloads and support.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note at workflow start                |
| Need Additional Help With Contentful? - Varritech offers consulting for custom Contentful implementations, advanced content modeling, API integration optimization, content migration, and automation workflow extension. Visit [varritech.com](https://varritech.com) for expert support.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Sticky Note near API publishing node         |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.