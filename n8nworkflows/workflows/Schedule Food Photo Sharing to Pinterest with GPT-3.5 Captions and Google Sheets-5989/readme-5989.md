Schedule Food Photo Sharing to Pinterest with GPT-3.5 Captions and Google Sheets

https://n8nworkflows.xyz/workflows/schedule-food-photo-sharing-to-pinterest-with-gpt-3-5-captions-and-google-sheets-5989


# Schedule Food Photo Sharing to Pinterest with GPT-3.5 Captions and Google Sheets

---

### 1. Workflow Overview

This workflow automates the scheduled sharing of highly-rated food photos from a Google Sheet to Pinterest, leveraging GPT-3.5 to generate engaging captions. Its primary target use case is for food bloggers, restaurants, or social media managers who want to streamline Pinterest content posting by combining spreadsheet management with AI-powered captions and direct API publishing.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow automatically at a defined interval (daily).
- **1.2 Data Retrieval:** Fetches food photo details and metadata from a Google Sheet.
- **1.3 Filtering:** Selects only photos rated 4 stars or higher and not yet posted.
- **1.4 AI Caption Generation:** Uses OpenAI GPT-3.5 to create concise, positive Pinterest captions based on customer feedback.
- **1.5 Pinterest Posting:** Uploads the photo along with the AI-generated caption to Pinterest via API.
- **1.6 Post-Processing:** Updates the Google Sheet to mark photos as posted.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution automatically, typically once daily, to check for new food photos ready for posting.

- **Nodes Involved:**  
  - Daily Post Scheduler

- **Node Details:**

  - **Daily Post Scheduler**  
    - Type: Schedule Trigger  
    - Configuration: Set for a recurring daily interval (default once per day)  
    - Key Expressions: None directly; triggers workflow start  
    - Input: None (trigger node)  
    - Output: Starts the workflow by connecting to "Fetch Food Photos from Sheet"  
    - Potential Failures: Misconfiguration of scheduling parameters could prevent trigger; n8n server downtime or time zone issues may affect timing  
    - Version Requirements: Compatible with n8n Schedule Trigger node version 1

---

#### 2.2 Data Retrieval

- **Overview:**  
  Retrieves all food photo entries and associated metadata from a configured Google Sheet to process further.

- **Nodes Involved:**  
  - Fetch Food Photos from Sheet

- **Node Details:**

  - **Fetch Food Photos from Sheet**  
    - Type: Google Sheets (Spreadsheet Get Operation)  
    - Configuration: Retrieves all rows from a specified Google Sheet using stored Google API credentials  
    - Credentials: Uses "Google Sheets - test" OAuth2 credentials  
    - Key Expressions: Fetches fields like 'Status', 'Rating', 'Feedback', 'Pin Title', 'Destination URL', 'Board ID', 'Image URL' from each row  
    - Input: Triggered by "Daily Post Scheduler"  
    - Output: Passes rows to the "Filter 4+ Star Dishes" node  
    - Potential Failures: Authentication errors if Google credentials expire, API quota limits, spreadsheet ID or range misconfiguration, empty or missing data rows  
    - Version Requirements: n8n Google Sheets node version 1

---

#### 2.3 Filtering

- **Overview:**  
  Filters the fetched rows to include only those with a 'Pending' status and a rating of 4 stars or higher, ensuring only relevant entries proceed.

- **Nodes Involved:**  
  - Filter 4+ Star Dishes

- **Node Details:**

  - **Filter 4+ Star Dishes**  
    - Type: If Node (Conditional Filtering)  
    - Configuration: Checks two conditions:  
      1. Status equals "Pending"  
      2. Rating is greater than or equal to 4  
    - Key Expressions:  
      - `{{$node['Google Sheets'].json['Status']}} == "Pending"`  
      - `{{$node['Google Sheets'].json['Rating']}} >= 4`  
    - Input: Receives data from "Fetch Food Photos from Sheet"  
    - Output: Passes qualifying rows to "AI Caption Generator"  
    - Potential Failures: Case sensitivity or formatting errors in Status field, numeric parsing issues in Rating  
    - Version Requirements: n8n If node version 1

---

#### 2.4 AI Caption Generation

- **Overview:**  
  Uses OpenAI GPT-3.5 to generate short, engaging Pinterest captions based on customer feedback from each filtered row.

- **Nodes Involved:**  
  - AI Caption Generator

- **Node Details:**

  - **AI Caption Generator**  
    - Type: OpenAI Node (Chat Completion with GPT-3.5)  
    - Configuration:  
      - Model: `gpt-3.5-turbo`  
      - Prompt Template:  
        > "Generate a concise, engaging Pinterest caption for a food photo based on this customer feedback: '{{Feedback}}'. Keep it under 100 characters and include a positive tone."  
      - Dynamic input: Uses `{{$node['Google Sheets'].json['Feedback']}}` for prompt injection  
    - Credentials: OpenAI API account credentials named "OpenAi account 2"  
    - Input: Filtered rows from "Filter 4+ Star Dishes"  
    - Output: Passes generated text to "Upload to Pinterest"  
    - Potential Failures: API rate limits, invalid or empty feedback text, network timeouts, authentication errors  
    - Version Requirements: Supports OpenAI API node version 1

---

#### 2.5 Pinterest Posting

- **Overview:**  
  Sends an HTTP POST request to Pinterest API to publish the food photo with the generated caption on the specified Pinterest board.

- **Nodes Involved:**  
  - Upload to Pinterest

- **Node Details:**

  - **Upload to Pinterest**  
    - Type: HTTP Request Node (POST)  
    - Configuration:  
      - URL: `https://api.pinterest.com/v5/pins`  
      - Method: POST  
      - Authentication: Header Authentication using Pinterest API token  
      - JSON Body Parameters:  
        ```json
        {
          "title": "{{$node['Google Sheets'].json['Pin Title']}}",
          "description": "{{$node['Generate Caption'].json['text']}}",
          "link": "{{$node['Google Sheets'].json['Destination URL']}}",
          "board_id": "{{$node['Google Sheets'].json['Board ID']}}",
          "media_source": {
            "source_type": "image_url",
            "url": "{{$node['Google Sheets'].json['Image URL']}}"
          }
        }
        ```  
    - Credentials: Pinterest API Token via HTTP Header Authentication node  
    - Input: Receives caption and photo metadata from "AI Caption Generator"  
    - Output: On success, triggers "Mark as Posted in Sheet"  
    - Potential Failures: API authentication failure, invalid board or media URLs, rate limiting, network issues, malformed JSON body  
    - Version Requirements: n8n HTTP Request node version 1

---

#### 2.6 Post-Processing

- **Overview:**  
  Updates the Google Sheet to mark the photo as posted, preventing duplicate posting on future runs.

- **Nodes Involved:**  
  - Mark as Posted in Sheet

- **Node Details:**

  - **Mark as Posted in Sheet**  
    - Type: Google Sheets (Update Operation)  
    - Configuration: Updates the row status or another field to indicate the photo has been posted  
    - Credentials: Same Google Sheets OAuth2 credentials as "Fetch Food Photos from Sheet"  
    - Input: Triggered by successful posting from "Upload to Pinterest"  
    - Output: None (workflow end)  
    - Potential Failures: Authentication errors, incorrect row identification, write permission issues, concurrent update conflicts  
    - Version Requirements: Google Sheets node version 1

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                         | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                          |
|---------------------------|-------------------------|---------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| Daily Post Scheduler       | Schedule Trigger        | Initiates workflow on schedule        | None                         | Fetch Food Photos from Sheet | Triggers the workflow at a scheduled time (e.g., once daily) to check for new food photos to post. |
| Fetch Food Photos from Sheet| Google Sheets           | Retrieves food photo rows and metadata| Daily Post Scheduler         | Filter 4+ Star Dishes        | Retrieves rows from the Google Sheet that contain food photos and metadata like rating and status.|
| Filter 4+ Star Dishes      | If Node                 | Filters photos with status "Pending" and rating >= 4 | Fetch Food Photos from Sheet | AI Caption Generator         | Filters only those food entries with high ratings (4 stars or above) and unposted status.          |
| AI Caption Generator       | OpenAI Node (GPT-3.5)   | Generates Pinterest captions from feedback | Filter 4+ Star Dishes         | Upload to Pinterest          | Uses AI (e.g., GPT/OpenAI) to create engaging and relevant captions for selected food photos.      |
| Upload to Pinterest        | HTTP Request            | Posts photo and caption to Pinterest API | AI Caption Generator          | Mark as Posted in Sheet      | Automatically posts the food photo with the generated caption to Pinterest via API.                |
| Mark as Posted in Sheet    | Google Sheets           | Updates sheet marking photo as posted | Upload to Pinterest           | None                        | Updates the Google Sheet to reflect that the photo has been successfully shared.                   |
| Sticky Note               | Sticky Note             | Notes and explanations                 | None                         | None                        | Various sticky notes describing each block’s purpose (see individual notes in positions).         |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Schedule Trigger** node named "Daily Post Scheduler".  
- Set interval to run once daily (default: every 24 hours).  
- No credentials required.

**Step 2:** Add a **Google Sheets** node named "Fetch Food Photos from Sheet".  
- Set resource to "Spreadsheet" and operation to "Get".  
- Configure to read rows from your target Google Sheet containing food photos metadata.  
- Connect credentials: use OAuth2 credentials with access to your spreadsheet ("Google Sheets - test").  
- Connect output of "Daily Post Scheduler" to this node.

**Step 3:** Add an **If** node named "Filter 4+ Star Dishes".  
- Configure two conditions under "String" and "Number" comparisons:  
  - Status equals "Pending" (string check)  
  - Rating greater or equal to 4 (number check)  
- Connect input from "Fetch Food Photos from Sheet".  
- Connect output to the next node on the "true" branch.

**Step 4:** Add an **OpenAI** node named "AI Caption Generator".  
- Select model "gpt-3.5-turbo".  
- Input prompt:  
  > Generate a concise, engaging Pinterest caption for a food photo based on this customer feedback: '{{$node["Google Sheets"].json["Feedback"]}}'. Keep it under 100 characters and include a positive tone.  
- Credentials: Attach OpenAI API credentials ("OpenAi account 2") with API key.  
- Connect "true" output from the If node to this OpenAI node.

**Step 5:** Add an **HTTP Request** node named "Upload to Pinterest".  
- Method: POST  
- URL: `https://api.pinterest.com/v5/pins`  
- Authentication: Header Auth using your Pinterest API token stored in HTTP Header Auth credentials.  
- Request Body (JSON parameters enabled):  
  ```json
  {
    "title": "{{$node['Google Sheets'].json['Pin Title']}}",
    "description": "{{$node['Generate Caption'].json['text']}}",
    "link": "{{$node['Google Sheets'].json['Destination URL']}}",
    "board_id": "{{$node['Google Sheets'].json['Board ID']}}",
    "media_source": {
      "source_type": "image_url",
      "url": "{{$node['Google Sheets'].json['Image URL']}}"
    }
  }
  ```  
- Connect input from "AI Caption Generator".

**Step 6:** Add a **Google Sheets** node named "Mark as Posted in Sheet".  
- Set operation to "Update" on the same spreadsheet.  
- Configure to update the row corresponding to the posted photo, marking the status as "Posted" or equivalent.  
- Connect input from "Upload to Pinterest".

**Step 7:** Add **Sticky Note** nodes as annotations for clarity at appropriate positions (optional but recommended for documentation).

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Purpose: Automatically share highly-rated food photos from a spreadsheet to Pinterest with AI-generated captions, and mark them as posted. | Workflow main objective (Sticky Note6)                          |
| Pinterest API documentation for posting pins: https://developers.pinterest.com/docs/api/v5/#operation/pins_create | Pinterest API Reference                                         |
| OpenAI API documentation: https://platform.openai.com/docs/models/gpt-3-5 | OpenAI API for prompt design and usage                          |
| Google Sheets API limits and authorization: https://developers.google.com/sheets/api/limits    | Important for managing quotas and authentication                |
| Scheduling best practices: Consider timezone and n8n server uptime to ensure timely triggers   | General execution note                                           |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---