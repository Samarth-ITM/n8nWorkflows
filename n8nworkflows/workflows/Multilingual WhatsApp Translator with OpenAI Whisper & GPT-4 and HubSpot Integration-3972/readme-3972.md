Multilingual WhatsApp Translator with OpenAI Whisper & GPT-4 and HubSpot Integration

https://n8nworkflows.xyz/workflows/multilingual-whatsapp-translator-with-openai-whisper---gpt-4-and-hubspot-integration-3972


# Multilingual WhatsApp Translator with OpenAI Whisper & GPT-4 and HubSpot Integration

### 1. Workflow Overview

This workflow is a **Multilingual WhatsApp Translator and Voice Transcriber** that integrates **OpenAI Whisper** for audio transcription, **GPT-4/GPT-4o** for advanced AI translation and cultural adaptation, and **HubSpot CRM** for contact and message management. It is designed to automate multilingual customer communication via WhatsApp, enabling businesses to transcribe voice messages, translate texts and voice transcriptions into appropriate languages with localized tone and expressions, and maintain seamless CRM records—all in real-time.

The workflow logic is grouped into the following blocks:

- **1.1 Input Reception & Preprocessing:** Receives incoming WhatsApp messages via webhook, identifies message type (audio/text), and fetches media if needed.
  
- **1.2 Language & Country Detection:** Uses phone prefix mapping and AI classifiers to detect the sender's country and preferred language.

- **1.3 Audio Transcription (OpenAI Whisper):** Converts audio messages to text.

- **1.4 AI Translation & Cultural Adaptation (GPT-4/GPT-4o):** Generates translations that adapt tone, slang, emojis, etc.

- **1.5 Contact Management (HubSpot CRM):** Searches for or creates contacts in HubSpot based on phone number and message metadata.

- **1.6 WhatsApp Reply & Notification:** Sends translated messages back via WhatsApp API and notifies relevant parties.

- **1.7 History & Message Logging:** Updates message history and CRM with translation and communication logs.

- **1.8 Maintenance & Cleanup:** Scheduled deletion of old executions to keep the environment clean.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Preprocessing

**Overview:**  
This block receives WhatsApp webhook calls containing incoming messages. It differentiates between audio and text messages, retrieves media URLs for audio, and prepares files for transcription.

**Nodes Involved:**  
- Webhook  
- Group Filter  
- Data&Hora  
- Set User  
- Filter Client  
- Who Sent?  
- Your Number  
- Audio / Audio1 / Audio2 (IF nodes)  
- GET Media / GET Media1 / GET Media2 (HTTP Request)  
- Convert to File / Convert to File1 / Convert to File2  

**Node Details:**

- **Webhook**  
  - Type: HTTP webhook entry point  
  - Role: Receives incoming WhatsApp messages triggered by Evolution API or similar providers.  
  - Config: Default webhook URL, supports POST requests.  
  - Edge Cases: Missing payload, unauthorized calls, malformed JSON.

- **Group Filter**  
  - Type: Filter  
  - Role: Filters or routes webhook data to downstream nodes based on message properties.  
  - Edge Cases: Incorrect filtering rules can skip valid messages.

- **Data&Hora**  
  - Type: Code  
  - Role: Extracts or formats date/time from incoming data.  
  - Key Variables: Timestamps from webhook payload.  
  - Edge Cases: Timezone mismatches.

- **Set User**  
  - Type: Set  
  - Role: Stores sender's user context for subsequent processing.  
  - Edge Cases: Missing user info.

- **Filter Client**  
  - Type: Filter  
  - Role: Differentiates between client messages and system/bot messages.  
  - Edge Cases: Misclassification could cause message loss.

- **Who Sent?**  
  - Type: IF  
  - Role: Checks if the message is from the user or the bot itself (to avoid loops). Routes accordingly.  
  - Edge Cases: Incorrect identification can cause infinite loops or missed responses.

- **Your Number**  
  - Type: IF  
  - Role: Checks if the sender's number matches the bot's number or a known contact.  
  - Edge Cases: Number format mismatches.

- **Audio / Audio1 / Audio2 (IF nodes)**  
  - Type: IF  
  - Role: Detects if the incoming message contains audio.  
  - Edge Cases: Unsupported media types, missing media URLs.

- **GET Media / GET Media1 / GET Media2**  
  - Type: HTTP Request  
  - Role: Downloads media files via URL provided in the WhatsApp message payload.  
  - Edge Cases: Media URL expires, network errors, large file sizes.

- **Convert to File / Convert to File1 / Convert to File2**  
  - Type: Convert to File  
  - Role: Converts downloaded media data into n8n file format for processing.  
  - Edge Cases: Unsupported file types, conversion errors.

---

#### 1.2 Language & Country Detection

**Overview:**  
This block maps phone prefixes to countries, detects the client's country, and assigns the preferred language for translation and response.

**Nodes Involved:**  
- PrefixMap  
- Language Map  
- Filter the Country  
- Country and Audio  
- Primary Text Classifier  
- Secondary Text Classifier  

**Node Details:**

- **PrefixMap**  
  - Type: Code  
  - Role: Contains a mapping of international phone prefixes to country codes.  
  - Configuration: Custom code mapping known prefixes to countries.  
  - Edge Cases: Unrecognized prefixes, ambiguous prefixes.

- **Language Map**  
  - Type: Set  
  - Role: Maps country codes to preferred languages or dialects.  
  - Edge Cases: Missing mappings for countries.

- **Filter the Country**  
  - Type: Filter  
  - Role: Filters messages based on detected country for further routing.  
  - Edge Cases: Incorrect filtering causing misrouted messages.

- **Country and Audio**  
  - Type: IF  
  - Role: Decides processing path based on whether the message has audio and country classification.  
  - Edge Cases: Incorrect classification leading to wrong processing.

- **Primary Text Classifier**  
  - Type: AI Text Classifier (Langchain)  
  - Role: Classifies text or metadata to confirm or identify country/language.  
  - Edge Cases: Classification errors, ambiguous text.

- **Secondary Text Classifier**  
  - Type: AI Text Classifier (Langchain)  
  - Role: Secondary check or fallback classification for language detection.  
  - Edge Cases: Same as primary classifier.

---

#### 1.3 Audio Transcription (OpenAI Whisper)

**Overview:**  
Transcribes audio messages into text using OpenAI Whisper integrated through Langchain OpenAI nodes.

**Nodes Involved:**  
- OpenAI / OpenAI1 / OpenAI2  
- AI Agent / AI Agent1 / AI Agent2  
- Calculator / Calculator1 / Calculator2  
- Think / Think1 / Think2  
- OpenAI Chat Model / OpenAI Chat Model2 / OpenAI Chat Model3  

**Node Details:**

- **OpenAI (Langchain OpenAI)**  
  - Type: AI transcription using Whisper model  
  - Role: Converts audio files to text transcripts.  
  - Configuration: Uses OpenAI credentials, model configured for Whisper.  
  - Edge Cases: Audio quality issues, API rate limits, long audio files.

- **AI Agent (Langchain Agent)**  
  - Type: AI agent orchestration node  
  - Role: Coordinates AI tools for transcription and translation tasks.  
  - Edge Cases: Failures in chained AI calls or tool orchestration.

- **Calculator (Langchain Tool Calculator)**  
  - Type: Tool to perform calculations or logic within AI agent  
  - Role: Supports AI agent in managing workflow logic or prompt calculations.  
  - Edge Cases: Logic errors or improper input data.

- **Think (Langchain Tool Think)**  
  - Type: AI tool node to perform reasoning steps  
  - Role: Assists AI agents with intermediate reasoning or processing.  
  - Edge Cases: Timeout or reasoning failures.

- **OpenAI Chat Model (Langchain LM Chat OpenAI)**  
  - Type: Chat language model node  
  - Role: Used for GPT-4/GPT-4o translation and interaction.  
  - Edge Cases: API errors, prompt misconfiguration.

---

#### 1.4 AI Translation & Cultural Adaptation (GPT-4/GPT-4o)

**Overview:**  
Generates culturally adapted translations of the original or transcribed messages, adjusting tone, slang, and emojis for a natural conversational style.

**Nodes Involved:**  
- AI Agent / AI Agent1 / AI Agent2  
- OpenAI Chat Model / OpenAI Chat Model2 / OpenAI Chat Model3  
- Calculator / Calculator1 / Calculator2  
- Think / Think1 / Think2  
- Text Mapping / Text Mapping1 / Text Mapping2  

**Node Details:**

- **AI Agent**  
  - Role: Orchestrates translation and adaptation tasks leveraging GPT-4/GPT-4o models.  
  - Configuration: Prompt templates include instructions for tone, slang, and emoji adaptation.  
  - Edge Cases: Incorrect prompt leading to unnatural translations.

- **OpenAI Chat Model**  
  - Role: Executes the actual GPT model calls for translation.  
  - Edge Cases: API rate limits, prompt length issues.

- **Text Mapping**  
  - Type: Set nodes  
  - Role: Maps AI responses into structured data for CRM and messaging.  
  - Edge Cases: Missing or malformed AI output fields.

---

#### 1.5 Contact Management (HubSpot CRM)

**Overview:**  
Manages contact records in HubSpot by searching for existing contacts or creating new ones, ensuring complete CRM data for all conversations.

**Nodes Involved:**  
- Search Contact (HubSpot node)  
- Contact Located (IF)  
- Filter Client (Filter)  
- Create Contact (HTTP Request)  
- Delay  
- WhatsApp Notification (HTTP Request)  

**Node Details:**

- **Search Contact**  
  - Type: HubSpot API node  
  - Role: Searches HubSpot contacts by phone number or other identifiers.  
  - Edge Cases: API rate limits, missing contact data.

- **Contact Located**  
  - Type: IF  
  - Role: Branches logic based on whether contact was found.  
  - Edge Cases: False negatives if search criteria are incomplete.

- **Filter Client**  
  - Role: Filters client messages for CRM processing.

- **Create Contact**  
  - Type: HTTP Request  
  - Role: Creates a new contact in HubSpot if not found.  
  - Edge Cases: API errors, missing required fields.

- **Delay**  
  - Role: Allows for rate limiting or pacing CRM operations.

- **WhatsApp Notification**  
  - Type: HTTP Request  
  - Role: Sends notification messages after contact creation or updates.  
  - Edge Cases: HTTP errors, invalid payload.

---

#### 1.6 WhatsApp Reply & Notification

**Overview:**  
Sends translated or transcribed messages back to the user through WhatsApp API using Evolution API HTTP requests, and manages chat flags/notifications.

**Nodes Involved:**  
- WhatsApp / WhatsApp1 / WhatsApp2 / WhatsApp3 (HTTP Request)  
- React / React1 / React2 / React3 (HTTP Request)  
- ReactFlag  
- React Flags  

**Node Details:**

- **WhatsApp Nodes**  
  - Type: HTTP Request  
  - Role: Send messages or media back to WhatsApp users.  
  - Configuration: Use Evolution API endpoints and credentials.  
  - Edge Cases: API errors, invalid message formatting, rate limiting.

- **React Nodes**  
  - Type: HTTP Request  
  - Role: Send reactions or flags to WhatsApp chats (e.g., read receipts).  
  - Edge Cases: Failure to update chat flags.

---

#### 1.7 History & Message Logging

**Overview:**  
Updates and stores message and translation history for each contact, maintaining logs in HubSpot or other storage systems.

**Nodes Involved:**  
- My History (Code)  
- History (Code)  
- History1 (Code)  
- Update Messages (HTTP Request)  
- Update Messages1 (HTTP Request)  
- Update My Messages (HTTP Request)  

**Node Details:**

- **Code Nodes (My History, History, History1)**  
  - Role: Prepare and format historical message data for storage.  
  - Edge Cases: Data consistency issues.

- **Update Messages / Update Messages1 / Update My Messages**  
  - Type: HTTP Request  
  - Role: Send updated message logs to HubSpot or other APIs.  
  - Edge Cases: API failures, data sync issues.

---

#### 1.8 Maintenance & Cleanup

**Overview:**  
Scheduled routine to delete old workflow executions to prevent clutter and maintain system performance.

**Nodes Involved:**  
- Delete Every Day (Schedule Trigger)  
- Search Executions (n8n node)  
- Loops (Split In Batches)  
- Delete Execution (n8n node)  
- Replace Me (NoOp)  

**Node Details:**

- **Delete Every Day**  
  - Type: Schedule Trigger  
  - Role: Triggers cleanup daily.  

- **Search Executions**  
  - Type: n8n node (system)  
  - Role: Searches workflow executions based on age or status.  

- **Loops**  
  - Type: Split In Batches  
  - Role: Processes execution deletions batch-wise to avoid rate limits.  

- **Delete Execution**  
  - Type: n8n node  
  - Role: Deletes individual executions.  

- **Replace Me**  
  - Type: No Operation  
  - Role: Placeholder node for workflow continuity.  

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                         | Input Node(s)                     | Output Node(s)                   | Sticky Note                              |
|---------------------|----------------------------------|---------------------------------------|----------------------------------|---------------------------------|-----------------------------------------|
| Webhook             | n8n-nodes-base.webhook            | Receives incoming WhatsApp messages   |                                  | Group Filter                    |                                         |
| Group Filter        | n8n-nodes-base.filter             | Filters incoming data                  | Webhook                         | Data&Hora                      |                                         |
| Data&Hora           | n8n-nodes-base.code               | Extracts/formats date & time           | Group Filter                   | Set User                       |                                         |
| Set User            | n8n-nodes-base.set                | Sets user context                      | Data&Hora                      | Search Contact                 |                                         |
| Search Contact      | n8n-nodes-base.hubspot            | Searches HubSpot contacts              | Set User                       | Contact Located               |                                         |
| Contact Located     | n8n-nodes-base.if                 | Branches on contact search result     | Search Contact                 | Filter Client / Create Contact |                                         |
| Filter Client       | n8n-nodes-base.filter             | Filters client messages for CRM       | Contact Located                | Who Sent?                    |                                         |
| Create Contact      | n8n-nodes-base.httpRequest        | Creates new HubSpot contact            | Contact Located                | Delay                         |                                         |
| Delay               | n8n-nodes-base.wait               | Rate limiting / pacing                 | Create Contact                 | Search Contact                |                                         |
| WhatsApp Notification| n8n-nodes-base.httpRequest        | Sends notification messages            | Create Contact / Search Contact |                               |                                         |
| Who Sent?           | n8n-nodes-base.if                 | Checks message sender identity         | Filter Client                 | Your Number / Audio            |                                         |
| Your Number         | n8n-nodes-base.if                 | Checks if sender is bot or user        | Who Sent?                    | Audio2 / Audio1                |                                         |
| Audio               | n8n-nodes-base.if                 | Checks for audio message                | Who Sent?                    | GET Media2 / Text Mapping      |                                         |
| Audio1              | n8n-nodes-base.if                 | Checks for audio message (alt)          | Your Number                  | GET Media1 / Text Mapping1     |                                         |
| Audio2              | n8n-nodes-base.if                 | Checks for audio message (alt)          | Your Number                  | GET Media / Text Mapping2      |                                         |
| GET Media            | n8n-nodes-base.httpRequest        | Downloads audio media                   | Audio2                       | Convert to File2               |                                         |
| GET Media1           | n8n-nodes-base.httpRequest        | Downloads audio media                   | Audio1                       | Convert to File                |                                         |
| GET Media2           | n8n-nodes-base.httpRequest        | Downloads audio media                   | Audio                        | Convert to File1               |                                         |
| Convert to File      | n8n-nodes-base.convertToFile      | Converts media to file format           | GET Media1                   | OpenAI                        |                                         |
| Convert to File1     | n8n-nodes-base.convertToFile      | Converts media to file format           | GET Media2                   | OpenAI1                       |                                         |
| Convert to File2     | n8n-nodes-base.convertToFile      | Converts media to file format           | GET Media                    | OpenAI2                       |                                         |
| OpenAI               | @n8n/n8n-nodes-langchain.openAi  | Transcribes audio (Whisper)             | Convert to File              | Text Mapping1                 |                                         |
| OpenAI1              | @n8n/n8n-nodes-langchain.openAi  | Transcribes audio (Whisper)             | Convert to File1             | Text Mapping                  |                                         |
| OpenAI2              | @n8n/n8n-nodes-langchain.openAi  | Transcribes audio (Whisper)             | Convert to File2             | Text Mapping2                 |                                         |
| Text Mapping         | n8n-nodes-base.set                | Maps AI transcription output            | OpenAI1                      | Filter2                      |                                         |
| Text Mapping1        | n8n-nodes-base.set                | Maps AI transcription output            | OpenAI                       | Filter1                      |                                         |
| Text Mapping2        | n8n-nodes-base.set                | Maps AI transcription output            | OpenAI2                      | Filter3                      |                                         |
| Filter1              | n8n-nodes-base.filter             | Filters based on country prefix          | Text Mapping1                | PrefixMap                    |                                         |
| PrefixMap            | n8n-nodes-base.code              | Maps phone prefix to country             | Filter1                      | Language Map                 |                                         |
| Language Map         | n8n-nodes-base.set                | Sets language based on country           | PrefixMap                    | Filter the Country           |                                         |
| Filter the Country   | n8n-nodes-base.filter             | Filters based on detected country         | Language Map                 | AI Agent1                   |                                         |
| AI Agent1            | @n8n/n8n-nodes-langchain.agent   | AI translation & adaptation               | Filter the Country           | Filter4                     |                                         |
| Filter4              | n8n-nodes-base.filter             | Filters valid translated messages         | AI Agent1                    | Code1                       |                                         |
| Code1                | n8n-nodes-base.code               | Custom code processing                     | Filter4                      | React2                      |                                         |
| React2               | n8n-nodes-base.httpRequest        | Sends WhatsApp reply                       | Code2                       | WhatsApp                    |                                         |
| WhatsApp             | n8n-nodes-base.httpRequest        | Sends WhatsApp message                      | React2                       |                             |                                         |
| AI Agent             | @n8n/n8n-nodes-langchain.agent   | AI translation & cultural adaptation        | Primary Text Classifier      | Filter                      |                                         |
| Filter               | n8n-nodes-base.filter             | Filters AI translation output                | AI Agent                     | Code                        |                                         |
| Code                 | n8n-nodes-base.code               | Custom code for post-AI processing            | Filter                       | React                       |                                         |
| React                | n8n-nodes-base.httpRequest        | Sends WhatsApp reply                       | Code                        | WhatsApp1                   |                                         |
| WhatsApp1            | n8n-nodes-base.httpRequest        | Sends WhatsApp message                      | React                       | React Flags                 |                                         |
| React Flags          | n8n-nodes-base.httpRequest        | Sends chat flags or reactions                | WhatsApp1                   | History1                    |                                         |
| History1             | n8n-nodes-base.code               | Prepares message history update               | React Flags                 | Update Messages1            |                                         |
| Update Messages1     | n8n-nodes-base.httpRequest        | Updates message history in CRM                 | History1                    |                             |                                         |
| Primary Text Classifier | @n8n/n8n-nodes-langchain.textClassifier | Classifies text for country/language       | Country and Audio           | AI Agent                   |                                         |
| Secondary Text Classifier | @n8n/n8n-nodes-langchain.textClassifier | Secondary text classification              | Primary Text Classifier      | AI Agent                   |                                         |
| Country and Audio    | n8n-nodes-base.if                 | Checks for country and audio presence          | Format Phone                | Primary Text Classifier / Brasil |                                         |
| Format Phone         | n8n-nodes-base.code              | Formats phone numbers                          | Update Messages / Update Messages | Country and Audio        |                                         |
| Brasil               | n8n-nodes-base.filter             | Filters messages from Brazil                   | Country and Audio           | Audio Only                 |                                         |
| Audio Only           | n8n-nodes-base.if                 | Checks if message is audio only                 | Brasil                      | Code3                      |                                         |
| Code3                | n8n-nodes-base.code               | Custom processing for Brazilian audio messages | Audio Only                  | React3                     |                                         |
| React3               | n8n-nodes-base.httpRequest        | Sends WhatsApp reply                           | Code3                       | WhatsApp3                  |                                         |
| WhatsApp3            | n8n-nodes-base.httpRequest        | Sends WhatsApp message                          | React3                      |                             |                                         |
| Search Executions    | n8n-nodes-base.n8n               | Searches workflow executions                   | Delete Every Day             | Loops                      |                                         |
| Delete Every Day     | n8n-nodes-base.scheduleTrigger   | Daily trigger for cleanup                          |                             | Search Executions           |                                         |
| Loops                | n8n-nodes-base.splitInBatches    | Batch processes executions for deletion          | Search Executions            | Delete Execution            |                                         |
| Delete Execution     | n8n-nodes-base.n8n               | Deletes individual executions                      | Loops                       | Replace Me                 |                                         |
| Replace Me           | n8n-nodes-base.noOp              | No operation placeholder                           | Delete Execution             | Loops                      |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Purpose: Receive WhatsApp messages via POST.  
   - Configure webhook URL and enable.

2. **Add Group Filter Node:**  
   - Type: Filter  
   - Purpose: Route webhook data based on message attributes.

3. **Add Data&Hora Node (Code):**  
   - Extract timestamp from incoming message payload.  
   - Format considering timezone (America/Sao_Paulo).  

4. **Add Set User Node:**  
   - Set user-related variables from payload for further processing.

5. **Add Search Contact Node (HubSpot):**  
   - Use HubSpot API credentials.  
   - Search contact by phone number extracted from message.

6. **Add Contact Located IF Node:**  
   - Branch if contact found or not.

7. **Add Filter Client Node:**  
   - Filters messages to identify client-originated messages.

8. **Add Create Contact Node (HTTP Request):**  
   - Use HubSpot API to create new contact if none found.  
   - Set required fields: name, phone.

9. **Add Delay Node:**  
   - For pacing contact creation to avoid API rate limits.

10. **Add WhatsApp Notification Node (HTTP Request):**  
    - Sends notification messages post contact creation or update.

11. **Add Who Sent? IF Node:**  
    - Check if message is from user or bot itself.

12. **Add Your Number IF Node:**  
    - Confirms if sender number is bot’s or client’s.

13. **Add Audio IF Nodes (Audio, Audio1, Audio2):**  
    - Detect if message contains audio media.

14. **Add GET Media HTTP Request Nodes (GET Media, GET Media1, GET Media2):**  
    - Download audio files using media URLs from WhatsApp payload.

15. **Add Convert to File Nodes (Convert to File, Convert to File1, Convert to File2):**  
    - Convert downloaded media to n8n file format.

16. **Add OpenAI Nodes (OpenAI, OpenAI1, OpenAI2):**  
    - Use OpenAI Whisper model for transcription.  
    - Set credentials for OpenAI API.

17. **Add Text Mapping Nodes (Text Mapping, Text Mapping1, Text Mapping2):**  
    - Map transcription results into structured fields.

18. **Add Filter1 Node:**  
    - Filters data for prefix mapping.

19. **Create PrefixMap Code Node:**  
    - Implement mapping of phone prefixes to countries.

20. **Create Language Map Set Node:**  
    - Map countries to preferred languages.

21. **Add Filter the Country Node:**  
    - Routes data by country.

22. **Add AI Agents (AI Agent, AI Agent1, AI Agent2):**  
    - Use Langchain agent nodes to orchestrate GPT-4/GPT-4o translation.  
    - Configure prompts to include tone, slang, emoji adaptation instructions.  
    - Connect appropriate AI language model nodes (OpenAI Chat Model, etc.).

23. **Add Calculator and Think Nodes:**  
    - Support AI agents with logic and intermediate reasoning.

24. **Add Filter Nodes Post AI Agent:**  
    - Filter valid AI outputs.

25. **Add Code Nodes for Post-Processing:**  
    - Customize handling of AI outputs.

26. **Add React and WhatsApp HTTP Request Nodes:**  
    - Send translated messages and reactions back to WhatsApp users.  
    - Configure Evolution API credentials.

27. **Add History Code Nodes (My History, History, History1):**  
    - Prepare message and translation history updates.

28. **Add Update Messages HTTP Request Nodes:**  
    - Send updates to HubSpot or other CRM endpoints.

29. **Add Schedule Trigger Node (Delete Every Day):**  
    - Set daily trigger for cleanup.

30. **Add Search Executions Node:**  
    - Search old workflow executions.

31. **Add Loops Node:**  
    - Split executions into batches for deletion.

32. **Add Delete Execution Node:**  
    - Deletes executions in batches.

33. **Add Replace Me NoOp Node:**  
    - Placeholder to complete loop cycle.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow by Amanda, AI automation expert, with 2+ years n8n & Make.com experience.                                                    | Author info from description                                    |
| Official support materials and templates available at **iloveflows.gumroad.com**                                                     | [iloveflows.gumroad.com](https://iloveflows.gumroad.com)        |
| Use affiliate link to start with n8n Cloud: **https://n8n.partnerlinks.io/amanda**                                                   | Affiliate link from description                                 |
| VPS server discount offered at Hostinger: **https://hostinger.com/vps** with referral code 'iloveflows'                             | [Hostinger VPS Discount](https://www.hostinger.com/cart?product=vps%3Avps_kvm_4&period=12&referral_type=cart_link&REFERRALCODE=iloveflows&referral_id=0196b5ab-28ce-710f-b543-2fd6a0d7699f) |
| Supports 80+ countries’ phone prefixes for language detection.                                                                        | PrefixMap node and related filters                              |
| Uses Evolution API for WhatsApp integration, compatible with chatbot frameworks.                                                      | HTTP Request nodes for WhatsApp                                 |
| Transcription powered by OpenAI Whisper, translation and adaptation by GPT-4/GPT-4o via Langchain nodes.                              | OpenAI nodes configuration                                      |
| Automatic HubSpot CRM integration for contact and message management.                                                                  | HubSpot nodes and HTTP Request nodes                            |

---

This structured documentation enables deep understanding, easy modification, and accurate reproduction of the entire workflow without requiring the original JSON.