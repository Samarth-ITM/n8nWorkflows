Send asteroid alerts from NASA to LINE using OpenAI and DeepL

https://n8nworkflows.xyz/workflows/send-asteroid-alerts-from-nasa-to-line-using-openai-and-deepl-11304


# Send asteroid alerts from NASA to LINE using OpenAI and DeepL

### 1. Workflow Overview

This workflow is designed to monitor potentially hazardous asteroids approaching Earth using NASA’s Near-Earth Object Web Service (NeoWs) API. It runs daily at a scheduled time, fetches asteroid data for the current day, filters and analyzes the threat level of detected asteroids, and sends notifications accordingly via LINE Notify. The alerts are generated with creative Sci-Fi style warnings using OpenAI and translated into Japanese using DeepL before being sent. If no threats are detected, a peaceful status report is sent instead.

Logical blocks:

- **1.1 Scheduled Data Retrieval:** Triggers the workflow daily and fetches NASA asteroid data for the current date.
- **1.2 Threat Analysis:** Filters the data to identify potentially hazardous asteroids and calculates their relative distance from the Moon.
- **1.3 Threat Level Decision:** Determines if any threats exist and branches accordingly.
- **1.4 AI Alert Generation & Translation:** Uses OpenAI to generate a Sci-Fi styled alert message for threats, then translates it into Japanese with DeepL.
- **1.5 Notification Dispatch:** Sends either the translated danger alert or a peace report via LINE Notify.
- **1.6 Documentation and Notes:** Includes sticky notes with descriptions, instructions, and step-wise explanations embedded in the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Retrieval

**Overview:**  
This block initiates the workflow daily at 9 AM and retrieves NASA’s asteroid data for the current day.

**Nodes Involved:**  
- Daily Schedule1  
- Get Asteroid Data1  

**Node Details:**

- **Daily Schedule1**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers the workflow once daily at 9:00 AM.  
  - *Configuration:* Interval trigger set to triggerAtHour = 9.  
  - *Inputs:* None (trigger).  
  - *Outputs:* Connects to Get Asteroid Data1.  
  - *Edge Cases:* Ensure server time zone matches intended schedule; failures if n8n instance is down at trigger time.

- **Get Asteroid Data1**  
  - *Type:* NASA Node  
  - *Role:* Fetches Near-Earth Object feed from NASA’s NeoWs API for the current date.  
  - *Configuration:* Resource set to asteroidNeoFeed; startDate and endDate both dynamically set to today’s date using expression `{{$today.format('YYYY-MM-DD')}}`.  
  - *Inputs:* Trigger from Daily Schedule1.  
  - *Outputs:* Outputs raw NASA data JSON to Filter & Calculate Distance1.  
  - *Edge Cases:* API quota exceeded, invalid or missing NASA API key, network errors, or no data returned for the day.

---

#### 1.2 Threat Analysis

**Overview:**  
Analyzes the fetched asteroid data to identify potentially hazardous asteroids and calculates their miss distance relative to the Moon’s average distance.

**Nodes Involved:**  
- Filter & Calculate Distance1  

**Node Details:**

- **Filter & Calculate Distance1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Filters NASA’s asteroid data for only those flagged as potentially hazardous and calculates the miss distance ratio relative to lunar distance.  
  - *Configuration:* Custom JavaScript code:  
    - Extracts the date key from the API response to access asteroid data.  
    - Filters asteroids with `is_potentially_hazardous_asteroid === true`.  
    - For each dangerous asteroid, calculates miss distance in km and divides by moon distance (384,400 km), rounding to 2 decimals.  
    - If no hazardous asteroids found, outputs a peace message.  
  - *Inputs:* Raw NASA asteroid data from Get Asteroid Data1.  
  - *Outputs:* JSON items flagged with `"type": "DANGER"` and relevant asteroid details or `"type": "PEACE"` with a peace message.  
  - *Edge Cases:*  
    - Empty or malformed API response.  
    - Missing or unexpected data structure.  
    - Division by zero should not occur due to fixed moonDistance constant.  
    - Potential runtime errors in JS if data keys are missing.

---

#### 1.3 Threat Level Decision

**Overview:**  
Evaluates the type flag from the previous node and routes the workflow to either alert generation or peace reporting paths.

**Nodes Involved:**  
- Check Threat Level1  

**Node Details:**

- **Check Threat Level1**  
  - *Type:* Switch  
  - *Role:* Routes workflow based on the `type` property in the JSON.  
  - *Configuration:* Two cases:  
    - `type == "DANGER"` routes to Generate SF Alert1 (AI generation).  
    - `type == "PEACE"` routes to Send Peace Report1 (peace notification).  
  - *Inputs:* Filtered asteroid data or peace message from Filter & Calculate Distance1.  
  - *Outputs:* Branches to either Generate SF Alert1 or Send Peace Report1.  
  - *Edge Cases:* Unexpected or missing `type` field causes no output; may require default/fallback handling.

---

#### 1.4 AI Alert Generation & Translation (Danger Path)

**Overview:**  
Generates a Sci-Fi styled alert message describing the asteroid threat using OpenAI, then translates it into Japanese using DeepL.

**Nodes Involved:**  
- Generate SF Alert1  
- Translate Alert1  

**Node Details:**

- **Generate SF Alert1**  
  - *Type:* OpenAI (Chat)  
  - *Role:* Generates a creative alert message based on asteroid data input.  
  - *Configuration:* Uses OpenAI chat completion API with default options. Input prompt is dynamically constructed from asteroid data (not explicitly detailed in the JSON but implied).  
  - *Inputs:* Receives asteroid threat data from Check Threat Level1.  
  - *Outputs:* Produces generated alert text under `$json.message.content`.  
  - *Credentials:* Requires configured OpenAI API key.  
  - *Edge Cases:* API rate limits, network errors, prompt failures, or empty response.  
  - *Version:* Compatible with n8n OpenAI node v1+.  
  - *Note:* Prompt design critical for relevant alerts.

- **Translate Alert1**  
  - *Type:* DeepL  
  - *Role:* Translates generated alert text from English to Japanese.  
  - *Configuration:*  
    - Text input is expression `{{$json.message.content}}` from OpenAI output.  
    - Target language set to Japanese (`JA`).  
  - *Inputs:* Receives OpenAI generated alert.  
  - *Outputs:* Produces `translatedText` JSON field with Japanese translation.  
  - *Credentials:* Requires DeepL API key.  
  - *Edge Cases:* Failed translation due to API limits, unsupported characters, or empty input.  
  - *Note:* Target language configurable; can be removed to send English alerts.

---

#### 1.5 Notification Dispatch

**Overview:**  
Sends the final alert messages via LINE Notify — either a danger alert with translated text or a daily peace report.

**Nodes Involved:**  
- Send Danger Alert1  
- Send Peace Report1  

**Node Details:**

- **Send Danger Alert1**  
  - *Type:* LINE  
  - *Role:* Sends emergency alerts to LINE Notify recipients.  
  - *Configuration:*  
    - Message includes alert emoji and prefix, plus translated alert text from DeepL node (`{{$json.translatedText}}`).  
  - *Inputs:* Translated alert from Translate Alert1.  
  - *Outputs:* None (end node).  
  - *Credentials:* Requires LINE Notify token.  
  - *Edge Cases:* Invalid or expired token, network issues, or message length limits.

- **Send Peace Report1**  
  - *Type:* LINE  
  - *Role:* Sends a daily peace report when no threats are detected.  
  - *Configuration:*  
    - Static message reporting no threats and wishing a peaceful day.  
  - *Inputs:* Routed from Check Threat Level1 when no danger found.  
  - *Outputs:* None (end node).  
  - *Credentials:* Requires LINE Notify token.  
  - *Edge Cases:* Same as Send Danger Alert1.

---

#### 1.6 Documentation and Notes

**Overview:**  
Provides embedded documentation and stepwise notes to explain workflow purpose, setup, and configuration instructions.

**Nodes Involved:**  
- Template Description1  
- Step 1 Note  
- Step 2 Note  
- Step 3 Note  

**Node Details:**

- **Template Description1**  
  - *Type:* Sticky Note  
  - *Role:* Detailed workflow description, setup instructions, API key requirements, and user guidance.  
  - *Content Highlights:*  
    - Explains the five main functional steps.  
    - Lists required API keys for NASA, OpenAI, DeepL, and LINE.  
    - Notes on alternative notification channels.  
    - Version and credential requirements.  
  - *Position:* Visible in workspace for user reference.

- **Step 1 Note, Step 2 Note, Step 3 Note**  
  - *Type:* Sticky Notes  
  - *Role:* Summarize each major processing step in the workflow visually near corresponding nodes.  
  - *Content:*  
    - Step 1: Fetch & Analyze hazardous asteroids.  
    - Step 2: AI generation and translation for danger alerts.  
    - Step 3: Notification sending via LINE.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                       | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                  |
|--------------------------|--------------------|------------------------------------|-----------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Daily Schedule1          | Schedule Trigger   | Daily trigger at 9 AM               | -                     | Get Asteroid Data1     | Step 1: Fetch & Analyze                                                                     |
| Get Asteroid Data1       | NASA               | Fetches asteroid data for today    | Daily Schedule1       | Filter & Calculate Distance1 | Step 1: Fetch & Analyze                                                                     |
| Filter & Calculate Distance1 | Code (JavaScript)   | Filters hazardous asteroids and computes relative distance | Get Asteroid Data1     | Check Threat Level1    | Step 1: Fetch & Analyze                                                                     |
| Check Threat Level1      | Switch             | Routes workflow by threat type     | Filter & Calculate Distance1 | Generate SF Alert1 / Send Peace Report1 | Step 1: Fetch & Analyze                                                                     |
| Generate SF Alert1       | OpenAI Chat        | Generates Sci-Fi alert message     | Check Threat Level1 (DANGER) | Translate Alert1       | Step 2: AI Generation (Danger Path)                                                         |
| Translate Alert1         | DeepL              | Translates alert message to Japanese | Generate SF Alert1     | Send Danger Alert1     | Step 2: AI Generation (Danger Path)                                                         |
| Send Danger Alert1       | LINE               | Sends translated danger alert      | Translate Alert1      | -                     | Step 3: Notification                                                                        |
| Send Peace Report1       | LINE               | Sends peace report if no threats   | Check Threat Level1 (PEACE) | -                     | Step 3: Notification                                                                        |
| Template Description1    | Sticky Note        | Workflow description and instructions | -                     | -                     | Contains detailed project overview and setup instructions                                   |
| Step 1 Note              | Sticky Note        | Notes on Step 1: Fetch & Analyze   | -                     | -                     | Step 1: Fetch & Analyze                                                                     |
| Step 2 Note              | Sticky Note        | Notes on Step 2: AI Generation     | -                     | -                     | Step 2: AI Generation (Danger Path)                                                         |
| Step 3 Note              | Sticky Note        | Notes on Step 3: Notification      | -                     | -                     | Step 3: Notification                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add `Schedule Trigger` node** named `Daily Schedule1`:  
   - Set to trigger daily at 9:00 AM (triggerAtHour = 9).  
   - No credentials needed.

3. **Add `NASA` node** named `Get Asteroid Data1`:  
   - Resource: `asteroidNeoFeed`.  
   - Set `startDate` and `endDate` to today's date using expression: `{{$today.format('YYYY-MM-DD')}}`.  
   - Configure NASA API credentials (API key from https://api.nasa.gov).  
   - Connect `Daily Schedule1` output to this node's input.

4. **Add `Code` node** named `Filter & Calculate Distance1`:  
   - Language: JavaScript.  
   - Paste the following JS code to:  
     - Extract today's asteroid list from `near_earth_objects` keyed by date.  
     - Filter for `is_potentially_hazardous_asteroid == true`.  
     - Calculate miss distance ratio relative to lunar distance (384,400 km).  
     - If none found, output peace message.  
   - Connect output of `Get Asteroid Data1` to this node.

5. **Add `Switch` node** named `Check Threat Level1`:  
   - Add two rules checking `{{$json.type}}`:  
     - Equals `DANGER` → route to Generate SF Alert1.  
     - Equals `PEACE` → route to Send Peace Report1.  
   - Connect output of `Filter & Calculate Distance1` to this node.

6. **Add `OpenAI` node** named `Generate SF Alert1`:  
   - Resource: Chat.  
   - Configure with your OpenAI API key.  
   - Input: asteroid data from `Check Threat Level1` danger branch.  
   - Construct prompt dynamically to generate a Sci-Fi style alert message based on asteroid details.  
   - Connect output of `Check Threat Level1` (DANGER path) to this node.

7. **Add `DeepL` node** named `Translate Alert1`:  
   - Text input: `{{$json.message.content}}` from OpenAI output.  
   - Translate to Japanese (`JA`) or your preferred language.  
   - Configure with DeepL API key.  
   - Connect `Generate SF Alert1` output to this node.

8. **Add `LINE` node** named `Send Danger Alert1`:  
   - Message:  
     ```
     🚨 [EMERGENCY ALERT] 🚨
     {{$json.translatedText}}
     ```  
   - Configure with your LINE Notify token.  
   - Connect `Translate Alert1` output to this node.

9. **Add `LINE` node** named `Send Peace Report1`:  
   - Static message:  
     ```
     🕊️ Daily Planetary Defense Report

     Patrol complete. No threats detected in Earth's vicinity today.

     Have a peaceful day!
     (Status: All Green)
     ```  
   - Configure with your LINE Notify token.  
   - Connect `Check Threat Level1` peace branch output to this node.

10. **Add Sticky Notes** as desired to explain each major step for ease of understanding and maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                         |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| NASA API key required: Obtain at https://api.nasa.gov                                                     | NASA API credential setup                                              |
| OpenAI API key required for AI alert message generation                                                  | OpenAI API credential setup                                            |
| DeepL API key required for translation; optional if English alerts preferred                             | DeepL API credential setup                                             |
| LINE Notify token required to send alerts                                                                | LINE Notify token generation: https://notify-bot.line.me/en/           |
| Alternative notification channels like Slack, Discord, or Email can replace LINE nodes                   | Workflow can be customized for different messaging platforms           |
| Ensure n8n instance timezone matches scheduling expectations                                            | Scheduling accuracy                                                    |
| The JavaScript code assumes certain NASA API response structure; review API docs for updates            | NASA NeoWs API documentation: https://api.nasa.gov/neo/                |
| OpenAI prompt design is critical for meaningful alerts; customize prompt in Generate SF Alert1 node      | OpenAI best practices: https://platform.openai.com/docs/guides/chat    |

---

*Disclaimer:*  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.