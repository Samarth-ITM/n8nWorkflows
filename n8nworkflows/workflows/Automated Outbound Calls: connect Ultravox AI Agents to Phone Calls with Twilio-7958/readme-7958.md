Automated Outbound Calls: connect Ultravox AI Agents to Phone Calls with Twilio

https://n8nworkflows.xyz/workflows/automated-outbound-calls--connect-ultravox-ai-agents-to-phone-calls-with-twilio-7958


# Automated Outbound Calls: connect Ultravox AI Agents to Phone Calls with Twilio

### 1. Workflow Overview

This workflow automates outbound phone calls by connecting Ultravox AI agents with Twilio’s telephony service. It enables an AI-driven conversation over a live phone call, where Ultravox generates the AI voice interaction and Twilio handles the public switched telephone network (PSTN) connectivity and media streaming.

The workflow is logically divided into three main blocks:

- **1.1 Initialization and Parameter Setup:** Accepts manual trigger input and defines critical parameters such as the Ultravox agent ID, Twilio phone number, and destination phone number.
- **1.2 Ultravox AI Call Creation:** Initiates a call session with the Ultravox AI agent via their API, establishing a media stream endpoint for the voice interaction.
- **1.3 Twilio Call Execution:** Places an outbound call through Twilio, connects the call media to the Ultravox stream in real-time using TwiML.

Supplementary sticky notes provide setup instructions for Twilio and Ultravox account configuration as well as usage guidelines.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Parameter Setup

**Overview:**  
This block begins the workflow manually and sets essential parameters needed for the outbound call, specifically the AI agent identifier, the Twilio number used to place the call, and the target phone number to call.

**Nodes Involved:**  
- Start Manually  
- Set Params

**Node Details:**  

- **Start Manually**  
  - *Type & Role:* Manual Trigger node; initiates the workflow on user demand.  
  - *Configuration:* No parameters; simply waits for manual execution.  
  - *Inputs/Outputs:* No input; outputs trigger to "Set Params".  
  - *Failure Modes:* None expected; manual trigger is stable.  
  - *Notes:* Entry point for the workflow.

- **Set Params**  
  - *Type & Role:* Set node; defines static variables for the workflow.  
  - *Configuration:* Sets three string parameters:  
    - `agent_id`: Ultravox AI agent unique identifier (masked in example).  
    - `twilio_number`: Outbound caller phone number (Twilio purchased number).  
    - `phone_number`: Destination phone number to call.  
  - *Key Expressions:* Static string values assigned; no expressions used.  
  - *Inputs/Outputs:* Input from "Start Manually"; outputs to "Create Ultravox Call".  
  - *Failure Modes:* Errors if parameters are missing or malformed (e.g., invalid phone number format).  
  - *Notes:* Parameters must be customized before running the workflow.

#### 2.2 Ultravox AI Call Creation

**Overview:**  
This block calls the Ultravox API to create a new outbound call session for the specified AI agent, requesting the media stream URL that will facilitate the conversation over the phone call.

**Nodes Involved:**  
- Create Ultravox Call

**Node Details:**  

- **Create Ultravox Call**  
  - *Type & Role:* HTTP Request node; sends a POST request to Ultravox API to initiate the AI call.  
  - *Configuration:*  
    - URL dynamically constructed using `agent_id` parameter: `https://api.ultravox.ai/api/agents/{{ $json.agent_id }}/calls`  
    - HTTP method: POST  
    - Body parameter `medium` set to an object indicating Twilio integration (`{ twilio: {} }`).  
    - Authentication via HTTP header (credentials stored securely).  
  - *Key Expressions:* URL includes parameter interpolation for `agent_id`.  
  - *Inputs/Outputs:* Receives input from "Set Params"; outputs to "Twilio Call".  
  - *Failure Modes:*  
    - Authentication errors if API key is invalid.  
    - Network timeouts or API errors.  
    - Invalid agent ID causing 404 or 400 responses.  
  - *Notes:* The response must include a `joinUrl` for Twilio to connect the media stream.

#### 2.3 Twilio Call Execution

**Overview:**  
This block makes the actual phone call via Twilio, connecting the live call audio to the Ultravox AI media stream using TwiML instructions.

**Nodes Involved:**  
- Twilio Call

**Node Details:**  

- **Twilio Call**  
  - *Type & Role:* Twilio node; places an outbound call and streams audio.  
  - *Configuration:*  
    - `to`: Destination phone number, taken dynamically from "Set Params" (`phone_number`).  
    - `from`: Twilio phone number, taken from "Set Params" (`twilio_number`).  
    - `twiml`: Enabled to provide custom TwiML commands.  
    - `message`: TwiML XML connecting the call audio stream to Ultravox’s media stream URL (`<Response><Connect><Stream url="{{ $json.joinUrl }}"/></Connect></Response>`).  
  - *Key Expressions:* The TwiML `Stream url` is dynamically set from the Ultravox API response (`joinUrl`).  
  - *Inputs/Outputs:* Input from "Create Ultravox Call"; no further outputs.  
  - *Failure Modes:*  
    - Twilio authentication errors (credentials misconfiguration).  
    - Invalid or unreachable phone numbers.  
    - TwiML syntax errors causing call failures.  
    - Network or API timeouts.  
  - *Notes:* Requires valid Twilio credentials and a purchased phone number.

---

### 3. Summary Table

| Node Name         | Node Type          | Functional Role                               | Input Node(s)   | Output Node(s)         | Sticky Note                                                                                                                                                     |
|-------------------|--------------------|-----------------------------------------------|-----------------|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Start Manually    | Manual Trigger     | Entry point to start the workflow manually    | -               | Set Params             | ## STEP 3 - Make a Call - Set your 'phone_number' in "Set Params" node - Execute WF                                                                              |
| Set Params        | Set                | Defines key parameters: agent_id, phone numbers| Start Manually  | Create Ultravox Call    | ## STEP 1 - Buy a Phone Number on Twilio - Log into your [Twilio Console](https://www.twilio.com/) - Navigate to Phone Numbers - Search for Available Numbers - Select Your Number - Click "Buy This Number" to finalize the purchase - Get "Account SID" and "Auth Token" and set these in "Twilio Call" node - Set 'twilio_number' in "Set Params" node (+1xxxxxxxxxx) <br> ## STEP 2 - Create an Agent on Ultravox - Log into your [Ultravox App](https://app.ultravox.ai/) - Click on "Agents" and "New Agent" - Set Voice, Tools (optional), System Prompt - Save and get your Agent ID - Set 'agent_id' in "Set Params" node |
| Create Ultravox Call | HTTP Request       | Initiates AI call session with Ultravox agent| Set Params      | Twilio Call            |                                                                                                                                                                |
| Twilio Call       | Twilio             | Places outbound call connecting to Ultravox AI stream | Create Ultravox Call | -                      |                                                                                                                                                                |
| Sticky Note       | Sticky Note        | Workflow purpose and general overview          | -               | -                      | ## Automated Outbound Calls: connect Ultravox AI Agents to Phone Calls with Twilio This workflow transforms n8n into a call automation system, where AI agents can talk directly with people over the phone using Twilio. This workflow integrates Ultravox AI voice agents with Twilio’s telephony service to fully automate outbound phone calls Ultravox generates the AI conversation and audio, while Twilio handles the PSTN connection to the actual phone network, with the two services connected in real-time via a media stream. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a "Manual Trigger" node named `Start Manually`.  
   - No special configuration needed.

2. **Create Set Node for Parameters:**  
   - Add a "Set" node named `Set Params`.  
   - Configure three string fields:  
     - `agent_id`: your Ultravox agent ID (e.g., `xxxxxx-xxxxx-xxxxx-xxxxxxxxxxx`).  
     - `twilio_number`: your Twilio phone number in E.164 format (e.g., `+1xxxxxx`).  
     - `phone_number`: the destination phone number to call (e.g., `+1xxxxxxx`).  
   - Connect `Start Manually` output to `Set Params` input.

3. **Create HTTP Request Node to Ultravox API:**  
   - Add an "HTTP Request" node named `Create Ultravox Call`.  
   - Set method to `POST`.  
   - Set URL to `https://api.ultravox.ai/api/agents/{{ $json.agent_id }}/calls` (use expression to insert `agent_id`).  
   - In the body parameters, add a parameter named `medium` with JSON value `{ "twilio": {} }`.  
   - Enable sending the body as JSON.  
   - Set authentication to HTTP Header Auth, configuring with your Ultravox API key.  
   - Connect `Set Params` output to `Create Ultravox Call` input.

4. **Create Twilio Node to Place Call:**  
   - Add a "Twilio" node named `Twilio Call`.  
   - Set Resource to `Call`.  
   - Set `To` field dynamically to `={{ $('Set Params').item.json.phone_number }}`.  
   - Set `From` field dynamically to `={{ $('Set Params').item.json.twilio_number }}`.  
   - Enable `twiml`.  
   - Set the `message` field to the following TwiML, using expression for `joinUrl`:  
     ```
     <Response><Connect><Stream url="{{ $json.joinUrl }}"/></Connect></Response>
     ```  
   - Configure Twilio credentials with your Twilio Account SID and Auth Token.  
   - Connect `Create Ultravox Call` output to `Twilio Call` input.

5. **Optional: Add Sticky Notes for Guidance:**  
   - Add sticky notes with setup instructions for Twilio phone number purchase, Ultravox agent creation, and call execution steps.

6. **Activate Workflow:**  
   - Save and activate the workflow.  
   - Trigger manually to initiate an automated outbound call.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Buy a dedicated phone number on Twilio to enable outbound calling. Set Account SID and Auth Token in the Twilio node credentials.       | [Twilio Console](https://www.twilio.com/)       |
| Create an AI agent on Ultravox with voice settings and system prompts. Retrieve the agent ID for use in the workflow.                   | [Ultravox App](https://app.ultravox.ai/)         |
| This workflow connects Ultravox AI-generated audio streams with Twilio's PSTN network in real-time via media streaming.                  | Internal workflow design                          |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created with n8n, respecting all content policies. It contains no illegal or protected elements. All data processed is legal and publicly accessible.