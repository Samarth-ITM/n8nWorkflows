Batch Scrape Website URLs from Google Sheets to Google Docs with Firecrawl

https://n8nworkflows.xyz/workflows/batch-scrape-website-urls-from-google-sheets-to-google-docs-with-firecrawl-7384


# Batch Scrape Website URLs from Google Sheets to Google Docs with Firecrawl

### 1. Workflow Overview

This workflow automates the batch scraping of website URLs listed in a Google Sheets spreadsheet, using the Firecrawl API to extract content and save it as individual markdown-formatted Google Docs in Google Drive. It is designed for AI chatbot developers, content managers, and data analysts who need to consolidate web content into organized documents for knowledge base creation, content analysis, or migration.

The logic is structured into these main blocks:

- **1.1 Trigger and Input Reception**  
  Receives the Google Sheets URL containing the list of pages to scrape via a chat trigger webhook.

- **1.2 URL Extraction and Validation**  
  Reads URLs from the specified Google Sheets tab, filters out empty rows, and checks for unprocessed URLs.

- **1.3 Batch Processing Loop**  
  Iterates over each valid URL sequentially to control rate limiting and processing order.

- **1.4 Webpage Scraping and Content Extraction**  
  Uses Firecrawl API to scrape each webpage, converting its content into markdown format.

- **1.5 Document Creation and Storage**  
  Creates a Google Doc for each scraped page in a designated Google Drive folder.

- **1.6 Progress Tracking and Status Update**  
  Updates the Google Sheets list marking each URL as "OK" once successfully processed.

- **1.7 Completion Notification**  
  Sends a JSON response confirming end of scraping with a direct link to the Google Drive folder containing all scraped documents.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Reception

- **Overview:**  
  This block triggers the workflow when a chat message containing the Google Sheets URL is received.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: Langchain Chat Trigger (webhook mode)  
    - Configuration: Public webhook enabled, response mode set to handle response via a response node  
    - Inputs: HTTP webhook payload with chatInput containing the Google Sheets URL  
    - Outputs: Emits JSON with `chatInput` field forwarding the URL  
    - Failures: Webhook may fail if the chat service is down or webhook URL is inaccessible

#### 2.2 URL Extraction and Validation

- **Overview:**  
  Reads the “Page to doc” sheet from the provided Google Sheets URL, filters out rows missing URLs, and checks if the URL has already been scraped.

- **Nodes Involved:**  
  - Get URL  
  - Row not empty  
  - If

- **Node Details:**  
  - **Get URL**  
    - Type: Google Sheets (read operation)  
    - Configuration: Reads sheet named "Page to doc" from the spreadsheet URL provided dynamically via expression `={{ $json.chatInput }}`  
    - Inputs: Trigger output containing chatInput URL  
    - Outputs: Rows of data including URL column  
    - Failures: Google Sheets API errors if URL invalid or access denied

  - **Row not empty**  
    - Type: Filter  
    - Configuration: Filters rows where the "URL" field exists and is not empty  
    - Inputs: Rows from Get URL  
    - Outputs: Only non-empty URL rows  
    - Edge cases: Rows with missing or malformed URLs filtered out silently

  - **If**  
    - Type: Conditional check  
    - Configuration: Checks if the `scraped` field is empty (meaning URL not yet processed)  
    - Inputs: Filtered rows with URLs  
    - Outputs: Proceeds only with URLs that have not been marked as scraped  
    - Failures: Logic errors if `scraped` field missing or misconfigured

#### 2.3 Batch Processing Loop

- **Overview:**  
  Processes each valid URL sequentially to avoid API rate limits and ensure orderly handling.

- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**  
  - **Loop Over Items**  
    - Type: Split In Batches  
    - Configuration: Default batch size 1 (sequential processing)  
    - Inputs: URLs passing the If condition  
    - Outputs: Single URL item per iteration passed downstream  
    - Edge cases: If batch processing interrupted, can resume from last processed URL by re-running

#### 2.4 Webpage Scraping and Content Extraction

- **Overview:**  
  Scrapes webpage content for each URL using Firecrawl API, converting it into markdown format.

- **Nodes Involved:**  
  - Scraping

- **Node Details:**  
  - **Scraping**  
    - Type: Firecrawl node (third-party web scraping service)  
    - Configuration: Operation set to “scrape” with default request options, uses Firecrawl API key credential  
    - Inputs: URL from Loop Over Items  
    - Outputs: JSON containing markdown content and metadata (including source URL)  
    - Failures: API call failures due to network, invalid URLs, or API quota exceeded

#### 2.5 Document Creation and Storage

- **Overview:**  
  Saves the scraped markdown content into a Google Doc, named after the source URL, inside a specified Google Drive folder.

- **Nodes Involved:**  
  - Create file markdown scraping

- **Node Details:**  
  - **Create file markdown scraping**  
    - Type: Google Drive (create file)  
    - Configuration: Creates a Google Doc with the name set dynamically to the scraped URL (`={{ $('Scraping').item.json.data.metadata.url }}`), content set to scraped markdown, saved to folder "Contenu scrapé" (folder ID `1ry3xvQ9UqM2Rf9C4-AoJdg1lfB9inh_5`)  
    - Inputs: Markdown content and metadata from Scraping node  
    - Outputs: Confirmation of file creation  
    - Failures: Google Drive API errors if folder ID invalid or insufficient permissions

#### 2.6 Progress Tracking and Status Update

- **Overview:**  
  Marks each URL in the Google Sheet as “OK” once its document has been created successfully.

- **Nodes Involved:**  
  - Scraped : OK

- **Node Details:**  
  - **Scraped : OK**  
    - Type: Google Sheets (update operation)  
    - Configuration: Updates the “Scrapé” column to "OK" for the corresponding URL row, matching by URL  
    - Inputs: Current URL from Loop Over Items and original spreadsheet URL from chat input  
    - Outputs: Updated spreadsheet row confirmation  
    - Failures: API errors if matching row not found or permission issues

- **Connections:**  
  After update, loops back to Loop Over Items for next URL processing

#### 2.7 Completion Notification

- **Overview:**  
  Once all URLs are processed, sends a JSON response with a message and link to the folder containing all scraped documents.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  
  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration: Sends a JSON response with text message including a clickable link to the Google Drive folder "Contenu scrapé"  
    - Inputs: Output of Loop Over Items after completion  
    - Outputs: HTTP response to trigger source  
    - Failures: Response failure if webhook connection lost or timeout

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                          | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                                                                                                                                         |
|----------------------------|----------------------------------|----------------------------------------|----------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received  | Langchain Chat Trigger            | Workflow trigger via chat message      | —                          | Get URL                       | Firecrawl batch scraping to Google Docs: AI chatbot developers, content managers, data analysts. Reads URLs from Google Sheets, scrapes pages, saves markdown docs, updates progress, final notification. See full workflow details and setup instructions in sticky note.                                                                                              |
| Get URL                    | Google Sheets                    | Reads URLs from provided spreadsheet   | When chat message received | Row not empty                 |                                                                                                                                                                                                                                                                                                                                                                     |
| Row not empty              | Filter                           | Filters out empty URL rows              | Get URL                    | If                           |                                                                                                                                                                                                                                                                                                                                                                     |
| If                        | If                              | Checks if URL already scraped           | Row not empty              | Loop Over Items               |                                                                                                                                                                                                                                                                                                                                                                     |
| Loop Over Items            | Split In Batches                 | Processes URLs one by one               | If                        | Respond to Webhook, Scraping  |                                                                                                                                                                                                                                                                                                                                                                     |
| Scraping                  | Firecrawl                       | Scrapes webpage and converts to markdown | Loop Over Items            | Create file markdown scraping |                                                                                                                                                                                                                                                                                                                                                                     |
| Create file markdown scraping | Google Drive                   | Creates Google Doc with scraped content | Scraping                   | Scraped : OK                 |                                                                                                                                                                                                                                                                                                                                                                     |
| Scraped : OK              | Google Sheets                   | Marks URL as scraped in the sheet       | Create file markdown scraping | Loop Over Items               |                                                                                                                                                                                                                                                                                                                                                                     |
| Respond to Webhook        | Respond to Webhook              | Sends completion message with folder link | Loop Over Items            | —                            |                                                                                                                                                                                                                                                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add a **Langchain Chat Trigger** node named "When chat message received"  
   - Set mode to "webhook", public true, response mode "responseNode"  
   - This node will receive chat input containing the Google Sheets URL.

2. **Add Google Sheets Read Node**  
   - Add a **Google Sheets** node named "Get URL"  
   - Set operation to "Read"  
   - Set Sheet Name to "Page to doc"  
   - Set Document ID dynamically via expression `={{ $json.chatInput }}` (input from chat trigger)  
   - Configure Google Sheets OAuth2 credentials.

3. **Add Filter Node**  
   - Add a **Filter** node named "Row not empty"  
   - Condition: Field "URL" exists and is not empty  
   - Connect "Get URL" output to this filter input.

4. **Add If Node**  
   - Add an **If** node named "If"  
   - Condition: Check if `scraped` field is empty (string empty check on `={{ $json.scraped }}`)  
   - Connect "Row not empty" output to this node.

5. **Add Split In Batches Node**  
   - Add **Split In Batches** node named "Loop Over Items"  
   - Keep default batch size 1 for sequential processing  
   - Connect the true output of "If" node to this node.

6. **Add Firecrawl Scrape Node**  
   - Add **Firecrawl** node named "Scraping"  
   - Set operation to "scrape"  
   - Configure Firecrawl API credentials  
   - Connect "Loop Over Items" output to this node.

7. **Add Google Drive Create File Node**  
   - Add **Google Drive** node named "Create file markdown scraping"  
   - Operation: Create file from text  
   - Name: use expression `={{ $('Scraping').item.json.data.metadata.url }}` to name file after scraped URL  
   - Content: set to scraped markdown content `={{ $('Scraping').item.json.data.markdown }}`  
   - Folder ID: set to your target folder "Contenu scrapé" (default ID `1ry3xvQ9UqM2Rf9C4-AoJdg1lfB9inh_5`)  
   - Configure Google Drive OAuth2 credentials  
   - Connect "Scraping" output to this node.

8. **Add Google Sheets Update Node**  
   - Add **Google Sheets** node named "Scraped : OK"  
   - Operation: Update row  
   - Sheet Name: "Page to doc"  
   - Document ID: expression from chat input URL `={{ $('When chat message received').item.json.chatInput }}`  
   - Matching column: "URL" to current item's URL `={{ $('Loop Over Items').item.json.URL }}`  
   - Set "Scrapé" column value to "OK"  
   - Configure Google Sheets OAuth2 credentials  
   - Connect "Create file markdown scraping" output to this node.

9. **Loop Back**  
   - Connect "Scraped : OK" node output back to "Loop Over Items" node to process next URL.

10. **Add Respond to Webhook Node**  
    - Add **Respond to Webhook** node named "Respond to Webhook"  
    - Set response type to JSON  
    - Response body:  
      ```json
      {
        "text": "Fin du scraping rendez vous dans le dossier [Contenu scrapé](https://drive.google.com/drive/folders/1ry3xvQ9UqM2Rf9C4-AoJdg1lfB9inh_5) pour retrouver vos pages, déplacez les docs vers votre document RAG si vous souhaitez les ajouter à la base de données de votre client"
      }
      ```  
    - Connect the "Loop Over Items" node’s main output (after all batches processed) to this node.

11. **Set Credentials**  
    - Configure and assign credentials for:  
      - Firecrawl API (API key)  
      - Google Sheets OAuth2  
      - Google Drive OAuth2

12. **Folder Setup**  
    - Create or designate a Google Drive folder for storing scraped docs  
    - Update folder ID in "Create file markdown scraping" node accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Firecrawl batch scraping to Google Docs: Intended for AI chatbot developers, content managers, and data analysts. Reads URLs from Google Sheets, scrapes pages to markdown, stores as Google Docs, updates progress, and sends final notification. Includes instructions for setup, customization, and use cases like knowledge base creation, competitor analysis, and content migration. Workflow handles duplicate prevention, empty row filtering, and sequential processing for rate limiting. See detailed notes in sticky note inside workflow JSON.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Embedded sticky note in workflow JSON                          |
| Default Google Drive folder ID: `1ry3xvQ9UqM2Rf9C4-AoJdg1lfB9inh_5` named "Contenu scrapé" — replace with your own folder ID for custom use.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Google Drive folder setup                                      |
| Requires Firecrawl API key with sufficient quota and Google OAuth2 credentials for Sheets and Drive access.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Credential setup                                               |
| For usage without chat interface, replace the chat trigger with a manual trigger node and set Google Sheets URL as a static variable in the "Get URL" node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Alternative trigger setup                                      |
| The workflow currently processes URLs sequentially to avoid scraping API rate limits; batch size can be adjusted in "Loop Over Items" node if desired, but caution for rate limits is advised.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Performance and rate limiting note                             |
| The output document format is Google Docs with markdown content; users can customize naming, folder structure, or output format as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Output customization                                          |
| Error handling is basic: the workflow continues processing remaining URLs if individual scrapes fail. Users may add retry or error logging logic as enhancements.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Error handling recommendation                                 |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.