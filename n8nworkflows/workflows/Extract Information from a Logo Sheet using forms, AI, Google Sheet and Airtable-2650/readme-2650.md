Extract Information from a Logo Sheet using forms, AI, Google Sheet and Airtable

https://n8nworkflows.xyz/workflows/extract-information-from-a-logo-sheet-using-forms--ai--google-sheet-and-airtable-2650


# Extract Information from a Logo Sheet using forms, AI, Google Sheet and Airtable

---

### 1. Workflow Overview

This n8n workflow automates the extraction of structured information from an uploaded image of a logo sheet (typically containing multiple product or tool logos in contextual grouping). It leverages AI vision capabilities to identify each product/tool, extract its attributes, and detect similarities between tools based on the visual context. The extracted data is then systematically upserted into Airtable tables for "Tools" and "Attributes," ensuring no duplicates and maintaining relational integrity via record IDs.

**Target use cases:**  
- Quickly ingest and catalog multiple product logos and their attributes from a single image upload.  
- Automate the enrichment and organization of product/tool databases with minimal manual input.  
- Enable continuous growth of a structured Airtable repository with contextual relationships (similarities) among tools.

**Logical Blocks:**

- **1.1 Input Reception and Preparation**  
  Handles form submission with image upload and optional prompt; prepares inputs for AI processing.

- **1.2 AI Processing and Parsing**  
  Uses a LangChain AI Agent with vision capabilities to extract structured JSON of tools, their attributes, and similarity relations.

- **1.3 Attribute Handling and Upsert**  
  Processes extracted attributes, checks existence in Airtable, creates missing attributes, and maps them to record IDs.

- **1.4 Tool Handling and Upsert**  
  Processes each tool, generates unique hashes as IDs, creates or updates tool records in Airtable, and merges existing data.

- **1.5 Similarity (Competitor) Mapping**  
  Processes similarity relations, ensures referenced similar tools exist in Airtable, maps their record IDs, and updates tool records accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preparation

- **Overview:**  
  This block waits for a user to submit a form with an image of a logo sheet and an optional text prompt. It prepares the prompt field for AI consumption.

- **Nodes Involved:**  
  - On form submission  
  - Map Agent Input  
  - gpt-4o (OpenAI Chat Model)  

- **Node Details:**  

  - **On form submission**  
    - Type: Form Trigger (Webhook)  
    - Role: Entry point; receives uploaded image and optional prompt from users via a public form at `/logo-sheet-feeder`.  
    - Configuration: Requires file upload (logo sheet image) and allows optional prompt for context.  
    - Inputs: HTTP request from form  
    - Outputs: Data object containing image binary and prompt text  
    - Edge Cases: Missing file (required), malformed uploads, slow uploads causing timeout.

  - **Map Agent Input**  
    - Type: Set node  
    - Role: Maps user prompt text to `chatInput` variable for AI agent.  
    - Configuration: Copies the prompt field from form to `chatInput`.  
    - Inputs: Output of form submission node  
    - Outputs: JSON with `chatInput` property  
    - Edge Cases: Empty or missing prompt field handled gracefully (optional).

  - **gpt-4o**  
    - Type: LangChain OpenAI Chat Node  
    - Role: Runs GPT-4o model to interpret the prompt and image.  
    - Configuration: Calls OpenAI API with model "gpt-4o".  
    - Inputs: `chatInput` text and attached image binary  
    - Outputs: AI-generated textual response  
    - Edge Cases: API rate limits, authentication errors, model latency, malformed prompt.

---

#### 1.2 AI Processing and Parsing

- **Overview:**  
  This block uses an AI agent specialized in vision to extract structured JSON of tools, attributes, and similarities from the logo sheet image. It then parses the output into usable JSON.

- **Nodes Involved:**  
  - Retrieve and Parser Agent  
  - Structured Output Parser  
  - JSON it  
  - Split Out Tools  

- **Node Details:**  

  - **Retrieve and Parser Agent**  
    - Type: LangChain Agent (AI Vision + Language)  
    - Role: Extracts information from the uploaded image and prompt, outputs structured JSON of tools with attributes and similarities.  
    - Configuration: System prompt instructs agent to produce a JSON array of objects with `name`, `attributes`, and `similar` fields per tool.  
    - Inputs: Output from GPT-4o model and image binary (passthrough)  
    - Outputs: Raw JSON string in `output` property  
    - Edge Cases: Partial extraction, missing tools, noisy images, AI hallucination.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser  
    - Role: Validates and structures the AI response according to a JSON schema example defining tools array structure.  
    - Inputs: Raw AI output from agent  
    - Outputs: Structured JSON object with `tools` array  
    - Edge Cases: JSON parsing errors, schema mismatches, invalid AI output.

  - **JSON it**  
    - Type: Set  
    - Role: Extracts the parsed JSON from AI agent output and sets it as main JSON flow data for next operations.  
    - Inputs: Structured JSON parser output  
    - Outputs: Parsed JSON with `tools` array ready for splitting  
    - Edge Cases: Empty or malformed JSON.

  - **Split Out Tools**  
    - Type: Split Out  
    - Role: Splits the `tools` array into individual tool objects for iterative processing.  
    - Inputs: JSON object with tools array  
    - Outputs: Individual tool JSON objects  
    - Edge Cases: Empty tools list, oversized arrays.

---

#### 1.3 Attribute Handling and Upsert

- **Overview:**  
  This block processes each tool's attributes: it splits attributes, checks if each attribute exists in Airtable, creates missing attributes, and maps attributes to Airtable record IDs for relational linking.

- **Nodes Involved:**  
  - Loop Over Attributes  
  - All Attributes  
  - Wait for Attribute Creation  
  - Split Out each Attribute String  
  - Check if Attribute exists  
  - Merge  
  - Map Attribute ID  
  - Change each Attribute to the corresponding RecID  

- **Node Details:**  

  - **Loop Over Attributes**  
    - Type: Split In Batches  
    - Role: Iterates over each attribute in a tool's attribute list for processing in manageable batches.  
    - Inputs: Individual tool JSONs from Split Out Tools  
    - Outputs: Batched attribute arrays for downstream nodes  
    - Edge Cases: Large attribute lists, empty attributes.

  - **Split Out each Attribute String**  
    - Type: Split Out  
    - Role: Splits attribute arrays into single attribute strings for individual checking.  
    - Inputs: Batched attributes  
    - Outputs: Single attribute strings  
    - Edge Cases: Empty attribute strings.

  - **Check if Attribute exists**  
    - Type: Airtable node (upsert operation)  
    - Role: Checks if attribute already exists in Airtable "Attributes" table by matching the attribute name, and creates it if missing.  
    - Configuration: Upsert by "Name" column in the Attributes table.  
    - Inputs: Single attribute strings  
    - Outputs: Record data including Airtable record ID and attribute name  
    - Edge Cases: Airtable API errors, rate limiting, missing credentials.

  - **Merge**  
    - Type: Merge  
    - Role: Combines newly created/confirmed attribute records back into single output flow.  
    - Inputs: Outputs from multiple attribute existence checks  
    - Outputs: Combined attributes with IDs  
    - Edge Cases: Merge conflicts or data mismatch.

  - **All Attributes**  
    - Type: Set  
    - Role: Collects all attribute records for a tool, preparing for mapping.  
    - Inputs: Merged attribute records  
    - Outputs: Aggregated attribute data  
    - Edge Cases: None significant.

  - **Wait for Attribute Creation**  
    - Type: Merge (chooseBranch)  
    - Role: Waits for all attribute creation/upsert operations to complete before further processing.  
    - Inputs: Attribute records  
    - Outputs: Synchronized output  
    - Edge Cases: Delays or timeouts in upstream nodes.

  - **Map Attribute ID**  
    - Type: Set  
    - Role: Maps each attribute name to its Airtable record ID for later linking to tools.  
    - Inputs: Outputs from Wait for Attribute Creation  
    - Outputs: Objects containing attribute name and corresponding Airtable ID  
    - Edge Cases: Missing attribute IDs.

  - **Change each Attribute to the corresponding RecID**  
    - Type: Code (JavaScript)  
    - Role: Replaces attribute names in the original tool JSON with their Airtable record IDs.  
    - Inputs: Tool JSONs with attribute name arrays and attribute ID map  
    - Outputs: Tool JSONs with attribute IDs instead of names  
    - Edge Cases: Missing attribute mappings, data inconsistency.

---

#### 1.4 Tool Handling and Upsert

- **Overview:**  
  This block processes each tool by generating a unique MD5 hash ID from the tool name, creating or updating the tool record in Airtable, and merging existing data with new attribute and similarity relations.

- **Nodes Involved:**  
  - Generate Unique Hash for Name  
  - Create if not Exist (Airtable)  
  - Only what we need  
  - Merge Old Data + RecID  
  - Determine Attributes we should save  

- **Node Details:**  

  - **Generate Unique Hash for Name**  
    - Type: Crypto (MD5)  
    - Role: Creates a deterministic unique hash from the lowercased, trimmed tool name to serve as a stable ID.  
    - Inputs: Tool name string  
    - Outputs: Tool hash string in `hash` property  
    - Edge Cases: Hash collisions (extremely unlikely), empty names.

  - **Create if not Exist**  
    - Type: Airtable node (upsert)  
    - Role: Creates or updates tool record in Airtable "Tools" table using the hash as unique key.  
    - Configuration: Matches on `Hash` column; sets `Name` and other fields as available.  
    - Inputs: Tool JSON with hash and name  
    - Outputs: Airtable record data including record ID and fields  
    - Edge Cases: Airtable API limits, missing credentials.

  - **Only what we need**  
    - Type: Set  
    - Role: Filters and organizes the Airtable record data for further processing, extracting existing attributes and similar tool references.  
    - Inputs: Airtable record output  
    - Outputs: JSON with `hash`, `id`, `existingAttributes`, and `existingSimilars` arrays  
    - Edge Cases: Missing fields in Airtable records.

  - **Merge Old Data + RecID**  
    - Type: Merge  
    - Role: Combines freshly created data with existing record IDs to maintain continuity.  
    - Inputs: Outputs from Create if not Exist and Generate Unique Hash for Name  
    - Outputs: Combined tool data  
    - Edge Cases: Merge conflicts.

  - **Determine Attributes we should save**  
    - Type: Code (JavaScript)  
    - Role: Compares new attribute IDs with existing ones in Airtable, determines which attributes need to be added to avoid duplicates.  
    - Inputs: Tool JSON with existing and new attribute IDs  
    - Outputs: Tool JSON with `savingAttributes` array containing unique attributes to save  
    - Edge Cases: Empty attribute lists.

---

#### 1.5 Similarity (Competitor) Mapping

- **Overview:**  
  This block processes similarity relationships between tools, ensures similar tools exist in Airtable, maps their record IDs, and updates the tools accordingly.

- **Nodes Involved:**  
  - Split Out similar  
  - Generate Unique Hash for Similar  
  - It Should exists (Airtable upsert)  
  - Merge1  
  - All Similar  
  - Merge2  
  - Change each Smiliar to the corresponding RecID  
  - Determine Similar we should save  
  - Save all this juicy data  

- **Node Details:**  

  - **Split Out similar**  
    - Type: Split Out  
    - Role: Splits the `similar` array from each tool into single similar tool names for processing.  
    - Inputs: Tool JSON with similarity arrays  
    - Outputs: Individual similar tool strings  
    - Edge Cases: Empty similarity lists.

  - **Generate Unique Hash for Similar**  
    - Type: Crypto (MD5)  
    - Role: Generates unique hash IDs for each similar tool name for Airtable matching.  
    - Inputs: Similar tool name strings  
    - Outputs: Hash string for each similar tool  
    - Edge Cases: Same as for tool name hashing.

  - **It Should exists**  
    - Type: Airtable node (upsert)  
    - Role: Ensures each similar tool exists in Airtable, creating new records if necessary.  
    - Configuration: Matches by hash in Tools table.  
    - Inputs: Similar tool hash and name  
    - Outputs: Airtable record data  
    - Edge Cases: Airtable API errors.

  - **Merge1**  
    - Type: Merge  
    - Role: Combines similar tool records for mapping.  
    - Inputs: Outputs from It Should exists and Generate Unique Hash for Similar  
    - Outputs: Combined similar tool data  
    - Edge Cases: None significant.

  - **All Similar**  
    - Type: Set  
    - Role: Collects all similar tool records with IDs for mapping.  
    - Inputs: Merge1 output  
    - Outputs: Map of similar tool names to Airtable record IDs  
    - Edge Cases: Missing records.

  - **Merge2**  
    - Type: Merge (chooseBranch)  
    - Role: Synchronizes and combines similar tool data for downstream mapping.  
    - Inputs: All Similar and other branches  
    - Outputs: Merged data for mapping IDs  
    - Edge Cases: Timing out or mismatched merges.

  - **Change each Smiliar to the corresponding RecID**  
    - Type: Code (JavaScript)  
    - Role: Replaces similar tool names in the original tool JSON with Airtable record IDs.  
    - Inputs: Tool JSONs and similar tool record mappings  
    - Outputs: Tool JSONs with `similar` arrays replaced by record IDs  
    - Edge Cases: Missing mappings.

  - **Determine Similar we should save**  
    - Type: Code (JavaScript)  
    - Role: Compares existing similar tool IDs with new ones, determines which need to be saved to avoid duplicates.  
    - Inputs: Tool JSON with existing and new similar IDs  
    - Outputs: Tool JSON with `savingSimilars` array  
    - Edge Cases: Empty lists.

  - **Save all this juicy data**  
    - Type: Airtable (upsert)  
    - Role: Updates the Tools Airtable record with the final set of attributes and similarity relations.  
    - Configuration: Upsert by `Hash` column, fields include `Attributes` and `Similar` linked record arrays.  
    - Inputs: Fully mapped tool data  
    - Outputs: Confirmation of save  
    - Edge Cases: Airtable API errors, partial updates.

---

### 3. Summary Table

| Node Name                       | Node Type                     | Functional Role                                  | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                          |
|--------------------------------|-------------------------------|-------------------------------------------------|---------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger                  | Entry point; receives logo sheet image and prompt | —                               | Map Agent Input                 | —                                                                                                  |
| Map Agent Input               | Set                           | Maps form prompt to chatInput for AI agent       | On form submission              | gpt-4o                         | —                                                                                                  |
| gpt-4o                        | LangChain OpenAI Chat         | Runs GPT-4o model to process prompt and image    | Map Agent Input                | Retrieve and Parser Agent       | —                                                                                                  |
| Retrieve and Parser Agent     | LangChain Agent               | Extracts tools, attributes, and similarity JSON | gpt-4o                         | JSON it                       | Eat the provided Images, Extract the Information out of them as "Tool -> Attributes" list.         |
| Structured Output Parser      | LangChain Output Parser       | Validates and structures AI JSON output          | Retrieve and Parser Agent       | Retrieve and Parser Agent       | —                                                                                                  |
| JSON it                      | Set                           | Extracts parsed JSON from AI output               | Retrieve and Parser Agent       | Split Out Tools                | —                                                                                                  |
| Split Out Tools              | Split Out                     | Splits tools array into individual tool objects  | JSON it                       | Loop Over Attributes, Wait for Attribute Creation | —                                                                                                  |
| Loop Over Attributes          | Split In Batches              | Iterates over each tool's attribute list          | Split Out Tools                | All Attributes, Split Out each Attribute String | —                                                                                                  |
| Split Out each Attribute String | Split Out                   | Splits attribute arrays into single strings      | Loop Over Attributes           | Check if Attribute exists, Merge | —                                                                                                  |
| Check if Attribute exists    | Airtable (upsert)             | Checks/creates attributes in Airtable             | Split Out each Attribute String | Merge                         | —                                                                                                  |
| Merge                        | Merge                        | Combines attribute check/create outputs            | Check if Attribute exists, Split Out each Attribute String | Map Attribute ID             | —                                                                                                  |
| All Attributes               | Set                           | Aggregates attribute records for tool              | Loop Over Attributes           | Wait for Attribute Creation   | ## Attribute Creation and Mapping those created or existing Ids                                    |
| Wait for Attribute Creation  | Merge (chooseBranch)          | Waits for all attribute creation/upserts to finish | All Attributes, Split Out Tools | Change each Attribute to the corresponding RecID | ## Attribute Creation and Mapping those created or existing Ids                                    |
| Map Attribute ID             | Set                           | Maps attribute names to Airtable record IDs        | Merge                         | Loop Over Attributes            | ## Attribute Creation and Mapping those created or existing Ids                                    |
| Change each Attribute to the corresponding RecID | Code                  | Replaces attribute names with Airtable record IDs | Wait for Attribute Creation   | Generate Unique Hash for Name   | ## Attribute Creation and Mapping those created or existing Ids                                    |
| Generate Unique Hash for Name | Crypto (MD5)                 | Generates stable hash ID for tool name             | Change each Attribute to the corresponding RecID | Create if not Exist           | ## Create the Tools (if not exists)                                                                |
| Create if not Exist          | Airtable (upsert)             | Creates/updates tool record in Airtable             | Generate Unique Hash for Name  | Only what we need, Merge Old Data + RecID | ## Create the Tools (if not exists)                                                                |
| Only what we need            | Set                           | Filters tool Airtable record fields for processing | Create if not Exist            | Merge Old Data + RecID          | ## Create the Tools (if not exists)                                                                |
| Merge Old Data + RecID       | Merge                        | Merges existing data with new tool data             | Only what we need, Create if not Exist | Determine Attributes we should save | ## Create the Tools (if not exists)                                                                |
| Determine Attributes we should save | Code                  | Determines new attributes to add to avoid duplicates | Merge Old Data + RecID         | Split Out similar               | ## Create the Tools (if not exists)                                                                |
| Split Out similar            | Split Out                     | Splits similarity arrays into individual names      | Determine Attributes we should save | Generate Unique Hash for Similar, Merge2 | ## Map Competitors                                                                                   |
| Generate Unique Hash for Similar | Crypto (MD5)               | Generates stable hash IDs for similar tool names    | Split Out similar             | It Should exists, Merge1        | ## Map Competitors                                                                                   |
| It Should exists            | Airtable (upsert)             | Ensures similar tools exist in Airtable              | Generate Unique Hash for Similar | Merge1                        | ## Map Competitors                                                                                   |
| Merge1                      | Merge                        | Combines similar tool records                         | It Should exists, Generate Unique Hash for Similar | All Similar                   | ## Map Competitors                                                                                   |
| All Similar                 | Set                           | Aggregates similar tool Airtable records             | Merge1                        | Merge2                         | ## Map Competitors                                                                                   |
| Merge2                      | Merge (chooseBranch)          | Synchronizes similar tool data                         | All Similar, Split Out similar  | Change each Smiliar to the corresponding RecID | ## Map Competitors                                                                                   |
| Change each Smiliar to the corresponding RecID | Code                  | Maps similar tool names to Airtable record IDs        | Merge2                        | Determine Similar we should save | ## Map Competitors                                                                                   |
| Determine Similar we should save | Code                   | Determines new similar tool relations to save         | Change each Smiliar to the corresponding RecID | Save all this juicy data      | ## Map Competitors                                                                                   |
| Save all this juicy data    | Airtable (upsert)             | Saves updated tools with attributes and similarity    | Determine Similar we should save | —                             | ## Map Competitors                                                                                   |
| Sticky Note                 | Sticky Note                   | Various instructional and structural notes            | —                             | —                             | See sticky notes in the workflow for detailed instructions, Airtable schema, and usage tips.       |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Airtable Bases and Tables**  
- Create an Airtable base named e.g. "AI Tools".  
- Create two tables:  
  - **Tools** with fields:  
    - Name (single line text, required)  
    - Attributes (linked records to Attributes table, multiple)  
    - Hash (single line text, unique key)  
    - Similar (linked records to Tools table, multiple)  
    - Optional fields: Description, Website, Category  
  - **Attributes** with fields:  
    - Name (single line text, required)  
    - Tools (linked records to Tools table, multiple)  

**Step 2: Configure Credentials in n8n**  
- Add Airtable Personal Access Token credentials with access to the above base.  
- Set up OpenAI API credentials with access to GPT-4o model.  

**Step 3: Create Trigger Node**  
- Add a **Form Trigger** node named "On form submission"  
- Configure it to accept file upload (image) and optional prompt, path: `/logo-sheet-feeder`  
- Enable webhook and test form availability.

**Step 4: Prepare AI Input**  
- Add a **Set** node "Map Agent Input" connected from the form node  
- Map the prompt input field to `chatInput` string field  

**Step 5: AI Model Invocation**  
- Add a **LangChain OpenAI Chat** node "gpt-4o" connected from "Map Agent Input"  
- Select model "gpt-4o" and link OpenAI credentials  

**Step 6: AI Agent for Vision Extraction**  
- Add a **LangChain Agent** node "Retrieve and Parser Agent"  
- Configure system prompt instructing extraction of JSON with `name`, `attributes`, and `similar` fields per tool  
- Enable passthrough for binary images  
- Connect from "gpt-4o" AI language model output  

**Step 7: Parse AI Output**  
- Add **LangChain Output Parser** node configured with JSON schema example matching expected structure  
- Connect as output parser for "Retrieve and Parser Agent"  
- Add a **Set** node "JSON it" to extract `output` JSON field from agent output  

**Step 8: Split Out Tools**  
- Add a **Split Out** node "Split Out Tools" splitting on `tools` array from JSON it node  

**Step 9: Process Attributes**  
- Add a **Split In Batches** node "Loop Over Attributes" connected from "Split Out Tools" output  
- Add a **Split Out** node "Split Out each Attribute String" connected from "Loop Over Attributes" (second output)  
- Add an **Airtable** node "Check if Attribute exists" configured to upsert by attribute `Name` in Attributes table  
- Add a **Merge** node combining outputs from "Check if Attribute exists" and "Split Out each Attribute String"  
- Add a **Set** node "All Attributes" to aggregate attributes  
- Add a **Merge** node "Wait for Attribute Creation" (chooseBranch mode) to wait for all attribute upserts  
- Add a **Set** node "Map Attribute ID" to map attribute names to record IDs  
- Add a **Code** node "Change each Attribute to the corresponding RecID" replacing attribute names with Airtable IDs  

**Step 10: Process Tools**  
- Add a **Crypto** node "Generate Unique Hash for Name" creating MD5 hash of tool names  
- Add an **Airtable** node "Create if not Exist" upserting by `Hash` in Tools table with `Name` field  
- Add a **Set** node "Only what we need" to filter Airtable fields and extract existing attributes and similars  
- Add a **Merge** node "Merge Old Data + RecID" combining new and existing data  
- Add a **Code** node "Determine Attributes we should save" to detect new attributes that need to be saved  

**Step 11: Process Similar Tools**  
- Add a **Split Out** node "Split Out similar" splitting tool similarity arrays  
- Add a **Crypto** node "Generate Unique Hash for Similar" hashing similar tool names  
- Add an **Airtable** node "It Should exists" upserting similar tools by hash in Tools table  
- Add a **Merge** node "Merge1" combining similar tool data  
- Add a **Set** node "All Similar" to aggregate similar tool records  
- Add a **Merge** node "Merge2" (chooseBranch mode) to synchronize for mapping  
- Add a **Code** node "Change each Smiliar to the corresponding RecID" to map similar names to Airtable IDs  
- Add a **Code** node "Determine Similar we should save" to find new similarity relations  
- Add an **Airtable** node "Save all this juicy data" to upsert final tool records with updated attributes and similars  

**Step 12: Connect Nodes Following the Workflow**  
- Chain nodes according to logical data flow described, respecting input/output connections as per the original workflow.  
- Ensure merge nodes combine appropriate branches and split nodes handle array expansions properly.  

**Step 13: Add Sticky Notes for Documentation**  
- Add sticky notes to explain each major block and important instructions as per original workflow for maintainability.  

**Step 14: Activate Workflow**  
- Test the form submission with a valid logo sheet image and optional prompt.  
- Monitor workflow execution and Airtable records for correctness.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                               | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow is designed for an "upload and forget it" use case. For complete extraction, consider multi-agent setups with validation steps.                                                                                                                                 | Workflow description                                |
| Uses MD5 hashes as stable unique IDs for tools and similar tools to avoid duplicates in Airtable.                                                                                                                                                                            | Workflow description                                |
| Airtable schema requires two tables: "Tools" and "Attributes" with linked record fields to maintain relations bidirectionally.                                                                                                                                              | Airtable Structure Sticky Notes                      |
| Default form URL: https://your-n8n.io/form/logo-sheet-feeder — customize as needed.                                                                                                                                                                                         | Workflow description                                |
| Example logo sheet image provided in workflow notes for prompt tuning: https://cloud.let-the-work-flow.com/workflow-data/example-ai-logo-sheet.jpg                                                                                                                          | Sticky Note with example image                       |
| Workflow creator and support: let the work flow — Workflow Automation & Development — https://let-the-work-flow.com                                                                                                                                                        | Workflow description and sticky notes                |

---

This document fully describes the workflow's structure, node configurations, logic blocks, and reproduction steps, enabling advanced users and automation agents to understand, replicate, or modify it efficiently.