Personalized Taiwan Indie Music Recommendations with AI, Star Sign & Weather via Spotify

https://n8nworkflows.xyz/workflows/personalized-taiwan-indie-music-recommendations-with-ai--star-sign---weather-via-spotify-4210


# Personalized Taiwan Indie Music Recommendations with AI, Star Sign & Weather via Spotify

### 1. Workflow Overview

This workflow provides personalized Taiwan indie music recommendations by integrating AI, astrology, weather data, and Spotify search. It targets users seeking music suggestions tailored to their current city, mood, birthday, today’s weather, and zodiac sign. The recommendation logic is powered primarily by OpenAI’s GPT models and refined via a structured data extraction process, culminating in a Spotify track search.

The workflow is logically divided into these blocks:

- **1.1 Manual Trigger & Input Setup:** Starts workflow manually and sets user parameters (city, mood, birthday, language).
- **1.2 AI-Powered Music Recommendation:** Uses OpenAI GPT to combine weather, star sign, and fortune information to recommend a Taiwan indie song.
- **1.3 Structured Data Extraction:** Parses the AI’s JSON response to extract detailed fields about the recommendation.
- **1.4 Spotify Track Search:** Searches Spotify for the recommended song and artist to retrieve a playable link.
- **1.5 Final Data Compilation:** Aggregates all information, including Spotify URL, into a final output object.
- **1.6 Contextual Notes:** Sticky notes provide instructions, changeable parameters, and credits.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & Input Setup

- **Overview:**  
  This block initiates the workflow manually and sets the basic input parameters for the recommendation, including the user's city, mood, birthday, and preferred reply language.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - infomation

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow upon manual execution.  
    - *Configuration:* No parameters; serves as the entry point.  
    - *Inputs:* None  
    - *Outputs:* Connects to "infomation" node.  
    - *Potential Failures:* None (manual trigger).  

  - **infomation**  
    - *Type:* Set Node  
    - *Role:* Defines user inputs: city ("Taipei"), mood ("Happy"), birthday ("1996/11/21"), and reply language ("zh-tw").  
    - *Configuration:* Hardcoded values for these fields; can be modified to customize user inputs.  
    - *Inputs:* Receives trigger from manual node.  
    - *Outputs:* Feeds inputs into "get song recommendation" node.  
    - *Edge Cases:* If values are missing or malformed (e.g., invalid birthday format), downstream nodes may fail or produce incorrect results.

---

#### 2.2 AI-Powered Music Recommendation

- **Overview:**  
  Uses OpenAI’s GPT-4o-mini model to generate a Taiwan indie music recommendation based on city weather, star sign derived from birthday, and zodiac fortune. The AI returns a JSON-formatted response containing multiple fields.

- **Nodes Involved:**  
  - get song recommendation

- **Node Details:**

  - **get song recommendation**  
    - *Type:* OpenAI (LangChain) Node  
    - *Role:* Executes a prompt-driven AI process to:  
      1. Determine today’s weather in the given city.  
      2. Derive user’s star sign from birthday.  
      3. Fetch daily fortune for that star sign.  
      4. Recommend a Taiwan indie song considering weather and fortune.  
      5. Explain the choice and highlight song features.  
    - *Configuration:*  
      - Model: GPT-4o-mini-search-preview  
      - Prompt includes system role with step-by-step instructions and variables for city, birthday, and reply language.  
      - Output expected in JSON.  
    - *Inputs:* Receives user data from "infomation".  
    - *Outputs:* Passes raw AI response to "Information Extractor".  
    - *Version Requirements:* Uses LangChain integration with OpenAI API v1.8.  
    - *Potential Failures:*  
      - API key or quota exhaustion.  
      - Prompt or variable interpolation errors.  
      - Unexpected AI output format causing extraction failures.  
      - Network or timeout issues.  

---

#### 2.3 Structured Data Extraction

- **Overview:**  
  Parses the AI-generated JSON string into structured fields for easier use downstream. Extracts date, city, weather, star sign, fortune, song, artist, and additional song information.

- **Nodes Involved:**  
  - Information Extractor

- **Node Details:**

  - **Information Extractor**  
    - *Type:* LangChain Information Extractor  
    - *Role:* Converts AI text output JSON into a schema-based JSON object with clearly defined fields.  
    - *Configuration:*  
      - Input text: From AI message content.  
      - Schema example defines expected keys: "todays date," "city," "weather," "star sign," "fortune," "song," "artist," "additional infomation".  
    - *Inputs:* Receives AI output from "get song recommendation".  
    - *Outputs:* Supplies parsed data to "Spotify" node.  
    - *Potential Failures:*  
      - Malformed or incomplete AI JSON response.  
      - Schema mismatch leading to extraction errors.  
      - Empty or missing content.  

---

#### 2.4 Spotify Track Search

- **Overview:**  
  Searches Spotify for the recommended Taiwan indie song using the artist and song name to retrieve track details and a Spotify URL.

- **Nodes Involved:**  
  - Spotify

- **Node Details:**

  - **Spotify**  
    - *Type:* Spotify node  
    - *Role:* Searches Spotify tracks to find the recommended song’s metadata and external URL.  
    - *Configuration:*  
      - Resource: track  
      - Operation: search  
      - Query: Combines extracted artist and song names (e.g., `"陳綺貞 小步舞曲"`).  
      - Limit: 1 (returns top result).  
    - *Inputs:* Receives parsed output from "Information Extractor".  
    - *Outputs:* Sends track metadata and URL to "Final Output".  
    - *Credentials:* Spotify OAuth2 API required (configured).  
    - *Potential Failures:*  
      - Spotify API authentication issues.  
      - No matching track found (empty results).  
      - Rate limits or service downtime.  

---

#### 2.5 Final Data Compilation

- **Overview:**  
  Aggregates all relevant data, including date, city, weather, star sign, fortune, song, artist, additional info, and Spotify URL into a unified JSON structure for output or further use.

- **Nodes Involved:**  
  - Final Output

- **Node Details:**

  - **Final Output**  
    - *Type:* Set Node  
    - *Role:* Consolidates extracted fields and Spotify link into a single output object under the property `output`.  
    - *Configuration:*  
      - Assigns fields like "todays date," "city," "weather," "star sign," "fortune," "artist," "song," "songlink," and "additional infomation".  
      - Values are pulled from "Information Extractor" and Spotify response using expressions.  
    - *Inputs:* Receives Spotify track info.  
    - *Outputs:* Final output of the workflow (no further nodes connected).  
    - *Potential Failures:*  
      - Missing fields if previous nodes failed or returned incomplete data.  
      - Expression evaluation errors if references are invalid.

---

#### 2.6 Contextual Notes

- **Overview:**  
  Sticky notes provide workflow context, instructions for customization, and credit information.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Content:* Detailed description of the workflow’s purpose, logic overview, and credit to the creator (n8nguide), including a link to [n8nguide](https://www.threads.com/@n8nguide.tw).  
    - *Role:* Documentation and user guidance.  

  - **Sticky Note1**  
    - *Type:* Sticky Note  
    - *Content:* Instructions to change location, birthday, and preferred output language.  
    - *Role:* User customization hints.

  - **Sticky Note2**  
    - *Type:* Sticky Note  
    - *Content:* Reminder to update credentials when necessary.  
    - *Role:* Maintenance reminder.

  - **Sticky Note3**  
    - *Type:* Sticky Note  
    - *Content:* Explains that the final output node tidies the data and can be called from other workflows.  
    - *Role:* Workflow modularity hint.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                     | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                      |
|---------------------------|----------------------------------|-----------------------------------|-----------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Workflow start                    | None                        | infomation                 |                                                                                                 |
| infomation                | Set                              | User input setup                  | When clicking ‘Test workflow’ | get song recommendation    | ### - Change your location<br>### - Change Your Birthday<br>### - Change Your prefer output language |
| get song recommendation   | OpenAI LangChain Chat             | AI music recommendation           | infomation                  | Information Extractor      |                                                                                                 |
| Information Extractor     | LangChain Information Extractor  | Parse AI JSON output              | get song recommendation      | Spotify                   |                                                                                                 |
| Spotify                   | Spotify node                     | Search for recommended song       | Information Extractor        | Final Output               | ### update credential                                                                           |
| Final Output              | Set                              | Aggregate all output data         | Spotify                     | None                      | this node tidy up things. feel free to call from other workflow.                                |
| Sticky Note               | Sticky Note                      | Workflow overview and credits     | None                        | None                      | ## Taiwan Indie Music Recommend Template by [n8nguide](https://www.threads.com/@n8nguide.tw)<br>This n8n workflow recommends Taiwan indie music based on a user's city, mood, birthday, today's weather, and zodiac sign. Here's a concise overview:... |
| Sticky Note1              | Sticky Note                      | User customization instructions   | None                        | None                      | ### - Change your location<br>### - Change Your Birthday<br>### - Change Your prefer output language |
| Sticky Note2              | Sticky Note                      | Credential update reminder        | None                        | None                      | ### update credential                                                                           |
| Sticky Note3              | Sticky Note                      | Final output node explanation     | None                        | None                      | this node tidy up things. feel free to call from other workflow.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No special configuration. This node starts the workflow.

2. **Create Set Node for User Inputs:**  
   - Name: `infomation`  
   - Type: Set  
   - Configure fields:  
     - `City` (string): e.g., "Taipei"  
     - `Mood` (string): e.g., "Happy"  
     - `birthday` (string): e.g., "1996/11/21" (format YYYY/MM/DD)  
     - `Reply Language` (string): e.g., "zh-tw" (output language code)  
   - Connect output of Manual Trigger to this node.

3. **Create OpenAI LangChain Chat Node (GPT-4o-mini):**  
   - Name: `get song recommendation`  
   - Type: OpenAI LangChain Chat  
   - Credentials: Configure OpenAI API key (e.g., "tkicorp n8n API with $10").  
   - Model: Select `gpt-4o-mini-search-preview`.  
   - Message setup:  
     - System message: instruct AI to act as Taiwan indie music expert, steps include fetching weather, zodiac, fortune, recommendation, explanation, and optional song highlight.  
     - Use expressions to inject variables from `infomation` node: `City`, `birthday`, and `Reply Language`.  
   - Connect output of `infomation` to this node.

4. **Create LangChain Information Extractor Node:**  
   - Name: `Information Extractor`  
   - Type: LangChain Information Extractor  
   - Input text expression: use AI response content from `get song recommendation`.  
   - Define JSON schema example with keys: "todays date", "city", "weather", "star sign", "fortune", "song", "artist", "additional infomation".  
   - Connect output of `get song recommendation` to this node.

5. **Create Spotify Node:**  
   - Name: `Spotify`  
   - Type: Spotify (search tracks)  
   - Credentials: Connect your Spotify OAuth2 account.  
   - Resource: Track  
   - Operation: Search  
   - Query: Use expression combining artist and song from `Information Extractor` output, e.g., `{{$json.output.artist}} {{$json.output.song}}`.  
   - Limit: 1  
   - Connect output of `Information Extractor` to this node.

6. **Create Set Node for Final Output:**  
   - Name: `Final Output`  
   - Type: Set  
   - Assign multiple fields under `output` property using expressions from `Information Extractor` and `Spotify` nodes:  
     - `todays date`  
     - `city`  
     - `weather`  
     - `star sign`  
     - `fortune`  
     - `artist`  
     - `song`  
     - `songlink` (Spotify external URL)  
     - `additional infomation`  
   - Connect output of `Spotify` to this node.

7. **Optional: Add Sticky Notes for Documentation:**  
   - Add sticky notes near respective nodes with instructions or credits as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                   | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow integrates AI language models, astrology, weather data, and Spotify to create personalized Taiwan indie music recommendations.                                                                                                                                                 | Workflow purpose overview                                                                        |
| Created and documented by [n8nguide](https://www.threads.com/@n8nguide.tw), providing a template for advanced n8n users to build AI-enhanced music recommendation systems.                                                                                                                     | Creator and credit link                                                                          |
| To customize this workflow, change the `City`, `birthday`, and `Reply Language` fields in the `infomation` node.                                                                                                                                                                            | User customization instructions                                                                 |
| Ensure Spotify OAuth2 credentials and OpenAI API keys are valid and updated to avoid authentication errors.                                                                                                                                                                                  | Credential maintenance note                                                                      |
| This workflow’s final output node can be reused or invoked by other workflows to integrate music recommendations into larger automation solutions.                                                                                                                                            | Workflow modularity note                                                                         |

---

**Disclaimer:** The provided content is generated from an automated n8n workflow. All data handled is legal and public. This workflow complies with current content policies and contains no illicit or offensive elements.