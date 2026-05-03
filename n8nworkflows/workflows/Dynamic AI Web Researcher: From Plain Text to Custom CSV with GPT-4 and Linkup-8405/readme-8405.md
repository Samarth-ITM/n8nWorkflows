Dynamic AI Web Researcher: From Plain Text to Custom CSV with GPT-4 and Linkup

https://n8nworkflows.xyz/workflows/dynamic-ai-web-researcher--from-plain-text-to-custom-csv-with-gpt-4-and-linkup-8405


# Dynamic AI Web Researcher: From Plain Text to Custom CSV with GPT-4 and Linkup

---

### 1. Workflow Overview

This workflow, titled **Dynamic AI Web Researcher: From Plain Text to Custom CSV with GPT-4 and Linkup**, automates the process of transforming a plain-text research request into a richly structured CSV file. It is designed for use cases involving detailed web research where users want to specify a research topic in natural language and receive a customized spreadsheet containing relevant entities and their enriched properties.

The workflow logically divides into four core blocks:

- **1.1 Input Reception**: Captures the user's research request via a form trigger.
- **1.2 AI Planning & Schema Generation**: Uses a powerful AI model (GPT-based) to analyze the request, identify the main research object, and dynamically generate JSON schemas and prompts for discovery and enrichment phases.
- **1.3 Data Discovery & Enrichment via Linkup API**: Executes structured web searches through the Linkup API to find matching items and then enriches each item with detailed properties in a loop.
- **1.4 Data Structuring and Output**: Cleans, formats, and converts the enriched data into a CSV file for export.

Supporting this structure are several sticky notes providing conceptual guidance and instructions.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This initial block captures the user’s free-text research query through a form submission, triggering the workflow.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; listens for user input via a form titled “New research” with a required textarea field “Describe your research.”  
  - *Configuration:* Uses webhook with a unique ID; form field is a single textarea requiring user input.  
  - *Input:* External HTTP form submission  
  - *Output:* JSON containing user’s research description under key `Describe your research`  
  - *Potential Failures:* Missing required field; webhook connection issues  
  - *Notes:* None

---

#### 2.2 AI Planning & Schema Generation

**Overview:**  
This block leverages a GPT-based AI to parse the user’s research description, identify the primary object of research, and dynamically create JSON schemas and prompts for subsequent discovery and enrichment steps.

**Nodes Involved:**  
- OpenAI Chat Model  
- Prepare prompts and schema  
- Sticky Note3 (Architect Brain)  
- Sticky Note4 (Loop explanation)  

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model node  
  - *Role:* Runs GPT-5 chat-latest model to process the user input into structured JSON for research planning  
  - *Configuration:* Model set to “gpt-5-chat-latest” with JSON object response format  
  - *Credentials:* OpenAI API key configured  
  - *Input:* JSON from “On form submission” node containing user prompt  
  - *Output:* Structured JSON with keys: ObjectName, discoveryQuery, discoverySchema, enrichmentQuery, enrichmentSchema  
  - *Potential Failures:* API authentication errors, rate limits, model unavailability, JSON parse errors  
  - *Notes:* Powerful AI required for accurate prompt interpretation

- **Prepare prompts and schema**  
  - *Type:* Langchain Chain LLM node  
  - *Role:* Uses AI to generate detailed discovery and enrichment prompts and corresponding JSON schemas based on the user’s request  
  - *Configuration:* The input is the user’s research description text; prompt instructs AI to return a fixed JSON object with five keys (ObjectName, discoveryQuery, discoverySchema, enrichmentQuery, enrichmentSchema). The schema for discovery and enrichment is strictly defined in the prompt.  
  - *Input:* Output from “OpenAI Chat Model” node  
  - *Output:* JSON object with research plan and schemas  
  - *Potential Failures:* AI failures in generating valid JSON, incorrect schema format, model timeouts  
  - *Notes:* This node embodies the "architect brain" of the workflow (Sticky Note3)

- **Sticky Note3 (Architect Brain)**  
  - Explains that this AI step defines the output schema and how to get there, emphasizing the need for a powerful AI model.

- **Sticky Note4 (Running a loop on each item)**  
  - Provides context for the upcoming looping over discovered items for enrichment.

---

#### 2.3 Data Discovery & Enrichment via Linkup API

**Overview:**  
This block performs actual web data gathering by querying the Linkup API twice: first to discover a list of items matching the criteria, then to enrich each item with detailed properties. It includes a loop that processes each item individually.

**Nodes Involved:**  
- Query Linkup to find the list  
- Split Out  
- Loop Over Items  
- Get object name and value  
- Query Linkup to find all properties for this item  
- Prepare final JSON for that item  
- Sticky Note (AI web-search to find all items)  
- Sticky Note1 (AI web-search to find all info about one item)  
- Sticky Note4 (Loop explanation)

**Node Details:**

- **Query Linkup to find the list**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to Linkup API to search for the initial list of items based on AI-generated discoveryQuery and discoverySchema  
  - *Configuration:*  
    - URL: `https://api.linkup.so/v1/search`  
    - Method: POST  
    - Authentication: Bearer token (Linkup API key)  
    - Body Parameters:  
      - `q`: discoveryQuery (from AI output)  
      - `depth`: "deep"  
      - `outputType`: "structured"  
      - `structuredOutputSchema`: JSON stringified discoverySchema  
      - `includeImages`: false  
  - *Input:* JSON from “Prepare prompts and schema” node  
  - *Output:* JSON containing a List array of discovered items under “List” key  
  - *Potential Failures:* HTTP errors, authentication errors, malformed query, API rate limits  
  - *Notes:* Requires Linkup API credentials (Sticky Note)

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the List array of discovered items into individual item outputs for batch processing  
  - *Configuration:* Field to split out is “List”  
  - *Input:* Output from “Query Linkup to find the list”  
  - *Output:* Each item individually passed downstream  
  - *Potential Failures:* Empty list could cause no downstream processing

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each individual item for enrichment processing and accumulates results  
  - *Configuration:* Default batch size (process items sequentially or in batches as configured)  
  - *Input:* Individual items from “Split Out”  
  - *Output:* Enriched items passed downstream; also connects to “Get object name and value” for enrichment  
  - *Potential Failures:* Large batch sizes may cause timeouts or memory issues

- **Get object name and value**  
  - *Type:* Set  
  - *Role:* Prepares key-value pairs by assigning the ObjectName (from AI planning) as key and the current property value as the value; essentially formats the item for enrichment queries  
  - *Configuration:* Assigns a dynamic key with the value from current item PropertyValue  
  - *Input:* From “Loop Over Items”  
  - *Output:* Structured JSON with object name as key and property value as value  
  - *Potential Failures:* Expression errors if ObjectName missing; empty PropertyValue

- **Query Linkup to find all properties for this item**  
  - *Type:* HTTP Request  
  - *Role:* Enriches current item by querying Linkup API with an enrichmentQuery and enrichmentSchema generated by AI, specific to each item  
  - *Configuration:*  
    - URL: `https://api.linkup.so/v1/search`  
    - Method: POST  
    - Authentication: Bearer token (Linkup API key)  
    - Body Parameters:  
      - `q`: Enrichment query string with current item property value and enrichmentQuery from AI output  
      - `depth`: "standard"  
      - `outputType`: "structured"  
      - `structuredOutputSchema`: JSON stringified enrichmentSchema  
      - `includeImages`: false  
  - *Input:* Output of “Get object name and value” node  
  - *Output:* JSON with detailed enriched properties for the item  
  - *Potential Failures:* API errors, malformed queries, authentication, rate limits  
  - *Notes:* Requires Linkup credentials (Sticky Note1)

- **Prepare final JSON for that item**  
  - *Type:* Code (JavaScript)  
  - *Role:* Merges the key-value pair of the ObjectName with the enriched properties; stringifies arrays and objects for CSV compatibility  
  - *Configuration:* Custom JavaScript code that:  
    - Extracts the ObjectName key and value  
    - Appends all enriched properties, converting arrays to comma-separated strings and objects to JSON strings  
  - *Input:* Output from “Query Linkup to find all properties for this item” and “Get object name and value” (via Split In Batches)  
  - *Output:* Cleaned, flat JSON suitable for CSV conversion  
  - *Potential Failures:* JS runtime errors, unexpected data types  
  - *Notes:* Returns one item per iteration to Loop Over Items node for aggregation

- **Sticky Notes**  
  - Remind users to connect Linkup credentials for the two HTTP Request nodes  
  - Explain the AI web search role in both discovery and enrichment phases  
  - Contextualize the looping mechanism

---

#### 2.4 Data Structuring and Output

**Overview:**  
Final block converts the aggregated enriched data into a CSV file for user consumption.

**Nodes Involved:**  
- Convert to CSV  
- Sticky Note5 (Output CSV explanation)

**Node Details:**

- **Convert to CSV**  
  - *Type:* Convert To File  
  - *Role:* Converts the enriched JSON data collected from the batch processing into a CSV file  
  - *Configuration:* Default CSV conversion options, no special parameters  
  - *Input:* Aggregated enriched JSON items from “Loop Over Items”  
  - *Output:* CSV file output for download or further processing  
  - *Potential Failures:* Large datasets might cause memory issues; malformed data could affect CSV integrity

- **Sticky Note5 (The output: A custom CSV)**  
  - Explains that the CSV has one row per enriched item and columns correspond to enriched properties.

---

### 3. Summary Table

| Node Name                        | Node Type                       | Functional Role                              | Input Node(s)                      | Output Node(s)                       | Sticky Note                                                                                                                                                                            |
|---------------------------------|--------------------------------|----------------------------------------------|----------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger                   | Captures user research request               | -                                | Prepare prompts and schema          |                                                                                                                                                                                        |
| OpenAI Chat Model               | Langchain OpenAI Chat Model    | Processes input to generate AI research plan | Prepare prompts and schema        | Prepare prompts and schema          |                                                                                                                                                                                        |
| Prepare prompts and schema      | Langchain Chain LLM            | Generates discovery/enrichment prompts & schemas | On form submission, OpenAI Chat Model | Query Linkup to find the list       | ## The architect brain: This AI step defines the output schema and how to get there. Make sure to connect a powerful AI model.                                                         |
| Query Linkup to find the list   | HTTP Request                   | Finds list of items matching discovery query | Prepare prompts and schema        | Split Out                          | ## AI web-search to find all items that match the conditions. Don't forget to connect your Linkup credentials.                                                                         |
| Split Out                      | Split Out                     | Splits list of items into individual items   | Query Linkup to find the list     | Loop Over Items                   | ## Running a loop on each item: Within this loop, each item will go through AI web search to enrich properties.                                                                         |
| Loop Over Items                | Split In Batches              | Iterates over each item for enrichment        | Split Out                        | Convert to CSV, Get object name and value | ## Running a loop on each item: Within this loop, each item will go through AI web search to enrich properties.                                                                         |
| Get object name and value       | Set                            | Prepares key-value pairs for enrichment query | Loop Over Items                  | Query Linkup to find all properties for this item | ## AI web-search to find all information about one item. Don't forget to connect your Linkup credentials.                                                                             |
| Query Linkup to find all properties for this item | HTTP Request                   | Enriches each item with detailed properties   | Get object name and value          | Prepare final JSON for that item   | ## AI web-search to find all information about one item. Don't forget to connect your Linkup credentials.                                                                             |
| Prepare final JSON for that item | Code                           | Cleans and merges enrichment data for CSV     | Query Linkup to find all properties for this item | Loop Over Items                    |                                                                                                                                                                                        |
| Convert to CSV                 | Convert To File                | Converts enriched JSON data to CSV file       | Loop Over Items                  | -                                | ## The output: A custom CSV. This CSV contains one row per enriched item, each column is an enriched property.                                                                          |
| Sticky Note                   | Sticky Note                   | Provides guidance and instructions            | -                                | -                                | ## AI web-search to find all items that match the conditions. Don't forget to connect your Linkup credentials.                                                                         |
| Sticky Note1                  | Sticky Note                   | Provides guidance for enrichment API usage    | -                                | -                                | ## AI web-search to find all information about one item. Don't forget to connect your Linkup credentials.                                                                             |
| Sticky Note2                  | Sticky Note                   | Workflow overview and instructions            | -                                | -                                | # **Dynamic AI Web Researcher** This workflow turns free-text research requests into custom CSVs with AI planning and web research. Connect OpenAI and Linkup credentials to run.     |
| Sticky Note3                  | Sticky Note                   | AI planning explanation                        | -                                | -                                | ## The architect brain: This AI step defines how the output of the research will look like and how to get there.                                                                        |
| Sticky Note4                  | Sticky Note                   | Explains loop over items for enrichment        | -                                | -                                | ## Running a loop on each item: Each item goes through AI web search to enrich properties, cleaned and prepared for CSV output.                                                       |
| Sticky Note5                  | Sticky Note                   | Explains final CSV output                       | -                                | -                                | ## The output: A custom CSV. This CSV contains one row per item found, columns are enriched properties from AI web search.                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Name: `On form submission`  
   - Type: Form Trigger  
   - Configure a webhook with an accessible URL.  
   - Create a form titled “New research” with a single required textarea field labeled “Describe your research.”  

2. **Create OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: Langchain OpenAI Chat Model  
   - Set model to `gpt-5-chat-latest`.  
   - Configure to output JSON objects.  
   - Set credentials with your OpenAI API key.  
   - Connect input from `On form submission`.  

3. **Create Chain LLM Node for Prompt & Schema Preparation**  
   - Name: `Prepare prompts and schema`  
   - Type: Langchain Chain LLM  
   - Input text: Expression referencing `Describe your research` field from form submission.  
   - Insert detailed prompt instructing AI to return JSON with keys: ObjectName, discoveryQuery, discoverySchema, enrichmentQuery, enrichmentSchema, following the structure described in the workflow overview.  
   - Connect input from `OpenAI Chat Model`.  

4. **Create HTTP Request Node for Discovery**  
   - Name: `Query Linkup to find the list`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.linkup.so/v1/search`  
   - Authentication: HTTP Bearer Token with Linkup API key.  
   - Body parameters:  
     - `q`: discoveryQuery from AI output  
     - `depth`: "deep"  
     - `outputType`: "structured"  
     - `structuredOutputSchema`: JSON.stringify of discoverySchema from AI output  
     - `includeImages`: false  
   - Connect input from `Prepare prompts and schema`.  

5. **Create Split Out Node to Split Discovered List**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Field to split out: `List` (from Linkup discovery response)  
   - Connect input from `Query Linkup to find the list`.  

6. **Create Split In Batches Node for Looping Over Items**  
   - Name: `Loop Over Items`  
   - Type: Split In Batches  
   - Default batch size (adjust for performance)  
   - Connect input from `Split Out`.  

7. **Create Set Node to Format Item Key-Value**  
   - Name: `Get object name and value`  
   - Type: Set  
   - Assignments: Create a dynamic field where the key is the ObjectName from `Prepare prompts and schema` and the value is the current item’s `PropertyValue`  
   - Connect input from `Loop Over Items`.  

8. **Create HTTP Request Node for Enrichment**  
   - Name: `Query Linkup to find all properties for this item`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.linkup.so/v1/search`  
   - Authentication: HTTP Bearer Token with Linkup API key  
   - Body parameters:  
     - `q`: Concatenate current item property value and enrichmentQuery from AI output  
     - `depth`: "standard"  
     - `outputType`: "structured"  
     - `structuredOutputSchema`: JSON.stringify of enrichmentSchema from AI output  
     - `includeImages`: false  
   - Connect input from `Get object name and value`.  

9. **Create Code Node to Prepare Final JSON**  
   - Name: `Prepare final JSON for that item`  
   - Type: Code (JavaScript)  
   - Code: Merge ObjectName key-value with enriched properties; stringify arrays and objects for CSV compatibility.  
   - Connect input from `Query Linkup to find all properties for this item`.  
   - Output connects back to `Loop Over Items` (to aggregate results).  

10. **Create Convert To File Node to Generate CSV**  
    - Name: `Convert to CSV`  
    - Type: Convert To File  
    - Default CSV settings  
    - Connect input from `Loop Over Items` (aggregated enriched data).  

11. **Add Sticky Notes**  
    - Add explanatory sticky notes as per the workflow to guide users on API credentials, AI model usage, and process logic.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                 | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| # **Dynamic AI Web Researcher** This workflow turns plain-text research requests into custom CSVs with AI planning and web research. Connect OpenAI and Linkup credentials to run.            | Workflow overview (Sticky Note2)                                                                    |
| ## The architect brain: This AI step defines how the output of the research will look like (CSV schema), and how to get there (prompts). Make sure to connect a powerful AI model.             | AI planning block explanation (Sticky Note3)                                                       |
| ## AI web-search to find all items that match the conditions. Don't forget to connect your Linkup credentials.                                                                                 | Applies to discovery HTTP Request node (Sticky Note)                                               |
| ## AI web-search to find all information about one item. Don't forget to connect your Linkup credentials.                                                                                      | Applies to enrichment HTTP Request node (Sticky Note1)                                            |
| ## Running a loop on each item: Each item goes through AI web search to enrich properties, cleaned and prepared for CSV output.                                                                | Loop block explanation (Sticky Note4)                                                              |
| ## The output: A custom CSV. This CSV contains one row per item found, each column is an enriched property from AI web search.                                                                  | Output block explanation (Sticky Note5)                                                            |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.

---