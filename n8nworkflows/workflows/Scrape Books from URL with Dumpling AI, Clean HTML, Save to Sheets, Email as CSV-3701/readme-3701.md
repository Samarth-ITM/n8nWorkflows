Scrape Books from URL with Dumpling AI, Clean HTML, Save to Sheets, Email as CSV

https://n8nworkflows.xyz/workflows/scrape-books-from-url-with-dumpling-ai--clean-html--save-to-sheets--email-as-csv-3701


# Scrape Books from URL with Dumpling AI, Clean HTML, Save to Sheets, Email as CSV

### 1. Workflow Overview

This workflow automates the extraction, processing, and delivery of book data scraped from a URL provided in a Google Sheet. It is designed for users who need to regularly gather structured product information (specifically books) from catalog-style websites and receive the data as a clean CSV file via email.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Watches a Google Sheet for new URLs to scrape.
- **1.2 Web Content Scraping:** Uses Dumpling AI to fetch and clean the HTML content of the target webpage.
- **1.3 HTML Parsing and Data Extraction:** Extracts the list of books and then individual book details (title and price) using CSS selectors.
- **1.4 Data Processing:** Splits the HTML array into individual book items and sorts them by price in descending order.
- **1.5 Data Conversion and Delivery:** Converts the sorted JSON data into a CSV file and sends it via Gmail as an email attachment.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new URL is added to a specified Google Sheet. It initiates the scraping process by passing the URL downstream.

- **Nodes Involved:**  
  - Trigger- Watches For new URL in Spreadsheet

- **Node Details:**

  - **Trigger- Watches For new URL in Spreadsheet**  
    - Type: Google Sheets Trigger  
    - Role: Monitors a Google Sheet for new rows added, specifically new URLs to scrape.  
    - Configuration:  
      - Event: `rowAdded` (triggers when a new row is added)  
      - Polling interval: every minute  
      - Sheet Name: dynamically linked to a sheet named "Sheet1" in a specified Google Sheets document  
      - Document ID: points to the Google Sheets document containing URLs  
      - Credentials: Uses OAuth2 credentials for Google Sheets access  
    - Inputs: None (trigger node)  
    - Outputs: Passes the new row data containing the URL downstream  
    - Edge Cases / Failures:  
      - Authentication errors if OAuth2 token expires or is invalid  
      - Missing or incorrect sheet/document ID causing trigger failure  
      - Rate limits on Google Sheets API  
    - Version-specific: Requires n8n version supporting Google Sheets Trigger node (version 1 or higher)  

#### 2.2 Web Content Scraping

- **Overview:**  
  Sends a POST request to Dumpling AI’s scraping API to retrieve cleaned HTML content of the URL provided by the trigger.

- **Nodes Involved:**  
  - Scrape Website Content with Dumpling AI

- **Node Details:**

  - **Scrape Website Content with Dumpling AI**  
    - Type: HTTP Request  
    - Role: Calls Dumpling AI API to scrape the webpage and return cleaned HTML content.  
    - Configuration:  
      - Method: POST  
      - URL: `https://app.dumplingai.com/api/v1/scrape`  
      - Headers:  
        - Content-Type: application/json  
        - Authorization: via HTTP Header Auth with API key (credential)  
      - Body (JSON):  
        - `url`: dynamically set from the Google Sheets trigger node output  
        - `format`: "html"  
        - `cleaned`: "True" (requests optimized, cleaned HTML)  
      - Authentication: HTTP Header Auth with API key  
      - Credentials: Header Auth account containing Dumpling AI API key  
    - Inputs: Receives URL from Google Sheets trigger  
    - Outputs: Returns JSON containing the cleaned HTML content under the property `content`  
    - Edge Cases / Failures:  
      - API key invalid or expired causing authentication failure  
      - Dumpling AI service downtime or rate limiting  
      - Malformed or unreachable URL causing scraping failure  
      - Timeout if the target website is slow or unresponsive  
    - Version-specific: Requires n8n version supporting HTTP Request node version 4.1 or higher  

#### 2.3 HTML Parsing and Data Extraction

- **Overview:**  
  Extracts the list of book containers from the full HTML, then splits the list into individual book HTML blocks for further extraction of title and price.

- **Nodes Involved:**  
  - Extract all books from the page  
  - Split HTML Array into Individual Books  
  - Extract individual book price

- **Node Details:**

  - **Extract all books from the page**  
    - Type: HTML  
    - Role: Parses the cleaned HTML content to extract all book container elements.  
    - Configuration:  
      - Operation: `extractHtmlContent`  
      - Data Property Name: `content` (input property containing HTML)  
      - Extraction Values:  
        - Key: `books`  
        - CSS Selector: `.row > li` (selects each book container element)  
        - Return Type: HTML string array (returns an array of HTML blocks)  
      - Returns an array of HTML snippets, each representing a book container  
    - Inputs: Receives cleaned HTML content from Dumpling AI node  
    - Outputs: JSON with `books` array containing HTML strings  
    - Edge Cases / Failures:  
      - CSS selector invalid or site layout changed causing empty or incorrect extraction  
      - Input HTML missing or malformed  
    - Version-specific: Requires HTML node version 1.2 or higher  

  - **Split HTML Array into Individual Books**  
    - Type: Split Out  
    - Role: Splits the array of book HTML blocks into individual items for separate processing.  
    - Configuration:  
      - Field to split out: `books` (the array extracted previously)  
    - Inputs: Receives JSON with `books` array from previous node  
    - Outputs: Emits one item per book HTML block  
    - Edge Cases / Failures:  
      - Empty or missing `books` array results in no output items  
    - Version-specific: Standard Split Out node (version 1)  

  - **Extract individual book price**  
    - Type: HTML  
    - Role: Extracts specific fields (title and price) from each individual book HTML block.  
    - Configuration:  
      - Operation: `extractHtmlContent`  
      - Data Property Name: `books` (each item is an HTML block)  
      - Extraction Values:  
        - `title`: CSS selector `h3 > a`, attribute `title` (extracts book title from anchor tag’s title attribute)  
        - `price`: CSS selector `.price_color` (extracts price text)  
    - Inputs: Receives individual book HTML block from Split Out node  
    - Outputs: JSON with `title` and `price` fields for each book  
    - Edge Cases / Failures:  
      - Missing or changed CSS selectors causing empty fields  
      - Price extracted as string, may require parsing for numeric operations  
    - Version-specific: HTML node version 1.2 or higher  

#### 2.4 Data Processing

- **Overview:**  
  Sorts the extracted book data by price in descending order to prioritize higher-priced books.

- **Nodes Involved:**  
  - Sort by price

- **Node Details:**

  - **Sort by price**  
    - Type: Sort  
    - Role: Orders the list of books based on the `price` field descending.  
    - Configuration:  
      - Sort Field: `price`  
      - Order: Descending  
    - Inputs: Receives JSON array of books with `title` and `price` fields  
    - Outputs: Sorted JSON array  
    - Edge Cases / Failures:  
      - Price is a string; if prices contain currency symbols or formatting, sorting may not be numeric as expected  
      - Empty input array results in no output  
    - Version-specific: Sort node version 1 or higher  

#### 2.5 Data Conversion and Delivery

- **Overview:**  
  Converts the sorted JSON data into a CSV file and sends it as an email attachment via Gmail.

- **Nodes Involved:**  
  - Convert to CSV File  
  - Send CSV via e-mail

- **Node Details:**

  - **Convert to CSV File**  
    - Type: Convert to File  
    - Role: Transforms JSON book data into a CSV file format suitable for download or attachment.  
    - Configuration:  
      - Default options (no special parameters)  
    - Inputs: Receives sorted JSON array of books  
    - Outputs: Binary CSV file data attached to the workflow item  
    - Edge Cases / Failures:  
      - Malformed JSON input could cause conversion failure  
      - Large datasets may impact performance or file size limits  
    - Version-specific: Convert to File node version 1.1 or higher  

  - **Send CSV via e-mail**  
    - Type: Gmail  
    - Role: Sends an email with the CSV file attached to a specified recipient.  
    - Configuration:  
      - Recipient email address: configured in the node parameter `sendTo` (empty in template, must be set)  
      - Subject: "bookstore csv"  
      - Message: "Hey, here's the scraped data from the online bookstore!"  
      - Attachments: CSV file from previous node’s binary data  
      - Email Type: Text  
      - Credentials: Gmail OAuth2 credentials configured for sending email  
    - Inputs: Receives binary CSV file from Convert to CSV node  
    - Outputs: Email sent confirmation  
    - Edge Cases / Failures:  
      - Authentication failure if Gmail OAuth2 token expires or is invalid  
      - Attachment size limits on Gmail  
      - Missing recipient email address causes failure  
    - Version-specific: Gmail node version 2.1 or higher  

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                         | Input Node(s)                          | Output Node(s)                     | Sticky Note                                                                                                   |
|----------------------------------|---------------------------|---------------------------------------|--------------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------|
| Trigger- Watches For new URL in Spreadsheet | Google Sheets Trigger      | Starts workflow on new URL in sheet   | None                                 | Scrape Website Content with Dumpling AI | See Sticky Note: "Trigger to Raw Book HTML" describing initial trigger and scraping steps                      |
| Scrape Website Content with Dumpling AI      | HTTP Request              | Fetches cleaned HTML from Dumpling AI | Trigger- Watches For new URL in Spreadsheet | Extract all books from the page    | See Sticky Note: "Trigger to Raw Book HTML" describing scraping and extraction                                  |
| Extract all books from the page                | HTML                      | Extracts all book containers from HTML | Scrape Website Content with Dumpling AI | Split HTML Array into Individual Books | See Sticky Note: "Trigger to Raw Book HTML" describing extraction of book containers                            |
| Split HTML Array into Individual Books         | Split Out                 | Splits book array into individual items | Extract all books from the page       | Extract individual book price      | See Sticky Note: "Trigger to Raw Book HTML" describing splitting array                                         |
| Extract individual book price                   | HTML                      | Extracts title and price from each book | Split HTML Array into Individual Books | Sort by price                     | See Sticky Note: "Parse, Sort, Export & Email" describing extraction of individual fields                       |
| Sort by price                                  | Sort                      | Sorts books by price descending        | Extract individual book price          | Convert to CSV File                | See Sticky Note: "Parse, Sort, Export & Email" describing sorting step                                         |
| Convert to CSV File                            | Convert to File            | Converts JSON to CSV file               | Sort by price                        | Send CSV via e-mail               | See Sticky Note: "Parse, Sort, Export & Email" describing conversion to CSV                                    |
| Send CSV via e-mail                            | Gmail                     | Sends CSV file as email attachment     | Convert to CSV File                  | None                             | See Sticky Note: "Parse, Sort, Export & Email" describing email sending                                        |
| Sticky Note3                                   | Sticky Note               | Overview and summary of workflow       | None                                 | None                             | "Scrape Books from URL with Dumpling AI, Clean HTML, Save to Sheets, Email as CSV" overview and setup notes    |
| Sticky Note                                   | Sticky Note               | Describes trigger and initial scraping steps | None                                 | None                             | Details steps 1-4: Trigger, Dumpling AI scrape, extract all books, split array                                  |
| Sticky Note1                                  | Sticky Note               | Describes parsing, sorting, export, and email steps | None                                 | None                             | Details steps 5-8: Extract individual data, sort, convert to CSV, send email                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure to watch a specific Google Sheet document and sheet (e.g., "Sheet1") for new rows added.  
   - Set event to `rowAdded` and polling interval to every minute.  
   - Connect Google Sheets OAuth2 credentials.  
   - This node outputs the new row data including the URL to scrape.

2. **Create HTTP Request Node to Dumpling AI**  
   - Type: HTTP Request  
   - Set method to POST  
   - URL: `https://app.dumplingai.com/api/v1/scrape`  
   - Headers: Add `Content-Type: application/json`  
   - Authentication: Use HTTP Header Auth with your Dumpling AI API key credential.  
   - Body (JSON):  
     ```json
     {
       "url": "={{ $json['url'] }}",
       "format": "html",
       "cleaned": "True"
     }
     ```  
     Replace `{{$json['url']}}` with the expression referencing the URL from the Google Sheets trigger node output.  
   - Connect input from Google Sheets Trigger node.

3. **Create HTML Node to Extract All Books**  
   - Type: HTML  
   - Operation: `extractHtmlContent`  
   - Data Property Name: `content` (the property containing HTML from Dumpling AI node)  
   - Extraction Values:  
     - Key: `books`  
     - CSS Selector: `.row > li`  
     - Return Array: true  
     - Return Value: `html`  
   - Connect input from Dumpling AI HTTP Request node.

4. **Create Split Out Node**  
   - Type: Split Out  
   - Field to split out: `books` (the array extracted in previous node)  
   - Connect input from HTML node extracting all books.

5. **Create HTML Node to Extract Individual Book Data**  
   - Type: HTML  
   - Operation: `extractHtmlContent`  
   - Data Property Name: `books` (each item is an HTML block)  
   - Extraction Values:  
     - `title`: CSS selector `h3 > a`, attribute `title`  
     - `price`: CSS selector `.price_color`  
   - Connect input from Split Out node.

6. **Create Sort Node**  
   - Type: Sort  
   - Sort Field: `price`  
   - Order: Descending  
   - Connect input from HTML node extracting individual book data.

7. **Create Convert to File Node**  
   - Type: Convert to File  
   - File Type: CSV (default)  
   - Connect input from Sort node.

8. **Create Gmail Node**  
   - Type: Gmail  
   - Set recipient email address in `sendTo` parameter.  
   - Subject: "bookstore csv"  
   - Message: "Hey, here's the scraped data from the online bookstore!"  
   - Attachments: Add binary data from Convert to File node (CSV file)  
   - Connect Gmail OAuth2 credentials.  
   - Connect input from Convert to File node.

9. **Connect all nodes in the order:**  
   Google Sheets Trigger → Dumpling AI HTTP Request → Extract all books → Split Out → Extract individual book data → Sort → Convert to CSV → Send email

10. **Test the workflow:**  
    - Add a new URL (e.g., `http://books.toscrape.com`) to the Google Sheet.  
    - Confirm the workflow triggers, scrapes, processes, and sends the CSV email.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow scrapes book data from a website, turns it into a CSV, saves it, and sends it by email.     | Overview sticky note at workflow start                                                          |
| APIs for Gmail, Google Sheets & Drive must be enabled in Google Cloud Console for credentials to work.    | Setup instructions in sticky notes                                                              |
| Dumpling AI requires an account and API key; usage consumes credits per request.                           | Setup instructions and authentication details                                                  |
| CSS selectors depend on website structure; update selectors if the target site layout changes.             | Important note on HTML node usage                                                               |
| Avoid scraping websites that prohibit automated scraping to comply with legal and ethical standards.      | General dependency and caution note                                                             |
| Dumpling AI documentation and signup: https://app.dumplingai.com                                         | External resource for API key and usage                                                         |
| Google Sheets trigger documentation: https://docs.n8n.io/nodes/n8n-nodes-base.googleSheetsTrigger/        | Reference for configuring Google Sheets trigger                                                 |
| Gmail node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.gmail/                                 | Reference for configuring Gmail node and OAuth2 credentials                                     |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "Scrape Books from URL with Dumpling AI, Clean HTML, Save to Sheets, Email as CSV" workflow in n8n.