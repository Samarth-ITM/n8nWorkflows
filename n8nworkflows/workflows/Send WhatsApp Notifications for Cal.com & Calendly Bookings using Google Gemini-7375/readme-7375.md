Send WhatsApp Notifications for Cal.com & Calendly Bookings using Google Gemini

https://n8nworkflows.xyz/workflows/send-whatsapp-notifications-for-cal-com---calendly-bookings-using-google-gemini-7375


# Send WhatsApp Notifications for Cal.com & Calendly Bookings using Google Gemini

### 1. Workflow Overview

This workflow automates the process of sending WhatsApp notifications for booking events originating from two popular scheduling platforms: Cal.com and Calendly. It listens for booking-related events such as creation, rescheduling, cancellation, and meeting completion, then generates context-aware, well-formatted WhatsApp messages using Google Gemini AI. Finally, it sends these messages instantly to the meeting host’s WhatsApp number via the WhatsApp Business API.

The workflow is logically divided into the following blocks:

- **1.1 Event Reception:** Receives booking event triggers from Cal.com and Calendly via webhook triggers.
- **1.2 Data Aggregation:** Merges incoming event data into a unified format for downstream processing.
- **1.3 AI Message Generation:** Uses Google Gemini through LangChain integration to generate WhatsApp message bodies formatted with markdown and emojis according to event type.
- **1.4 Context Memory:** Maintains session memory to provide contextual awareness for the AI agent.
- **1.5 WhatsApp Notification Dispatch:** Sends the generated message to the recipient’s WhatsApp number using the WhatsApp Business API.
- **1.6 Documentation Note:** Provides a sticky note with workflow description and setup instructions for user reference.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Reception

- **Overview:** Listens for incoming booking events from Cal.com and Calendly via webhook triggers.
- **Nodes Involved:**  
  - Cal.com Events  
  - Calendly Events  

- **Node Details:**

  - **Cal.com Events**  
    - Type: `calTrigger` (Cal.com webhook trigger)  
    - Role: Captures booking-related events such as booking created, rescheduled, cancelled, and meeting ended.  
    - Configuration: Subscribes to events `BOOKING_RESCHEDULED`, `BOOKING_CREATED`, `BOOKING_CANCELLED`, `MEETING_ENDED`. Uses Cal.com API credentials.  
    - Input: External webhook calls from Cal.com.  
    - Output: Emits event data downstream.  
    - Edge Cases: Webhook registration failures, event filtering issues, API auth errors.

  - **Calendly Events**  
    - Type: `calendlyTrigger` (Calendly webhook trigger)  
    - Role: Captures booking events for invitee creations and cancellations.  
    - Configuration: Listens to `invitee.created` and `invitee.canceled` events. Uses OAuth2 credentials for authentication.  
    - Input: External webhook calls from Calendly.  
    - Output: Emits event data downstream.  
    - Edge Cases: OAuth token expiration, webhook subscription errors, event payload inconsistencies.

#### 2.2 Data Aggregation

- **Overview:** Combines incoming event data from both Cal.com and Calendly triggers into a unified dataset for message generation.  
- **Nodes Involved:**  
  - Merge Data  

- **Node Details:**

  - **Merge Data**  
    - Type: `aggregate` node  
    - Role: Aggregates all incoming items into a single array combining Cal.com and Calendly event data.  
    - Configuration: Uses the “aggregateAllItemData” operation to merge all items into one.  
    - Input: Receives event data from both Cal.com Events and Calendly Events nodes.  
    - Output: Emits merged data downstream.  
    - Edge Cases: Empty inputs, data format mismatches, potential duplication of events.

#### 2.3 AI Message Generation

- **Overview:** Generates a WhatsApp message body formatted with markdown and emojis, tailored to the booking event context using a LangChain AI agent powered by Google Gemini.  
- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Simple Memory  
  - Message Generator  

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: `lmChatGoogleGemini` (LangChain Chat Language Model node)  
    - Role: Provides natural language generation capability using Google Gemini (PaLM API).  
    - Configuration: Uses Google Palm API credentials. No special options configured.  
    - Input: Receives prompt and context from the Message Generator node.  
    - Output: AI-generated text passed to Message Generator.  
    - Edge Cases: API rate limits, auth failures, generation timeouts.

  - **Simple Memory**  
    - Type: `memoryBufferWindow` (LangChain memory buffer)  
    - Role: Stores conversation context using a session key to enable context-aware AI responses.  
    - Configuration: Uses a custom session key "heello" to maintain conversation state.  
    - Input: Stores and retrieves memory for the Message Generator node.  
    - Output: Provides memory context into the AI agent.  
    - Edge Cases: Memory overflow, incorrect session key usage.

  - **Message Generator**  
    - Type: `agent` (LangChain AI agent node)  
    - Role: The core AI agent that formats booking event data into a WhatsApp message using a detailed prompt and system message.  
    - Configuration:  
      - Input text is the JSON stringified merged event data.  
      - System message instructs the AI to output only the WhatsApp message body with specific markdown formatting, emoji usage, and message templates for each event type (created, rescheduled, cancelled, ended).  
      - Uses Google Gemini as the language model and Simple Memory for context.  
    - Input: Receives merged event data and memory context.  
    - Output: Outputs a single WhatsApp message body as plain text.  
    - Edge Cases: Improper event data formatting, AI output errors, prompt parsing failures.

#### 2.4 WhatsApp Notification Dispatch

- **Overview:** Sends the AI-generated WhatsApp message to the meeting host’s phone number via WhatsApp Business API.  
- **Nodes Involved:**  
  - WhatsApp Notification  

- **Node Details:**

  - **WhatsApp Notification**  
    - Type: `whatsApp` (WhatsApp Business API node)  
    - Role: Sends the formatted text message to the specified phone number using WhatsApp API.  
    - Configuration:  
      - Operation: send  
      - Phone Number ID: `661627073710535` (WhatsApp Business phone number identifier)  
      - Recipient Phone Number: placeholder “mobile number” that should be updated with actual recipient number.  
      - Text Body: Populated dynamically from the AI-generated message output.  
      - Credentials: Uses WhatsApp API credentials named "cashmate production creds".  
    - Input: Receives text message body from Message Generator.  
    - Output: Sends message via WhatsApp API, no further output.  
    - Edge Cases: Invalid phone number, API auth failure, message quota limits, network errors.

#### 2.5 Documentation Note

- **Overview:** Provides an embedded sticky note with detailed description, feature list, supported events, and setup instructions for user reference.  
- **Nodes Involved:**  
  - Sticky Note  

- **Node Details:**

  - **Sticky Note**  
    - Type: `stickyNote` (Documentation node)  
    - Role: Contains a formatted markdown note describing the workflow’s purpose, features, supported events, and setup tips.  
    - Configuration: Large note covering overview, supported events, features, setup steps, and a pro tip for customizing AI message templates.  
    - Input: None (informational)  
    - Output: None  
    - Edge Cases: None

---

### 3. Summary Table

| Node Name             | Node Type                                 | Functional Role                      | Input Node(s)                 | Output Node(s)          | Sticky Note                                                                                                                             |
|-----------------------|-------------------------------------------|------------------------------------|------------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Cal.com Events         | calTrigger                                | Booking event webhook from Cal.com | External webhook              | Merge Data               |                                                                                                                                         |
| Calendly Events        | calendlyTrigger                           | Booking event webhook from Calendly| External webhook              | Merge Data               |                                                                                                                                         |
| Merge Data            | aggregate                                | Aggregate event data from both APIs| Cal.com Events, Calendly Events | Message Generator         |                                                                                                                                         |
| Google Gemini Chat Model | lmChatGoogleGemini (LangChain)            | AI language model for message gen  | Message Generator (ai_languageModel) | Message Generator        |                                                                                                                                         |
| Simple Memory          | memoryBufferWindow (LangChain memory)    | Maintains AI context memory         | Message Generator (ai_memory) | Message Generator        |                                                                                                                                         |
| Message Generator      | agent (LangChain AI agent)                | Generates WhatsApp message          | Merge Data, Simple Memory, Google Gemini Chat Model | WhatsApp Notification     |                                                                                                                                         |
| WhatsApp Notification  | whatsApp                                 | Sends WhatsApp message to recipient| Message Generator             | None                    |                                                                                                                                         |
| Sticky Note            | stickyNote                               | Documentation and setup instructions| None                         | None                    | ## 📅 Calendar Booking Notifications to WhatsApp - Workflow monitors Cal.com & Calendly bookings and sends formatted WhatsApp messages. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cal.com Events Node**  
   - Type: `calTrigger`  
   - Configure webhook to listen to events: `BOOKING_RESCHEDULED`, `BOOKING_CREATED`, `BOOKING_CANCELLED`, `MEETING_ENDED`  
   - Set credentials for Cal.com API connection  
   - Connect output to `Merge Data` node

2. **Create Calendly Events Node**  
   - Type: `calendlyTrigger`  
   - Configure webhook to listen to events: `invitee.created`, `invitee.canceled`  
   - Use OAuth2 credentials for Calendly account  
   - Connect output to `Merge Data` node

3. **Create Merge Data Node**  
   - Type: `aggregate`  
   - Operation: Aggregate all item data (`aggregateAllItemData`)  
   - Connect inputs from Cal.com Events and Calendly Events nodes  
   - Connect output to Message Generator node

4. **Create Google Gemini Chat Model Node**  
   - Type: `lmChatGoogleGemini` from LangChain nodes  
   - Use Google Palm API credentials  
   - Leave default options unless customization required  
   - Connect as AI language model input to Message Generator node

5. **Create Simple Memory Node**  
   - Type: `memoryBufferWindow` from LangChain nodes  
   - Set `sessionKey` to a fixed string, e.g., `"heello"`  
   - Set `sessionIdType` to `customKey`  
   - Connect memory to Message Generator’s AI memory input

6. **Create Message Generator Node**  
   - Type: `agent` (LangChain AI agent)  
   - Parameter `text`: set expression to stringify JSON data: `={{ $json.data.toJsonString() }}`  
   - System message: paste the detailed message formatting instructions provided in the original workflow, including markdown formatting, emoji use, and event-specific templates  
   - Set prompt type to `define`  
   - Connect inputs:  
     - Main input from Merge Data node  
     - AI language model input from Google Gemini Chat Model node  
     - AI memory input from Simple Memory node  
   - Connect output to WhatsApp Notification node

7. **Create WhatsApp Notification Node**  
   - Type: `whatsApp` node  
   - Operation: `send`  
   - Set `phoneNumberId` to your WhatsApp Business phone number ID (e.g., `661627073710535`)  
   - Set `recipientPhoneNumber` to the meeting host’s mobile number (update dynamically if possible)  
   - Set `textBody` to expression: `={{ $json.output }}` (output from Message Generator)  
   - Configure WhatsApp API credentials  
   - Connect input from Message Generator node

8. **Add Sticky Note for Documentation**  
   - Type: `stickyNote`  
   - Paste descriptive content explaining workflow purpose, features, supported events, and setup instructions  
   - Place somewhere visible on the canvas for user reference

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow supports instant WhatsApp notifications for Cal.com & Calendly booking events with rich formatting and context awareness. | Workflow description (Sticky Note node content).                                                        |
| Customize AI message templates in the Message Generator node to align with your brand voice and messaging style.                   | See system message in Message Generator configuration.                                                  |
| Requires valid API credentials for Cal.com, Calendly (OAuth2), Google Palm API, and WhatsApp Business API to function properly.    | Credential setup instructions implicit in node configurations.                                          |
| WhatsApp markdown formatting and emoji usage enhance message readability and user experience.                                        | Specified in the AI agent system prompt.                                                                |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.