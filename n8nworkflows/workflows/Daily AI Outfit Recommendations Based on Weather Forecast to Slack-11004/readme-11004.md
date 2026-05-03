Daily AI Outfit Recommendations Based on Weather Forecast to Slack

https://n8nworkflows.xyz/workflows/daily-ai-outfit-recommendations-based-on-weather-forecast-to-slack-11004


# Daily AI Outfit Recommendations Based on Weather Forecast to Slack

### 1. Workflow Overview

This workflow, titled **"Today's Outfit Forecast"**, automates the daily generation and delivery of a weather-based outfit recommendation. It is designed to run every day at 6 AM, fetch the current weather forecast for Tokyo, generate a stylistic outfit suggestion using AI, create an image card displaying the weather and recommendation, and post this image to a designated Slack channel.

The workflow is logically divided into four main blocks:

- **1.1 Get & Format Weather Data:** Triggered daily, it retrieves weather data for Tokyo from OpenWeatherMap and extracts key information (description, current, max, and min temperatures).

- **1.2 Generate AI Outfit Advice:** Sends the formatted weather data to an AI language model acting as a stylist, which produces a gender-neutral, concise outfit recommendation.

- **1.3 Create Forecast Image Card:** Combines the weather data and AI-generated advice to dynamically create a visually styled vertical image card summarizing the forecast and outfit advice.

- **1.4 Send Image to Slack:** Uploads the generated image card to a specified Slack channel with a friendly morning message.

---

### 2. Block-by-Block Analysis

#### 2.1 Block: Get & Format Weather Data

**Overview:**  
This block triggers daily at 6 AM, fetches the weather data for Tokyo using the OpenWeatherMap API, and extracts essential weather details into a structured format for downstream use.

**Nodes Involved:**  
- Daily 6AM Trigger  
- Get Weather Data  
- Format Weather Data

**Node Details:**

- **Daily 6AM Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow every day at 6 AM.  
  - *Configuration:* Scheduled to trigger at hour 6 daily.  
  - *Connections:* Output leads to "Get Weather Data".  
  - *Failures:* Possible issues include scheduler misconfiguration or n8n not running at scheduled time.

- **Get Weather Data**  
  - *Type:* OpenWeatherMap Node  
  - *Role:* Fetches current weather data for Tokyo.  
  - *Configuration:* City name set to "Tokyo". Requires valid OpenWeatherMap API credentials (not shown).  
  - *Connections:* Output leads to "Format Weather Data".  
  - *Failures:* API key errors, network issues, or invalid city name cause failures.

- **Format Weather Data**  
  - *Type:* Set Node  
  - *Role:* Extracts and renames weather fields into simplified JSON keys for later nodes.  
  - *Configuration:* Extracts:  
    - `weatherDescription` from `$json.weather[0].description`  
    - `tempCurrent` from `$json.main.temp`  
    - `tempMax` from `$json.main.temp_max`  
    - `tempMin` from `$json.main.temp_min`  
  - *Connections:* Output leads to "Generate Outfit Advice".  
  - *Edge Cases:* Missing or malformed API response could cause expression failures.

---

#### 2.2 Block: Generate AI Outfit Advice

**Overview:**  
This block sends the formatted weather data to an AI stylist agent, which generates a gender-neutral outfit recommendation tailored to the day's weather.

**Nodes Involved:**  
- Generate Outfit Advice  
- OpenRouter Chat Model

**Node Details:**

- **Generate Outfit Advice**  
  - *Type:* LangChain Agent Node  
  - *Role:* Constructs a prompt with weather details and sends it to the AI model to generate outfit advice.  
  - *Configuration:*  
    - Input text includes weather description, current, max, and min temperatures.  
    - System message instructs AI to act as a stylist using gender-neutral language and to produce 4-6 lines mentioning tops, bottoms, shoes, outerwear, and a closing advice line.  
    - Prompt type set to "define" for fixed instruction.  
  - *Connections:* Output feeds into "Store Advice Text".  
  - *Dependencies:* Uses "OpenRouter Chat Model" as the language model provider.  
  - *Edge Cases:* AI model errors, rate limits, malformed prompt expressions.  
  - *Version:* LangChain integration (version 3).  

- **OpenRouter Chat Model**  
  - *Type:* LangChain Language Model Node  
  - *Role:* Provides AI chat capabilities to the Agent node.  
  - *Configuration:* Requires OpenRouter API key configured in credentials.  
  - *Connections:* Linked as language model input to "Generate Outfit Advice".  
  - *Failures:* Authentication errors, quota exceeded, network issues.

---

#### 2.3 Block: Create Forecast Image Card

**Overview:**  
This block creates a stylish, vertically oriented image card combining the date, weather information, and AI-generated outfit advice as text overlays.

**Nodes Involved:**  
- Store Advice Text  
- Create Image Card

**Node Details:**

- **Store Advice Text**  
  - *Type:* Set Node  
  - *Role:* Stores the AI-generated outfit text under the key `outfitText` for easy reference in the image creation node.  
  - *Configuration:* Assigns `outfitText` = output from "Generate Outfit Advice".  
  - *Connections:* Output feeds into "Create Image Card".  
  - *Failures:* Expression errors if previous node output missing.

- **Create Image Card**  
  - *Type:* Edit Image Node  
  - *Role:* Dynamically creates an image with a light background and overlays multiple text blocks: date/location, weather summary, and outfit advice.  
  - *Configuration:*  
    - Image size: 1080x1920 pixels (portrait).  
    - Background color: light blue (#E8F4F8).  
    - Text overlays (with positions and font sizes):  
      - Date and city at top (large font).  
      - Weather description and temperatures below date.  
      - AI outfit advice wrapped to ~35 characters per line.  
    - Text colors vary for emphasis.  
  - *Key Expressions:* Uses current date formatted `yyyy-MM-dd`, weather data from "Format Weather Data", outfit advice from "Store Advice Text".  
  - *Connections:* Output leads to "Upload a file".  
  - *Failures:* Text expression errors, image generation failures.

---

#### 2.4 Block: Send Image to Slack

**Overview:**  
Uploads the generated image card file to a pre-configured Slack channel with a morning greeting.

**Nodes Involved:**  
- Upload a file

**Node Details:**

- **Upload a file**  
  - *Type:* Slack Node  
  - *Role:* Uploads the created image file to Slack.  
  - *Configuration:*  
    - Resource: File upload.  
    - File name: "outfit-forecast.png".  
    - Initial comment: "Good morning! Here is today's outfit forecast."  
    - Channel ID: User must specify their Slack channel.  
    - Requires Slack OAuth2 credentials configured in n8n.  
  - *Connections:* Receives input from "Create Image Card".  
  - *Failures:* Slack auth errors, channel not configured, file size limits.

---

### 3. Summary Table

| Node Name           | Node Type                        | Functional Role                       | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                               |
|---------------------|---------------------------------|------------------------------------|-----------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Daily 6AM Trigger    | Schedule Trigger                | Starts workflow daily at 6 AM      | —                     | Get Weather Data        |                                                                                                                           |
| Get Weather Data     | OpenWeatherMap                  | Fetch weather data for Tokyo       | Daily 6AM Trigger     | Format Weather Data     | **Setup:** Add API key to this node; change city as needed.                                                               |
| Format Weather Data  | Set                            | Extract and format weather info    | Get Weather Data       | Generate Outfit Advice  |                                                                                                                           |
| Generate Outfit Advice | LangChain Agent                | Generate AI outfit recommendation  | Format Weather Data    | Store Advice Text       | Sends weather to AI stylist for gender-neutral outfit advice.                                                             |
| OpenRouter Chat Model | LangChain Language Model       | AI model for text generation       | —                     | Generate Outfit Advice  | Add OpenRouter API key to this node.                                                                                       |
| Store Advice Text    | Set                            | Store AI outfit text                | Generate Outfit Advice | Create Image Card       |                                                                                                                           |
| Create Image Card    | Edit Image                     | Create styled image with weather & outfit info | Store Advice Text      | Upload a file           | Image size 1080x1920 px, light blue background; text overlays date, weather, outfit advice.                                |
| Upload a file        | Slack                          | Upload image to Slack channel      | Create Image Card      | —                       | **IMPORTANT:** Configure Slack channel and credentials here.                                                               |
| Main Overview        | Sticky Note                    | Workflow description and setup     | —                     | —                       | Explains how workflow runs, setup steps for API keys and Slack channel.                                                    |
| Group 1              | Sticky Note                    | Block 1 description                 | —                     | —                       | Describes Get & Format Weather block.                                                                                      |
| Group 2              | Sticky Note                    | Block 2 description                 | —                     | —                       | Describes AI outfit advice generation block.                                                                               |
| Group 3              | Sticky Note                    | Block 3 description                 | —                     | —                       | Describes image creation block.                                                                                            |
| Group 4              | Sticky Note                    | Block 4 description                 | —                     | —                       | Describes Slack upload block.                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow named "Today's Outfit Forecast".**

2. **Add a Schedule Trigger node: "Daily 6AM Trigger"**  
   - Set to trigger daily at 6:00 AM.  
   - No credentials needed.

3. **Add an OpenWeatherMap node: "Get Weather Data"**  
   - Connect trigger output to this node's input.  
   - Set `cityName` parameter to `"Tokyo"`.  
   - Configure and add your OpenWeatherMap API key credentials.

4. **Add a Set node: "Format Weather Data"**  
   - Connect from "Get Weather Data".  
   - Create four assignments:  
     - `weatherDescription` = `{{$json.weather[0].description}}` (string)  
     - `tempCurrent` = `{{$json.main.temp}}` (number)  
     - `tempMax` = `{{$json.main.temp_max}}` (number)  
     - `tempMin` = `{{$json.main.temp_min}}` (number)

5. **Add a LangChain Agent node: "Generate Outfit Advice"**  
   - Connect from "Format Weather Data".  
   - In the text parameter, insert:  
     ```
     Today's Weather: {{$json.weatherDescription}}
     Current Temp: {{$json.tempCurrent}}°C
     Max Temp: {{$json.tempMax}}°C
     Min Temp: {{$json.tempMin}}°C
     ```  
   - Set system message:  
     ```
     You are a stylist. Based on the provided weather and temperatures, suggest a casual outfit for the day in English.
     Requirements:
     - Use gender-neutral expressions.
     - Specifically mention tops, bottoms, shoes, and outerwear if necessary.
     - The entire text should be about 4-6 lines.
     - The last line should be a short piece of advice for the day.
     ```  
   - Set prompt type to "define".  
   - Connect this node to an OpenRouter Chat Model node as the language model.

6. **Add a LangChain Chat Model node: "OpenRouter Chat Model"**  
   - No input connections; this node serves as the AI model provider for the Agent node.  
   - Configure your OpenRouter API credentials.

7. **Add a Set node: "Store Advice Text"**  
   - Connect from "Generate Outfit Advice".  
   - Assign a new string variable `outfitText` with the value `{{$json.output}}` (the AI-generated text).

8. **Add an Edit Image node: "Create Image Card"**  
   - Connect from "Store Advice Text".  
   - Configure multi-step image operations:  
     - Create a new image 1080x1920 px with background color `#E8F4F8`.  
     - Add text layers:  
       - Date and location: `{{$now.toFormat('yyyy-MM-dd')}} Tokyo` at position (90,150), font size 48, color `#333333`.  
       - Weather description and temps:  
         ```
         Weather: {{$json.weatherDescription}}
         Current: {{$json.tempCurrent}}°C   Max: {{$json.tempMax}}°C   Min: {{$json.tempMin}}°C
         ```  
         at (90,350), font size 42, color `#55555`.  
       - Outfit advice text: `{{$json.outfitText}}` at (90,600), font size 48, line length 35 characters.  
   - Use expressions to reference data from previous nodes as shown.

9. **Add a Slack node: "Upload a file"**  
   - Connect from "Create Image Card".  
   - Set resource to "file".  
   - File name: `outfit-forecast.png`.  
   - Initial comment: "Good morning! Here is today's outfit forecast."  
   - Select your Slack channel (must be configured).  
   - Add Slack OAuth2 credentials.

10. **Add sticky notes for documentation (optional but recommended):**  
    - Overview of workflow purpose and setup instructions.  
    - Block descriptions for each logical section.

11. **Activate the workflow and test.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                        | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow runs daily at 6 AM to help users start their day with AI-curated outfit advice based on real-time weather.                                                                             | Workflow purpose                                                                                         |
| OpenWeatherMap API key and OpenRouter API key are required to enable weather data retrieval and AI text generation respectively.                                                                    | API credential setup instructions                                                                       |
| Slack channel selection and OAuth2 credentials must be configured in the Slack node for successful image posting.                                                                                   | Slack integration setup                                                                                   |
| Image card size is optimized for vertical mobile viewing (1080x1920 pixels), suitable for Slack and mobile devices.                                                                                 | Image design rationale                                                                                   |
| The AI stylist prompt is designed to produce concise, gender-neutral advice covering key clothing items and a final tip.                                                                            | Prompt design details                                                                                    |
| Official n8n documentation on OpenWeatherMap node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.openWeatherMap/                                                                | Node documentation                                                                                       |
| LangChain integration reference: https://docs.n8n.io/integrations/ai/langchain/                                                                                                                     | Node documentation                                                                                       |
| Slack file upload documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/                                                                                           | Node documentation                                                                                       |

---

**Disclaimer:** The text and workflow content originate exclusively from an automated process created with n8n, strictly complying with content policies. No illegal or protected data is included. All processed data is lawful and public.