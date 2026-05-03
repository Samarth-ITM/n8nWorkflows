Generate & Schedule Social Media Content with GPT-4 and Buffer from Google Sheets

https://n8nworkflows.xyz/workflows/generate---schedule-social-media-content-with-gpt-4-and-buffer-from-google-sheets-7517


# Generate & Schedule Social Media Content with GPT-4 and Buffer from Google Sheets

### 1. Workflow Overview

This workflow automates the generation and scheduling of social media content across multiple platforms using AI (OpenAI GPT-4 and DALL-E) in combination with Google Sheets as a content calendar and Buffer for scheduling posts. It is designed for marketing teams, content creators, small businesses, and agencies to streamline consistent social media presence.

The workflow logic is grouped into the following functional blocks:

- **1.1 Trigger & Configuration Setup**: Daily scheduled trigger initiates the workflow and sets key configuration variables.
- **1.2 Content Calendar Reading & Filtering**: Reads the content calendar from Google Sheets and filters content scheduled for the current day.
- **1.3 AI Content Generation**: Uses OpenAI GPT-4 to generate platform-specific social media content based on filtered topics.
- **1.4 Content Processing & Scheduling**: Parses AI-generated content, validates it, and schedules posts via Buffer while updating Google Sheets with status.
- **1.5 No Content Handling**: Handles cases where no content is scheduled for the day by sending an appropriate JSON response.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration Setup

- **Overview:** This block initiates the workflow daily at 9 AM and sets essential configuration variables such as Google Sheet IDs and brand details.
- **Nodes Involved:**  
  - Daily Content Generation Trigger  
  - Configuration Variables  

- **Node Details:**

  - **Daily Content Generation Trigger**  
    - Type: Cron Trigger  
    - Config: Runs daily at 9:00 AM (cron expression `0 9 * * *`)  
    - Inputs: None (trigger node)  
    - Outputs: Initiates Configuration Variables  
    - Failures: Cron misconfiguration or execution downtime  
    - Version: 1  

  - **Configuration Variables**  
    - Type: Set node  
    - Config: Defines variables including `GOOGLE_SHEET_ID`, `CONTENT_WORKSHEET`, `BRAND_VOICE`, `COMPANY_NAME`, `TARGET_AUDIENCE`  
    - Inputs: From trigger  
    - Outputs: Passes config to Google Sheets read node  
    - Failures: Misconfigured values or missing variables  
    - Version: 3.2  

#### 2.2 Content Calendar Reading & Filtering

- **Overview:** Reads the full content calendar from Google Sheets and filters posts scheduled for today that are pending and valid.
- **Nodes Involved:**  
  - Read Content Calendar (Sticky Note)  
  - Read Content Calendar (Google Sheets)  
  - Filter Content Note (Sticky Note)  
  - Filter Today's Content (Code)  
  - Check if Content Exists (If)  

- **Node Details:**

  - **Read Content Calendar (Sticky Note)**  
    - Function: Documentation on expected sheet columns and data  
    - Position: Informational only  

  - **Read Content Calendar (Google Sheets)**  
    - Type: Google Sheets node  
    - Config: Reads columns A through G from the sheet named as per `CONTENT_WORKSHEET` variable in the Google Sheet identified by `GOOGLE_SHEET_ID`  
    - Credentials: Google Sheets OAuth2 required  
    - Inputs: From Configuration Variables  
    - Outputs: Raw calendar data to code filter node  
    - Failures: OAuth errors, sheet access errors, invalid sheet name or ID  
    - Version: 4.4  

  - **Filter Content Note (Sticky Note)**  
    - Function: Documentation for filtering logic  
    - Position: Informational only  

  - **Filter Today's Content (Code)**  
    - Type: Code node (JavaScript)  
    - Config: Filters rows for today's date where status is not 'Completed' or 'Posted' and topic is non-empty. Converts date formats if needed.  
    - Inputs: Raw calendar rows  
    - Outputs: Array of content items scheduled for today or a single message if none found  
    - Key expressions: Uses current date in ISO format, accesses configuration variables for brand voice, company, audience  
    - Failures: Date parsing errors, empty data, expression failures  
    - Version: 2  

  - **Check if Content Exists (If)**  
    - Type: If node  
    - Config: Checks if filtered content has `hasContent` true  
    - Inputs: Filtered content array  
    - Outputs:  
      - True branch: Proceed to AI generation  
      - False branch: No content response  
    - Failures: Logical errors in condition, empty input  
    - Version: 2  

#### 2.3 AI Content Generation

- **Overview:** For each filtered content item, generates platform-optimized social media posts via OpenAI GPT-4 with detailed instructions for Twitter, LinkedIn, and Instagram.
- **Nodes Involved:**  
  - AI Content Note (Sticky Note)  
  - Generate Social Content with AI (OpenAI Chat)  

- **Node Details:**

  - **AI Content Note (Sticky Note)**  
    - Function: Explains AI content generation details per platform, including character limits, tone, and hashtags  
    - Position: Informational only  

  - **Generate Social Content with AI**  
    - Type: OpenAI Chat node (Langchain)  
    - Config:  
      - Model: GPT-4  
      - Max tokens: 1000  
      - Temperature: 0.7  
      - System message: Expert social media content creator instructions for Twitter, LinkedIn, Instagram with JSON output format  
      - User message: Injects dynamic variables (topic, content type, brand voice, company, target audience, keywords, platforms) via expressions  
    - Credentials: OpenAI API key required  
    - Inputs: Filtered content items  
    - Outputs: AI-generated JSON content for each platform  
    - Failures: API quota/execution errors, malformed JSON response, timeout  
    - Version: 1.3  

#### 2.4 Content Processing & Scheduling

- **Overview:** Parses AI output, validates and formats content per platform, then schedules posts via Buffer API and updates Google Sheets with status and content snippet.
- **Nodes Involved:**  
  - Process & Schedule Note (Sticky Note)  
  - Process Generated Content (Code)  
  - Schedule Post via Buffer (HTTP Request)  
  - Update Sheet Status (Google Sheets)  

- **Node Details:**

  - **Process & Schedule Note (Sticky Note)**  
    - Function: Describes processing steps including parsing AI response, character validation, scheduling, and spreadsheet updates  
    - Position: Informational only  

  - **Process Generated Content**  
    - Type: Code node (JavaScript)  
    - Config:  
      - Parses AI JSON response safely, falls back to default content if parsing fails  
      - Extracts requested platforms, maps to social content  
      - Outputs array of platform-specific objects with content, hashtags, topic, row index, and scheduled timestamp  
      - Contains a small typo in variable name (`requerestedPlatforms` should be `requestedPlatforms`) which may cause runtime error  
    - Inputs: AI-generated content and filtered original data  
    - Outputs: Content objects for scheduling  
    - Failures: JSON parse errors, typo causing undefined variable, empty platforms, malformed AI response  
    - Version: 2  

  - **Schedule Post via Buffer**  
    - Type: HTTP Request node  
    - Config:  
      - URL: Buffer API endpoint `/1/updates/create.json`  
      - Method: POST  
      - Content-Type: `application/x-www-form-urlencoded`  
      - Body parameters: `text` (post content), `profile_ids[]` (Buffer profile ID), `now` (false for scheduled post)  
      - Headers: Content-Type set  
      - Authentication: HTTP header auth with Buffer API token  
      - Note: `profile_ids[]` needs to be replaced with actual Buffer profile IDs for the target social accounts  
    - Inputs: From Process Generated Content  
    - Outputs: Buffer API response to Google Sheets updater  
    - Failures: API auth errors, invalid profile IDs, rate limits, network issues  
    - Version: 4.2  

  - **Update Sheet Status**  
    - Type: Google Sheets node  
    - Config:  
      - Operation: Update row where `Date` column matches current date  
      - Updates columns:  
        - F (Status) to "Scheduled"  
        - G (Generated Content) to first 100 characters of content snippet with ellipsis  
      - Uses same sheet and document ID variables as earlier  
      - Credentials: Google Sheets OAuth2  
    - Inputs: Buffer API scheduling result  
    - Outputs: Final confirmation  
    - Failures: Sheet update conflicts, OAuth errors, mismatched date for update  
    - Version: 4.4  

#### 2.5 No Content Handling

- **Overview:** Returns a JSON response indicating no content is scheduled today.
- **Nodes Involved:**  
  - No Content Response (Respond to Webhook)  

- **Node Details:**

  - **No Content Response**  
    - Type: Respond to Webhook node  
    - Config:  
      - Response type: JSON  
      - Response body includes status "no_content", message, and date  
    - Inputs: From Check if Content Exists (False branch)  
    - Outputs: Ends workflow with JSON response  
    - Failures: Misconfiguration of webhook, response formatting errors  
    - Version: 1  

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                          | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                                           |
|----------------------------|-----------------------------|----------------------------------------|--------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Daily Content Generation Trigger | Cron Trigger               | Initiates daily workflow run            | -                              | Configuration Variables         |                                                                                                                                      |
| Configuration Variables     | Set                         | Defines key constants & config variables | Daily Content Generation Trigger | Read Content Calendar           |                                                                                                                                      |
| Read Calendar Note          | Sticky Note                 | Describes expected Google Sheets columns | -                              | -                              | 📊 **Step 1: Read Content Calendar** - Explains data columns expected                                                                |
| Read Content Calendar       | Google Sheets               | Reads full content calendar from Sheets | Configuration Variables        | Filter Today's Content          |                                                                                                                                      |
| Filter Content Note         | Sticky Note                 | Explains filtering logic                 | -                              | -                              | 🎯 **Step 2: Filter Today's Content** - Criteria for filtering today's valid posts                                                    |
| Filter Today's Content      | Code                        | Filters calendar data for today's posts | Read Content Calendar          | Check if Content Exists         |                                                                                                                                      |
| Check if Content Exists     | If                          | Branches workflow based on content availability | Filter Today's Content          | Generate Social Content with AI; No Content Response |                                                                                                                                      |
| AI Content Note             | Sticky Note                 | Describes AI content generation specifics | -                              | -                              | 🤖 **Step 3: Generate AI Content** - Platform-specific generation details                                                             |
| Generate Social Content with AI | OpenAI Chat               | Generates platform-specific social posts | Check if Content Exists (True)  | Process Generated Content       |                                                                                                                                      |
| Process & Schedule Note     | Sticky Note                 | Explains processing & scheduling steps   | -                              | -                              | ✅ **Step 4: Process & Schedule** - Parsing AI content, scheduling posts, updating sheet, notifications                                |
| Process Generated Content   | Code                        | Parses AI response, prepares scheduling data | Generate Social Content with AI | Schedule Post via Buffer        |                                                                                                                                      |
| Schedule Post via Buffer    | HTTP Request                | Sends post scheduling requests to Buffer | Process Generated Content      | Update Sheet Status             |                                                                                                                                      |
| Update Sheet Status         | Google Sheets               | Updates Google Sheets with post status   | Schedule Post via Buffer       | -                              |                                                                                                                                      |
| Setup Instructions          | Sticky Note                 | Provides setup instructions and credential requirements | -                              | -                              | ⚙️ **CONFIGURATION REQUIRED** - Lists credentials and template setup steps                                                             |
| No Content Response         | Respond to Webhook          | Returns JSON response if no content today | Check if Content Exists (False) | -                              |                                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Name: `Daily Content Generation Trigger`  
   - Type: Cron Trigger  
   - Configuration: Set cron expression to `0 9 * * *` (runs daily at 9 AM)  

2. **Create Set Node for Configuration Variables**  
   - Name: `Configuration Variables`  
   - Type: Set  
   - Assign variables:  
     - `GOOGLE_SHEET_ID`: Your Google Sheet ID  
     - `CONTENT_WORKSHEET`: "Content Calendar" (or your sheet name)  
     - `BRAND_VOICE`: e.g., "professional, engaging, and informative"  
     - `COMPANY_NAME`: Your company name  
     - `TARGET_AUDIENCE`: e.g., "marketing professionals and business owners"  
   - Connect output of Cron Trigger to this node  

3. **Create Google Sheets Read Node**  
   - Name: `Read Content Calendar`  
   - Type: Google Sheets  
   - Credentials: Set up Google Sheets OAuth2 credentials  
   - Document ID: Set to expression: `={{ $json.GOOGLE_SHEET_ID }}` from Configuration Variables  
   - Sheet Name: Expression: `={{ $json.CONTENT_WORKSHEET }}`  
   - Range: `A:G` (columns including Date, Topic, Platforms, Content Type, Keywords, Status, Generated Content)  
   - Connect from `Configuration Variables` node  

4. **Create Code Node to Filter Today's Content**  
   - Name: `Filter Today's Content`  
   - Type: Code  
   - Language: JavaScript  
   - Code: Implement logic to:  
     - Parse the date from each row (handle `MM/DD/YYYY` or `YYYY-MM-DD`)  
     - Filter rows where date is today, topic is not empty, and status is not "Completed" or "Posted"  
     - Output array with content details and configuration variables injected (brand voice, company, etc.)  
   - Connect from Google Sheets Read node  

5. **Create If Node to Check Content Presence**  
   - Name: `Check if Content Exists`  
   - Type: If  
   - Condition: Check if `hasContent` equals `true` in JSON  
   - Connect from Filter Today's Content node  

6. **Create OpenAI Chat Node for Content Generation**  
   - Name: `Generate Social Content with AI`  
   - Type: OpenAI Chat (Langchain)  
   - Credentials: Connect your OpenAI API key  
   - Model: GPT-4  
   - Max Tokens: 1000  
   - Temperature: 0.7  
   - System Message: Provide detailed instructions for Twitter, LinkedIn, Instagram content with formatting and JSON output  
   - User Message: Template with expressions to insert topic, content type, brand voice, company, target audience, keywords, platforms  
   - Connect True output from If node to this node  

7. **Create Code Node to Process AI Content**  
   - Name: `Process Generated Content`  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     - Parse AI JSON response safely, fallback to defaults if error  
     - Extract platforms from original data, map to AI content  
     - Output array of platform-specific content objects with required properties: platform, content, hashtags, topic, row index, date, scheduled time  
   - Connect from AI Content Generation node  

8. **Create HTTP Request Node to Schedule Posts via Buffer**  
   - Name: `Schedule Post via Buffer`  
   - Type: HTTP Request  
   - Authentication: HTTP Header Auth with Buffer API token  
   - Method: POST  
   - URL: `https://api.bufferapp.com/1/updates/create.json`  
   - Headers: `Content-Type`: `application/x-www-form-urlencoded`  
   - Body Parameters (form-urlencoded):  
     - `text`: `={{ $json.content }}`  
     - `profile_ids[]`: Your Buffer profile ID(s)  
     - `now`: `false` to schedule rather than post immediately  
   - Connect from Process Generated Content node  

9. **Create Google Sheets Update Node to Update Status**  
   - Name: `Update Sheet Status`  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2  
   - Document ID, Sheet Name: same as read node (expressions from config)  
   - Operation: Update  
   - Match Column: Date (A) equals current content date  
   - Update Columns:  
     - F: "Scheduled"  
     - G: First 100 characters of generated content with ellipsis (expression)  
   - Connect from Buffer scheduling node  

10. **Create Respond to Webhook Node for No Content Case**  
    - Name: `No Content Response`  
    - Type: Respond to Webhook  
    - Response Type: JSON  
    - Response Body: JSON with status "no_content", message, and date from input  
    - Connect False output from If node (Check if Content Exists)  

11. **Create Sticky Notes**  
    - Add sticky notes at appropriate positions to document steps and instructions as in the original workflow for clarity:  
      - Workflow Instructions (overview)  
      - Read Calendar Note  
      - Filter Content Note  
      - AI Content Note  
      - Process & Schedule Note  
      - Setup Instructions  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow requires Google Sheets with a content calendar template having columns Date, Topic, Platforms, Content Type, Keywords, Status, Generated Content. | Provided in Setup Instructions sticky note      |
| OpenAI API keys are required for GPT-4 chat completion and DALL-E image generation (DALL-E usage is described but not explicitly in nodes here).           | See OpenAI API documentation                     |
| Buffer API credentials and profile IDs are required to schedule posts. Please obtain profile IDs from your Buffer dashboard.                               | Buffer API docs: https://buffer.com/developers  |
| Date values in Google Sheets must be in ISO format (YYYY-MM-DD) or MM/DD/YYYY; code node handles both formats.                                              | Mentioned in Filter Today's Content code node   |
| Ensure OAuth2 credentials for Google Sheets have read/write permissions for the target spreadsheet.                                                        | n8n credential setup                              |
| The code node 'Process Generated Content' contains a typo (`requerestedPlatforms`), which should be corrected to `requestedPlatforms` to avoid runtime errors. | Important fix to apply before production         |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. All processing complies fully with valid content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.