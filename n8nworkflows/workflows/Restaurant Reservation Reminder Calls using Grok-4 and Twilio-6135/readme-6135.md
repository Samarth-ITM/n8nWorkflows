Restaurant Reservation Reminder Calls using Grok-4 and Twilio

https://n8nworkflows.xyz/workflows/restaurant-reservation-reminder-calls-using-grok-4-and-twilio-6135


# Restaurant Reservation Reminder Calls using Grok-4 and Twilio

### 1. Workflow Overview

This workflow automates reminder phone calls for restaurant reservations using AI-generated conversational scripts and Twilio's telephony services. It targets restaurants or similar venues aiming to reduce no-shows and improve customer engagement by sending polite, personalized calls reminding customers of their bookings.

The workflow is logically divided into these blocks:

- **1.1 Triggering Block**: Handles manual and scheduled initiation of the workflow.
- **1.2 Data Retrieval Block**: Fetches daily reservation data from a Google Sheet.
- **1.3 Processing Loop Block**: Iterates over each reservation to generate and deliver reminders.
- **1.4 AI Conversation Generation Block**: Uses AI (Grok-4 and LangChain Agent) to create personalized call scripts.
- **1.5 Call Execution Block**: Uses Twilio to place calls delivering the AI-generated messages.
- **1.6 Post-Call Update and Wait Block**: Updates the Google Sheet marking reservations as called and waits before processing the next batch.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering Block

- **Overview:** Initiates the workflow either manually or automatically every day at 11 AM.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Enables manual workflow start for testing or on-demand execution.  
    - Configuration: Default manual trigger, no parameters.  
    - Input: None  
    - Output: Connects to "Get daily reservations"  
    - Edge cases: None significant; manual start depends on user action.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow daily at 11:00 AM.  
    - Configuration: Set with a rule to trigger at hour 11 daily.  
    - Input: None  
    - Output: Connects to "Get daily reservations"  
    - Edge cases: Timezone considerations may affect trigger time; ensure server timezone aligns with intended schedule.

---

#### 2.2 Data Retrieval Block

- **Overview:** Retrieves all reservation entries for the current day from a Google Sheet, filtering out those already called.
- **Nodes Involved:**  
  - Get daily reservations

- **Node Details:**

  - **Get daily reservations**  
    - Type: Google Sheets (Read operation)  
    - Role: Reads reservation records filtered where the 'DATE' equals today's date and 'CALLED' is empty (not yet called).  
    - Configuration:  
      - Document: Google Sheets document identified by ID (linked to a shared booking spreadsheet)  
      - Sheet: First sheet (gid=0)  
      - Filters:  
        - 'CALLED' column is empty  
        - 'DATE' column matches the current date formatted as 'yyyy-LL-dd'  
    - Input: Trigger nodes (manual or scheduled)  
    - Output: Data items with reservations to call  
    - Credentials: Google Sheets OAuth2 account  
    - Edge cases:  
      - Empty or missing sheet data  
      - Authentication failure with Google API  
      - Date format mismatch causing filter failure

---

#### 2.3 Processing Loop Block

- **Overview:** Processes reservation data one by one in batches to handle each call sequentially.
- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Splits the array of reservations into batches of size 1 to process them individually.  
    - Configuration: Default batch size (1), no special options.  
    - Input: Output from "Get daily reservations"  
    - Output: Individual reservation objects, forwarded to AI conversation generation  
    - Edge cases:  
      - Empty input array (no reservations today)  
      - Batch processing delays if too many entries

---

#### 2.4 AI Conversation Generation Block

- **Overview:** Constructs a personalized, polite phone call script for each reservation using AI models.
- **Nodes Involved:**  
  - Grok 4  
  - Secretary Agent

- **Node Details:**

  - **Grok 4**  
    - Type: LangChain OpenRouter Language Model Chat (AI model interface)  
    - Role: Provides the AI model "x-ai/grok-4" for natural language generation.  
    - Configuration: Model set to Grok-4, no additional parameters.  
    - Input: Text prompt from "Secretary Agent" node  
    - Output: AI-generated conversational text  
    - Credentials: OpenRouter API key  
    - Edge cases: API rate limits, network failures, model unavailability

  - **Secretary Agent**  
    - Type: LangChain Agent (AI agent node)  
    - Role: Formats and sends reservation details to AI model to generate reminder call dialogue.  
    - Configuration:  
      - System message defines the assistant as a restaurant secretary for "Bella Napoli".  
      - Input prompt includes reservation details: Name, Time, Number of People.  
      - Output parser enabled to extract clean message text.  
    - Inputs: Single reservation item from "Loop Over Items"  
    - Outputs: AI-generated text used in the Twilio call  
    - Edge cases: Expression errors if fields missing, API failures  
    - Version-specific: Uses version 1.8 LangChain agent features.

---

#### 2.5 Call Execution Block

- **Overview:** Places a phone call to the customer using Twilio, reading out the AI-generated reminder message in Italian.
- **Nodes Involved:**  
  - Make a call

- **Node Details:**

  - **Make a call**  
    - Type: Twilio node (Call resource)  
    - Role: Initiates a phone call to the customer’s phone number using a configured Twilio number.  
    - Configuration:  
      - To: Customer phone number with international prefix (constructed dynamically).  
      - From: Twilio phone number (configured with country code "+177526xxxxxx" placeholder).  
      - TwiML: XML script instructing Twilio to speak the AI-generated message with voice "alice" in Italian ("it-IT").  
      - Uses Twilio API credentials.  
    - Inputs: AI conversation text from "Secretary Agent"  
    - Outputs: Connected to update reservation status  
    - Edge cases:  
      - Invalid phone numbers  
      - Twilio API authentication failures  
      - Call failures (busy lines, no answer)  
      - TwiML syntax errors

---

#### 2.6 Post-Call Update and Wait Block

- **Overview:** Marks the reservation as called in Google Sheets and waits 2 minutes before processing the next call to avoid flooding.
- **Nodes Involved:**  
  - Update reservation  
  - Wait

- **Node Details:**

  - **Update reservation**  
    - Type: Google Sheets (Update operation)  
    - Role: Updates the 'CALLED' column to "x" for the current reservation row to mark it as called.  
    - Configuration:  
      - Uses the row_number from the reservation to update the exact row.  
      - Mapping mode defined with all columns listed, only 'CALLED' changed.  
      - Sheet and document same as data retrieval node.  
    - Inputs: Output from "Make a call"  
    - Outputs: Connected to "Wait" node  
    - Credentials: Google Sheets OAuth2  
    - Edge cases:  
      - Row number mismatch causing wrong row update  
      - Google API errors or permission issues

  - **Wait**  
    - Type: Wait node  
    - Role: Pauses workflow execution for 2 minutes before next iteration.  
    - Configuration: 2 minutes delay  
    - Inputs: From "Update reservation"  
    - Outputs: Loops back to "Loop Over Items" to process next reservation  
    - Edge cases: Workflow timeout if many iterations, unintended long delays if error occurs

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                              | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                               |
|---------------------------|---------------------------------|----------------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Manual workflow start                         | None                         | Get daily reservations       |                                                                                                          |
| Schedule Trigger          | Schedule Trigger                | Automatic daily start at 11:00 AM             | None                         | Get daily reservations       |                                                                                                          |
| Get daily reservations    | Google Sheets (Read)            | Retrieves today's reservations from sheet    | When clicking ‘Execute workflow’, Schedule Trigger | Loop Over Items             |                                                                                                          |
| Loop Over Items           | SplitInBatches                 | Processes reservations one by one in batches | Get daily reservations        | Secretary Agent (via second output), Wait (via first output) |                                                                                                          |
| Secretary Agent           | LangChain Agent (AI Conversation) | Generates AI reminder call script             | Loop Over Items               | Make a call                  |                                                                                                          |
| Grok 4                   | LangChain LM Chat (OpenRouter) | Provides AI language model                     | Secretary Agent              | Secretary Agent              |                                                                                                          |
| Make a call               | Twilio Call                    | Places phone call with AI-generated message  | Secretary Agent              | Update reservation           | ## STEP 1 Register to Twilio and buy a phone number - Set text-to-speech language and geo permissions - Setup Twilio node and sender number |
| Update reservation        | Google Sheets (Update)          | Marks reservation as called in sheet          | Make a call                  | Wait                        |                                                                                                          |
| Wait                      | Wait                          | Waits 2 minutes before next call               | Update reservation           | Loop Over Items             |                                                                                                          |
| Sticky Note               | Sticky Note                   | Provides Step 1 instructions                   | None                        | None                       | ## STEP 1 Register to Twilio and buy a phone number - Set the [text-to-speach](https://console.twilio.com/us1/develop/voice/settings/text-to-speech) language - Set the [geo permissions](https://console.twilio.com/us1/develop/voice/settings/geo-permissions) - Set up the Twilio node - Set up the sender in the field "From" |
| Sticky Note1              | Sticky Note                   | Provides Step 2 instructions                   | None                        | None                       | ## STEP 2 - Clone this [Google Sheet](https://docs.google.com/spreadsheets/d/1lQh-199bQe-HKwmVI_6cS-Xv1OiGo1B4-OpR6kbNg7E/edit?usp=sharing) - Phone field must contain international prefix without "+" |
| Sticky Note2              | Sticky Note                   | Workflow description                           | None                        | None                       | ## AI-Powered Reservation Reminder Calls with Twilio - Automates reminder calls using AI and Twilio. Adaptable to other venues. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’" with default settings.  
   - Add a **Schedule Trigger** node named "Schedule Trigger" configured to trigger daily at 11:00 AM.

2. **Create Google Sheets Node to Retrieve Reservations**  
   - Add a **Google Sheets** node named "Get daily reservations".  
   - Set operation to "Read Rows".  
   - Configure document ID to the Google Sheet containing reservation data.  
   - Set sheet to the first sheet (gid=0).  
   - Apply filters:  
     - 'CALLED' column is empty (no value)  
     - 'DATE' column matches current date (`={{ $now.format('yyyy-LL-dd') }}`).  
   - Connect outputs of both trigger nodes to this node.  
   - Set Google Sheets OAuth2 credentials.

3. **Add SplitInBatches Node to Process Reservations One by One**  
   - Add a **SplitInBatches** node named "Loop Over Items".  
   - Set batch size to 1.  
   - Connect output of "Get daily reservations" to this node.

4. **Add AI Model Node - Grok 4**  
   - Add a **LangChain LM Chat (OpenRouter)** node named "Grok 4".  
   - Select model "x-ai/grok-4".  
   - Configure OpenRouter API credentials.

5. **Add LangChain Agent Node for Secretary Dialogue**  
   - Add a **LangChain Agent** node named "Secretary Agent".  
   - Configure prompt with:  
     - System message describing role as restaurant secretary at "Bella Napoli".  
     - Input text template including reservation details:  
       ```
       Details booking appointment:
       Name: {{ $json.NAME }}
       Time: {{ $json.TIME }}
       N. People: {{ $json['N. PEOPLE'] }}
       ```
   - Enable output parsing to extract plain text message.  
   - Connect "Loop Over Items" output (batch item) to this node input.  
   - Connect output of "Grok 4" to "Secretary Agent" as AI language model.  
   - Connect output of "Secretary Agent" back to "Grok 4" as input (this is implicit in LangChain agent config).

6. **Add Twilio Node to Place Calls**  
   - Add a **Twilio** node named "Make a call".  
   - Set resource to "Call".  
   - Configure "To" number dynamically: `=+{{ $('Get daily reservations').item.json.PHONE }}`  
     (Ensure phone numbers include international prefix without "+".)  
   - Set "From" number with your Twilio purchased phone number (e.g., "+177526xxxxxx").  
   - Enable TwiML option and enter the following XML:  
     ```xml
     <?xml version="1.0" encoding="UTF-8"?>
     <Response>
       <Say voice="alice" language="it-IT">
         {{ $json.output }}
       </Say>
     </Response>
     ```  
   - Connect output of "Secretary Agent" to this node.  
   - Configure Twilio API credentials.

7. **Add Google Sheets Node to Update Reservation as Called**  
   - Add a **Google Sheets** node named "Update reservation".  
   - Set operation to "Update Row".  
   - Configure document and sheet same as "Get daily reservations".  
   - Define mapping to update 'CALLED' column to "x" using the `row_number` field to target the correct row.  
   - Connect output of "Make a call" to this node.  
   - Use Google Sheets OAuth2 credentials.

8. **Add Wait Node to Pause Between Calls**  
   - Add a **Wait** node named "Wait".  
   - Set duration to 2 minutes.  
   - Connect output of "Update reservation" to this node.

9. **Connect Wait Node Back to Loop Over Items**  
   - Connect output of "Wait" node to the second input of "Loop Over Items" node to continue processing next batch item.

10. **Add Sticky Notes for Documentation (Optional)**  
    - Add sticky notes with instructions for:  
      - Step 1: Twilio setup including text-to-speech language and geo permissions.  
      - Step 2: Google Sheet cloning and phone number formatting.  
      - General workflow description.

**Important Credential Setup:**

- **Google Sheets OAuth2:** Configure to allow read/write access to the reservation sheet.  
- **Twilio API:** Register, buy a phone number, configure voice settings, and obtain API credentials.  
- **OpenRouter API:** Obtain API key for Grok-4 language model.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Register to Twilio and buy a phone number. Set the [text-to-speech language](https://console.twilio.com/us1/develop/voice/settings/text-to-speech) and [geo permissions](https://console.twilio.com/us1/develop/voice/settings/geo-permissions). | Twilio account setup instructions                                                                                       |
| Clone the reservation Google Sheet from [here](https://docs.google.com/spreadsheets/d/1lQh-199bQe-HKwmVI_6cS-Xv1OiGo1B4-OpR6kbNg7E/edit?usp=sharing). Ensure the Phone field contains the international prefix without "+".                           | Google Sheets template and phone formatting guidelines                                                                   |
| This workflow automates AI-powered reservation reminder calls with Twilio and can be adapted for other venues requiring automated customer reminders.                                                                             | General workflow description                                                                                             |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.