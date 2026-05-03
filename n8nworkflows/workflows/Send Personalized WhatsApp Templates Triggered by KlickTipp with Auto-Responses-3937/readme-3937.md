Send Personalized WhatsApp Templates Triggered by KlickTipp with Auto-Responses

https://n8nworkflows.xyz/workflows/send-personalized-whatsapp-templates-triggered-by-klicktipp-with-auto-responses-3937


# Send Personalized WhatsApp Templates Triggered by KlickTipp with Auto-Responses

### 1. Workflow Overview

This workflow automates sending personalized WhatsApp message templates triggered by KlickTipp events and processes incoming WhatsApp messages for auto-responses or subscription management. It integrates KlickTipp’s outbound triggers with WhatsApp Business Cloud API to deliver dynamic, real-time WhatsApp messages using pre-approved templates populated with subscriber data. Incoming WhatsApp messages are monitored to detect cancellation requests (e.g., "STOP") and automatically tag or subscribe contacts in KlickTipp accordingly.

The workflow is structured in four main logical blocks:

- **1.1 Data Reception via Webhook or WhatsApp Message**  
  Captures incoming events either from KlickTipp Outbound triggers or WhatsApp messages sent to the business account.

- **1.2 Data Filtering and Message Content Evaluation**  
  Processes the incoming WhatsApp messages to filter user-generated content and checks for cancellation keywords.

- **1.3 Sending WhatsApp Message Templates**  
  Sends personalized WhatsApp message templates triggered by KlickTipp or as auto-responses based on user messages.

- **1.4 Contact Subscription and Tagging in KlickTipp**  
  Manages contact subscription status in KlickTipp lists based on user interactions, such as opting out.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Reception via Webhook or WhatsApp Message

- **Overview:**  
  This block serves as the workflow’s entry points, receiving event data either when KlickTipp triggers an Outbound or when a new WhatsApp message is received by the WhatsApp Business Cloud API.

- **Nodes Involved:**  
  - KlickTipp Outbound triggered  
  - New message in WhatsApp  
  - Sticky Note2 (documentation)

- **Node Details:**

  - **KlickTipp Outbound triggered**  
    - *Type & Role:* Custom KlickTipp Trigger node; listens for specific subscriber events (Outbound triggers) from KlickTipp to start the workflow.  
    - *Configuration:* Uses KlickTipp API credentials; no additional parameters needed.  
    - *Input/Output:* No input nodes; outputs event data to WhatsApp message sending node.  
    - *Potential Failures:* Authentication errors with KlickTipp API; webhook delivery failures.  
    - *Sub-workflows:* None.

  - **New message in WhatsApp**  
    - *Type & Role:* WhatsApp Trigger node; listens for incoming WhatsApp messages to the business account.  
    - *Configuration:* Configured to listen for message updates only; uses WhatsApp Business Cloud OAuth2 credentials.  
    - *Input/Output:* No input nodes; outputs message data to filtering node.  
    - *Potential Failures:* OAuth token expiry; message webhook delivery issues; rate limiting by WhatsApp API.  
    - *Sub-workflows:* None.

  - **Sticky Note2**  
    - *Role:* Provides visual documentation labeling this block as “Data reception via Webhook call or message.”  
    - *No functional configuration.*

---

#### 2.2 Data Filtering and Message Content Evaluation

- **Overview:**  
  This block filters incoming WhatsApp messages to exclude automated messages from triggering loops and evaluates whether the message content indicates a cancellation request (e.g., messages starting with "STOP").

- **Nodes Involved:**  
  - Filter user messages  
  - Cancellation check  
  - Sticky Note3 (documentation)

- **Node Details:**

  - **Filter user messages**  
    - *Type & Role:* Filter node; ensures only messages originating from users (with message content present) are processed further.  
    - *Configuration:* Checks that the first message object exists in the incoming JSON data.  
    - *Input:* From “New message in WhatsApp” node.  
    - *Output:* Passes user messages to “Cancellation check,” blocks automated or empty messages.  
    - *Potential Failures:* Expression errors if message data is malformed or missing.  
    - *Version-specific:* Uses version 2.2 of the filter node.

  - **Cancellation check**  
    - *Type & Role:* Switch node; inspects the incoming WhatsApp message text normalized by removing whitespace and case sensitivity to determine if it starts with “stop.”  
    - *Configuration:* Two rule branches:  
      - Messages not starting with “stop” (continues normal processing)  
      - Messages starting with “stop” (triggers auto-responder and subscription actions)  
    - *Input:* From “Filter user messages.”  
    - *Output:*  
      - If message starts with “stop”: route to “Sending WhatsApp auto-responder template” and “Subscribe number to opt-out from WA messages” nodes.  
      - Otherwise: no further downstream nodes in this branch.  
    - *Potential Failures:* Expression parsing errors; non-standard message formats.  
    - *Version-specific:* Switch node version 3.2.

  - **Sticky Note3**  
    - *Role:* Visual label for this block: “Data filtering and message check.”

---

#### 2.3 Sending WhatsApp Message Templates

- **Overview:**  
  Sends personalized WhatsApp message templates either as a response to KlickTipp Outbound triggers or as an auto-response to user messages detected as cancellation requests.

- **Nodes Involved:**  
  - Sending WhatsApp offer template  
  - Sending WhatsApp auto-responder template  
  - Sticky Note4 (documentation)

- **Node Details:**

  - **Sending WhatsApp offer template**  
    - *Type & Role:* WhatsApp node; sends a pre-approved WhatsApp message template when KlickTipp triggers an Outbound event.  
    - *Configuration:*  
      - Template: “offer_for_manual|de”  
      - Dynamic placeholders populate from KlickTipp custom fields: first name, product/service, and link ending.  
      - Includes a URL button with a dynamic text parameter.  
      - Phone number formatted by replacing international dialing prefix “00” with “+”.  
      - Uses WhatsApp Business Cloud API credentials.  
    - *Input:* From “KlickTipp Outbound triggered.”  
    - *Output:* None downstream in this workflow.  
    - *Potential Failures:* Template approval issues; invalid phone numbers; credential expiry; API rate limits.

  - **Sending WhatsApp auto-responder template**  
    - *Type & Role:* WhatsApp node; sends a cancellation confirmation or support forwarding template message when the user sends “STOP”.  
    - *Configuration:*  
      - Template: “auto_forward_to_support|de”  
      - Placeholder uses sender’s profile name from message data.  
      - Recipient phone number extracted from the incoming WhatsApp message “from” field.  
      - Uses WhatsApp API credentials.  
    - *Input:* From “Cancellation check” switch node (for “stop” messages).  
    - *Output:* Passes control to subscription node.  
    - *Potential Failures:* Missing profile name; invalid recipient number; API errors.

  - **Sticky Note4**  
    - *Role:* Documentation label for “Sending WhatsApp message templates.”

---

#### 2.4 Contact Subscription and Tagging in KlickTipp

- **Overview:**  
  Subscribes or tags contacts in KlickTipp lists based on WhatsApp interactions, typically to opt-out users who send cancellation keywords.

- **Nodes Involved:**  
  - Subscribe number to opt-out from WA messages  
  - Sticky Note5 (documentation)

- **Node Details:**

  - **Subscribe number to opt-out from WA messages**  
    - *Type & Role:* Custom KlickTipp node; subscribes the WhatsApp sender to a specific KlickTipp list (e.g., opt-out list) using their phone number.  
    - *Configuration:*  
      - List ID: “358895” (target list for opt-outs)  
      - Subscriber resource, operation subscribe  
      - Phone number formatted with “+” prefix concatenated with WhatsApp sender ID  
      - Uses KlickTipp API credentials.  
    - *Input:* From “Sending WhatsApp auto-responder template.”  
    - *Output:* None downstream in this workflow.  
    - *Potential Failures:* Incorrect list ID; phone number formatting errors; API authentication failures.

  - **Sticky Note5**  
    - *Role:* Documentation label for “Contact subscription and tagging.”

---

### 3. Summary Table

| Node Name                          | Node Type                | Functional Role                                   | Input Node(s)                     | Output Node(s)                                   | Sticky Note                                                                                              |
|-----------------------------------|--------------------------|-------------------------------------------------|----------------------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| KlickTipp Outbound triggered       | Custom KlickTipp Trigger | Entry point for KlickTipp Outbound events       |                                  | Sending WhatsApp offer template                  | ## Data reception via Webhook call or message                                                          |
| New message in WhatsApp            | WhatsApp Trigger         | Entry point for incoming WhatsApp messages      |                                  | Filter user messages                             | ## Data reception via Webhook call or message                                                          |
| Filter user messages              | Filter                   | Filters valid user messages from WhatsApp       | New message in WhatsApp           | Cancellation check                              | ## Data filtering and message check                                                                     |
| Cancellation check                | Switch                   | Checks if message starts with "STOP" keyword   | Filter user messages             | Sending WhatsApp auto-responder template, Subscribe number to opt-out from WA messages | ## Data filtering and message check                                                                     |
| Sending WhatsApp offer template    | WhatsApp                 | Sends WhatsApp message template from KlickTipp  | KlickTipp Outbound triggered     |                                                 | ## Sending WhatsApp message templates                                                                   |
| Sending WhatsApp auto-responder template | WhatsApp                 | Sends auto-response WhatsApp template for "STOP" messages | Cancellation check               | Subscribe number to opt-out from WA messages    | ## Sending WhatsApp message templates                                                                   |
| Subscribe number to opt-out from WA messages | Custom KlickTipp         | Subscribes sender to KlickTipp opt-out list     | Sending WhatsApp auto-responder template |                                                 | ## Contact subscription and tagging                                                                     |
| Sticky Note2                     | Sticky Note              | Documentation for data reception block           |                                  |                                                 | ## Data reception via Webhook call or message                                                          |
| Sticky Note3                     | Sticky Note              | Documentation for data filtering block           |                                  |                                                 | ## Data filtering and message check                                                                     |
| Sticky Note4                     | Sticky Note              | Documentation for sending WhatsApp messages      |                                  |                                                 | ## Sending WhatsApp message templates                                                                   |
| Sticky Note5                     | Sticky Note              | Documentation for subscription and tagging       |                                  |                                                 | ## Contact subscription and tagging                                                                     |
| Sticky Note                      | Sticky Note              | Overall introduction and workflow description    |                                  |                                                 | See detailed introduction and benefits in workflow notes section                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**

   - **Create “KlickTipp Outbound triggered” node:**  
     - Type: Custom KlickTipp Trigger  
     - Configure with KlickTipp API credentials.  
     - No additional parameters needed.  
     - This node listens for outbound events (e.g., subscriber tags) from KlickTipp.

   - **Create “New message in WhatsApp” node:**  
     - Type: WhatsApp Trigger  
     - Configure with WhatsApp Business Cloud OAuth2 credentials.  
     - Set to listen for “messages” updates only.

2. **Create Filtering and Message Check Nodes**

   - **Create “Filter user messages” node:**  
     - Type: Filter  
     - Condition: Check that the JSON path `$json.messages[0]` exists (i.e., message content is present).  
     - Connect input from “New message in WhatsApp.”  
     - Output to “Cancellation check.”

   - **Create “Cancellation check” node:**  
     - Type: Switch  
     - Two rules:  
       - Rule 1 (False branch): Message text (normalized by removing whitespace and lowering case) NOT starting with “stop.” Expression:  
         ```
         {{$json.messages[0].text.body.toLowerCase().replace(/\s+/g, '')}}
         ```
         Check operation: Not starts with “stop”  
       - Rule 2 (True branch): Message text starts with “stop.”  
     - Connect input from “Filter user messages.”  
     - Outputs:  
       - True to “Sending WhatsApp auto-responder template” and “Subscribe number to opt-out from WA messages.”  
       - False: no further downstream.

3. **Create WhatsApp Message Sending Nodes**

   - **Create “Sending WhatsApp offer template” node:**  
     - Type: WhatsApp  
     - Use WhatsApp API credentials.  
     - Template: “offer_for_manual|de.”  
     - Set template body parameters to fill placeholders with KlickTipp custom fields:  
       - First name: `{{$json.CustomFieldFirstName}}`  
       - Product/Service: `{{$json.CustomField217373}}`  
       - Link ending: `{{$json.CustomField217511}}`  
     - Add a URL button component with dynamic text from field: `{{$json.CustomField218042}}`  
     - Format recipient phone number by replacing leading “00” with “+”:  
       ```
       {{$json.PhoneNumber.replace(/^00/, '+')}}
       ```  
     - Connect input from “KlickTipp Outbound triggered.”

   - **Create “Sending WhatsApp auto-responder template” node:**  
     - Type: WhatsApp  
     - Use WhatsApp API credentials.  
     - Template: “auto_forward_to_support|de.”  
     - Pass sender’s profile name as a placeholder:  
       ```
       {{$json.contacts[0].profile.name}}
       ```  
     - Recipient phone number:  
       ```
       {{$json.messages[0].from}}
       ```  
     - Connect input from “Cancellation check” node (true branch).

4. **Create KlickTipp Subscription Node**

   - **Create “Subscribe number to opt-out from WA messages” node:**  
     - Type: Custom KlickTipp node  
     - Configure with KlickTipp API credentials.  
     - Operation: Subscribe to subscriber list.  
     - List ID: “358895” (opt-out list).  
     - Subscriber phone number: prefix “+” plus WhatsApp sender ID:  
       ```
       '+' + $json.contacts[0].wa_id
       ```  
     - Connect input from “Sending WhatsApp auto-responder template.”

5. **Add Sticky Notes for Documentation**

   - Create sticky notes near relevant node groups with titles and descriptions matching:  
     - “Data reception via Webhook call or message” (near triggers)  
     - “Data filtering and message check” (near filtering nodes)  
     - “Sending WhatsApp message templates” (near WhatsApp message nodes)  
     - “Contact subscription and tagging” (near KlickTipp subscribe node)  
     - A large sticky note with workflow introduction, benefits, setup instructions, and testing notes to be placed clearly for reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow relies on a community node for KlickTipp triggers, limiting it to self-hosted n8n environments.                                                                                                                                           | Community node disclaimer                                                                                                 |
| Pre-approved WhatsApp message templates are required for sending outbound messages outside the 24-hour session window.                                                                                                                                 | WhatsApp Business API requirements                                                                                         |
| Phone number formatting replaces "00" with "+" to comply with WhatsApp and KlickTipp API phone number format expectations.                                                                                                                              | Phone number formatting best practice                                                                                      |
| Pro Tip: Customize the domain link endings in message templates per campaign or product line to enable targeted redirects.                                                                                                                              | Campaign expansion idea                                                                                                    |
| Cooldown Warning: Repeated tests via KlickTipp Outbound trigger may require a 6-hour wait unless routed through a campaign to bypass cooldown restrictions.                                                                                              | Testing & deployment note                                                                                                  |
| Campaign ideas include combining WhatsApp with KlickTipp email series, adding product interest tags, and performing A/B tests on messaging CTAs and timings.                                                                                             | Campaign expansion ideas                                                                                                   |
| The workflow automates multi-channel engagement by integrating WhatsApp with KlickTipp for dynamic, personalized user outreach and campaign control.                                                                                                    | Workflow benefits                                                                                                          |

---

This structured reference document enables advanced users and AI agents to fully understand, reproduce, and adapt the workflow for personalized WhatsApp marketing automation integrated with KlickTipp.