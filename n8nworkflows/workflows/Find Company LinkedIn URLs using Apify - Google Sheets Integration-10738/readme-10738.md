Find Company LinkedIn URLs using Apify - Google Sheets Integration

https://n8nworkflows.xyz/workflows/find-company-linkedin-urls-using-apify---google-sheets-integration-10738


# Find Company LinkedIn URLs using Apify - Google Sheets Integration

### 1. Workflow Overview

This workflow automates the process of finding LinkedIn URLs for companies listed in a Google Sheet by leveraging the Apify LinkedIn Company URL Finder actor. It is designed for sales, recruiting, marketing, research, and CRM enrichment teams who need to quickly map company names to their official LinkedIn pages and store these URLs in a new Google Sheet. The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Setup and input of the Google Sheet URL and sheet name containing company names.
- **1.2 Data Retrieval:** Reading company names from the specified Google Sheet.
- **1.3 Data Preparation:** Formatting the retrieved data to meet Apify actor input requirements.
- **1.4 Apify Actor Execution:** Running the Apify LinkedIn Company URL Finder actor for the input company list.
- **1.5 Result Retrieval:** Fetching the output dataset from Apify.
- **1.6 Output Storage:** Creating a new Google Sheet and appending the found LinkedIn URLs and company data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block sets the Google Sheet URL and sheet name where company names reside. It triggers the workflow manually and initializes input parameters.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Set google sheet URL & original sheet name

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution  
    - Configuration: No parameters, triggers workflow on user click  
    - Inputs: None  
    - Outputs: Connects to "Set google sheet URL & original sheet name"  
    - Failure Types: None expected  
    - Notes: Provides manual start control.

  - **Set google sheet URL & original sheet name**  
    - Type: Set Node  
    - Role: Defines variables `google_sheet_url` and `google_sheet_name` for subsequent nodes  
    - Configuration: Hardcoded string values; user must replace placeholder URL and sheet name with actual ones before execution  
    - Inputs: From manual trigger  
    - Outputs: Connects to "Create new sheet for founded compnaies" and "Get URLs from first sheet"  
    - Failure Types: Incorrect URL or sheet name will cause downstream failures in Google Sheets node  
    - Notes: Critical for input configuration; must be updated by user.

---

#### 2.2 Data Retrieval

- **Overview:**  
  Reads the list of company names from the specified Google Sheet to be processed.

- **Nodes Involved:**  
  - Get URLs from first sheet  
  - wait for previous nodes to finish

- **Node Details:**

  - **Get URLs from first sheet**  
    - Type: Google Sheets (Read)  
    - Role: Fetches data from the specified sheet (`google_sheet_name` in `google_sheet_url`)  
    - Configuration: Reads sheet by name dynamically using expressions `={{ $json.google_sheet_name }}` and URL `={{ $json.google_sheet_url }}`  
    - Credentials: Google Sheets OAuth2 with read access  
    - Inputs: Receives from "Set google sheet URL & original sheet name"  
    - Outputs: Connects to "wait for previous nodes to finish" (index 1)  
    - Failure Types: Authentication errors, invalid sheet name, empty sheet, or API timeouts  
    - Notes: Data expected includes company names per row.

  - **wait for previous nodes to finish**  
    - Type: Merge (Wait)  
    - Role: Synchronizes branches and ensures previous nodes have completed before continuing  
    - Configuration: Default merge mode (wait)  
    - Inputs: From "Get URLs from first sheet" (main input index 1) and "Create new sheet for founded compnaies" (main input index 0)  
    - Outputs: Connects to "format data for Apify INPUT type"  
    - Failure Types: Potential deadlocks if inputs do not arrive.

---

#### 2.3 Data Preparation

- **Overview:**  
  Formats the list of company names into the specific input schema required by the Apify LinkedIn Company URL Finder actor.

- **Nodes Involved:**  
  - format data for Apify INPUT type

- **Node Details:**

  - **format data for Apify INPUT type**  
    - Type: Code (JavaScript)  
    - Role: Transforms the array of company names into a single string with line-separated queries for Apify actor input  
    - Configuration: Iterates over all rows from "Get URLs from first sheet", concatenating the `name` property (company name) into a string under `queries` key  
    - Key Expression: Uses `$('Get URLs from first sheet').all()` to access previous node data  
    - Inputs: From "wait for previous nodes to finish"  
    - Outputs: Connects to "Run Actor on Apify"  
    - Failure Types: Expression errors if input data structure changes or missing `name` field  
    - Notes: Must maintain input format as per Apify actor schema: https://apify.com/dev_fusion/linkedin-profile-scraper/input-schema

---

#### 2.4 Apify Actor Execution

- **Overview:**  
  Runs the Apify LinkedIn Company URL Finder actor with the prepared input to retrieve LinkedIn URLs.

- **Nodes Involved:**  
  - Run Actor on Apify  
  - Get Results from Apify

- **Node Details:**

  - **Run Actor on Apify**  
    - Type: Apify Node (Actor Run)  
    - Role: Executes the Apify actor `9X6Pju8NeHNTvzRxF` (LinkedIn Company URL Finder) with the provided input  
    - Configuration: Uses `customBody` to pass input JSON from previous node; actor selected from store  
    - Credentials: Apify API key with access to this actor  
    - Inputs: From "format data for Apify INPUT type"  
    - Outputs: Connects to "Get Results from Apify"  
    - Failure Types: API errors, actor timeouts, invalid input format, quota limits on Apify account  
    - Notes: Execution is asynchronous; output dataset ID is needed for results retrieval.

  - **Get Results from Apify**  
    - Type: Apify Node (Dataset Fetch)  
    - Role: Retrieves the results dataset using the `defaultDatasetId` from the actor run output  
    - Configuration: Dataset resource, datasetId passed dynamically from previous node JSON  
    - Credentials: Apify API key  
    - Inputs: From "Run Actor on Apify"  
    - Outputs: Connects to "Add company urls into the new Sheet"  
    - Failure Types: Dataset not found, API errors, network issues.

---

#### 2.5 Output Storage

- **Overview:**  
  Creates a new Google Sheet within the same Google Sheet document and appends the retrieved LinkedIn URLs along with company info.

- **Nodes Involved:**  
  - Create new sheet for founded compnaies  
  - Add company urls into the new Sheet

- **Node Details:**

  - **Create new sheet for founded compnaies**  
    - Type: Google Sheets (Create Sheet)  
    - Role: Creates a new sheet tab in the existing Google Sheet document to store results  
    - Configuration: Sheet title dynamically generated as `company-urls-dd/mm-HH:mm` using date formatting at runtime  
    - Credentials: Google Sheets OAuth2 with write access  
    - Inputs: From "Set google sheet URL & original sheet name"  
    - Outputs: Connects to "wait for previous nodes to finish" (main input index 0)  
    - Failure Types: API quota, permission errors, naming conflicts  
    - Notes: Ensures results are stored separately.

  - **Add company urls into the new Sheet**  
    - Type: Google Sheets (Append Rows)  
    - Role: Appends the retrieved company LinkedIn URL data into the newly created sheet  
    - Configuration: Uses auto-mapping of fields to Google Sheet columns; sheetName referenced by sheet ID from "Create new sheet for founded compnaies" node; documentId from initial settings node  
    - Credentials: Google Sheets OAuth2 with write access  
    - Inputs: From "Get Results from Apify"  
    - Outputs: None (end of workflow)  
    - Failure Types: Data mapping issues, permission errors, API limits  
    - Notes: Columns include comprehensive company and profile info as per Apify output.

---

### 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                              | Input Node(s)                         | Output Node(s)                          | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|----------------------------------|----------------------------|----------------------------------------------|-------------------------------------|----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger             | Manual start trigger                         | None                                | Set google sheet URL & original sheet name |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Set google sheet URL & original sheet name | Set Node                  | Input initialization - sets Google Sheet URL and sheet name | When clicking ‘Execute workflow’    | Create new sheet for founded compnaies, Get URLs from first sheet |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Create new sheet for founded compnaies | Google Sheets (Create Sheet) | Creates new sheet tab for output results     | Set google sheet URL & original sheet name | wait for previous nodes to finish        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Get URLs from first sheet         | Google Sheets (Read)       | Reads company names from original sheet      | Set google sheet URL & original sheet name | wait for previous nodes to finish        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| wait for previous nodes to finish | Merge Node                | Synchronizes input data before proceeding    | Create new sheet for founded compnaies, Get URLs from first sheet | format data for Apify INPUT type          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| format data for Apify INPUT type  | Code (JavaScript)          | Formats company names for Apify actor input  | wait for previous nodes to finish    | Run Actor on Apify                      |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Run Actor on Apify                | Apify Node (Actor Run)     | Runs Apify LinkedIn Company URL Finder actor | format data for Apify INPUT type     | Get Results from Apify                   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Get Results from Apify            | Apify Node (Dataset Fetch) | Fetches results dataset from Apify           | Run Actor on Apify                   | Add company urls into the new Sheet      |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Add company urls into the new Sheet | Google Sheets (Append Rows) | Appends retrieved LinkedIn URLs to new sheet | Get Results from Apify               | None                                   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Sticky Note7                     | Sticky Note                | Describes workflow purpose and usage         | None                                | None                                   | ## Try It Out!\n\nThis n8n template shows how to populate a Google Spreadsheet with LinkedIn company URLs automatically using the [Apify LinkedIn Company URL Finder actor](https://apify.com/anchor/linkedin-company-url-finder) from [Anchor](https://apify.com/anchor). It will create a new sheet with the matched LinkedIn URLs.\n\nYou can use it to speed up lead research, keep CRM records consistent, or prep outreach lists — all directly inside n8n.\n\n### Who is this for\n- Sales Teams: Map accounts to their official LinkedIn pages fast.\n- Recruiters: Locate company pages before sourcing.\n- Growth Marketers: Clean and enrich account lists at scale.\n- Researchers: Track competitors and market segments.\n- CRM Builders: Normalize company records with an authoritative URL.\n- Lead-Gen Agencies: Deliver verified company URLs at volume.\n\n### How it works\n- Write a list of company names in Google Sheets (one per row)\n- The Apify node resolves each name to its LinkedIn company page\n- The results are then stored in a new Google Sheet\n\n### How to use\n\nIn Google Sheets:\n- Create a Google Sheet, rename the sheet companies, and add all the company names you want to resolve (one per row)\n\n\nIn this Workflow:\n- Open “Set google sheet URL & original sheet name” and replace the example Google Sheet URL, and the name of the sheet where your company names are.\n\n\nIn the n8n credentials:\n- Connect your Google Sheets account with read and write privileges.\n- Connect your Apify account.\n\n\nIn Apify:\n- Sign up for this [Apify Actor](https://apify.com/anchor/linkedin-company-url-finder)￼ \n\n### Requirements\n- Apify account with access to LinkedIn Company URL Finder.\n- A list of company names to process.\n\n### Need Help?\nOpen an issue directly on Apify! Avg answer in less than 24h\n\n\nHappy URL Finding! |
| Sticky Note                      | Sticky Note                | Shows input example JSON                     | None                                | None                                   | ## Input Example\n```json\n[\n {\n   \"google_sheet_url\": \"https://docs.google.com/spreadsheets/d/1ffxxxxxxxxxxvI/\",\n   \"google_sheet_name\": \"profiles\"\n }\n]\n```                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Sticky Note1                     | Sticky Note                | Shows output example screenshot               | None                                | None                                   | ## Output Example\n\n![google sheet output example](https://i.postimg.cc/nczBd890/Screenshot-2025-11-11-at-21-41-19.png)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Sticky Note2                     | Sticky Note                | Shows example of Google Sheet input screenshot | None                                | None                                   | ## Google Sheet INPUT Example\n\n![google sheet example](https://i.postimg.cc/xdKzknBG/Screenshot-2025-11-11-at-21-41-32.png)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node for Input Configuration**  
   - Type: Set  
   - Name: `Set google sheet URL & original sheet name`  
   - Parameters:  
     - `google_sheet_url`: Set to your Google Sheet document URL (string).  
     - `google_sheet_name`: Set to the name of the sheet tab containing company names (string).  
   - Connect output of Manual Trigger to this node.

3. **Create Google Sheets Node to Create Results Sheet**  
   - Type: Google Sheets  
   - Operation: Create Sheet  
   - Document ID: Use the URL from `google_sheet_url` (expression: `={{ $json.google_sheet_url }}`)  
   - Title: Use expression for dynamic title, e.g., `company-urls-{{new Date().format('dd/mm-HH:mm')}}`  
   - Connect output of Set node to this node.  
   - Credentials: Google Sheets OAuth2 with read/write privileges.

4. **Create Google Sheets Node to Read Company Names**  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Document ID: Use expression `={{ $json.google_sheet_url }}`  
   - Sheet Name: Use expression `={{ $json.google_sheet_name }}`  
   - Connect output of Set node to this node.  
   - Credentials: Google Sheets OAuth2 with read privileges.

5. **Create Merge Node to Wait for Both Previous Nodes**  
   - Type: Merge  
   - Mode: Wait for all inputs  
   - Connect outputs of "Create new sheet for founded companies" (main input 0) and "Get URLs from first sheet" (main input 1) to this node.

6. **Create Code Node to Format Data for Apify Input**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     let listOfCompanies = ''
     for (const item of $('Get URLs from first sheet').all()) {
       listOfCompanies += `\n${item.json.name}`
     }
     return { 
       json: {
         queries : listOfCompanies
       }
     }
     ```  
   - Connect output of Merge node to this node.

7. **Create Apify Actor Run Node**  
   - Type: Apify  
   - Actor ID: `9X6Pju8NeHNTvzRxF` (LinkedIn Company URL Finder)  
   - Actor Source: Store  
   - Custom Body: Use expression `={{ $json }}` to pass formatted input  
   - Connect output of Code node to this node.  
   - Credentials: Apify API key with access to this actor.

8. **Create Apify Dataset Fetch Node**  
   - Type: Apify  
   - Resource: Datasets  
   - Dataset ID: Use expression `={{ $json.defaultDatasetId }}` to fetch results from actor run  
   - Connect output of Apify Actor Run node to this node.  
   - Credentials: Apify API key.

9. **Create Google Sheets Append Node to Add Results**  
   - Type: Google Sheets  
   - Operation: Append Rows  
   - Document ID: Use expression from the Set node `={{ $json.google_sheet_url }}`  
   - Sheet Name: Use sheet ID from "Create new sheet for founded companies" node `={{ $('Create new sheet for founded compnaies').first().json.sheetId }}`  
   - Columns: Auto-map input data (from Apify results dataset) to sheet columns  
   - Connect output of Apify Dataset Fetch node to this node.  
   - Credentials: Google Sheets OAuth2 with write privileges.

10. **Ensure All Node Connections Follow the Above Step Order**  
    Manual Trigger → Set Input → Create Results Sheet → Read Company Names → Merge Wait → Format Data → Run Apify Actor → Get Results → Append to Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This n8n template demonstrates automatically populating a Google Spreadsheet with LinkedIn company URLs using the [Apify LinkedIn Company URL Finder actor](https://apify.com/anchor/linkedin-company-url-finder). It supports sales, recruiting, marketing, research, CRM normalization, and lead generation workflows.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | [Apify LinkedIn Company URL Finder](https://apify.com/anchor/linkedin-company-url-finder)                        |
| Input example JSON format and Google Sheet structure are critical: company names must be in a single column (header `name` expected).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Input Example Sticky Note in workflow                                                                            |
| Output is stored in a newly created sheet with dynamic timestamped naming to prevent overwrites.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Output Example Sticky Note with sample screenshot                                                                |
| To use this workflow, configure Google Sheets OAuth2 credentials with read/write access and an Apify account with access to the LinkedIn Company URL Finder actor.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Credential setup                                                                                                 |
| For detailed Apify actor input schema and troubleshooting, refer to the [Apify LinkedIn Profile Scraper input schema](https://apify.com/dev_fusion/linkedin-profile-scraper/input-schema).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Apify documentation link                                                                                        |
| For support on the Apify actor or issues, open an issue directly on Apify platform—average response time under 24 hours.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Support note from sticky note                                                                                     |

---

This documentation provides a complete, structured understanding of the LinkedIn URL Finder workflow, enabling both advanced users and AI agents to reproduce, modify, or troubleshoot the workflow effectively.