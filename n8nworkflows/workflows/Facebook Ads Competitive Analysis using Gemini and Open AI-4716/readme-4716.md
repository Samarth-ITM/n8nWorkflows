Facebook Ads Competitive Analysis using Gemini and Open AI

https://n8nworkflows.xyz/workflows/facebook-ads-competitive-analysis-using-gemini-and-open-ai-4716


# Facebook Ads Competitive Analysis using Gemini and Open AI

---
### 1. Workflow Overview

This workflow, titled **"Facebook Ads Competitive Analysis using Gemini and Open AI"**, is designed to automate the process of collecting, processing, and analyzing competitor Facebook ads. It receives user input via a form submission, scrapes relevant ads, processes media content (videos and images), applies AI-powered analysis (using Gemini and OpenAI), and finally stores the analyzed data into Google Sheets for reporting or further use.

The workflow logically separates into the following blocks:

- **1.1 Input Reception:** Captures user input via form submission to trigger the workflow.
- **1.2 Ads Scraping and Routing:** Scrapes Facebook ads data and routes processing based on media type.
- **1.3 Video Processing Pipeline:** Downloads videos, extracts content, decodes video via Gemini API, and loops through video data.
- **1.4 Image Processing and AI Analysis:** Sends image data to OpenAI for analysis and prepares final image results.
- **1.5 Data Merging and Storage:** Merges video and image analysis results and saves to Google Sheets.
- **1.6 Control and Timing:** Manages batch processing and wait/retry mechanisms to handle asynchronous operations and API rate limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for form submissions from users, which act as the entry point for the workflow. It collects parameters needed to start scraping ads.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Starts the workflow when a user submits a form.  
    - Configuration: Uses a webhook ID to receive HTTP POST requests from the form. No additional parameters.  
    - Input: External HTTP request (form data).  
    - Output: Passes form data to the next node ("Scrape Ads").  
    - Edge Cases: Missing or malformed form data could lead to empty scraping parameters or failure downstream.

---

#### 2.2 Ads Scraping and Routing

- **Overview:**  
  This block scrapes Facebook ads data based on form inputs and uses a Switch node to route processing paths depending on the media type detected (video or image).

- **Nodes Involved:**  
  - Scrape Ads  
  - Switch

- **Node Details:**

  - **Scrape Ads**  
    - Type: HTTP Request  
    - Role: Queries Facebook or an ads database/API to collect competitor ads data.  
    - Configuration: Parameters likely set dynamically from form data (not explicitly shown).  
    - Input: Data from "On form submission".  
    - Output: Passes scraped ads data to "Switch".  
    - Edge Cases: HTTP errors, authentication issues, empty or malformed response data.

  - **Switch**  
    - Type: Switch  
    - Role: Routes data flow based on media type (e.g., video vs. image).  
    - Configuration: Contains conditional logic on scraped ads properties (e.g., media type field).  
    - Input: Scraped ads data from "Scrape Ads".  
    - Output:  
      - If video: routes to "Download Video" node.  
      - If image: routes to "OpenAI" node.  
    - Edge Cases: Unrecognized media types or missing media fields cause routing errors or dropped data.

---

#### 2.3 Video Processing Pipeline

- **Overview:**  
  This block handles video ads by downloading the video, extracting relevant data from the file, decoding video content via Gemini API, and iterating over video data items for further processing.

- **Nodes Involved:**  
  - Download Video  
  - Extract from File  
  - Loop Over Items  
  - Wait  
  - Gemini Video Decode  
  - Final Video Values

- **Node Details:**

  - **Download Video**  
    - Type: HTTP Request  
    - Role: Downloads video files from URLs obtained in ads data.  
    - Configuration: Uses video URL from input data.  
    - Input: From "Switch" when media type is video.  
    - Output: Passes downloaded video file to "Extract from File".  
    - Edge Cases: Download failures, invalid URLs, large file size/timeouts.

  - **Extract from File**  
    - Type: Extract From File  
    - Role: Extracts metadata or content from the downloaded video file.  
    - Configuration: Extracts relevant chunks or metadata for processing.  
    - Input: Video file from "Download Video".  
    - Output: Passes extracted items to "Loop Over Items".  
    - Edge Cases: Unsupported file formats, corrupted files.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes video data items in batches for controlled throughput.  
    - Configuration: Batch size not specified but typically set to avoid API rate limits.  
    - Input: Extracted video data from "Extract from File".  
    - Output: Two outputs—First to "Final Video Values", second to "Wait" for delay between batches.  
    - Edge Cases: Batch size misconfiguration, empty batches.

  - **Wait**  
    - Type: Wait  
    - Role: Introduces delay between batch processing to handle rate limiting or asynchronous processing.  
    - Configuration: Uses a webhook ID, suggesting asynchronous wait or external trigger before proceeding.  
    - Input: From "Loop Over Items" batch output.  
    - Output: To "Gemini Video Decode" for next batch processing.  
    - Edge Cases: Timeout delays, webhook trigger failures.

  - **Gemini Video Decode**  
    - Type: HTTP Request  
    - Role: Sends video content/data to Gemini API for decoding or advanced processing.  
    - Configuration: Retries twice on failure, retry on fail enabled.  
    - Input: Triggered after "Wait" node completes.  
    - Output: Feeds back into "Loop Over Items" for iterative processing.  
    - Edge Cases: API errors, rate limits, timeouts, invalid responses.

  - **Final Video Values**  
    - Type: Set  
    - Role: Prepares or formats the final video analysis results for merging.  
    - Configuration: Sets or transforms data fields as needed.  
    - Input: Main batch output from "Loop Over Items".  
    - Output: To "Merge" node.  
    - Edge Cases: Incorrect data mappings or missing fields.

---

#### 2.4 Image Processing and AI Analysis

- **Overview:**  
  This block processes image-based ads by sending data to OpenAI for analysis and preparing the final image data for merging.

- **Nodes Involved:**  
  - OpenAI  
  - Final Image Values

- **Node Details:**

  - **OpenAI**  
    - Type: OpenAI (via LangChain)  
    - Role: Uses OpenAI’s GPT or related models to analyze or generate insights from image data.  
    - Configuration: Parameters not fully detailed but likely include prompt templates and model selection.  
    - Input: Routed from "Switch" when media type is image.  
    - Output: Passes AI-generated insights to "Final Image Values".  
    - Edge Cases: API key or quota issues, malformed prompt, API timeouts.

  - **Final Image Values**  
    - Type: Set  
    - Role: Formats the AI output for final merging.  
    - Configuration: Sets or adjusts output fields to match expected schema.  
    - Input: From "OpenAI".  
    - Output: To "Merge" node.  
    - Edge Cases: Missing or malformed AI output.

---

#### 2.5 Data Merging and Storage

- **Overview:**  
  This block merges the processed video and image data streams and stores the combined dataset into a Google Sheet for structured storage and reporting.

- **Nodes Involved:**  
  - Merge  
  - Save

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines the final video and image data streams into one unified dataset.  
    - Configuration: Likely uses "Wait" or "Merge by index" mode to synchronize streams.  
    - Input: Receives from "Final Video Values" (main output 0) and "Final Image Values" (main output 1).  
    - Output: Sends merged data to "Save".  
    - Edge Cases: Data mismatches, uneven array lengths, synchronization issues.

  - **Save**  
    - Type: Google Sheets  
    - Role: Inserts or updates rows in a Google Sheet to save analysis results.  
    - Configuration: Specific spreadsheet and sheet name parameters not detailed but required.  
    - Input: From "Merge".  
    - Output: Terminal node.  
    - Edge Cases: Authentication errors, quota limits, malformed data rows.

---

#### 2.6 Control and Timing

- **Overview:**  
  Manages flow control for asynchronous processing and retries, primarily for video data decoding.

- **Nodes Involved:**  
  - Wait (also part of Video Processing)  
  - Gemini Video Decode (retry enabled)

- **Node Details:**  
  Covered above with video processing nodes.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                          | Input Node(s)               | Output Node(s)          | Sticky Note                                   |
|---------------------|--------------------------------|----------------------------------------|-----------------------------|-------------------------|-----------------------------------------------|
| On form submission  | Form Trigger                   | Entry point, receives user input       | -                           | Scrape Ads              |                                               |
| Scrape Ads          | HTTP Request                  | Scrape competitor Facebook ads         | On form submission          | Switch                  |                                               |
| Switch              | Switch                       | Routes data flow by media type          | Scrape Ads                  | Download Video, OpenAI  |                                               |
| Download Video      | HTTP Request                  | Downloads video files                   | Switch                      | Extract from File       |                                               |
| Extract from File   | Extract From File             | Extracts content/metadata from video   | Download Video              | Loop Over Items         |                                               |
| Loop Over Items     | Split In Batches              | Processes video data in batches         | Extract from File           | Final Video Values, Wait|                                               |
| Wait                | Wait                         | Delays batch processing                  | Loop Over Items             | Gemini Video Decode     |                                               |
| Gemini Video Decode | HTTP Request                 | Calls Gemini API to decode video content| Wait                        | Loop Over Items         |                                               |
| Final Video Values  | Set                          | Formats video analysis results           | Loop Over Items             | Merge                   |                                               |
| OpenAI              | OpenAI (LangChain)           | AI analysis of image data                | Switch                      | Final Image Values      |                                               |
| Final Image Values  | Set                          | Formats AI image analysis results        | OpenAI                      | Merge                   |                                               |
| Merge               | Merge                        | Combines video and image data streams   | Final Video Values, Final Image Values | Save |                                               |
| Save                | Google Sheets                | Saves final data into Google Sheets     | Merge                       | -                       |                                               |
| Sticky Note         | Sticky Note                  | Comment/annotation                       | -                           | -                       |                                               |
| Sticky Note1        | Sticky Note                  | Comment/annotation                       | -                           | -                       |                                               |
| Sticky Note2        | Sticky Note                  | Comment/annotation                       | -                           | -                       |                                               |
| Sticky Note3        | Sticky Note                  | Comment/annotation                       | -                           | -                       |                                               |
| Sticky Note4        | Sticky Note                  | Comment/annotation                       | -                           | -                       |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" Node:**
   - Type: Form Trigger  
   - Configure webhook to receive form submissions that specify competitor details or query parameters.

2. **Create "Scrape Ads" Node:**
   - Type: HTTP Request  
   - Configure to call Facebook Ads API or another source to scrape ads based on form input.  
   - Pass form submission parameters dynamically.

3. **Create "Switch" Node:**
   - Type: Switch  
   - Add conditions to check the media type of each scraped ad item (e.g., `mediaType == 'video'` or `mediaType == 'image'`).  
   - Connect "Scrape Ads" output to this node.

4. **Video Processing Path:**

   4.1 **Create "Download Video" Node:**
    - Type: HTTP Request  
    - Configure to download video file from ad URL.  
    - Connect "Switch" output for video media type.

   4.2 **Create "Extract from File" Node:**
    - Type: Extract From File  
    - Configure to extract video metadata or segments.

   4.3 **Create "Loop Over Items" Node:**
    - Type: Split In Batches  
    - Configure batch size suitable for API limits (e.g., 5 or 10).  
    - Input from "Extract from File".

   4.4 **Create "Final Video Values" Node:**
    - Type: Set  
    - Format the output data as needed for merging.

   4.5 **Create "Wait" Node:**
    - Type: Wait  
    - Configure delay or set to wait for an external webhook trigger to pace batch processing.

   4.6 **Create "Gemini Video Decode" Node:**
    - Type: HTTP Request  
    - Configure HTTP request to Gemini API endpoint with retry enabled (max 2 tries) and retry on failure.  
    - Connect from "Wait" node, output back to "Loop Over Items" to continue batch processing.

5. **Image Processing Path:**

   5.1 **Create "OpenAI" Node:**
    - Type: OpenAI (LangChain)  
    - Configure with appropriate credentials and prompt templates to analyze image ads.  
    - Connect "Switch" output for image media type.

   5.2 **Create "Final Image Values" Node:**
    - Type: Set  
    - Format AI output for merging.

6. **Create "Merge" Node:**
   - Type: Merge  
   - Configure to merge two input streams:  
     - Input 0: "Final Video Values" output  
     - Input 1: "Final Image Values" output

7. **Create "Save" Node:**
   - Type: Google Sheets  
   - Configure credentials and target spreadsheet/sheet.  
   - Map merged data fields to sheet columns.

8. **Connect the nodes accordingly:**

   - "On form submission" → "Scrape Ads" → "Switch"  
   - "Switch" video output → "Download Video" → "Extract from File" → "Loop Over Items"  
   - "Loop Over Items" batch output → "Final Video Values" → "Merge" (input 0)  
   - "Loop Over Items" batch wait output → "Wait" → "Gemini Video Decode" → back to "Loop Over Items"  
   - "Switch" image output → "OpenAI" → "Final Image Values" → "Merge" (input 1)  
   - "Merge" → "Save"

9. **Credentials Setup:**
   - OpenAI node requires valid OpenAI API credentials.  
   - Google Sheets node requires OAuth2 credentials with appropriate access scopes.  
   - Gemini Video Decode requires access credentials and endpoint details for the Gemini API.

10. **Default values and constraints:**
    - Batch sizes and wait times should be tuned to avoid API rate limits.  
    - Retry policies on HTTP nodes to handle transient failures.  
    - Proper error handling or fallback routes can be added for robustness.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                 |
|---------------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow automates competitive analysis of Facebook ads using AI and video decoding APIs.         | Workflow purpose                               |
| Uses Gemini API with retry and wait loops to handle asynchronous video processing.                 | Video processing robustness                     |
| OpenAI integration leverages LangChain node for advanced AI analysis on images.                    | AI media analysis                              |
| Google Sheets used as final data repository for easy reporting and integration.                    | Data storage                                   |
| For detailed OpenAI API usage guidance, refer to https://platform.openai.com/docs/api-reference    | OpenAI API docs                                |
| Use n8n official Google Sheets node docs for authentication setup: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleSheets/ | Google Sheets integration docs                 |

---

**Disclaimer:** The provided text is derived solely from an automated n8n workflow export. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.