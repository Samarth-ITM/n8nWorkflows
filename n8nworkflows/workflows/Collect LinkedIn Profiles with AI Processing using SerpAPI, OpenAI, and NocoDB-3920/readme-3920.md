Collect LinkedIn Profiles with AI Processing using SerpAPI, OpenAI, and NocoDB

https://n8nworkflows.xyz/workflows/collect-linkedin-profiles-with-ai-processing-using-serpapi--openai--and-nocodb-3920


# Collect LinkedIn Profiles with AI Processing using SerpAPI, OpenAI, and NocoDB

### 1. Workflow Overview

This workflow automates the collection of LinkedIn profile data based on user-defined search criteria such as keyword and location. It uses SerpAPI to perform Google searches for LinkedIn profiles, processes and enriches the data with OpenAI to extract company names and convert follower counts into numeric values, cleans the dataset by discarding irrelevant metadata, and finally outputs the results in two forms: an Excel file downloadable by the user and a database table stored in NocoDB.

**Target Use Cases:**  
- Lead generation and prospecting from LinkedIn profiles  
- Market research based on LinkedIn user data  
- Building datasets of professional profiles filtered by keyword and geography  

**Logical Blocks:**  
- **1.1 Input Reception & Parameter Setup:** Accepts user input parameters for LinkedIn profile search filters.  
- **1.2 Data Retrieval via SerpAPI:** Queries Google search via SerpAPI to retrieve LinkedIn profiles matching criteria.  
- **1.3 Data Processing & Transformation:** Splits search results, selects relevant fields, and enriches data with OpenAI.  
- **1.4 Data Cleaning:** Removes unnecessary OpenAI metadata and extracts final useful fields.  
- **1.5 Data Output:** Combines final data and stores it into an Excel file and a NocoDB table for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parameter Setup

**Overview:**  
This block initializes the workflow and defines the search parameters including keyword, location, language, number of results, and search engine settings.

**Nodes Involved:**  
- Manual Trigger  
- Search parameter  
- Sticky Note10 (explains parameter setup)

**Node Details:**  

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Starts the workflow manually  
  - Configuration: No parameters; simply triggers the flow  
  - Inputs: None  
  - Outputs: Connects to "Search parameter" node  
  - Potential Failures: None expected here

- **Search parameter**  
  - Type: Set node  
  - Role: Defines search criteria parameters as static values  
  - Configuration:  
    - Keyword: "nocode"  
    - Location: "Germany"  
    - Number of search results to be returned: 20  
    - Host language: "en"  
    - Geolocation: "de"  
    - Search engine: "google"  
    - Site: "linkedin.com/in" (limits search to LinkedIn profiles)  
  - Inputs: From "Manual Trigger"  
  - Outputs: To "Google search w/ SerpAPI"  
  - Edge Cases: User must update parameters as needed before running; static values limit flexibility without modification  
  - Notes: Sticky Note10 provides detailed explanation on parameters and links to SerpAPI docs

- **Sticky Note10**  
  - Type: Sticky Note  
  - Role: Documentation and guidance on search parameter setup  
  - Content: Detailed explanation of parameters for Google search via SerpAPI with links to SerpAPI and n8n docs

---

#### 2.2 Data Retrieval via SerpAPI

**Overview:**  
This block makes an authenticated HTTP request to SerpAPI to perform a Google search query based on the parameters set earlier, retrieving LinkedIn profile search results.

**Nodes Involved:**  
- Google search w/ SerpAPI  
- Sticky Note1 (credentials and overview)  
- Sticky Note2 (data filtering explanation)  
- Sticky Note3 (data discard explanation)

**Node Details:**  

- **Google search w/ SerpAPI**  
  - Type: HTTP Request node  
  - Role: Executes Google search via SerpAPI API  
  - Configuration:  
    - URL: https://serpapi.com/search  
    - Query parameters dynamically built from input:  
      - q: "site:linkedin.com/in" + Keyword + Location  
      - hl: Host language  
      - gl: Geolocation  
      - num: Number of results  
      - engine: Search engine (google)  
    - Authentication: Uses SerpAPI credentials stored in n8n  
  - Inputs: From "Search parameter"  
  - Outputs: To "Turn search results into individual items"  
  - Edge Cases:  
    - API quota limits or invalid credentials cause request failure  
    - Network timeouts or SerpAPI service downtime  
  - Notes: Sticky Note1 explains credential setup; Sticky Notes2 and 3 explain metadata filtering rationale

- **Sticky Note1, 2, 3**  
  - Provide setup instructions and clarify that only organic results are preserved for further processing

---

#### 2.3 Data Processing & Transformation

**Overview:**  
This block splits the list of search results, selects and renames fields for clarity, and enriches each profile’s data by extracting the company name and converting follower count text to numeric values using OpenAI.

**Nodes Involved:**  
- Turn search results into individual items  
- Edit Fields  
- Company name & followers  
- Sticky Note4 (OpenAI usage instructions)

**Node Details:**  

- **Turn search results into individual items**  
  - Type: SplitOut node  
  - Role: Splits the 'organic_results' array from SerpAPI into individual items for processing  
  - Configuration: Field to split out: "organic_results"  
  - Inputs: From "Google search w/ SerpAPI"  
  - Outputs: To "Edit Fields"  
  - Edge Cases: If no search results, node outputs zero items, halting downstream processing

- **Edit Fields**  
  - Type: Set node  
  - Role: Extracts and renames relevant fields for clarity and workflow consistency  
  - Configuration:  
    - Maps:  
      - NameInLinkedinProfile ← title  
      - linkedinUrl ← link  
      - Snippet ← snippet  
      - Followers ← displayed_link (holds follower count text)  
      - Keyword ← from "Search parameter" node  
      - Location ← from "Search parameter" node  
      - Rich snippet ← rich_snippet.top.extensions  
      - snippet_highlighted_words ← snippet_highlighted_words  
  - Inputs: From "Turn search results into individual items"  
  - Outputs: To "Company name & followers"  
  - Edge Cases: Missing fields in input JSON can cause empty values

- **Company name & followers**  
  - Type: OpenAI (Language Model Chat) node  
  - Role: Uses GPT-4o model to extract company name from profile title or snippet and convert follower count text into a numeric value  
  - Configuration:  
    - Model: GPT-4o  
    - Prompt: "Transform {{Followers}} into a number and extract where possible the name of the company in {{NameInLinkedinProfile}} or {{Snippet}}. Do not output things like location or name, only followers and company_name."  
    - JSON output enabled  
  - Inputs: From "Edit Fields"  
  - Outputs: To "Discard meta data"  
  - Credentials: OpenAI API key configured in n8n  
  - Edge Cases:  
    - OpenAI API quota exceeded or invalid credentials cause failure  
    - Unexpected input formats can cause extraction errors or incomplete data  
  - Notes: Sticky Note4 provides setup and usage instructions

---

#### 2.4 Data Cleaning

**Overview:**  
Filters the output from OpenAI to retain only the essential fields: numeric follower count and company name.

**Nodes Involved:**  
- Discard meta data  
- Sticky Note (explains the discard)

**Node Details:**  

- **Discard meta data**  
  - Type: Set node  
  - Role: Keeps only the final cleaned fields: followers_number and NameOfCompany as extracted from OpenAI response  
  - Configuration:  
    - followers_number ← message.content.followers (number)  
    - NameOfCompany ← message.content.company_name (string)  
  - Inputs: From "Company name & followers"  
  - Outputs: To "Generate final data via merge"  
  - Edge Cases: If OpenAI output is malformed or missing fields, values may be empty

- **Sticky Note (at this stage)**  
  - Explains this node discards irrelevant OpenAI metadata

---

#### 2.5 Data Output

**Overview:**  
Combines original fields with cleaned data, then outputs the complete dataset both as a downloadable Excel file and as records in a NocoDB database table.

**Nodes Involved:**  
- Generate final data via merge  
- LinkedIn profiles in Excel for download  
- Store data in a NocoDB table  
- Sticky Note5, Sticky Note9, Sticky Note11 (output explanations)

**Node Details:**  

- **Generate final data via merge**  
  - Type: Merge node  
  - Role: Combines streams of data by position: original profile data with cleaned OpenAI output  
  - Configuration: Mode "combine" by position  
  - Inputs:  
    - From "Edit Fields" (original fields) - main input 2 (index 1)  
    - From "Discard meta data" (cleaned fields) - main input 1 (index 0)  
  - Outputs: To Excel and NocoDB nodes  
  - Edge Cases: Mismatched input lengths could cause data misalignment

- **LinkedIn profiles in Excel for download**  
  - Type: ConvertToFile node  
  - Role: Converts merged data into an Excel spreadsheet (.xlsx) file for download  
  - Configuration: Operation: xlsx  
  - Inputs: From "Generate final data via merge"  
  - Outputs: None (file available for download in UI)  
  - Notes: Sticky Note9 instructs user to open node and click download button to get Excel file  
  - Edge Cases: Large datasets might cause performance issues or file size limits

- **Store data in a NocoDB table**  
  - Type: NocoDB node  
  - Role: Inserts the merged data into a configured NocoDB database table for persistent storage  
  - Configuration:  
    - Project ID and Table ID set to user’s NocoDB project/table  
    - Operation: create (insert new rows)  
    - Data to Send: autoMapInputData (maps all input fields)  
    - Authentication: NocoDB API Token credentials  
  - Inputs: From "Generate final data via merge"  
  - Outputs: None  
  - Edge Cases:  
    - Misconfigured or missing NocoDB credentials cause failures  
    - Table schema mismatches can cause insertion errors  
  - Notes: Sticky Note11 provides detailed setup instructions for NocoDB table fields and node configuration

- **Sticky Note5, 9, 11**  
  - Provide explanations for final output generation, Excel file creation, and NocoDB storage setup and usage

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                                     | Input Node(s)                | Output Node(s)                              | Sticky Note                                                                                                                  |
|----------------------------------|---------------------|----------------------------------------------------|------------------------------|---------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger                   | Trigger             | Starts the workflow                                | None                         | Search parameter                            |                                                                                                                              |
| Search parameter                 | Set                 | Defines search criteria parameters                 | Manual Trigger               | Google search w/ SerpAPI                    | Sticky Note10: Explains Google search parameters setup with links to docs                                                     |
| Google search w/ SerpAPI         | HTTP Request        | Executes Google search via SerpAPI                  | Search parameter             | Turn search results into individual items  | Sticky Note1: Setup instructions for SerpAPI credentials; Sticky Note2 & 3: Data filtering explanations                      |
| Turn search results into individual items | SplitOut            | Splits search results array into individual items  | Google search w/ SerpAPI     | Edit Fields                                |                                                                                                                              |
| Edit Fields                     | Set                 | Selects and renames relevant fields                 | Turn search results into individual items | Company name & followers                   |                                                                                                                              |
| Company name & followers        | OpenAI (LM Chat)    | Extracts company name and converts follower count  | Edit Fields                  | Discard meta data                          | Sticky Note4: Instructions for OpenAI setup and prompt usage                                                                 |
| Discard meta data               | Set                 | Keeps only relevant cleaned fields                  | Company name & followers     | Generate final data via merge               | Sticky Note (near OpenAI discard): Explains removal of irrelevant OpenAI metadata                                              |
| Generate final data via merge    | Merge               | Combines original and cleaned data                  | Edit Fields, Discard meta data | LinkedIn profiles in Excel for download, Store data in a NocoDB table | Sticky Note5: Explains final data creation                                                                                     |
| LinkedIn profiles in Excel for download | ConvertToFile       | Creates Excel file for download                      | Generate final data via merge | None                                        | Sticky Note9: Explains Excel node usage and download process                                                                  |
| Store data in a NocoDB table     | NocoDB              | Stores data in NocoDB database table                 | Generate final data via merge | None                                        | Sticky Note11: Detailed setup for NocoDB table and node configuration                                                        |
| Sticky Note1                    | Sticky Note         | Guidance on SerpAPI credential setup                 | None                         | None                                        |                                                                                                                              |
| Sticky Note2                    | Sticky Note         | Explains filtering to organic search results only   | None                         | None                                        |                                                                                                                              |
| Sticky Note3                    | Sticky Note         | Explains discarding irrelevant metadata             | None                         | None                                        |                                                                                                                              |
| Sticky Note4                    | Sticky Note         | Explains OpenAI usage for company name & followers | None                         | None                                        |                                                                                                                              |
| Sticky Note5                    | Sticky Note         | Explains final data output creation                  | None                         | None                                        |                                                                                                                              |
| Sticky Note9                    | Sticky Note         | Explains Excel file creation and download            | None                         | None                                        |                                                                                                                              |
| Sticky Note10                   | Sticky Note         | Explains SerpAPI Google search parameter setup      | None                         | None                                        |                                                                                                                              |
| Sticky Note11                   | Sticky Note         | Explains NocoDB database setup and data storage     | None                         | None                                        |                                                                                                                              |
| Sticky Note12                   | Sticky Note         | Overview of problem solved by workflow               | None                         | None                                        |                                                                                                                              |
| Sticky Note13                   | Sticky Note         | Overview of workflow actions                          | None                         | None                                        |                                                                                                                              |
| Sticky Note14                   | Sticky Note         | Step-by-step instructions summary                     | None                         | None                                        |                                                                                                                              |
| Sticky Note15                   | Sticky Note         | Explanation of workflow logic steps                   | None                         | None                                        |                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Manual Trigger node:**  
   - Name: Manual Trigger  
   - No parameters needed

3. **Add Set node to define search parameters:**  
   - Name: Search parameter  
   - Add fields:  
     - Keyword: string, e.g., "nocode"  
     - Location: string, e.g., "Germany"  
     - Number of search results to be returned: string (number), e.g., 20  
     - Host language: string, e.g., "en"  
     - Geolocation: string, e.g., "de"  
     - Search engine: string, "google"  
     - site: string, "linkedin.com/in"  
   - Connect Manual Trigger → Search parameter

4. **Add HTTP Request node for SerpAPI search:**  
   - Name: Google search w/ SerpAPI  
   - Method: GET  
   - URL: https://serpapi.com/search  
   - Query Parameters (all dynamic expressions referencing Search parameter node):  
     - q = `site:{{ $json.site }} {{ $json.Keyword }} {{ $json.Location }}`  
     - hl = `{{ $json['Host langauge'] }}`  
     - gl = `{{ $json.Geolocation }}`  
     - num = `{{ $json['Number of search results to be returned'] }}`  
     - engine = `{{ $json['Search engine'] }}`  
   - Authentication: Use SerpAPI credentials (set up in n8n credentials)  
   - Connect Search parameter → Google search w/ SerpAPI

5. **Add SplitOut node to process organic results:**  
   - Name: Turn search results into individual items  
   - Field to split out: "organic_results"  
   - Connect Google search w/ SerpAPI → Turn search results into individual items

6. **Add Set node to select and rename relevant fields:**  
   - Name: Edit Fields  
   - Set fields (with expressions referencing current JSON and Search parameter node):  
     - NameInLinkedinProfile = `{{$json.title}}`  
     - linkedinUrl = `{{$json.link}}`  
     - Snippet = `{{$json.snippet}}`  
     - Followers = `{{$json.displayed_link}}`  
     - Keyword = `{{ $('Search parameter').item.json.Keyword }}`  
     - Location = `{{ $('Search parameter').item.json.Location }}`  
     - Rich snippet = `{{$json.rich_snippet.top.extensions}}`  
     - snippet_highlighted_words = `{{$json.snippet_highlighted_words}}`  
   - Connect Turn search results into individual items → Edit Fields

7. **Add OpenAI node for enrichment:**  
   - Name: Company name & followers  
   - Credentials: OpenAI API (configure with API key)  
   - Model: GPT-4o (or desired model)  
   - Parameters:  
     - Messages: one message with content:  
       `Transform  {{ $json.Followers }} into a number and extract where possible the name of the company in {{ $json.NameInLinkedinProfile }} or in {{ $json.Snippet }} Do not output things like location or name, only followers and company_name`  
   - Enable JSON output  
   - Connect Edit Fields → Company name & followers

8. **Add Set node to discard extraneous OpenAI metadata:**  
   - Name: Discard meta data  
   - Set fields:  
     - followers_number = `{{$json.message.content.followers}}` (number)  
     - NameOfCompany = `{{$json.message.content.company_name}}` (string)  
   - Connect Company name & followers → Discard meta data

9. **Add Merge node to combine original and cleaned data:**  
   - Name: Generate final data via merge  
   - Mode: Combine  
   - Combine by: Position  
   - Connect Edit Fields (main input 2) and Discard meta data (main input 1) to this node

10. **Add ConvertToFile node to export Excel:**  
    - Name: LinkedIn profiles in Excel for download  
    - Operation: xlsx  
    - Connect Generate final data via merge → LinkedIn profiles in Excel for download

11. **Add NocoDB node to store data in database:**  
    - Name: Store data in a NocoDB table  
    - Authentication: NocoDB API token credentials (set up with your API token)  
    - Project ID: Your NocoDB project ID  
    - Table: Your configured table name (see next step)  
    - Operation: Create (insert)  
    - Data to Send: autoMapInputData  
    - Connect Generate final data via merge → Store data in a NocoDB table

12. **Set up NocoDB table:**  
    - Create fields:  
      - NameInLinkedinProfile (Single line text)  
      - NameOfCompany (Single line text)  
      - linkedinUrl (URL)  
      - Followers (Single line text)  
      - followers_number (Number)  
      - Keyword (Single line text)  
      - Location (Single line text)  
      - Rich snippet (Long text)  
      - snippet_highlighted_words (Long text)

13. **Review and test workflow:**  
    - Verify all credentials (SerpAPI, OpenAI, NocoDB) are correctly configured  
    - Run the workflow from Manual Trigger  
    - Check output Excel file by opening the ConvertToFile node and downloading the file  
    - Confirm data insertion into NocoDB table

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow uses SerpAPI.com to perform Google searches while avoiding blocks or CAPTCHA issues.                                              | https://serpapi.com                                                                                        |
| OpenAI GPT-4o model is used to parse textual follower counts into numbers and extract company names.                                       | https://openai.com                                                                                         |
| NocoDB is used as an open-source alternative to Airtable for storing and managing data in a database accessible via API.                 | https://nocodb.com                                                                                         |
| Detailed SerpAPI search parameters and their explanations are available at SerpAPI blog and n8n documentation.                           | https://serpapi.com/blog/google-search-parameters/ and https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolserpapi/ |
| OpenAI integration in n8n documented at https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatopenai/ |                                                                                                            |
| NocoDB integration documentation for n8n available at https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.nocodb/            |                                                                                                            |
| To prevent errors during execution, ensure each credential is properly created and tested before running the workflow.                    |                                                                                                            |
| The workflow is designed to be beginner-friendly, minimizing costly API usage by targeting only relevant data and reducing calls where possible. |                                                                                                            |

---

This comprehensive documentation should enable users and AI agents to fully understand, reproduce, and extend the workflow with confidence.