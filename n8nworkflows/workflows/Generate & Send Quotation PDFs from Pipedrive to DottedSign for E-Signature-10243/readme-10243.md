Generate & Send Quotation PDFs from Pipedrive to DottedSign for E-Signature

https://n8nworkflows.xyz/workflows/generate---send-quotation-pdfs-from-pipedrive-to-dottedsign-for-e-signature-10243


# Generate & Send Quotation PDFs from Pipedrive to DottedSign for E-Signature

### 1. Workflow Overview

This workflow automates the generation and delivery of sales quotations by integrating Pipedrive CRM with the DottedSign e-signature platform. It triggers when a deal in Pipedrive moves to a specific stage (the "Quotation" stage), collects all relevant deal data, generates a PDF quotation using an HTML template and a PDF conversion service (Gotenberg), uploads the generated PDF back to Pipedrive, and sends the document for electronic signature via DottedSign.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Initial Condition Check:** Watches for a specific stage change in Pipedrive deals to start the process.
- **1.2 Data Collection from Pipedrive:** Retrieves deal details, associated products, contact person, organization data, and custom field mappings.
- **1.3 Data Cleaning and Mapping:** Processes and merges collected data into a structured format suitable for template filling.
- **1.4 PDF Quotation Generation:** Builds an HTML quotation, converts it to a PDF using Gotenberg, and prepares the file.
- **1.5 Upload & E-Signature Process:** Uploads the PDF back to Pipedrive, obtains an access token from DottedSign, and creates a signing task to send the quote for signature.
- **1.6 Activity Logging:** Creates an activity record in Pipedrive related to the quotation.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initial Condition Check

**Overview:**  
This block monitors changes on deals in Pipedrive and filters for those moved specifically to the "Quotation" stage (stage ID 7). Only qualifying deals proceed.

**Nodes Involved:**  
- Pipedrive Trigger  
- If Stage is 'Quotation'  

**Node Details:**

- **Pipedrive Trigger**  
  - Type: Pipedrive Trigger node  
  - Role: Watches all deal changes in Pipedrive  
  - Configured to trigger on any deal change event  
  - Credentials: Uses sandbox Pipedrive API credentials  
  - Output: Emits deal data JSON for further processing  
  - Failure types: API connectivity issues, webhook registration failures  

- **If Stage is 'Quotation'**  
  - Type: If node (conditional filter)  
  - Role: Checks if deal's stage_id equals 7 (Quotation stage) and previous stage_id was less than 7  
  - Uses expressions to compare current and previous stage IDs  
  - Input: Pipedrive Trigger node  
  - Output: Passes deals that satisfy conditions to next steps  
  - Failure types: Expression evaluation errors if data fields missing  

---

#### 1.2 Data Collection from Pipedrive

**Overview:**  
Fetches detailed data for the deal, associated person, organization, deal products, and retrieves custom field definitions for mapping.

**Nodes Involved:**  
- Get Pipedrive Custom Fields  
- Split Out Custom Fields  
- editFlag=True (Filter node)  
- Get a deal  
- Get many deal products  
- Get a person  
- Get an organization  
- Create an activity  

**Node Details:**

- **Get Pipedrive Custom Fields**  
  - Type: HTTP Request  
  - Role: Retrieves all deal custom fields definitions from Pipedrive API  
  - Uses HTTP Query Auth with Pipedrive API token  
  - Input: If Stage is 'Quotation' node  
  - Output: JSON array of custom deal fields  
  - Edge cases: API token expiration, rate limits  

- **Split Out Custom Fields**  
  - Type: Split Out node  
  - Role: Splits the array of custom fields into individual items for filtering  
  - Input: Get Pipedrive Custom Fields  
  - Output: Individual custom field objects  

- **editFlag=True**  
  - Type: Filter node  
  - Role: Filters only items where `edit_flag` is true (used for further processing)  
  - Input: Split Out Custom Fields  
  - Output: Filtered custom fields for aggregation  

- **Get a deal**  
  - Type: Pipedrive node (Get operation)  
  - Role: Retrieves full details of the deal by ID  
  - Input: If Stage is 'Quotation'  
  - Output: Deal details JSON  

- **Get many deal products**  
  - Type: Pipedrive node (GetAll operation for dealProduct resource)  
  - Role: Retrieves all products associated with the deal  
  - Input: If Stage is 'Quotation'  
  - Output: Array of deal products  

- **Get a person**  
  - Type: Pipedrive node (Get operation for person resource)  
  - Role: Retrieves contact person details linked to the deal  
  - Input: If Stage is 'Quotation'  
  - Output: Person details JSON  

- **Get an organization**  
  - Type: Pipedrive node (Get operation for organization resource)  
  - Role: Retrieves organization details linked to the deal  
  - Input: If Stage is 'Quotation'  
  - Output: Organization details JSON  

- **Create an activity**  
  - Type: Pipedrive node (Create operation for activity resource)  
  - Role: Creates a "Quotation" activity linked to the deal for logging purposes  
  - Input: If Stage is 'Quotation'  
  - Parameters: Subject includes quotation number and date  
  - Output: Activity details JSON  

---

#### 1.3 Data Cleaning and Mapping

**Overview:**  
Aggregates all collected data streams, cleans up and maps custom field IDs to user-friendly names and values, and prepares a unified data object for the quotation template.

**Nodes Involved:**  
- Aggregate  
- Aggregate1  
- field_mappings (Set)  
- deal_details (Set)  
- person_details (Set)  
- deal_products (Set)  
- org_details (Set)  
- activity_details (Set)  
- MergeAllData (Merge)  
- Clean & Map Data (Code)  

**Node Details:**

- **Aggregate** and **Aggregate1**  
  - Type: Aggregate nodes  
  - Role: Aggregate all item data from filtered custom fields and deal products respectively  
  - Input: editFlag=True (custom fields), Get many deal products  
  - Output: Aggregated arrays  

- **Set nodes (field_mappings, deal_details, person_details, deal_products, org_details, activity_details)**  
  - Type: Set nodes  
  - Role: Rename and structure JSON data streams for clarity and easier merging  
  - Input: Respective data retrieval nodes  
  - Output: JSON objects with explicit property names  

- **MergeAllData**  
  - Type: Merge node (combine mode)  
  - Role: Combines all structured data streams into one JSON object  
  - Input: field_mappings, deal_details, person_details, deal_products, org_details, activity_details  
  - Output: Single combined data object for processing  

- **Clean & Map Data**  
  - Type: Code node (JavaScript)  
  - Role:  
    - Converts custom field IDs to readable field names and labels  
    - Resolves enum and set type custom fields to their label values  
    - Maps nested objects to readable values  
    - Embeds person and organization details within cleaned deal object  
  - Input: MergeAllData  
  - Output: Cleaned and mapped JSON data for HTML template  
  - Edge cases: Missing data fields, malformed custom field mappings  

---

#### 1.4 PDF Quotation Generation

**Overview:**  
Generates HTML content for the quotation using the cleaned data, converts it to a binary HTML file, then sends it to Gotenberg for PDF conversion.

**Nodes Involved:**  
- setProductRows (Code)  
- Generate Quote HTML (HTML)  
- Convert to HTML (ConvertToFile)  
- Gotenberg to PDF (HTTP Request)  
- Rename PDF (Code)  

**Node Details:**

- **setProductRows**  
  - Type: Code node  
  - Role: Dynamically generates HTML table rows for the product list in the quotation  
  - Uses fields: product name, quantity, currency, unit price, discount, total sum  
  - Returns concatenated HTML string for insertion in main template  
  - Input: Clean & Map Data's deal_products array  
  - Output: HTML snippet for products table rows  

- **Generate Quote HTML**  
  - Type: HTML node  
  - Role: Main quotation HTML template with placeholders filled using expressions from cleaned data and product rows  
  - Contains inline CSS to style the quotation document  
  - Input: setProductRows output for products table rows, plus cleaned deal data  
  - Output: Final HTML string for conversion  

- **Convert to HTML**  
  - Type: ConvertToFile node  
  - Role: Converts the HTML string into a binary file with UTF-8 encoding and MIME type text/html  
  - Input: Generate Quote HTML  
  - Output: Binary data representing the HTML file  

- **Gotenberg to PDF**  
  - Type: HTTP Request node  
  - Role: Sends the binary HTML file to a Gotenberg server endpoint to convert HTML to PDF  
  - Configured as multipart/form-data with the HTML file attached  
  - URL must be replaced with the user's Gotenberg instance  
  - Output: PDF file binary response  
  - Edge cases: Network timeouts, incorrect Gotenberg URL, conversion errors  

- **Rename PDF**  
  - Type: Code node  
  - Role: Sets the binary file name to 'result.pdf' for consistency  
  - Input: Gotenberg to PDF output  
  - Output: Renamed binary file for next steps  

---

#### 1.5 Upload & E-Signature Process

**Overview:**  
Uploads the generated PDF to the Pipedrive deal files, converts it to Base64, obtains a DottedSign access token, and creates a DottedSign signing task to send the quotation for e-signature.

**Nodes Involved:**  
- Create a file (Pipedrive)  
- Extract from File  
- Get DottedSign Access Token (HTTP Request)  
- MergeForDS (Merge)  
- DottedSign-CreateTask (HTTP Request)  

**Node Details:**

- **Create a file**  
  - Type: Pipedrive node (file resource create)  
  - Role: Uploads the PDF file to the "Files" section of the deal and links it to the activity  
  - Inputs: Binary PDF from Rename PDF, deal ID, activity ID  
  - Output: File upload response  
  - Edge cases: File size limits, API errors  

- **Extract from File**  
  - Type: ExtractFromFile node  
  - Role: Converts the binary PDF file into Base64 string property required by DottedSign API  
  - Input: Create a file output (binary PDF)  
  - Output: Base64 encoded PDF string  

- **Get DottedSign Access Token**  
  - Type: HTTP Request node  
  - Role: Requests OAuth2 client_credentials access token from DottedSign API  
  - Requires client_id and client_secret in body parameters (to be configured by user)  
  - Output: Access token JSON  

- **MergeForDS**  
  - Type: Merge node (combine mode)  
  - Role: Combines Base64 encoded PDF and access token into single data object for next request  
  - Input: Extract from File and Get DottedSign Access Token  

- **DottedSign-CreateTask**  
  - Type: HTTP Request node  
  - Role: Creates a signing task in DottedSign with PDF and signer information  
  - Sends JSON body including:  
    - Signer name and email from person details  
    - Signature field settings with coordinates on PDF page  
    - Base64 encoded PDF file and file name (sanitized)  
    - CC info with owner's contact  
  - Headers include Authorization Bearer token  
  - Edge cases: Invalid token, coordinate mismatch for signature field, API rate limits  

---

#### 1.6 Activity Logging

**Overview:**  
Creates a Pipedrive activity to log the creation of the quotation for traceability.

**Nodes Involved:**  
- Create an activity  
- activity_details (Set)  

**Node Details:**

- **Create an activity**  
  - Already described in Data Collection block; creates quotation activity linked to deal  
  - Input: If Stage is 'Quotation'  
  - Output: Activity details  

- **activity_details**  
  - Type: Set node  
  - Role: Stores activity details for use downstream (e.g., linking files)  

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                                     | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                 |
|--------------------------|---------------------------|----------------------------------------------------|--------------------------|-------------------------------|-----------------------------------------------------------------------------|
| Pipedrive Trigger        | Pipedrive Trigger         | Watches deal changes to trigger workflow            | -                        | If Stage is 'Quotation'        | STEP 1: Trigger & Data Collection                                          |
| If Stage is 'Quotation'  | If                        | Filters deals that entered Quotation stage (ID=7)  | Pipedrive Trigger         | Get Pipedrive Custom Fields, Get a deal, Get many deal products, Get a person, Get an organization, Create an activity | STEP 1: Trigger & Data Collection                                          |
| Get Pipedrive Custom Fields | HTTP Request             | Retrieves deal custom field definitions             | If Stage is 'Quotation'   | Split Out Custom Fields        | STEP 1: Trigger & Data Collection                                          |
| Split Out Custom Fields  | Split Out                 | Splits custom fields array for filtering            | Get Pipedrive Custom Fields | editFlag=True                | STEP 1: Trigger & Data Collection                                          |
| editFlag=True            | Filter                    | Filters custom fields where edit_flag is true       | Split Out Custom Fields   | Aggregate                     | STEP 1: Trigger & Data Collection                                          |
| Aggregate                | Aggregate                 | Aggregates filtered custom fields                    | editFlag=True             | field_mappings(Set)            | STEP 2: Data Processing                                                    |
| Get a deal               | Pipedrive                 | Retrieves full deal details by ID                    | If Stage is 'Quotation'   | deal_details(Set)              | STEP 2: Data Processing                                                    |
| Get many deal products   | Pipedrive                 | Retrieves all products associated with deal         | If Stage is 'Quotation'   | Aggregate1                    | STEP 2: Data Processing                                                    |
| Aggregate1               | Aggregate                 | Aggregates deal products data                         | Get many deal products    | deal_products(Set)             | STEP 2: Data Processing                                                    |
| Get a person             | Pipedrive                 | Retrieves contact person linked to deal              | If Stage is 'Quotation'   | person_details(Set)            | STEP 2: Data Processing                                                    |
| Get an organization      | Pipedrive                 | Retrieves organization details linked to deal       | If Stage is 'Quotation'   | org_details(Set)               | STEP 2: Data Processing                                                    |
| Create an activity       | Pipedrive                 | Creates a 'Quotation' activity for logging           | If Stage is 'Quotation'   | activity_details(Set)          | STEP 1: Trigger & Data Collection                                          |
| field_mappings           | Set                       | Renames aggregated custom fields data                | Aggregate                 | MergeAllData                  | STEP 2: Data Processing                                                    |
| deal_details             | Set                       | Renames deal detail data                              | Get a deal                | MergeAllData                  | STEP 2: Data Processing                                                    |
| person_details           | Set                       | Renames person detail data                            | Get a person              | MergeAllData                  | STEP 2: Data Processing                                                    |
| deal_products            | Set                       | Renames deal products data                            | Aggregate1                | MergeAllData                  | STEP 2: Data Processing                                                    |
| org_details              | Set                       | Renames organization detail data                      | Get an organization       | MergeAllData                  | STEP 2: Data Processing                                                    |
| activity_details         | Set                       | Renames activity detail data                           | Create an activity        | MergeAllData                  | STEP 1: Trigger & Data Collection                                          |
| MergeAllData             | Merge                     | Combines all data streams into one JSON object       | field_mappings, deal_details, person_details, deal_products, org_details, activity_details | Clean & Map Data            | STEP 2: Data Processing                                                    |
| Clean & Map Data         | Code                      | Maps custom fields to readable names and merges data | MergeAllData              | setProductRows                | STEP 2: Data Processing                                                    |
| setProductRows           | Code                      | Generates HTML rows for product list                  | Clean & Map Data          | Generate Quote HTML           | STEP 3: PDF Generation                                                     |
| Generate Quote HTML      | HTML                      | Creates full quotation HTML template                   | setProductRows            | Convert to HTML               | STEP 3: PDF Generation                                                     |
| Convert to HTML          | ConvertToFile             | Converts HTML string to binary file                    | Generate Quote HTML       | Gotenberg to PDF             | STEP 3: PDF Generation                                                     |
| Gotenberg to PDF         | HTTP Request              | Converts HTML binary to PDF via Gotenberg service     | Convert to HTML           | Rename PDF                   | STEP 3: PDF Generation                                                     |
| Rename PDF               | Code                      | Renames binary PDF file to 'result.pdf'                | Gotenberg to PDF          | Create a file, Extract from File | STEP 4: Final Actions                                                  |
| Create a file            | Pipedrive                 | Uploads PDF file to Pipedrive deal files               | Rename PDF                | Get DottedSign Access Token   | STEP 4: Final Actions                                                      |
| Extract from File        | ExtractFromFile           | Converts PDF binary to Base64 for API                   | Create a file             | MergeForDS                   | STEP 4: Final Actions                                                      |
| Get DottedSign Access Token | HTTP Request            | Obtains OAuth2 token from DottedSign                    | Create a file             | MergeForDS                   | STEP 4: Final Actions                                                      |
| MergeForDS               | Merge                     | Combines Base64 PDF and access token                    | Extract from File, Get DottedSign Access Token | DottedSign-CreateTask       | STEP 4: Final Actions                                                      |
| DottedSign-CreateTask    | HTTP Request              | Creates DottedSign e-signature task with PDF and signer | MergeForDS                | -                            | STEP 4: Final Actions                                                      |
| Sticky Note              | Sticky Note               | Overview and instructions for the workflow              | -                        | -                            | Contains detailed project overview and setup instructions                 |
| Sticky Note2             | Sticky Note               | Describes STEP 1 block                                    | -                        | -                            | Covers trigger and data collection overview                              |
| Sticky Note3             | Sticky Note               | Describes STEP 2 block                                    | -                        | -                            | Covers data processing and mapping                                       |
| Sticky Note4             | Sticky Note               | Describes STEP 3 block                                    | -                        | -                            | Covers PDF generation steps                                              |
| Sticky Note5             | Sticky Note               | Describes STEP 4 block                                    | -                        | -                            | Covers file upload and e-signature steps                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Pipedrive Trigger Node**  
   - Type: Pipedrive Trigger  
   - Set to trigger on "deal" entity changes  
   - Connect Pipedrive credentials with admin access  
   - Position: Initial node  

2. **Add If Node to Filter Deals**  
   - Type: If  
   - Condition: `$json.data.stage_id == 7` (Quotation stage) AND `$json.previous.stage_id < 7`  
   - Input: Pipedrive Trigger output  
   - This ensures the workflow runs only when deal moves into stage 7  

3. **Get Pipedrive Custom Fields**  
   - Type: HTTP Request  
   - URL: `https://<your-pipedrive-domain>/api/v1/dealFields`  
   - Authentication: HTTP Query Auth with API token  
   - Input: If node (true branch)  

4. **Split Out Custom Fields**  
   - Type: Split Out  
   - Field to split: `data` from previous HTTP Request  

5. **Filter Custom Fields with edit_flag True**  
   - Type: Filter node  
   - Condition: `$json.edit_flag == true`  

6. **Aggregate Custom Fields**  
   - Type: Aggregate  
   - Aggregate all filtered custom fields into array  

7. **Get Deal Details**  
   - Type: Pipedrive node (Get)  
   - Resource: Deal  
   - Parameter: `dealId = {{$json.data.id}}`  
   - Input: If node (true branch)  

8. **Get Deal Products**  
   - Type: Pipedrive node (GetAll)  
   - Resource: dealProduct  
   - Parameter: `dealId = {{$json.meta.entity_id}}`  
   - Return all products, no limit  

9. **Aggregate Deal Products**  
   - Type: Aggregate  
   - Aggregate all deal products into array  

10. **Get Person Details**  
    - Type: Pipedrive node (Get)  
    - Resource: Person  
    - Parameter: `personId = {{$json.data.person_id}}`  

11. **Get Organization Details**  
    - Type: Pipedrive node (Get)  
    - Resource: Organization  
    - Parameter: `organizationId = {{$json.data.org_id}}`  

12. **Create Quotation Activity**  
    - Type: Pipedrive node (Create)  
    - Resource: Activity  
    - Parameters:  
      - Type: "Quoatation" (note the typo, keep for reproduction or fix)  
      - Subject: `Quotation/ {{$json.data.id}}/{{ $json.data.update_time.toDateTime().format('yyyyMMdd') }}`  
      - Link to deal ID  

13. **Create Set Nodes** for each data stream: `field_mappings`, `deal_details`, `person_details`, `deal_products`, `org_details`, `activity_details`  
    - Each Set node assigns its input JSON data to a named property accordingly  

14. **Merge All Data**  
    - Type: Merge (combine mode)  
    - Inputs: all Set nodes above  
    - Output: combined JSON with all required data  

15. **Add Code Node "Clean & Map Data"**  
    - JavaScript code to:  
      - Map custom field keys to names  
      - Resolve enum/set values to labels  
      - Clean nested object fields  
      - Embed person and organization details inside deal data  
    - Input: MergeAllData output  

16. **Add Code Node "setProductRows"**  
    - JavaScript code to generate HTML table rows for each product  
    - Uses product name, quantity, unit price, discount, totals  
    - Input: cleaned data from previous Code node  

17. **Add HTML Node "Generate Quote HTML"**  
    - Paste provided HTML template with embedded expressions for dynamic data  
    - Include the product rows HTML from `setProductRows` node  
    - Input: setProductRows output  

18. **Convert HTML to Binary File**  
    - Type: ConvertToFile  
    - Encoding: UTF-8  
    - File name: index.html  
    - MIME type: text/html  

19. **Add HTTP Request Node "Gotenberg to PDF"**  
    - Method: POST  
    - URL: your Gotenberg instance endpoint (e.g., `http://gotenberg.yourhost/forms/chromium/convert/html`)  
    - Content Type: multipart/form-data  
    - Attach file from previous node binary data  
    - Output: PDF binary file  

20. **Rename PDF File**  
    - Code node to rename binary file name to 'result.pdf'  

21. **Upload PDF to Pipedrive Deal**  
    - Pipedrive node (Create file)  
    - Link file to deal ID and activity ID  
    - Use binary PDF from Rename PDF node  

22. **Extract PDF as Base64**  
    - ExtractFromFile node  
    - Converts binary PDF to Base64 string for API consumption  

23. **Get DottedSign Access Token**  
    - HTTP Request node  
    - POST to `https://api.sandbox.dottedsign.com/v1/oauth/token`  
    - Body parameters: `grant_type=client_credentials`, `client_id`, `client_secret` (user to configure)  
    - Headers: accept and content-type application/json  

24. **Merge Base64 PDF and Access Token**  
    - Merge node (combine mode)  
    - Inputs: Base64 PDF and DottedSign token  

25. **Create DottedSign Signing Task**  
    - HTTP Request node  
    - POST to `https://api.sandbox.dottedsign.com/v1/sign_tasks`  
    - JSON body includes:  
      - Signer info (name, email) from person details  
      - Signature field configuration (page and coordinates)  
      - Base64 PDF as `base64_file`  
      - File name sanitized from deal title  
      - CC info with owner details  
    - Authorization header with Bearer token from previous step  

26. **Connect all nodes following the described flow**  
    - Respect input/output dependencies as in the original workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow automates your sales quotation process by connecting Pipedrive with DottedSign. When a deal moves to a specific stage in Pipedrive, it generates a PDF quotation, uploads it to the deal, and sends it for e-signature via DottedSign, saving time and eliminating manual work. Requires Pipedrive admin and DottedSign developer accounts, and a self-hosted Gotenberg instance for PDF conversion. Customize the workflow by editing the stage trigger, Gotenberg URL, and DottedSign signature coordinates. | See Sticky Note content in the workflow for detailed overview and setup instructions                      |
| The quotation HTML template includes embedded CSS and dynamic placeholders using n8n expressions. To customize appearance or content, edit the Generate Quote HTML node. Signature field coordinates in DottedSign task creation must be adjusted to fit the PDF layout.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | HTML Template inside "Generate Quote HTML" node                                                          |
| The workflow creates a "Quotation" activity in Pipedrive to log the process and link uploaded files.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Node: Create an activity                                                                                  |
| The DottedSign API integration uses OAuth2 client credentials flow. Users must supply client_id and client_secret in the Get DottedSign Access Token node body parameters.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | DottedSign API documentation: https://developers.dottedsign.com/reference/oauth2                      |
| Gotenberg is a self-hosted HTML to PDF conversion service. Replace the placeholder URL in the "Gotenberg to PDF" node with your server.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Gotenberg docs: https://gotenberg.dev/docs/getting-started/installation                                   |
| The workflow handles edge cases such as missing data fields in custom field mappings and deals with Pipedrive API rate limits or authentication errors by design but implementing retry/error handling externally is recommended.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Best practice for error handling in n8n workflows                                                        |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects all applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.