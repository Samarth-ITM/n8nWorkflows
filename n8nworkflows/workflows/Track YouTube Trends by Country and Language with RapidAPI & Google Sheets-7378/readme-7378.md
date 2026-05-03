Track YouTube Trends by Country and Language with RapidAPI & Google Sheets

https://n8nworkflows.xyz/workflows/track-youtube-trends-by-country-and-language-with-rapidapi---google-sheets-7378


# Track YouTube Trends by Country and Language with RapidAPI & Google Sheets

### 1. Workflow Overview

This workflow, titled **"Track YouTube Trends by Country and Language with RapidAPI & Google Sheets"**, is designed to fetch trending YouTube video data based on user-selected country and language inputs. It then logs the retrieved trending video details into a Google Sheet for tracking and analysis.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Collect user input for country and language via a web form.
- **1.2 API Data Retrieval:** Use the collected input to request trending YouTube video data from a RapidAPI service.
- **1.3 Data Reformatting:** Process and extract relevant video details from the API response.
- **1.4 Data Logging:** Append the extracted video details into a Google Sheet for persistent storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by presenting a form to the user to collect the target country and language. Submission of this form triggers the entire workflow.

- **Nodes Involved:**  
  - `On form submission`

- **Node Details:**

  - **Node:** On form submission  
    - **Type & Role:** Form Trigger — Listens for a user submission of a predefined form.  
    - **Configuration:**  
      - Form titled "YouTube Trend Finder"  
      - Two required fields: `country` (string), `language` (string)  
      - Triggers the workflow immediately upon form submission.  
    - **Expressions/Variables:** The form input values for `country` and `language` become accessible as `$json.country` and `$json.language`.  
    - **Input/Output Connections:**  
      - No input nodes (trigger node).  
      - Output connects to `Trend Finder Api Request`.  
    - **Edge Cases / Failure Modes:**  
      - Missing required fields (handled by form validation).  
      - Network or webhook failures preventing trigger reception.  
    - **Sub-workflow:** None.

---

#### 2.2 API Data Retrieval

- **Overview:**  
  Sends a POST request to the YouTube Trend Finder API via RapidAPI, passing the user-provided country and language as form parameters. This fetches trending video data specific to the selected locale.

- **Nodes Involved:**  
  - `Trend Finder Api Request`

- **Node Details:**

  - **Node:** Trend Finder Api Request  
    - **Type & Role:** HTTP Request — Makes an external API call.  
    - **Configuration:**  
      - Method: POST  
      - URL: `https://youtube-trend-finder.p.rapidapi.com/trend.php`  
      - Content Type: multipart/form-data  
      - Body Parameters: `country` and `language` from form input.  
      - Headers:  
        - `x-rapidapi-host: youtube-trend-finder.p.rapidapi.com`  
        - `x-rapidapi-key: your key` (user must provide valid RapidAPI key)  
    - **Expressions/Variables:** Uses `={{ $json.country }}` and `={{ $json.language }}` to dynamically insert values.  
    - **Input/Output Connections:**  
      - Input: from `On form submission`  
      - Output: to `Re format output`  
    - **Version Specifics:** Uses n8n HTTP Request node version 4.2, ensure compatibility.  
    - **Edge Cases / Failure Modes:**  
      - API key invalid or missing → authentication error.  
      - API rate limits or downtime → HTTP errors/timeouts.  
      - Invalid country/language input → API returns empty or error responses.  
      - Network connectivity issues.  
    - **Sub-workflow:** None.

---

#### 2.3 Data Reformatting

- **Overview:**  
  Processes the raw API response to extract the relevant details (thumbnail, title, link, tags) and prepares a simplified JSON array that can be ingested by Google Sheets.

- **Nodes Involved:**  
  - `Re format output`

- **Node Details:**

  - **Node:** Re format output  
    - **Type & Role:** Code (JavaScript) — Executes custom JavaScript to manipulate data.  
    - **Configuration:**  
      - Script returns `$input.first().json.data` — extracts the `data` field from the first input item’s JSON payload.  
    - **Expressions/Variables:** Accesses API response via `$input.first().json`.  
    - **Input/Output Connections:**  
      - Input: from `Trend Finder Api Request`  
      - Output: to `Google Sheets` node  
    - **Edge Cases / Failure Modes:**  
      - API response structure changes → script fails or returns undefined.  
      - Empty or malformed data → downstream nodes may receive invalid input.  
    - **Sub-workflow:** None.

---

#### 2.4 Data Logging

- **Overview:**  
  Appends the reformatted trending video data (thumbnail URL, title, video link, tags) as new rows into a specified Google Sheet document for record-keeping.

- **Nodes Involved:**  
  - `Google Sheets`

- **Node Details:**

  - **Node:** Google Sheets  
    - **Type & Role:** Google Sheets (Append) — Appends new rows to an existing sheet.  
    - **Configuration:**  
      - Operation: Append  
      - Document ID: `1GtSSz6EiZBDdRUfjlkN3VOsbHGsjDz2jRqiZKSBwRyc` (preconfigured target spreadsheet)  
      - Sheet Name: `Sheet1` (gid=0)  
      - Columns mapped automatically for fields: `thumbnail`, `title`, `link`, `tags`  
      - Authentication: Service Account credentials linked (`Google Docs account`)  
      - Mapping Mode: Auto map input data fields to columns  
    - **Expressions/Variables:** Automatically maps incoming JSON fields to the sheet columns.  
    - **Input/Output Connections:**  
      - Input: from `Re format output`  
      - Output: None (terminal node)  
    - **Edge Cases / Failure Modes:**  
      - Authentication failure → invalid or expired service account credentials  
      - Sheet not found or permission denied → API errors  
      - Data type mismatches or invalid cell data → possible append errors  
      - API quota limits or Google Sheets downtime  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                             | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                             |
|------------------------|----------------------|---------------------------------------------|-------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger          | Collect user input (country, language)      | -                       | Trend Finder Api Request | ## 1. **On form submission** (`Form Trigger`) - Presents a web form with required fields: `country` and `language`.       |
| Trend Finder Api Request | HTTP Request         | Sends API request to YouTube Trend Finder   | On form submission       | Re format output         | ## 2. **Trend Finder API Request** (`HTTP Request`) - Sends a `POST` request to RapidAPI with form inputs.               |
| Re format output       | Code                  | Extracts and reformats API response data    | Trend Finder Api Request | Google Sheets            | ## 3. **Re format output** (`Code`) - Extracts relevant data and prepares simplified JSON for the sheet.                |
| Google Sheets          | Google Sheets (Append)| Appends trending video data to Google Sheet | Re format output         | -                       | ## 4. **Google Sheets** (`Append to Sheet`) - Appends video details (title, link, tags, thumbnail) to the spreadsheet.  |
| Sticky Note            | Sticky Note           | Documentation and comments                   | -                       | -                       | # 🧠 YouTube Trend Finder – Workflow Summary (Overview and nodes description)                                           |
| Sticky Note1           | Sticky Note           | Comments on form trigger node                | -                       | -                       | ## 1. **On form submission** (`Form Trigger`) - Presents form and triggers workflow on submission.                       |
| Sticky Note2           | Sticky Note           | Comments on API request node                  | -                       | -                       | ## 2. **Trend Finder API Request** (`HTTP Request`) - Sends POST request with form data.                                |
| Sticky Note3           | Sticky Note           | Comments on code node                         | -                       | -                       | ## 3. **Re format output** (`Code`) - Extracts relevant video data.                                                     |
| Sticky Note4           | Sticky Note           | Comments on Google Sheets append node        | -                       | -                       | ## 4. **Google Sheets** (`Append to Sheet`) - Appends formatted data to Google Sheet.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Add a **Form Trigger** node named `On form submission`.  
   - Set form title: `"YouTube Trend Finder"`.  
   - Add two required form fields:  
     - `country` (text input, required)  
     - `language` (text input, required)  
   - Save and activate the webhook to generate the trigger URL.

2. **Create HTTP Request Node**  
   - Add an **HTTP Request** node named `Trend Finder Api Request`.  
   - Set HTTP Method: `POST`.  
   - Set URL: `https://youtube-trend-finder.p.rapidapi.com/trend.php`.  
   - Under "Body Parameters" choose `multipart/form-data`. Add parameters:  
     - `country` with value expression: `={{ $json.country }}`  
     - `language` with value expression: `={{ $json.language }}`  
   - Add Headers:  
     - `x-rapidapi-host`: `youtube-trend-finder.p.rapidapi.com`  
     - `x-rapidapi-key`: Your valid RapidAPI key (replace `"your key"`).  
   - Connect output from `On form submission` to this node.

3. **Create Code Node for Output Reformatting**  
   - Add a **Code** node named `Re format output`.  
   - Set language: JavaScript.  
   - Insert this code snippet:  
     ```javascript
     return $input.first().json.data;
     ```  
   - Connect output from `Trend Finder Api Request` to this node.

4. **Create Google Sheets Append Node**  
   - Add a **Google Sheets** node named `Google Sheets`.  
   - Set operation: `Append`.  
   - Set Document ID: `1GtSSz6EiZBDdRUfjlkN3VOsbHGsjDz2jRqiZKSBwRyc` (or your target spreadsheet ID).  
   - Set Sheet Name: `Sheet1` (gid=0).  
   - Set columns with fields: `thumbnail`, `title`, `link`, `tags`. Use automatic mapping mode.  
   - Configure authentication with a **Google Service Account** credential that has write access to the target spreadsheet.  
   - Connect output from `Re format output` to this node.

5. **Save and Activate the Workflow**  
   - Test the form by submitting country and language values.  
   - Verify trending videos are appended to the Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow uses the RapidAPI YouTube Trend Finder API; ensure you have a valid API key and quota for usage.                        | https://rapidapi.com/                                            |
| Google Sheets authentication is via a service account; the service account must have edit permissions on the target spreadsheet.    | Google Cloud Console & Google Sheets API documentation          |
| The form trigger node allows for easy user input collection and can be embedded or shared for external submissions.                  | n8n Form Trigger node documentation                             |
| This workflow can be adapted for other video platforms or extended with additional filtering, error handling, or notification nodes. | n8n community forums and workflows repository                    |

---

_Disclaimer_: The provided text is generated from an automated n8n workflow. It complies fully with content policies and contains no illegal or protected data. All data handled is legal and publicly accessible.