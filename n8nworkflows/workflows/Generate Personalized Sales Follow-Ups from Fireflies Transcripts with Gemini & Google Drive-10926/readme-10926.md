Generate Personalized Sales Follow-Ups from Fireflies Transcripts with Gemini & Google Drive

https://n8nworkflows.xyz/workflows/generate-personalized-sales-follow-ups-from-fireflies-transcripts-with-gemini---google-drive-10926


# Generate Personalized Sales Follow-Ups from Fireflies Transcripts with Gemini & Google Drive

### 1. Workflow Overview

This workflow automates the generation of personalized sales follow-up messages based on meeting transcripts from Fireflies.ai, triggered by new Google Calendar events. It targets sales or client engagement teams who want to deepen client relationships by sending highly tailored email follow-ups derived from actual meeting content. The workflow integrates Google Calendar, Fireflies.ai GraphQL API, Google Gemini AI language model, a Langchain AI Agent, and Google Drive for storage.

Logical blocks:

- **1.1 Input Reception:** Trigger on new Google Calendar event creation and extract relevant appointment data.
- **1.2 Transcript Retrieval:** Query Fireflies.ai GraphQL API to fetch meeting transcripts for the guest participant.
- **1.3 Transcript Validation:** Filter out cases where no transcript data is available to avoid unnecessary processing.
- **1.4 AI Follow-Up Generation:** Use an AI Agent combined with Google Gemini model to generate 12 personalized sales follow-up messages based on the transcript.
- **1.5 Storage:** Save the generated messages as a text file in a dedicated Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow when a new Google Calendar event is created and extracts key appointment metadata for downstream use.

**Nodes Involved:**  
- Google Calendar Trigger  
- Edit Fields  
- Sticky Note (annotating this block)

**Node Details:**

- **Google Calendar Trigger**  
  - Type: Trigger node for Google Calendar events  
  - Config: Polls every minute for newly created events in a specific calendar (email: bilalnaseer01@gmail.com)  
  - Inputs: External trigger from Google Calendar  
  - Outputs: Emits event data including start/end time, creator, and attendees  
  - Edge cases: Calendar API auth failure, event data missing expected fields, rate limits  
  - Credentials: Google Calendar OAuth2  

- **Edit Fields**  
  - Type: Set node to map and rename fields  
  - Config: Extracts and renames start time, end time, creator email, and guest email from the calendar event JSON to standardized fields (`startTimeOfAppointment`, `endTimeOfAppointment`, `createdBy`, and `guest`)  
  - Inputs: From Google Calendar Trigger  
  - Outputs: JSON with simplified, uniform event data for next nodes  
  - Edge cases: Missing or malformed attendee or creator data could cause expression failures  

- **Sticky Note (1)**  
  - Content: "## 1. Get Appointment Data\n\nTriggers when a appointment is created in Google Calendar"  
  - Purpose: Documentation aid

---

#### 2.2 Transcript Retrieval

**Overview:**  
This block queries Fireflies.ai GraphQL API using the guest’s email to retrieve meeting transcripts, including detailed sentences and summaries.

**Nodes Involved:**  
- GraphQL  
- Sticky Note1

**Node Details:**

- **GraphQL**  
  - Type: GraphQL query node  
  - Config:  
    - Endpoint: `https://api.fireflies.ai/graphql`  
    - Query: Fetches transcripts for a given guest email, retrieving id, title, date, organizer, participants, duration, sentences (text, speaker, start/end time), and summary (overview, action items, keywords)  
    - Variables: Dynamically set guestEmail from `guest` field in JSON (from Edit Fields node)  
    - Authentication: Header authentication with stored credentials  
  - Inputs: From Edit Fields node (providing guest email)  
  - Outputs: Transcript data array under `data.transcripts`  
  - Edge cases: API auth failure, empty or missing transcript data, malformed responses, rate limiting  

- **Sticky Note1**  
  - Content: "## 2. Fetch Meeting Transcript\n\nQueries the Fireflies API using the guest's email to fetch the full meeting transcript and then formats it into a clean structure for the AI to process."  

---

#### 2.3 Transcript Validation

**Overview:**  
This block checks if any transcripts were returned; if none, the workflow halts to avoid unnecessary AI processing.

**Nodes Involved:**  
- Filter  
- Sticky Note3

**Node Details:**

- **Filter**  
  - Type: Filter node  
  - Config: Conditions check that `data.transcripts` is not empty  
  - Inputs: From GraphQL node (transcript data)  
  - Outputs: Only proceeds if transcripts exist  
  - Edge cases: If transcript array is empty or missing, downstream nodes do not execute  
  - Version: Uses expression version 2.2 for condition evaluation  

- **Sticky Note3**  
  - Content: "## 3. Skip if Empty\n\nIf there is no meeting with the guest earlier it will not proceed"  

---

#### 2.4 AI Follow-Up Generation

**Overview:**  
This block uses a Langchain AI Agent powered by Google Gemini to generate 12 highly personalized follow-up messages based on the meeting transcript.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Sticky Note2

**Node Details:**

- **AI Agent**  
  - Type: Langchain AI Agent node  
  - Config:  
    - Prompt: Instructions to generate 12 distinct, persuasive follow-up messages referencing specific pain points and topics from the Fireflies transcript  
    - Uses expression to insert guest email and transcript JSON string dynamically from previous nodes (`Edit Fields` and `GraphQL`)  
  - Inputs: From Filter node (ensuring transcript exists)  
  - Outputs: Raw text output of 12 messages  
  - Edge cases: AI generation timeouts, malformed transcript input causing prompt failures, token limits  
  - Sub-workflow: Integrates with Google Gemini Chat Model as language model backend  

- **Google Gemini Chat Model**  
  - Type: AI language model node (Google Gemini 2.5 Flash Lite)  
  - Config: Model name fixed as "models/gemini-2.5-flash-lite"  
  - Credentials: Google PaLM API OAuth2  
  - Inputs: Connected as AI language model backend to AI Agent  
  - Edge cases: API limits, auth failures, latency  

- **Sticky Note2**  
  - Content: "## 4. Generate & Store AI Messages\n\nThe AI Agent uses the transcript to generate 12 personalized follow-up messages. Finally, these messages are saved to a specific Google Drive folder."  

---

#### 2.5 Storage

**Overview:**  
The generated follow-up messages are stored as a text file in a dedicated Google Drive folder named "Followup Messages," using the guest’s email as the filename.

**Nodes Involved:**  
- Google Drive

**Node Details:**

- **Google Drive**  
  - Type: Google Drive node  
  - Config:  
    - Operation: Create file from text content  
    - Filename: Set dynamically to guest email  
    - Content: Raw AI Agent output text (12 messages)  
    - Drive: "My Drive"  
    - Folder: Hardcoded folder ID (corresponds to "Followup Messages")  
  - Inputs: From AI Agent node output  
  - Outputs: Confirmation of file creation  
  - Edge cases: OAuth token expiration, folder permissions, quota limits  
  - Credentials: Google Drive OAuth2  

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                         | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                   |
|-----------------------|----------------------------------|---------------------------------------|------------------------|-----------------------|-----------------------------------------------------------------------------------------------|
| Google Calendar Trigger| Google Calendar Trigger           | Trigger on new calendar event          | -                      | Edit Fields            | ## 1. Get Appointment Data\n\nTriggers when a appointment is created in Google Calendar       |
| Edit Fields           | Set                              | Extract and rename appointment details | Google Calendar Trigger | GraphQL                |                                                                                               |
| GraphQL               | GraphQL Query                    | Query Fireflies transcripts             | Edit Fields             | Filter                 | ## 2. Fetch Meeting Transcript\n\nQueries the Fireflies API using the guest's email           |
| Filter                | Filter                           | Skip if no transcript data              | GraphQL                 | AI Agent               | ## 3. Skip if Empty\n\nIf there is no meeting with the guest earlier it will not proceed      |
| AI Agent              | Langchain AI Agent               | Generate 12 personalized follow-ups    | Filter                  | Google Drive           | ## 4. Generate & Store AI Messages\n\nGenerates 12 personalized messages and saves to Drive   |
| Google Gemini Chat Model| AI Language Model (Google Gemini)| Backend language model for AI Agent    | - (linked internally)   | AI Agent               |                                                                                               |
| Google Drive          | Google Drive                     | Save messages as text file              | AI Agent                | -                      |                                                                                               |
| Sticky Note           | Sticky Note                     | Documentation                          | -                      | -                      | ## 1. Get Appointment Data\n\nTriggers when a appointment is created in Google Calendar       |
| Sticky Note1          | Sticky Note                     | Documentation                          | -                      | -                      | ## 2. Fetch Meeting Transcript\n\nQueries the Fireflies API using the guest's email           |
| Sticky Note3          | Sticky Note                     | Documentation                          | -                      | -                      | ## 3. Skip if Empty\n\nIf there is no meeting with the guest earlier it will not proceed      |
| Sticky Note2          | Sticky Note                     | Documentation                          | -                      | -                      | ## 4. Generate & Store AI Messages\n\nThe AI Agent uses the transcript to generate messages   |
| Sticky Note4          | Sticky Note                     | Documentation                          | -                      | -                      | ## Video Tutorial\n\n** Detailed Youtube Tutorial **. @[youtube](5t9xXCz4DzM)                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger**  
   - Type: Google Calendar Trigger  
   - Credentials: Connect your Google Calendar OAuth2 account with event read permissions  
   - Configuration:  
     - Poll interval: Every minute  
     - Trigger on: Event Created  
     - Calendar: Select target calendar (e.g., your email calendar)  

2. **Create Edit Fields Node**  
   - Type: Set  
   - Connect input from Google Calendar Trigger  
   - Add fields with expressions:  
     - `startTimeOfAppointment` = `{{$json["start"]["dateTime"]}}`  
     - `endTimeOfAppointment` = `{{$json["end"]["dateTime"]}}`  
     - `createdBy` = `{{$json["creator"]["email"]}}`  
     - `guest` = `{{$json["attendees"][0]["email"]}}`  

3. **Create GraphQL Node**  
   - Type: GraphQL  
   - Connect input from Edit Fields node  
   - Configuration:  
     - Endpoint: `https://api.fireflies.ai/graphql`  
     - Query (copy exactly):  
       ```graphql
       query GetTranscriptsByGuest($guestEmail: String!) {
         transcripts(participant_email: $guestEmail) {
           id
           title
           date
           organizer_email
           participants
           duration
           sentences {
             text
             speaker_name
             start_time
             end_time
           }
           summary {
             overview
             action_items
             keywords
           }
         }
       }
       ```  
     - Variables:  
       ```json
       {"guestEmail": "={{ $json.guest }}"}
       ```  
     - Authentication: Header Auth  
       - Set up HTTP Header Auth credentials with Fireflies API key  

4. **Create Filter Node**  
   - Type: Filter  
   - Connect input from GraphQL node  
   - Condition:  
     - Check that `$json.data.transcripts` is not empty (use array operation "not empty")  
   - If true, allow workflow to proceed; else stop  

5. **Create Google Gemini Chat Model Node**  
   - Type: AI Language Model (Google Gemini)  
   - Credentials: Link Google PaLM API OAuth2 credentials  
   - Set model name: `models/gemini-2.5-flash-lite`  

6. **Create AI Agent Node**  
   - Type: Langchain AI Agent  
   - Connect main input from Filter node  
   - Connect AI language model input to Google Gemini Chat Model node  
   - Prompt configuration (use "define" prompt type) with text:  
     ```
     Your Objective:

     Based strictly on the provided transcript, write a sequence of 12 persuasive follow-up messages to be sent to {{ $('Edit Fields').item.json.guest }} via email. The goal is to re-engage the client, address their specific concerns mentioned in the call, and gently guide them towards making a purchase.

     Instructions:

     Deep Personalization: Do not write generic messages.

     Your messages must reference specific topics, pain points, or interests that mentioned during the call. Use the exact context from the transcript.

     TRANSCRIPT: {{ $('GraphQL').item.json.data.transcripts[0].toJsonString() }}

     Core Task: Generate exactly 12 distinct follow-up messages.
     Final Output Format:
     Number each message clearly (Message 1, Message 2, etc.).
     Do not add any extra explanations, introductions, or comments.
     Provide only the raw text of the 12 messages as the final output.
     ```  

7. **Create Google Drive Node**  
   - Type: Google Drive  
   - Connect input from AI Agent node  
   - Credentials: Google Drive OAuth2  
   - Configuration:  
     - Operation: Create file from text  
     - Name: `={{ $('Edit Fields').item.json.guest }}` (use guest email as filename)  
     - Content: `={{ $json.output }}` (AI Agent output)  
     - Drive: My Drive  
     - Folder: Select or create folder "Followup Messages" and use its folder ID  

8. **Link nodes in order:**  
   Google Calendar Trigger → Edit Fields → GraphQL → Filter → AI Agent (with Google Gemini model linked) → Google Drive  

9. **Optionally add Sticky Notes for documentation** with relevant descriptions at each block for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Detailed YouTube Tutorial available for this workflow: **[YouTube Video](https://youtu.be/5t9xXCz4DzM)**       | Video tutorial linked in Sticky Note4            |
| Fireflies.ai GraphQL API documentation: https://docs.fireflies.ai/graphql                                    | Useful for custom transcript queries             |
| Google Gemini (PaLM) API guide: https://developers.generativeai.google/tutorials                               | Reference for AI language model configuration    |
| Google Drive API quotas and limits: https://developers.google.com/drive/api/v3/quotas                          | Important to monitor for production use           |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with valid content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.