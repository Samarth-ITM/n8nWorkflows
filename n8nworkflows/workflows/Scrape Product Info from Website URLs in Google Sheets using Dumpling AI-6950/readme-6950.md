Scrape Product Info from Website URLs in Google Sheets using Dumpling AI

https://n8nworkflows.xyz/workflows/scrape-product-info-from-website-urls-in-google-sheets-using-dumpling-ai-6950


# Scrape Product Info from Website URLs in Google Sheets using Dumpling AI

### 1. Workflow Overview

This workflow automates the extraction of product information from website URLs listed in a Google Sheets document using Dumpling AI's API. It is designed for users who maintain lists of product URLs and want to enrich their data with structured product details automatically.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Monitors a Google Sheet for new rows containing website URLs.
- **1.2 AI Extraction:** Sends the detected URLs to Dumpling AI to extract structured product details.
- **1.3 Data Processing:** Splits the extracted product data into individual items if multiple products are returned.
- **1.4 Data Output:** Appends the structured product information back into another Google Sheet for further use or analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new rows added to a specified Google Sheet. Each new row is expected to contain a website URL to process.

- **Nodes Involved:**  
  - Watch New Website URL in Google Sheets

- **Node Details:**

  - **Watch New Website URL in Google Sheets**  
    - **Type & Role:** Trigger node monitoring Google Sheets for new row additions.
    - **Configuration:**  
      - Triggers on the event "rowAdded".  
      - Polls every minute to check for new rows.  
      - Connected to a specific Google Sheet document and sheet (Sheet1, gid=0) containing URLs.  
      - Uses OAuth2 credentials for Google Sheets API access.  
    - **Expressions/Variables:** None beyond static document and sheet IDs.  
    - **Input/Output:** No input; outputs new row data including the website URL in the `Website` field.  
    - **Version Requirements:** Compatible with n8n v1+.  
    - **Edge Cases:**  
      - Missing or malformed URLs in new rows.  
      - Google Sheets API authentication or permission errors.  
      - Delays if polling interval is too frequent or API rate limits are reached.  

#### 1.2 AI Extraction

- **Overview:**  
  Sends the URL obtained from the input block to Dumpling AI to extract product information such as product name, price, review, and description.

- **Nodes Involved:**  
  - Extract Product Info with Dumpling AI

- **Node Details:**

  - **Extract Product Info with Dumpling AI**  
    - **Type & Role:** HTTP Request node calling Dumpling AI's extraction API.  
    - **Configuration:**  
      - POST request to `https://app.dumplingai.com/api/v1/extract`.  
      - Sends JSON body containing the URL (`{{ $json.Website }}`) and a schema defining expected fields (`productName`, `price`, `review` (optional), `productDescription`).  
      - Authentication via HTTP Header Auth with credentials named "Dumpling AI-n8n".  
    - **Expressions/Variables:** Uses expression to inject the URL dynamically from the incoming JSON: `{{ $json.Website }}`.  
    - **Input/Output:** Input is the URL from the previous node; output is the Dumpling AI JSON response, which includes a `results` array of products.  
    - **Version Requirements:** Requires n8n version supporting HTTP Request node v4.2 or higher.  
    - **Edge Cases:**  
      - API authentication errors (invalid token or expired credentials).  
      - Timeout or network errors.  
      - Dumpling AI returning no results or malformed JSON.  
      - Schema mismatch or unexpected response structure.

#### 1.3 Data Processing

- **Overview:**  
  Splits the array of extracted products from Dumpling AI into individual items for separate processing.

- **Nodes Involved:**  
  - Split Extracted Products

- **Node Details:**

  - **Split Extracted Products**  
    - **Type & Role:** Split Out node used to iterate over array data.  
    - **Configuration:**  
      - Splits the input JSON array found in the `results` field into multiple separate outputs, each representing one product.  
    - **Expressions/Variables:** Field to split is statically set to `results`.  
    - **Input/Output:** Input is the Dumpling AI response containing multiple products; output is one item per product.  
    - **Version Requirements:** Compatible with n8n v1+.  
    - **Edge Cases:**  
      - `results` field missing or empty, resulting in zero outputs.  
      - Unexpected data types preventing splitting.

#### 1.4 Data Output

- **Overview:**  
  Appends each product's extracted details into a dedicated Google Sheet named "product details".

- **Nodes Involved:**  
  - Append Product Info to Google Sheets

- **Node Details:**

  - **Append Product Info to Google Sheets**  
    - **Type & Role:** Google Sheets node appending rows to a sheet.  
    - **Configuration:**  
      - Operation set to "append".  
      - Target sheet is identified by document ID (same as input sheet) and sheet ID `1143304403` (sheet named "product details").  
      - Columns mapped explicitly: `productName`, `price`, `productDescription` are passed from the split JSON. Note: `review` is defined in the schema but not mapped here.  
      - Does not convert fields to string or attempt type conversion; writes values as-is.  
      - Uses OAuth2 credentials for Google Sheets.  
    - **Expressions/Variables:** Fields mapped using expressions like `={{ $json.price }}`, `={{ $json.productName }}`, etc.  
    - **Input/Output:** Input from the Split Extracted Products node; output is the confirmation of the append operation.  
    - **Version Requirements:** Requires n8n Google Sheets node v4.6 or later.  
    - **Edge Cases:**  
      - Appending to a non-existent sheet or invalid sheet ID.  
      - Permission errors with Google Sheets API.  
      - Data type mismatches or missing field values.  

---

### 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                      | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                 |
|----------------------------------|----------------------------|------------------------------------|----------------------------------|--------------------------------|---------------------------------------------------------------------------------------------|
| Watch New Website URL in Google Sheets | Google Sheets Trigger      | Monitor new URL rows in Google Sheets | None                             | Extract Product Info with Dumpling AI |                                                                                             |
| Extract Product Info with Dumpling AI  | HTTP Request               | Send URL to Dumpling AI for extraction | Watch New Website URL in Google Sheets | Split Extracted Products          |                                                                                             |
| Split Extracted Products           | Split Out                  | Split multiple extracted products into individual items | Extract Product Info with Dumpling AI | Append Product Info to Google Sheets |                                                                                             |
| Append Product Info to Google Sheets    | Google Sheets              | Append extracted product data to output Google Sheet | Split Extracted Products          | None                           |                                                                                             |
| Sticky Note                      | Sticky Note                | Documentation note on workflow purpose | None                             | None                           | ### 🛍️ Scrape Product Info from Website URLs in Google Sheets using Dumpling AI<br>This workflow monitors a Google Sheet for new rows containing website URLs. When a new URL is added:<br>1. It sends the URL to **Dumpling AI** to extract product data like `productName`, `price`, and `productDescription`<br>2. If multiple products are found, it splits them into individual items<br>3. It appends the cleaned data into another sheet titled **"product details"** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node: "Watch New Website URL in Google Sheets"**  
   - Add a **Google Sheets Trigger** node.  
   - Set event to `rowAdded`.  
   - Configure polling interval to every 1 minute.  
   - Select the Google Sheets document by entering or selecting the document ID (`1yUCXVTVaFZTAx3x-5CgqwOx1syJ0Xhop9XcPZUFKojI`).  
   - Set the sheet to monitor as Sheet1 (`gid=0`).  
   - Connect OAuth2 credentials for Google Sheets (create or select an existing Google Sheets OAuth2 credential).  
   - This node will emit new rows containing URLs under the field `Website`.

2. **Add the HTTP Request Node: "Extract Product Info with Dumpling AI"**  
   - Add an **HTTP Request** node.  
   - Set the request method to `POST`.  
   - Set the URL to `https://app.dumplingai.com/api/v1/extract`.  
   - Under "Body Parameters", choose `JSON` type with the following body (use expression):  
     ```json
     {
       "url": "{{ $json.Website }}",
       "schema": {
         "productName": "string",
         "price": "string",
         "review": "string?",
         "productDescription": "string"
       }
     }
     ```  
   - Enable "Send Body" and specify body content type as JSON.  
   - Under Authentication, choose HTTP Header Auth, providing the API key or token for Dumpling AI (use credential named "Dumpling AI-n8n").  
   - Connect the output of the Google Sheets Trigger node to this node.

3. **Add a Split Out Node: "Split Extracted Products"**  
   - Add a **Split Out** node.  
   - Set the field to split out as `results` (this corresponds to the array of products returned by Dumpling AI).  
   - Connect the output of the HTTP Request node to this node.

4. **Add the Google Sheets Node: "Append Product Info to Google Sheets"**  
   - Add a **Google Sheets** node.  
   - Set operation to `append`.  
   - Select the same Google Sheets document (`1yUCXVTVaFZTAx3x-5CgqwOx1syJ0Xhop9XcPZUFKojI`).  
   - Choose the target sheet ID `1143304403` (sheet named "product details").  
   - Define columns mapping:  
     - `productName` → `={{ $json.productName }}`  
     - `price` → `={{ $json.price }}`  
     - `productDescription` → `={{ $json.productDescription }}`  
   - Do not enable type conversion or convert fields to string.  
   - Connect the output of the Split Out node to this node.  
   - Use the same Google Sheets OAuth2 credentials as in the trigger node.

5. **Add a Sticky Note (Optional)**  
   - Add a **Sticky Note** node for documentation within the workflow canvas, summarizing the workflow’s purpose and logic as described.

6. **Final Steps**  
   - Verify all nodes are connected in the order:  
     Watch New Website URL in Google Sheets → Extract Product Info with Dumpling AI → Split Extracted Products → Append Product Info to Google Sheets.  
   - Activate the workflow once all credentials and configurations are verified.  
   - Test by adding a new website URL row in the input Google Sheet and observe the product data appended in the output sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow uses Dumpling AI for extracting structured product data from URLs listed in Google Sheets. | Dumpling AI API documentation: https://app.dumplingai.com/api/v1/extract                                |
| Google Sheets OAuth2 credentials must have read/write permissions on the specified sheets.          | Google Sheets API OAuth2 setup: https://developers.google.com/sheets/api/guides/authorizing                 |
| Polling interval is set to 1 minute; adjust as needed to balance responsiveness and API quotas.     | Consider Google Sheets API quotas and Dumpling AI rate limits when scaling.                              |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.