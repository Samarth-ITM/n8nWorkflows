Real-time X Post Monitoring & Auto-Categorization with Airtop

https://n8nworkflows.xyz/workflows/real-time-x-post-monitoring---auto-categorization-with-airtop-4206


# Real-time X Post Monitoring & Auto-Categorization with Airtop

### 1. Workflow Overview

This workflow, titled **"Real-time X Post Monitoring & Auto-Categorization with Airtop"**, is designed to monitor real-time search results on X (formerly Twitter) and automatically extract and categorize relevant posts. It aims to support community engagement, lead discovery, thought leadership tracking, or competitive analysis by filtering high-signal posts according to user-defined categories.

The workflow logically breaks down into these key blocks:

- **1.1 Trigger & Input Initialization:** Receives parameters (Airtop profile, X search URL, relevant categories) from an external workflow.
- **1.2 Airtop Browser Session Management:** Starts a browser session using Airtop with the specified profile, navigates to the search URL, and manages the session lifecycle.
- **1.3 Data Extraction & Classification:** Extracts up to 10 valid English-language posts from the search results, classifies them into categories or marks them as `[NA]` if unrelated.
- **1.4 Post-Processing & Filtering:** Parses the extraction results, filters out posts categorized as `[NA]`.
- **1.5 Output Generation:** Produces a structured JSON array of relevant posts for downstream workflows.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Initialization

- **Overview:**  
  This block triggers the workflow when executed by another workflow and sets up input variables needed downstream.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Inputs

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point, triggers this workflow from an external workflow and receives input parameters.  
    - Configuration: Expects inputs named `airtop_profile`, `x_url`, and `relevant_categories`.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to `Inputs` node.  
    - Edge Cases: Missing or malformed inputs could cause downstream failures. No retry or error handling configured here.

  - **Inputs**  
    - Type: Set  
    - Role: Maps incoming parameters to variables used in Airtop nodes.  
    - Configuration: Assigns three fields:  
       - `url` ← `x_url` from trigger input  
       - `profile` ← `airtop_profile` from trigger input  
       - `categories` ← `relevant_categories` from trigger input  
    - Inputs: From trigger node  
    - Outputs: Connects to `Session` node  
    - Edge Cases: If input data types are incorrect or missing, subsequent Airtop API calls may fail.

---

#### 2.2 Airtop Browser Session Management

- **Overview:**  
  This block manages the Airtop browser session lifecycle: starting a session with the specified profile, opening the target URL, extracting data, and terminating the session.

- **Nodes Involved:**  
  - Session  
  - Window  
  - Extract posts  
  - End session

- **Node Details:**

  - **Session**  
    - Type: Airtop node  
    - Role: Starts a browser session using an Airtop profile proxy.  
    - Configuration:  
      - Proxy mode: Integrated  
      - Profile name: Taken dynamically from `Inputs.profile`  
      - Credential: Airtop API key (official org)  
    - Inputs: From `Inputs` node  
    - Outputs: Connects to `Window` node  
    - Edge Cases: Authentication failure, invalid profile name, or Airtop API downtime.

  - **Window**  
    - Type: Airtop node  
    - Role: Opens a browser window/tab at the specified X search URL.  
    - Configuration:  
      - URL: Dynamic from `Inputs.url`  
      - Resource: Window (to open new browser window)  
      - Credential: Airtop API key  
    - Inputs: From `Session` node  
    - Outputs: Connects to `Extract posts` node  
    - Edge Cases: Invalid URL, navigation timeout, network issues.

  - **Extract posts**  
    - Type: Airtop node (extraction resource)  
    - Role: Extracts structured posts data and classifies posts based on categories from the page content.  
    - Configuration:  
      - Prompt: A detailed instruction to extract up to 10 non-sponsored English posts containing `/status/` in URL, including writer, time, text, URL, and category classification.  
      - Categories: Passed dynamically from `Inputs.categories`  
      - Output schema: JSON Schema defining expected post object structure  
      - Pagination: Infinite scroll mode enabled (to load more posts if needed)  
      - Interaction mode: Auto (automated scraping)  
      - Credential: Airtop API key  
    - Inputs: From `Window` node  
    - Outputs: Connects to `End session` node  
    - Edge Cases:  
      - Extraction failure if page structure changes or content is missing.  
      - Classification errors if categories are ill-defined.  
      - API rate limits or timeout.

  - **End session**  
    - Type: Airtop node  
    - Role: Terminates the browser session to free resources.  
    - Configuration: Operation set to "terminate"  
    - Credential: Airtop API key  
    - Inputs: From `Extract posts` node  
    - Outputs: Connects to `Parse JSON output` node  
    - Edge Cases: Session termination failure or delay.

---

#### 2.3 Post-Processing & Filtering

- **Overview:**  
  Parses the raw extraction JSON output and filters out any posts categorized as `[NA]`.

- **Nodes Involved:**  
  - Parse JSON output  
  - Filter out [NA] posts

- **Node Details:**

  - **Parse JSON output**  
    - Type: Code (JavaScript)  
    - Role: Parses the stringified JSON response from Airtop and maps each post to a separate item.  
    - Configuration:  
      - Code snippet parses `Extract posts` last node’s `modelResponse` JSON, extracts `posts` array, and returns an array of JSON objects with post details. Returns empty array if no posts.  
    - Inputs: From `End session` node  
    - Outputs: Connects to `Filter out [NA] posts` node  
    - Edge Cases:  
      - JSON parse errors if response is malformed.  
      - Empty post list handling.

  - **Filter out [NA] posts**  
    - Type: Filter  
    - Role: Removes any posts from the output where the `category` field is exactly `[NA]`.  
    - Configuration:  
      - Condition: `category` does not contain `NA` (case sensitive, strict string validation)  
    - Inputs: From `Parse JSON output` node  
    - Outputs: Workflow output (implicit)  
    - Edge Cases: All posts filtered out resulting in empty output.

---

#### 2.4 Documentation & Metadata

- **Overview:**  
  Contains a large sticky note with detailed documentation describing the workflow’s purpose, use cases, input/output definitions, and setup instructions.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides comprehensive README-style information embedded within the workflow canvas.  
    - Content:  
      - Explains the use case of real-time monitoring of X search results.  
      - Lists input parameters and expected outputs.  
      - Details the step-by-step workflow logic.  
      - Specifies setup requirements such as Airtop profile and API key.  
      - Suggests possible next steps and integrations (Slack alerts, engagement workflows).  
    - Inputs/Outputs: None  
    - Edge Cases: None (informational only)

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                              | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                                                                                                                     |
|-----------------------------|---------------------------|----------------------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger | Entry trigger receiving external inputs       | None                         | Inputs                      |                                                                                                                                                                                                                                |
| Inputs                      | Set                       | Maps trigger inputs to usable variables       | When Executed by Another Workflow | Session                   |                                                                                                                                                                                                                                |
| Session                    | Airtop                    | Starts Airtop browser session with profile    | Inputs                       | Window                      |                                                                                                                                                                                                                                |
| Window                     | Airtop                    | Opens browser window at given X search URL    | Session                      | Extract posts               |                                                                                                                                                                                                                                |
| Extract posts              | Airtop                    | Extracts and classifies posts from search page | Window                       | End session                 |                                                                                                                                                                                                                                |
| End session                | Airtop                    | Terminates Airtop browser session              | Extract posts                | Parse JSON output           |                                                                                                                                                                                                                                |
| Parse JSON output          | Code (JavaScript)         | Parses extraction JSON into discrete posts    | End session                  | Filter out [NA] posts       |                                                                                                                                                                                                                                |
| Filter out [NA] posts      | Filter                    | Filters out posts categorized as `[NA]`       | Parse JSON output            | (workflow output)           |                                                                                                                                                                                                                                |
| Sticky Note                | Sticky Note               | Embedded workflow documentation and README    | None                        | None                        | README - Monitor X for Relevant Community Posts - Explains use case, inputs, outputs, logic, setup, and next steps.                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.
   - Configure it to expect inputs:  
     - `airtop_profile` (string)  
     - `x_url` (string)  
     - `relevant_categories` (string)

2. **Create Inputs Node:**
   - Add a **Set** node named `Inputs`.
   - Connect `When Executed by Another Workflow` → `Inputs`.
   - Assign variables:  
     - `url` = Expression: `{{$json["x_url"]}`  
     - `profile` = Expression: `{{$json["airtop_profile"]}`  
     - `categories` = Expression: `{{$json["relevant_categories"]}`

3. **Create Session Node:**
   - Add an **Airtop** node named `Session`.
   - Connect `Inputs` → `Session`.
   - Set parameters:  
     - Proxy: `integrated`  
     - Profile Name: Expression: `{{$json["profile"]}`  
   - Assign Airtop API credentials.

4. **Create Window Node:**
   - Add an **Airtop** node named `Window`.
   - Connect `Session` → `Window`.
   - Set parameters:  
     - URL: Expression: `{{$json["url"]}`  
     - Resource: `window`  
   - Use the same Airtop API credentials.

5. **Create Extract posts Node:**
   - Add an **Airtop** node named `Extract posts`.
   - Connect `Window` → `Extract posts`.
   - Set parameters:  
     - Resource: `extraction`  
     - Prompt: Use the detailed prompt to extract up to 10 posts, including the dynamic categories:  
       ```
       This is X search results. Extract up to 10 non-sponsored posts in English. For each post extract post writer, time, post text and post URL. Return only posts that appear in the search results. A valid post URL includes /status/
       
       Also classify posts based on the following categories:
       {{ $('Inputs').item.json.categories }}
       
       IMPORTANT anything that is not directly related to categories above should return [NA]
       ```
     - Output Schema: JSON schema specifying posts array with fields `writer`, `time`, `text`, `url`, `category`.  
     - Pagination Mode: `infinite-scroll`  
     - Interaction Mode: `auto`  
   - Use Airtop API credentials.

6. **Create End session Node:**
   - Add an **Airtop** node named `End session`.
   - Connect `Extract posts` → `End session`.
   - Set parameters:  
     - Operation: `terminate`  
   - Use Airtop API credentials.

7. **Create Parse JSON output Node:**
   - Add a **Code** node named `Parse JSON output`.
   - Connect `End session` → `Parse JSON output`.
   - Code:
     ```javascript
     const posts = JSON.parse($('Extract posts').last().json.data.modelResponse).posts;

     if (!posts.length) {
       return [{ json: {} }];
     }

     return posts.map(post => ({ json: post }));
     ```
   - Ensure node runs with type version supporting current JS syntax.

8. **Create Filter out [NA] posts Node:**
   - Add a **Filter** node named `Filter out [NA] posts`.
   - Connect `Parse JSON output` → `Filter out [NA] posts`.
   - Conditions:  
     - Field: `category`  
     - Operator: Does Not Contain  
     - Value: `NA`  
     - Case Sensitive: true  
     - Validation: strict string

9. **(Optional) Add Sticky Note:**
   - Add a **Sticky Note** node with the README content describing workflow purpose, inputs, outputs, and setup.

10. **Set Workflow Settings:**
    - Configure execution order as necessary.
    - Ensure Airtop API credentials are set and valid.
    - Mark the workflow as inactive or active as per usage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| The workflow requires an Airtop browser profile authenticated on X to function properly. Ensure the profile is logged in and active before running the workflow.                                                                                                                                                                                                                                                                                                                                                                  | Setup Requirements section in Sticky Note                    |
| Input `relevant_categories` should be a text-based list of categories guiding post classification. The more precise and well-defined the categories, the better the classification results.                                                                                                                                                                                                                                                                                                                                         | Workflow logic and prompt in Extract posts node              |
| This workflow is designed to be triggered from another workflow, not run standalone. It accepts parameters dynamically to allow flexible integration in larger automation pipelines.                                                                                                                                                                                                                                                                                                                                              | When Executed by Another Workflow node description           |
| To extend functionality, consider feeding the output into engagement workflows (reply, retweet) or Slack alert workflows for team visibility. Sentiment analysis or company-mention detection can be added to improve categorization.                                                                                                                                                                                                                                                                                                  | Sticky Note next steps section                                |
| Airtop API key credential must be configured in n8n with the official organization’s key. Without correct credentials, all Airtop nodes will fail authentication.                                                                                                                                                                                                                                                                                                                                                                    | Credential requirement for Airtop nodes                       |
| X’s page structure or API changes may break the extraction prompt; monitor and update the prompt or output schema accordingly.                                                                                                                                                                                                                                                                                                                                                                                                       | Edge case considerations in Extract posts node               |
| Infinite scroll mode in extraction enables scraping beyond initially loaded posts but may increase runtime; adjust as needed.                                                                                                                                                                                                                                                                                                                                                                                                         | Extract posts node configuration                              |
| Sticky Note contains a comprehensive README with detailed use case, input/output descriptions, and setup instructions. It is highly recommended to keep it updated with workflow changes.                                                                                                                                                                                                                                                                                                                                             | Sticky Note content                                          |

---

**Disclaimer:**  
The provided text and workflow are created solely through an automated n8n workflow. The process fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.