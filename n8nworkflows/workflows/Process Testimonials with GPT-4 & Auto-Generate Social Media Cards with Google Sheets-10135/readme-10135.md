Process Testimonials with GPT-4 & Auto-Generate Social Media Cards with Google Sheets

https://n8nworkflows.xyz/workflows/process-testimonials-with-gpt-4---auto-generate-social-media-cards-with-google-sheets-10135


# Process Testimonials with GPT-4 & Auto-Generate Social Media Cards with Google Sheets

### 1. Workflow Overview

This n8n workflow automates the processing of customer testimonials submitted via a webhook. It validates and cleans incoming data, enhances testimonial text using OpenAI's GPT-4 Turbo, generates a styled HTML testimonial card, converts it to an image, uploads that image to Google Drive, updates a Google Sheets database, and sends Slack notifications for both new submissions and social media posting readiness. It also includes a scheduled approval checking mechanism to identify testimonials approved for social media posting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initial Validation:** Receives testimonial data via webhook, validates and cleans it.
- **1.2 AI Text Enhancement:** Uses OpenAI GPT-4 Turbo to polish the testimonial text.
- **1.3 HTML Template Generation & Image Conversion:** Creates an HTML card for the testimonial and converts it to a PNG image.
- **1.4 Cloud Upload & Database Update:** Uploads the image to Google Drive and updates Google Sheets with testimonial metadata.
- **1.5 Notification & Approval Workflow:** Sends Slack notifications for new testimonials and monitors approved testimonials for social media posting.
- **1.6 Scheduled Approval Check & Posting Preparation:** Periodically checks Google Sheets for approved but not posted testimonials, then notifies for manual social media posting.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initial Validation

**Overview:**  
This block receives testimonial submissions via a webhook, extracts and validates key fields, cleans data, and sets default values or fallbacks for missing optional fields.

**Nodes Involved:**  
- Sticky Note (Intro)  
- Webhook Trigger  
- Sticky Note1 (Data Validation description)  
- Data Validation (Code node)

**Node Details:**

- **Webhook Trigger**  
  - Type: Webhook  
  - Role: Entry point for POST submissions with testimonial data  
  - Config: Path = `/testimonial-webhook`, HTTP method = POST, Response mode = last node  
  - Inputs: External webhook request  
  - Outputs to: Data Validation node  
  - Edge cases: Missing or malformed payloads, unexpected field names  
  - Notes: Expected fields include name, designation/role, testimonial text, photo URL, and email.

- **Data Validation (Code)**  
  - Type: Code (JavaScript)  
  - Role: Cleans and validates input data; normalizes field names; sets defaults  
  - Key logic:  
    - Extracts fields with fallback naming (e.g., name or Name)  
    - Validates presence and length of name and testimonial (minimum length 2 for name, 10 for testimonial)  
    - Sets default designation ("Valued Customer") if missing  
    - Generates fallback avatar URL using UI Avatars if photo URL missing or invalid  
    - Sanitizes testimonial text by removing HTML tags and extra spaces to prevent XSS  
  - Input: Webhook payload  
  - Output: Cleaned testimonial objects with fields: name, designation, testimonial, photoUrl, email, timestamp, originalLength, source  
  - Failure modes: Throws error if no valid testimonials remain after filtering  
  - Version: Code node v2  
  - Edge cases: Missing required fields, invalid URLs for photo, empty testimonial text

#### 1.2 AI Text Enhancement

**Overview:**  
This block sends the cleaned testimonial text to OpenAI GPT-4 Turbo to improve grammar, spelling, and readability while preserving original tone and length.

**Nodes Involved:**  
- Sticky Note2 (AI Enhancement instructions)  
- Set AI Prompt  
- Sticky Note3 (OpenAI processing explanation)  
- OpenAI Enhancement (Langchain OpenAI node)  
- Sticky Note4 (Extract AI response description)  
- Extract AI Response (Code node)

**Node Details:**

- **Set AI Prompt**  
  - Type: Set  
  - Role: Constructs prompt string for OpenAI with instructions and original testimonial embedded  
  - Key expression: uses `{{ $json.testimonial }}` to insert testimonial text  
  - Output: JSON with key `prompt` containing the AI instruction and testimonial  
  - Input: Cleaned data from Data Validation  
  - Output to: OpenAI Enhancement node

- **OpenAI Enhancement**  
  - Type: Langchain OpenAI node  
  - Role: Calls OpenAI GPT-4 Turbo Preview model with prompt  
  - Config: Model = "gpt-4-turbo-preview", Max tokens = 300, Temperature = 0.3  
  - Credentials: OpenAI API key required  
  - Input: Prompt string from Set AI Prompt  
  - Output: Raw AI response with improved testimonial text  
  - Edge cases: API rate limits, auth failures, network issues

- **Extract AI Response (Code)**  
  - Type: Code (JavaScript)  
  - Role: Extracts enhanced testimonial text from multiple possible OpenAI response formats; merges with original data; adds metadata fields (`enhanced`, `aiModel`)  
  - Input: OpenAI response and previous node data  
  - Output: JSON including original testimonial, enhanced testimonial text, and enhancement metadata  
  - Failure modes: Fallback to original testimonial if extraction fails

#### 1.3 HTML Template Generation & Image Conversion

**Overview:**  
Generates a visually appealing HTML testimonial card and converts it into a PNG image using an external API.

**Nodes Involved:**  
- Sticky Note5 (HTML template generation explanation)  
- Generate HTML Template (Code node)  
- Sticky Note6 (HTML to Image conversion service info)  
- HTML/CSS to Image (htmlcsstoimage node)  
- Sticky Note7 (Download image explanation)  
- Download Image (HTTP Request)

**Node Details:**

- **Generate HTML Template (Code)**  
  - Type: Code (JavaScript)  
  - Role: Produces a fully styled HTML document with testimonial text, author photo or fallback initials, 5-star rating, and branded styling.  
  - Key features: Uses Google Fonts Inter, gradient backgrounds, XSS-safe text escaping, image error fallback to initials.  
  - Outputs: `html` content and a unique `fileName` for the testimonial image  
  - Input: Enhanced testimonial data with author info and photo URL  
  - Output to: HTML/CSS to Image node

- **HTML/CSS to Image**  
  - Type: htmlcsstoimage  
  - Role: Converts HTML content to PNG image at 800x600 pixels  
  - Config: Uses API key for https://htmlcsstoimg.com  
  - Input: HTML content from previous node  
  - Output: JSON with image URL  
  - Edge cases: API failures, rate limits, invalid HTML

- **Download Image (HTTP Request)**  
  - Type: HTTP Request  
  - Role: Downloads PNG image binary from URL returned by HTML/CSS to Image node  
  - Config: GET method, binary response format  
  - Input: Image URL  
  - Output: Binary PNG data for upload  
  - Edge cases: Network errors, invalid URL

#### 1.4 Cloud Upload & Database Update

**Overview:**  
Uploads the PNG testimonial image to a Google Drive folder and updates a Google Sheets database with testimonial metadata and image details.

**Nodes Involved:**  
- Sticky Note8 (Google Drive upload explanation)  
- Upload to Google Drive  
- Sticky Note9 (Database update explanation)  
- Update Google Sheet  
- Sticky Note10 (Slack notification about new testimonial)  
- Send Slack Notification

**Node Details:**

- **Upload to Google Drive**  
  - Type: Google Drive node  
  - Role: Uploads PNG binary file to a specified Drive folder with dynamic filename  
  - Config: Folder ID and Drive ID configured; filename taken from HTML template node  
  - Credentials: Google Drive OAuth2 credentials needed  
  - Input: Binary PNG from Download Image node  
  - Output: Metadata including shareable webViewLink and Drive file ID  
  - Edge cases: Authentication failures, quota limits

- **Update Google Sheet**  
  - Type: Google Sheets node  
  - Role: Appends or updates testimonial record in a specific sheet  
  - Config: Matches records by Drive ID; fields include name, email, enhanced status, testimonial text, image link, timestamp, source, and status "Pending Approval"  
  - Credentials: Google Sheets OAuth2 credentials required  
  - Edge cases: API limits, permission issues, data mapping errors

- **Send Slack Notification**  
  - Type: Slack node  
  - Role: Sends formatted Slack message alerting team of new testimonial submission with details and image preview link  
  - Config: Channel ID, Slack API credentials needed  
  - Content includes customer info, enhanced and original testimonial text, database status, and action checklist  
  - Edge cases: Slack API rate limits, credential expiration

#### 1.5 Notification & Approval Workflow

**Overview:**  
Sends Slack notifications and manages testimonial approval by monitoring the Google Sheet and filtering approved testimonials for social media posting.

**Nodes Involved:**  
- Sticky Note11 (Schedule trigger explanation)  
- Every 5 Minutes (Schedule Trigger)  
- Sticky Note12 (Read database description)  
- Read Google Sheet  
- Sticky Note13 (Conditional filter explanation)  
- IF Approved & Not Posted (If node)  
- Sticky Note14 (Social media ready notification)  
- Notify Ready to Post (Slack node)  
- Sticky Note15 (Mark as processed explanation)  
- Mark as Posted (Google Sheets update node)

**Node Details:**

- **Every 5 Minutes**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow every 5 minutes to check for approved testimonials to post  
  - Config: Interval = 5 minutes

- **Read Google Sheet**  
  - Type: Google Sheets node  
  - Role: Reads all testimonial records including status, posted status, and Drive ID  
  - Input: None (triggered by schedule)  
  - Output: All rows for filtering

- **IF Approved & Not Posted**  
  - Type: If node  
  - Role: Filters records to only those with status = "Approved" and "Posted to Social" field empty (i.e., not posted)  
  - Output:  
    - True: Testimonials ready to post  
    - False: Skip

- **Notify Ready to Post (Slack)**  
  - Type: Slack node  
  - Role: Sends Slack notification with customer and testimonial info, image preview, and ready-to-post social media text with hashtags  
  - Credentials: Slack API key required

- **Mark as Posted (Google Sheets Update)**  
  - Type: Google Sheets node  
  - Role: Updates testimonial record to mark "Posted to Social" as "Ready - Manual Post Required" and sets "Posted At" timestamp  
  - Matching done by Drive ID  
  - Input: Data from IF node true output

#### 1.6 Setup & Testing Notes (Non-executable nodes)

- Sticky Notes with instructions on credential setup (OpenAI, HTML/CSS to Image, Google Drive, Google Sheets, Slack)  
- Sticky Note with example test JSON payload and expected results to verify workflow functionality

---

### 3. Summary Table

| Node Name             | Node Type                        | Functional Role                        | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                                               |
|-----------------------|---------------------------------|-------------------------------------|-----------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Sticky Note           | stickyNote                      | Workflow start info                  |                             |                             | ## 🚀 WORKFLOW START: TESTIMONIAL COLLECTION... Expected Data: name, designation, testimonial_text, photo_url, email         |
| Webhook Trigger       | webhook                        | Receives testimonial submissions    |                             | Data Validation              |                                                                                                                           |
| Sticky Note1          | stickyNote                      | Data validation explanation          |                             |                             | ## ✅ DATA VALIDATION & CLEANING... Extracts data, validates, cleans, prevents XSS                                          |
| Data Validation       | code                           | Cleans and validates input data      | Webhook Trigger             | Set AI Prompt                |                                                                                                                           |
| Sticky Note2          | stickyNote                      | AI enhancement instructions          |                             |                             | ## 🤖 AI ENHANCEMENT SETUP... Fix grammar, keep natural, no fake details                                                   |
| Set AI Prompt         | set                            | Constructs prompt for OpenAI         | Data Validation             | OpenAI Enhancement           |                                                                                                                           |
| Sticky Note3          | stickyNote                      | OpenAI processing explanation        |                             |                             | ## 🧠 OPENAI PROCESSING... Sends prompt to GPT-4 Turbo, improves text                                                      |
| OpenAI Enhancement    | langchain.openAi               | Calls OpenAI model                   | Set AI Prompt               | Extract AI Response          |                                                                                                                           |
| Sticky Note4          | stickyNote                      | AI response extraction explanation   |                             |                             | ## 📦 EXTRACT AI RESPONSE... Extracts enhanced text, merges with original                                                  |
| Extract AI Response   | code                           | Extracts AI improved testimonial     | OpenAI Enhancement          | Generate HTML Template       |                                                                                                                           |
| Sticky Note5          | stickyNote                      | HTML template generation explanation |                             |                             | ## 🎨 HTML TEMPLATE GENERATION... Creates testimonial card with styling                                                   |
| Generate HTML Template| code                           | Generates HTML testimonial card      | Extract AI Response         | HTML/CSS to Image            |                                                                                                                           |
| Sticky Note6          | stickyNote                      | HTML to image conversion explanation |                             |                             | ## 📸 HTML TO IMAGE CONVERSION... Converts HTML to PNG image                                                              |
| HTML/CSS to Image     | htmlcsstoimage                 | Converts HTML to PNG image           | Generate HTML Template      | Download Image               |                                                                                                                           |
| Sticky Note7          | stickyNote                      | Image download explanation           |                             |                             | ## ⬇️ DOWNLOAD IMAGE... Fetches PNG and prepares for Drive upload                                                         |
| Download Image        | httpRequest                    | Downloads binary PNG from URL        | HTML/CSS to Image           | Upload to Google Drive       |                                                                                                                           |
| Sticky Note8          | stickyNote                      | Google Drive upload explanation      |                             |                             | ## ☁️ GOOGLE DRIVE UPLOAD... Uploads image, sets filename, returns link                                                  |
| Upload to Google Drive| googleDrive                   | Uploads PNG to Drive folder          | Download Image              | Update Google Sheet          |                                                                                                                           |
| Sticky Note9          | stickyNote                      | Database update explanation          |                             |                             | ## 📊 DATABASE UPDATE... Saves testimonial data, status, links                                                           |
| Update Google Sheet   | googleSheets                  | Appends/updates testimonial record   | Upload to Google Drive      | Send Slack Notification     |                                                                                                                           |
| Sticky Note10         | stickyNote                      | Slack notification explanation       |                             |                             | ## 📢 SLACK NOTIFICATION... Includes testimonial details, image preview, status                                           |
| Send Slack Notification| slack                         | Sends notification of new testimonial| Update Google Sheet         |                             |                                                                                                                           |
| Sticky Note11         | stickyNote                      | Approval check schedule explanation  |                             |                             | ## ⏰ SCHEDULE TRIGGER - APPROVAL CHECKER... Checks every 5 minutes                                                        |
| Every 5 Minutes       | scheduleTrigger               | Triggers approval check periodically |                             | Read Google Sheet           |                                                                                                                           |
| Sticky Note12         | stickyNote                      | Database read explanation            |                             |                             | ## 📖 READ DATABASE... Retrieves testimonials and statuses                                                                |
| Read Google Sheet     | googleSheets                  | Reads testimonial data from Sheet    | Every 5 Minutes            | IF Approved & Not Posted     |                                                                                                                           |
| Sticky Note13         | stickyNote                      | Conditional filtering explanation    |                             |                             | ## ✔️ CONDITIONAL FILTER... Passes approved and not posted testimonials                                                   |
| IF Approved & Not Posted| if                            | Filters testimonials for posting     | Read Google Sheet           | Notify Ready to Post         |                                                                                                                           |
| Sticky Note14         | stickyNote                      | Ready-to-post notification explanation|                             |                             | ## 📱 SOCIAL MEDIA READY NOTIFICATION... Contains post-ready details                                                     |
| Notify Ready to Post  | slack                         | Sends Slack alert for social posting | IF Approved & Not Posted    | Mark as Posted              |                                                                                                                           |
| Sticky Note15         | stickyNote                      | Mark as processed explanation        |                             |                             | ## 🔄 MARK AS PROCESSED... Updates record marking testimonial as posted                                                   |
| Mark as Posted        | googleSheets                  | Updates Google Sheet with posting status| Notify Ready to Post        |                             |                                                                                                                           |
| Sticky Note16         | stickyNote                      | Credential setup instructions        |                             |                             | ## 📋 **SETUP CREDENTIALS FIRST**... Lists all required API keys and setup instructions                                   |
| Sticky Note17         | stickyNote                      | Testing instructions and sample data |                             |                             | ## 🧪 **TEST WORKFLOW**... Sample JSON payload and expected outcomes                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `testimonial-webhook`  
   - Response Mode: Last Node  
   - Purpose: Receive incoming testimonial submissions.

2. **Add Code Node for Data Validation**  
   - Type: Code (JavaScript)  
   - Paste the validation script that:  
     - Extracts fields (name, designation, testimonial, photo URL, email) from various possible keys  
     - Validates required fields (name min length 2, testimonial min length 10)  
     - Sets defaults (designation = "Valued Customer")  
     - Generates fallback avatar URL if photo missing or invalid  
     - Cleans testimonial text (removes HTML tags and extra spaces)  
     - Throws error if no valid testimonials remain  
   - Connect Webhook Trigger output to this node.

3. **Add Set Node to Construct AI Prompt**  
   - Type: Set  
   - Create a string field `prompt` with value:  
     ```
     You are a professional editor. Polish the following testimonial to be grammatically perfect while maintaining the original tone, enthusiasm, and authenticity. Keep it natural and conversational. Do not add fake details or change the meaning. Keep the length similar to the original.

     Original testimonial:
     {{ $json.testimonial }}

     Return ONLY the improved testimonial text, nothing else. No quotes, no preamble.
     ```
   - Connect Data Validation output here.

4. **Add OpenAI Node (Langchain OpenAI)**  
   - Type: Langchain OpenAI  
   - Model: gpt-4-turbo-preview  
   - Max Tokens: 300  
   - Temperature: 0.3  
   - Message content: Use the `prompt` field from previous node  
   - Credentials: Configure with your OpenAI API key  
   - Connect Set AI Prompt output here.

5. **Add Code Node to Extract AI Response**  
   - Type: Code  
   - Paste extraction script that:  
     - Parses various OpenAI response formats to extract enhanced testimonial text  
     - Merges enhanced testimonial with original data  
     - Adds flags for enhanced and aiModel  
     - Falls back to original testimonial if extraction fails  
   - Connect OpenAI node output here.

6. **Add Code Node to Generate HTML Template**  
   - Type: Code  
   - Paste HTML generation script:  
     - Creates a styled testimonial card with gradient background, star rating, author photo/fallback initials, and brand logo  
     - Escapes HTML special characters to prevent XSS  
     - Generates unique filename from customer name and timestamp  
   - Connect Extract AI Response output here.

7. **Add HTML/CSS to Image Node**  
   - Type: htmlcsstoimage  
   - HTML Content: Use `{{ $json.html }}` from previous node  
   - Credentials: Configure with your htmlcsstoimg.com API key  
   - Connect Generate HTML Template output here.

8. **Add HTTP Request Node to Download Image**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use `{{ $json.image_url }}` from previous node  
   - Response Format: File/Binary  
   - Connect HTML/CSS to Image output here.

9. **Add Google Drive Node to Upload Image**  
   - Type: Google Drive  
   - Operation: Upload  
   - File Name: Use filename from Generate HTML Template node  
   - Folder ID: Set to your "testimonial data" folder ID  
   - Credentials: Google Drive OAuth2 credentials required  
   - Connect Download Image output here.

10. **Add Google Sheets Node to Update Database**  
    - Type: Google Sheets  
    - Operation: Append or Update (match by Drive ID)  
    - Sheet Name and Document ID: Your testimonial Google Sheet and tab  
    - Map columns: Name, Email, Designation, Testimonial, Original Testimonial, Image Link, Status ("Pending Approval"), Enhanced, Timestamp, Source, etc.  
    - Credentials: Google Sheets OAuth2 credentials required  
    - Connect Upload to Google Drive output here.

11. **Add Slack Node to Send New Testimonial Notification**  
    - Type: Slack  
    - Configure channel and message with testimonial details, image preview link, status, and action steps  
    - Credentials: Slack API key required  
    - Connect Update Google Sheet output here.

12. **Create Schedule Trigger Node for Approval Checking**  
    - Type: Schedule Trigger  
    - Interval: Every 5 minutes

13. **Add Google Sheets Node to Read Testimonials**  
    - Type: Google Sheets  
    - Reads all testimonial records from the database sheet  
    - Credentials: Google Sheets OAuth2  
    - Connect Schedule Trigger here.

14. **Add If Node to Filter Approved and Not Yet Posted**  
    - Condition:  
      - Status equals "Approved"  
      - Posted to Social field is empty or not "Yes"  
    - Connect Read Google Sheet output here.

15. **Add Slack Node to Notify Ready to Post**  
    - Send Slack message with details and ready-to-post social media text with hashtags  
    - Credentials: Slack API key  
    - Connect IF node True output here.

16. **Add Google Sheets Node to Mark as Posted**  
    - Update testimonial record to set "Posted to Social" to "Ready - Manual Post Required" and "Posted At" timestamp  
    - Match by Drive ID  
    - Connect Slack notify node here.

17. **Set up API Credentials**  
    - OpenAI API key (https://platform.openai.com)  
    - HTML/CSS to Image API key (https://htmlcsstoimg.com)  
    - Google Drive OAuth2 (Google Cloud Console)  
    - Google Sheets OAuth2 (same Google Cloud project)  
    - Slack API key (https://api.slack.com/apps)

18. **Test Workflow**  
    - Use provided sample JSON payload via webhook POST with proper headers (`Content-Type: application/json`)  
    - Verify successful processing, image upload, Google Sheets update, and Slack notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Setup credentials carefully before running workflow: OpenAI, HTML/CSS to Image, Google Drive, Google Sheets, Slack | See Sticky Note16 - detailed setup instructions                                                    |
| Sample test JSON payload for webhook submission included in Sticky Note17                                        | Use this to test the workflow: `{ "name": "Sarah Johnson", "designation": "Marketing Director", "photo_url": "https://i.pravatar.cc/400?img=5", "testimonial_text": "Working with this team was amazing!", "email": "sample@gmail.com" }` |
| Workflow uses GPT-4 Turbo Preview model for AI enhancement                                                      | Requires OpenAI API access                                                                          |
| Image generation uses https://htmlcsstoimg.com service                                                          | Requires API key                                                                                   |
| Google Drive uploads images to a specified folder with dynamic filenames                                         | Folder ID must be configured                                                                       |
| Google Sheets database tracks testimonial metadata and posting status                                           | Sheet ID and tab name must be configured                                                          |
| Slack notifications assist team in reviewing and posting testimonials                                           | Slack channel ID and API credentials required                                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. All data handled is legal and public, and the workflow complies with current content policies.