Extract LinkedIn search results into a Google Sheet with SourceGeek

https://n8nworkflows.xyz/workflows/extract-linkedin-search-results-into-a-google-sheet-with-sourcegeek-13647


# Extract LinkedIn search results into a Google Sheet with SourceGeek

# Workflow Reference: Extract LinkedIn Search Results to Google Sheets

This document provides a technical breakdown of an n8n workflow designed to automate the extraction of LinkedIn search results (Basic, Sales Navigator, or Recruiter) and save the contact details into a Google Sheet using the SourceGeek integration.

## 1. Workflow Overview

The workflow automates the process of scraping LinkedIn lead data by accepting a search URL via an n8n Form. It identifies the type of LinkedIn search, initiates a background extraction job through SourceGeek, polls for completion, flattens the resulting contact data, and appends it to a centralized Google Spreadsheet.

### Logical Blocks
1.  **Input & Type Detection:** Captures the URL and determines if it is a Basic, Sales Navigator, or Recruiter search.
2.  **Extraction Initialization:** Triggers the specific SourceGeek operation based on the detected search type.
3.  **Status Polling Loop:** Monitors the asynchronous background job until the extraction is finished.
4.  **Data Transformation & Export:** Converts the nested JSON response into a flat array and writes the rows to Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Input & Type Detection
*   **Overview:** Receives the user input and categorizes the LinkedIn environment to ensure the correct scraper is used.
*   **Nodes Involved:** `On form submission`, `Code in JavaScript`, `Switch`.
*   **Node Details:**
    *   **On form submission (Form Trigger):** Presents a UI with one field: "Search Results URL".
    *   **Code in JavaScript:** Analyzes the URL string. If it contains `/sales/`, it tags it as `sales_navigator`; `/talent/` becomes `recruiter`; and `/search/` becomes `basic`.
    *   **Switch:** Routes the workflow to one of three branches based on the `linkedinType` variable.

### 2.2 Extraction Initialization
*   **Overview:** Contacts the SourceGeek API to start the scraping process.
*   **Nodes Involved:** `Import contacts from basic search`, `Import contacts from sales navigator search`, `Import contacts from recruiter search`.
*   **Node Details:**
    *   **Technical Role:** These are SourceGeek-specific nodes.
    *   **Configuration:** Each is set to its respective operation (e.g., `importContactsSalesNavSearch`).
    *   **Parameters:** `maxResults` is set to 10 by default.
    *   **Edge Cases:** Invalid LinkedIn URLs or expired SourceGeek API keys will cause these nodes to fail.

### 2.3 Status Polling Loop
*   **Overview:** LinkedIn extraction is asynchronous. This block checks the job status repeatedly until it is marked as "COMPLETED".
*   **Nodes Involved:** `Get_Run_ID`, `Get tool run ID`, `If Job Run is Complete`, `Wait until job is completed`.
*   **Node Details:**
    *   **Get_Run_ID (Code):** Normalizes the `toolRunId` from the previous step regardless of which search branch was taken.
    *   **Get tool run ID (SourceGeek):** Queries the SourceGeek API for the current status of the specific `runId`.
    *   **If Job Run is Complete:** A conditional check. If the status equals `COMPLETED`, it proceeds; otherwise, it moves to the Wait node.
    *   **Wait until job is completed:** Pauses the workflow (default wait) before looping back to the "Get tool run ID" node.

### 2.4 Data Transformation & Export
*   **Overview:** Prepares the raw JSON payload for spreadsheet insertion.
*   **Nodes Involved:** `Convert to array for insert`, `Append row in sheet`.
*   **Node Details:**
    *   **Convert to array for insert (Code):** Iterates through the `data.contacts` array returned by SourceGeek. It maps fields like `firstName`, `lastName`, and constructs a full `linkedinProfileUrl` from the username.
    *   **Append row in sheet (Google Sheets):** Maps the processed JS object keys (Company, Location, Job title, etc.) to the corresponding columns in the target Google Sheet.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On form submission | Form Trigger | User Input | None | Code in JavaScript | Fill in a LinkedIn search results page from Basic, Sales Navigator or Recruiter |
| Code in JavaScript | Code | URL Classification | On form submission | Switch | Getting search results from LinkedIn and placing them in a Google Sheet |
| Switch | Switch | Routing | Code in JavaScript | Import search nodes | Check, based on the url, which Action should be taken |
| Import contacts from basic search | SourceGeek | Start Scraping | Switch | Get_Run_ID | Returns a Run ID while the actual 'list generating' is happening in the background |
| Import contacts from sales nav search | SourceGeek | Start Scraping | Switch | Get_Run_ID | Returns a Run ID while the actual 'list generating' is happening in the background |
| Import contacts from recruiter search | SourceGeek | Start Scraping | Switch | Get_Run_ID | Returns a Run ID while the actual 'list generating' is happening in the background |
| Get_Run_ID | Code | ID Normalization | Import search nodes | Get tool run ID | Returns a Run ID while the actual 'list generating' is happening in the background |
| Get tool run ID | SourceGeek | Status Check | Get_Run_ID, Wait | If Job Run is Complete | Loop through this Action which wait untill the complete list is generated |
| If Job Run is Complete | If | Status Logic | Get tool run ID | Convert to array / Wait | Loop through this Action which wait untill the complete list is generated |
| Wait until job is completed | Wait | Throttling | If Job Run is Complete | Get tool run ID | If not ready. wait 5 seconds and try again |
| Convert to array for insert | Code | Data Formatting | If Job Run is Complete | Append row in sheet | When ready, convert the output and append the data to rows in a Google Sheet |
| Append row in sheet | Google Sheets | Data Export | Convert to array | None | When ready, convert the output and append the data to rows in a Google Sheet |

---

## 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Form Trigger** node. Add a text field labeled "Search Results URL".
2.  **URL Parsing:** Connect a **Code** node (JavaScript). Use logic to check if the input URL contains "sales", "talent", or "search" and output a `linkedinType` string.
3.  **Router:** Connect a **Switch** node. Set the routing based on the `linkedinType` value (basic, sales_navigator, recruiter).
4.  **SourceGeek Integration:** 
    *   Add three **SourceGeek** nodes, one for each branch of the Switch. 
    *   Configure them with your SourceGeek API credentials.
    *   Set the "URL" parameter to reference the form submission: `{{ $('On form submission').item.json['Search Results URL'] }}`.
5.  **ID Capture:** Connect all three SourceGeek nodes to a single **Code** node. Use logic to capture the `toolRunId` into a standard variable.
6.  **Polling Loop:**
    *   Add another **SourceGeek** node using the "Get tool run" operation, passing the `runId`.
    *   Connect an **If** node to check if the status field equals `COMPLETED`.
    *   Connect the "False" output of the If node to a **Wait** node (set for 5-10 seconds), then connect the Wait node back to the "Get tool run" node.
7.  **Data Flattening:** Connect the "True" output of the If node to a **Code** node. Use a `for` loop to iterate over `item.json.data.contacts` and return a clean array of contact objects.
8.  **Storage:** Connect a **Google Sheets** node. Set the operation to "Append". Map your contact object fields (e.g., `firstName`, `companyName`) to your sheet columns.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| How it works: Use this workflow to retrieve search results from any LinkedIn platform and append them in a Google Sheet. | General Workflow Logic |
| The SourceGeek node is used for the extraction process. | Integration Requirement |
| SourceGeek API Documentation | [https://sourcegeek.io/](https://sourcegeek.io/) |
| The workflow includes a built-in polling loop to handle LinkedIn's asynchronous data generation. | Operational Strategy |