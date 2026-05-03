Transform Voice Notes into Business Reviews with Groq Whisper & GPT-5 to Google Slides

https://n8nworkflows.xyz/workflows/transform-voice-notes-into-business-reviews-with-groq-whisper---gpt-5-to-google-slides-10869


# Transform Voice Notes into Business Reviews with Groq Whisper & GPT-5 to Google Slides

---

### 1. Workflow Overview

This workflow automates the transformation of client voice notes into polished business review presentations on Google Slides using AI services. It is designed for Customer Success Managers (CSMs) who record voice notes via Telegram during or after client meetings. The workflow transcribes audio using Groq Whisper, checks for sensitive information with Guardrails, structures the content into business review sections using GPT-5, and then generates a Google Slides presentation in the client's shared Google Drive folder.

The workflow logic can be grouped into these functional blocks:

- **1.1 Input Reception and Filtering:** Receives Telegram messages, filters for voice notes only, and downloads the audio file.
- **1.2 Audio Preparation and Transcription:** Converts the audio to .ogg format and transcribes it using Groq Whisper API.
- **1.3 Security and Compliance Check:** Analyzes transcription for sensitive or confidential information using Guardrails and alerts admins if violations are detected.
- **1.4 AI Content Structuring:** Uses GPT-5 models to extract and organize transcription into structured business review sections: Value Realized, Recommendations, and Next Steps.
- **1.5 Review and Client Data Collection:** Sends the structured content to the CSM via email, requesting client-specific information (name, date, Google Drive folder) through a form.
- **1.6 Google Drive Validation and Presentation Setup:** Validates the provided Google Drive folder, generates a presentation name, copies a template file to the client folder.
- **1.7 Slide Population:** Replaces placeholders in the copied Google Slides presentation with AI-generated content.
- **1.8 Error Handling:** Detects errors such as invalid Drive links and sends alert emails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Filtering

**Overview:**  
This block triggers on Telegram messages, filters out non-voice messages, and downloads the voice note audio file.

**Nodes Involved:**  
- Telegram Trigger  
- If (voice note check)  
- No Operation, do nothing  
- Get a file  
- Need .ogg for next node  

**Node Details:**  

- **Telegram Trigger**  
  - *Type:* Trigger node for Telegram updates.  
  - *Config:* Watches for message updates on a Telegram bot.  
  - *Expressions:* None.  
  - *Connections:* Outputs to If node.  
  - *Failures:* Telegram API errors, webhook issues.  
  - *Credentials:* Telegram API OAuth2.  

- **If**  
  - *Type:* Conditional filter node.  
  - *Config:* Checks if incoming message has a voice file_id property (voice note presence).  
  - *Expressions:* `={{ $json.message.voice.file_id }}` exists.  
  - *Connections:* True → Get a file; False → No Operation, do nothing.  
  - *Failures:* Expression evaluation errors if JSON structure changes.  

- **No Operation, do nothing**  
  - *Type:* No-op placeholder for non-voice messages.  
  - *Config:* No parameters.  
  - *Connections:* None.  
  - *Failures:* None.  

- **Get a file**  
  - *Type:* Telegram file download.  
  - *Config:* Downloads voice note using `file_id` from incoming message.  
  - *Expressions:* Uses `$json.message.voice.file_id`.  
  - *Connections:* Outputs to Need .ogg for next node.  
  - *Failures:* Telegram file download errors.  

- **Need .ogg for next node**  
  - *Type:* Code node for binary data processing.  
  - *Config:* Renames downloaded binary file to `voice.ogg` and sets MIME type to `audio/ogg`.  
  - *Expressions:* JavaScript code loops over items, adjusts binary metadata.  
  - *Connections:* Outputs to transcription node.  
  - *Failures:* Binary data absent or malformed input.  

---

#### 1.2 Audio Preparation and Transcription

**Overview:**  
Prepares audio file and sends it to Groq Whisper API for transcription.

**Nodes Involved:**  
- Get cost effective transcript from Groq  

**Node Details:**  

- **Get cost effective transcript from Groq**  
  - *Type:* HTTP Request node.  
  - *Config:* POST multipart-form to Groq Whisper transcription endpoint with `voice.ogg` binary data. Model used is `whisper-large-v3-turbo`.  
  - *Expressions:* Uses binary data from previous node.  
  - *Connections:* Outputs transcription text JSON to Guardrails node.  
  - *Failures:* Network errors, API errors, rate limits, invalid audio format.  
  - *Credentials:* Groq API key required.  

---

#### 1.3 Security and Compliance Check

**Overview:**  
Checks transcription text for PII and sensitive business data using Guardrails AI node and sends alert emails if violations occur.

**Nodes Involved:**  
- Guardrails  
- Send a message (email alert)  
- Sticky Note4 (comment explaining)  

**Node Details:**  

- **Guardrails**  
  - *Type:* LangChain guardrails node for content validation.  
  - *Config:* Custom guardrail for sensitive info detection with threshold 0.7. Also includes PII and NSFW checks. Input is transcription text from Groq.  
  - *Expressions:* `={{ $json.text }}` contains transcription.  
  - *Connections:* On success → Classy McClassface (AI structuring); On violation → Send a message (email alert).  
  - *Failures:* API outages, false positives/negatives in detection.  

- **Send a message**  
  - *Type:* Gmail node sending alert email.  
  - *Config:* Sends email to admin@example.com with details of the guardrail violation including confidence score and input text.  
  - *Expressions:* Uses guardrail JSON output for message body and subject.  
  - *Connections:* None (terminal for violation path).  
  - *Failures:* Gmail API auth errors, email delivery failures.  
  - *Credentials:* Gmail OAuth2 required.  

- **Sticky Note4**  
  - *Type:* Sticky Note for documentation.  
  - *Content:* Explains that this block notifies admin about guardrail violations.  

---

#### 1.4 AI Content Structuring

**Overview:**  
Uses GPT-5 models to parse and structure the transcription into three business review sections: Value Realized, Recommendations, Next Steps.

**Nodes Involved:**  
- Cost effective model - GPT5 nano  
- Cost effective - GPT5 mini  
- Structured Output Parser  
- Classy McClassface  

**Node Details:**  

- **Cost effective model - GPT5 nano**  
  - *Type:* LangChain OpenAI chat model node.  
  - *Config:* Uses GPT-5 nano model as cost-effective initial step.  
  - *Connections:* Outputs to Guardrails node (earlier in chain) — here probably used for guardrail? Confirmed as input to Guardrails.  

- **Cost effective - GPT5 mini**  
  - *Type:* LangChain OpenAI chat model node.  
  - *Config:* Uses GPT-5 mini for better output quality.  
  - *Connections:* Outputs to Classy McClassface for final structuring.  

- **Structured Output Parser**  
  - *Type:* LangChain output parser node.  
  - *Config:* Defines strict JSON schema requiring three properties: `value_realization`, `recommendations`, `next_steps`.  
  - *Connections:* Attached as output parser for Classy McClassface node.  
  - *Failures:* Schema validation failures if AI response malforms JSON.  

- **Classy McClassface**  
  - *Type:* LangChain agent node (AI assistant).  
  - *Config:* Receives transcription text, instructs to reorganize into three clear sections, rephrase for clarity and professionalism, and return exactly the required JSON structure.  
  - *Expressions:* Prompt includes transcription input via `{{ $json.guardrailsInput }}`.  
  - *Connections:* Outputs structured JSON to "Ask CSM to review" email node.  
  - *Failures:* AI model errors, prompt issues, output format errors.  
  - *Version:* Requires LangChain integration and GPT-5 model access.  

---

#### 1.5 Review and Client Data Collection

**Overview:**  
Sends the AI-structured content via Gmail email to the CSM, requesting client info through a custom form. Pauses workflow until form submission.

**Nodes Involved:**  
- Ask CSM to review and provide customer infos  

**Node Details:**  

- **Ask CSM to review and provide customer infos**  
  - *Type:* Gmail node with send-and-wait and custom form response.  
  - *Config:* Email includes summarized transcription sections and requests client name, business review date, and Google Drive folder link.  
  - *Expressions:* Inserts AI output sections dynamically.  
  - *Connections:* Outputs to Check Google Drive Link, Set CSM's company name, and Convert date format nodes after form submission.  
  - *Failures:* Email delivery failure, user not responding, malformed form input.  
  - *Credentials:* Gmail OAuth2.  

---

#### 1.6 Google Drive Validation and Presentation Setup

**Overview:**  
Validates client-provided Google Drive folder link, aggregates client name and date, constructs presentation name, copies template to client folder.

**Nodes Involved:**  
- Check Google Drive Link  
- Valid Google drive ID  
- Set CSM's company name  
- Convert date format  
- Merge Company name + date  
- Aggregate Date + Name  
- Create Presentation name  
- Merge Shared Folder + Presentation Name  
- Aggregate Shared Folder + Presentation Name  
- Copy template to customer Folder  

**Node Details:**  

- **Check Google Drive Link**  
  - *Type:* Google Drive node.  
  - *Config:* Checks existence of folder by folder ID from client form input. On error continues but routes to error email.  
  - *Connections:* Valid ID → Valid Google drive ID; Error → Sends error email.  
  - *Failures:* Invalid folder link, permission denied.  
  - *Credentials:* Google Drive OAuth2.  

- **Valid Google drive ID**  
  - *Type:* Set node.  
  - *Config:* Saves validated Drive folder link from form input for downstream use.  
  - *Connections:* To Merge Shared Folder + Presentation Name.  

- **Set CSM's company name**  
  - *Type:* Set node.  
  - *Config:* Stores CSM's company name and customer name from form data.  
  - *Connections:* To Merge Company name + date.  

- **Convert date format**  
  - *Type:* DateTime node.  
  - *Config:* Formats business review date from form input to `y LLL d` (e.g., 2025 Nov 15).  
  - *Connections:* To Merge Company name + date.  

- **Merge Company name + date**  
  - *Type:* Merge node (combine data).  
  - *Config:* Merges outputs from Set CSM's company name and Convert date format.  
  - *Connections:* To Aggregate Date + Name.  

- **Aggregate Date + Name**  
  - *Type:* Aggregate node.  
  - *Config:* Aggregates all merged data into one item.  
  - *Connections:* To Create Presentation name.  

- **Create Presentation name**  
  - *Type:* Set node.  
  - *Config:* Builds presentation name string combining formatted date, CSM company name, and client name.  
  - *Connections:* To Merge Shared Folder + Presentation Name.  

- **Merge Shared Folder + Presentation Name**  
  - *Type:* Merge node.  
  - *Config:* Merges valid Google Drive folder data and presentation name.  
  - *Connections:* To Aggregate Shared Folder + Presentation Name.  

- **Aggregate Shared Folder + Presentation Name**  
  - *Type:* Aggregate node.  
  - *Config:* Aggregates merged folder and name data.  
  - *Connections:* To Copy template to customer Folder.  

- **Copy template to customer Folder**  
  - *Type:* Google Drive node.  
  - *Config:* Copies a predefined Google Slides template file into client's validated folder. The copied file is renamed with the generated presentation name.  
  - *Connections:* Outputs to slide population nodes.  
  - *Failures:* Permission errors, invalid folder ID, template file missing.  
  - *Credentials:* Google Drive OAuth2.  

---

#### 1.7 Slide Population

**Overview:**  
Fills Google Slides template placeholders with AI-generated structured content for Value Realized, Recommendations, and Next Steps in parallel.

**Nodes Involved:**  
- Fill-in the Value Realization slide  
- Fill-in the Recommendations slide  
- Fill-in the Next Steps slide  

**Node Details:**  

- **Fill-in the Value Realization slide**  
  - *Type:* Google Slides node.  
  - *Config:* Replaces placeholder `value_realized_placeholder` with AI output `value_realization`.  
  - *Expressions:* Uses `$('Classy McClassface').item.json.output.value_realization`.  
  - *Failures:* API permission issues, presentation not found.  
  - *Credentials:* Google Slides OAuth2.  

- **Fill-in the Recommendations slide**  
  - *Type:* Google Slides node.  
  - *Config:* Replaces placeholder `recommendations_placeholder` with AI output `recommendations`.  

- **Fill-in the Next Steps slide**  
  - *Type:* Google Slides node.  
  - *Config:* Replaces placeholder `next_steps_placeholder` with AI output `next_steps`.  

---

#### 1.8 Error Handling

**Overview:**  
Handles errors such as invalid Google Drive links by sending alert emails to the admin.

**Nodes Involved:**  
- Sends error if Google Drive link invalid  
- Sticky Note1 (comment on error handling)  

**Node Details:**  

- **Sends error if Google Drive link invalid**  
  - *Type:* Gmail node.  
  - *Config:* Sends email to admin@example.com with the error details if the Drive folder link is invalid.  
  - *Failures:* Email delivery issues.  
  - *Credentials:* Gmail OAuth2.  

- **Sticky Note1**  
  - *Type:* Sticky Note.  
  - *Content:* Explains error handling for Google Drive link invalidation.  

---

### 3. Summary Table

| Node Name                             | Node Type                           | Functional Role                                  | Input Node(s)                          | Output Node(s)                                           | Sticky Note                                                                                                   |
|-------------------------------------|-----------------------------------|-------------------------------------------------|--------------------------------------|---------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger                    | Telegram Trigger                  | Entry point, listens for Telegram messages      | -                                    | If                                                      | See Sticky Note6 for Telegram input description                                                             |
| If                                 | If                               | Filters messages to only voice notes             | Telegram Trigger                     | Get a file (true), No Operation (false)                 | See Sticky Note6                                                                                              |
| No Operation, do nothing            | No Operation                     | Discards non-voice messages                       | If (false branch)                    | -                                                       | See Sticky Note5 about discarding non-audio messages                                                        |
| Get a file                         | Telegram                         | Downloads voice note audio                        | If (true branch)                    | Need .ogg for next node                                  | See Sticky Note7 for transcription context                                                                  |
| Need .ogg for next node             | Code                            | Converts audio file metadata to .ogg format      | Get a file                         | Get cost effective transcript from Groq                  | See Sticky Note7                                                                                              |
| Get cost effective transcript from Groq | HTTP Request                    | Transcribes audio via Groq Whisper API            | Need .ogg for next node             | Guardrails                                              | See Sticky Note7                                                                                              |
| Guardrails                        | LangChain Guardrails            | Checks transcription for sensitive info          | Get cost effective transcript from Groq | Classy McClassface (pass), Send a message (violation) | See Sticky Note4 for guardrail violation detection                                                           |
| Send a message                    | Gmail                           | Sends alert email on guardrail violation          | Guardrails (violation output)       | -                                                       | See Sticky Note4                                                                                              |
| Cost effective model - GPT5 nano    | LangChain OpenAI Chat           | Cost effective GPT-5 nano model for guardrails   | Get cost effective transcript from Groq | Guardrails                                             |                                                                                                              |
| Cost effective - GPT5 mini           | LangChain OpenAI Chat           | GPT-5 mini model for content structuring          | Guardrails (pass output)            | Classy McClassface                                      | See Sticky Note7                                                                                              |
| Structured Output Parser            | LangChain Output Parser         | Parses GPT-5 output into structured JSON          | Classy McClassface (output parser)  | Classy McClassface (final output)                       |                                                                                                              |
| Classy McClassface                 | LangChain Agent                 | AI assistant that structures transcription        | Guardrails (pass output), Cost effective - GPT5 mini | Ask CSM to review and provide customer infos          |                                                                                                              |
| Ask CSM to review and provide customer infos | Gmail                       | Sends structured content to CSM, collects client info | Classy McClassface                 | Check Google Drive Link, Set CSM's company name, Convert date format | See Sticky Note                                                                                               |
| Check Google Drive Link             | Google Drive                    | Validates client Google Drive folder link         | Ask CSM to review and provide customer infos | Valid Google drive ID (valid), Sends error if Google Drive link invalid (error) | See Sticky Note1 about error handling                                                                         |
| Valid Google drive ID               | Set                            | Stores validated folder link                       | Check Google Drive Link             | Merge Shared Folder + Presentation Name                  |                                                                                                              |
| Set CSM's company name              | Set                            | Sets CSM company and client names                  | Ask CSM to review and provide customer infos | Merge Company name + date                               |                                                                                                              |
| Convert date format                | DateTime                       | Formats business review date                        | Ask CSM to review and provide customer infos | Merge Company name + date                               |                                                                                                              |
| Merge Company name + date           | Merge                          | Merges company name and formatted date             | Set CSM's company name, Convert date format | Aggregate Date + Name                                  |                                                                                                              |
| Aggregate Date + Name               | Aggregate                      | Aggregates company name and date data               | Merge Company name + date           | Create Presentation name                                 |                                                                                                              |
| Create Presentation name            | Set                            | Creates presentation filename string                | Aggregate Date + Name               | Merge Shared Folder + Presentation Name                  |                                                                                                              |
| Merge Shared Folder + Presentation Name | Merge                          | Merges folder link and presentation name           | Valid Google drive ID, Create Presentation name | Aggregate Shared Folder + Presentation Name            |                                                                                                              |
| Aggregate Shared Folder + Presentation Name | Aggregate                      | Aggregates folder & name info                        | Merge Shared Folder + Presentation Name | Copy template to customer Folder                        |                                                                                                              |
| Copy template to customer Folder   | Google Drive                   | Copies Google Slides template to client folder      | Aggregate Shared Folder + Presentation Name | Fill-in the Value Realization slide, Fill-in the Recommendations slide, Fill-in the Next Steps slide | See Sticky Note8 for slide population description                                                             |
| Fill-in the Value Realization slide | Google Slides                 | Replaces Value Realized placeholder                  | Copy template to customer Folder   | -                                                       |                                                                                                              |
| Fill-in the Recommendations slide  | Google Slides                 | Replaces Recommendations placeholder                 | Copy template to customer Folder   | -                                                       |                                                                                                              |
| Fill-in the Next Steps slide        | Google Slides                 | Replaces Next Steps placeholder                       | Copy template to customer Folder   | -                                                       |                                                                                                              |
| Sends error if Google Drive link invalid | Gmail                           | Sends email if Google Drive link is invalid         | Check Google Drive Link (error)    | -                                                       | See Sticky Note1 for error handling                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API credentials.  
   - Set to listen for message updates only.  

2. **Add If node to filter voice notes:**  
   - Condition: Check if `message.voice.file_id` exists.  
   - True branch leads to downloading the file; False branch leads to No Operation node.  

3. **Add No Operation node:**  
   - Used for discarding non-voice messages.  

4. **Add Telegram node to download voice file:**  
   - Use `file_id` from Telegram message voice object.  
   - Credentials: Telegram API OAuth2.  

5. **Add Code node to rename audio binary:**  
   - JavaScript code to rename binary file to `voice.ogg` and set MIME type to `audio/ogg`.  

6. **Add HTTP Request node to call Groq Whisper API:**  
   - POST to `https://api.groq.com/openai/v1/audio/transcriptions`.  
   - Send `voice.ogg` as multipart-form data parameter `file`.  
   - Model: `whisper-large-v3-turbo`.  
   - Credentials: Groq API key.  

7. **Add LangChain Guardrails node:**  
   - Configure with PII and custom guardrails for sensitive info detection.  
   - Set threshold to 0.7.  
   - Input: transcription text from Groq node.  
   - On success: connect to GPT-5 mini node; on violation: connect to alert email node.  

8. **Add Gmail node to send alert email on guardrail violation:**  
   - Send to admin@example.com with violation details.  
   - Credentials: Gmail OAuth2.  

9. **Add GPT-5 mini LangChain chat node:**  
   - Use model `gpt-5-mini`.  
   - Input: transcription text from Guardrails node if passed.  

10. **Add LangChain Agent node named “Classy McClassface”:**  
    - Prompt: Instructions to reorganize transcription into JSON with keys `value_realization`, `recommendations`, `next_steps`.  
    - Output parser: Attach Structured Output Parser node.  

11. **Add Structured Output Parser node:**  
    - Define manual JSON schema requiring the three keys above.  

12. **Add Gmail node “Ask CSM to review and provide customer infos”:**  
    - Email includes AI structured content summary.  
    - Configure form fields: Customer Name (string), Date of business review (date), Customer Google Drive folder (string URL).  
    - Use Send and Wait mode.  
    - Credentials: Gmail OAuth2.  

13. **Add Google Drive node “Check Google Drive Link”:**  
    - Validate the folder ID from form submission.  
    - On success: proceed; on error: send error email.  
    - Credentials: Google Drive OAuth2.  

14. **Add Gmail node to send error email if Drive link invalid:**  
    - Send to admin@example.com with error details.  

15. **Add Set nodes:**  
    - “Valid Google drive ID”: Store validated folder link URL.  
    - “Set CSM’s company name”: Store company name and customer name from form.  

16. **Add DateTime node to convert date format:**  
    - Format date from form input to `y LLL d` (e.g., 2025 Nov 15).  

17. **Add Merge and Aggregate nodes:**  
    - Merge company name and date.  
    - Aggregate merged data.  
    - Create presentation name combining date, CSM company, and customer name.  
    - Merge with validated Google Drive folder info.  
    - Aggregate for final use.  

18. **Add Google Drive node to copy template:**  
    - Copy your predefined Google Slides template file to the validated folder.  
    - Rename copy with generated presentation name.  

19. **Add three Google Slides nodes to fill placeholders:**  
    - Replace `value_realized_placeholder` with AI output value realization.  
    - Replace `recommendations_placeholder` with AI output recommendations.  
    - Replace `next_steps_placeholder` with AI output next steps.  
    - Credentials: Google Slides OAuth2.  

20. **Add Sticky Notes throughout the workflow for documentation:**  
    - Include notes on input filtering, transcription, guardrails, email review, error handling, and slide population.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| 🎙️ Voice-to-Slides: Business Review Kickstarter - Turn your messy voice notes into structured Google Slides draft. Record insights via Telegram, AI transcribes and structures content, you review and provide client info.        | See Sticky Note5 in workflow for detailed explanation.                                                         |
| Setup instructions include creating a Telegram bot via @BotFather, preparing a Google Slides template with exact placeholders: `value_realized_placeholder`, `recommendations_placeholder`, `next_steps_placeholder`.             | Sticky Note5                                                                                                    |
| Guardrail sensitivity is adjustable; default threshold is 0.7. May produce false positives. Can be removed if confident in input.                                                                                             | Sticky Note5                                                                                                    |
| Voice note tips: Keep under 5 minutes for best transcription quality.                                                                                                                     | Sticky Note5                                                                                                    |
| Business review template expected structure and a sample output image are included as references for users to design their slides accordingly.                                                                                 | Sticky Note19 and Sticky Note20 include images detailing template and output structure.                         |
| Workflow author and contact info: emir.belkahia@gmail.com, LinkedIn: [linkedin.com/in/emirbelkahia](https://www.linkedin.com/in/emirbelkahia/)                                                                               | Sticky Note12                                                                                                   |
| Video walkthrough available at YouTube: [umirp2J85RM](https://youtube.com/umirp2J85RM)                                                                                                                                          | Sticky Note9                                                                                                    |

---

**Disclaimer:**  
The text analyzed and documented here derives exclusively from an automated workflow built with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected elements; all data processed is legal and public.

---