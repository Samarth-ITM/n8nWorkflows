Create Film Location Database with Google Maps, AI Analysis & Google Sheets

https://n8nworkflows.xyz/workflows/create-film-location-database-with-google-maps--ai-analysis---google-sheets-11087


# Create Film Location Database with Google Maps, AI Analysis & Google Sheets

### 1. Workflow Overview

This workflow, titled **AI Location Scout Assistant**, is designed to serve as an automated film location scouting tool. It accepts user input keywords describing desired filming locations, queries the Google Maps Places API to find matching locations, leverages AI to generate creative, concise descriptions for each location from a professional film director’s perspective, stores the enriched location data into a Google Sheet, and finally sends a notification on Slack to inform users that the scouting list is ready.

The workflow is logically divided into the following blocks:

- **1.1 Input & Configuration**  
  Captures user chat commands and sets global variables (Google Maps API key, Slack channel ID, Google Sheet ID) for use throughout the process.

- **1.2 Search & Data Preparation**  
  Queries the Google Places API with the user keyword, splits the resulting list into individual locations, and limits the number of processed locations to control API usage and costs.

- **1.3 AI Analysis & Output**  
  Iterates through each location, uses an AI agent to generate a creative director’s commentary, appends the enriched data to a Google Sheet, and sends a Slack notification upon completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration

**Overview:**  
This block handles the reception of user input through a chat interface, manages the memory buffer for contextual chat continuity, and sets up necessary configuration parameters (API keys and IDs) required for subsequent API calls and notifications.

**Nodes Involved:**  
- Chat Trigger  
- Simple Memory  
- Workflow Configuration  
- Main Sticky (documentation note)  
- Config Group (documentation note)

**Node Details:**

- **Chat Trigger**  
  - *Type:* Langchain Chat Trigger node  
  - *Role:* Entry point for user input, accepting chat commands publicly via a webhook.  
  - *Configuration:*  
    - Public webhook enabled  
    - Loads previous session memory from RAM to maintain context  
  - *Key Expressions:* None directly, but input is passed as `$json.chatInput` downstream.  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Connected to "Workflow Configuration" node  
  - *Failure Modes:* Network/webhook failures, memory loading issues  
  - *Version:* 1.4  

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Stores recent chat messages to maintain conversational context.  
  - *Configuration:* Default, no custom parameters set.  
  - *Inputs:* Receives AI memory input from Chat Trigger  
  - *Outputs:* None connected downstream explicitly here  
  - *Failure Modes:* Memory overflow or corruption unlikely but possible in high load  
  - *Version:* 1.3  

- **Workflow Configuration**  
  - *Type:* Set node  
  - *Role:* Holds global configuration variables: Google Maps API Key, Slack Channel ID, Google Sheet ID  
  - *Configuration:*  
    - Three string parameters with placeholder values to be replaced by the user:  
      - `googleMapsApiKey`  
      - `slackChannelId`  
      - `googleSheetId`  
  - *Inputs:* From Chat Trigger  
  - *Outputs:* To Google Maps Places Search node  
  - *Failure Modes:* Missing or incorrect API keys or IDs will cause failures downstream  
  - *Version:* 3.4  

- **Main Sticky & Config Group**  
  - Sticky notes providing detailed instructions and overview for this block.  
  - No runtime role.

---

#### 2.2 Search & Data Preparation

**Overview:**  
This block queries Google Places API using the user’s keyword, processes the returned list of locations by splitting results into individual entries, and applies a limit to the number of locations processed to control API usage and manage cost.

**Nodes Involved:**  
- Google Maps Places Search  
- Split Out  
- Limit  
- Search Group (documentation note)

**Node Details:**

- **Google Maps Places Search**  
  - *Type:* HTTP Request node  
  - *Role:* Queries Google Places Text Search API to find locations matching the user input  
  - *Configuration:*  
    - Method: GET  
    - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`  
    - Query parameters:  
      - `query`: user input from chat (`{{$json.chatInput}}`)  
      - `key`: Google Maps API Key from "Workflow Configuration"  
    - Response format: JSON  
  - *Inputs:* From Workflow Configuration  
  - *Outputs:* To "Split Out" node  
  - *Failure Modes:* API key invalid or quota exceeded, malformed queries, network timeouts  
  - *Version:* 4.3  

- **Split Out**  
  - *Type:* Split Out node  
  - *Role:* Extracts the array of location results (`results` field) from Google Maps API response into separate items  
  - *Configuration:*  
    - Field to split out: `results`  
  - *Inputs:* From Google Maps Places Search  
  - *Outputs:* To "Limit" node  
  - *Failure Modes:* If `results` is missing or empty, downstream nodes receive no data  
  - *Version:* 1  

- **Limit**  
  - *Type:* Limit node  
  - *Role:* Restricts the number of location items to process (set to 2 by default)  
  - *Configuration:*  
    - Max items: 2 (configurable)  
  - *Inputs:* From Split Out  
  - *Outputs:* To Loop Over Items node  
  - *Failure Modes:* If limit is set too low, fewer locations are processed; if too high, cost may increase  
  - *Version:* 1  

- **Search Group**  
  - Sticky note documenting this block’s purpose and usage.

---

#### 2.3 AI Analysis & Output

**Overview:**  
This block iterates over each limited location, runs an AI agent to generate a concise, cinematic description of the location from a professional director’s viewpoint, appends the detailed data including AI commentary to a Google Sheet, and sends a Slack notification once all locations are processed.

**Nodes Involved:**  
- Loop Over Items  
- AI Location Analyzer  
- OpenRouter Chat Model  
- Append to Google Sheets  
- Send Slack Notification  
- Process Group (documentation note)

**Node Details:**

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Loops through each location item sequentially for processing  
  - *Configuration:* Default batch size (1 implicitly)  
  - *Inputs:* From Limit node  
  - *Outputs:* Two outputs:  
    1. To Send Slack Notification (after all items processed)  
    2. To AI Location Analyzer (per item processing)  
  - *Failure Modes:* Loop interruptions or batch size misconfiguration may cause incomplete processing  
  - *Version:* 3  

- **AI Location Analyzer**  
  - *Type:* Langchain Agent node  
  - *Role:* Uses AI to generate a 50-character or less creative director’s commentary about the location  
  - *Configuration:*  
    - Text input template includes location name, address, category, and rating:  
      ```
      場所名: {{ $json.name }}
      住所: {{ $json.formatted_address }}
      カテゴリ: {{ $json.types }}
      評価: {{ $json.rating }}
      ```  
    - System message (AI persona): Professional film director asked to write an attractive intro within 50 characters  
    - Prompt type: define  
  - *Inputs:* From Loop Over Items (individual location JSON)  
  - *Outputs:* To Append to Google Sheets  
  - *Version:* 3  
  - *Failure Modes:* AI model timeouts, malformed input data, unexpected AI output format  

- **OpenRouter Chat Model**  
  - *Type:* Langchain OpenRouter Chat Model node  
  - *Role:* Specifies the AI language model used by the AI Location Analyzer agent  
  - *Configuration:* Default options, no custom parameters  
  - *Inputs:* From AI Location Analyzer (ai_languageModel input)  
  - *Outputs:* Back to AI Location Analyzer (language model output)  
  - *Failure Modes:* Model unavailability, API key issues (not shown here but required for OpenRouter)  
  - *Version:* 1  

- **Append to Google Sheets**  
  - *Type:* Google Sheets node  
  - *Role:* Appends processed location data and AI commentary to a predefined Google Sheet  
  - *Configuration:*  
    - Operation: Append  
    - Sheet Name: "Sheet1"  
    - Document ID: From Workflow Configuration (`googleSheetId`)  
    - Columns mapped:  
      - 場所名 (Place Name): from location name  
      - 住所 (Address): from formatted_address  
      - 評価(星) (Rating): from rating  
      - GoogleMapリンク (Google Map Link): constructed URL using place_id  
      - AI監督のコメント (AI Director's Comment): AI output `$json.output`  
    - Matching columns include 場所名 for possible deduplication  
  - *Inputs:* From AI Location Analyzer  
  - *Outputs:* Back to Loop Over Items (to continue processing next item)  
  - *Failure Modes:* Google Sheets API quota limits, permission issues, incorrect document ID  
  - *Version:* 4.7  

- **Send Slack Notification**  
  - *Type:* Slack node  
  - *Role:* Sends a notification message to a configured Slack channel after all locations processed  
  - *Configuration:*  
    - Text includes:  
      - Confirmation message  
      - Original search keyword  
      - Link to the Google Sheet  
    - Channel ID from Workflow Configuration  
    - Authentication via OAuth2  
  - *Inputs:* From Loop Over Items (after completion)  
  - *Failure Modes:* Slack API authentication failure, invalid channel ID, network errors  
  - *Version:* 2.3  

- **Process Group**  
  - Sticky note documenting this block’s purpose and flow.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                           | Input Node(s)                | Output Node(s)                      | Sticky Note                                                                                                                                                                                                                                                                                                                                                   |
|-------------------------|--------------------------------------|------------------------------------------|-----------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Chat Trigger            | Langchain Chat Trigger                | Entry point for user input via chat      | None                        | Workflow Configuration            | See "Main Sticky" for overall workflow description; also "Config Group" explains this block.                                                                                                                                                                                                                                                                |
| Simple Memory           | Langchain Memory Buffer Window        | Maintains chat context                    | Chat Trigger (ai_memory)     | None                            | See "Config Group" note                                                                                                                                                                                                                                                                                                                                     |
| Workflow Configuration  | Set                                  | Holds API keys and IDs                    | Chat Trigger                | Google Maps Places Search         | See "Config Group" note                                                                                                                                                                                                                                                                                                                                     |
| Google Maps Places Search | HTTP Request                        | Queries Google Places API                 | Workflow Configuration       | Split Out                       | See "Search Group" note                                                                                                                                                                                                                                                                                                                                     |
| Split Out               | Split Out                            | Splits Google Places results array       | Google Maps Places Search    | Limit                           | See "Search Group" note                                                                                                                                                                                                                                                                                                                                     |
| Limit                   | Limit                               | Limits number of locations to process    | Split Out                   | Loop Over Items                 | See "Search Group" note                                                                                                                                                                                                                                                                                                                                     |
| Loop Over Items         | Split In Batches                    | Loops through individual locations       | Limit                       | AI Location Analyzer, Send Slack Notification | See "Process Group" note                                                                                                                                                                                                                                                                                                                                   |
| AI Location Analyzer    | Langchain Agent                     | Generates AI commentary for each location | Loop Over Items             | Append to Google Sheets          | See "Process Group" note                                                                                                                                                                                                                                                                                                                                   |
| OpenRouter Chat Model   | Langchain OpenRouter Chat Model      | Provides AI language model                | AI Location Analyzer (ai_languageModel) | AI Location Analyzer      | See "Process Group" note                                                                                                                                                                                                                                                                                                                                   |
| Append to Google Sheets | Google Sheets                       | Saves location data with AI commentary   | AI Location Analyzer        | Loop Over Items                 | See "Process Group" note                                                                                                                                                                                                                                                                                                                                   |
| Send Slack Notification | Slack                              | Sends completion notification to Slack   | Loop Over Items (after all processed) | None                          | See "Process Group" note                                                                                                                                                                                                                                                                                                                                   |
| Main Sticky             | Sticky Note                        | Documentation                            | None                        | None                          | Contains full workflow overview and setup instructions                                                                                                                                                                                                                                                                                                     |
| Config Group            | Sticky Note                        | Documentation for Input & Config block    | None                        | None                          | Explains the first block                                                                                                                                                                                                                                                                                                                                   |
| Search Group            | Sticky Note                        | Documentation for Search & Data Prep block | None                        | None                          | Explains the second block                                                                                                                                                                                                                                                                                                                                  |
| Process Group           | Sticky Note                        | Documentation for AI Analysis & Output block | None                        | None                          | Explains the third block                                                                                                                                                                                                                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: Langchain Chat Trigger  
   - Configure webhook as public  
   - Enable option to load previous session memory ("memory")  
   - Position near top-left for clarity  

2. **Create Simple Memory Node**  
   - Type: Langchain Memory Buffer Window  
   - Connect its `ai_memory` input to the Chat Trigger’s `ai_memory` output  
   - No special parameters needed  

3. **Create Workflow Configuration Node**  
   - Type: Set node  
   - Add three string parameters:  
     - `googleMapsApiKey` (enter your Google Maps Places API key)  
     - `slackChannelId` (your Slack channel ID)  
     - `googleSheetId` (your Google Sheet document ID)  
   - Connect Chat Trigger’s main output to this node  

4. **Create Google Maps Places Search Node**  
   - Type: HTTP Request node  
   - Method: GET  
   - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`  
   - Query parameters: `query` = `={{ $json.chatInput }}`, `key` = `={{ $('Workflow Configuration').first().json.googleMapsApiKey }}`  
   - Set response format to JSON  
   - Connect Workflow Configuration’s output to this node  

5. **Create Split Out Node**  
   - Type: Split Out node  
   - Field to split out: `results`  
   - Connect Google Maps Places Search output here  

6. **Create Limit Node**  
   - Type: Limit node  
   - Set max items to 2 (or adjust as desired to control API usage)  
   - Connect Split Out output here  

7. **Create Loop Over Items Node**  
   - Type: Split In Batches  
   - No special batch size needed (default processes one item at a time)  
   - Connect Limit node output here  

8. **Create AI Location Analyzer Node**  
   - Type: Langchain Agent node  
   - Text input template:  
     ```
     場所名: {{ $json.name }}
     住所: {{ $json.formatted_address }}
     カテゴリ: {{ $json.types }}
     評価: {{ $json.rating }}
     ```  
   - System message: "あなたはプロの映像監督です。渡された場所情報をもとに、撮影スポットとしての魅力を50文字以内で魅力的に紹介してください。" (You are a professional film director. Based on the given location information, write an attractive introduction as a filming spot within 50 characters.)  
   - Prompt type: define  
   - Connect Loop Over Items main output (index 0) to this node  

9. **Create OpenRouter Chat Model Node**  
   - Type: Langchain OpenRouter Chat Model  
   - Default settings  
   - Connect AI Location Analyzer’s `ai_languageModel` input to this node’s output  

10. **Create Append to Google Sheets Node**  
    - Type: Google Sheets  
    - Operation: Append  
    - Sheet Name: "Sheet1"  
    - Document ID: `={{ $('Workflow Configuration').first().json.googleSheetId }}`  
    - Map columns:  
      - 場所名: `={{ $('Loop Over Items').item.json.name }}`  
      - 住所: `={{ $('Loop Over Items').item.json.formatted_address }}`  
      - 評価(星): `={{ $('Loop Over Items').item.json.rating }}`  
      - GoogleMapリンク: `=https://www.google.com/maps/place/?q=place_id:{{ $('Loop Over Items').item.json.place_id }}`  
      - AI監督のコメント: `={{ $json.output }}` (output from AI Location Analyzer)  
    - Connect AI Location Analyzer output to this node  
    - Connect this node’s output back to Loop Over Items (to continue looping)  

11. **Create Send Slack Notification Node**  
    - Type: Slack  
    - Authentication: OAuth2 (configure Slack credentials)  
    - Channel ID: `={{ $('Workflow Configuration').first().json.slackChannelId }}`  
    - Message text:  
      ```
      🎬 ロケハンリスト作成完了！
      
      キーワード: {{ $('Chat Trigger').first().json.chatInput }}
      
      シートを確認してください: https://docs.google.com/spreadsheets/d/{{ $('Workflow Configuration').first().json.googleSheetId }}
      ```  
    - Connect Loop Over Items second output (index 1) here (fires after loop completion)  

12. **Arrange nodes visually into three groups with sticky notes:**  
    - Group 1: Input & Configuration (Chat Trigger, Simple Memory, Workflow Configuration, Config Group sticky)  
    - Group 2: Search & Data Prep (Google Maps Places Search, Split Out, Limit, Search Group sticky)  
    - Group 3: AI Analysis & Output (Loop Over Items, AI Location Analyzer, OpenRouter Chat Model, Append to Google Sheets, Send Slack Notification, Process Group sticky)  

13. **Configure credentials:**  
    - Google Cloud Console: Enable Places API and generate API key for Google Maps  
    - Google Sheets: Share the sheet with the service account or your OAuth2 credentials  
    - Slack: Create an app, add OAuth2 scopes for posting messages, and configure OAuth2 credentials in n8n  

14. **Test the workflow:**  
    - Trigger Chat Trigger webhook with a keyword (e.g., "Cyberpunk streets in Tokyo")  
    - Confirm Google Maps returns results  
    - Confirm AI generates creative commentary  
    - Confirm rows appended in Google Sheet  
    - Confirm Slack notification is sent to specified channel  

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                    |
|----------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Workflow acts as a personal film location scout using Google Maps and AI to enrich location data.                         | Overview in Main Sticky note                                                                                       |
| Setup instructions include enabling Google Places API, creating Google Sheet with specific headers, and Slack channel ID. | See detailed instructions in Main Sticky note                                                                     |
| Use of Langchain nodes for AI processing shows integration of n8n with advanced AI agents via OpenRouter.                  | Requires OpenRouter API access credentials (not included in workflow JSON)                                         |
| Slack notification includes direct link to the Google Sheet for easy access.                                               | Slack message text contains link: https://docs.google.com/spreadsheets/d/{{GoogleSheetId}}                         |
| Default limit of 2 locations is set to control API usage; adjust according to your quota and needs.                        | Limit node parameter                                                                                               |
| All user-facing text in the AI prompt and comments is in Japanese, reflecting the target audience or use case.            | System message and sheet headers are in Japanese (e.g., 場所名, 住所, 評価(星), AI監督のコメント, GoogleMapリンク) |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.