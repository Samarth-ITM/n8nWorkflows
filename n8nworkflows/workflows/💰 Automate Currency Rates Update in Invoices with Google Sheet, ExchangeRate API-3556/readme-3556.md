💰 Automate Currency Rates Update in Invoices with Google Sheet, ExchangeRate API

https://n8nworkflows.xyz/workflows/---automate-currency-rates-update-in-invoices-with-google-sheet--exchangerate-api-3556


# 💰 Automate Currency Rates Update in Invoices with Google Sheet, ExchangeRate API

### 1. Workflow Overview

This n8n workflow automates the process of fetching live currency exchange rates from the ExchangeRate API, updating a Google Sheet with the latest rates, and archiving historical exchange rate data for record-keeping. It is designed for finance teams, analysts, and spreadsheet users who require up-to-date USD-based currency exchange rates integrated directly into their Google Sheets without coding.

The workflow is logically divided into three main blocks:

- **1.1 Trigger and API Data Retrieval**: Scheduled trigger initiates the workflow daily and calls the ExchangeRate API to fetch the latest USD-based currency rates.
- **1.2 Data Processing and Formatting**: Extracts and formats key fields from the API response, preparing the data for Google Sheets.
- **1.3 Google Sheets Update and Archiving**: Updates the main "Rate Sheet" tab with the latest exchange rates (using upsert logic) and appends the full data to an "Archives" tab for historical tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and API Data Retrieval

**Overview:**  
This block initiates the workflow on a daily schedule and performs an HTTP request to the ExchangeRate API to retrieve the latest currency exchange rates based on USD.

**Nodes Involved:**  
- Trigger - 08:00 am  
- USD Query

**Node Details:**

- **Trigger - 08:00 am**  
  - *Type & Role:* Schedule Trigger node; starts the workflow automatically every day at 08:00 UTC.  
  - *Configuration:* Set to trigger at hour 7 (UTC time zone).  
  - *Input/Output:* No input; outputs a trigger event to the next node.  
  - *Failure Modes:* Possible failure if n8n scheduler is disabled or misconfigured.  
  - *Notes:* Can be adjusted to other frequencies as needed.

- **USD Query**  
  - *Type & Role:* HTTP Request node; calls the ExchangeRate API endpoint to fetch latest USD exchange rates.  
  - *Configuration:*  
    - URL: `https://v6.exchangerate-api.com/v6/<YOUR_API_KEY>/latest/USD` (replace `<YOUR_API_KEY>` with a valid API key).  
    - Method: GET (default).  
    - No additional headers or authentication beyond the API key in URL.  
  - *Input/Output:* Receives trigger from schedule node; outputs JSON response with exchange rates.  
  - *Failure Modes:*  
    - Invalid or missing API key results in authentication errors.  
    - Network timeouts or API downtime.  
    - Rate limits imposed by ExchangeRate API.  
  - *Notes:* API key must be replaced before running; see sticky note for setup instructions.

---

#### 2.2 Data Processing and Formatting

**Overview:**  
Processes the raw API response by extracting the relevant fields and formatting the timestamp into a human-readable string.

**Nodes Involved:**  
- Format Output to JSON  
- Filter Fields  
- Final Outputs (merge node)

**Node Details:**

- **Format Output to JSON**  
  - *Type & Role:* Code node; extracts the `conversion_rates` object from the API response.  
  - *Configuration:* JavaScript code snippet:  
    ```js
    const rates = items[0].json.conversion_rates;
    return [{ json: rates }];
    ```  
  - *Input/Output:* Input is the full API JSON response; output is a simplified JSON object containing only the currency rates.  
  - *Failure Modes:*  
    - If API response structure changes or `conversion_rates` is missing, code will fail.  
  - *Notes:* Ensures only the rates data is passed downstream.

- **Filter Fields**  
  - *Type & Role:* Set node; adds two fields: `base_currency` and a formatted `time_last_update_utc` string.  
  - *Configuration:*  
    - `base_currency` set from `$json.base_code` (base currency code from API).  
    - `time_last_update_utc` formatted from the API's `time_last_update_utc` field into a readable date and time string in UTC timezone.  
  - *Input/Output:* Receives the simplified rates JSON; outputs enriched JSON with the two additional fields.  
  - *Failure Modes:*  
    - If date field is missing or malformed, formatting may fail or produce incorrect output.  
  - *Notes:* Uses JavaScript expressions for date formatting.

- **Final Outputs**  
  - *Type & Role:* Merge node; combines two inputs: one from `Filter Fields` and one directly from `Format Output to JSON`.  
  - *Configuration:* Mode set to "combineBySql" to merge data sets.  
  - *Input/Output:* Receives two streams of data; outputs a combined dataset for Google Sheets update.  
  - *Failure Modes:*  
    - Mismatched data structures could cause merge errors.  
  - *Notes:* Ensures both enriched metadata and raw rates are available downstream.

---

#### 2.3 Google Sheets Update and Archiving

**Overview:**  
Updates the main Google Sheet tab with the latest exchange rates using upsert logic and appends a full snapshot of rates to an archive tab for historical tracking.

**Nodes Involved:**  
- Update Rate Sheet  
- Archive Rates

**Node Details:**

- **Update Rate Sheet**  
  - *Type & Role:* Google Sheets node; updates existing rows or inserts new rows in the "Rate Sheet" tab based on `base_currency`.  
  - *Configuration:*  
    - Operation: Update (upsert).  
    - Document ID: User must specify the Google Sheet document ID or URL.  
    - Sheet Name: `gid=0` (main tab, typically "Rate Sheet").  
    - Matching Columns: `base_currency` (used to find existing rows).  
    - Columns: Auto-mapped from input JSON fields including `base_currency`, `time_last_update_utc`, and all currency codes (USD, AED, AFN, etc.).  
    - Attempt to convert types: Disabled (values stored as strings).  
  - *Input/Output:* Receives merged data from `Final Outputs`; outputs updated rows info.  
  - *Failure Modes:*  
    - Authentication failure if Google Sheets OAuth2 credentials are invalid.  
    - Mismatched columns or missing sheet/tab.  
    - API quota limits or network errors.  
  - *Notes:* Requires Google Sheets API OAuth2 credentials configured in n8n.

- **Archive Rates**  
  - *Type & Role:* Google Sheets node; appends the full rates data as a new row in the "Archives" tab to keep historical records.  
  - *Configuration:*  
    - Operation: Append.  
    - Document ID: Same as above (user must specify).  
    - Sheet Name: `Archives` (tab name or sheet ID 249536536).  
    - Columns: Auto-mapped similarly to "Update Rate Sheet" node (all currency codes plus metadata fields).  
    - Matching Columns: None (append only).  
  - *Input/Output:* Receives merged data from `Final Outputs`; outputs append confirmation.  
  - *Failure Modes:*  
    - Same as above: authentication, sheet/tab existence, API limits.  
  - *Notes:* Keeps a time-series record of all exchange rates for audit or analysis.

---

### 3. Summary Table

| Node Name          | Node Type             | Functional Role                         | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                                                                |
|--------------------|-----------------------|---------------------------------------|------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger - 08:00 am | Schedule Trigger      | Starts workflow daily at 08:00 UTC    | -                      | USD Query                   |                                                                                                                                                            |
| USD Query          | HTTP Request          | Fetches latest USD exchange rates     | Trigger - 08:00 am     | Format Output to JSON        | Put your API Key for exchange rate: Sign up and replace <YOUR_API_KEY> in HTTP request URL. See [API Documentation](https://www.exchangerate-api.com/docs/overview) |
| Format Output to JSON | Code                 | Extracts conversion_rates from API    | USD Query              | Final Outputs, Filter Fields |                                                                                                                                                            |
| Filter Fields      | Set                   | Adds base_currency and formatted date | Format Output to JSON   | Final Outputs               |                                                                                                                                                            |
| Final Outputs      | Merge                 | Combines enriched and raw rate data   | Filter Fields, Format Output to JSON | Update Rate Sheet, Archive Rates |                                                                                                                                                            |
| Update Rate Sheet  | Google Sheets         | Upserts latest rates into main sheet  | Final Outputs          | -                           | Update Results in Google Sheets: Use template, add credentials, select file and sheet. See [Google Sheets Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Archive Rates      | Google Sheets         | Appends rates snapshot to archive tab | Final Outputs          | -                           | Same as above                                                                                                                                              |
| Sticky Note        | Sticky Note           | Instructions for API key setup        | -                      | -                           | See above                                                                                                                                                |
| Sticky Note4       | Sticky Note           | Instructions for Google Sheets setup  | -                      | -                           | See above                                                                                                                                                |
| Sticky Note3       | Sticky Note           | Link to tutorial video                 | -                      | -                           | [🎥 Watch My Tutorial](https://www.youtube.com/watch?v=T8UFxu8Y9A)                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Trigger - 08:00 am`  
   - Set to trigger daily at 08:00 UTC (triggerAtHour: 7 in UTC).  
   - No credentials needed.

2. **Create an HTTP Request node**  
   - Name: `USD Query`  
   - Connect input from `Trigger - 08:00 am`.  
   - Set method to GET.  
   - URL: `https://v6.exchangerate-api.com/v6/<YOUR_API_KEY>/latest/USD` (replace `<YOUR_API_KEY>` with your valid ExchangeRate API key).  
   - No authentication needed beyond API key in URL.  
   - Enable retry on failure for robustness.

3. **Create a Code node**  
   - Name: `Format Output to JSON`  
   - Connect input from `USD Query`.  
   - Paste the following JavaScript code:  
     ```js
     const rates = items[0].json.conversion_rates;
     return [{ json: rates }];
     ```  
   - This extracts only the conversion rates object from the API response.

4. **Create a Set node**  
   - Name: `Filter Fields`  
   - Connect input from `Format Output to JSON`.  
   - Add two fields:  
     - `base_currency` (string): set to `={{ $json.base_code }}` (from original API response).  
     - `time_last_update_utc` (string): set to a JavaScript expression that formats the API timestamp into a readable UTC date and time string:  
       ```js
       ={{ new Date($json["time_last_update_utc"]).toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric', timeZone: 'UTC' }) + ' at ' + new Date($json["time_last_update_utc"]).toISOString().substring(11, 16) + ' UTC' }}
       ```  
   - Enable retry on failure.

5. **Create a Merge node**  
   - Name: `Final Outputs`  
   - Connect two inputs:  
     - Input 1: from `Filter Fields` (enriched data).  
     - Input 2: from `Format Output to JSON` (raw rates).  
   - Set mode to `combineBySql` to merge datasets.

6. **Create a Google Sheets node**  
   - Name: `Update Rate Sheet`  
   - Connect input from `Final Outputs`.  
   - Set operation to `update` (upsert).  
   - Configure credentials for Google Sheets API (OAuth2).  
   - Specify the Google Sheet document ID or URL.  
   - Set Sheet Name to the main tab (e.g., `gid=0` or tab name "Rate Sheet").  
   - Set matching column to `base_currency` for upsert logic.  
   - Configure columns to auto-map all currency codes plus `base_currency` and `time_last_update_utc`.  
   - Disable type conversion (store as strings).  
   - Enable retry on failure.

7. **Create another Google Sheets node**  
   - Name: `Archive Rates`  
   - Connect input from `Final Outputs`.  
   - Set operation to `append`.  
   - Use the same Google Sheets credentials and document ID.  
   - Set Sheet Name to the archive tab (e.g., tab named "Archives" or sheet ID 249536536).  
   - Auto-map all columns as above.  
   - No matching columns needed (append only).  
   - Enable retry on failure.

8. **Add Sticky Notes** (optional but recommended)  
   - Add notes to document API key setup, Google Sheets setup, and tutorial links for user guidance.

9. **Test the workflow**  
   - Run manually or wait for scheduled trigger.  
   - Verify Google Sheets tabs update correctly with latest rates and archives.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Use the publicly available Google Sheet template with tabs `Rate Sheet` and `Archives` for seamless integration.                                                                                                               | [Template Sheet](https://docs.google.com/spreadsheets/d/1SjzMb2q-6-byx9qmHbkrLseBWj9jEGduinH_5xi-c7g/edit?usp=sharing) |
| Replace `<YOUR_API_KEY>` in the HTTP Request node URL with your valid ExchangeRate API key obtained from https://www.exchangerate-api.com/.                                                                                     | [ExchangeRate API Documentation](https://www.exchangerate-api.com/docs/overview)                                 |
| Configure Google Sheets API credentials in n8n using OAuth2 to allow read/write access to your spreadsheet.                                                                                                                     | [Google Sheets Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets)          |
| You can adjust the schedule trigger frequency to run hourly, daily, or on-demand as needed.                                                                                                                                     | n8n Schedule Trigger node settings                                                                               |
| Optional: Add Slack or email notification nodes to alert when base currency rates change significantly.                                                                                                                         | Custom extension                                                                                                 |
| For detailed step-by-step instructions, watch the tutorial video by the workflow author.                                                                                                                                         | [🎥 Watch My Tutorial](https://www.youtube.com/watch?v=T8UFxu8Y9zA)                                               |
| Connect with the author for more finance automation workflows on LinkedIn.                                                                                                                                                        | [LinkedIn Profile](https://www.linkedin.com/in/samir-saci)                                                       |
| This workflow was built and tested with n8n version 1.85.4.                                                                                                                                                                      | n8n version compatibility                                                                                        |

---

This documentation provides a complete, structured reference to understand, reproduce, and maintain the "Automate Currency Rates Update in Invoices with Google Sheet, ExchangeRate API" workflow. It covers all nodes, logic blocks, configuration details, and potential failure points to ensure smooth operation and easy customization.