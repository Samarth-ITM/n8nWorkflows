Instagram Visual Analysis with Apify Scraping, OpenAI GPT-5 & Google Sheets

https://n8nworkflows.xyz/workflows/instagram-visual-analysis-with-apify-scraping--openai-gpt-5---google-sheets-8068


# Instagram Visual Analysis with Apify Scraping, OpenAI GPT-5 & Google Sheets

### 1. Workflow Overview

This workflow automates the process of gathering recent Instagram post images for specified usernames, performing visual analysis on those images using advanced AI (OpenAI GPT-5), and managing the input and output data via Google Sheets. It integrates Apify scraping to retrieve Instagram profile media, processes images through an AI agent for content analysis, and organizes the data flow to enable quick visual insights.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Filtering:** Accepts manual trigger execution and retrieves Instagram usernames from a Google Sheet, filtering to a specific row.
- **1.2 Instagram Profile Data Scraping:** Uses Apify’s Instagram Profile Scraper API to fetch recent post data for the target username.
- **1.3 Image Extraction and HTTP Fetch:** Extracts individual posts from the dataset and retrieves the actual image binaries via HTTP.
- **1.4 AI Visual Analysis:** Sends the retrieved image data to an AI agent powered by OpenAI GPT-5 for detailed image content analysis.
- **1.5 Credential and Setup Notes:** Embedded guidance nodes providing setup instructions for credentials and configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

- **Overview:** This block starts the workflow manually, reads Instagram usernames from a Google Sheet, and filters the input to a specific row to control which user is processed.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Get Google Sheet  
  - Filter

##### Node Details:

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually by user action.  
  - Configuration: No parameters; triggers the workflow run on demand.  
  - Inputs: None  
  - Outputs: Connects to Get Google Sheet node.  
  - Edge Cases: None inherent, but user must manually trigger workflow.

- **Get Google Sheet**  
  - Type: Google Sheets  
  - Role: Reads the spreadsheet containing Instagram usernames.  
  - Configuration:  
    - Document ID set to a specific Google Sheet.  
    - Sheet Name: “Sheet1” (gid=0), targeting a worksheet with a `user` column.  
  - Credentials: Google Sheets OAuth2 (requires prior OAuth2 credential setup).  
  - Inputs: Receives trigger from manual trigger node.  
  - Outputs: Passes rows to Filter node.  
  - Edge Cases: Sheet access denied, incorrect document ID, or no rows present.

- **Filter**  
  - Type: Filter  
  - Role: Filters rows from the Google Sheet to select only the second row (row_number = 2).  
  - Configuration:  
    - Condition: `row_number` equals 2 (strict number comparison).  
  - Inputs: Data from Get Google Sheet node.  
  - Outputs: Passes filtered user data to Scrape Details node.  
  - Edge Cases: No row 2, condition expression failure.

---

#### 2.2 Instagram Profile Data Scraping

- **Overview:** This block uses Apify’s Instagram Profile Scraper API to fetch detailed profile data including recent posts for the specified username.
- **Nodes Involved:**  
  - Scrape Details

##### Node Details:

- **Scrape Details**  
  - Type: HTTP Request  
  - Role: Calls Apify API to scrape Instagram profile data.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.apify.com/v2/acts/apify~instagram-profile-scraper/run-sync-get-dataset-items`  
    - Body: JSON containing `usernames` array with the username from the filtered Google Sheet row (`{{$json.User}}`).  
    - Authentication: HTTP Query Auth (Apify API token passed as query parameter).  
  - Credentials: HTTP Query Auth (Apify API token must be configured).  
  - Inputs: Receives user data from Filter node.  
  - Outputs: Passes profile data JSON to Split Out node.  
  - Edge Cases: API token invalid/expired, rate limits, username not found, network timeouts.

---

#### 2.3 Image Extraction and HTTP Fetch

- **Overview:** Extracts individual recent posts from the scraped Instagram data, then fetches the image binaries for each post via HTTP.
- **Nodes Involved:**  
  - Split Out  
  - HTTP Request

##### Node Details:

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of recent posts (`latestPosts`) into individual items for processing.  
  - Configuration:  
    - Field to split out: `latestPosts` (array of posts from Apify scrape).  
  - Inputs: Receives Instagram profile data from Scrape Details node.  
  - Outputs: Sends single post objects to HTTP Request node.  
  - Edge Cases: `latestPosts` missing or empty.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Fetches the actual media (image) binary data from the post’s display URL.  
  - Configuration:  
    - URL: Dynamic expression set to post’s `displayUrl` field.  
    - Method: GET (default).  
    - No authentication or special headers specified.  
  - Inputs: Single post object from Split Out node.  
  - Outputs: Sends image binary data with metadata to AI Agent node.  
  - Edge Cases: Broken or invalid URL, timeouts, large image size issues.

---

#### 2.4 AI Visual Analysis

- **Overview:** Takes the fetched Instagram image binary and sends it to an AI agent running OpenAI GPT-5 for visual content analysis and interpretation.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

##### Node Details:

- **AI Agent**  
  - Type: Langchain Agent (n8n AI agent node)  
  - Role: Accepts image data as input, performs visual analysis, and produces a textual description and analysis.  
  - Configuration:  
    - Text input: "image: data" (indicates image binary input).  
    - System message: Instructs to analyze the content of the image URL and output analysis plus the image.  
    - Passthrough binary images enabled (to process image data directly).  
    - Prompt type: Define (custom prompt).  
  - Inputs: Receives binary image data from HTTP Request node.  
  - Outputs: Outputs AI-generated text analysis.  
  - Edge Cases: AI API errors, unsupported image formats, rate limits, timeout, malformed inputs.

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides the underlying GPT-5 language model service for the AI Agent.  
  - Configuration:  
    - Model: GPT-5 specified (vision-capable).  
    - Credentials: OpenAI API key.  
  - Inputs: Connected as AI language model for AI Agent node.  
  - Outputs: Feeds language model responses to AI Agent.  
  - Edge Cases: API key invalid or expired, network issues, quota exceeded.

---

#### 2.5 Credential and Setup Notes

- **Overview:** Provides embedded instructional sticky notes explaining the setup steps for credentials and workflow usage.
- **Nodes Involved:**  
  - Sticky Note53 (Main overview and description)  
  - Sticky Note1 (Google Sheets OAuth2 setup)  
  - Sticky Note61 (Google Sheets OAuth2 details)  
  - Sticky Note63 (Apify HTTP Query Auth setup)  
  - Sticky Note64 (OpenAI API key setup)

##### Node Details:

- Sticky Notes contain detailed instructions about:  
  - How to configure Google Sheets OAuth2 credentials.  
  - How to obtain and configure the Apify API token and HTTP Query Auth credential in n8n.  
  - How to configure the OpenAI API key credential.  
  - Workflow usage context and contact information for support.

---

### 3. Summary Table

| Node Name                      | Node Type                    | Functional Role                        | Input Node(s)                | Output Node(s)            | Sticky Note                                                                                                   |
|--------------------------------|------------------------------|--------------------------------------|-----------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger               | Starts the workflow manually         | -                           | Get Google Sheet           |                                                                                                              |
| Get Google Sheet                | Google Sheets                | Reads Instagram usernames from sheet | When clicking ‘Execute workflow’ | Filter                    | Sticky Note1, Sticky Note61: Setup Google Sheets OAuth2 credential and select spreadsheet/worksheet           |
| Filter                         | Filter                      | Filters to row 2 (selects user)       | Get Google Sheet             | Scrape Details            |                                                                                                              |
| Scrape Details                 | HTTP Request                | Calls Apify API to scrape Instagram data | Filter                      | Split Out                 | Sticky Note63: Setup Apify HTTP Query Auth credential and API token                                          |
| Split Out                     | Split Out                   | Splits Instagram posts array          | Scrape Details               | HTTP Request              |                                                                                                              |
| HTTP Request                   | HTTP Request                | Fetches image binary from post URL    | Split Out                   | AI Agent                  |                                                                                                              |
| AI Agent                      | Langchain Agent             | Performs image content visual analysis | HTTP Request                | -                         | Sticky Note64: Setup OpenAI API key; uses GPT-5 vision-capable model                                         |
| OpenAI Chat Model              | Langchain OpenAI Chat Model | Provides GPT-5 language model backend | -                           | AI Agent (languageModel input) |                                                                                                              |
| Sticky Note53                 | Sticky Note                 | Workflow overview and description     | -                           | -                         | Explains overall workflow purpose and usage                                                                |
| Sticky Note1                  | Sticky Note                 | Google Sheets OAuth2 setup instructions | -                           | -                         | Provides step-by-step for Google Sheets OAuth2 credential setup                                             |
| Sticky Note61                 | Sticky Note                 | Google Sheets OAuth2 setup details    | -                           | -                         | Additional Google Sheets OAuth2 connection details                                                         |
| Sticky Note63                 | Sticky Note                 | Apify API token and HTTP Query Auth setup | -                           | -                         | Details Apify token setup and HTTP Query Auth credential configuration                                      |
| Sticky Note64                 | Sticky Note                 | OpenAI API key setup instructions     | -                           | -                         | Instructions for OpenAI API key credential setup and model selection                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.  
   - No special parameters; it will start the workflow manually.

2. **Add Google Sheets Node:**  
   - Add a **Google Sheets** node named `Get Google Sheet`.  
   - Configure it to **Read Rows** from your target Google Sheet:  
     - Set Document ID to your Google Sheet containing Instagram usernames.  
     - Set Sheet Name to the appropriate worksheet (e.g., "Sheet1" or by gid).  
   - Connect Google Sheets OAuth2 credential (create under n8n Credentials):  
     - Create Credential → Google Sheets OAuth2.  
     - Authenticate with your Google account and grant access.  
   - Connect `When clicking ‘Execute workflow’` output to this node’s input.

3. **Add Filter Node:**  
   - Add a **Filter** node named `Filter`.  
   - Set condition to filter rows where `row_number` equals `2` (number type).  
   - Connect output of `Get Google Sheet` to input of this Filter node.

4. **Add HTTP Request Node for Apify API:**  
   - Add an **HTTP Request** node named `Scrape Details`.  
   - Set method to POST.  
   - Set URL to: `https://api.apify.com/v2/acts/apify~instagram-profile-scraper/run-sync-get-dataset-items`.  
   - Set body type to JSON; configure body to:  
     ```json
     {
       "usernames": ["{{$json.User}}"]
     }
     ```  
   - Enable sending body as JSON.  
   - Configure HTTP Query Auth credential for Apify API token:  
     - Create Credential → HTTP Query Auth.  
     - Add query param `token=<YOUR_APIFY_TOKEN>`.  
   - Connect Filter node output to this HTTP Request.

5. **Add Split Out Node:**  
   - Add **Split Out** node named `Split Out`.  
   - Configure it to split the field named `latestPosts` (which contains the array of recent posts).  
   - Connect `Scrape Details` output to this node’s input.

6. **Add HTTP Request Node for Image Fetching:**  
   - Add **HTTP Request** node named `HTTP Request`.  
   - Configure URL dynamically with an expression to use the post’s `displayUrl` field: `={{ $json.displayUrl }}`.  
   - Use GET method (default).  
   - Connect `Split Out` output to this node’s input.

7. **Add Langchain AI Agent Node:**  
   - Add **AI Agent** node named `AI Agent`.  
   - Configure text input as `image: data` to pass image binary for analysis.  
   - Set system message prompt to instruct analysis of the image content and output the image plus analysis.  
   - Enable passthrough of binary images for processing.  
   - Set prompt type as Define (custom prompt).  
   - Connect `HTTP Request` node output to this node’s input.

8. **Add OpenAI Chat Model Node:**  
   - Add **OpenAI Chat Model** node named `OpenAI Chat Model`.  
   - Set model to GPT-5 (or your preferred vision-capable OpenAI model).  
   - Connect OpenAI API credential (create under n8n Credentials → OpenAI API).  
   - Connect this node to the AI Agent node’s `ai_languageModel` input.

9. **Connect Workflow:**  
   - Connect nodes as follows:  
     Manual Trigger → Get Google Sheet → Filter → Scrape Details → Split Out → HTTP Request → AI Agent  
     OpenAI Chat Model → AI Agent (language model input)  

10. **Test Workflow:**  
    - Trigger manually.  
    - Verify usernames are read from Google Sheets.  
    - Confirm Instagram data is fetched.  
    - Ensure images are requested correctly.  
    - Check AI Agent returns image analysis.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| ### Scrape Instagram images with Apify and analyze them using OpenAI in n8n for fast visual insights (Google Sheets + HTTP + OpenAI). Pull recent Instagram post media for any username, fetch the image binaries, and run automated visual analysis with OpenAI — all orchestrated in n8n. Use a Google Sheet to supply target usernames and store results anywhere you like. | Workflow purpose and general description                                                           |
| Setup instructions for Google Sheets OAuth2 credential, Apify HTTP Query Auth with API token, and OpenAI API key for GPT-5 model integration.                                                                                                                                                                                                                                                    | See sticky notes for detailed credential setup steps                                               |
| Contact for customization help: Email rbreen@ynteractive.com, LinkedIn: [Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/), Website: [ynteractive.com](https://ynteractive.com)                                                                                                                                                                                                       | Support and customization contact information                                                     |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.