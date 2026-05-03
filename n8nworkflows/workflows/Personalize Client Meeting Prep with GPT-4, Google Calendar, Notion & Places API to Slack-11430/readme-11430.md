Personalize Client Meeting Prep with GPT-4, Google Calendar, Notion & Places API to Slack

https://n8nworkflows.xyz/workflows/personalize-client-meeting-prep-with-gpt-4--google-calendar--notion---places-api-to-slack-11430


# Personalize Client Meeting Prep with GPT-4, Google Calendar, Notion & Places API to Slack

### 1. Workflow Overview

This n8n workflow automates personalized preparation for client meetings by integrating GPT-4, Google Calendar, Notion, Google Places API, and Slack. Its core purpose is to enhance client interactions by recommending tailored gift shops and quiet cafes near meeting locations, based on customer preferences stored in Notion.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Triggered by relevant Google Calendar events.
- **1.2 Configuration & Filtering:** Sets parameters and filters events for client meetings.
- **1.3 Data Enrichment:** Extracts company names and fetches customer preferences from Notion.
- **1.4 Location Search:** Queries Google Places API for gift shops and cafes near the meeting.
- **1.5 AI Recommendation:** Uses GPT-4 to analyze data and generate personalized recommendations.
- **1.6 Notification:** Sends the AI-generated recommendations to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Listens to Google Calendar events, triggering the workflow when events are created or updated. It detects relevant events such as meetings or client visits.

**Nodes Involved:**  
- Google Calendar Trigger

**Node Details:**  
- **Google Calendar Trigger**  
  - Type: Trigger node for Google Calendar events  
  - Configuration: Polls every minute for event updates in a specified calendar (configured via `YOUR_CALENDAR_ID`)  
  - Input/Output: No input; outputs event data on trigger  
  - Credentials: Requires Google Calendar OAuth2 credentials  
  - Edge Cases: Possible auth expiry or API rate limiting; calendar ID must be correct; triggers on all updates, so filtering is necessary downstream

#### 2.2 Configuration & Filtering

**Overview:**  
Sets necessary API keys and parameters, then filters events to only process those related to client meetings or visits.

**Nodes Involved:**  
- Workflow Configuration (Set node)  
- Filter Client Visit Events (If node)

**Node Details:**  
- **Workflow Configuration**  
  - Type: Set node  
  - Role: Defines key variables such as Google Places API key, search radius (default 1000m), and minimum rating (default 4.5)  
  - Input/Output: Receives event from trigger, outputs enriched data with config variables  
  - Edge Cases: Must replace placeholder API key `YOUR_GOOGLE_PLACES_API_KEY` with a valid key

- **Filter Client Visit Events**  
  - Type: If node (conditional)  
  - Role: Filters events based on keywords in the event summary: "visit", "meeting", "greeting", "dinner", "client"  
  - Input: From Workflow Configuration  
  - Output: Passes only matching events forward  
  - Edge Cases: Case-insensitive string matching; events without these keywords are discarded

#### 2.3 Data Enrichment

**Overview:**  
Extracts the client company name from the event title and retrieves the corresponding customer preferences from a Notion database.

**Nodes Involved:**  
- Extract Company Name (Code node)  
- Get Customer Preferences from Notion (Notion node)

**Node Details:**  
- **Extract Company Name**  
  - Type: Code (JavaScript) node  
  - Role: Parses the event summary to find a company name using regex patterns matching common company suffixes (Inc, Corp, LLC, Ltd, etc.)  
  - Key Expression: Uses regex to extract company name or defaults to first word  
  - Input: Filter Client Visit Events output  
  - Output: Adds `extractedCompany` field to JSON  
  - Edge Cases: May fail if event summary format is unexpected or company name is missing

- **Get Customer Preferences from Notion**  
  - Type: Notion node (databasePage getAll operation)  
  - Role: Queries Notion database filtering where "Company Name" title contains `extractedCompany`  
  - Configuration: Requires Notion database ID and API credentials  
  - Output: Retrieves customer preferences field (rich text)  
  - Edge Cases: No matching company leads to empty results; Notion API rate limits or credential errors possible

#### 2.4 Location Search

**Overview:**  
Uses Google Places API to find gift shops (bakeries, confectioneries) and quiet cafes near the meeting location, applying configured search radius and keywords.

**Nodes Involved:**  
- Search Gift Shops (HTTP Request node)  
- Search Nearby Cafes (HTTP Request node)

**Node Details:**  
- **Search Gift Shops**  
  - Type: HTTP Request  
  - Role: Calls Google Places Nearby Search API with type `bakery` and keyword `pastry sweets gift`  
  - Parameters: Location (from input JSON, defaults to Tokyo station coords), radius (from config), API key  
  - Output: JSON list of gift shop candidates  
  - Edge Cases: Google Places API quota limits, invalid API key, no results found

- **Search Nearby Cafes**  
  - Type: HTTP Request  
  - Role: Calls Google Places Nearby Search API with type `cafe` and keyword `quiet work cafe`  
  - Same parameter setup as gift shops  
  - Output: JSON list of cafe candidates  
  - Edge Cases: Same as gift shops

#### 2.5 AI Recommendation

**Overview:**  
Processes gathered data with GPT-4 to select the best gift shop and cafe, providing personalized explanations based on customer preferences.

**Nodes Involved:**  
- AI Gift Recommendation (OpenAI node)

**Node Details:**  
- **AI Gift Recommendation**  
  - Type: OpenAI (GPT-4) via LangChain node  
  - Role: Receives customer preferences and place search results; prompts GPT-4 to pick one gift shop and one cafe with reasoning  
  - Key Prompt Elements: Includes client company, preferences, lists of candidates, instructions for output format  
  - Output: Text message formatted for Slack posting  
  - Credentials: Requires OpenAI API key  
  - Edge Cases: Prompt failures, API rate limits, incomplete input data

#### 2.6 Notification

**Overview:**  
Delivers the AI-generated personalized recommendation message to a specified Slack channel.

**Nodes Involved:**  
- Send Slack Notification (Slack node)

**Node Details:**  
- **Send Slack Notification**  
  - Type: Slack node  
  - Role: Posts the GPT-4 output text to a configured Slack channel (channel ID must be updated)  
  - Input: Text from AI Gift Recommendation  
  - Credentials: Slack OAuth2 credentials required  
  - Edge Cases: Slack API errors, invalid channel ID, authentication expiry

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                         | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                                   |
|----------------------------|-----------------------------|---------------------------------------|---------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------|
| Google Calendar Trigger     | Google Calendar Trigger     | Triggers workflow on calendar events  | —                         | Workflow Configuration          |                                                                                                               |
| Workflow Configuration      | Set                         | Defines API keys and parameters       | Google Calendar Trigger    | Filter Client Visit Events      | ### ⚙️ Step 1: Configuration Set your API keys and search parameters here. Replace Google Places API key.     |
| Filter Client Visit Events  | If                          | Filters relevant client meeting events| Workflow Configuration    | Extract Company Name            | ### 🔍 Step 2: Event Filtering Filters events by keywords: visit, meeting, client, dinner, greeting             |
| Extract Company Name        | Code                        | Extracts company name from event title| Filter Client Visit Events| Get Customer Preferences from Notion | ### 📋 Step 3: Data Enrichment Extract company and get preferences from Notion database                      |
| Get Customer Preferences from Notion | Notion              | Retrieves customer preferences        | Extract Company Name       | Search Nearby Cafes, Search Gift Shops |                                                                                                               |
| Search Gift Shops           | HTTP Request                | Searches nearby gift shops            | Get Customer Preferences from Notion | AI Gift Recommendation         | ### 🗺️ Step 4: Location Search Finds gift shops and cafes near meeting location                             |
| Search Nearby Cafes         | HTTP Request                | Searches nearby cafes                 | Get Customer Preferences from Notion | AI Gift Recommendation         |                                                                                                               |
| AI Gift Recommendation      | OpenAI (LangChain)          | Generates personalized recommendation | Search Gift Shops, Search Nearby Cafes | Send Slack Notification        | ### 🤖 Step 5: AI Recommendation GPT-4 recommends best gift shop and cafe with reasoning                      |
| Send Slack Notification     | Slack                       | Sends recommendation to Slack channel| AI Gift Recommendation    | —                              | ### 📤 Step 6: Notification Sends the personalized recommendations to Slack channel                          |
| Sticky Note - Overview      | Sticky Note                 | Documentation overview                | —                         | —                              | ##  Automate Personalized Gift & Cafe Recommendations with GPT-4, Google Calendar & Notion                    |
| Sticky Note - Config        | Sticky Note                 | Configuration instructions            | —                         | —                              | ### ⚙️ Step 1: Configuration Set your API keys and search parameters here                                    |
| Sticky Note - Filter        | Sticky Note                 | Event filtering instructions          | —                         | —                              | ### 🔍 Step 2: Event Filtering Filters calendar events to only process client visits and meetings            |
| Sticky Note - Enrich        | Sticky Note                 | Data enrichment instructions          | —                         | —                              | ### 📋 Step 3: Data Enrichment Extracts company name and fetches customer preferences from Notion            |
| Sticky Note - Search        | Sticky Note                 | Location search instructions          | —                         | —                              | ### 🗺️ Step 4: Location Search Searches for gift shops and cafes near the meeting location                   |
| Sticky Note - AI            | Sticky Note                 | AI recommendation instructions        | —                         | —                              | ### 🤖 Step 5: AI Recommendation GPT-4 analyzes preferences and recommends gift shop and cafe                |
| Sticky Note - Notify        | Sticky Note                 | Slack notification instructions       | —                         | —                              | ### 📤 Step 6: Notification Sends the personalized recommendation to your Slack channel                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger Node**  
   - Type: Google Calendar Trigger  
   - Configure to poll every minute on event updates (`eventUpdated`)  
   - Select your calendar ID (`YOUR_CALENDAR_ID`)  
   - Connect Google Calendar OAuth2 credentials

2. **Create Set Node for Workflow Configuration**  
   - Name: Workflow Configuration  
   - Add variables:  
     - `googlePlacesApiKey` (string): Your Google Places API key  
     - `searchRadius` (number): e.g., 1000 (meters)  
     - `minRating` (number): e.g., 4.5  
   - Connect output from Google Calendar Trigger

3. **Create If Node to Filter Client Visit Events**  
   - Name: Filter Client Visit Events  
   - Condition: Check if event summary (from trigger) contains any of:  
     - "visit", "meeting", "greeting", "dinner", "client" (case-insensitive)  
   - Connect output from Workflow Configuration

4. **Create Code Node to Extract Company Name**  
   - Name: Extract Company Name  
   - JavaScript code:  
     ```js
     const summary = $input.first().json.summary || '';
     const companyMatch = summary.match(/([^\s]+(?:Inc|Corp|LLC|Ltd|Company|Co\.))/);
     const companyName = companyMatch ? companyMatch[0] : summary.split(' ')[0];
     return [{ json: { ...$input.first().json, extractedCompany: companyName } }];
     ```  
   - Connect output from Filter Client Visit Events

5. **Create Notion Node to Get Customer Preferences**  
   - Name: Get Customer Preferences from Notion  
   - Resource: Database Page, Operation: Get All  
   - Filter: `Company Name|title` contains `={{ $json.extractedCompany }}`  
   - Set your Notion database ID (`YOUR_NOTION_DATABASE_ID`)  
   - Connect Notion API credentials  
   - Connect output from Extract Company Name

6. **Create HTTP Request Node to Search Gift Shops**  
   - Name: Search Gift Shops  
   - URL: `https://maps.googleapis.com/maps/api/place/nearbysearch/json`  
   - Query parameters:  
     - `location`: `={{ $json.location || '35.6812,139.7671' }}` (default Tokyo station coords)  
     - `radius`: `={{ $json.searchRadius }}`  
     - `type`: `bakery`  
     - `keyword`: `pastry sweets gift`  
     - `key`: `={{ $json.googlePlacesApiKey }}`  
   - Connect output from Get Customer Preferences from Notion

7. **Create HTTP Request Node to Search Nearby Cafes**  
   - Name: Search Nearby Cafes  
   - Same URL and parameters as gift shops, except:  
     - `type`: `cafe`  
     - `keyword`: `quiet work cafe`  
   - Connect output from Get Customer Preferences from Notion

8. **Create OpenAI Node for AI Gift Recommendation**  
   - Name: AI Gift Recommendation  
   - Model: GPT-4 (or equivalent available version)  
   - Prompt: Craft a prompt incorporating:  
     - Client company name (`extractedCompany`)  
     - Customer preferences from Notion  
     - JSON-serialized gift shop and cafe candidates  
     - Instructions to select one gift shop and one cafe with reasoning  
   - Connect outputs from Search Gift Shops and Search Nearby Cafes as inputs  
   - Connect OpenAI API credentials

9. **Create Slack Node to Send Notification**  
   - Name: Send Slack Notification  
   - Text: Use expression to pass AI Gift Recommendation output text (`={{ $json.output[0].content[0].text }}`)  
   - Select your Slack channel by name or ID (`YOUR_CHANNEL_ID`)  
   - Connect Slack OAuth2 credentials  
   - Connect output from AI Gift Recommendation

10. **Connect Workflow Nodes in Order:**  
    - Google Calendar Trigger → Workflow Configuration → Filter Client Visit Events → Extract Company Name → Get Customer Preferences from Notion → (parallel) Search Gift Shops & Search Nearby Cafes → AI Gift Recommendation → Send Slack Notification

11. **Testing & Adjustments:**  
    - Replace placeholders: Google Places API key, Notion database ID, Google Calendar ID, Slack channel ID  
    - Validate API credentials for all services  
    - Test with sample calendar events containing keywords to verify end-to-end functionality

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow enhances client engagement by combining AI-driven personalization with real-world location data and calendar triggers.| Workflow Overview Sticky Note                                                                        |
| Setup requires Google Calendar, Notion, Google Places API, OpenAI, and Slack accounts with respective API keys and OAuth2 credentials. | Setup instructions in Sticky Notes and inline node comments                                          |
| Slack message output is formatted to be friendly and readable, directly suitable for posting to channels without further editing.   | Example Slack Output in Overview Sticky Note                                                         |
| Customize keywords in the filter node to match your calendar event naming conventions for best results.                             | Sticky Note - Filter                                                                                  |
| Notion database must have "Company Name" as Title field and "Preferences" as text field to properly retrieve customer info.        | Sticky Note - Enrich                                                                                  |
| Google Places API search radius and minimum rating can be tuned for local preferences and quality control.                          | Sticky Note - Config                                                                                  |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automation workflow. It adheres strictly to relevant content policies and contains no illegal or protected elements. All processed data is legal and publicly accessible.