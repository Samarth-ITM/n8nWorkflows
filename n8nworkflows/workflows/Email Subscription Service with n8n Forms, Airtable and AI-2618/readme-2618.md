Email Subscription Service with n8n Forms, Airtable and AI

https://n8nworkflows.xyz/workflows/email-subscription-service-with-n8n-forms--airtable-and-ai-2618


# Email Subscription Service with n8n Forms, Airtable and AI

### 1. Workflow Overview

This workflow implements a fully automated Email Subscription Service that allows users to subscribe to receive regular AI-generated factoids on topics of their choice. It uses n8n forms for user input, Airtable as a subscriber database, AI models for content and image generation, and Gmail for email delivery. The workflow supports subscribing, unsubscribing, scheduled message sending on different intervals, and logs email deliveries.

The workflow is logically divided into the following functional blocks:

- **1.1 Subscribe Flow**: Captures user subscription data via an n8n form, stores or updates the subscriber in Airtable, and sends a confirmation email.

- **1.2 Unsubscribe Flow**: Captures unsubscribe requests via an n8n form and updates subscriber status in Airtable to inactive.

- **1.3 Scheduled Trigger & Subscriber Search**: A scheduled trigger runs daily at 9 AM, querying Airtable for active subscribers based on their selected intervals (daily, weekly, surprise).

- **1.4 Subscriber Event Preparation & Concurrent Processing**: For each found subscriber, prepares event data and triggers a subworkflow to generate AI content and images concurrently.

- **1.5 AI Content & Image Generation Subworkflow**: Generates a unique factoid text using AI agents, queries Wikipedia for reference, uses a Groq chat model, generates an illustrative image, and resizes it.

- **1.6 Email Composition & Sending**: Constructs personalized emails including the generated content, image attachment, and an unsubscribe link, then sends the email via Gmail.

- **1.7 Logging**: Updates the Airtable record with the timestamp of the last sent email for tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Subscribe Flow

- **Overview:**  
  This block captures user subscription requests via an n8n form, stores or updates subscriber information in Airtable, and sends a confirmation email.

- **Nodes Involved:**  
  - `Subscribe Form` (Form Trigger)  
  - `Create Subscriber` (Airtable Upsert)  
  - `confirmation email1` (Gmail Send)

- **Node Details:**

  - **Subscribe Form**  
    - Type: Form Trigger  
    - Role: Captures topic, email, and frequency from user input via a web form.  
    - Config: Path `free-factoids-subscribe`, form fields for topic (textarea), email, frequency (dropdown: daily, weekly, surprise me).  
    - Outputs: JSON with user's email, topic, frequency, and submission timestamp.  
    - Edge Cases: Invalid email format, missing required fields.

  - **Create Subscriber**  
    - Type: Airtable Node (Upsert)  
    - Role: Adds or updates subscriber record in Airtable with email, topic, status (set to active), interval, and start day (derived from submission date).  
    - Key Expressions: Uses form inputs and submission date formatted as day of the week.  
    - Input: JSON from `Subscribe Form`.  
    - Output: Airtable record confirmation.  
    - Failure Modes: Airtable API auth errors, duplicate entries, network issues.

  - **confirmation email1**  
    - Type: Gmail Node (Send Email)  
    - Role: Sends confirmation email to the subscriber.  
    - Config: Sends to email from form, includes topic and frequency in body, subject fixed as confirmation.  
    - Input: From `Create Subscriber` output, references form data via expressions.  
    - Failure Modes: Gmail auth errors, rate limits.

#### 1.2 Unsubscribe Flow

- **Overview:**  
  Handles unsubscribe requests by accepting subscriber ID and reason via an n8n form, then updates the subscriber status to inactive in Airtable.

- **Nodes Involved:**  
  - `Unsubscribe Form` (Form Trigger)  
  - `Update Subscriber` (Airtable Update)

- **Node Details:**

  - **Unsubscribe Form**  
    - Type: Form Trigger  
    - Role: Receives unsubscribe requests.  
    - Config: Path `free-factoids-unsubscribe`, fields: required ID, multi-select reasons for unsubscribing.  
    - Notes: Uses pre-fill form fields to identify user by ID rather than email, preventing unauthorized unsubscribe requests.  
    - Failure Modes: Missing ID, invalid IDs.

  - **Update Subscriber**  
    - Type: Airtable Node (Update)  
    - Role: Sets subscriber status to inactive based on ID from form.  
    - Input: ID from unsubscribe form submission.  
    - Failure Modes: Airtable auth, non-existent ID.

#### 1.3 Scheduled Trigger & Subscriber Search

- **Overview:**  
  Runs daily at 9 AM, searches Airtable for subscribers active on daily, weekly, or surprise intervals to determine who should receive emails.

- **Nodes Involved:**  
  - `Schedule Trigger` (Schedule Trigger)  
  - `Search daily` (Airtable Search)  
  - `Search weekly` (Airtable Search)  
  - `Search surprise` (Airtable Search)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow daily at 9:00 AM.  
    - Config: Interval set to 9 AM daily.  
    - Failure Cases: Missed triggers if n8n is offline.

  - **Search daily**  
    - Type: Airtable Search  
    - Role: Finds active subscribers with daily intervals.  
    - Filter Formula: `{Status} = 'active' AND {Interval} = 'daily'`  
    - Failure: Airtable API errors.

  - **Search weekly**  
    - Type: Airtable Search  
    - Role: Finds active subscribers with weekly intervals who haven't been sent an email in the last 7 days.  
    - Filter Formula: `{Status} = 'active' AND {Interval} = 'weekly' AND {Last Sent} <= DATEADD(TODAY(), -7, 'days')`  
    - Edge: Correct date handling critical to avoid missing sends.

  - **Search surprise**  
    - Type: Airtable Search  
    - Role: Finds active subscribers with 'surprise' interval.  
    - Filter Formula: `{Status} = 'active' AND {Interval} = 'surprise'`  
    - Followed by conditional random filtering to decide whether to send.

#### 1.4 Subscriber Event Preparation & Concurrent Processing

- **Overview:**  
  Prepares subscriber data for processing and concurrently executes a subworkflow to generate and send emails.

- **Nodes Involved:**  
  - `Should Send?` (Code Node)  
  - `Should Send = True` (Filter Node)  
  - `Create Event` (Set Node)  
  - `Execute Workflow` (Execute Workflow)  
  - `Execute Workflow Trigger` (Execute Workflow Trigger)  
  - `Execution Data` (Execution Data Node)

- **Node Details:**

  - **Should Send?**  
    - Type: Code Node  
    - Role: For 'surprise' interval subscribers, randomly decides with ~10% chance (luckyPick == 8) whether to send email.  
    - Output: Adds boolean `should_send` to JSON.  
    - Edge: Randomness could cause uneven distribution.

  - **Should Send = True**  
    - Type: Filter Node  
    - Role: Passes only subscribers where `should_send` is true.  
    - Input: Output from `Should Send?`  
    - Output: Subscribers to be processed.

  - **Create Event**  
    - Type: Set Node  
    - Role: Maps subscriber data fields (email, topic, interval, id, created_at) into consistent event object for processing.  
    - Input: Airtable subscriber record JSON.  
    - Output: Structured event JSON.

  - **Execute Workflow**  
    - Type: Execute Workflow  
    - Role: Runs the same workflow (self-reference) for each subscriber event concurrently without waiting.  
    - Config: Mode "each", waitForSubWorkflow false, workflow ID is current workflow ID.  
    - Purpose: Enables concurrent email generation and sending for scalability.

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Used within subworkflow execution to trigger next steps with subscriber data.

  - **Execution Data**  
    - Type: Execution Data Node  
    - Role: Exposes current execution data for downstream nodes to use, such as AI content generation.

#### 1.5 AI Content & Image Generation Subworkflow

- **Overview:**  
  Generates a unique factoid about the requested topic using AI and creates an accompanying child-friendly illustration image.

- **Nodes Involved:**  
  - `Content Generation Agent` (LangChain Agent)  
  - `Wikipedia` (LangChain Wikipedia Tool)  
  - `Groq Chat Model` (LangChain Language Model)  
  - `Window Buffer Memory` (LangChain Memory Buffer Window)  
  - `Generate Image` (OpenAI Image Generation)  
  - `Resize Image` (Image Edit)

- **Node Details:**

  - **Content Generation Agent**  
    - Type: LangChain Agent  
    - Role: Generates new factoid text on the subscriber's topic, ensuring uniqueness.  
    - Prompt: `"Generate an new factoid on the following topic: \"{{ $json.topic.replace('\"','') }}\" Ensure it is unique and not one generated previously."`  
    - Uses Wikipedia tool and Groq Chat Model as underlying AI resources.  
    - Failure Modes: AI API limits, prompt errors, memory context issues.

  - **Wikipedia**  
    - Type: LangChain Wikipedia Tool  
    - Role: Provides reference information to AI agent for fact generation.  
    - Input: Topic from event data.  
    - Failure: Wikipedia API unavailability.

  - **Groq Chat Model**  
    - Type: LangChain Language Model  
    - Role: AI language model used by agent for generating natural language output.  
    - Model: 'llama-3.3-70b-versatile'  
    - Credentials: Groq API credentials required.

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Stores recent session data keyed by subscriber email to prevent repetition.  
    - Session Key: `scheduled_send_{{ $json.email }}`  
    - Purpose: Helps ensure uniqueness of generated factoids.

  - **Generate Image**  
    - Type: OpenAI Image Generation Node  
    - Role: Generates child-friendly illustration to complement the factoid text.  
    - Prompt: `Generate a child-friendly illustration which compliments the following paragraph:\n{{ $json.output }}`  
    - Credentials: OpenAI API key required.

  - **Resize Image**  
    - Type: Image Edit Node  
    - Role: Resizes generated image to 480x360 pixels for email attachments.  
    - Failure Modes: Image processing errors, unsupported formats.

#### 1.6 Email Composition & Sending

- **Overview:**  
  Constructs a personalized email with AI-generated factoid and image, includes unsubscribe link, and sends it via Gmail.

- **Nodes Involved:**  
  - `Set Email Vars` (Set Node)  
  - `Send Message` (Gmail Send)  
  - `Log Last Sent` (Airtable Update)

- **Node Details:**

  - **Set Email Vars**  
    - Type: Set Node  
    - Role: Prepares email parameters: recipient(s), subject line, HTML message body (with formatted factoid), and unsubscribe link.  
    - Key Expressions:  
      - `to`: subscriber email and a fixed internal email (jim@height.io) for monitoring.  
      - `subject`: "Your {interval} factoid" dynamically based on subscriber interval.  
      - `message`: HTML snippet with bolded topic and italic factoid text.  
      - `unsubscribe_link`: URL with subscriber ID parameter for unsubscribe form.  
    - Includes binary image attachment from resized image node.

  - **Send Message**  
    - Type: Gmail Node (Send Email)  
    - Role: Sends the composed email to subscriber.  
    - Config: HTML formatted email, attachment included, attribution disabled to keep message clean.  
    - Credentials: Gmail OAuth2 required.  
    - Failure Modes: Gmail API limits, auth errors, invalid email addresses.

  - **Log Last Sent**  
    - Type: Airtable Update  
    - Role: Updates subscriber record with current timestamp in `Last Sent` field to track email sending.  
    - Input: Subscriber ID from execution data, timestamp set to execution time.  
    - Failure Modes: Airtable API errors.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                                  | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                              |
|-------------------------|-----------------------------|-------------------------------------------------|---------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------|
| Subscribe Form          | Form Trigger                | Capture subscription data                         |                                 | Create Subscriber               | ### 1. Subscribe flow Use a form to allow users to subscribe to the service.                           |
| Create Subscriber       | Airtable (Upsert)           | Add/update subscriber in Airtable                 | Subscribe Form                  | confirmation email1             |                                                                                                         |
| confirmation email1     | Gmail Send                  | Send confirmation email                           | Create Subscriber               |                                |                                                                                                         |
| Unsubscribe Form        | Form Trigger                | Capture unsubscribe requests                       |                                 | Update Subscriber              | ### 2. Unsubscribe flow * Uses Form's pre-fill field feature to identify user * Doesn't use "email" as identifier so you can't unsubscribe others |
| Update Subscriber       | Airtable (Update)           | Set subscriber status to inactive                  | Unsubscribe Form               |                                |                                                                                                         |
| Schedule Trigger        | Schedule Trigger            | Trigger daily at 9 AM                              |                                 | Search daily, Search weekly, Search surprise | ### 3. Scheduled Trigger * Runs every day at 9am * Handles all 3 frequency types * Send emails concurrently |
| Search daily            | Airtable Search             | Find active daily subscribers                       | Schedule Trigger               | Create Event                   |                                                                                                         |
| Search weekly           | Airtable Search             | Find active weekly subscribers needing emails      | Schedule Trigger               | Create Event                   |                                                                                                         |
| Search surprise         | Airtable Search             | Find active surprise interval subscribers          | Schedule Trigger               | Should Send?                  |                                                                                                         |
| Should Send?            | Code Node                  | Randomly decide if surprise subscriber should send | Search surprise                | Should Send = True             |                                                                                                         |
| Should Send = True      | Filter Node                | Filter subscribers where sending is true           | Should Send?                  | Create Event                   |                                                                                                         |
| Create Event            | Set Node                   | Prepare event data for each subscriber              | Search daily, Search weekly, Should Send = True | Execute Workflow            | ### 4. Using Subworkflows to run executions concurrently This configuration is desired when sequential execution is slow and unnecessary. Also if one email fails, it doesn't fail the execution for everyone else. |
| Execute Workflow        | Execute Workflow           | Run subworkflow concurrently for each event         | Create Event                  |                                |                                                                                                         |
| Execute Workflow Trigger| Execute Workflow Trigger   | Trigger subworkflow steps with subscriber data      | Execute Workflow              | Execution Data                |                                                                                                         |
| Execution Data          | Execution Data Node        | Expose current execution data                        | Execute Workflow Trigger      | Content Generation Agent      | ### 5. Use Execution Data to Filter Logs If you've registered for community+ or are on n8n cloud, best practice is to use execution node to allow filtering of execution logs. |
| Content Generation Agent| LangChain Agent            | Generate unique factoid text via AI                  | Execution Data                | Generate Image                | ### 6. Use AI to Generate Factoid and Image Use an AI agent to automate the generation of factoids as requested by the user. This is a simple example but we recommend a adding a unique touch to stand out from the crowd! |
| Wikipedia               | LangChain Wikipedia Tool   | Provide reference info to AI agent                   | Content Generation Agent (ai_tool) |                                |                                                                                                         |
| Groq Chat Model         | LangChain Language Model   | AI language model used by agent                      | Content Generation Agent (ai_languageModel) |                                |                                                                                                         |
| Window Buffer Memory    | LangChain Memory Buffer    | Store recent session data for uniqueness             | Content Generation Agent (ai_memory) |                                |                                                                                                         |
| Generate Image          | OpenAI Image Generation    | Generate illustration image for factoid              | Content Generation Agent      | Resize Image                 |                                                                                                         |
| Resize Image            | Image Edit Node            | Resize generated image to standard dimensions        | Generate Image                | Set Email Vars               |                                                                                                         |
| Set Email Vars          | Set Node                   | Prepare email fields and unsubscribe link            | Resize Image                 | Send Message                 | ### 7. Send Email to User Finally, send a message to the user with both text and image. Log the event in the Airtable for later analysis if required. ![Screenshot of email result](https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto/v1/n8n-workflows/dbpctdhohj3vlewy6oyc) |
| Send Message            | Gmail Send                 | Send the personalized email with content and image   | Set Email Vars               | Log Last Sent                |                                                                                                         |
| Log Last Sent           | Airtable Update            | Update subscriber's Last Sent timestamp               | Send Message                 |                                |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Subscribe Form Node**  
   - Type: Form Trigger  
   - Path: `free-factoids-subscribe`  
   - Form Title: "Learn something every day!"  
   - Form Fields:  
     - `topic` (textarea, required)  
     - `email` (email, required)  
     - `frequency` (dropdown: daily, weekly, surprise me, required)  
   - Response Text: "Thanks! Your factoid is on its way!"

2. **Create Airtable Upsert Node ("Create Subscriber")**  
   - Connect input from Subscribe Form.  
   - Airtable Base: Connect to your copied Airtable base.  
   - Table: Use subscriber table.  
   - Operation: Upsert based on `Email` field.  
   - Fields to set:  
     - Email: `{{$json.email}}`  
     - Topic: `{{$json.topic}}`  
     - Status: `active`  
     - Interval: `{{$json.frequency}}`  
     - Start Day: `{{$json.submittedAt.toDateTime().format('EEE')}}`

3. **Create Gmail Send Node ("confirmation email1")**  
   - Connect input from Airtable Upsert.  
   - Send to: `{{$json.email}}` from Subscribe Form.  
   - Subject: "Learn something every day confirmation"  
   - Message: Include topic and frequency using expressions referencing Subscribe Form data.  
   - Credentials: Configure Gmail OAuth2 credentials.

4. **Create Unsubscribe Form Node**  
   - Type: Form Trigger  
   - Path: `free-factoids-unsubscribe`  
   - Form Title: "Unsubscribe from Learn Something Every Day"  
   - Form Fields:  
     - `ID` (required)  
     - `Reason For Unsubscribe` (dropdown multi-select: Emails not relevant, Too many Emails, I did not sign up)  
   - Description: "We're sorry to see you go! Please take a moment to help us improve the service."

5. **Create Airtable Update Node ("Update Subscriber")**  
   - Connect input from Unsubscribe Form.  
   - Operation: Update record by `ID` field.  
   - Set `Status` to `inactive`.

6. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Trigger Time: Daily at 9:00 AM.

7. **Create Airtable Search Nodes for Intervals**  
   - Search daily: Filter `{Status} = 'active' AND {Interval} = 'daily'`  
   - Search weekly: Filter `{Status} = 'active' AND {Interval} = 'weekly' AND {Last Sent} <= DATEADD(TODAY(), -7, 'days')`  
   - Search surprise: Filter `{Status} = 'active' AND {Interval} = 'surprise'`

8. **For "surprise" subscribers, add Code Node "Should Send?"**  
   - Run once per item  
   - JS Code:  
     ```js
     const luckyPick = Math.floor(Math.random() * 10) + 1;
     $input.item.json.should_send = luckyPick == 8;
     return $input.item;
     ```
   
9. **Add Filter Node "Should Send = True"**  
   - Condition: `$json.should_send == true`

10. **Create Set Node "Create Event"**  
    - Input from all three search outputs (daily, weekly, filtered surprise)  
    - Map fields: `email`, `topic`, `interval`, `id`, `created_at` from Airtable record.

11. **Add Execute Workflow Node**  
    - Mode: each  
    - Wait for subworkflow: false  
    - Workflow ID: Set to current workflow ID (self-reference) for concurrent processing.  
    - Input from "Create Event".

12. **Within Subworkflow Execution:**  
    - Add Execute Workflow Trigger Node to receive event data.  
    - Add Execution Data Node for data exposure.

13. **Add LangChain Agent Node ("Content Generation Agent")**  
    - Prompt:  
      ```
      Generate an new factoid on the following topic: "{{ $json.topic.replace('"','') }}"
      Ensure it is unique and not one generated previously.
      ```  
    - Connect tools: Wikipedia Tool, Groq Chat Model, Window Buffer Memory for context.

14. **Add Wikipedia Tool Node**  
    - Connected as AI tool for LangChain Agent.

15. **Add Groq Chat Model Node**  
    - Configure with Groq API credentials.  
    - Connected as AI language model for Agent.

16. **Add Window Buffer Memory Node**  
    - Session Key: `scheduled_send_{{ $json.email }}`  
    - Connected as AI memory for Agent.

17. **Add OpenAI Image Generation Node ("Generate Image")**  
    - Prompt:  
      ```
      Generate a child-friendly illustration which compliments the following paragraph:
      {{ $json.output }}
      ```  
    - Resource: Image  
    - Credentials: OpenAI API key.

18. **Add Image Edit Node ("Resize Image")**  
    - Operation: Resize  
    - Width: 480  
    - Height: 360  
    - Input from "Generate Image".

19. **Add Set Node ("Set Email Vars")**  
    - Prepare email parameters:  
      - `to`: `{{$json.email}},jim@height.io` (monitoring email)  
      - `subject`: `"Your {{$json.interval}} factoid"`  
      - `message`: HTML body including bold topic and italic factoid text.  
      - `unsubscribe_link`: URL with subscriber ID for unsubscribe form.  
    - Include image binary data as attachment.

20. **Add Gmail Send Node ("Send Message")**  
    - Use prepared email vars.  
    - Send HTML email with attachment.  
    - Credentials: Gmail OAuth2.

21. **Add Airtable Update Node ("Log Last Sent")**  
    - Update subscriber record last sent timestamp with current time.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                        | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This n8n template demonstrates a subscriber service using n8n Forms as frontend, Airtable for storage, AI for content generation, and Gmail for email delivery.                                    | Workflow description, use case scenario.                                                        |
| Make a copy of sample Airtable here for database schema: https://airtable.com/appL3dptT6ZTSzY9v/shrLukHafy5bwDRfD                                                                                   | Airtable base required for operation.                                                           |
| Join n8n community for support: Discord (https://discord.com/invite/XPKeKXeB7d), Forum (https://community.n8n.io/)                                                                                   | Community support links.                                                                         |
| Using subworkflows with concurrent execution improves scalability and fault tolerance, so one email failure does not affect others.                                                                | Sticky Note on concurrent execution usage.                                                      |
| Use Execution Data node for better execution logs filtering if on n8n cloud or community+ subscription.                                                                                             | Best practice for execution log management.                                                     |
| The AI content generation uses LangChain agent with Wikipedia tool and Groq chat model, combined with memory buffer to ensure uniqueness of generated content.                                      | AI model integration details.                                                                   |
| Email includes an unsubscribe link with a unique subscriber ID, secured by using IDs rather than emails to prevent unwanted unsubscribes.                                                          | Security best practice for unsubscribe operations.                                             |
| The image generated is resized to standard dimensions to ensure compatibility with email clients.                                                                                                  | Email attachment optimization.                                                                  |
| The workflow supports extension to other messaging platforms such as WhatsApp, Telegram, or social media by replacing the Gmail send node accordingly.                                            | Customization potential note.                                                                   |

---

This document fully describes the workflow’s architecture, node-by-node configuration, and provides instructions to rebuild it manually. It anticipates likely failure points such as API authentication, form validation errors, and AI rate limits, enabling reliable deployment and customization.