Turn Monoprix Delivery Emails into Calendar Events using ChatGPT and Google Calendar

https://n8nworkflows.xyz/workflows/turn-monoprix-delivery-emails-into-calendar-events-using-chatgpt-and-google-calendar-4771


# Turn Monoprix Delivery Emails into Calendar Events using ChatGPT and Google Calendar

---

### 1. Workflow Overview

This n8n workflow automates the process of converting Monoprix grocery delivery confirmation emails into Google Calendar events. It targets users who want to keep track of their grocery deliveries seamlessly by having delivery details automatically added to their shared family Google Calendar. The workflow integrates Gmail for email retrieval, OpenAI’s GPT model for information extraction, and Google Calendar for event creation.

The logical blocks are:

- **1.1 Input Reception:** Gmail Trigger node that listens for new Monoprix order confirmation emails.
- **1.2 AI Processing:** OpenAI Chat Model node and Information Extractor node that parse and extract key delivery details from the email content.
- **1.3 Data Transformation:** Set node formats and prepares the extracted data fields for calendar event creation.
- **1.4 Calendar Integration:** Google Calendar node creates a calendar event based on the extracted and formatted delivery information.
- **1.5 Workflow Setup Notes:** Sticky Note nodes provide user instructions and customization hints.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block listens for incoming emails from Monoprix confirming grocery deliveries, filtering specifically by sender and subject.
- **Nodes Involved:** `Pull email from Gmail`
- **Node Details:**

  - **Pull email from Gmail**
    - *Type:* Gmail Trigger
    - *Technical Role:* Event trigger node that polls Gmail account for new emails matching specified filters.
    - *Configuration:*
      - Filter query: `"Confirmation de commande Monoprix.fr"` (subject match).
      - Filter sender: `courses@monoprix.fr`.
      - Polling schedule: Runs every hour at minute 32 and every 15 minutes.
    - *Expressions:* Uses Gmail search filters internally.
    - *Input:* None (trigger node).
    - *Output:* Passes new matching email data (including email body in HTML) to the next node.
    - *Edge Cases / Failures:* Gmail authentication errors; email format changes that cause filtering to fail; Gmail API rate limits or downtime.
    - *Notes:* Credentials required: Gmail OAuth2.

#### 1.2 AI Processing

- **Overview:** This block uses OpenAI GPT-4 model via LangChain integration to extract structured delivery information from the email body.
- **Nodes Involved:** `OpenAI Chat Model`, `Extract key info`
- **Node Details:**

  - **OpenAI Chat Model**
    - *Type:* LangChain OpenAI Chat Model node
    - *Technical Role:* Sends the email content to GPT-4o-mini for language understanding and information extraction.
    - *Configuration:*
      - Model selected: `gpt-4o-mini`, a specialized GPT-4 variant.
      - No custom options set.
    - *Input:* Email content JSON from Gmail trigger.
    - *Output:* Language model response with extracted data.
    - *Credentials:* Requires OpenAI API key.
    - *Edge Cases:* API rate limits; invalid or malformed email body causing extraction errors; network issues.
  
  - **Extract key info**
    - *Type:* LangChain Information Extractor node
    - *Technical Role:* Applies a system prompt template instructing the AI to extract specific fields reliably from the email’s HTML content.
    - *Configuration:*
      - System prompt instructs extraction of: `order_id`, `delivery_date`, `delivery_time_start`, `delivery_time_end`, and inferred `delivery_time_zone`.
      - Ignores fields like `delivery_address`, `delivery_fee`, `order_items` to reduce noise.
      - Input format modeled on Monoprix confirmation email structure.
      - Attributes explicitly defined with required fields and types.
    - *Input:* HTML email body as text.
    - *Output:* JSON object with extracted fields.
    - *Edge Cases:* Email format changes; missing or ambiguous data; time zone inference may fail if address is unclear.
    - *Notes:* This node depends on the OpenAI Chat Model node’s output.

#### 1.3 Data Transformation

- **Overview:** Prepares and formats the extracted fields into explicit values suitable for creating calendar events.
- **Nodes Involved:** `Set GCal fields`
- **Node Details:**

  - **Set GCal fields**
    - *Type:* Set node
    - *Technical Role:* Maps extracted output fields into individual calendar event parameters.
    - *Configuration:*
      - Creates new fields:
        - `order_id` from `output.order_id`
        - `delivery_date`
        - `delivery_time_range` as concatenation of start and end times with dash
        - `delivery_datetime_start` combining date, start time, and timezone
        - `delivery_datetime_end` combining date, end time, and timezone
    - *Input:* JSON from Information Extractor node.
    - *Output:* Formatted JSON with calendar fields.
    - *Edge Cases:* Missing or invalid field values may cause downstream errors.
    - *Expressions:* Uses interpolation expressions like `={{ $json.output.order_id }}`.

#### 1.4 Calendar Integration

- **Overview:** Creates a Google Calendar event with the delivery details, adjusting start and end times for buffer periods.
- **Nodes Involved:** `Update Calendar`
- **Node Details:**

  - **Update Calendar**
    - *Type:* Google Calendar node
    - *Technical Role:* Inserts the delivery event into a shared Google Calendar.
    - *Configuration:*
      - Calendar ID: Shared family calendar ID `8d1klcuno3tcg773mh0s6geqic@group.calendar.google.com`.
      - Start time: `delivery_datetime_start` minus 30 minutes (buffer before delivery window).
      - End time: `delivery_datetime_end` plus 30 minutes (buffer after delivery window).
      - Event summary: Emoji and time range included, e.g. "🛒 Grocery delivery (07:00-08:00)".
      - Event description includes order ID and delivery details.
      - No default reminders set.
    - *Input:* JSON from Set node with formatted date/time and order data.
    - *Output:* Confirmation or event data (not used downstream).
    - *Credentials:* Google Calendar OAuth2 required.
    - *Edge Cases:* Invalid date/time format causing API errors; calendar access permissions; API rate limits.

#### 1.5 Workflow Setup Notes

- **Overview:** Provides users with instructions and tips for adapting or running the workflow.
- **Nodes Involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`
- **Node Details:**

  - **Sticky Note**
    - Instructions to add Gmail, ChatGPT, and Google Calendar credentials.
    - Advises testing workflow before activation.
  
  - **Sticky Note1**
    - Suggests changing email and subject filters to adapt workflow to other grocery providers.
  
  - **Sticky Note2**
    - Mentions updating the email format in the AI extraction step if switching providers.
  
  - **Sticky Note3**
    - Encourages updating calendar event content as needed.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                      | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                          |
|---------------------|----------------------------------|------------------------------------|--------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Pull email from Gmail| Gmail Trigger                    | Input reception of Monoprix emails | —                        | Extract key info         | Setup: Add Gmail, ChatGPT, and Google Calendar credentials; test and activate workflow              |
| OpenAI Chat Model    | LangChain OpenAI Chat Model       | AI processing of email content     | Pull email from Gmail    | Extract key info         |                                                                                                    |
| Extract key info     | LangChain Information Extractor   | Extracts delivery details via AI   | Pull email from Gmail, OpenAI Chat Model | Set GCal fields        | Update email format if using another grocery provider                                               |
| Set GCal fields      | Set Node                        | Data transformation for calendar   | Extract key info         | Update Calendar         | Update email format if using another grocery provider                                               |
| Update Calendar      | Google Calendar                 | Create calendar event               | Set GCal fields          | —                       | Feel free to update calendar content                                                               |
| Sticky Note          | Sticky Note                    | Setup instructions                  | —                        | —                       | Add your Gmail, ChatGPT, and Google Calendar credentials; test workflow                             |
| Sticky Note1         | Sticky Note                    | Instructions for provider changes  | —                        | —                       | Update email and subject if using another grocery delivery provider                                |
| Sticky Note2         | Sticky Note                    | Instructions for email format change| —                        | —                       | Update email format for extraction step if using another provider                                  |
| Sticky Note3         | Sticky Note                    | Instructions for calendar content  | —                        | —                       | Update calendar content as desired                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**
   - Type: Gmail Trigger
   - Credentials: Set up Gmail OAuth2 credentials with access to your account.
   - Parameters:
     - Filters:
       - Query: `"Confirmation de commande Monoprix.fr"`
       - Sender: `courses@monoprix.fr`
     - Polling:
       - Every hour at minute 32
       - Every 15 minutes
   - Connect this node’s output to the next AI processing node.

2. **Create OpenAI Chat Model Node**
   - Type: LangChain OpenAI Chat Model
   - Credentials: Enter your OpenAI API key.
   - Parameters:
     - Model: Select `gpt-4o-mini`.
     - Use default options.
   - Connect input from Gmail Trigger node (email content).
   - Output connects to Information Extractor node.

3. **Create Information Extractor Node**
   - Type: LangChain Information Extractor
   - Parameters:
     - Input text: Use `{{$json.html}}` from email body.
     - System Prompt Template: Use the detailed extraction instructions for Monoprix order confirmation emails:
       - Extract fields: `order_id`, `delivery_date`, `delivery_time_start`, `delivery_time_end`, `delivery_time_zone` (inferred).
       - Ignore unnecessary fields like `delivery_address`, `delivery_fee`, `amount_paid`, `order_items`.
       - Format and example of input email body as per Monoprix.
     - Attributes: Define fields with required flags and types (date, string).
   - Connect input from OpenAI Chat Model node.
   - Output connects to Set node.

4. **Create Set Node for Google Calendar Fields**
   - Type: Set
   - Parameters:
     - Create new fields:
       - `order_id` = `{{$json.output.order_id}}`
       - `delivery_date` = `{{$json.output.delivery_date}}`
       - `delivery_time_range` = `{{$json.output.delivery_time_start}}-{{$json.output.delivery_time_end}}`
       - `delivery_datetime_start` = `{{$json.output.delivery_date}} {{$json.output.delivery_time_start}} {{$json.output.delivery_time_zone}}`
       - `delivery_datetime_end` = `{{$json.output.delivery_date}} {{$json.output.delivery_time_end}} {{$json.output.delivery_time_zone}}`
   - Connect input from Information Extractor node.
   - Output connects to Google Calendar node.

5. **Create Google Calendar Node**
   - Type: Google Calendar
   - Credentials: Configure Google Calendar OAuth2 with access to the target calendar.
   - Parameters:
     - Calendar ID: Use your family/shared calendar ID.
     - Start time: Use expression `{{$json.delivery_datetime_start.toDateTime().minus(30, 'minutes')}}` to buffer 30 minutes earlier.
     - End time: Use expression `{{$json.delivery_datetime_end.toDateTime().plus(30, 'minutes')}}` to buffer 30 minutes later.
     - Summary: `"🛒 Grocery delivery ({{$json.delivery_time_range}})"`
     - Description: Include order ID, delivery date, and time range.
     - Disable default reminders.
   - Connect input from Set node.

6. **Add Sticky Notes (Optional)**
   - Add sticky note nodes in the editor for instructions:
     - Setup steps for credentials and testing.
     - Guidance on changing email filters and formats for other providers.
     - Tips on updating calendar event content.

7. **Activate Workflow**
   - Test the workflow using the Test button.
   - After successful test, activate the workflow for automatic operation.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                            |
|-------------------------------------------------------------------------------------------------|-------------------------------------------|
| Workflow tailored for Monoprix grocery delivery emails; can be adapted for other providers.     | Change email filter and extraction prompts as needed. |
| Uses LangChain nodes for AI extraction, leveraging OpenAI GPT-4o-mini for robust parsing.       | OpenAI API required.                       |
| Google Calendar event creation includes 30-minute buffers before and after delivery window.     | Improves calendar visibility and preparation time. |
| Shared calendar ID is configurable to fit family or group calendars.                            | Replace with your own calendar ID.        |
| Instructions recommend testing workflow before activation to verify credentials and logic.      | Sticky notes inside workflow detail this. |

---

Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.

---