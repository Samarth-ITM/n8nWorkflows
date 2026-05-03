Parse Ycombinator news page

https://n8nworkflows.xyz/workflows/parse-ycombinator-news-page-1250


# Parse Ycombinator news page

### 1. Workflow Overview

This workflow automates the extraction of the latest news data from the Ycombinator news webpage (https://news.ycombinator.com/) and compiles it into a neatly structured spreadsheet file. The file is named dynamically based on the current date and then sent via email as an attachment.  
It addresses the limitation of the current n8n version (0.141.1) where item list nodes need to extract variables individually before merging, rather than extracting multiple columns at once.

**Logical blocks:**

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Data Retrieval:** HTTP request to fetch the Ycombinator news webpage.
- **1.3 Data Extraction:** Extract news titles and URLs separately from the HTML content.
- **1.4 Data Structuring:** Split extracted arrays into individual list items and merge them by index to align titles with URLs.
- **1.5 File Generation:** Create a spreadsheet file named with the current date containing the news data.
- **1.6 Notification:** Email the generated file as an attachment to recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:**  
  This block starts the workflow manually via user interaction.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters; simple manual start node.  
  - **Input connections:** None (entry point).  
  - **Output connections:** Connects to HTTP Request node.  
  - **Edge Cases:** None; manual trigger generally reliable.

---

#### 2.2 Data Retrieval

- **Overview:**  
  Retrieves the full HTML content of the Ycombinator news front page.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **Name:** HTTP Request  
  - **Type:** HTTP Request (GET)  
  - **Configuration:**  
    - URL: `https://news.ycombinator.com/`  
    - Response Format: File (binary)  
    - Full response enabled (returns full HTTP response)  
    - Batch Interval: 500 ms (rate limiting/safety)  
    - Allow unauthorized certificates: true (safe here since it's a public site, but could be risky if the site changes)  
  - Input connections: From Manual Trigger  
  - Output connections: To HTML Extract node  
  - Edge Cases:  
    - HTTP errors (e.g., network issues, 404, 500)  
    - Site structure changes causing invalid response  
    - Timeout or slow response  

---

#### 2.3 Data Extraction

- **Overview:**  
  Parses the HTML binary response to extract news titles and their URLs as separate arrays.

- **Nodes Involved:**  
  - HTML Extract

- **Node Details:**  
  - **Name:** HTML Extract  
  - **Type:** HTML Extract  
  - **Configuration:**  
    - Source Data: Binary (from HTTP Request)  
    - Extraction values:  
      - `news_title`: text content from elements matching `.storylink` CSS selector; returns array  
      - `news_url`: href attribute of elements matching `.storylink`; returns array  
  - Input connections: From HTTP Request  
  - Output connections: To two itemList nodes (`list news title`, `list news url`) in parallel  
  - Edge Cases:  
    - CSS selector `.storylink` changes or disappears (site redesign)  
    - No elements found returns empty arrays  
    - Extraction failures if response is not valid HTML  

---

#### 2.4 Data Structuring

- **Overview:**  
  Converts the arrays of titles and URLs into item lists and merges them by their index to create structured pairs.

- **Nodes Involved:**  
  - list news title  
  - list news url  
  - Merge

- **Node Details:**  
  - **list news title**  
    - Type: Item Lists  
    - Configuration: Splits out the field `news_title` into individual items  
    - Input: From HTML Extract  
    - Output: To Merge node (input 0)  
    - Edge Cases: Empty or uneven data arrays may cause misalignment  

  - **list news url**  
    - Type: Item Lists  
    - Configuration: Splits out the field `news_url` into individual items  
    - Input: From HTML Extract  
    - Output: To Merge node (input 1)  
    - Edge Cases: Same as above  

  - **Merge**  
    - Type: Merge  
    - Configuration: Mode set to `mergeByIndex` to align items from both lists by their array position  
    - Input: Two inputs from the two item list nodes  
    - Output: Structured combined data with both title and URL per item  
    - Edge Cases:  
      - Unequal list lengths can cause missing data or truncated merges  
      - Merge conflicts if arrays are empty  

---

#### 2.5 File Generation

- **Overview:**  
  Converts the merged structured data to a spreadsheet file (.xls by default) with a dynamically generated filename including the current date.

- **Nodes Involved:**  
  - Spreadsheet File

- **Node Details:**  
  - **Name:** Spreadsheet File  
  - **Type:** Spreadsheet File  
  - **Configuration:**  
    - Operation: `toFile` (creates a file output)  
    - Options:  
      - `fileName`: Expression generating `"Ycombinator_news_YYYY-MM-DD.<fileFormat>"` dynamically using JavaScript Date ISO string split to get date only  
      - `sheetName`: `"Latest news"`  
    - File format parameter (`fileFormat`) must be passed or defaulted (commonly `xls`)  
  - Input connections: From Merge node  
  - Output connections: To Send email notification node  
  - Edge Cases:  
    - File format parameter missing or invalid  
    - File writing errors (permission, storage)  
    - Expression failures for file name generation  

---

#### 2.6 Notification

- **Overview:**  
  Sends an email with the generated spreadsheet attached to configured recipients.

- **Nodes Involved:**  
  - Send email notification

- **Node Details:**  
  - **Name:** Send email notification  
  - **Type:** Email Send  
  - **Configuration:**  
    - Subject: `"Ycombinator news"`  
    - Body text: `"Here are the latest news attached!"`  
    - Attachments: Passes the binary file data from Spreadsheet File node  
    - To Email / From Email: Must be set by user or credential defaults  
    - Credentials: SMTP credentials configured externally (OAuth2 or username/password)  
  - Input connections: From Spreadsheet File node  
  - Output connections: None (terminal node)  
  - Edge Cases:  
    - SMTP authentication failures  
    - Invalid recipient email addresses  
    - Attachment size limits or email server restrictions  
    - Network or SMTP server downtime  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role          | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                                 |
|-------------------------|---------------------|-------------------------|-------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger      | Start workflow          | —                       | HTTP Request               |                                                                                                                             |
| HTTP Request            | HTTP Request        | Fetch Ycombinator page  | On clicking 'execute'    | HTML Extract               | Get news page                                                                                                               |
| HTML Extract            | HTML Extract        | Extract news titles & URLs | HTTP Request           | list news title, list news url | Extract news data                                                                                                           |
| list news title          | Item Lists          | Split titles array      | HTML Extract             | Merge                      |                                                                                                                             |
| list news url            | Item Lists          | Split URLs array        | HTML Extract             | Merge                      |                                                                                                                             |
| Merge                   | Merge               | Combine titles & URLs   | list news title, list news url | Spreadsheet File        |                                                                                                                             |
| Spreadsheet File        | Spreadsheet File    | Generate spreadsheet    | Merge                    | Send email notification    | The "fileName" option uses dynamic date: `Ycombinator_news_{{new Date().toISOString().split('T', 1)[0]}}.{{$parameter["fileFormat"]}}` |
| Send email notification | Email Send          | Send spreadsheet by email | Spreadsheet File        | —                          |                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node**  
   - Name it `On clicking 'execute'`  
   - No special configuration needed.

3. **Add an HTTP Request node**  
   - Name: `HTTP Request`  
   - Set HTTP Method to `GET`  
   - Set URL to `https://news.ycombinator.com/`  
   - Enable `Full Response` option to get the full HTTP response  
   - Set `Response Format` to `File` (binary output)  
   - Set `Batch Interval` to 500 ms  
   - Enable `Allow Unauthorized Certificates` (optional, for flexibility)  
   - Connect the Manual Trigger node output to this node input.

4. **Add an HTML Extract node**  
   - Name: `HTML Extract`  
   - Set `Source Data` to `Binary` (input from HTTP Request)  
   - Under `Extraction Values`, add two entries:  
     - Key: `news_title`, CSS Selector: `.storylink`, Return Array: true, Return Value: (default text content)  
     - Key: `news_url`, CSS Selector: `.storylink`, Return Array: true, Return Value: attribute, Attribute: `href`  
   - Connect HTTP Request node output to this node input.

5. **Add two Item Lists nodes**  
   - First one:  
     - Name: `list news title`  
     - Set `Field to Split Out` to `news_title`  
     - Connect HTML Extract node output to this node input.  
   - Second one:  
     - Name: `list news url`  
     - Set `Field to Split Out` to `news_url`  
     - Connect HTML Extract node output to this node input.

6. **Add a Merge node**  
   - Name: `Merge`  
   - Set `Mode` to `mergeByIndex`  
   - Connect the output of `list news title` node to input 0 (first input)  
   - Connect the output of `list news url` node to input 1 (second input)

7. **Add a Spreadsheet File node**  
   - Name: `Spreadsheet File`  
   - Set `Operation` to `toFile`  
   - Under `Options` configure:  
     - `fileName` to the expression:  
       ``` 
       Ycombinator_news_{{new Date().toISOString().split('T', 1)[0]}}.{{$parameter["fileFormat"]}} 
       ```  
     - `sheetName` to `Latest news`  
   - Ensure you provide a workflow parameter called `fileFormat` with a default value, e.g., `xls` or `xlsx`  
   - Connect Merge node output to this node input.

8. **Add an Email Send node**  
   - Name: `Send email notification`  
   - Set the `Subject` to `Ycombinator news`  
   - Set the `Text` to `Here are the latest news attached!`  
   - Set `Attachments` to `data` (the file from Spreadsheet File node)  
   - Configure `To Email` and `From Email` addresses accordingly  
   - Set SMTP credentials (OAuth2 or username/password) in the Credentials tab  
   - Connect Spreadsheet File node output to this node input.

9. **Verify connections:**  
   - Manual Trigger → HTTP Request → HTML Extract → (`list news title`, `list news url`) → Merge → Spreadsheet File → Send email notification

10. **Save and execute the workflow manually to test.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Current n8n version (0.141.1) requires extracting each variable separately with itemList nodes; future versions may allow multi-column extraction. | Describes workflow limitation and future expectations                                                  |
| The dynamic filename expression uses JavaScript Date methods and n8n parameter access: `Ycombinator_news_{{new Date().toISOString().split('T', 1)[0]}}.{{$parameter["fileFormat"]}}` | Important for generating date-based file names                                                        |
| Workflow screenshot available in the original source (fileId:543)                                                                            | Visual aid for node layout and connections                                                             |
| SMTP credentials are required for email sending; ensure valid configuration to avoid delivery failures                                        | Email sending best practice                                                                             |
| Ycombinator news page structure relies on `.storylink` class; monitor for site changes that could break extraction                           | Maintenance tip for workflow durability                                                                |

---

This document fully describes the workflow for parsing Ycombinator news and distributing it via email as a dated spreadsheet file, enabling reproduction, modification, and troubleshooting.