Multi-Platform Social Media Publisher with Blotato, GPT-4 Mini & Airtable

https://n8nworkflows.xyz/workflows/multi-platform-social-media-publisher-with-blotato--gpt-4-mini---airtable-3816


# Multi-Platform Social Media Publisher with Blotato, GPT-4 Mini & Airtable

### 1. Workflow Overview

This workflow automates multi-platform social media publishing using AI-generated content and integrates Airtable, n8n, and Blotato APIs. It supports scheduling and posting videos and images to platforms including Instagram, YouTube, TikTok, Facebook, LinkedIn, Twitter (X), Threads, Bluesky, and Pinterest. The workflow fetches content metadata and media URLs from Airtable, uploads media to Blotato, prepares platform-specific post data, and publishes posts via Blotato’s API. It also includes a Pinterest board ID extraction sub-process.

Logical blocks:

- **1.1 Input Reception & Data Retrieval**: Manual trigger and Airtable record retrieval.
- **1.2 AI Processing**: Ensures valid YouTube video titles using GPT-4 Mini.
- **1.3 Data Preparation**: Sets account IDs and final texts for publishing.
- **1.4 Media Upload**: Uploads images and videos to Blotato.
- **1.5 Multi-Platform Publishing**: Posts content to multiple social media platforms via Blotato.
- **1.6 Post-Publish Status Update**: Updates Airtable records with publishing status.
- **1.7 Pinterest Board ID Extraction**: A form-triggered sub-process to extract Pinterest board IDs from URLs.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Retrieval

**Overview:**  
This block starts the workflow manually and fetches the relevant content record from Airtable for publishing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Airtable Record ID (Set)  
- Airtable (Read Record)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for testing the workflow manually.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Triggers next node.

- **Airtable Record ID**  
  - Type: Set  
  - Role: Sets the Airtable record ID to fetch.  
  - Configuration: Sets a fixed string `"recJBpGmgd7nuLpfe"` as `airtableID` to specify the record to retrieve.  
  - Inputs: Manual Trigger output  
  - Outputs: Passes the record ID to Airtable node.

- **Airtable**  
  - Type: Airtable node (read)  
  - Role: Fetches the content record from Airtable base "Social Media System" and table "Media Creation" using the record ID.  
  - Configuration: Uses personal access token credential; fetches record by ID.  
  - Inputs: Airtable Record ID node  
  - Outputs: JSON with content metadata, including media URLs, text scripts, and social media account IDs.  
  - Edge Cases:  
    - Authentication failure if token invalid.  
    - Record not found or deleted.  
    - Network timeout.

---

#### 1.2 AI Processing

**Overview:**  
Generates a valid, viral YouTube Shorts title using GPT-4 Mini based on the original media title.

**Nodes Involved:**  
- Ensure Valid YouTube Title (OpenAI)

**Node Details:**

- **Ensure Valid YouTube Title**  
  - Type: OpenAI (Langchain) node  
  - Role: Takes the original media title and rewrites it to a viral YouTube Shorts title with max 100 characters.  
  - Configuration:  
    - Model: GPT-4.1-MINI  
    - Prompt: Provides instructions to re-write title for virality, outputting JSON with `youtube_title`.  
  - Inputs: Airtable node output (content metadata)  
  - Outputs: JSON with `youtube_title` property.  
  - Edge Cases:  
    - API quota exceeded or authentication failure.  
    - Model timeout or response format errors.  
    - Input title missing or empty.

---

#### 1.3 Data Preparation

**Overview:**  
Prepares all necessary data for publishing, including social media account IDs and final text content for each platform.

**Nodes Involved:**  
- Prepare for Publish (Set)

**Node Details:**

- **Prepare for Publish**  
  - Type: Set  
  - Role: Aggregates social media account IDs and final text snippets from Airtable and AI nodes into a single JSON object for downstream use.  
  - Configuration:  
    - Sets fixed account IDs for Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Pinterest, Bluesky.  
    - Extracts `final_text_long` and `final_text_short` from Airtable fields `Script` and `Text for X` respectively.  
  - Inputs: Output from Ensure Valid YouTube Title and Airtable  
  - Outputs: JSON object with IDs and text for publishing.  
  - Edge Cases:  
    - Missing or incorrect account IDs.  
    - Missing text fields.

---

#### 1.4 Media Upload

**Overview:**  
Uploads the media files (image and video) from Airtable URLs to Blotato media hosting service.

**Nodes Involved:**  
- Upload Image to Blotato (HTTP Request)  
- Upload Video to Blotato (HTTP Request)

**Node Details:**

- **Upload Image to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads image URL from Airtable to Blotato media endpoint.  
  - Configuration:  
    - POST to `https://backend.blotato.com/v2/media`  
    - Body parameter: `url` set dynamically from Airtable field `Image URL`.  
    - Auth: HTTP Header Auth with Blotato API key.  
  - Inputs: Prepare for Publish node  
  - Outputs: JSON response with hosted media URL.  
  - Edge Cases:  
    - Invalid image URL or inaccessible resource.  
    - API rate limits or authentication failure.  
    - Network errors.

- **Upload Video to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads video URL from Airtable to Blotato media endpoint.  
  - Configuration:  
    - POST to `https://backend.blotato.com/v2/media`  
    - Body parameter: `url` set dynamically from Airtable field `Video URL`.  
    - Auth: HTTP Header Auth with Blotato API key.  
  - Inputs: Upload Image to Blotato node  
  - Outputs: JSON response with hosted media URL.  
  - Edge Cases:  
    - Invalid video URL or inaccessible resource.  
    - API rate limits or authentication failure.  
    - Network errors.

---

#### 1.5 Multi-Platform Publishing

**Overview:**  
Publishes the uploaded media and prepared text content to multiple social media platforms via Blotato API.

**Nodes Involved:**  
- [Instagram] Publish via Blotato (HTTP Request)  
- [Facebook] Publish via Blotato (HTTP Request)  
- [Linkedin] Publish via Blotato (HTTP Request)  
- [Tiktok] Publish via Blotato (HTTP Request)  
- [Youtube] Publish via Blotato (HTTP Request)  
- [Threads] Publish via Blotato (HTTP Request)  
- [Twitter] Publish via Blotato (HTTP Request)  
- [Bluesky] Publish via Blotato (HTTP Request)  
- [Pinterest] Publish via Blotato (HTTP Request)  
- Sticky Note2 (visual label)

**Node Details (Representative):**

- **[Instagram] Publish via Blotato**  
  - Type: HTTP Request  
  - Role: Posts to Instagram via Blotato API.  
  - Configuration:  
    - POST to `https://backend.blotato.com/v2/posts`  
    - JSON body includes `targetType: instagram`, text from `final_text_short`, media URL from uploaded media, and Instagram account ID.  
    - Auth: HTTP Header Auth with Blotato API key.  
  - Inputs: Upload Video to Blotato node  
  - Outputs: API response indicating success or failure.  
  - Edge Cases:  
    - Auth failure or invalid account ID.  
    - Media URL missing or invalid.  
    - API rate limits or platform-specific restrictions.

- **Other platform nodes** follow similar patterns with platform-specific fields such as `pageId` for Facebook, `boardId` for Pinterest, and additional TikTok-specific flags (privacy, duet, stitch, comments).  
- All have `onError` set to `continueErrorOutput` to allow the workflow to continue publishing to other platforms even if one fails.

---

#### 1.6 Post-Publish Status Update

**Overview:**  
Updates the Airtable record to reflect the publishing status after attempting to post on Instagram (with partial setup for other platforms).

**Nodes Involved:**  
- Airtable: Posted Instagram (Update)  
- Airtable: Posted Instagram1 (Update)  
- Sticky Note3 (visual label)

**Node Details:**

- **Airtable: Posted Instagram**  
  - Type: Airtable (update)  
  - Role: Updates the record’s `Production` status to "Completed".  
  - Configuration: Uses record ID from Airtable, updates field `Production` to "Completed".  
  - Inputs: [Instagram] Publish via Blotato node output  
  - Outputs: Updated Airtable record.  
  - Edge Cases:  
    - Airtable API errors or auth failure.  
    - Record locked or deleted.

- **Airtable: Posted Instagram1**  
  - Type: Airtable (update)  
  - Role: Updates the record’s `Production` status to "In progress".  
  - Configuration: Similar to above but sets status to "In progress".  
  - Inputs: [Instagram] Publish via Blotato node output (parallel output)  
  - Outputs: Updated Airtable record.

- **Note:** Sticky Note3 mentions that updating post status for all platforms is incomplete and left for customization.

---

#### 1.7 Pinterest Board ID Extraction

**Overview:**  
Extracts Pinterest board ID from a user-submitted Pinterest board URL using a form trigger and HTML scraping.

**Nodes Involved:**  
- Pinterest System (tm) (Form Trigger)  
- Grab Pinterest Board Page (HTTP Request)  
- Pinterest Page Sleuth (Code)

**Node Details:**

- **Pinterest System (tm)**  
  - Type: Form Trigger  
  - Role: Receives Pinterest Board URL input from user via form.  
  - Configuration: Single required field for Pinterest Board URL.  
  - Inputs: External HTTP form submission  
  - Outputs: JSON with user input URL.

- **Grab Pinterest Board Page**  
  - Type: HTTP Request  
  - Role: Fetches HTML content of the Pinterest board page.  
  - Configuration:  
    - GET request to the submitted Pinterest Board URL.  
    - Custom headers to mimic browser user agent and accept headers.  
  - Inputs: Pinterest System (tm) output  
  - Outputs: HTML page content as string.

- **Pinterest Page Sleuth**  
  - Type: Code (JavaScript)  
  - Role: Parses the HTML content to extract Pinterest board metadata including board ID.  
  - Configuration:  
    - Uses regex to find JSON embedded in a script tag with id `__PWS_INITIAL_PROPS__`.  
    - Parses JSON and extracts board ID, name, description, privacy, pin count, follower count, creation date, and owner info.  
    - Handles errors and logs issues.  
  - Inputs: Grab Pinterest Board Page output  
  - Outputs: JSON with extracted board info or error message.  
  - Edge Cases:  
    - HTML structure changes breaking regex.  
    - Invalid or private boards.  
    - Network or parsing errors.

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                     | Input Node(s)                    | Output Node(s)                                                                                         | Sticky Note                                                                                                         |
|--------------------------------|---------------------------|-----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger            | Workflow entry point for testing  | None                            | Airtable Record ID                                                                                     |                                                                                                                     |
| Airtable Record ID              | Set                       | Sets Airtable record ID to fetch  | When clicking ‘Test workflow’   | Airtable                                                                                              |                                                                                                                     |
| Airtable                       | Airtable                  | Fetches content record             | Airtable Record ID              | Ensure Valid YouTube Title                                                                             |                                                                                                                     |
| Ensure Valid YouTube Title      | OpenAI (Langchain)        | Generates viral YouTube title     | Airtable                       | Prepare for Publish                                                                                     |                                                                                                                     |
| Prepare for Publish             | Set                       | Prepares account IDs and texts    | Ensure Valid YouTube Title      | Upload Image to Blotato                                                                                 |                                                                                                                     |
| Upload Image to Blotato         | HTTP Request              | Uploads image to Blotato           | Prepare for Publish             | Upload Video to Blotato                                                                                 |                                                                                                                     |
| Upload Video to Blotato         | HTTP Request              | Uploads video to Blotato           | Upload Image to Blotato         | [Instagram] Publish via Blotato, [Facebook] Publish via Blotato, [Linkedin] Publish via Blotato, [Tiktok] Publish via Blotato, [Youtube] Publish via Blotato, [Threads] Publish via Blotato, [Twitter] Publish via Blotato, [Bluesky] Publish via Blotato, [Pinterest] Publish via Blotato |                                                                                                                     |
| [Instagram] Publish via Blotato | HTTP Request              | Publishes post to Instagram        | Upload Video to Blotato         | Airtable: Posted Instagram, Airtable: Posted Instagram1                                              | # Publish to Social Media                                                                                           |
| [Facebook] Publish via Blotato  | HTTP Request              | Publishes post to Facebook         | Upload Video to Blotato         |                                                                                                      | # Publish to Social Media                                                                                           |
| [Linkedin] Publish via Blotato  | HTTP Request              | Publishes post to LinkedIn         | Upload Video to Blotato         |                                                                                                      | # Publish to Social Media                                                                                           |
| [Tiktok] Publish via Blotato    | HTTP Request              | Publishes post to TikTok           | Upload Video to Blotato         |                                                                                                      | # Publish to Social Media                                                                                           |
| [Youtube] Publish via Blotato   | HTTP Request              | Publishes post to YouTube          | Upload Video to Blotato         |                                                                                                      | # Publish to Social Media                                                                                           |
| [Threads] Publish via Blotato   | HTTP Request              | Publishes post to Threads          | Upload Video to Blotato         |                                                                                                      | # Publish to Social Media                                                                                           |
| [Twitter] Publish via Blotato   | HTTP Request              | Publishes post to Twitter (X)     | Upload Video to Blotato         |                                                                                                      | # Publish to Social Media                                                                                           |
| [Bluesky] Publish via Blotato   | HTTP Request              | Publishes post to Bluesky          | Upload Video to Blotato         |                                                                                                      | # Publish to Social Media                                                                                           |
| [Pinterest] Publish via Blotato | HTTP Request              | Publishes post to Pinterest        | Upload Video to Blotato         |                                                                                                      | # Publish to Social Media                                                                                           |
| Airtable: Posted Instagram      | Airtable                  | Updates record status to Completed | [Instagram] Publish via Blotato |                                                                                                      | # Update Post Status *note I didn't finish attaching all the platforms, left the labor to y'all :)                 |
| Airtable: Posted Instagram1     | Airtable                  | Updates record status to In progress | [Instagram] Publish via Blotato |                                                                                                      | # Update Post Status *note I didn't finish attaching all the platforms, left the labor to y'all :)                 |
| Pinterest System (tm)           | Form Trigger              | Receives Pinterest Board URL      | None                            | Grab Pinterest Board Page                                                                              | # Pinterest Page Sleuth - Use either testing or active URL respectively depending if your workflow is active or not |
| Grab Pinterest Board Page       | HTTP Request              | Fetches Pinterest board HTML      | Pinterest System (tm)           | Pinterest Page Sleuth                                                                                   | # Pinterest Page Sleuth - Use either testing or active URL respectively depending if your workflow is active or not |
| Pinterest Page Sleuth           | Code                      | Extracts Pinterest board ID       | Grab Pinterest Board Page       |                                                                                                      | # Pinterest Page Sleuth - Use either testing or active URL respectively depending if your workflow is active or not |
| Sticky Note1                   | Sticky Note               | Quick debug links and info        | None                           | None                                                                                                 | Quick Debug Checking - Links to social media for checking post success and Blotato failed posts                    |
| Sticky Note2                   | Sticky Note               | Visual label for publishing block | None                           | None                                                                                                 | # Publish to Social Media                                                                                            |
| Sticky Note3                   | Sticky Note               | Visual label for update post status | None                           | None                                                                                                 | # Update Post Status *note I didn't finish attaching all the platforms, left the labor to y'all :)                 |
| Sticky Note4                   | Sticky Note               | Notes on YouTube title length     | None                           | None                                                                                                 | May Not Be Necessary - Added due to title length concerns                                                          |
| Sticky Note5                   | Sticky Note               | Airtable setup and credential instructions | None                           | None                                                                                                 | Detailed Airtable setup and n8n credential creation instructions with affiliate links                              |
| Sticky Note6                   | Sticky Note               | Blotato affiliate and API key info | None                           | None                                                                                                 | Blotato Affiliate Link, Please Support My Work: https://blotato.com/?ref=max                                      |
| Sticky Note7                   | Sticky Note               | Blotato login and account ID notes | None                           | None                                                                                                 | Instructions on connecting each social media account in Blotato and copying account IDs                            |
| Sticky Note8                   | Sticky Note               | Pinterest Page Sleuth description | None                           | None                                                                                                 | # Pinterest Page Sleuth - Use either testing or active URL respectively depending if your workflow is active or not |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters.

2. **Create Set Node for Airtable Record ID**  
   - Name: `Airtable Record ID`  
   - Type: Set  
   - Add field: `airtableID` (string) with value `"recJBpGmgd7nuLpfe"` (replace with your record ID).  
   - Connect output of Manual Trigger to this node.

3. **Create Airtable Node to Fetch Record**  
   - Name: `Airtable`  
   - Type: Airtable  
   - Operation: Read record by ID  
   - Base: Select your Airtable base (e.g., "Social Media System")  
   - Table: Select your table (e.g., "Media Creation")  
   - Record ID: Use expression to get from previous node: `{{$json.airtableID}}`  
   - Credentials: Create or select Airtable personal access token credential with scopes: `data.records:read`, `data.records:write`, `schema.bases:read`.  
   - Connect output of `Airtable Record ID` to this node.

4. **Create OpenAI Node for YouTube Title**  
   - Name: `Ensure Valid YouTube Title`  
   - Type: OpenAI (Langchain)  
   - Model: GPT-4.1-MINI  
   - Messages:  
     - User content: `CURRENT_TITLE:\n{{ $json['Media Title'] }}`  
     - Assistant content: Task instructions to rewrite title for virality, max 100 chars, output JSON with `youtube_title`.  
   - Enable JSON output.  
   - Credentials: OpenAI API key credential.  
   - Connect output of `Airtable` to this node.

5. **Create Set Node to Prepare Publish Data**  
   - Name: `Prepare for Publish`  
   - Type: Set  
   - Fields (JSON mode):  
     - `instagram_id`: your Instagram account ID (e.g., `"2244"`)  
     - `youtube_id`: your YouTube account ID (e.g., `"1300"`)  
     - `tiktok_id`, `facebook_id`, `facebook_page_id`, `threads_id`, `twitter_id`, `linkedin_id`, `pinterest_id`, `pinterested_board_id`, `bluesky_id`: set accordingly  
     - `final_text_long`: expression `{{ $('Airtable').item.json.Script.toJsonString() }}`  
     - `final_text_short`: expression `{{ $('Airtable').item.json['Text for X'].toJsonString() }}`  
   - Connect output of `Ensure Valid YouTube Title` to this node.

6. **Create HTTP Request Node to Upload Image to Blotato**  
   - Name: `Upload Image to Blotato`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://backend.blotato.com/v2/media`  
   - Body Parameters: `url` set to `{{ $('Airtable').item.json['Image URL'] }}`  
   - Authentication: HTTP Header Auth with Blotato API key credential.  
   - Connect output of `Prepare for Publish` to this node.

7. **Create HTTP Request Node to Upload Video to Blotato**  
   - Name: `Upload Video to Blotato`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://backend.blotato.com/v2/media`  
   - Body Parameters: `url` set to `{{ $('Airtable').item.json['Video URL'] }}`  
   - Authentication: HTTP Header Auth with Blotato API key credential.  
   - Connect output of `Upload Image to Blotato` to this node.

8. **Create HTTP Request Nodes for Each Platform Publish via Blotato**  
   For each platform (Instagram, Facebook, LinkedIn, TikTok, YouTube, Threads, Twitter, Bluesky, Pinterest):  
   - Name: `[Platform] Publish via Blotato`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://backend.blotato.com/v2/posts`  
   - Authentication: HTTP Header Auth with Blotato API key credential.  
   - JSON Body: Construct according to platform requirements, using expressions to insert:  
     - `accountId` from `Prepare for Publish` node (e.g., `{{ $('Prepare for Publish').item.json.instagram_id }}`)  
     - `text` from `final_text_short` or `final_text_long` as appropriate  
     - `mediaUrls` from uploaded media URL (`{{ $json.url }}` or `{{ $('Upload Image to Blotato').item.json.url }}`)  
     - Platform-specific fields (e.g., Facebook `pageId`, Pinterest `boardId`)  
   - Set `onError` to `continueErrorOutput` to allow parallel publishing.  
   - Connect output of `Upload Video to Blotato` to all these publish nodes in parallel.

9. **Create Airtable Update Nodes for Post-Publish Status**  
   - Name: `Airtable: Posted Instagram` and `Airtable: Posted Instagram1`  
   - Type: Airtable (Update)  
   - Base/Table: Same as fetch node  
   - Record ID: Use expression from Airtable fetch node  
   - Fields to update: `Production` status to "Completed" or "In progress" respectively  
   - Connect outputs of `[Instagram] Publish via Blotato` node to these update nodes.

10. **Create Pinterest Board ID Extraction Sub-Workflow**  
    - Form Trigger Node: `Pinterest System (tm)`  
      - Single required field: Pinterest Board URL  
    - HTTP Request Node: `Grab Pinterest Board Page`  
      - GET request to URL from form input  
      - Set headers to mimic browser user agent  
    - Code Node: `Pinterest Page Sleuth`  
      - JavaScript code to parse HTML and extract board ID and metadata  
    - Connect nodes in sequence: Form Trigger → HTTP Request → Code Node

11. **Add Sticky Notes**  
    - Add sticky notes as visual labels and documentation inside the workflow for clarity, including setup instructions, debug links, and platform-specific notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Quick Debug Checking: Contains links to social media profiles for verifying successful posts and a link to Blotato failed posts dashboard. Replace links with your own.                                                                                                                                                                                                                                                                                                                                                           | Sticky Note1 content                                                                                         |
| Publish to Social Media: Visual label grouping all platform publish nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note2 content                                                                                         |
| Update Post Status: Notes that updating post status for all platforms is incomplete and left for user customization.                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note3 content                                                                                         |
| May Not Be Necessary: Added due to concerns about YouTube title length exceeding 100 characters.                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note4 content                                                                                         |
| How to Add Example Table and Connect n8n to Airtable: Detailed step-by-step instructions for setting up Airtable base, creating personal access token with required scopes, and adding credentials in n8n. Includes affiliate signup links.                                                                                                                                                                                                                                                                                      | Sticky Note5 content with affiliate links: https://airtable.com/invite/r/6UyZyAAd and https://n8n.partnerlinks.io/aiwithapex |
| Blotato Affiliate Link: Please support the author’s work by signing up via https://blotato.com/?ref=max. API key required for Blotato nodes.                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note6 content                                                                                         |
| Blotato Login and Account ID Notes: Instructions to log into Blotato, connect each social media platform individually (avoid "connect all pages"), and copy account IDs and page IDs for Facebook and Pinterest board IDs.                                                                                                                                                                                                                                                                                                          | Sticky Note7 content                                                                                         |
| Pinterest Page Sleuth: Explains usage of Pinterest board ID extraction sub-process, recommending using testing or active URLs depending on workflow state.                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note8 content                                                                                         |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting. It highlights key configuration details, node roles, data flow, and potential failure points for robust operation.