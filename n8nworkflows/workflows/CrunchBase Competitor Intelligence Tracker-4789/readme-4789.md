CrunchBase Competitor Intelligence Tracker

https://n8nworkflows.xyz/workflows/crunchbase-competitor-intelligence-tracker-4789


# CrunchBase Competitor Intelligence Tracker

### 1. Workflow Overview

This workflow, titled **CrunchBase Competitor Intelligence Tracker**, automates the process of monitoring competitor companies by fetching their latest data from Crunchbase and creating actionable review tasks in ClickUp. It is designed for teams that need to stay updated on competitors’ funding rounds, descriptions, and other key business information, enabling timely internal review and response.

The workflow is logically divided into two main blocks:

- **1.1 Input & Preparation:** Accepts manual input of the competitor’s name, processes it into a URL-friendly slug for API querying.
- **1.2 Data Retrieval & Task Creation:** Queries Crunchbase API with the slug to retrieve company data, then automatically creates a detailed task in ClickUp for team review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Preparation

**Overview:**  
This block handles manual initiation and input preparation. It collects the competitor’s name and converts it into a slug format compatible with Crunchbase’s API requirements.

**Nodes Involved:**  
- Manual Trigger  
- Set Competitor Name  
- Generate Crunchbase Slug

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger  
  - *Role:* Allows manual execution of the workflow for testing or on-demand runs.  
  - *Configuration:* No parameters; triggers workflow when clicked manually in n8n.  
  - *Connections:* Outputs to Set Competitor Name.  
  - *Failures:* None expected as it’s manual; workflow not triggered if not clicked.  

- **Set Competitor Name**  
  - *Type:* Set  
  - *Role:* Defines the competitor company name to track.  
  - *Configuration:* Static assignment of a string variable `Competitor` (default example: `"OpenAI"`).  
  - *Expressions:* None; value is hardcoded but could be replaced by dynamic input in future.  
  - *Connections:* Input from Manual Trigger; outputs to Generate Crunchbase Slug.  
  - *Failures:* None under normal conditions; if value is empty, subsequent nodes may fail.  

- **Generate Crunchbase Slug**  
  - *Type:* Code (JavaScript)  
  - *Role:* Converts the competitor name into a Crunchbase API slug.  
  - *Configuration:*  
    - Converts text to lowercase.  
    - Removes all non-alphanumeric characters except spaces and hyphens.  
    - Replaces spaces with hyphens.  
  - *Key Expression:* Custom JS code transforming `item.json.Competitor` into `slug`.  
  - *Connections:* Input from Set Competitor Name; outputs to Fetch Crunchbase Data.  
  - *Failures:* If `Competitor` is missing or empty, slug will be empty, causing API call to fail.  
  - *Version:* Uses typeVersion 2 of Code node.  

---

#### 1.2 Data Retrieval & Task Creation

**Overview:**  
This block takes the slug and performs a Crunchbase API call to fetch company details, then creates a review task in ClickUp with the retrieved data.

**Nodes Involved:**  
- Fetch Crunchbase Data  
- Create Review Task in ClickUp

**Node Details:**

- **Fetch Crunchbase Data**  
  - *Type:* HTTP Request  
  - *Role:* Queries Crunchbase API v4 to fetch organization data using the slug.  
  - *Configuration:*  
    - Method: GET  
    - URL: `https://api.crunchbase.com/api/v4/entities/organizations/{{ $json.slug }}`  
    - Query Parameter: `user_key` (Crunchbase API Key, must be replaced with a valid key)  
    - Sends no body payload.  
  - *Expressions:* Uses `{{ $json.slug }}` from previous node for dynamic URL path.  
  - *Connections:* Input from Generate Crunchbase Slug; outputs to Create Review Task in ClickUp.  
  - *Failures / Edge Cases:*  
    - Invalid or missing API key → authentication errors.  
    - Invalid slug → 404 not found or empty data.  
    - API rate limits or downtime.  
    - Network timeout.  
  - *Version:* 4.2  

- **Create Review Task in ClickUp**  
  - *Type:* ClickUp Node  
  - *Role:* Creates a task in ClickUp with a summary of the fetched competitor data for team review.  
  - *Configuration:*  
    - Task Name: Fixed string `"Review Crunchbase Update and inform your manager"`  
    - Content: Uses expressions to insert company properties such as:  
      - Name  
      - Last updated date  
      - Short description  
      - Total funding (USD)  
      - Last funding type  
      - Homepage URL  
    - Priority: 3 (medium)  
    - Status, DueDate, Assignees: left empty for manual adjustment.  
  - *Expressions:* Pulls data from `$json.data.properties` fields from Crunchbase API response.  
  - *Connections:* Input from Fetch Crunchbase Data; no output (end node).  
  - *Failures / Edge Cases:*  
    - Missing or invalid ClickUp credentials → auth errors.  
    - Missing or incomplete data fields → task content may have blanks.  
    - API rate limits or connectivity issues with ClickUp.  
  - *Version:* 1  

---

### 3. Summary Table

| Node Name              | Node Type       | Functional Role                       | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                       |
|------------------------|-----------------|------------------------------------|------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| Manual Trigger         | Trigger         | Initiates workflow manually          | —                      | Set Competitor Name         | Explains manual run for testing/debugging                                                       |
| Set Competitor Name    | Set             | Defines competitor company name      | Manual Trigger          | Generate Crunchbase Slug    | Describes setting competitor name manually, example input included                              |
| Generate Crunchbase Slug | Code (JavaScript) | Converts competitor name to API slug | Set Competitor Name     | Fetch Crunchbase Data       | Details slug generation logic for Crunchbase API compatibility                                  |
| Fetch Crunchbase Data  | HTTP Request    | Fetches competitor data from Crunchbase | Generate Crunchbase Slug | Create Review Task in ClickUp | Describes API call, required API key, and sample data retrieved                                 |
| Create Review Task in ClickUp | ClickUp        | Creates a task summarizing competitor data | Fetch Crunchbase Data    | —                          | Explains task creation with data fields for internal review                                     |
| Sticky Note            | Sticky Note     | Documentation section 1              | —                      | —                          | Contains detailed explanations of Section 1 (Input & Preparation)                               |
| Sticky Note1           | Sticky Note     | Documentation section 2              | —                      | —                          | Contains detailed explanations of Section 2 (Fetch + Create Task)                              |
| Sticky Note9           | Sticky Note     | Workflow assistance/contact info     | —                      | —                          | Support contact and useful social links                                                         |
| Sticky Note4           | Sticky Note     | Full workflow explanation and overview | —                      | —                          | Extended explanation of workflow purpose, nodes, and outcomes                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Manual Trigger`  
   - Purpose: Manually start the workflow for testing and manual runs.  
   - No parameters needed.  

3. **Add a Set node:**  
   - Name: `Set Competitor Name`  
   - Connect input from `Manual Trigger`.  
   - In Parameters, under `Values to Set`, add a string field named `Competitor`.  
   - Set default value, e.g., `"OpenAI"`. This can be changed later or replaced by dynamic input.  

4. **Add a Code node:**  
   - Name: `Generate Crunchbase Slug`  
   - Connect input from `Set Competitor Name`.  
   - Use JavaScript code to transform `Competitor` into a slug:  
     ```js
     return items.map(item => {
       const name = item.json.Competitor || "";
       const slug = name
         .toLowerCase()
         .trim()
         .replace(/[^a-z0-9\s-]/g, "")
         .replace(/\s+/g, "-");
       return { json: { ...item.json, slug } };
     });
     ```  
   - Purpose: Ensure the competitor name fits Crunchbase API slug format.  

5. **Add an HTTP Request node:**  
   - Name: `Fetch Crunchbase Data`  
   - Connect input from `Generate Crunchbase Slug`.  
   - Set Method: GET.  
   - URL: `https://api.crunchbase.com/api/v4/entities/organizations/{{ $json.slug }}` (use expression editor).  
   - Query Parameters: Add `user_key` with your Crunchbase API key (replace `"YOUR_API_KEY"`).  
   - Leave body empty (GET request).  
   - Purpose: Retrieve competitor organization data from Crunchbase API.  

6. **Add a ClickUp node:**  
   - Name: `Create Review Task in ClickUp`  
   - Connect input from `Fetch Crunchbase Data`.  
   - Configure credentials with valid ClickUp OAuth2 credentials.  
   - Task Name: `"Review Crunchbase Update and inform your manager"` (static).  
   - Content field: Use expressions to insert dynamic data from Crunchbase response:  
     ```
     Company: {{ $json.data.properties.name }}
     Last Updated: {{ $json.data.properties.updated_at }}
     Description: {{ $json.data.properties.short_description }}
     Total Funding: {{ $json.data.properties.total_funding_usd }}
     Last Funding: {{ $json.data.properties.last_funding_type }}
     Homepage: {{ $json.data.properties.homepage_url }}
     ```  
   - Set Priority to 3 (medium).  
   - Leave Status, Due Date, and Assignees empty or configure as needed.  

7. **Save and activate the workflow.**  
   - Run manually via `Manual Trigger` for testing.  
   - Verify output task in ClickUp.  

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Crunchbase API requires a valid user key for authentication. Replace `"YOUR_API_KEY"` with your actual key.                | Crunchbase API documentation: https://developer.crunchbase.com/docs/getting-started                                 |
| ClickUp node requires OAuth2 credentials setup with your personal/team account.                                             | ClickUp API docs: https://clickup.com/api                                                                           |
| Manual Trigger is ideal for testing; replace with Cron or Webhook trigger for full automation.                              | n8n trigger types: https://docs.n8n.io/nodes/trigger/                                                                |
| Workflow assistance contact: Yaron@nofluff.online; video tutorials at https://www.youtube.com/@YaronBeen/videos             | Support and community resources                                                                                      |
| Workflow is designed for single competitor tracking; for bulk competitor lists, replace Set node with a list iteration loop.| Consider using SplitInBatches or Google Sheets node for scalable automation                                         |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and extending the **CrunchBase Competitor Intelligence Tracker** workflow in n8n.