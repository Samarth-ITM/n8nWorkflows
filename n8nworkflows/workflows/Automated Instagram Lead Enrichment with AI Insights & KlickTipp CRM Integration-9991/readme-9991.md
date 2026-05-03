Automated Instagram Lead Enrichment with AI Insights & KlickTipp CRM Integration

https://n8nworkflows.xyz/workflows/automated-instagram-lead-enrichment-with-ai-insights---klicktipp-crm-integration-9991


# Automated Instagram Lead Enrichment with AI Insights & KlickTipp CRM Integration

### 1. Workflow Overview

This workflow automates the enrichment of Instagram leads using AI-generated marketing insights and integrates enriched lead data into the KlickTipp CRM system. It targets digital marketers, social media managers, and coaches who engage prospective leads via Instagram DMs and wish to automate personalized outreach, lead enrichment, and segmentation without manual data entry.

The workflow is composed of three main logical blocks:

- **1.1 Data Reception:** Captures Instagram user data submitted through a JotForm linked via Instagram DM and verifies user existence in a Google Sheets matching table.
- **1.2 Data Enrichment:** Enriches the Instagram user profile by fetching detailed profile data via the Facebook Graph API, then generates AI-powered marketing insights through OpenAI, parsing them into structured JSON.
- **1.3 Saving Data:** Conditionally subscribes the enriched lead data to KlickTipp CRM with appropriate tags and mapped custom fields, differentiating between professional and personal Instagram profiles.

Additionally, a complementary **DM Message Flow** triggers personalized Instagram DMs to users based on KlickTipp tag or campaign events, inviting them to fill the JotForm to initiate the enrichment loop.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Reception

- **Overview:**  
  This block listens for form submissions from Instagram users who received a JotForm via DM. It then checks if the Instagram username exists in a Google Sheets matching table to retrieve related Instagram comment IDs, ensuring accurate lead matching.

- **Nodes Involved:**  
  - Listen to submission from Instagram DM (JotForm Trigger)  
  - Look for entry in matching table (Google Sheets)  
  - Get user profile (Facebook Graph API)  
  - Profile type check (Switch)

- **Node Details:**

  1. **Listen to submission from Instagram DM**  
     - Type: JotForm Trigger (Webhook)  
     - Role: Listens for real-time submissions from a specific JotForm sent via Instagram DM after a user comments on a post.  
     - Configuration: Monitors form with ID `252682338805059` using JotForm API credentials.  
     - Key Variables: Captures user data such as name, email, and Instagram username from submission JSON.  
     - Input: HTTP webhook from JotForm submission.  
     - Output: JSON with user form data.  
     - Edge Cases: Missing or malformed form data; webhook failures; API credential expiration.

  2. **Look for entry in matching table**  
     - Type: Google Sheets (Lookup)  
     - Role: Checks if the Instagram username from the form submission exists in a predefined Google Sheet to find associated Instagram comment payload/ID.  
     - Configuration: Uses Google Sheets OAuth2 credentials; filters by "Instagram username" column matching the submitted username; targets Sheet1 (`gid=0`) in a specific Google Spreadsheet URL.  
     - Key Variables: Lookup value: `{{$json.instagram_username}}`.  
     - Input: Output from JotForm trigger.  
     - Output: Matching row data, if found, including Instagram ID comment payload.  
     - Edge Cases: No matching username found (empty result), Google Sheets API rate limits or auth errors.

  3. **Get user profile**  
     - Type: Facebook Graph API  
     - Role: Retrieves Instagram profile details (bio, followers, follows, media count, profile image) for the username from the form submission.  
     - Configuration: Uses Facebook Graph API v23.0 with a Page node `17841465090245583`, requesting `business_discovery` fields for the username variable; uses Facebook Graph API credentials.  
     - Key Variables: Username used in the request: `{{$json.instagram_username}}`.  
     - Input: Output from Google Sheets lookup.  
     - Output: Instagram profile data or error (continues workflow on error).  
     - Edge Cases: Permissions missing (`instagram_basic`, `pages_show_list`, `business_management`), API errors, private or personal profiles with no business discovery data.

  4. **Profile type check**  
     - Type: Switch  
     - Role: Determines if the Instagram profile data fetched is a professional (business) or personal profile by checking the existence of `business_discovery.id`.  
     - Configuration: Two routes:  
       - "Professional profile": if `business_discovery.id` exists (string exists).  
       - "Personal profile": fallback if statusCode exists (number exists).  
     - Input: Output from Facebook Graph API node.  
     - Output: Routes to either "Generate Insights" or fallback subscription node.  
     - Edge Cases: Unexpected data schema could misroute data; null or missing fields.

---

#### 2.2 Data Enrichment

- **Overview:**  
  This block enriches the Instagram profile data by generating actionable marketing insights via an AI agent and structuring the output for easy integration into CRM fields.

- **Nodes Involved:**  
  - Generate Insights (Langchain AI Agent)  
  - OpenAI Chat Model (Language Model)  
  - Structured Output Parser (Langchain JSON Parser)

- **Node Details:**

  1. **Generate Insights**  
     - Type: Langchain AI Agent  
     - Role: Uses AI to analyze Instagram profile data (bio, follower counts, etc.) to produce targeted marketing insights useful for segmentation and personalized messaging.  
     - Configuration:  
       - Input prompt dynamically constructed with Instagram profile info fields: biography, followers_count, follows_count, media_count, profile_picture_url.  
       - Prompt asks for interests, demographics, tone, motivators for email marketing segmentation.  
       - Uses output parser to ensure structured results.  
     - Input: Data routed from "Profile type check" (professional profile path).  
     - Output: AI-generated insights JSON.  
     - Edge Cases: AI timeouts, no useful content generated, API quota limits, prompt failures.

  2. **OpenAI Chat Model**  
     - Type: Langchain OpenAI Chat  
     - Role: Provides the actual GPT-4.1-mini language model backend for AI processing.  
     - Configuration: Model set as `gpt-4.1-mini` with default options, uses OpenAI API credentials.  
     - Input: Text prompt from "Generate Insights" node.  
     - Output: AI-generated text for further parsing.  
     - Edge Cases: API errors, rate limiting, invalid credentials.

  3. **Structured Output Parser**  
     - Type: Langchain Output Parser (Structured JSON)  
     - Role: Parses AI text output into a clean JSON schema matching expected insight structure.  
     - Configuration: Example JSON schema provided as a template for output formatting.  
     - Input: AI text from OpenAI Chat Model.  
     - Output: Structured JSON to be used in later nodes.  
     - Edge Cases: Parsing errors if AI output deviates from schema, malformed JSON.

---

#### 2.3 Saving Data

- **Overview:**  
  This block subscribes the lead to KlickTipp CRM with enriched insights and user profile metrics or with basic data if enrichment is unavailable. It uses different tags to differentiate enriched and basic contacts.

- **Nodes Involved:**  
  - Subscribe contact with user insights (KlickTipp)  
  - Subscribe contact with username (KlickTipp)

- **Node Details:**

  1. **Subscribe contact with user insights**  
     - Type: KlickTipp (Subscriber subscribe operation)  
     - Role: Subscribes Instagram user to KlickTipp with enriched AI insights and Instagram profile data, tagging them appropriately.  
     - Configuration:  
       - Email: extracted from JotForm submission.  
       - Tag ID: `13614965` (presumably for enriched contacts).  
       - List ID: `358895`.  
       - Custom fields mapped include first name, last name, Instagram username, biography, AI-generated insights, followers, follows, media count, and Instagram comment ID from Google Sheets.  
       - Uses KlickTipp API credentials.  
     - Input: Output from "Generate Insights" node.  
     - Output: KlickTipp subscription confirmation or error.  
     - Edge Cases: KlickTipp API rate limits, invalid email formats, missing required fields.

  2. **Subscribe contact with username**  
     - Type: KlickTipp (Subscriber subscribe operation)  
     - Role: Subscribes users without enriched profile data using basic details and a different tag.  
     - Configuration:  
       - Email and name from JotForm submission.  
       - Tag ID: `13659866` (likely for basic contacts).  
       - List ID: `358895`.  
       - Fields include first name, last name, Instagram username, and Instagram comment ID.  
       - Uses KlickTipp API credentials.  
     - Input: Output from "Profile type check" personal profile branch.  
     - Output: KlickTipp subscription result.  
     - Edge Cases: Same as above, plus potential for incomplete data due to lack of enrichment.

---

#### 2.4 DM Message Flow (Supporting Block)

- **Overview:**  
  This flow triggers on KlickTipp tag or campaign events to send personalized Instagram Direct Messages, inviting users to fill in a JotForm as part of the lead capture and enrichment loop.

- **Nodes Involved:**  
  - KlickTipp Trigger  
  - Send personalized DM to user (HTTP Request)

- **Node Details:**

  1. **KlickTipp Trigger**  
     - Type: KlickTipp Trigger (Webhook)  
     - Role: Listens for KlickTipp subscriber tag or campaign actions to initiate sending personalized Instagram DMs.  
     - Configuration: Uses KlickTipp API credentials, no additional parameters.  
     - Input: KlickTipp events.  
     - Output: Trigger data for DM sending.  
     - Edge Cases: Missed triggers if webhook downtime, missing API permissions.

  2. **Send personalized DM to user**  
     - Type: HTTP Request  
     - Role: Sends a custom Instagram DM message via Instagram Graph API to prompt form submission.  
     - Configuration:  
       - POST to `https://graph.instagram.com/v21.0/me/messages`.  
       - Message text uses dynamic interpolation: `"Hallo {{ $json.CustomFieldFirstName }} 👋*. Das ist eine persönliche Nachricht."`  
       - Recipient ID hardcoded as `"2213136375820874"` (likely a placeholder for the actual recipient).  
       - Auth via HTTP Header Auth credentials for Instagram.  
     - Input: Trigger data from KlickTipp.  
     - Output: API response from Instagram.  
     - Edge Cases: API permission errors, invalid recipient ID, message formatting errors.

---

### 3. Summary Table

| Node Name                      | Node Type                       | Functional Role                        | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                   |
|--------------------------------|--------------------------------|-------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Listen to submission from Instagram DM | JotForm Trigger               | Listen for Instagram DM form submissions | Webhook (external JotForm)       | Look for entry in matching table    | This node listens to the submissions of the form that was sent to the instagram user via DM after commenting our post.  |
| Look for entry in matching table | Google Sheets                  | Check username in matching table    | Listen to submission from Instagram DM | Get user profile                   | This node checks if there is an entry in the matching table for this username.                 |
| Get user profile                | Facebook Graph API              | Fetch Instagram profile data        | Look for entry in matching table  | Profile type check                 | This node gets the instagram profile using the username from the form submission.             |
| Profile type check             | Switch                         | Determine profile type (business/personal) | Get user profile                 | Generate Insights; Subscribe contact with username |                                                                                              |
| Generate Insights              | Langchain AI Agent             | Generate AI marketing insights      | Profile type check (business)     | Subscribe contact with user insights | This AI Agent generates marketing insights from the instagram profile information.            |
| OpenAI Chat Model               | Langchain OpenAI Chat          | Language model backend for AI       | Generate Insights                 | Structured Output Parser            |                                                                                              |
| Structured Output Parser        | Langchain Output Parser        | Parse AI output into structured JSON | OpenAI Chat Model                | Generate Insights (continue)        |                                                                                              |
| Subscribe contact with user insights | KlickTipp (Subscriber subscribe) | Subscribe enriched lead to KlickTipp | Generate Insights                | (End)                            | This node subscribes the instagram user to KlickTipp with the enriched data.                  |
| Subscribe contact with username | KlickTipp (Subscriber subscribe) | Subscribe basic lead data to KlickTipp | Profile type check (personal)   | (End)                            | This node subscribes the instagram user to KlickTipp with the enriched data.                  |
| KlickTipp Trigger              | KlickTipp Trigger               | Trigger on KlickTipp tags/campaigns | External KlickTipp event          | Send personalized DM to user       |                                                                                              |
| Send personalized DM to user   | HTTP Request                   | Send Instagram DM to prompt form fill | KlickTipp Trigger               | (End)                            | This node sends out a DM to a user in Instagram in order to prompt a form submission.         |
| Sticky Note                   | Sticky Note                   | Documentation and grouping          | N/A                              | N/A                               | See detailed notes for blocks in section 5.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger node:**  
   - Type: `JotForm Trigger`  
   - Set form ID to `252682338805059`.  
   - Use JotForm API credentials.  
   - Purpose: Listen for Instagram DM form submissions.

2. **Add Google Sheets node (Lookup):**  
   - Type: `Google Sheets`  
   - Configure OAuth2 credentials for Google Sheets.  
   - Set document URL to your matching table spreadsheet.  
   - Target Sheet1 (`gid=0`).  
   - Add filter: lookup "Instagram username" column for value `{{$json.instagram_username}}`.  
   - Connect output of JotForm Trigger to this node.

3. **Add Facebook Graph API node:**  
   - Type: `Facebook Graph API`  
   - Use Facebook Graph API credentials with `instagram_basic`, `pages_show_list`, `business_management` permissions.  
   - Set API version to `v23.0`.  
   - Configure to query Instagram business discovery fields for username from Google Sheets output:  
     `business_discovery.username({{ $json.instagram_username }}){id,username,name,biography,followers_count,follows_count,media_count,profile_picture_url}`.  
   - Connect Google Sheets lookup node output here.

4. **Add Switch node (Profile type check):**  
   - Type: `Switch`  
   - Condition 1 (Professional profile): Check if `business_discovery.id` exists and is a string.  
   - Condition 2 (Personal profile): Fallback if `statusCode` exists (number).  
   - Connect Facebook Graph API node output here.

5. **Add Langchain AI Agent node (Generate Insights):**  
   - Type: `Langchain Agent`  
   - Use an AI prompt analyzing Instagram user bio, followers, follows, media count, profile image for marketing insights.  
   - Map dynamic input fields from Facebook Graph API output.  
   - Enable output parser.  
   - Connect "Professional profile" output from Switch node to this node.

6. **Add Langchain OpenAI Chat Model node:**  
   - Type: `Langchain OpenAI Chat`  
   - Use OpenAI API credentials.  
   - Select model `gpt-4.1-mini`.  
   - Connect to Langchain AI Agent node.

7. **Add Langchain Structured Output Parser node:**  
   - Type: `Output Parser Structured JSON`  
   - Provide example JSON schema for AI output formatting.  
   - Connect OpenAI Chat Model output to this node.  
   - Connect output back to AI Agent node (creates a feedback loop for parsing).

8. **Add KlickTipp subscribe node (Subscribe contact with user insights):**  
   - Type: `KlickTipp` (Subscriber subscribe)  
   - Use KlickTipp API credentials.  
   - Set email to `{{$node["Listen to submission from Instagram DM"].item.json.Email}}`.  
   - Assign tag ID for enriched contacts (e.g., `13614965`).  
   - Map custom fields with data: first/last name, Instagram username, biography, AI insights JSON, followers, follows, media count, Instagram comment ID from Google Sheets.  
   - Connect output of AI Agent node here.

9. **Add KlickTipp subscribe node (Subscribe contact with username):**  
   - Type: `KlickTipp` (Subscriber subscribe)  
   - Use KlickTipp API credentials.  
   - Set email and names from JotForm node.  
   - Assign tag ID for basic contacts (e.g., `13659866`).  
   - Map minimal fields: first/last name, Instagram username, Instagram comment ID.  
   - Connect "Personal profile" output of Switch node here.

10. **Build DM Message Flow:**  
    - Add KlickTipp Trigger node with KlickTipp API credentials to listen for tag/campaign events.  
    - Add HTTP Request node to send Instagram DM via Graph API:  
      - URL: `https://graph.instagram.com/v21.0/me/messages`  
      - Method: POST  
      - Body: JSON with message text and recipient ID (dynamic recipient ID to be set as needed).  
      - Authentication: HTTP Header Auth with Instagram credentials.  
    - Connect KlickTipp Trigger node to HTTP Request node.

11. **Add Sticky Notes:**  
    - Add descriptive sticky notes to each logical block to document functionality, API permission requirements, and best practices.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow uses **KlickTipp community nodes** available for **self-hosted n8n instances only**. It is designed for digital marketers to automate Instagram DM outreach, data enrichment, and CRM segmentation. The process includes personalized DM sending, JotForm submission listening, Instagram profile enrichment via Facebook Graph API, AI-powered marketing insights creation, and KlickTipp CRM subscription.                                                                                                                                                  | Sticky Note3 in the workflow.                                                                                     |
| API Permissions Reminder: Ensure your Meta app has permissions `instagram_basic`, `pages_show_list`, and `business_management` to retrieve Instagram business discovery data. Without these, the Facebook Graph API will not return enriched profile fields.                                                                                                                                                                                                                                                                                                                                                               | Sticky Note10 in the workflow.                                                                                    |
| The JotForm Trigger listens for form submissions sent via Instagram DM after users comment on posts, capturing lead contact info and Instagram username. This is critical for correlating Instagram engagement with leads and triggering enrichment.                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note9 and Sticky Note6.                                                                                    |
| The AI prompt can be customized to focus on attributes most useful for your target audience, such as purchase intent, tone, or niche interests. Adjust the prompt text in the "Generate Insights" node accordingly.                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note7.                                                                                                     |
| Add KlickTipp custom fields for Instagram-specific data like bio, followers count, and AI insights to enable dynamic segmentation and personalized campaign automation.                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note8.                                                                                                     |
| The Instagram DM sending node uses the Instagram Graph API and requires a valid recipient user ID. Customize the message content and recipient dynamically to fit your campaign needs.                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note9 and "Send personalized DM to user" node notes.                                                      |
| Workflow credits and setup instructions are embedded in sticky notes for ease of understanding and replication.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | See all sticky notes for setup guidance.                                                                          |

---

**Disclaimer:** The provided text is sourced solely from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All processed data are legal and publicly available.