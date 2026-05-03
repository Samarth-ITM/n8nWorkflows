Automated Competitor Intelligence: CrunchBase to ClickUp Tracking Workflow

https://n8nworkflows.xyz/workflows/automated-competitor-intelligence--crunchbase-to-clickup-tracking-workflow-4728


# Automated Competitor Intelligence: CrunchBase to ClickUp Tracking Workflow

### 1. Workflow Overview

This workflow, named **"Competitor tracking on CrunchBase"**, automates competitor intelligence gathering by querying company data from Crunchbase and creating actionable review tasks in ClickUp. It is designed for teams needing to track competitor updates efficiently without manual data collection.

**Use Cases:**  
- Marketing or competitive intelligence teams monitoring competitor funding, descriptions, or other key company data.  
- Product or business teams receiving automated alerts on competitor changes.  
- Streamlining manual research into automated workflows.

**Logical Blocks:**  
- **1.1 Input & Preparation:** Receives manual trigger, sets the competitor name, and formats it into a Crunchbase-compatible slug.  
- **1.2 Data Retrieval & Task Creation:** Fetches competitor data from Crunchbase API and creates a task in ClickUp for team review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Preparation

**Overview:**  
This block allows user-controlled initiation of the workflow with manual input of the competitor's name and converts that name into a slug format suitable for Crunchbase API usage.

**Nodes Involved:**  
- Manual Trigger  
- Set Competitor Name  
- Generate Crunchbase Slug

**Node Details:**

- **Manual Trigger**  
  - *Type:* Manual trigger node  
  - *Role:* Starts the workflow on user command, enabling testing or manual runs.  
  - *Config:* No parameters; simply a button to manually start.  
  - *Connections:* Outputs to "Set Competitor Name" node.  
  - *Edge Cases:* No scheduled runs; if forgotten to trigger manually, the workflow won't run.  
  - *Notes:* Ideal for debugging or initial testing.  

- **Set Competitor Name**  
  - *Type:* Set node  
  - *Role:* Defines the competitor company name as a workflow variable.  
  - *Config:* Sets a string field `Competitor` with a fixed value ("OpenAI" by default).  
  - *Connections:* Receives input from "Manual Trigger", outputs to "Generate Crunchbase Slug".  
  - *Edge Cases:* If competitor name is empty or malformed, later steps may fail or return invalid slugs.  
  - *Notes:* Can be replaced with dynamic input sources (e.g., Google Sheets) for full automation.  

- **Generate Crunchbase Slug**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Converts the competitor name into a Crunchbase-compatible slug for API calls.  
  - *Config:* Custom JS script that lowercases, removes special characters, replaces spaces with hyphens.  
  - *Key Expression:*  
    ```js
    const slug = name.toLowerCase().trim().replace(/[^a-z0-9\s-]/g, "").replace(/\s+/g, "-");
    ```  
  - *Connections:* Input from "Set Competitor Name", outputs to "Fetch Crunchbase Data".  
  - *Edge Cases:* Names with unusual characters or empty strings may produce invalid slugs.  
  - *Version Requirements:* Code node version 2 or higher recommended for modern JS syntax.  

---

#### 1.2 Data Retrieval & Task Creation

**Overview:**  
This block fetches the competitor's detailed data from Crunchbase using the slug, then creates a task in ClickUp to notify the team for review.

**Nodes Involved:**  
- Fetch Crunchbase Data  
- Create Review Task in ClickUp

**Node Details:**

- **Fetch Crunchbase Data**  
  - *Type:* HTTP Request node  
  - *Role:* Calls Crunchbase API to retrieve company information using the slug.  
  - *Config:*  
    - Method: GET  
    - URL Template: `https://api.crunchbase.com/api/v4/entities/organizations/{{ $json.slug }}`  
    - Query Parameter: `user_key` with Crunchbase API key placeholder `YOUR_API_KEY`  
    - Sends query parameters, no body.  
  - *Connections:* Input from "Generate Crunchbase Slug", outputs to "Create Review Task in ClickUp".  
  - *Edge Cases:*  
    - API key missing or invalid → authentication errors.  
    - Incorrect slug → 404 or empty data.  
    - API rate limits or downtime → request failures or timeouts.  
  - *Version Requirements:* HTTP Request node version 4.2 or higher recommended.  
  - *Notes:* Requires valid Crunchbase API key credential configured in n8n.  

- **Create Review Task in ClickUp**  
  - *Type:* ClickUp node  
  - *Role:* Creates a new task in ClickUp with company data summary for team to review.  
  - *Config:*  
    - Task Name: Static "Review Crunchbase Update and inform your manager"  
    - Content: Populated with Crunchbase data fields: company name, last update, description, funding, homepage URL.  
    - Priority set to 3 (medium)  
    - Assignees and due date left empty (configurable)  
  - *Connections:* Input from "Fetch Crunchbase Data", no output (terminal node).  
  - *Edge Cases:*  
    - Invalid ClickUp credentials → auth errors.  
    - API rate limits or network errors.  
    - Missing or malformed Crunchbase data results in incomplete task description.  
  - *Notes:* Requires ClickUp OAuth2 credentials configured in n8n.  

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                            | Input Node(s)          | Output Node(s)                   | Sticky Note                                                                                   |
|-------------------------|--------------------|------------------------------------------|-----------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger          | Manual Trigger      | Starts workflow manually                  | -                     | Set Competitor Name             | Allows manual testing and debugging of the workflow                                          |
| Set Competitor Name     | Set                | Defines competitor name                   | Manual Trigger        | Generate Crunchbase Slug        | Allows customization of competitor; can be replaced by dynamic data source                   |
| Generate Crunchbase Slug| Code               | Converts competitor name to Crunchbase slug | Set Competitor Name   | Fetch Crunchbase Data           | Ensures proper slug format for Crunchbase API URL                                            |
| Fetch Crunchbase Data   | HTTP Request       | Retrieves competitor data from Crunchbase | Generate Crunchbase Slug | Create Review Task in ClickUp | Requires Crunchbase API key; fetches company info like funding, description, homepage        |
| Create Review Task in ClickUp | ClickUp          | Creates a review task in ClickUp          | Fetch Crunchbase Data | -                               | Automates task creation for team review of competitor updates                                |
| Sticky Note             | Sticky Note        | Documentation and guidance notes          | -                     | -                               | Section 1: Input & Preparation details                                                       |
| Sticky Note1            | Sticky Note        | Documentation and guidance notes          | -                     | -                               | Section 2: Fetch + Create Task details                                                       |
| Sticky Note4            | Sticky Note        | Overall workflow summary and instructions | -                     | -                               | Full workflow explanation and outcomes                                                      |
| Sticky Note9            | Sticky Note        | Support contact and resource links        | -                     | -                               | Workflow assistance contacts and learning resources                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Manual Trigger" Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually for testing or on-demand runs.  
   - No parameters required.  

2. **Create "Set Competitor Name" Node**  
   - Type: Set  
   - Purpose: Define the competitor's name to track.  
   - Configuration: Add a string field named `Competitor` with a default value, e.g., `"OpenAI"`.  
   - Connect output of "Manual Trigger" to input of this node.  

3. **Create "Generate Crunchbase Slug" Node**  
   - Type: Code (JavaScript)  
   - Purpose: Transform competitor name into Crunchbase slug format.  
   - Code snippet:  
     ```js
     return items.map(item => {
       const name = item.json.Competitor || "";
       const slug = name.toLowerCase().trim().replace(/[^a-z0-9\s-]/g, "").replace(/\s+/g, "-");
       return { json: {...item.json, slug } };
     });
     ```  
   - Connect output of "Set Competitor Name" to input of this node.  

4. **Create "Fetch Crunchbase Data" Node**  
   - Type: HTTP Request  
   - Purpose: Retrieve competitor data from Crunchbase API.  
   - Method: GET  
   - URL: `https://api.crunchbase.com/api/v4/entities/organizations/{{ $json.slug }}`  
   - Query Parameters: `user_key` with your Crunchbase API key (set via credentials or environment variable).  
   - Connect output of "Generate Crunchbase Slug" to input of this node.  
   - Ensure you have Crunchbase API credentials configured in n8n.  

5. **Create "Create Review Task in ClickUp" Node**  
   - Type: ClickUp  
   - Purpose: Create a task summarizing competitor data for team review.  
   - Parameters:  
     - Name: `"Review Crunchbase Update and inform your manager"`  
     - Content (task description): Populate dynamically with Crunchbase data fields, for example:  
       ```
       Company: {{ $json.data.properties.name }}
       Last Updated: {{ $json.data.properties.updated_at }}
       Description: {{ $json.data.properties.short_description }}
       Total Funding: {{ $json.data.properties.total_funding_usd }}
       Last Funding: {{ $json.data.properties.last_funding_type }}
       Homepage: {{ $json.data.properties.homepage_url }}
       ```  
     - Priority: 3 (medium)  
     - Assignees: Leave empty or configure as needed  
     - Due Date: Leave empty or configure as needed  
   - Connect output of "Fetch Crunchbase Data" to input of this node.  
   - Configure ClickUp OAuth2 credentials in n8n beforehand.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                      |
|------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Workflow assistance and support contact: Yaron@nofluff.online                                                                | Support email                                                      |
| Explore more tips and tutorials on YouTube and LinkedIn by Yaron Been                                                         | YouTube: https://www.youtube.com/@YaronBeen/videos                 |
| LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                              | Professional profile                                               |
| The workflow automates competitor tracking from Crunchbase to ClickUp task creation, removing manual monitoring overhead.      | Workflow purpose                                                  |
| Ensure Crunchbase API key and ClickUp OAuth2 credentials are securely configured in n8n for proper operation.                 | Credential management                                             |

---

**Disclaimer:**  
The text provided originates exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.