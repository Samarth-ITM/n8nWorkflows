Capture Website Form Submissions to Notion CRM Database

https://n8nworkflows.xyz/workflows/capture-website-form-submissions-to-notion-crm-database-5505


# Capture Website Form Submissions to Notion CRM Database

### 1. Workflow Overview

This workflow captures website form submissions and stores the cleaned data into a Notion CRM database. It is designed for use cases such as project inquiries, job applications, client onboarding, or internal requests submitted via a web form. The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Receives incoming form data via a webhook triggered by form submissions.
- **1.2 Data Parsing and Cleaning:** Processes and restructures the raw form data into a standardized format.
- **1.3 Data Storage:** Maps and inserts the cleaned data as a new page in a Notion CRM database.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  This block receives form submission data from a website using an HTTP POST webhook. It acts as the entry point for all data and triggers the workflow.

- **Nodes Involved:**  
  - Form Submission Hook

- **Node Details:**  

  - **Form Submission Hook**  
    - Type: Webhook  
    - Role: Receives incoming HTTP POST requests containing form data.  
    - Configuration:  
      - HTTP Method: POST  
      - Webhook Path: A unique identifier string (`34e9fb3f-f6bd-4a44-bb58-6fe58ffe4a78`) used to generate the webhook URL.  
      - No additional options configured.  
    - Key Expressions/Variables: None (raw POST body is used)  
    - Input Connections: None (starting node)  
    - Output Connections: Passes data to "Parse + Clean Lead Data" node  
    - Version Requirements: Requires n8n version supporting Webhook node v2 or higher for stable webhook management.  
    - Potential Failures:  
      - Webhook URL misconfiguration or unavailability  
      - Invalid or missing POST data  
      - HTTP method mismatch (should be POST)  
    - Sticky Note: Guidance on setting up the webhook URL in the website form and testing submission.

#### Block 1.2: Data Parsing and Cleaning

- **Overview:**  
  This block processes the raw JSON payload from the webhook, extracting specific fields and normalizing them into a clean object structure suitable for downstream processing.

- **Nodes Involved:**  
  - Parse + Clean Lead Data

- **Node Details:**  

  - **Parse + Clean Lead Data**  
    - Type: Code (JavaScript)  
    - Role: Extracts and renames specific fields from the webhook payload to create a structured data object.  
    - Configuration:  
      - JavaScript Code processes all incoming items:  
        - Extracted fields: `name`, `email`, `businessName`, `message` (renamed to `project intent/need`), `timeline`, `budget`  
        - Returns an array of cleaned items with these fields as properties.  
    - Key Expressions/Variables:  
      - Uses `$input.all()` to access all incoming webhook items  
      - Maps `message` field to `project intent/need` for clarity  
    - Input Connections: Receives raw webhook data from "Form Submission Hook"  
    - Output Connections: Passes cleaned data to "Notion" node  
    - Version Requirements: Requires n8n supporting Code node v2 or higher for improved JS execution environment  
    - Potential Failures:  
      - Missing expected fields in incoming data may cause undefined values  
      - JavaScript runtime errors if payload structure changes unexpectedly  
    - Sticky Note: Explains the parsing logic and the fields extracted.

#### Block 1.3: Data Storage

- **Overview:**  
  This block takes the cleaned data and creates a new page in a Notion database, mapping each cleaned field to the corresponding Notion property.

- **Nodes Involved:**  
  - Notion

- **Node Details:**  

  - **Notion**  
    - Type: Notion node  
    - Role: Creates a new page in a specified Notion database with mapped data fields.  
    - Configuration:  
      - `pageId`: Configured with a URL mode for specifying the target Notion database page URL (not preset in the JSON, must be set during setup)  
      - Block UI: Empty, default settings  
      - Options: None specified  
    - Key Expressions/Variables:  
      - Maps cleaned data fields from the Code node output to Notion database properties, e.g.:  
        - Name → `{{$json["name"]}}`  
        - Email → `{{$json["email"]}}`  
        - Business Name → `{{$json["businessName"]}}`  
        - Project Intent/Need → `{{$json["project intent/need"]}}`  
        - Timeline → `{{$json["timeline"]}}`  
        - Budget → `{{$json["budget"]}}`  
    - Input Connections: Receives cleaned data from "Parse + Clean Lead Data"  
    - Output Connections: None (terminal node)  
    - Version Requirements: Requires n8n with Notion node version 2.2 or higher for enhanced Notion API integration  
    - Potential Failures:  
      - Authentication errors if Notion credentials are invalid or expired  
      - Missing or incorrect Notion database ID or page URL  
      - Property mapping mismatches (e.g., data types incompatible)  
      - API rate limits or downtime  
    - Sticky Note: Provides detailed mapping instructions for Notion properties.

---

### 3. Summary Table

| Node Name              | Node Type        | Functional Role              | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                                  |
|------------------------|------------------|-----------------------------|-----------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Form Submission Hook   | Webhook          | Receives form submission data| None                  | Parse + Clean Lead Data   | 1. **Webhook Node**  • Purpose: Receives form data.  • Setup instructions for webhook URL and testing form submission.       |
| Parse + Clean Lead Data| Code             | Parses and cleans form data | Form Submission Hook   | Notion                   | **Parse Data**  • Extracts fields: name, email, businessName, message (as project intent/need), timeline, budget.              |
| Notion                | Notion           | Stores data into Notion DB  | Parse + Clean Lead Data| None                     | ## Mapping to Notion  • Maps each cleaned data field to corresponding Notion property.                                         |
| Sticky Note           | Sticky Note      | Documentation               | None                  | None                     | ## Website Form Submission to Notion CRM Setup  **Author David Olusola**  Workflow flexibility and customization notes.       |
| Sticky Note1          | Sticky Note      | Documentation               | None                  | None                     | 1.**Webhook Node** Setup details and test instructions.                                                                       |
| Sticky Note2          | Sticky Note      | Documentation               | None                  | None                     | **Parse Data** Extraction and restructuring explanation.                                                                       |
| Sticky Note3          | Sticky Note      | Documentation               | None                  | None                     | ## Mapping to Notion property mapping instructions with examples.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a new Webhook node named "Form Submission Hook".  
   - Set HTTP Method to `POST`.  
   - Define the Webhook Path as a unique string (e.g., `34e9fb3f-f6bd-4a44-bb58-6fe58ffe4a78`).  
   - Save and activate the node to generate the webhook URL.  
   - Configure your website form to send a POST request to this webhook URL upon submission.

2. **Add Code Node for Data Parsing**  
   - Add a Code node named "Parse + Clean Lead Data".  
   - Connect it to the "Form Submission Hook" node.  
   - Configure it with the following JavaScript code:

   ```javascript
   const items = $input.all();

   const updatedItems = items.map((item) => {
     const { name, email, businessName, message, timeline, budget } = item.json.body;
     return {
       name,
       email,
       businessName,
       "project intent/need": message,
       timeline,
       budget,
     };
   });

   return updatedItems;
   ```

   - This code extracts expected fields from each form submission and renames `message` to `project intent/need`.

3. **Add Notion Node**  
   - Add a Notion node named "Notion".  
   - Connect it to the "Parse + Clean Lead Data" node.  
   - Configure Notion credentials (OAuth2 or Integration Token) with appropriate access to your target Notion workspace.  
   - Set the `pageId` parameter by specifying the URL of your Notion CRM database where you want to create new pages.  
   - Map each Notion database property to the corresponding field from the Code node output using expressions like:  
     - Name → `{{$json["name"]}}`  
     - Email → `{{$json["email"]}}`  
     - Business Name → `{{$json["businessName"]}}`  
     - Project Intent/Need → `{{$json["project intent/need"]}}`  
     - Timeline → `{{$json["timeline"]}}`  
     - Budget → `{{$json["budget"]}}`

4. **Test the Workflow**  
   - Activate the workflow.  
   - Submit the website form with test data.  
   - Confirm data appears correctly as a new page in your Notion CRM database.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| The workflow is highly customizable. You can add or remove form fields and update the Code node and Notion mappings accordingly.                                    | Sticky Note by author David Olusola                                |
| For detailed setup of the webhook node and testing form submissions, follow the guidelines in Sticky Note1.                                                         | Sticky Note1 in the workflow                                      |
| Detailed mapping instructions for Notion properties ensure data consistency and proper CRM database integration.                                                    | Sticky Note3 in the workflow                                      |
| To expand functionality, consider adding error handling nodes (e.g., Function or IF nodes) to manage missing data or API errors gracefully.                         | General best practice recommendation                              |
| Ensure your Notion credentials have sufficient permissions to create pages in the target database to avoid authentication or permission errors.                      | Notion API and n8n credential setup                               |
| For live demos and further customization ideas, check out n8n community forums and the official Notion API documentation.                                            | https://community.n8n.io/ and https://developers.notion.com/docs    |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.