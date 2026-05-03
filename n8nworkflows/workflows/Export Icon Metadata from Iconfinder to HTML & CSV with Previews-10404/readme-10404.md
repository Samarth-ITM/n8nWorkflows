Export Icon Metadata from Iconfinder to HTML & CSV with Previews

https://n8nworkflows.xyz/workflows/export-icon-metadata-from-iconfinder-to-html---csv-with-previews-10404


# Export Icon Metadata from Iconfinder to HTML & CSV with Previews

### 1. Workflow Overview

This workflow automates the export of icon metadata from an Iconfinder user account into two output formats: an HTML file with icon previews and a CSV file listing all icons along with their associated metadata. It targets designers, developers, or asset managers who want to catalog or review their icon collections efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Authentication & User Setup:** Sets API authorization and user ID for subsequent API calls.
- **1.2 Iconset Retrieval & Pagination Preparation:** Fetches the list of iconsets owned by the user and prepares multiple paginated API requests to retrieve all icons in each iconset.
- **1.3 Icon Data Retrieval & Extraction:** Retrieves detailed icon data, extracts key metadata such as tags, icon names, and generates preview URLs.
- **1.4 Output Generation:** Produces an HTML preview document and a CSV export file with the icon metadata.

Supporting nodes provide manual triggering and user guidance via sticky notes.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Authentication & User Setup

- **Overview:**  
  Initializes API credentials and user identifier required for all API requests.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Auth Settings  

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - *Type:* Manual Trigger  
     - *Role:* Entry point for manual execution  
     - *Configuration:* No parameters; triggers the workflow on demand  
     - *Connections:* Outputs to "Auth Settings"  
     - *Edge Cases:* None  

  2. **Auth Settings**  
     - *Type:* Set  
     - *Role:* Stores authorization token and user ID in workflow variables  
     - *Configuration:*  
       - `authorization` set as string "Bearer YOUR_TOKEN_HERE" (placeholder for API token)  
       - `user` set as string "YOUR USER ID" (placeholder for numeric user ID)  
     - *Connections:* Outputs to "Merge" and "Get Iconsets"  
     - *Edge Cases:* If token or user ID are missing/invalid, API calls will fail authentication.  
     - *Notes:* Requires user to replace placeholders with actual values before execution.

---

#### Block 1.2: Iconset Retrieval & Pagination Preparation

- **Overview:**  
  Queries the Iconfinder API for the list of iconsets belonging to the user and prepares paginated request URLs to fetch all icons from each iconset.

- **Nodes Involved:**  
  - Get Iconsets  
  - Split Iconsets  
  - Merge  

- **Node Details:**

  1. **Get Iconsets**  
     - *Type:* HTTP Request  
     - *Role:* Retrieves all iconsets for the user (max 100 per request)  
     - *Configuration:*  
       - URL templated with user ID: `https://api.iconfinder.com/v4/users/{{ $json.user }}/iconsets?count=100&offset=0`  
       - Header includes Authorization token from previous node  
     - *Inputs:* From "Auth Settings" (with credentials)  
     - *Outputs:* JSON response with iconsets array  
     - *Edge Cases:*  
       - API rate limits or authorization failure  
       - Large number of iconsets may need multiple paginations (not implemented here)  

  2. **Split Iconsets**  
     - *Type:* Code  
     - *Role:* Processes iconsets to generate multiple paginated URLs for icon retrieval  
     - *Configuration:*  
       - Iterates over iconsets, calculates number of requests needed per iconset based on icon count (max 100 icons per request)  
       - Creates output items each containing a URL to fetch a page of icons from a specific iconset  
     - *Inputs:* Output of "Get Iconsets"  
     - *Outputs:* Multiple items with fields: iconset_id, iconset_name, iconset_identifier, URL, offset  
     - *Edge Cases:*  
       - Iconsets with zero icons result in zero URLs  
       - Large icon counts generate multiple URLs correctly  
       - If iconsets field missing or empty, outputs empty list  

  3. **Merge**  
     - *Type:* Merge  
     - *Role:* Combines outputs from two branches to synchronize data for next API calls  
     - *Configuration:* Mode set to "combine" with "combineAll" option  
     - *Inputs:* In main branch 0 from "Auth Settings", branch 1 from "Split Iconsets"  
     - *Outputs:* Combined items for next node  
     - *Edge Cases:* Syncs data streams; may cause issues if one input is empty.

---

#### Block 1.3: Icon Data Retrieval & Extraction

- **Overview:**  
  Fetches icon details from each paginated URL, extracts relevant metadata including tags, icon names, iconset names, and selects best preview image URLs.

- **Nodes Involved:**  
  - Get Icons Details  
  - Extract Tags & Name  

- **Node Details:**

  1. **Get Icons Details**  
     - *Type:* HTTP Request  
     - *Role:* Fetches icon data per page URL generated previously  
     - *Configuration:*  
       - URL taken dynamically from input item (`{{$json.url}}`)  
       - Authorization header included from input (`{{$json.authorization}}`)  
     - *Inputs:* From "Merge" node  
     - *Outputs:* JSON response with icon details array  
     - *Edge Cases:*  
       - API errors if URL or authorization invalid  
       - Network timeouts or rate limiting possible  

  2. **Extract Tags & Name**  
     - *Type:* Code  
     - *Role:* Processes icon details to extract and normalize metadata per icon  
     - *Configuration:*  
       - Iterates over icons in each page  
       - Selects best preview URL by choosing largest raster size ≤128px or fallback to largest available  
       - Extracts filename from URL and normalizes icon name (removes suffixes, replaces underscores with spaces)  
       - Determines iconset name either from input or parses from URL if missing  
       - Packages icon ID, iconset ID, iconset name, icon name, tags (joined string), and preview URL for output  
     - *Inputs:* Output of "Get Icons Details"  
     - *Outputs:* Items with normalized icon metadata  
     - *Edge Cases:*  
       - Missing raster sizes or vector formats handled with fallback  
       - Missing iconset names attempt heuristic extraction from URL  
       - Empty or malformed URLs result in empty preview or name fields  

---

#### Block 1.4: Output Generation

- **Overview:**  
  Generates two output files: an HTML file displaying icons with previews and metadata in a table, and a CSV file listing all icons with metadata for cataloging.

- **Nodes Involved:**  
  - Create CSV  
  - Create HTML  

- **Node Details:**

  1. **Create CSV**  
     - *Type:* Convert To File  
     - *Role:* Converts JSON icon metadata to CSV file format  
     - *Configuration:* Default CSV conversion of incoming JSON data fields (icon_id, iconset_id, iconset_name, name, tags, preview_url)  
     - *Inputs:* Output of "Extract Tags & Name"  
     - *Outputs:* Binary CSV file  
     - *Edge Cases:*  
       - Empty input causes empty CSV file  
       - Special characters in names/tags correctly escaped by default CSV converter  

  2. **Create HTML**  
     - *Type:* Code  
     - *Role:* Constructs an HTML document string embedding icon previews in a styled table  
     - *Configuration:*  
       - Builds HTML with header, styling, and table rows for each icon  
       - Embeds preview image (64x64 px), icon name, tags, and iconset name  
       - Encodes resulting HTML string as base64 binary data with proper MIME type  
       - Outputs as downloadable file named "icons.html"  
     - *Inputs:* Output of "Extract Tags & Name"  
     - *Outputs:* Binary HTML file  
     - *Edge Cases:*  
       - Empty input results in minimal HTML with zero icons  
       - Missing preview URLs result in empty image src attributes  

---

#### Supporting Nodes: Instructions and Manual Trigger

- **Sticky Note (Multiple):**  
  Provide detailed user instructions on:  
  - Registering an application and obtaining API key  
  - How to determine user ID from an icon URL  
  - How to configure and execute the workflow  

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                              | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                             |
|----------------------------|---------------------|----------------------------------------------|-----------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Start workflow manually                       | -                           | Auth Settings                   |                                                                                                                         |
| Auth Settings              | Set                 | Set authorization token and user ID          | When clicking ‘Execute workflow’ | Merge, Get Iconsets             | ## 3. Config & Execute<br>1) Open the "Auth Settings" node and set the `authorization` value to: Bearer YOUR_TOKEN_HERE<br>2) Change the `user` value to the user_id obtained from previous step.<br>3) Execute the workflow. |
| Get Iconsets               | HTTP Request        | Retrieve user’s iconsets                       | Auth Settings               | Split Iconsets                  |                                                                                                                         |
| Split Iconsets             | Code                | Generate paginated icon list URLs             | Get Iconsets                | Merge                          |                                                                                                                         |
| Merge                     | Merge               | Combine auth info and paginated icon URLs    | Auth Settings, Split Iconsets | Get Icons Details              |                                                                                                                         |
| Get Icons Details          | HTTP Request        | Fetch icon details per paginated URL          | Merge                       | Extract Tags & Name            |                                                                                                                         |
| Extract Tags & Name        | Code                | Extract metadata and best preview URLs       | Get Icons Details           | Create CSV, Create HTML        |                                                                                                                         |
| Create CSV                 | Convert To File     | Generate CSV file from icon metadata          | Extract Tags & Name         | -                             |                                                                                                                         |
| Create HTML                | Code                | Generate HTML file with icon previews          | Extract Tags & Name         | -                             |                                                                                                                         |
| Sticky Note                | Sticky Note         | Instruction: Check User ID                    | -                           | -                             | ## 2. Check User ID<br>Instructions to obtain user_id from icon URL and test authorization token                       |
| Sticky Note1               | Sticky Note         | Instruction: Register API Key                  | -                           | -                             | ## 1. Register an Application to Obtain an API Key<br>https://www.iconfinder.com/account/applications                   |
| Sticky Note2               | Sticky Note         | Instruction: Config & Execute                  | -                           | -                             | ## 3. Config & Execute<br>Execution instructions                                                                        |
| Sticky Note3               | Sticky Note         | Workflow overview and purpose                  | -                           | -                             | ## Iconfinder Export<br>Overview of workflow purpose and usage                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: Manual start of the workflow

2. **Create a Set node for authorization**  
   - Name: "Auth Settings"  
   - Add two string fields:  
     - `authorization`: value `Bearer YOUR_TOKEN_HERE` (replace with your actual API token)  
     - `user`: value `YOUR USER ID` (replace with numeric user ID obtained from API)  
   - Connect "When clicking ‘Execute workflow’" to this node

3. **Create an HTTP Request node to get iconsets**  
   - Name: "Get Iconsets"  
   - HTTP Method: GET  
   - URL: `https://api.iconfinder.com/v4/users/{{ $json.user }}/iconsets?count=100&offset=0`  
   - Headers:  
     - Key: `Authorization`  
     - Value: `={{$json.authorization}}` (read from previous node)  
   - Connect from "Auth Settings"

4. **Create a Code node to split iconsets into paginated URLs**  
   - Name: "Split Iconsets"  
   - JavaScript code (copy logic to: iterate iconsets, calculate requests per iconset for 100 icons max, create URL list with offsets)  
   - Connect from "Get Iconsets"

5. **Create a Merge node**  
   - Name: "Merge"  
   - Mode: "Combine" with option "Combine All" (to synchronize streams)  
   - Connect branch 0 input from "Auth Settings" (authorization and user info)  
   - Connect branch 1 input from "Split Iconsets"

6. **Create an HTTP Request node to get icon details**  
   - Name: "Get Icons Details"  
   - HTTP Method: GET  
   - URL: `={{ $json.url }}` (dynamically from merged input)  
   - Headers:  
     - Key: `Authorization`  
     - Value: `={{ $json.authorization }}`  
   - Connect from "Merge"

7. **Create a Code node to extract tags and icon names**  
   - Name: "Extract Tags & Name"  
   - JavaScript code (copy logic to parse icons array, select best preview URL, normalize names, extract tags, and iconset names)  
   - Connect from "Get Icons Details"

8. **Create a Convert To File node to produce CSV**  
   - Name: "Create CSV"  
   - Input: JSON from "Extract Tags & Name"  
   - Default settings for CSV conversion  
   - Connect from "Extract Tags & Name"

9. **Create a Code node to generate HTML file**  
   - Name: "Create HTML"  
   - JavaScript code to build an HTML table with icon previews, styling, and encode as base64 file output named `icons.html`  
   - Connect from "Extract Tags & Name"

10. **Add Sticky Notes for user instructions (optional but recommended):**  
    - Register API Key instructions with link: https://www.iconfinder.com/account/applications  
    - How to check user ID by querying a known icon URL  
    - How to configure and run the workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Register an application to obtain an API Key at: https://www.iconfinder.com/account/applications                                        | Sticky Note1                                                    |
| To find your user ID, navigate to any icon URL: https://www.iconfinder.com/icons/ICON-ID/icon-name, then test with "Check User ID" node | Sticky Note                                                     |
| Workflow exports: HTML file with icon previews and CSV catalog of all user icons for easy review and reuse                             | Sticky Note3                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.