Sprint Cycle Announcements with Form Input, GPT-4 and Slack

https://n8nworkflows.xyz/workflows/sprint-cycle-announcements-with-form-input--gpt-4-and-slack-4510


# Sprint Cycle Announcements with Form Input, GPT-4 and Slack

---

### 1. Workflow Overview

This workflow automates the announcement of new sprint cycle starts in a team Slack channel using form input data, GPT-4 AI text generation, and Slack integration. It is designed to gather sprint cycle details via a form, generate a warm, engaging Slack message with GPT-4 based on that input, and post it automatically to a specified Slack channel.

Logical blocks included:

- **1.1 Input Reception:** Captures sprint cycle details from a form submission.
- **1.2 Data Preparation:** Sets static or configurable data such as the team name and message tone.
- **1.3 AI Processing:** Generates a formatted Slack message using GPT-4 and a Langchain agent node.
- **1.4 Slack Posting:** Sends the generated message to a designated Slack channel.
- **1.5 Documentation & Guidance:** Sticky notes provide instructions and reminders for setup and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Captures details about the sprint cycle start via a form triggered webhook, including cycle start date, goal, and additional notes.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **Type:** Form Trigger node  
- **Technical Role:** Listens for submitted form data with specified fields.  
- **Configuration:**  
  - Form title: "Cycle Start"  
  - Fields:  
    - "When does your cycle start?" (Date, required)  
    - "What's the cycle goal?" (Textarea, required)  
    - "Anything else to add?" (Optional)  
  - Webhook ID ensures unique webhook URL for form submissions.  
- **Key expressions/variables:** Outputs raw JSON data accessible downstream (e.g., `$('On form submission').item.json['When does your cycle start?']`).  
- **Input/Output:** No input; outputs form submission data.  
- **Edge cases:**  
  - Missing required fields will prevent submission.  
  - Webhook errors or unreachable form may block input.  
  - Date format inconsistencies should be handled downstream if needed.  

---

#### 1.2 Data Preparation

**Overview:**  
Sets static or default data values required for message generation, namely the team name and message tone.

**Nodes Involved:**  
- Set up

**Node Details:**  
- **Type:** Set node  
- **Technical Role:** Defines static JSON fields for use in downstream nodes.  
- **Configuration:**  
  - Sets "Team Name" to a placeholder string ("Team Name here").  
  - Sets "Message Tone" to "Scary" (likely customizable to "Warm" as recommended).  
- **Key expressions/variables:** Values used in the message generation prompt expressions (e.g., `{{ $json['Team Name'] }}`, `{{ $json['Message Tone'] }}`).  
- **Input/Output:** Receives form submission JSON; outputs augmented JSON with these fields.  
- **Edge cases:**  
  - Hardcoded values require manual update to reflect actual team and tone.  
  - Missing or misconfigured values may affect message quality or relevance.

---

#### 1.3 AI Processing

**Overview:**  
Generates a warm, formatted Slack message announcing the sprint cycle start using GPT-4 with Langchain integration.

**Nodes Involved:**  
- OpenAI Chat Model  
- Generate Slack Message

**Node Details:**  

- **OpenAI Chat Model**  
  - **Type:** Langchain OpenAI Chat Model node  
  - **Technical Role:** Provides GPT-4 language model capabilities for text generation.  
  - **Configuration:**  
    - Model: "gpt-4.1-mini" (a GPT-4 variant).  
    - Credentials: OpenAI API key configured.  
  - **Input/Output:** Receives prompt input from "Generate Slack Message" agent; outputs GPT-4 responses.  
  - **Edge cases:**  
    - API authentication errors.  
    - Rate limits and timeouts.  
    - Model unavailability or version mismatch.  

- **Generate Slack Message**  
  - **Type:** Langchain Agent node  
  - **Technical Role:** Defines the prompt and controls how the AI generates the Slack message.  
  - **Configuration:**  
    - Prompt text instructs the AI to:  
      - Greet the team warmly.  
      - Announce the cycle start with relative date phrasing (e.g., "Yesterday", "Today").  
      - Format cycle goals and additional notes with Slack-specific markdown (bold titles, emoji usage).  
      - Use message tone from input variables.  
      - Properly format URLs as Slack hyperlinks `<http://url | label>`.  
      - Keep message concise and engaging.  
    - Uses expressions to inject form data and static parameters:  
      - `{{ $json['Team Name'] }}`, `{{ $('On form submission').item.json['When does your cycle start?'] }}`, `{{ $today }}`, `{{ $json['Message Tone'] }}`, etc.  
  - **Input/Output:** Receives JSON from "Set up" node; sends prompt to OpenAI Chat Model; outputs generated text.  
  - **Edge cases:**  
    - Expression evaluation errors if referenced data missing or malformed.  
    - AI output may occasionally be off-tone or improperly formatted (requires validation).  
    - Network or API failures impacting prompt submission.

---

#### 1.4 Slack Posting

**Overview:**  
Sends the AI-generated message to a specified Slack channel using OAuth2 authentication.

**Nodes Involved:**  
- Send message to Slack

**Node Details:**  
- **Type:** Slack node (n8n native)  
- **Technical Role:** Posts a message to Slack channel.  
- **Configuration:**  
  - Text: Uses the generated message output from "Generate Slack Message" node (`{{$json.output}}`).  
  - Channel: Hardcoded to Slack channel ID "C08V1DRQPLZ" (channel named "nikhil-lab").  
  - Authentication: OAuth2 with stored Slack credentials.  
- **Input/Output:** Input is the formatted message; output is Slack API response.  
- **Edge cases:**  
  - OAuth token expiration or invalid credentials.  
  - Slack API rate limits or errors.  
  - Channel ID misconfiguration leading to failed posts.  

---

#### 1.5 Documentation & Guidance

**Overview:**  
Sticky notes provide contextual instructions and reminders for setup and customization of the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3

**Node Details:**  
- **Type:** Sticky Note nodes (visual only, no functional processing)  
- **Content Highlights:**  
  - Sticky Note: Explains the workflow’s purpose — sending Slack messages on sprint cycle start with form inputs.  
  - Sticky Note1: Reminder to update team name and message tone (warm tone recommended).  
  - Sticky Note2: Reminder to update OpenAI credentials or alternatively the AI model.  
  - Sticky Note3: Reminder to update Slack credentials and Slack channel name/id.  
- **Edge cases:** None (non-executable).  

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                       | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                   |
|-----------------------|--------------------------------|-------------------------------------|-----------------------|-----------------------|-----------------------------------------------------------------------------------------------|
| On form submission    | Form Trigger                   | Receives sprint cycle details via form | None                  | Set up                | "## Send a slack message when a new Sprint Cycle starts 🎯 The Message will have the following elements - Cycle Start Date - Cycle Goal - Additional Notes *Start with submitting the form URL from the first node* 🏎️" |
| Set up               | Set                           | Sets static data: Team Name, Message Tone | On form submission    | Generate Slack Message | "**1.☝️Update team name** **1.1 ☝️Set message tone (warm recommended)**"                       |
| Generate Slack Message | Langchain Agent               | Builds prompt to generate Slack message | Set up                | Send message to Slack  |                                                                                               |
| OpenAI Chat Model     | Langchain OpenAI Chat Model  | Provides GPT-4 model response       | Generate Slack Message | Generate Slack Message | "**2.☝️Update Open AI credentials** *or use a different Model*"                              |
| Send message to Slack | Slack                         | Posts message to Slack channel      | Generate Slack Message | None                  | "**2.1.☝️Update Slack Credentials** **2.2.☝️Update Slack Channel Name**"                      |
| Sticky Note           | Sticky Note                   | Documentation and guidance          | None                  | None                  | See above                                                                                     |
| Sticky Note1          | Sticky Note                   | Documentation and guidance          | None                  | None                  | See above                                                                                     |
| Sticky Note2          | Sticky Note                   | Documentation and guidance          | None                  | None                  | See above                                                                                     |
| Sticky Note3          | Sticky Note                   | Documentation and guidance          | None                  | None                  | See above                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**  
   - Add a Form Trigger node named "On form submission".  
   - Set "Form Title" to "Cycle Start".  
   - Add three form fields:  
     - "When does your cycle start?" (Date, required)  
     - "What's the cycle goal?" (Textarea, required)  
     - "Anything else to add?" (optional)  
   - Save and note the webhook URL generated for external form submission.

2. **Add a Set Node for Static Data:**  
   - Add a Set node named "Set up".  
   - Add two string fields:  
     - "Team Name" with value "Team Name here" (update as needed).  
     - "Message Tone" with value "Scary" (recommended to update to "Warm" for better tone).  
   - Connect "On form submission" main output to "Set up" main input.

3. **Add Langchain Agent Node for Slack Message Generation:**  
   - Add a Langchain Agent node named "Generate Slack Message".  
   - Configure the prompt text to instruct the AI to:  
     - Create a warm Slack announcement message for the sprint cycle start.  
     - Use form input fields and static values via expressions:  
       - Team name: `{{ $json['Team Name'] }}`  
       - Cycle start date: `{{ $('On form submission').item.json['When does your cycle start?'] }}`  
       - Cycle goal: `{{ $('On form submission').item.json['What\'s the cycle goal?'] }}`  
       - Additional notes: `{{ $('On form submission').item.json['Anything else to add?'] }}`  
       - Message tone: `{{ $json['Message Tone'] }}`  
       - Current date reference: `{{ $today }}` (make sure to define or pass current date).  
     - Include formatting instructions for Slack markdown and link handling.  
   - Connect "Set up" main output to "Generate Slack Message" main input.

4. **Add Langchain OpenAI Chat Model Node:**  
   - Add a Langchain OpenAI Chat Model node named "OpenAI Chat Model".  
   - Select model "gpt-4.1-mini" (or other GPT-4 variant as preferred).  
   - Assign OpenAI API credentials.  
   - Connect "Generate Slack Message" node’s language model input to this node.

5. **Connect Langchain Agent to OpenAI Chat Model:**  
   - In "Generate Slack Message" node, ensure it uses "OpenAI Chat Model" as the AI model node.  
   - This is typically done via the "ai_languageModel" connection input.

6. **Add Slack Node to Post Message:**  
   - Add a Slack node named "Send message to Slack".  
   - Set the operation to "Post Message".  
   - Set text to `{{$json.output}}` to consume generated message.  
   - Select channel by ID (e.g., "C08V1DRQPLZ") or update to your target Slack channel.  
   - Configure Slack OAuth2 credentials.  
   - Connect "Generate Slack Message" main output to "Send message to Slack" main input.

7. **Add Sticky Notes (Optional):**  
   - Add sticky notes to document workflow purpose, reminders to update team name, message tone, OpenAI credentials, Slack credentials, and channel name.  
   - Position them near relevant nodes for clarity.

8. **Testing and Activation:**  
   - Test form submission by sending data to the webhook URL.  
   - Verify message generation correctness and Slack posting.  
   - Adjust team name, message tone, or Slack channel as needed.  
   - Activate workflow when ready.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                    |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------|
| The workflow uses GPT-4 via Langchain integration to generate human-like Slack messages.            | AI Processing block                                |
| Slack message formatting follows Slack markdown and link conventions `<http://url | text>`          | See prompt instructions in "Generate Slack Message" node |
| Recommended to use a "warm" message tone for team announcements to increase engagement.              | Sticky Note1 advice                               |
| Credentials for OpenAI API and Slack OAuth2 must be configured and tested before activation.         | Sticky Note2 and Sticky Note3                      |
| Workflow designed for sprint cycle announcements but can be adapted for other recurring team updates | General use case                                  |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---