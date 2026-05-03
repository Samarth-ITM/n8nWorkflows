SEO Competitor Analysis & Data Logging with Semrush API and Google Sheets

https://n8nworkflows.xyz/workflows/seo-competitor-analysis---data-logging-with-semrush-api-and-google-sheets-7463


# SEO Competitor Analysis & Data Logging with Semrush API and Google Sheets

### 1. Workflow Overview

This workflow automates a comprehensive SEO competitor analysis process for a given website by integrating the Semrush API (via a RapidAPI endpoint) and Google Sheets for data logging. It is designed for SEO specialists, digital marketers, or data analysts who want to quickly retrieve competitor metrics, organic keywords, backlink data, and organic pages information, and then store these insights systematically in Google Sheets for reporting or further analysis.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures the target website URL from a user-submitted form.
- **1.2 Data Retrieval:** Sends a POST request to the competitor analysis API (Semrush via RapidAPI) to fetch various SEO data.
- **1.3 Data Reformatting:** Processes and reformats the raw API responses into structured JSON arrays suitable for Google Sheets.
- **1.4 Data Logging:** Appends the reformatted data into multiple Google Sheets documents, each dedicated to a specific SEO data category (domain overview, organic competitors, organic pages, organic keywords).

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon a website URL submission via an integrated form, initiating the competitor analysis process.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point node that listens for form submissions containing the website URL.  
    - Configuration:  
      - Form title: "Competitor Analysis"  
      - One required field labeled "website" to input the target domain.  
    - Expressions/Variables: Outputs the submitted website URL as `$json.website`.  
    - Connections: Outputs to the "Competitor Analysis" HTTP Request node.  
    - Failure modes: Form not submitted, missing required field, webhook connection errors.  
    - Version: 2.2

---

#### 1.2 Data Retrieval

- **Overview:**  
  This block sends the target website to the Semrush competitor analysis API via RapidAPI to retrieve various SEO metrics including domain overview, organic competitors, organic pages, and organic keywords.

- **Nodes Involved:**  
  - Competitor Analysis

- **Node Details:**  

  - **Competitor Analysis**  
    - Type: HTTP Request  
    - Role: Sends a POST request to the Semrush competitor analysis endpoint to fetch SEO data.  
    - Configuration:  
      - URL: `https://competitor-analysis-semrush.p.rapidapi.com/competitor.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body parameter: Website domain passed dynamically from the form submission (`{{$json.website}}`).  
      - Headers:  
        - `x-rapidapi-host`: `competitor-analysis-semrush.p.rapidapi.com`  
        - `x-rapidapi-key`: User must provide own RapidAPI key.  
    - Expressions/Variables: Uses expression to inject website from previous node.  
    - Connections: Outputs to all four reformatting code nodes simultaneously.  
    - Failure modes: API authentication errors (invalid/missing API key), network errors, incorrect domain input, API rate limits or downtime.  
    - Version: 4.2

---

#### 1.3 Data Reformatting

- **Overview:**  
  This block contains four separate code nodes that extract and restructure different sections of the API response into clean JSON arrays for insertion into Google Sheets.

- **Nodes Involved:**  
  - Re format output  
  - Reformat  
  - Reformat 2  
  - Reformat2

- **Node Details:**  

  - **Re format output**  
    - Type: Code (JavaScript)  
    - Role: Extracts `domainOverview` from the API response for domain-level metrics.  
    - Code: `return $input.first().json.data.semrushAPI.domainOverview`  
    - Input: Raw API response JSON  
    - Output: Structured object representing domain overview metrics.  
    - Connections: Outputs to "Domain overview" Google Sheets node.  
    - Failure modes: Missing or unexpected API response structure, null data.

  - **Reformat**  
    - Type: Code (JavaScript)  
    - Role: Extracts `organicCompetitors` array from the API response for competitor domains.  
    - Code: `return $input.first().json.data.semrushAPI.organicCompetitors;`  
    - Connections: Outputs to "Organic Competitor" Google Sheets node.  
    - Failure modes: Empty or malformed organic competitors data.

  - **Reformat 2**  
    - Type: Code (JavaScript)  
    - Role: Extracts `organicPages` array for details about the website’s organic pages.  
    - Code: `return $input.first().json.data.semrushAPI.organicPages;`  
    - Connections: Outputs to "Organic Pages" Google Sheets node.  
    - Failure modes: Missing or incomplete organic pages data.

  - **Reformat2**  
    - Type: Code (JavaScript)  
    - Role: Extracts `domainOrganicSearchKeywords` array for organic keyword data.  
    - Code: `return $input.first().json.data.semrushAPI.domainOrganicSearchKeywords;`  
    - Connections: Outputs to "organic keywords" Google Sheets node.  
    - Failure modes: Missing or malformed keyword data.

---

#### 1.4 Data Logging

- **Overview:**  
  These nodes append the cleaned and structured data into designated Google Sheets documents and sheets for persistent storage and reporting.

- **Nodes Involved:**  
  - Domain overview  
  - Organic Competitor  
  - Organic Pages  
  - organic keywords

- **Node Details:**  

  - **Domain overview**  
    - Type: Google Sheets  
    - Role: Appends domain overview metrics such as organic keywords and traffic to a specified sheet.  
    - Configuration:  
      - Operation: Append  
      - Sheet name: "domainOverview" (sheet ID 1271965358)  
      - Document ID: User must provide the Google Sheets document ID (referenced as "Seo n8n")  
      - Authentication: Google Service Account credentials  
      - Auto-mapping enabled to match input data keys to columns.  
    - Inputs: Reformatted domain overview data.  
    - Failure modes: Authentication errors, spreadsheet permission issues, schema mismatch, API quota limits.  
    - Version: 4.6

  - **Organic Competitor**  
    - Type: Google Sheets  
    - Role: Logs competitor domains and related SEO metrics into the “organicCompetitors” sheet.  
    - Configuration similar to Domain overview but targeting sheet ID 1979882207 ("organicCompetitors").  
    - Inputs: Reformatted organic competitors array.  
    - Failure modes: Same as above, plus potential data size limitations.

  - **Organic Pages**  
    - Type: Google Sheets  
    - Role: Stores organic pages data including URL, traffic percentage, and keyword counts.  
    - Sheet ID: 1802882998 ("organicPages")  
    - Inputs: Reformatted organic pages array.  
    - Failure modes: Same as above.

  - **organic keywords**  
    - Type: Google Sheets  
    - Role: Appends detailed organic keyword metrics including competition, CPC, volume, and trends.  
    - Sheet ID: 185010398 ("domainOrganicSearchKeywords")  
    - Inputs: Reformatted organic keywords array.  
    - Failure modes: Same as above.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                                  | Input Node(s)           | Output Node(s)                     | Sticky Note                                                                                         |
|---------------------|--------------------|-------------------------------------------------|-------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger       | Captures website URL from user form submission  | —                       | Competitor Analysis               | **On form submission**  Captures the website URL from a user submission, triggering the analysis workflow. |
| Competitor Analysis  | HTTP Request       | Fetches SEO data from Semrush API via RapidAPI  | On form submission      | Re format output, Reformat, Reformat 2, Reformat2 | **Competitor Analysis**  Fetches backlink data from the **Top Backlink Checker API** and other competitor analysis information via a POST request. |
| Re format output     | Code               | Extracts domain overview data                     | Competitor Analysis      | Domain overview                  | **Reformat output**  Processes and reformats the domain overview data retrieved from the **Semrush API** for further analysis. |
| Domain overview      | Google Sheets      | Logs domain overview metrics into Google Sheet   | Re format output         | —                                | **Domain overview**  Appends the domain overview data, such as organic traffic and keywords, into the Google Sheets document for analysis. |
| Reformat             | Code               | Extracts organic competitors array                | Competitor Analysis      | Organic Competitor               | **Reformat**  Reformats the list of organic competitors from the **Semrush API** for storing in Google Sheets. |
| Organic Competitor   | Google Sheets      | Logs organic competitor data                       | Reformat                 | —                                | **Organic Competitor**  Logs competitor information such as keywords, relevance, and organic traffic into the appropriate Google Sheets. |
| Reformat 2           | Code               | Extracts organic pages data                        | Competitor Analysis      | Organic Pages                   | **Reformat 2**  Reformats organic pages' data retrieved from **Semrush API** for detailed analysis. |
| Organic Pages        | Google Sheets      | Logs organic pages data                            | Reformat 2               | —                                | **Organic Pages**  Stores the re-formatted organic pages data, including traffic percentage and keyword counts, into Google Sheets. |
| Reformat2            | Code               | Extracts organic keywords array                    | Competitor Analysis      | organic keywords                | **Reformat2**  Prepares organic keyword data for the next stage of processing. |
| organic keywords     | Google Sheets      | Logs detailed organic keyword metrics             | Reformat2                | —                                | **organic keywords**  Logs the organic keywords data, including competition, CPC, and search volume, into the Google Sheets document. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Form Trigger** node named "On form submission".
   - Configure form title as "Competitor Analysis".
   - Add one required field labeled "website".
   - This node listens for form submissions to start the workflow.

2. **Add HTTP Request Node for Data Retrieval:**
   - Add an **HTTP Request** node named "Competitor Analysis".
   - Set method to POST.
   - Set URL to `https://competitor-analysis-semrush.p.rapidapi.com/competitor.php`.
   - Under Body Parameters, add one parameter:
     - Name: "website"
     - Value: Use expression to reference `{{$json.website}}` from the trigger.
   - Under Headers, add:
     - `x-rapidapi-host`: `competitor-analysis-semrush.p.rapidapi.com`
     - `x-rapidapi-key`: Enter your RapidAPI key.
   - Set content type to multipart/form-data.
   - Connect "On form submission" node output to this node's input.

3. **Add Four Code Nodes to Reformat API Response:**

   - **Re format output**  
     - Add a **Code** node named "Re format output".  
     - Code: `return $input.first().json.data.semrushAPI.domainOverview`  
     - Connect output of "Competitor Analysis" to this node.

   - **Reformat**  
     - Add a **Code** node named "Reformat".  
     - Code: `return $input.first().json.data.semrushAPI.organicCompetitors;`  
     - Connect output of "Competitor Analysis" to this node.

   - **Reformat 2**  
     - Add a **Code** node named "Reformat 2".  
     - Code: `return $input.first().json.data.semrushAPI.organicPages;`  
     - Connect output of "Competitor Analysis" to this node.

   - **Reformat2**  
     - Add a **Code** node named "Reformat2".  
     - Code: `return $input.first().json.data.semrushAPI.domainOrganicSearchKeywords;`  
     - Connect output of "Competitor Analysis" to this node.

4. **Set Up Google Sheets Nodes for Data Logging:**

   - For all Google Sheets nodes below, configure authentication using a Google Service Account credential with appropriate API access, and specify the Google Sheets document ID (referred to as "Seo n8n" in the original workflow). Ensure the Google Sheets document exists with sheets having the specified IDs or names.

   - **Domain overview**  
     - Add a **Google Sheets** node named "Domain overview".  
     - Set operation to Append.  
     - Set document ID to your SEO tracking Google Sheet.  
     - Set sheet name to "domainOverview" (or the corresponding sheet ID 1271965358).  
     - Enable auto-mapping of input data to columns.  
     - Connect input from "Re format output".

   - **Organic Competitor**  
     - Add a **Google Sheets** node named "Organic Competitor".  
     - Operation: Append.  
     - Sheet name: "organicCompetitors" (sheet ID 1979882207).  
     - Connect input from "Reformat".

   - **Organic Pages**  
     - Add a **Google Sheets** node named "Organic Pages".  
     - Operation: Append.  
     - Sheet name: "organicPages" (sheet ID 1802882998).  
     - Connect input from "Reformat 2".

   - **organic keywords**  
     - Add a **Google Sheets** node named "organic keywords".  
     - Operation: Append.  
     - Sheet name: "domainOrganicSearchKeywords" (sheet ID 185010398).  
     - Connect input from "Reformat2".

5. **Test the Workflow:**
   - Submit a test website URL via the form to trigger the workflow.
   - Verify each Google Sheet is receiving and appending data correctly.
   - Handle authentication errors or data format issues if they arise.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                    | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow uses the Semrush API accessed via RapidAPI; users must obtain and configure their own RapidAPI key for authentication.                                                          | Semrush API on RapidAPI: https://rapidapi.com/                                                        |
| Google Sheets nodes require a **Google Service Account** credential with edit access to the target spreadsheet, and the Google Sheets document must have the appropriate sheet tabs created.   | Google Sheets API credentials setup: https://developers.google.com/sheets/api/guides/authorizing |
| The workflow records multiple SEO metrics for competitor analysis, including backlinks, organic keywords, competitor domains, and organic pages for holistic SEO insight and tracking.          | SEO analysis best practices: https://moz.com/beginners-guide-to-seo/competitive-analysis          |
| All nodes have sticky notes explaining their purpose and functionality to assist in future modifications or debugging.                                                                           | Sticky notes are embedded within the workflow for user guidance.                                  |

---

**Disclaimer:** The text provided above is derived exclusively from an automated workflow created with n8n, respecting all applicable content policies and handling only legal and public data.