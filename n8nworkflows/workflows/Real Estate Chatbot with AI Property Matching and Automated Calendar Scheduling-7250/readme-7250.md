Real Estate Chatbot with AI Property Matching and Automated Calendar Scheduling

https://n8nworkflows.xyz/workflows/real-estate-chatbot-with-ai-property-matching-and-automated-calendar-scheduling-7250


# Real Estate Chatbot with AI Property Matching and Automated Calendar Scheduling

### 1. Workflow Overview

This n8n workflow, titled **"Real Estate Chatbot with AI Property Matching and Automated Calendar Scheduling"**, is designed to provide a comprehensive conversational assistant for real estate customer interactions. It handles user inquiries about property rental and purchase, intelligently extracts personal and property-related information, performs property searches, displays results with images, and automates appointment scheduling via calendar and email integration. The workflow also supports escalation to human agents when requested and manages appointment confirmations and modifications through incoming email processing.

The workflow logic is organized into the following main blocks:

- **1.1 Input Reception & Initial Parsing**  
  Receives user messages via webhook, extracts URLs, and routes for further processing.

- **1.2 Content Filtering and AI Conversation Management**  
  Uses an AI language model to filter non-property inquiries, extract personal and property requirements, and manage conversation flow including handoff requests.

- **1.3 Data Parsing and Readiness Validation**  
  Parses AI-generated JSON output to structured data, validates completeness of required personal information.

- **1.4 Property Search and Data Enrichment**  
  Queries a PostgreSQL database to find matching properties, enriches results with media links, and formats output with AI for user presentation.

- **1.5 Response Delivery**  
  Sends formatted chatbot responses back to the user.

- **1.6 Appointment Scheduling & Calendar Management**  
  Automates appointment booking, availability checking, calendar event creation/updating/deletion, and confirmation email sending, coordinated through an AI agent processing Gmail triggers.

- **1.7 Human Agent Handoff**  
  Detects requests to talk to a human agent, collects missing contact details and customer messages, sends a detailed handoff email.

- **1.8 Conversation Memory Management**  
  Uses PostgreSQL for conversational context retention across sessions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initial Parsing

- **Overview:**  
  Accepts incoming chat messages via webhook, extracts any embedded property URLs, and directs flow depending on presence of links.

- **Nodes Involved:**  
  - Webhook  
  - Code (Extract URLs)  
  - If (Check if link present)  
  - Postgres (Fetch property from DB by URL)  
  - If1 (Check property availability)  
  - Code1 (Format single property details)  
  - Send Formatted Response

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook Trigger  
    - Config: POST method on custom path, responds via response node  
    - Inputs: External user messages  
    - Outputs: Passes user message JSON downstream  
    - Potential Failures: Missing or malformed requests, timeout  

  - **Code (Extract URLs)**  
    - Type: JavaScript code node  
    - Function: Extracts first URL and all URLs from incoming message text using regex  
    - Inputs: Message from Webhook  
    - Outputs: JSON with originalMessage, first link, all links  
    - Failure Cases: No links found returns null link  

  - **If (Check link presence)**  
    - Type: Conditional node  
    - Condition: Link field not empty  
    - Directs flow to DB query or AI conversation depending on link presence  
    - Failure Cases: Expression errors if link field missing  

  - **Postgres (Property by URL)**  
    - Type: PostgreSQL query node  
    - Query: Select property where propertyguru_url or 99.co_url matches link  
    - Inputs: Link from Code node  
    - Outputs: Property data if found  
    - Failures: DB connection errors, no matching property  

  - **If1 (Check Availability)**  
    - Type: Conditional node  
    - Condition: Property availability equals "Immediate"  
    - Routes to format single property or fallback to AI conversation  
    - Edge Cases: Null availability fields  

  - **Code1 (Format Single Property)**  
    - Type: JavaScript code node  
    - Function: Summarizes detailed property information into descriptive text  
    - Inputs: Property DB records  
    - Outputs: Human-friendly property summary  
    - Failures: Missing fields, empty results  

  - **Send Formatted Response**  
    - Type: Respond to Webhook  
    - Sends summarized property data as chatbot response  

---

#### 2.2 Content Filtering and AI Conversation Management

- **Overview:**  
  Uses a LangChain agent with OpenAI GPT-4 to filter messages for real estate relevance, extract personal and property details, manage sequential question flow, and detect human agent handoff requests.

- **Nodes Involved:**  
  - LLM conversation (LangChain Agent Node)  
  - PostgreSQL Memory (Conversation context memory)  
  - OpenAI Chat Model (GPT-4)  

- **Node Details:**

  - **LLM conversation**  
    - Type: LangChain Agent node integrating OpenAI GPT-4  
    - Config: Complex system prompt with content filter, stepwise extraction instructions, and workflow logic for info collection and handoff  
    - Input: raw user message text  
    - Output: JSON-formatted extracted info or handoff instructions  
    - Uses PostgreSQL Memory for context retention  
    - Failure Scenarios: API limits, prompt parsing errors, unexpected user inputs  

  - **PostgreSQL Memory**  
    - Type: LangChain memory plugin backed by PostgreSQL  
    - Config: Uses "user_details" table for session memory, custom session key  
    - Purpose: Maintains chat state, avoids redundant questions  
    - Failures: DB connection, session key mismatches  

  - **OpenAI Chat Model**  
    - Type: Language model node (GPT-4) used by agent  
    - Config: Temperature 0 for deterministic output  
    - Role: Interprets user inputs, applies content filter and extraction logic  

---

#### 2.3 Data Parsing and Readiness Validation

- **Overview:**  
  Parses the JSON output string from the AI agent, normalizes personal and property info, validates completeness of user contact information to determine readiness to proceed.

- **Nodes Involved:**  
  - Parse Data (Code node)  
  - Ready Check (If node)  

- **Node Details:**

  - **Parse Data**  
    - Type: JavaScript code node  
    - Function: Extracts personal details (name, phone, email) and property requirements (type, location, budget, timeline, citizenship) from AI JSON output embedded in text  
    - Also cleans location string and normalizes budget to integer  
    - Outputs a readiness boolean indicating if all personal info is present  
    - Edge Cases: Malformed JSON, missing fields, parsing errors logged  

  - **Ready Check**  
    - Type: Conditional node  
    - Condition: readiness boolean equals true  
    - Routes to property search or loops back for collecting missing info  

---

#### 2.4 Property Search and Data Enrichment

- **Overview:**  
  Queries property database using extracted criteria, fetches associated media, merges datasets, and formats output with AI to produce user-friendly property listings.

- **Nodes Involved:**  
  - Query Properties (PostgreSQL)  
  - Get Property Media (PostgreSQL)  
  - construct_image_url (Code node)  
  - Merge (Merge node)  
  - Format Results with AI (LangChain Agent)  
  - Format Response (Set node)  
  - Send Formatted Response (Respond node)  
  - OpenAI Chat Model3 (GPT-4o-mini)  

- **Node Details:**

  - **Query Properties**  
    - Type: PostgreSQL query node  
    - Query: Filters properties by type, nearest MRT, and budget upper bound, orders by price ascending, limits to 5 results  
    - Inputs: property.type, property.locations, property.budget  
    - Failures: DB errors, no results  

  - **Get Property Media**  
    - Type: PostgreSQL query node  
    - Query: Fetches storage path of media images for given property ID  
    - Inputs: property IDs from Query Properties  
    - Outputs: Storage path(s) for images  

  - **construct_image_url**  
    - Type: JavaScript code node  
    - Builds full public URLs from storage paths  
    - Deduplicates images, returns first image URL per property  

  - **Merge**  
    - Type: Merge node (combine mode with join on property ID)  
    - Combines property data with media URLs  

  - **Format Results with AI**  
    - Type: LangChain Agent node  
    - System prompt instructs to format property results in engaging, professional style including images and key details  
    - Uses GPT-4o-mini  
    - Input: merged property data and user search criteria  

  - **Format Response**  
    - Type: Set node  
    - Assigns AI output text to response field  

  - **Send Formatted Response**  
    - Type: Respond to Webhook node  
    - Sends formatted property listings to user  

---

#### 2.5 Appointment Scheduling & Calendar Management

- **Overview:**  
  Automates scheduling of property viewing appointments based on user input, checking calendar availability, creating/updating Google Calendar events, and sending Gmail confirmations. Uses AI agent to process incoming Gmail appointment emails.

- **Nodes Involved:**  
  - Gmail Trigger (polling new appointment emails)  
  - AI Agent (LangChain Agent for appointment management)  
  - Google Calendar (delete event)  
  - Google Calendar1 (get all events)  
  - get_Calendar (get availability)  
  - create_Calendar1 (create event)  
  - update_Calendar (update event)  
  - Gmail1 (send confirmation email)  
  - OpenAI Chat Model1 (GPT-4o-mini)  

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Gmail polling trigger node  
    - Polls for new emails hourly  
    - Outputs email data for AI processing  

  - **AI Agent (Appointment Management)**  
    - LangChain agent node with detailed system prompt to process appointment emails: confirms, reschedules, checks calendar availability, and manages communication  
    - Integrates Google Calendar and Gmail nodes for actions  
    - Handles errors such as conflicts, ambiguous requests, missing info  

  - **Google Calendar nodes**  
    - Create, update, delete, and get events on specified calendar  
    - Inputs from AI agent include event ID, start/end times, summary, location, etc.  
    - Failures: API rate limits, invalid date/time formats  

  - **Gmail1**  
    - Sends confirmation or clarification emails based on AI agent instructions  

---

#### 2.6 Human Agent Handoff

- **Overview:**  
  Upon detection of user requests to speak with a human agent, collects any missing personal details sequentially, gathers a specific message from the user, then sends a detailed email handoff to a designated agent email.

- **Nodes Involved:**  
  - Human agent (Gmail node)  
  - LLM conversation (agent node for message composition)  

- **Node Details:**

  - **Human agent (Gmail node)**  
    - Sends email to fixed address (markkevin109@gmail.com)  
    - Email content includes customer details, message, and property requirements as filled by AI agent variables  
    - Subject customizable via AI variables  
    - Failures: SMTP errors, missing email fields  

---

#### 2.7 Conversation Memory Management

- **Overview:**  
  Maintains session context and stores user personal details to avoid repeated questions and enable smooth multi-turn conversations.

- **Nodes Involved:**  
  - PostgreSQL Memory  
  - Save Personal Info (Postgres node)  

- **Node Details:**

  - **Save Personal Info**  
    - Inserts collected personal info into user_info table  
    - Inputs: name, phone, email from parsed data  
    - Ensures persistent user record for future sessions  

---

### 3. Summary Table

| Node Name           | Node Type                                     | Functional Role                                | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                                                         |
|---------------------|-----------------------------------------------|-----------------------------------------------|---------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Webhook             | HTTP Webhook Trigger                          | Entry point for user chat messages            | -                         | Code                        | A comprehensive real estate chatbot automation system handling inquiries, property search, and appointment scheduling.              |
| Code                | Code (JS)                                     | Extracts URLs from user messages               | Webhook                   | If                          |                                                                                                                                     |
| If                  | Conditional node                              | Checks presence of URL in message              | Code                      | Postgres / LLM conversation |                                                                                                                                     |
| Postgres            | PostgreSQL Query                              | Fetches property details by URL                 | If                        | If1                         |                                                                                                                                     |
| If1                 | Conditional node                              | Checks property availability                    | Postgres                  | Code1 / LLM conversation    |                                                                                                                                     |
| Code1                | Code (JS)                                    | Summarizes property details for response       | If1                       | Send Formatted Response      |                                                                                                                                     |
| Send Formatted Response | Respond to Webhook                         | Sends property summary back to user             | Code1 / Format Response    | -                           |                                                                                                                                     |
| LLM conversation    | LangChain Agent (GPT-4)                       | Content filtering, info extraction, conversation flow | If / Human agent / Gmail  | Parse Data / Send Formatted Response |                                                                                                                                     |
| PostgreSQL Memory   | LangChain Memory (Postgres)                   | Conversation memory for user details           | -                         | LLM conversation            |                                                                                                                                     |
| Parse Data          | Code (JS)                                     | Parses AI output JSON, normalizes data         | LLM conversation          | Ready Check                 |                                                                                                                                     |
| Ready Check         | Conditional node                              | Checks if personal info is complete             | Parse Data                | Query Properties / Save Personal Info |                                                                                                                                     |
| Query Properties    | PostgreSQL Query                              | Searches properties by criteria                 | Ready Check               | Get Property Media / Merge   |                                                                                                                                     |
| Get Property Media  | PostgreSQL Query                              | Fetches media storage path for properties       | Query Properties          | construct_image_url          |                                                                                                                                     |
| construct_image_url | Code (JS)                                     | Builds full image URLs                           | Get Property Media        | Merge                       |                                                                                                                                     |
| Merge               | Merge node                                    | Combines property data with media URLs          | Query Properties, construct_image_url | Format Results with AI       |                                                                                                                                     |
| Format Results with AI | LangChain Agent (GPT-4o-mini)               | Formats property listings as natural language  | Merge                     | Format Response             |                                                                                                                                     |
| Format Response     | Set node                                      | Sets formatted text field                        | Format Results with AI    | Send Formatted Response      |                                                                                                                                     |
| Gmail               | Gmail Tool                                    | Sends emails for appointment confirmations      | AI Agent                  | LLM conversation            |                                                                                                                                     |
| Gmail1              | Gmail Tool                                    | Sends appointment confirmation emails           | AI Agent                  | AI Agent                    |                                                                                                                                     |
| Gmail Trigger       | Gmail Trigger                                 | Polls incoming emails for appointment management | -                        | AI Agent                    |                                                                                                                                     |
| AI Agent            | LangChain Agent                               | Manages appointment confirmations, rescheduling | Gmail Trigger             | Gmail1, Google Calendar nodes |                                                                                                                                     |
| Google Calendar     | Google Calendar Tool                          | Deletes calendar events                          | AI Agent                  | AI Agent                    |                                                                                                                                     |
| Google Calendar1    | Google Calendar Tool                          | Gets all calendar events                         | AI Agent                  | AI Agent                    |                                                                                                                                     |
| get_Calendar        | Google Calendar Tool                          | Checks calendar availability                     | AI Agent                  | LLM conversation            |                                                                                                                                     |
| create_Calendar1    | Google Calendar Tool                          | Creates new calendar events                       | AI Agent                  | AI Agent                    |                                                                                                                                     |
| update_Calendar     | Google Calendar Tool                          | Updates existing calendar events                  | AI Agent                  | AI Agent                    |                                                                                                                                     |
| Human agent         | Gmail Tool                                    | Sends human handoff email                         | LLM conversation          | LLM conversation            |                                                                                                                                     |
| Save Personal Info  | PostgreSQL Query                              | Saves collected personal user details            | Ready Check               | -                           |                                                                                                                                     |
| OpenAI Chat Model   | LangChain LM Chat (GPT-4)                     | Used by LLM conversation for interpretation     | -                         | LLM conversation            |                                                                                                                                     |
| OpenAI Chat Model1  | LangChain LM Chat (GPT-4o-mini)               | Used by AI Agent for appointment email processing | -                        | AI Agent                    |                                                                                                                                     |
| OpenAI Chat Model3  | LangChain LM Chat (GPT-4o-mini)               | Used by Format Results with AI                    | -                         | Format Results with AI      |                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - Method: POST  
   - Path: Custom path (e.g., "d22c99cc-a257-4c69-9419-34bb82a14026")  
   - Response Mode: Response Node  
   - Purpose: Receive incoming user chat messages.

2. **Add Code Node ("Code")**  
   - Extract the first URL and all URLs from the incoming message text using regex.  
   - Inputs: Webhook output  
   - Outputs: JSON with originalMessage, link, allLinks.

3. **Add If Node ("If")**  
   - Condition: Check if the extracted link is not empty.  
   - True branch: Property URL processing; False branch: AI conversation.

4. **Add Postgres Node ("Postgres")**  
   - Query DB for property matching the URL (propertyguru_url or 99.co_url).  
   - Input: link from Code node.

5. **Add If Node ("If1")**  
   - Condition: Check if property availability is "Immediate".  
   - True branch: Format single property; False branch: proceed to AI conversation.

6. **Add Code Node ("Code1")**  
   - Summarize property details into a descriptive text for user.

7. **Add Respond to Webhook Node ("Send Formatted Response")**  
   - Return the property summary to the user.

8. **Add LangChain Agent Node ("LLM conversation")**  
   - Configure with custom system prompt including content filtering, extraction, handoff, and stepwise info collection.  
   - Use GPT-4 model with temperature 0.  
   - Input: User message text.  
   - Output: JSON extraction or handoff instructions.

9. **Add PostgreSQL Memory Node ("PostgreSQL Memory")**  
   - Configure with user_details table, custom session key.  
   - Connect as memory for LLM conversation node.

10. **Add Code Node ("Parse Data")**  
    - Parse AI output JSON to extract personal and property info.  
    - Normalize and clean fields (location, budget).  
    - Determine readiness for next steps.

11. **Add If Node ("Ready Check")**  
    - Condition: Is personal info complete (name, phone, email).  
    - True branch: Proceed with property search; False branch: loop back for info collection.

12. **Add Postgres Node ("Query Properties")**  
    - Query new_properties table filtering by type, nearest MRT, budget.  
    - Limit 5 results sorted by price ascending.

13. **Add Postgres Node ("Get Property Media")**  
    - Query property_media table for image storage path by property_id.

14. **Add Code Node ("construct_image_url")**  
    - Construct full image URLs from storage paths.

15. **Add Merge Node ("Merge")**  
    - Combine property data with media URLs on property ID.

16. **Add LangChain Agent Node ("Format Results with AI")**  
    - Use GPT-4o-mini model with prompt to format property listings in natural, engaging language including images and key features.

17. **Add Set Node ("Format Response")**  
    - Assign formatted text to response field.

18. **Add Respond to Webhook Node ("Send Formatted Response")**  
    - Send AI formatted property listings as chatbot response.

19. **Add Postgres Node ("Save Personal Info")**  
    - Insert collected personal data into user_info table.

20. **Add Gmail Trigger Node ("Gmail Trigger")**  
    - Poll Gmail for incoming appointment emails every hour.

21. **Add LangChain Agent Node ("AI Agent")**  
    - Configure with detailed system prompt to process appointment confirmations, reschedules, cancellations.  
    - Integrate Google Calendar and Gmail nodes for event management and communication.

22. **Add Google Calendar Nodes:**  
    - "get_Calendar": Check availability on user-suggested dates.  
    - "create_Calendar1": Create calendar events on confirmation.  
    - "update_Calendar": Update existing events.  
    - "Google Calendar": Delete events as needed.  
    - "Google Calendar1": Retrieve all events for scheduling logic.

23. **Add Gmail Nodes:**  
    - "Gmail1": Send appointment confirmation emails.  
    - "Human agent": Send handoff emails to human agents when requested.

24. **Configure Credentials:**  
    - PostgreSQL credentials for DB queries.  
    - OpenAI API key for LangChain GPT nodes.  
    - Gmail OAuth2 credentials for Gmail nodes.  
    - Google Calendar OAuth2 credentials for calendar nodes.

25. **Set Up Database Structure:**  
    - user_details table for memory.  
    - user_info table for personal info storage.  
    - new_properties and property_media tables with appropriate fields.

26. **Ensure Proper Node Connections:**  
    - Webhook → Code (URL extraction) → If (link check) → Postgres → If1 → Code1 → Send Formatted Response  
    - If no URL: Webhook → Code → If → LLM conversation → Parse Data → Ready Check  
    - Ready Check True → Query Properties → Get Property Media → construct_image_url → Merge → Format Results with AI → Format Response → Send Formatted Response  
    - Ready Check True → Save Personal Info  
    - Gmail Trigger → AI Agent → Google Calendar nodes and Gmail1 for appointment processing  
    - Human agent handoff integrated within LLM conversation node processing.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The workflow implements strict content filtering to block non-property inquiries upfront, improving user experience and compliance. | See system prompt in LLM conversation node. |
| Uses PostgreSQL for persistent conversation memory and user data storage, enabling multi-turn dialogue without repeated questions. | Database tables: user_details, user_info. |
| Appointment management is fully automated via Gmail polling and Google Calendar API integration, minimizing manual scheduling overhead. | Gmail Trigger and AI Agent nodes. |
| AI formatting of property results ensures professional, engaging user responses highlighting key features and images. | Format Results with AI node prompt. |
| Human agent handoff process collects missing user info and specific messages before sending detailed emails for seamless escalation. | Human agent Gmail node and LLM conversation logic. |
| Workflow designed for Singapore real estate market context (HDB, condos, MRT stations). | Prompts and queries reference local property types and locations. |

---

**Disclaimer:**  
The text and design are exclusively based on an automated workflow created with n8n, following applicable content policies and containing no illegal or inappropriate content. All data handled is legal and public.