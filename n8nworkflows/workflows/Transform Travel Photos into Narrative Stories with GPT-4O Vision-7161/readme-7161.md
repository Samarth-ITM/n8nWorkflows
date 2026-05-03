Transform Travel Photos into Narrative Stories with GPT-4O Vision

https://n8nworkflows.xyz/workflows/transform-travel-photos-into-narrative-stories-with-gpt-4o-vision-7161


# Transform Travel Photos into Narrative Stories with GPT-4O Vision

### 1. Workflow Overview

This workflow, titled **"Transform Travel Photos into Narrative Stories with GPT-4O Vision"**, is designed to accept a batch of travel photos uploaded via webhook, analyze each photo using OpenAI's GPT-4O Vision model, and generate a rich, immersive, first-person travel narrative based on the photo content and metadata.  

The workflow targets users who want to convert their travel photo collections into engaging stories, capturing sensory details, emotions, and cultural observations. It is ideal for travelers, storytellers, bloggers, or social media content creators seeking automated narrative generation from visual inputs.

The workflow logic is organized into the following functional blocks:

- **1.1 Entry Point & Input Extraction:** Receives photos via HTTP POST, extracts photo array.
- **1.2 Photo Processing:** Splits photos, converts base64 images to binary, extracts metadata.
- **1.3 AI Vision Analysis:** Analyzes each photo’s content using GPT-4O Vision model.
- **1.4 Photo Grouping by Day:** Organizes photos chronologically by day with themes.
- **1.5 Story Generation:** Uses GPT-4.1 to create a structured travel story from grouped photo data.
- **1.6 Final Formatting & Response:** Packages the generated story for client consumption.

---

### 2. Block-by-Block Analysis

#### 2.1 Entry Point & Input Extraction

- **Overview:**  
  This block listens for incoming photo uploads via a webhook and extracts the `photos` array from the request body, preparing it for downstream processing.

- **Nodes Involved:**  
  - Photo Upload Webhook  
  - Get Photos Array

- **Node Details:**  

  - **Photo Upload Webhook**  
    - Type: Webhook  
    - Role: Receives POST requests at `/tripteller-upload` URL path  
    - Configuration: HTTP method POST, response mode set to last node  
    - Input: HTTP POST body expected to contain a `photos` array  
    - Output: Passes raw JSON payload downstream  
    - Edge Cases: Missing `photos` array or invalid payload may cause failures or empty downstream data  
    - Sticky Note: Explains entry point and expected input format

  - **Get Photos Array**  
    - Type: Set  
    - Role: Extracts the `photos` array from webhook JSON body into a clean workflow variable  
    - Configuration: Sets `photos` field with expression referencing webhook data `body.photos`  
    - Input: Webhook JSON  
    - Output: JSON object with `photos` array  
    - Edge Cases: If `photos` field is missing or empty, downstream nodes receive empty arrays  
    - Sticky Note: Describes data extraction purpose

---

#### 2.2 Photo Processing

- **Overview:**  
  Splits the photos array into individual items for parallel processing, converts each photo from base64 to binary file format, and extracts photo metadata such as filename, size, and timestamp.

- **Nodes Involved:**  
  - Split Photos Array  
  - Convert to File  
  - Set Photo Attribs  
  - Merge (Combine)

- **Node Details:**

  - **Split Photos Array**  
    - Type: Split Out  
    - Role: Takes the `photos` array and creates separate workflow executions for each photo  
    - Configuration: Splits on field `photos`, outputs individual photo objects in `data`  
    - Input: Photos array from previous node  
    - Output: Each photo object separately  
    - Edge Cases: Empty photo arrays lead to no splits; large arrays may cause performance impacts  
    - Sticky Note: Explains splitting purpose

  - **Convert to File**  
    - Type: Convert to File  
    - Role: Converts base64 photo data to binary file format required for image analysis  
    - Configuration: Source property is `data.data` (base64), filename taken from `data.filename`  
    - Input: Individual photo JSON with base64 data  
    - Output: Binary file object with original filename preserved  
    - Edge Cases: Invalid base64 data causes conversion failure  
    - Sticky Note: Details binary conversion requirements and filename preservation

  - **Set Photo Attribs**  
    - Type: Set  
    - Role: Extracts and sets photo metadata attributes such as filename, size, and last modified timestamp  
    - Configuration: Copies these fields from photo data into the workflow JSON  
    - Input: Individual photo JSON  
    - Output: Photo JSON enriched with metadata fields  
    - Edge Cases: Missing metadata fields could cause incomplete grouping or sorting  
    - Sticky Note: Describes metadata extraction and importance for chronology

  - **Combine (Merge)**  
    - Type: Merge  
    - Role: Combines the outputs of `Convert to File` and `Set Photo Attribs` for each photo into a single JSON object  
    - Configuration: Combine mode with preference for last data in case of clashes, combining by position  
    - Input: Binary file data and metadata JSONs from parallel branches  
    - Output: Single unified photo object with binary and metadata  
    - Edge Cases: If either input is missing, merge may fail or produce incomplete data  

---

#### 2.3 AI Vision Analysis

- **Overview:**  
  Uses the GPT-4O Vision-enabled model to analyze each photo’s visual content, extracting descriptions, objects, scenes, and emotional cues.

- **Nodes Involved:**  
  - Vision Analysis

- **Node Details:**  

  - **Vision Analysis**  
    - Type: OpenAI Node (Langchain integration)  
    - Role: Analyzes the binary photo using GPT-4O Vision model  
    - Configuration:  
      - Model: `gpt-4o` (Vision-enabled GPT-4)  
      - Resource: Image  
      - Input Type: Base64 (converted binary file)  
      - Operation: Analyze  
    - Credentials: OpenAI API key configured  
    - Input: Binary image file (from Combine node)  
    - Output: JSON with detailed photo description, themes, and analysis  
    - Edge Cases: API failures (auth, rate limits), invalid image data, timeouts  
    - Sticky Note: Explains AI image analysis details and credential use

---

#### 2.4 Photo Grouping by Day

- **Overview:**  
  Aggregates the analyzed photos into trip days based on their timestamps, sorting photos chronologically within each day and extracting daily themes.

- **Nodes Involved:**  
  - Group Photos by Day (Code node)

- **Node Details:**

  - **Group Photos by Day**  
    - Type: Code (JavaScript)  
    - Role: Groups photo JSON objects by their `timestamp` date, sorts them, and builds structured day objects including date, day number, themes, photo count  
    - Configuration: Custom JS code that:  
      - Converts timestamps to human-readable dates  
      - Groups photos by date string  
      - Sorts photos within each day by timestamp  
      - Extracts unique themes per day  
      - Returns an array of day objects with metadata and photos  
    - Input: Array of photo JSONs with analysis and metadata  
    - Output: JSON with:  
      - `tripDays`: Array of day objects  
      - `totalDays`: Number of days  
      - `totalPhotos`: Total photos processed  
      - `dateRange`: Start and end dates of trip  
    - Edge Cases: Missing or invalid timestamps cause grouping errors or misordering  
    - Sticky Note: Describes grouping logic and output structure

---

#### 2.5 Story Generation

- **Overview:**  
  Creates a first-person immersive travel story using GPT-4.1 based on the grouped photo data and analyses, applying narrative style and structure.

- **Nodes Involved:**  
  - Generate Travel Story

- **Node Details:**

  - **Generate Travel Story**  
    - Type: OpenAI Node (Langchain integration)  
    - Role: Uses GPT-4.1 to generate a structured travel story from trip photo data  
    - Configuration:  
      - Model: `gpt-4.1`  
      - Temperature: 0.7 (balanced creativity)  
      - Top P: 1  
      - Penalties: None (presence and frequency penalty = 0)  
      - Messages:  
        - System prompt sets role as professional storyteller with instructions for first-person narrative, day-by-day structure, sensory details, tone options, and Markdown formatting  
        - User prompt includes serialized tripDays JSON data for content reference  
      - JSON output enabled  
    - Credentials: OpenAI API key configured  
    - Input: Grouped tripDays JSON object  
    - Output: JSON with story content in structured message format  
    - Edge Cases: API failures, malformed input JSON, token limits  
    - Sticky Note: Details narrative prompt design and expected output format

---

#### 2.6 Final Formatting & Response

- **Overview:**  
  Wraps the generated story into a final response object signaling success and making the story content accessible for client applications.

- **Nodes Involved:**  
  - Format Final Response

- **Node Details:**

  - **Format Final Response**  
    - Type: Set  
    - Role: Sets a success flag and packages the story content under the `storybook` key  
    - Configuration:  
      - Sets `success` boolean to true  
      - Sets `storybook` object with story content extracted from the previous node's JSON response  
    - Input: Story JSON from Generate Travel Story  
    - Output: Final response JSON ready to send back to client  
    - Edge Cases: Missing story content would result in incomplete response  
    - Sticky Note: Describes output formatting and readiness for client consumption

---

### 3. Summary Table

| Node Name             | Node Type                     | Functional Role                      | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                 |
|-----------------------|-------------------------------|------------------------------------|------------------------|----------------------------|---------------------------------------------------------------------------------------------|
| Photo Upload Webhook   | Webhook                       | Entry point for photo upload       | -                      | Get Photos Array            | 📌 ENTRY POINT - Listens for POST requests at `/tripteller-upload` - Expects photos array in request body |
| Get Photos Array       | Set                           | Extracts photos array from webhook | Photo Upload Webhook    | Split Photos Array          | 📌 DATA EXTRACTION - Extracts 'photos' array from webhook payload - Creates clean data structure |
| Split Photos Array     | Split Out                     | Splits photos array into items     | Get Photos Array        | Convert to File, Set Photo Attribs | 📌 PHOTO SEPARATION - Splits photos array into individual items - Each photo becomes separate execution |
| Convert to File        | Convert to File               | Converts base64 to binary file     | Split Photos Array      | Vision Analysis             | 📌 BINARY CONVERSION - Converts base64 photo data to binary format - Preserves original filename |
| Set Photo Attribs      | Set                           | Extracts photo metadata            | Split Photos Array      | Combine                    | 📌 METADATA EXTRACTION - Captures filename, size, timestamp - Essential for chronological ordering |
| Combine               | Merge                         | Merges binary file with metadata   | Convert to File, Set Photo Attribs | Group Photos by Day  |                                                                                             |
| Vision Analysis        | OpenAI (Langchain)            | Analyzes photo content visually    | Combine                 | Combine                    | 📌 AI IMAGE ANALYSIS - Model: GPT-4o Vision - Analyzes photo content, scenes, emotions        |
| Group Photos by Day    | Code (JavaScript)             | Groups photos by day and themes    | Combine                 | Generate Travel Story       | 📌 CHRONOLOGICAL ORGANIZATION - Groups photos by date - Sorts and extracts themes per day     |
| Generate Travel Story  | OpenAI (Langchain)            | Generates immersive travel story   | Group Photos by Day      | Format Final Response       | 📌 AI STORY CREATION - Model: GPT-4.1 - Narrative generation with structure and sensory detail |
| Format Final Response  | Set                           | Formats final output response      | Generate Travel Story   | -                          | 📌 OUTPUT FORMATTING - Creates response with success flag and story content                   |
| Sticky Note            | Sticky Note                   | Documentation                     | -                      | -                          | Various notes linked contextually to nodes                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Photo Upload Webhook`  
   - HTTP Method: POST  
   - Path: `tripteller-upload`  
   - Response Mode: Last Node  
   - Purpose: Receives photo upload POST requests with JSON body including `photos` array.

2. **Create Set Node**  
   - Name: `Get Photos Array`  
   - Operation: Set  
   - Assign `photos` field with expression: `={{ $item(0).$node["Photo Upload Webhook"].json["body"]["photos"] }}`  
   - Connect from `Photo Upload Webhook`.

3. **Create Split Out Node**  
   - Name: `Split Photos Array`  
   - Operation: Split Out  
   - Field to split: `photos`  
   - Destination Field Name: `data`  
   - Connect from `Get Photos Array`.

4. **Create Convert to File Node**  
   - Name: `Convert to File`  
   - Operation: To Binary  
   - Source Property: `data.data` (base64 image data)  
   - File Name: Expression `={{ $json.data.filename }}`  
   - Connect from `Split Photos Array`.

5. **Create Set Node for Metadata**  
   - Name: `Set Photo Attribs`  
   - Operation: Set  
   - Assign fields:  
     - `filename` = `={{ $json.data.filename }}`  
     - `size` = `={{ $json.data.size }}`  
     - `timestamp` = `={{ $json.data.lastModified }}`  
   - Connect from `Split Photos Array`.

6. **Create Merge Node**  
   - Name: `Combine`  
   - Mode: Combine (Merge)  
   - Combine by: Position  
   - Clash Handling: Prefer last  
   - Connect inputs:  
     - Input 1: From `Convert to File`  
     - Input 2: From `Set Photo Attribs`.

7. **Create OpenAI Node for Vision Analysis**  
   - Name: `Vision Analysis`  
   - Resource: Image  
   - Operation: Analyze  
   - Model: `gpt-4o` (Vision-enabled GPT-4)  
   - Input Type: Base64  
   - Credentials: Configure with valid OpenAI API key  
   - Connect from `Combine`.

8. **Connect `Vision Analysis` output back to `Combine`**  
   - Connect the output of `Vision Analysis` to the same `Combine` node for further processing.

9. **Create Code Node to Group Photos by Day**  
   - Name: `Group Photos by Day`  
   - Language: JavaScript  
   - Code: Use provided JavaScript to group photos by date, sort, extract themes, and build structured trip day objects (see description in 2.4)  
   - Connect from `Combine`.

10. **Create OpenAI Node for Story Generation**  
    - Name: `Generate Travel Story`  
    - Model: `gpt-4.1`  
    - Temperature: 0.7  
    - Top P: 1  
    - Presence Penalty: 0  
    - Frequency Penalty: 0  
    - Messages:  
      - System Role: Professional storyteller, narrative style, first-person past tense, markdown formatting, tone options  
      - User Role: Input tripDays JSON with photos and analysis, instruct to create detailed story  
    - Credentials: Configure with same OpenAI API key  
    - Connect from `Group Photos by Day`.

11. **Create Set Node for Final Formatting**  
    - Name: `Format Final Response`  
    - Assign:  
      - `success` = `true` (boolean)  
      - `storybook` = `={{ $json.message.content }}` (story content from previous node)  
    - Connect from `Generate Travel Story`.

12. **Activate the workflow** and test by sending a POST request with a JSON body containing a `photos` array structured as expected (each photo with base64 data, filename, size, lastModified timestamp, and optional themes).

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                        |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------|
| The workflow uses OpenAI GPT-4O Vision for advanced image analysis and GPT-4.1 for story generation. Credentials must be set up beforehand. | OpenAI API documentation and n8n credential setup    |
| The narrative output is formatted in Markdown for rich presentation with headings and paragraphs. | Markdown formatting guidelines                         |
| The workflow expects photos with metadata: filename, size, and timestamp for proper grouping. | Data input schema requirements                         |
| The webhook path `/tripteller-upload` must be accessible and secured as needed.               | API endpoint configuration                             |
| The grouping JavaScript function is essential for chronological storytelling and theme extraction. | Custom code node - modify with care                    |
| For troubleshooting API rate limits or errors, monitor OpenAI usage and handle exceptions externally. | OpenAI API best practices                              |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated n8n workflow designed with respect to current content policies. No illegal, offensive, or protected content is included. All data handled is legal and publicly available.