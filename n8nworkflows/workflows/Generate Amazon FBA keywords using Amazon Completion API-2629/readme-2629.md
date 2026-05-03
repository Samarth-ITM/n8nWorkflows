Generate Amazon FBA keywords using Amazon Completion API

https://n8nworkflows.xyz/workflows/generate-amazon-fba-keywords-using-amazon-completion-api-2629


# Generate Amazon FBA keywords using Amazon Completion API

### 1. Workflow Overview

This workflow automates the generation of Amazon FBA keywords by leveraging the Amazon Completion API and Airtable integration. It is designed to help Amazon sellers discover relevant keywords for advertising their products effectively. The workflow performs the following logical blocks:

- **1.1 Input Reception:** Receives the initial keyword input via webhook.
- **1.2 Retrieve Existing Data:** Fetches existing keyword data from Airtable based on the input.
- **1.3 Query Amazon Completion API:** Sends the keyword to Amazon's completion API to get related keyword suggestions.
- **1.4 Process API Response:** Extracts, cleans, aggregates, and formats the returned keywords.
- **1.5 Store Results:** Updates the Airtable record with the generated keywords for future use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block handles receiving the initial keyword input triggered externally, typically from Airtable or other sources via webhook.

- **Nodes Involved:**  
  - Receive Keyword

- **Node Details:**  
  - **Receive Keyword**  
    - *Type & Role:* Webhook node; entry point receiving keyword input.  
    - *Configuration:* Path set to a unique webhook ID allowing secure external HTTP POST requests.  
    - *Key Expressions:* None (payload expected in incoming request).  
    - *Connections:* Output connected to "Get airtable data".  
    - *Edge Cases:* Missing or malformed payload could cause empty or invalid processing. Webhook URL must be publicly accessible.  
    - *Version:* 1.1.

---

#### 2.2 Retrieve Existing Data

- **Overview:**  
Queries Airtable to retrieve record data for the provided keyword ID, ensuring the workflow works with the relevant record.

- **Nodes Involved:**  
  - Get airtable data

- **Node Details:**  
  - **Get airtable data**  
    - *Type & Role:* Airtable node; fetches data from a specific Airtable base and table.  
    - *Configuration:* Uses a base and table configured for Amazon keywords. The record ID is dynamically assigned from the webhook input’s query parameter `q`.  
    - *Credentials:* Airtable Personal Access Token required.  
    - *Input Connections:* From "Receive Keyword".  
    - *Output Connections:* To "Get Amazon keywords".  
    - *Edge Cases:* Invalid or missing record ID in query parameter causes failure or no data returned. API rate limits or token expiration could cause errors.  
    - *Version:* 2.1.

---

#### 2.3 Query Amazon Completion API

- **Overview:**  
Sends the keyword fetched from Airtable to Amazon's Completion API to retrieve keyword suggestions related to the original product keyword.

- **Nodes Involved:**  
  - Get Amazon keywords

- **Node Details:**  
  - **Get Amazon keywords**  
    - *Type & Role:* HTTP Request node; calls Amazon’s keyword completion API.  
    - *Configuration:* URL dynamically constructed using the field `$json.Keyword` from Airtable data, with parameters for marketplace and alias fixed (`ATVPDKIKX0DER` and `aps`).  
    - *Input:* Receives Airtable record JSON with the keyword.  
    - *Output:* JSON response containing keyword suggestions.  
    - *Edge Cases:* API downtime, network errors, rate limiting, or unexpected response formats can cause failure. Unauthenticated or malformed requests are rejected.  
    - *Version:* 4.1.

---

#### 2.4 Process API Response

- **Overview:**  
Extracts the keyword suggestions from the API response, cleans and aggregates them into a single string suitable for storage.

- **Nodes Involved:**  
  - Format output  
  - Clean Keywords  
  - Aggregate keywords  
  - Combine into string

- **Node Details:**  
  - **Format output**  
    - *Type & Role:* Split Out node; splits the `suggestions` array from the API response to process each suggestion individually.  
    - *Input:* API response JSON with `suggestions` array.  
    - *Output:* Individual keyword suggestion items.  
    - *Edge Cases:* Empty or missing `suggestions` field could result in zero outputs.  
    - *Version:* 1.
  
  - **Clean Keywords**  
    - *Type & Role:* Set node; assigns extracted suggestions to a new array field `Keywords`.  
    - *Configuration:* Converts incoming suggestion values into an array assigned to `Keywords`.  
    - *Input:* From "Format output".  
    - *Output:* JSON with `Keywords` array.  
    - *Edge Cases:* Conversion errors are ignored due to option set.  
    - *Version:* 3.3.
  
  - **Aggregate keywords**  
    - *Type & Role:* Aggregate node; combines all keyword arrays into a single aggregated array.  
    - *Input:* Receives multiple keyword arrays from "Clean Keywords".  
    - *Output:* Single aggregated array of keywords.  
    - *Edge Cases:* If no keywords exist, output could be empty array.  
    - *Version:* 1.
  
  - **Combine into string**  
    - *Type & Role:* Code node; converts aggregated keyword array into a comma-separated string for Airtable storage.  
    - *Configuration:* JavaScript code joins the keywords with a comma and space separator.  
    - *Input:* Aggregated keyword array.  
    - *Output:* JSON with `keywords` string field.  
    - *Edge Cases:* Empty arrays produce empty string.  
    - *Version:* 2.

---

#### 2.5 Store Results

- **Overview:**  
Updates the original Airtable record with the newly generated keywords string in the designated field.

- **Nodes Involved:**  
  - Save keywords

- **Node Details:**  
  - **Save keywords**  
    - *Type & Role:* Airtable node; updates the existing record with new keyword data.  
    - *Configuration:* Uses the same base and table as the retrieval node. Matches record by ID from the original Airtable data. Updates the `Keyword output` field with the combined keyword string.  
    - *Credentials:* Airtable Personal Access Token.  
    - *Input:* Receives JSON with `keywords` string and original record ID.  
    - *Edge Cases:* Record ID mismatch or loss of connection can cause update failure. Airtable API limits or token expiration must be considered.  
    - *Version:* 2.1.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                  | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                                   |
|--------------------|---------------------|--------------------------------|---------------------|------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note        | Sticky Note         | Informational note              |                     |                        | ## How to build your own Amazon keywords tool with n8n (For free and no coding) [💡 Read more](https://rumjahn.com/how-to-build-your-own-amazon-keywords-tool-with-n8n-for-free-and-no-coding/) |
| Sticky Note1       | Sticky Note         | Informational note              |                     |                        | ## Send keywords You need to send the workflow a keyword through webhook. See airtable example. [Download airtable here.](https://airtable.com/invite/l?inviteId=invgv9FzNB258bm5Z&inviteToken=6f820e142d3324318254c1768fa57809b3ef0bcb7212ea27730fd2d140c69ad5&utm_medium=email&utm_source=product_team&utm_content=transactional-alerts) |
| Sticky Note2       | Sticky Note         | Informational note              |                     |                        | ## Send to Amazon Amazon has a completion API that gives you keyword data.                                   |
| Sticky Note3       | Sticky Note         | Informational note              |                     |                        | ## Save keywords Download airtable example to save keywords. [Download airtable here.](https://airtable.com/invite/l?inviteId=invgv9FzNB258bm5Z&inviteToken=6f820e142d3324318254c1768fa57809b3ef0bcb7212ea27730fd2d140c69ad5&utm_medium=email&utm_source=product_team&utm_content=transactional-alerts) |
| Receive Keyword    | Webhook             | Input Reception                |                     | Get airtable data      |                                                                                                              |
| Get airtable data  | Airtable             | Retrieve existing keyword data | Receive Keyword      | Get Amazon keywords    |                                                                                                              |
| Get Amazon keywords| HTTP Request         | Query Amazon Completion API    | Get airtable data    | Format output          |                                                                                                              |
| Format output      | Split Out            | Extract suggestions array      | Get Amazon keywords  | Clean Keywords         |                                                                                                              |
| Clean Keywords     | Set                  | Assign suggestions to Keywords | Format output        | Aggregate keywords     |                                                                                                              |
| Aggregate keywords | Aggregate            | Combine all keywords           | Clean Keywords       | Combine into string    |                                                                                                              |
| Combine into string| Code                 | Create keywords string         | Aggregate keywords   | Save keywords          |                                                                                                              |
| Save keywords      | Airtable             | Update Airtable record         | Combine into string  |                        |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Receive Keyword`  
   - Type: Webhook  
   - Configure path to a unique endpoint (e.g., `e1df17af-e8b8-4261-ba45-aba7106c65bd`)  
   - Response Mode: `Last Node`  
   - This node will accept external HTTP POST requests containing the keyword ID (expected in query parameter `q`).

2. **Create Airtable Node to Retrieve Data:**  
   - Name: `Get airtable data`  
   - Type: Airtable  
   - Credentials: Set up Airtable Personal Access Token with access to your base and table  
   - Base: Select your Airtable base (e.g., `appGZ14ny5J2PYbq8`)  
   - Table: Select your table (e.g., `tblvK8Nq4Jqb2Ubun`)  
   - Operation: `Get Record` by ID  
   - Record ID: Set expression to `{{$json.query.q}}` to use the query parameter from webhook  
   - Connect output of `Receive Keyword` to this node.

3. **Create HTTP Request Node to Call Amazon API:**  
   - Name: `Get Amazon keywords`  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: Use expression to construct URL:  
     `https://completion.amazon.com/api/2017/suggestions?mid=ATVPDKIKX0DER&alias=aps&prefix={{ $json.Keyword }}`  
   - Connect output of `Get airtable data` to this node.

4. **Create Split Out Node to Extract Suggestions:**  
   - Name: `Format output`  
   - Type: Split Out  
   - Field to Split Out: `suggestions` (the array field in API response)  
   - Connect output of `Get Amazon keywords` to this node.

5. **Create Set Node to Clean Keywords:**  
   - Name: `Clean Keywords`  
   - Type: Set  
   - Create new field `Keywords` of type Array and assign value `={{ $json.value }}` (the individual suggestion text)  
   - Enable option to ignore conversion errors  
   - Connect output of `Format output` to this node.

6. **Create Aggregate Node:**  
   - Name: `Aggregate keywords`  
   - Type: Aggregate  
   - Fields to aggregate: `Keywords` (aggregate all keyword arrays into one)  
   - Connect output of `Clean Keywords` to this node.

7. **Create Code Node to Combine into String:**  
   - Name: `Combine into string`  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     return [{
       json: {
         keywords: items[0].json.Keywords.join(", ")
       }
     }];
     ```  
   - Connect output of `Aggregate keywords` to this node.

8. **Create Airtable Node to Save Keywords:**  
   - Name: `Save keywords`  
   - Type: Airtable  
   - Credentials: Same Airtable Personal Access Token  
   - Base & Table: Same as retrieval node  
   - Operation: Update record  
   - Record ID: Use expression to get original ID: `={{ $('Get airtable data').item.json.id }}`  
   - Map the field `Keyword output` to `{{$json.keywords}}` (output string from code node)  
   - Connect output of `Combine into string` to this node.

9. **Connect nodes in the following order:**  
   `Receive Keyword` → `Get airtable data` → `Get Amazon keywords` → `Format output` → `Clean Keywords` → `Aggregate keywords` → `Combine into string` → `Save keywords`

10. **Test webhook by sending a request with query parameter `q` set to a valid Airtable record ID containing the initial keyword.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| This workflow gives you Amazon keywords for your Amazon FBA business.                                                                       | [Workflow overview and details by Rumjahn](https://rumjahn.com/how-to-build-your-own-amazon-keywords-tool-with-n8n-for-free-and-no-coding/) |
| You need to send the workflow a keyword through webhook. Example Airtable base provided for sending keywords.                              | [Download Airtable example](https://airtable.com/invite/l?inviteId=invgv9FzNB258bm5Z&inviteToken=6f820e142d3324318254c1768fa57809b3ef0bcb7212ea27730fd2d140c69ad5&utm_medium=email&utm_source=product_team&utm_content=transactional-alerts) |
| Amazon has a Completion API that provides keyword data useful for advertising campaigns.                                                    |                                                                                                                                |
| Download Airtable example to save the keywords with this workflow.                                                                           | Same Airtable download link as above.                                                                                            |
| Credentials required: Airtable Personal Access Token with permission to read and write to the specified base and table.                     |                                                                                                                                |
| Handle API rate limits and token expiration for Airtable and Amazon API to ensure robustness.                                                |                                                                                                                                |
| Webhook URL must be publicly accessible to receive external requests.                                                                        |                                                                                                                                |

---

This document fully describes the workflow’s structure, node configurations, error considerations, and reproduction steps, enabling advanced users and automation agents to understand, maintain, or extend the Amazon FBA keyword generation process.