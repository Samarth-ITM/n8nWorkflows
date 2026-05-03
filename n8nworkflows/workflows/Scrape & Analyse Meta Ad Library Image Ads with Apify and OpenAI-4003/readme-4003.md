Scrape & Analyse Meta Ad Library Image Ads with Apify and OpenAI

https://n8nworkflows.xyz/workflows/scrape---analyse-meta-ad-library-image-ads-with-apify-and-openai-4003


# Scrape & Analyse Meta Ad Library Image Ads with Apify and OpenAI

### 1. Workflow Overview

This workflow, titled **Scrape & Analyse Meta Ad Library Image Ads with Apify and OpenAI**, automates the process of collecting image-based advertising content from Meta’s Ad Library, analyzing it using AI, and storing the structured insights alongside original images for further use. It targets marketers, researchers, and analysts interested in understanding visual ad strategies on Facebook.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initial Setup:** Triggering the workflow and setting initial parameters such as the target ad library URL, scraping limits, and output configurations.
- **1.2 Data Scraping and Preprocessing:** Using Apify to scrape Meta ads, calculating runtime in days, sorting and filtering ads by reach and format.
- **1.3 Limiting and Selecting Ads for Analysis:** Constraining the number of image ads to analyze based on parameters.
- **1.4 Image Download and Storage:** Downloading ad images and saving originals to Google Drive.
- **1.5 AI-Based Content Analysis:** Using OpenAI’s GPT-4o model via LangChain nodes to analyze image content, parse structured outputs, and extract insights.
- **1.6 Data Merging and Storage:** Combining AI analysis with ad metadata and storing all results in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initial Setup

- **Overview:** This block triggers the workflow manually and sets all initial user-configured parameters such as the target Meta Ad Library URL, scrape limits, and output folder.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Settings  
  - Clean Prompt

- **Node Details:**

  1. **When clicking ‘Test workflow’**  
     - *Type:* Manual Trigger  
     - *Role:* Entry point to manually start the workflow.  
     - *Config:* No parameters, simply triggers downstream nodes.  
     - *Connections:* Outputs to “Settings.”  
     - *Edge cases:* N/A since manual trigger.

  2. **Settings**  
     - *Type:* Set  
     - *Role:* Defines and stores input parameters such as:  
       - Meta Ad Library URL  
       - Maximum ads to scrape (default 300)  
       - Maximum ads to analyze (default 10)  
       - Google Drive folder for images  
       - Google Sheets target  
     - *Config:* User inputs configured here before running.  
     - *Connections:* Outputs to “Clean Prompt.”  
     - *Edge cases:* Missing or malformed URL could cause scraping failure downstream.

  3. **Clean Prompt**  
     - *Type:* Code  
     - *Role:* Prepares or sanitizes AI prompt text before sending to OpenAI.  
     - *Config:* Custom JavaScript for prompt cleaning (not raw JSON shown).  
     - *Connections:* Outputs to “Scrape Meta Ad Library with Apify.”  
     - *Edge cases:* Script errors or invalid prompt text might cause AI failures.

---

#### 1.2 Data Scraping and Preprocessing

- **Overview:** Scrapes Meta Ad Library using Apify, calculates ad runtime, sorts ads by reach or time running, and filters only image ads for further processing.
- **Nodes Involved:**  
  - Scrape Meta Ad Library with Apify  
  - Calculate Runtime in Days  
  - Sort by Reach or Days Running  
  - Filter only Image Ads

- **Node Details:**

  1. **Scrape Meta Ad Library with Apify**  
     - *Type:* HTTP Request  
     - *Role:* Calls Apify API to scrape Meta ads based on provided URL and parameters.  
     - *Config:* Uses Apify credentials, URL, maximum scrape limits.  
     - *Connections:* Outputs to “Calculate Runtime in Days.”  
     - *Edge cases:* API rate limits, network errors, invalid credentials.

  2. **Calculate Runtime in Days**  
     - *Type:* Set  
     - *Role:* Calculates how many days each ad has been running based on metadata timestamps.  
     - *Config:* Uses expressions to compute date differences.  
     - *Connections:* Outputs to “Sort by Reach or Days Running.”  
     - *Edge cases:* Missing or malformed date fields could cause calculation errors.

  3. **Sort by Reach or Days Running**  
     - *Type:* Sort  
     - *Role:* Sorts ads descending by reach or runtime days to prioritize top-performing ads.  
     - *Config:* Sort keys configured to ad reach or days running.  
     - *Connections:* Outputs to “Filter only Image Ads.”  
     - *Edge cases:* Ads missing reach or runtime data may be sorted incorrectly.

  4. **Filter only Image Ads**  
     - *Type:* Filter  
     - *Role:* Filters dataset to retain only ads that contain images (excluding video or other formats).  
     - *Config:* Condition based on ad media type field.  
     - *Connections:* Outputs to “Limit Images to Analyze.”  
     - *Edge cases:* Incorrect media type metadata may include/exclude wrong ads.

---

#### 1.3 Limiting and Selecting Ads for Analysis

- **Overview:** Limits the number of image ads to analyze further, respecting user parameter for maximum analysis count.
- **Nodes Involved:**  
  - Limit Images to Analyze  
  - Pass relevant Fields

- **Node Details:**

  1. **Limit Images to Analyze**  
     - *Type:* Limit  
     - *Role:* Restricts the number of ads passed on for detailed analysis to avoid overload.  
     - *Config:* Uses “max ads to analyze” from Settings.  
     - *Connections:* Outputs to “Pass relevant Fields.”  
     - *Edge cases:* If limit is zero or too high, may cause no analysis or resource exhaustion.

  2. **Pass relevant Fields**  
     - *Type:* Set  
     - *Role:* Selects and formats only necessary fields from ads for downstream processing (e.g., ad ID, image URL).  
     - *Config:* Filters out extraneous data.  
     - *Connections:* Outputs to “Download Image.”  
     - *Edge cases:* Missing fields may cause errors in image download or analysis.

---

#### 1.4 Image Download and Storage

- **Overview:** Downloads each image ad and saves it to Google Drive for archival and future reference.
- **Nodes Involved:**  
  - Download Image  
  - Save Image to Google Drive

- **Node Details:**

  1. **Download Image**  
     - *Type:* HTTP Request  
     - *Role:* Downloads ad image binary data from URL.  
     - *Config:* Configured for GET request, binary mode enabled.  
     - *Connections:* Outputs to “Save Image to Google Drive” and “Analyze Image Contents.”  
     - *Edge cases:* Broken URLs, timeouts, or access denied errors.

  2. **Save Image to Google Drive**  
     - *Type:* Google Drive  
     - *Role:* Uploads downloaded image files to a specified Google Drive folder.  
     - *Config:* Uses configured Google Drive OAuth2 credentials and target folder ID.  
     - *Connections:* Outputs to “Merge Data.”  
     - *Edge cases:* Insufficient permissions, quota exceeded, or API errors.

---

#### 1.5 AI-Based Content Analysis

- **Overview:** Uses OpenAI GPT-4o via LangChain nodes to analyze image ad contents and produce structured insights about visual elements, hooks, offers, CTAs, and psychological triggers.
- **Nodes Involved:**  
  - Analyze Image Contents  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  1. **Analyze Image Contents**  
     - *Type:* LangChain Agent  
     - *Role:* Orchestrates AI analysis using the chat model and output parser with retry on failure.  
     - *Config:* Max 2 tries, connected to AI components.  
     - *Connections:* Receives AI chat model and output parser outputs; outputs to “Merge Data.”  
     - *Edge cases:* AI failures, timeouts, or malformed responses.

  2. **OpenAI Chat Model**  
     - *Type:* LangChain LM Chat OpenAI  
     - *Role:* Sends cleaned AI prompts to GPT-4o model to generate analysis text.  
     - *Config:* Uses OpenAI API key, GPT-4o model specification.  
     - *Connections:* Provides AI-generated text to “Analyze Image Contents.”  
     - *Edge cases:* API rate limits, authentication failures.

  3. **Structured Output Parser**  
     - *Type:* LangChain Output Parser Structured  
     - *Role:* Parses AI-generated text into structured JSON or fields matching the analysis schema.  
     - *Config:* Predefined schema for visual description, hook, offer, CTA, triggers.  
     - *Connections:* Feeds parsed structured data back to “Analyze Image Contents.”  
     - *Edge cases:* Parse failures if AI output is malformed or unexpected.

---

#### 1.6 Data Merging and Storage

- **Overview:** Combines original ad metadata, AI analysis results, and image references; then stores comprehensive records in Google Sheets.
- **Nodes Involved:**  
  - Merge Data  
  - Store Data in Google Sheets

- **Node Details:**

  1. **Merge Data**  
     - *Type:* Merge  
     - *Role:* Merges two input streams — the saved image data and AI analysis — into consolidated records.  
     - *Config:* Merge type likely “Merge by Index” or “Merge by Key.”  
     - *Connections:* Outputs merged dataset to “Store Data in Google Sheets.”  
     - *Edge cases:* Mismatched inputs could cause data misalignment.

  2. **Store Data in Google Sheets**  
     - *Type:* Google Sheets  
     - *Role:* Writes the merged structured data into a user-specified Google Sheets spreadsheet for tracking and reporting.  
     - *Config:* Uses Google Sheets OAuth2 credentials, sheet ID, and mapping configuration.  
     - *Connections:* Terminal node.  
     - *Edge cases:* Write permission denied, quota limits, or sheet not found.

---

### 3. Summary Table

| Node Name                     | Node Type                       | Functional Role                                 | Input Node(s)              | Output Node(s)                        | Sticky Note                         |
|-------------------------------|--------------------------------|------------------------------------------------|----------------------------|-------------------------------------|-----------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                 | Workflow manual start trigger                   |                            | Settings                            |                                   |
| Settings                      | Set                            | Defines input parameters                        | When clicking ‘Test workflow’ | Clean Prompt                       |                                   |
| Clean Prompt                  | Code                           | Cleans and prepares AI prompt                   | Settings                   | Scrape Meta Ad Library with Apify   |                                   |
| Scrape Meta Ad Library with Apify | HTTP Request                 | Scrapes Meta Ad Library via Apify API            | Clean Prompt               | Calculate Runtime in Days           |                                   |
| Calculate Runtime in Days     | Set                            | Calculates ad runtime in days                    | Scrape Meta Ad Library with Apify | Sort by Reach or Days Running  |                                   |
| Sort by Reach or Days Running | Sort                           | Sorts ads by reach or days running               | Calculate Runtime in Days  | Filter only Image Ads               |                                   |
| Filter only Image Ads         | Filter                         | Filters to retain only image ads                 | Sort by Reach or Days Running | Limit Images to Analyze          |                                   |
| Limit Images to Analyze       | Limit                          | Limits number of image ads to analyze            | Filter only Image Ads      | Pass relevant Fields               |                                   |
| Pass relevant Fields          | Set                            | Selects necessary fields for further processing | Limit Images to Analyze    | Download Image                    |                                   |
| Download Image                | HTTP Request                   | Downloads image binary data                       | Pass relevant Fields       | Save Image to Google Drive, Analyze Image Contents |                                   |
| Save Image to Google Drive    | Google Drive                   | Uploads image to Google Drive                     | Download Image             | Merge Data                        |                                   |
| Analyze Image Contents        | LangChain Agent                | Runs AI analysis agent on image content          | Download Image, OpenAI Chat Model, Structured Output Parser | Merge Data |                                   |
| OpenAI Chat Model             | LangChain LM Chat OpenAI       | Sends prompts to OpenAI GPT-4o model             | Analyze Image Contents (ai_languageModel) | Analyze Image Contents           |                                   |
| Structured Output Parser      | LangChain Output Parser Structured | Parses AI output into structured fields          | Analyze Image Contents (ai_outputParser) | Analyze Image Contents           |                                   |
| Merge Data                   | Merge                          | Combines AI analysis and image storage data      | Save Image to Google Drive, Analyze Image Contents | Store Data in Google Sheets      |                                   |
| Store Data in Google Sheets   | Google Sheets                  | Saves all combined data to Google Sheets          | Merge Data                 |                                     |                                   |
| Sticky Note                  | Sticky Note                    | (Empty content)                                   |                            |                                     |                                   |
| Sticky Note1                 | Sticky Note                    | (Empty content)                                   |                            |                                     |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create manual trigger node**  
   - Name: “When clicking ‘Test workflow’”  
   - Type: Manual Trigger  
   - No parameters. Connect output to next node.

2. **Create Settings node**  
   - Type: Set  
   - Define variables:  
     - `metaAdLibraryUrl` (string): e.g. "https://www.facebook.com/ads/library/?page_id=12345"  
     - `maxAdsToScrape` (number, default 300)  
     - `maxAdsToAnalyze` (number, default 10)  
     - `googleDriveFolderId` (string): Google Drive folder for images  
     - `googleSheetId` (string): Target Google Sheets ID  
   - Connect input from manual trigger. Connect output to next node.

3. **Create Clean Prompt node**  
   - Type: Code  
   - Purpose: Sanitize and prepare AI prompt text before sending to OpenAI.  
   - Implement JavaScript code to clean any user inputs or construct prompt templates.  
   - Connect input from Settings node. Connect output to next node.

4. **Create HTTP Request node “Scrape Meta Ad Library with Apify”**  
   - Type: HTTP Request  
   - Method: POST or GET depending on Apify API  
   - URL: Apify Meta Ad Library scraping endpoint  
   - Authentication: Apify API Key credentials  
   - Body: Include `metaAdLibraryUrl`, `maxAdsToScrape` from Settings  
   - Connect input from Clean Prompt node. Connect output to next node.

5. **Create Set node “Calculate Runtime in Days”**  
   - Type: Set  
   - Use expressions to compute days running: e.g., difference between current date and ad start date  
   - Connect input from Apify scraping node. Connect output to next node.

6. **Create Sort node “Sort by Reach or Days Running”**  
   - Type: Sort  
   - Sort descending by ad reach or days running to prioritize ads  
   - Connect input from previous node. Connect output to next node.

7. **Create Filter node “Filter only Image Ads”**  
   - Type: Filter  
   - Condition: Media type equals ‘image’  
   - Connect input from Sort node. Connect output to next node.

8. **Create Limit node “Limit Images to Analyze”**  
   - Type: Limit  
   - Limit count to `maxAdsToAnalyze` from Settings  
   - Connect input from Filter node. Connect output to next node.

9. **Create Set node “Pass relevant Fields”**  
   - Type: Set  
   - Select only necessary fields such as ad ID, image URL, metadata for downstream processing  
   - Connect input from Limit node. Connect output to next node.

10. **Create HTTP Request node “Download Image”**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: image URL from ad fields  
    - Response Format: Binary  
    - Connect input from Set node. Connect outputs to two nodes simultaneously: “Save Image to Google Drive” and “Analyze Image Contents.”

11. **Create Google Drive node “Save Image to Google Drive”**  
    - Type: Google Drive  
    - Operation: Upload File  
    - Folder ID: `googleDriveFolderId` from Settings  
    - File Data: Binary data from image download  
    - Connect input from “Download Image.” Connect output to “Merge Data.”

12. **Create LangChain Agent node “Analyze Image Contents”**  
    - Type: LangChain Agent  
    - Max retries: 2  
    - Connect AI language model and output parser nodes as inputs.  
    - Connect input from “Download Image.” Connect output to “Merge Data.”

13. **Create LangChain LM Chat OpenAI node “OpenAI Chat Model”**  
    - Type: LangChain LM Chat OpenAI  
    - Model: GPT-4o  
    - Credentials: OpenAI API key  
    - Connect input/output to “Analyze Image Contents” (ai_languageModel).

14. **Create LangChain Output Parser Structured node “Structured Output Parser”**  
    - Type: LangChain Output Parser Structured  
    - Schema: Define fields for visual description, hook, offer, CTA, psychological triggers  
    - Connect input/output to “Analyze Image Contents” (ai_outputParser).

15. **Create Merge node “Merge Data”**  
    - Type: Merge  
    - Mode: Merge by index or key (e.g., ad ID)  
    - Inputs:  
      - From “Save Image to Google Drive” (image info)  
      - From “Analyze Image Contents” (AI analysis)  
    - Connect output to next node.

16. **Create Google Sheets node “Store Data in Google Sheets”**  
    - Type: Google Sheets  
    - Operation: Append Rows or Update Rows  
    - Spreadsheet ID: `googleSheetId` from Settings  
    - Map merged data fields (ad metadata + AI analysis + image link) to sheet columns  
    - Connect input from “Merge Data.” No outputs (terminal).

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow requires Apify account for scraping Meta Ad Library data effectively.                        | Apify registration: https://apify.com/signup                                                     |
| OpenAI GPT-4o is used for deep content analysis; ensure your API key has access to GPT-4o or equivalent.   | OpenAI API docs: https://platform.openai.com/docs                                               |
| Google Drive and Sheets OAuth2 credentials must be configured in n8n for respective nodes to work.         | Google API Console: https://console.cloud.google.com/apis/credentials                            |
| Adjust AI prompts in “Analyze Image Contents” node to extract customized insights beyond defaults provided. | Modify prompt templates in Code node or LangChain agent parameters                              |
| For large-scale analysis, consider batch processing or increasing API rate limits.                         | Monitor API usage and quotas in your Apify and OpenAI dashboards                                 |
| Useful blog explaining Meta Ad Library scraping and AI analysis workflows: https://example-blog.com/meta-ad-library-analysis | External resource for workflow inspiration and troubleshooting                                  |

---

This structured documentation provides all necessary details to understand, reproduce, and customize the workflow, as well as anticipate possible failure points and integration requirements.