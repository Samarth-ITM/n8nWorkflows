Automated Customer Onboarding with HubSpot, Gmail, and AI-powered Scheduling

https://n8nworkflows.xyz/workflows/automated-customer-onboarding-with-hubspot--gmail--and-ai-powered-scheduling-9036


# Automated Customer Onboarding with HubSpot, Gmail, and AI-powered Scheduling

### 1. Workflow Overview

This workflow automates the customer onboarding process by integrating HubSpot CRM, Gmail, and Google Calendar with AI-powered scheduling and email generation. It targets businesses that want to streamline onboarding new contacts by automatically sending personalized welcome emails, scheduling welcome calls, and assigning customer success managers (CSMs).

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Triggering:** Capturing new customer events from HubSpot or via webhook.
- **1.2 Data Enrichment:** Retrieving detailed contact info and owner assignment from HubSpot.
- **1.3 AI-driven Email Composition and Scheduling:** Using AI agents to draft personalized welcome emails and schedule calendar events dynamically.
- **1.4 Email Dispatch and CRM Update:** Sending the welcome email via Gmail and updating the contact owner in HubSpot.
- **1.5 Calendar Management Tools:** A set of nodes and an AI agent managing calendar event creation, updates, retrieval, and deletion.
- **1.6 Workflow Utilities and Notes:** Setup instructions, company data settings, and documentation sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Triggering

- **Overview:** This block listens for new customer events from HubSpot or webhook POST requests to initiate the onboarding process.
- **Nodes Involved:**
  - Webhook
  - HubSpot Trigger
  - Enter your company data here (Set node)
  - Get list of owners
  - Split Out owners
  - Get current owner
  - If a contact is created
- **Node Details:**

1. **Webhook**
   - Type: HTTP Webhook trigger
   - Role: Receives incoming POST requests, typically from HubSpot or other external services.
   - Config: Path and POST method set; no special options.
   - Input: External HTTP POST
   - Output: JSON payload of the incoming event
   - Edge cases: Network errors, invalid payloads, or unexpected subscription types.

2. **HubSpot Trigger**
   - Type: HubSpot event trigger node
   - Role: Listens specifically for HubSpot events related to CRM such as contact creation.
   - Config: Default event subscription; authenticated via HubSpot Developer API credentials.
   - Output: HubSpot event data.
   - Edge cases: Authentication failures, rate limits, event filter misconfiguration.

3. **Enter your company data here**
   - Type: Set node
   - Role: Stores static company-related variables such as company name, sender email, sender name, and company activity.
   - Config: Hard-coded values for company metadata, which are used downstream.
   - Edge cases: Incorrect email format or inconsistent sender email with HubSpot configuration may cause issues.

4. **Get list of owners**
   - Type: HTTP Request (HubSpot API call)
   - Role: Retrieves a list of owners (CSMs) from HubSpot for assignment.
   - Config: Authenticated via HubSpot OAuth2 credentials; GET request to `/crm/v3/owners`.
   - Edge cases: API failures, expired tokens, or empty owner list.

5. **Split Out owners**
   - Type: Split Out node
   - Role: Splits the array of owners from the API response into individual items for filtering.
   - Config: Splits on the `results` field.
   - Edge cases: Empty or malformed results field.

6. **Get current owner**
   - Type: Filter node
   - Role: Filters owners to find the one matching the sender_email set in the company data node.
   - Config: Condition checks if the owner's email equals the sender_email.
   - Edge cases: No matching owner found, case sensitivity errors.

7. **If a contact is created**
   - Type: If node
   - Role: Checks if the incoming event from the webhook is a contact creation event by evaluating the `subscriptionType` field.
   - Config: Condition equals `contact.creation`.
   - Output: Proceeds only if true.
   - Edge cases: Incorrect event types, missing fields.

---

#### 1.2 Data Enrichment

- **Overview:** This block fetches full contact details from HubSpot and prepares for personalized communications.
- **Nodes Involved:**
  - Get all info about the contact
- **Node Details:**

1. **Get all info about the contact**
   - Type: HubSpot node (contact retrieval)
   - Role: Retrieves detailed contact properties from HubSpot using contact ID from the webhook.
   - Config: Uses OAuth2 authentication; extracts `objectId` from webhook payload dynamically.
   - Edge cases: Contact not found, API errors, permission issues.

---

#### 1.3 AI-driven Email Composition and Scheduling

- **Overview:** Uses AI agents to draft personalized welcome emails and schedule a welcome call by interacting with calendar tools.
- **Nodes Involved:**
  - Write a personalized message (LangChain Agent)
  - calendarAgent (LangChain Tool Workflow)
  - OpenAI Chat Model2
  - Structured Output Parser
  - Transforms markdown to HTML
- **Node Details:**

1. **Write a personalized message**
   - Type: LangChain AI Agent node
   - Role: Generates a tailored welcome email message. Instructed to notify the recipient about an upcoming invitation for a meeting and to use the calendarAgent to schedule it.
   - Config: Uses a detailed prompt with sender and recipient variables, calendar interaction instructions, and expected JSON output.
   - Input: Contact information and sender’s metadata.
   - Output: Structured JSON with `subject` and `body`.
   - Edge cases: AI generation errors, malformed JSON output, or failed calendar scheduling.

2. **calendarAgent**
   - Type: LangChain Tool Workflow node
   - Role: Invokes a sub-workflow responsible for calendar event management (create, get, update, delete).
   - Config: Calls the current workflow as a tool with no required input parameters.
   - Edge cases: Circular calls, workflow ID mismatches.

3. **OpenAI Chat Model2**
   - Type: LangChain OpenAI chat model node
   - Role: Secondary AI model used for chat-based language generation (GPT-4o-mini).
   - Config: Authenticated with OpenAI API credentials.
   - Edge cases: API rate limits, model unavailability.

4. **Structured Output Parser**
   - Type: LangChain output parser
   - Role: Parses AI output into a structured JSON format, ensuring the email subject and body are properly extracted.
   - Config: Uses a JSON schema example with `subject` and `body`.
   - Edge cases: Parsing failures if AI output deviates from expected format.

5. **Transforms markdown to HTML**
   - Type: Markdown node
   - Role: Converts the markdown-formatted email body into HTML for email dispatch.
   - Edge cases: Malformed markdown, conversion errors.

---

#### 1.4 Email Dispatch and CRM Update

- **Overview:** Sends the personalized email via Gmail and updates the assigned owner in HubSpot.
- **Nodes Involved:**
  - Send the message
  - Set owner to contact
- **Node Details:**

1. **Send the message**
   - Type: Gmail node
   - Role: Sends the generated welcome email to the contact’s email address.
   - Config: Uses OAuth2 Gmail credentials; BCCs a fixed internal email; Subject and Body dynamically set from AI output.
   - Edge cases: Authentication errors, email delivery failures, invalid recipient email.

2. **Set owner to contact**
   - Type: HubSpot node (contact update)
   - Role: Assigns the contact owner in HubSpot to the matched owner found earlier.
   - Config: Updates contact owner field with owner ID dynamically.
   - Edge cases: API failures, permission issues.

---

#### 1.5 Calendar Management Tools

- **Overview:** Implements calendar event management functionalities via Google Calendar nodes and an AI agent to interact with them.
- **Nodes Involved:**
  - Calendar Agent (LangChain Agent)
  - Create Event with Attendee
  - Create Event
  - Get Events
  - Delete Event
  - Update Event
- **Node Details:**

1. **Calendar Agent**
   - Type: LangChain Agent node
   - Role: Acts as a calendar assistant AI that processes queries to create, get, delete, or update calendar events.
   - Config: System prompt defines calendar management responsibilities and usage rules; uses current date/time context.
   - Edge cases: API failures, incorrect AI instructions, unknown calendar IDs.

2. **Create Event with Attendee**
   - Type: Google Calendar node
   - Role: Creates calendar events with specified attendees.
   - Config: Uses AI-derived event start/end times, title, and attendee email; authenticated via Google OAuth2.
   - Edge cases: Invalid calendar ID, attendee email, or time conflicts.

3. **Create Event**
   - Type: Google Calendar node
   - Role: Creates calendar events without attendees.
   - Config: Similar to above but no attendees.
   - Edge cases: Same as above.

4. **Get Events**
   - Type: Google Calendar node
   - Role: Retrieves all calendar events within a specified time range.
   - Config: Time range dynamically set using AI input or defaults (today to 7 days ahead).
   - Edge cases: API limits, time zone mismatches.

5. **Delete Event**
   - Type: Google Calendar node
   - Role: Deletes a calendar event by ID.
   - Config: Requires prior retrieval of event ID from AI input.
   - Edge cases: Event not found, permission denied.

6. **Update Event**
   - Type: Google Calendar node
   - Role: Updates event start and end times by ID.
   - Config: Uses AI-provided new start and end times.
   - Edge cases: Invalid event ID, conflicting updates.

---

#### 1.6 Workflow Utilities and Notes

- **Overview:** Contains documentation, setup instructions, and company-specific static data.
- **Nodes Involved:**
  - Sticky Notes (multiple)
  - When Executed by Another Workflow
  - OpenAI Model
- **Node Details:**

1. **Sticky Notes**
   - Purpose: Provide detailed setup instructions for webhook creation (n8n and HubSpot), company data notes, calendar tool attribution, and contact info for support.
   - Content: Includes detailed markdown instructions and credits.
   - Edge cases: None, purely informational.

2. **When Executed by Another Workflow**
   - Type: Execute Workflow Trigger node
   - Role: Allows this workflow to be triggered from another workflow with passthrough input.
   - Edge cases: Input mismatch or missing data when called externally.

3. **OpenAI Model**
   - Type: LangChain OpenAI model node (gpt-4o)
   - Role: Provides AI language model support for calendar agent.
   - Edge cases: Same as other OpenAI nodes.

---

### 3. Summary Table

| Node Name                    | Node Type                        | Functional Role                             | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                      |
|------------------------------|---------------------------------|---------------------------------------------|---------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook                      | HTTP Webhook Trigger             | Entry point via POST webhook                 | -                               | Enter your company data here     | ## What does it do? Objective: Streamline onboarding, trigger webhook or CRM trigger, send emails, schedule calls, assign CSM |
| HubSpot Trigger              | HubSpot Trigger                  | Entry point via HubSpot events               | -                               | Enter your company data here     | ## What does it do? (shared with Webhook node)                                                                  |
| Enter your company data here | Set                             | Stores static company and sender info       | Webhook, HubSpot Trigger         | Get list of owners               | ## Set your data and your company's The sender_email you set here has to be the same as the one you use in hubspot |
| Get list of owners           | HTTP Request (HubSpot API)       | Retrieves list of CRM owners                 | Enter your company data here     | Split Out owners                |                                                                                                                 |
| Split Out owners             | Split Out                       | Splits owners array to individual items     | Get list of owners               | Get current owner               |                                                                                                                 |
| Get current owner            | Filter                         | Finds owner matching sender_email            | Split Out owners                 | If a contact is created          |                                                                                                                 |
| If a contact is created      | If                             | Checks if event is contact creation          | Get current owner                | Get all info about the contact   |                                                                                                                 |
| Get all info about the contact | HubSpot                       | Retrieves detailed contact info              | If a contact is created          | Write a personalized message     |                                                                                                                 |
| Write a personalized message | LangChain AI Agent             | Generates personalized welcome email & schedules call | Get all info about the contact | Transforms markdown to HTML      | ## Email writer - This agent writes a personalized Email - Uses the calendar Agent tool to create an appointment on an empty slot. Feel free to personalize the prompt |
| Transforms markdown to HTML  | Markdown                       | Converts markdown email body to HTML         | Write a personalized message    | Send the message                |                                                                                                                 |
| Send the message             | Gmail                         | Sends the welcome email                       | Transforms markdown to HTML      | Set owner to contact             |                                                                                                                 |
| Set owner to contact         | HubSpot                       | Sets the contact owner in CRM                 | Send the message                | -                               |                                                                                                                 |
| Calendar Agent              | LangChain Agent                | AI calendar assistant for managing events    | Multiple calendar nodes          | Success, Try Again              | ## Calendar tool This part has been borrowed from the excellent [Nate Herk](https://www.youtube.com/@nateherk) youtube channel |
| Create Event with Attendee  | Google Calendar Tool           | Creates event with attendee                    | Calendar Agent                  | Calendar Agent                 |                                                                                                                 |
| Create Event                | Google Calendar Tool           | Creates event without attendee                 | Calendar Agent                  | Calendar Agent                 |                                                                                                                 |
| Get Events                  | Google Calendar Tool           | Retrieves calendar events                      | Calendar Agent                  | Calendar Agent                 |                                                                                                                 |
| Delete Event                | Google Calendar Tool           | Deletes specified calendar event               | Calendar Agent                  | Calendar Agent                 |                                                                                                                 |
| Update Event                | Google Calendar Tool           | Updates calendar event                          | Calendar Agent                  | Calendar Agent                 |                                                                                                                 |
| When Executed by Another Workflow | Execute Workflow Trigger   | Allows external workflows to trigger this one | -                             | Calendar Agent                 |                                                                                                                 |
| OpenAI Chat Model2           | LangChain OpenAI Chat Model   | Secondary AI model for chat-based generation  | -                             | Write a personalized message     |                                                                                                                 |
| Structured Output Parser    | LangChain Output Parser       | Parses AI output into JSON                      | -                             | Write a personalized message     |                                                                                                                 |
| OpenAI Model                | LangChain OpenAI Model        | AI language model for calendar agent           | -                             | Calendar Agent                 |                                                                                                                 |
| Sticky Note                 | Sticky Note                   | Setup instructions, info, credits              | -                             | -                               | Multiple sticky notes with detailed setup instructions and contact info (see section 1.6)                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Webhook` node:**
   - Set HTTP method to POST.
   - Define a unique webhook path (e.g., `/06d29616-8fa9-42cf-8b5f-abe856083c75`).
   - Leave response mode as default.
   - This node triggers when a new customer event is posted.

3. **Add a `HubSpot Trigger` node:**
   - Authenticate with HubSpot Developer API credentials.
   - Subscribe to contact creation events or other relevant CRM events.
   - This node triggers on new HubSpot events.

4. **Add a `Set` node named “Enter your company data here”:**
   - Define variables:
     - `company_name`: e.g., "Pollup Data Services"
     - `sender_name`: e.g., "Thomas Vié"
     - `sender_email`: e.g., "zeerobug@gmail.com" (must match HubSpot sender email)
     - `company_activity`: brief company description.
   - Connect both the Webhook and HubSpot Trigger nodes to this node.

5. **Add an `HTTP Request` node named “Get list of owners”:**
   - Set method to GET.
   - URL: `https://api.hubapi.com/crm/v3/owners`
   - Authenticate with HubSpot OAuth2 credentials.
   - Connect from “Enter your company data here”.

6. **Add a `Split Out` node named “Split Out owners”:**
   - Set field to split: `results`
   - Connect from “Get list of owners”.

7. **Add a `Filter` node named “Get current owner”:**
   - Condition: owner’s email equals `sender_email` from “Enter your company data here”.
   - Connect from “Split Out owners”.

8. **Add an `If` node named “If a contact is created”:**
   - Condition: Check if `subscriptionType` in webhook payload equals `contact.creation`.
   - Connect from “Get current owner”.

9. **Add a `HubSpot` node named “Get all info about the contact”:**
   - Operation: Get contact by ID.
   - ID: Use `objectId` from webhook payload.
   - Authenticate with HubSpot OAuth2.
   - Connect from “If a contact is created”.

10. **Add a `LangChain Agent` node named “Write a personalized message”:**
    - Configure prompt to produce a personalized welcome email JSON with subject and body.
    - Include instructions to use the calendarAgent to schedule a meeting.
    - Pass sender and recipient variables from previous nodes.
    - Connect from “Get all info about the contact”.

11. **Add a `Markdown` node named “Transforms markdown to HTML”:**
    - Mode: markdownToHtml.
    - Input: The `body` field from AI output.
    - Connect from “Write a personalized message”.

12. **Add a `Gmail` node named “Send the message”:**
    - Configure OAuth2 Gmail credentials.
    - Set recipient to contact’s email.
    - BCC to internal email (e.g., thomas@pollup.net).
    - Subject and message body from AI output and markdown conversion.
    - Connect from “Transforms markdown to HTML”.

13. **Add a `HubSpot` node named “Set owner to contact”:**
    - Operation: Update contact owner.
    - Email: Contact’s email.
    - Owner ID: From “Get current owner”.
    - Connect from “Send the message”.

14. **Calendar Management Sub-Workflow:**

    - Add Google Calendar OAuth2 credentials.
    - Create nodes:
      - `Calendar Agent` (LangChain Agent) with system prompt for calendar management.
      - `Create Event with Attendee`, `Create Event`, `Get Events`, `Delete Event`, `Update Event` nodes configured to use data from AI agent.
    - Connect all these calendar nodes with the `Calendar Agent` node as AI tool inputs and outputs.
    - This sub-workflow is invoked via the `calendarAgent` tool workflow node inside the main workflow.

15. **Add a `LangChain Tool Workflow` node named “calendarAgent”:**
    - Set to call the current workflow’s ID.
    - No input parameters required.
    - Connect from `OpenAI Chat Model2` and to `Write a personalized message`.

16. **Add `OpenAI` nodes:**
    - `OpenAI Chat Model2`: GPT-4o-mini model with OpenAI API credentials.
    - `OpenAI Model`: GPT-4o model for calendar agent.
    - Connect appropriately for AI text generation and agent processing.

17. **Add `Execute Workflow Trigger` node named “When Executed by Another Workflow”:**
    - Input source: passthrough.
    - Connect to `Calendar Agent` for external triggering capability.

18. **Add Sticky Notes for documentation:**
    - Include instructions for webhook setup, company data notes, calendar tool credits, and contact info as per original workflow.

19. **Activate the workflow to begin listening and processing events.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow objective is to streamline onboarding by automatically sending welcome emails, scheduling welcome calls, and assigning CSMs. Triggered via webhook or HubSpot events.                                                                                                                                                                                                                                                                                                                                                              | Workflow purpose                                                                                     |
| Setup instructions for webhook creation in n8n and HubSpot, including setting webhook URLs and testing.                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note1 content                                                                                |
| Company data must be consistent especially the sender_email, which should match HubSpot configuration.                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note2 content                                                                                |
| Calendar management tool implementation inspired by Nate Herk’s YouTube channel: https://www.youtube.com/@nateherk                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note3 content                                                                                |
| Email writer AI agent uses calendarAgent tool to schedule appointments in free slots; prompt text can be personalized.                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note4 content                                                                                |
| Contact for workflow modification or assistance: [thomas@pollup.net](mailto:thomas@pollup.net), and link to other workflows: https://n8n.io/creators/zeerobug/                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note7 content                                                                                |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.