Scrape Business Data from Google Maps via SerpAPI and Export to Sheets & Excel

https://n8nworkflows.xyz/workflows/scrape-business-data-from-google-maps-via-serpapi-and-export-to-sheets---excel-7541


# Scrape Business Data from Google Maps via SerpAPI and Export to Sheets & Excel

---

### 1. Workflow Overview

This workflow is designed to scrape business data from Google Maps using the SerpAPI service and export the processed data into Google Sheets and Excel formats. It is targeted at users who need to automate the extraction of structured location-based business information and maintain it in spreadsheet forms for further analysis or reporting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Receives chat-triggered input, sets up initial variables and prepares the environment.
- **1.2 Data Extraction from SerpAPI:** Calls SerpAPI to retrieve Google Maps data, then splits and extracts relevant details.
- **1.3 Data Preparation and Transformation:** Formats and prepares the extracted data for export.
- **1.4 Data Export:** Converts data into XLSX format and upserts it into Google Sheets for persistent storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block listens for an incoming chat message to trigger the process, sets up initial variables, and prepares the environment for the data extraction.

- **Nodes Involved:**  
  - When chat message received  
  - Variables Chat  
  - Initialization

- **Node Details:**  

  - **When chat message received**  
    - *Type:* Chat Trigger (Langchain integration)  
    - *Role:* Entry point; listens for chat messages to start the workflow  
    - *Configuration:* Uses a webhook with ID `182139f3-e81b-4d8a-91cd-4ee7a5474f88`  
    - *Inputs:* External chat messages  
    - *Outputs:* Passes data to Variables Chat node  
    - *Failures:* Possible webhook connection issues or chat integration errors  

  - **Variables Chat**  
    - *Type:* Set node  
    - *Role:* Initializes or sets workflow variables based on chat input  
    - *Configuration:* Likely sets variables needed for subsequent API calls or data processing  
    - *Inputs:* From the chat trigger  
    - *Outputs:* Passes data to Initialization node  
    - *Failures:* Expression or variable setting errors if input format unexpected  

  - **Initialization**  
    - *Type:* Set node  
    - *Role:* Prepares any additional environment parameters, such as API keys or request options  
    - *Configuration:* Sets static or dynamic parameters required for SerpAPI call  
    - *Inputs:* From Variables Chat  
    - *Outputs:* Passes data to Extract SerpAPI Map node  
    - *Failures:* Misconfiguration of parameters or missing credentials  

---

#### 2.2 Data Extraction from SerpAPI

- **Overview:**  
  This block performs the actual scraping of Google Maps business data via SerpAPI and organizes the response data for further processing.

- **Nodes Involved:**  
  - Extract SerpAPI Map  
  - Split Out SerpAPI  
  - Extract Data  

- **Node Details:**  

  - **Extract SerpAPI Map**  
    - *Type:* SerpAPI node  
    - *Role:* Executes a request to SerpAPI to obtain Google Maps data based on parameters set upstream  
    - *Configuration:* Uses credentials for SerpAPI; queries Google Maps data (e.g., business listings)  
    - *Inputs:* From Initialization  
    - *Outputs:* Raw API response to Split Out SerpAPI  
    - *Failures:* API authentication errors, rate limiting, request timeouts, malformed queries  

  - **Split Out SerpAPI**  
    - *Type:* Split Out node  
    - *Role:* Splits the JSON array response from SerpAPI into individual items for granular processing  
    - *Configuration:* Defaults to splitting each element of the array  
    - *Inputs:* From Extract SerpAPI Map  
    - *Outputs:* Each item passed individually to Extract Data node  
    - *Failures:* Input data not in expected array format, empty results  

  - **Extract Data**  
    - *Type:* Set node  
    - *Role:* Extracts and sets specific fields from each individual business data item for further use  
    - *Configuration:* Uses expressions to select fields like name, address, phone, etc. from each split item  
    - *Inputs:* From Split Out SerpAPI  
    - *Outputs:* Passes structured data to Prepare Data node  
    - *Failures:* Missing fields in data, expression evaluation errors  

---

#### 2.3 Data Preparation and Transformation

- **Overview:**  
  This block formats the extracted data into a consistent structure and prepares it for export to spreadsheet formats.

- **Nodes Involved:**  
  - Prepare Data

- **Node Details:**  

  - **Prepare Data**  
    - *Type:* Set node  
    - *Role:* Finalizes data formatting, possibly adding or transforming fields to match spreadsheet schema  
    - *Configuration:* Map fields, set default values, clean or format strings/numbers as needed  
    - *Inputs:* From Extract Data  
    - *Outputs:* Passes data to Get Data in XLSX and Upsert Data in Sheets nodes  
    - *Failures:* Data inconsistency, format mismatches  

---

#### 2.4 Data Export

- **Overview:**  
  Converts prepared data into an Excel file and updates a Google Sheet with the new data, enabling easy access and analysis.

- **Nodes Involved:**  
  - Get Data in XLSX  
  - Upsert Data in Sheets

- **Node Details:**  

  - **Get Data in XLSX**  
    - *Type:* Convert to File node  
    - *Role:* Converts the JSON data into an XLSX file format  
    - *Configuration:* Default conversion parameters; likely outputs a binary Excel file  
    - *Inputs:* From Prepare Data  
    - *Outputs:* Exported file for download or further processing  
    - *Failures:* Large data size causing memory issues, unsupported data types  

  - **Upsert Data in Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Inserts or updates rows in a Google Sheets spreadsheet with the prepared data  
    - *Configuration:* Configured with Google OAuth2 credentials, target spreadsheet ID, and worksheet name  
    - *Inputs:* From Prepare Data  
    - *Outputs:* Final step; confirmation of data upsert  
    - *Failures:* Authentication errors, quota limits, invalid spreadsheet ID or permissions  

---

### 3. Summary Table

| Node Name             | Node Type                 | Functional Role                      | Input Node(s)             | Output Node(s)          | Sticky Note             |
|-----------------------|---------------------------|------------------------------------|---------------------------|-------------------------|-------------------------|
| When chat message received | Chat Trigger (Langchain)  | Entry point; receives chat trigger | —                         | Variables Chat          |                         |
| Variables Chat        | Set                       | Sets initial variables from input  | When chat message received | Initialization          |                         |
| Initialization        | Set                       | Prepares API parameters             | Variables Chat            | Extract SerpAPI Map     |                         |
| Extract SerpAPI Map   | SerpAPI                   | Calls SerpAPI to get Google Maps data | Initialization            | Split Out SerpAPI       |                         |
| Split Out SerpAPI     | Split Out                 | Splits API response array           | Extract SerpAPI Map       | Extract Data            |                         |
| Extract Data          | Set                       | Extracts business fields            | Split Out SerpAPI         | Prepare Data            |                         |
| Prepare Data          | Set                       | Formats data for export             | Extract Data              | Get Data in XLSX, Upsert Data in Sheets |                         |
| Get Data in XLSX      | Convert to File           | Converts data to Excel file         | Prepare Data              | —                       |                         |
| Upsert Data in Sheets | Google Sheets             | Inserts/updates data in Google Sheets | Prepare Data              | —                       |                         |
| Sticky Note1          | Sticky Note               | —                                  | —                         | —                       |                         |
| Sticky Note2          | Sticky Note               | —                                  | —                         | —                       |                         |
| Sticky Note4          | Sticky Note               | —                                  | —                         | —                       |                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Chat Trigger (Langchain)** node named "When chat message received."  
   - Configure webhook ID or create a new webhook for triggering.  

2. **Set Initial Variables:**  
   - Add a **Set** node named "Variables Chat."  
   - Configure to set variables needed for extraction based on chat input (e.g., search queries).  
   - Connect "When chat message received" → "Variables Chat."  

3. **Prepare Initialization Parameters:**  
   - Add a **Set** node named "Initialization."  
   - Define API parameters such as SerpAPI key, search parameters, location, etc.  
   - Connect "Variables Chat" → "Initialization."  

4. **Add SerpAPI Node:**  
   - Add a **SerpAPI** node named "Extract SerpAPI Map."  
   - Configure with SerpAPI credentials (API key).  
   - Set the API request to query Google Maps data (e.g., business listings).  
   - Connect "Initialization" → "Extract SerpAPI Map."  

5. **Split API Response:**  
   - Add a **Split Out** node named "Split Out SerpAPI."  
   - Configure to split the JSON array response from SerpAPI into individual items.  
   - Connect "Extract SerpAPI Map" → "Split Out SerpAPI."  

6. **Extract Fields from Each Item:**  
   - Add a **Set** node named "Extract Data."  
   - Use expressions to extract business name, address, phone, website, ratings, etc., from each split item.  
   - Connect "Split Out SerpAPI" → "Extract Data."  

7. **Prepare Data for Export:**  
   - Add a **Set** node named "Prepare Data."  
   - Format data fields, add missing fields if needed, and prepare consistent schema for export.  
   - Connect "Extract Data" → "Prepare Data."  

8. **Convert Data to XLSX:**  
   - Add a **Convert to File** node named "Get Data in XLSX."  
   - Configure to convert JSON data into XLSX format.  
   - Connect "Prepare Data" → "Get Data in XLSX."  

9. **Upsert Data into Google Sheets:**  
   - Add a **Google Sheets** node named "Upsert Data in Sheets."  
   - Configure with Google OAuth2 credentials.  
   - Set spreadsheet ID and worksheet name.  
   - Configure operation as 'Upsert' or 'Append' rows based on business logic.  
   - Connect "Prepare Data" → "Upsert Data in Sheets."  

10. **Test Workflow:**  
    - Trigger the webhook with a valid chat message containing necessary parameters.  
    - Confirm data is fetched, processed, converted, and exported correctly.  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                       |
|------------------------------------------------------------------------------|-------------------------------------|
| The workflow relies on valid SerpAPI credentials. Obtain from https://serpapi.com | SerpAPI official site                |
| Google Sheets node requires OAuth2 credentials with spreadsheet access.       | Google Cloud Console for API keys   |
| Chat Trigger uses Langchain integration; ensure proper webhook setup.         | n8n Langchain docs                  |
| Conversion to XLSX may fail on very large datasets due to memory limits.      | n8n node "Convert to File" docs     |
| No sticky notes contain content or comments; consider adding notes for clarity if modifying the workflow. | Internal workflow maintenance       |

---

*Disclaimer:* The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.