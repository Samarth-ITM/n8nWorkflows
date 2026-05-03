Daily Personalized Air & Pollen Health Alerts with Ambee API and AI via Email

https://n8nworkflows.xyz/workflows/daily-personalized-air---pollen-health-alerts-with-ambee-api-and-ai-via-email-3699


# Daily Personalized Air & Pollen Health Alerts with Ambee API and AI via Email

### 1. Workflow Overview

This workflow automates the daily delivery of personalized air quality and pollen health alerts via email, leveraging real-time environmental data from Ambee’s APIs and AI-generated health advice. It is designed primarily for individuals sensitive to environmental conditions—such as allergy sufferers or people with respiratory issues—providing them with actionable, easy-to-understand summaries and recommendations.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Input Setup:** Automates daily trigger and sets user location and profile data.
- **1.2 Data Retrieval:** Fetches current air quality and pollen data from Ambee APIs based on the user’s coordinates.
- **1.3 AI Processing:** Uses an AI agent powered by OpenAI’s GPT-4 to generate a friendly, personalized summary and health tips based on the retrieved data and user profile.
- **1.4 Email Delivery:** Sends the AI-generated message via Gmail to the user’s email address.
- **1.5 Documentation & Guidance:** Sticky notes provide setup instructions and contextual information for API keys, coordinates, and user profile configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Input Setup

**Overview:**  
This block initiates the workflow daily at a set time and prepares the necessary inputs: geographic coordinates and user profile details.

**Nodes Involved:**  
- Schedule Trigger  
- Set Your Location Coordinates  
- Set User Profile  
- Sticky Note1 (Location instructions)  
- Sticky Note2 (User profile instructions)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Fires the workflow every day at 7:00 AM to automate data retrieval and messaging.  
  - *Configuration:* Interval trigger set to hour 7 daily.  
  - *Connections:* Outputs to "Set Your Location Coordinates".  
  - *Potential Failures:* None typical; ensure n8n instance is running at trigger time.

- **Set Your Location Coordinates**  
  - *Type:* Set  
  - *Role:* Defines latitude and longitude for the location to monitor (default Braunschweig, Germany).  
  - *Configuration:* Assigns string values for `lat` = "52.267" and `lng` = "10.533".  
  - *Connections:* Outputs to "Set User Profile".  
  - *Edge Cases:* Coordinates must be valid; invalid or missing coordinates will cause API requests to fail or return no data.

- **Set User Profile**  
  - *Type:* Set  
  - *Role:* Defines user-specific parameters such as age and health sensitivities to tailor AI suggestions.  
  - *Configuration:* Assigns `Age` = "25" and `Health sensitivities` = "Allergic to Pollen".  
  - *Connections:* Outputs to "Get Air data".  
  - *Edge Cases:* Missing or incomplete profile data may reduce AI personalization quality.

- **Sticky Note1 (Location instructions)**  
  - *Type:* Sticky Note  
  - *Role:* Provides user guidance on how to find and input location coordinates.  
  - *Content Summary:* Explains how to obtain latitude and longitude, with example coordinates for Braunschweig.

- **Sticky Note2 (User profile instructions)**  
  - *Type:* Sticky Note  
  - *Role:* Explains the importance of user profile data for AI personalization.  
  - *Content Summary:* Suggests including age and health sensitivities; allows adding more info if desired.

---

#### 2.2 Data Retrieval

**Overview:**  
Fetches current air quality and pollen data from Ambee APIs using the user’s location coordinates.

**Nodes Involved:**  
- Get Air data  
- Get Pollen data  
- Sticky Note (Ambee API key instructions)

**Node Details:**

- **Get Air data**  
  - *Type:* HTTP Request  
  - *Role:* Calls Ambee’s air quality API endpoint to retrieve latest air data by latitude and longitude.  
  - *Configuration:*  
    - URL: `https://api.ambeedata.com/latest/by-lat-lng`  
    - Query parameters: `lat` and `lng` dynamically set from "Set Your Location Coordinates" node.  
    - Header: `x-api-key` required (user must input their Ambee API key here).  
    - Redirects enabled.  
  - *Connections:* Outputs to "Get Pollen data".  
  - *Edge Cases:*  
    - API key missing or invalid → authentication failure.  
    - Network errors or API downtime → request failure.  
    - Invalid coordinates → empty or error response.

- **Get Pollen data**  
  - *Type:* HTTP Request  
  - *Role:* Calls Ambee’s pollen API endpoint to retrieve latest pollen data by latitude and longitude.  
  - *Configuration:*  
    - URL: `https://api.ambeedata.com/latest/pollen/by-lat-lng`  
    - Query parameters: same as air data node.  
    - Header: same API key header.  
  - *Connections:* Outputs to "AI Agent".  
  - *Edge Cases:* Same as "Get Air data".

- **Sticky Note (Ambee API key instructions)**  
  - *Type:* Sticky Note  
  - *Role:* Provides detailed steps to obtain a valid Ambee API key, emphasizing use of work or university email addresses.  
  - *Content Summary:*  
    - Sign up at https://www.getambee.com with organizational email.  
    - Copy API key and paste into HTTP Request headers.  
    - Personal emails like Gmail/Outlook are not accepted.

---

#### 2.3 AI Processing

**Overview:**  
Processes the combined environmental data and user profile through an AI agent to generate a personalized, friendly summary and health advice.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Think

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Orchestrates AI prompt processing, combining environmental data and user profile to generate output text.  
  - *Configuration:*  
    - Uses a detailed system message prompt that:  
      - Introduces the assistant’s role as a caring helper.  
      - Provides placeholders for environmental data (air and pollen) and user profile.  
      - Specifies output format: friendly summary with actual data values and 3–5 practical suggestions.  
    - Pulls data dynamically from "Get Air data" and "Get Pollen data" nodes, and user profile node.  
  - *Connections:*  
    - AI language model input connected to "OpenAI Chat Model".  
    - Tool input connected to "Think" node.  
    - Output connected to "Gmail" node (email sending).  
  - *Edge Cases:*  
    - Missing or malformed data may cause prompt errors or incomplete output.  
    - API rate limits or OpenAI errors may cause failures.  
    - Expression errors in prompt variables could break message generation.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides GPT-4.1 model for natural language generation.  
  - *Configuration:*  
    - Model: `gpt-4.1`  
    - Credentials: OpenAI API key configured.  
  - *Connections:* Feeds output to "AI Agent".  
  - *Edge Cases:*  
    - API key invalid or quota exceeded → request failure.  
    - Network issues → timeout or errors.

- **Think**  
  - *Type:* LangChain Tool (Think)  
  - *Role:* Supports internal AI processing steps (e.g., reasoning or intermediate steps).  
  - *Configuration:* Default.  
  - *Connections:* Connected as AI tool input to "AI Agent".  
  - *Edge Cases:* Minimal; depends on upstream data quality.

---

#### 2.4 Email Delivery

**Overview:**  
Sends the AI-generated personalized health alert email to the user.

**Nodes Involved:**  
- Gmail

**Node Details:**

- **Gmail**  
  - *Type:* Gmail Tool  
  - *Role:* Sends the personalized message via Gmail SMTP using OAuth2 credentials.  
  - *Configuration:*  
    - Recipient email: `simoroosvelt@gmail.com` (can be customized).  
    - Subject and message body dynamically set from AI Agent output using `$fromAI` expressions.  
    - Email type: plain text.  
    - Credentials: Gmail OAuth2 account configured.  
  - *Connections:* Receives input from "AI Agent".  
  - *Edge Cases:*  
    - OAuth2 token expired or invalid → authentication failure.  
    - Recipient email invalid → delivery failure.  
    - Network issues → send failure.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                         |
|---------------------------|--------------------------------|----------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger               | Triggers workflow daily at 7 AM        |                             | Set Your Location Coordinates |                                                                                                                     |
| Set Your Location Coordinates | Set                          | Sets latitude and longitude values     | Schedule Trigger             | Set User Profile            | Sticky Note1: Explains how to find and input location coordinates with example for Braunschweig.                    |
| Set User Profile          | Set                           | Defines user age and health sensitivities | Set Your Location Coordinates | Get Air data               | Sticky Note2: Explains importance of user profile for AI personalization.                                           |
| Get Air data              | HTTP Request                  | Fetches air quality data from Ambee API | Set User Profile             | Get Pollen data             | Sticky Note: Instructions to obtain Ambee API key with organizational email; personal emails not accepted.          |
| Get Pollen data           | HTTP Request                  | Fetches pollen data from Ambee API     | Get Air data                 | AI Agent                   | Sticky Note: Same as above.                                                                                          |
| AI Agent                 | LangChain Agent               | Generates personalized summary and tips | Get Pollen data              | Gmail                      |                                                                                                                     |
| OpenAI Chat Model         | LangChain OpenAI Chat Model   | Provides GPT-4.1 for AI text generation |                             | AI Agent                   |                                                                                                                     |
| Think                    | LangChain Tool (Think)        | Supports AI agent reasoning             |                             | AI Agent                   |                                                                                                                     |
| Gmail                    | Gmail Tool                    | Sends personalized email                | AI Agent                    |                            |                                                                                                                     |
| Sticky Note              | Sticky Note                   | Instructions for Ambee API key          |                             |                            | See above.                                                                                                          |
| Sticky Note1             | Sticky Note                   | Instructions for location coordinates   |                             |                            | See above.                                                                                                          |
| Sticky Note2             | Sticky Note                   | Instructions for user profile setup     |                             |                            | See above.                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 7:00 AM (hour 7).  
   - Connect output to next node.

2. **Add a Set node named "Set Your Location Coordinates":**  
   - Assign two string fields:  
     - `lat` = "52.267"  
     - `lng` = "10.533" (Braunschweig example)  
   - Connect input from Schedule Trigger.  
   - Connect output to next node.

3. **Add a Set node named "Set User Profile":**  
   - Assign string fields:  
     - `Age` = "25"  
     - `Health sensitivities` = "Allergic to Pollen"  
   - Connect input from "Set Your Location Coordinates".  
   - Connect output to next node.

4. **Add an HTTP Request node named "Get Air data":**  
   - Method: GET  
   - URL: `https://api.ambeedata.com/latest/by-lat-lng`  
   - Query parameters:  
     - `lat` = expression: `={{ $('Set Your Location Coordinates').item.json.lat }}`  
     - `lng` = expression: `={{ $('Set Your Location Coordinates').item.json.lng }}`  
   - Headers:  
     - `x-api-key`: your Ambee API key (must be obtained as per instructions).  
   - Enable redirects.  
   - Connect input from "Set User Profile".  
   - Connect output to next node.

5. **Add an HTTP Request node named "Get Pollen data":**  
   - Method: GET  
   - URL: `https://api.ambeedata.com/latest/pollen/by-lat-lng`  
   - Query parameters same as "Get Air data".  
   - Header: same `x-api-key`.  
   - Connect input from "Get Air data".  
   - Connect output to next node.

6. **Add a LangChain Agent node named "AI Agent":**  
   - Configure with a system message prompt that:  
     - Introduces the assistant as a caring helper.  
     - Includes placeholders for environmental data and user profile, dynamically referencing:  
       - Air data fields from "Get Air data" node JSON (e.g., AQI, PM2.5, pollutant).  
       - Pollen data from "Get Pollen data" node JSON.  
       - User profile fields from "Set User Profile".  
     - Specifies output format: friendly summary with actual values and 3–5 personalized suggestions.  
   - Set prompt type to "define".  
   - Connect input from "Get Pollen data".  
   - Connect AI language model input to next node.

7. **Add a LangChain OpenAI Chat Model node named "OpenAI Chat Model":**  
   - Model: GPT-4.1  
   - Credentials: configure with your OpenAI API key.  
   - Connect output to "AI Agent" language model input.

8. **Add a LangChain Tool node named "Think":**  
   - Default configuration.  
   - Connect as AI tool input to "AI Agent".

9. **Add a Gmail node named "Gmail":**  
   - Configure recipient email (e.g., your email address).  
   - Subject: set to expression `$fromAI('Subject', '', 'string')` to receive AI-generated subject.  
   - Message: set to expression `$fromAI('Message', '', 'string')` to receive AI-generated message body.  
   - Email type: plain text.  
   - Credentials: configure Gmail OAuth2 credentials.  
   - Connect input from "AI Agent".

10. **Add Sticky Notes as needed for documentation:**  
    - Instructions for Ambee API key acquisition.  
    - Instructions for location coordinates setup.  
    - Instructions for user profile setup.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Ambee API key requires registration with a work or university email; personal emails like Gmail or Outlook are not accepted.                                                                                                            | https://www.getambee.com                         |
| Coordinates can be found via Google Maps or GPS services; example used is Braunschweig, Germany (lat: 52.267, lng: 10.533).                                                                                                              | Location setup instructions in Sticky Note1     |
| User profile customization allows tailoring AI suggestions; minimum fields are age and health sensitivities (e.g., asthma, pollen allergy).                                                                                              | User profile instructions in Sticky Note2       |
| AI prompt is designed for friendly, supportive messaging suitable for email but can be adapted for other platforms.                                                                                                                     | AI Agent system message prompt                   |
| Gmail node uses OAuth2 for authentication; ensure credentials are valid and token refreshed as needed.                                                                                                                                   | Gmail OAuth2 credential setup                     |
| OpenAI GPT-4.1 model is used; ensure API key has sufficient quota and permissions.                                                                                                                                                         | OpenAI API key setup                             |

---

This documentation provides a detailed, structured reference to understand, reproduce, and maintain the "Daily Personalized Air & Pollen Health Alerts" workflow, ensuring clarity on each component’s function, configuration, and potential issues.