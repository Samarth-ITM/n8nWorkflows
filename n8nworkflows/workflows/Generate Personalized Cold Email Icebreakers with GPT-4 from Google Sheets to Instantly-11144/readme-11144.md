Generate Personalized Cold Email Icebreakers with GPT-4 from Google Sheets to Instantly

https://n8nworkflows.xyz/workflows/generate-personalized-cold-email-icebreakers-with-gpt-4-from-google-sheets-to-instantly-11144


# Generate Personalized Cold Email Icebreakers with GPT-4 from Google Sheets to Instantly

### 1. Workflow Overview

This workflow automates the process of extracting lead data from a Google Sheets spreadsheet, cleansing and formatting the names, generating personalized cold email icebreakers using GPT-4, and adding these enriched leads into an Instantly email campaign system. It is designed for use cases involving mass cold email outreach campaigns where personalized introductions increase engagement rates.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger and read rows from Google Sheets containing raw lead data.
- **1.2 Data Validation and Filtering:** Check for required lead information completeness and validity.
- **1.3 Data Cleansing:** Format and clean lead names via AI to improve personalization quality.
- **1.4 AI Icebreaker Generation:** Use GPT-4 to generate personalized icebreaker sentences for each lead.
- **1.5 Lead Upload to Instantly:** Add the lead with generated icebreaker to an Instantly campaign.
- **1.6 Loop Control and Notification:** Manage batch processing and notify via Telegram upon workflow completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Starts the workflow manually and retrieves lead data rows from a specific Google Sheets document and sheet.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet (Google Sheets)

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually.  
    - Configuration: Default manual trigger, no parameters.  
    - Connections: Outputs to "Get row(s) in sheet".  
    - Edge Cases: None. Requires user to start workflow.

  - **Get row(s) in sheet**  
    - Type: Google Sheets node  
    - Role: Reads rows from a Google Sheets document (Lead data source).  
    - Configuration:  
      - Document ID: Specific Google Sheet identified.  
      - Sheet Name: "Sheet3" (gid=621890324).  
      - Credentials: Google Sheets OAuth2 account.  
    - Outputs: Rows of lead data with fields such as firstName, lastName, summary, email, company, jobTitle.  
    - Connections: Outputs to "If has all variables" node.  
    - Edge Cases:  
      - Google API authentication failure.  
      - Empty or malformed sheet.  
      - Network/timeouts.  
    - Notes: A sticky note reminds user to ensure Google Sheets API is enabled in Google Cloud Console.

---

#### 2.2 Data Validation and Filtering

- **Overview:**  
  Filters out incomplete or invalid lead records by checking presence and format of critical variables.

- **Nodes Involved:**  
  - If has all variables (If node)  
  - Append or update row in sheet (Google Sheets)

- **Node Details:**  

  - **If has all variables**  
    - Type: If Condition node  
    - Role: Checks that firstName, lastName, summary, and email fields are present and that email matches a valid regex format.  
    - Configuration:  
      - All conditions combined with AND:  
        - firstName not empty  
        - lastName not empty  
        - summary not empty  
        - email not empty and matches email regex pattern.  
    - Connections:  
      - True: to "Append or update row in sheet"  
      - False: no further processing (filtered out).  
    - Edge Cases: Missing or invalid data will cause leads to be skipped.  
    - Notes: Ensures only leads with complete info proceed.

  - **Append or update row in sheet**  
    - Type: Google Sheets node  
    - Role: Adds or updates lead rows in a different sheet ("Law firm test sheet") within the same Google Sheets document, for organization and tracking filtered leads.  
    - Configuration:  
      - Document ID same as input sheet.  
      - Sheet Name: "Law firm test sheet" (gid=1305071092).  
      - Columns mapped: email, summary, lastName, firstName.  
      - Matching column for update: firstName.  
      - Operation: appendOrUpdate.  
      - Credentials: same Google Sheets OAuth2 account.  
    - Connections: Outputs to "Loop Over Items" node for batch processing.  
    - Edge Cases: Google API errors, duplicate first names causing updates rather than appends.  
    - Notes: Sticky note indicates this step organizes filtered leads in a new sheet for clarity.

---

#### 2.3 Loop Control and Batch Processing

- **Overview:**  
  Processes leads in manageable batches to control workflow execution and avoid overloads.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - Limit (Limit node)

- **Node Details:**  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Splits the list of leads into batches, controlling how many are processed simultaneously.  
    - Configuration: Default batch size (not explicitly set, so default applies).  
    - Connections:  
      - Main output to "Limit" and "Format Names".  
    - Edge Cases: Large datasets could still cause delays or API rate limits.  
    - Notes: Sticky note describes looping over each row pulled from the sheet.

  - **Limit**  
    - Type: Limit  
    - Role: Controls the number of outputs passed downstream to prevent flooding subsequent nodes with multiple outputs at once.  
    - Configuration: Default (no limit explicitly set).  
    - Connections: Output to "Notify Master" node.  
    - Edge Cases: Prevents issues with nodes that cannot handle high concurrency or large output batches.

---

#### 2.4 Data Cleansing (Name Formatting)

- **Overview:**  
  Cleans and standardizes first and last names using GPT-4 Mini model to remove unwanted symbols, titles, and punctuation, preserving natural name structures.

- **Nodes Involved:**  
  - Format Names (OpenAI GPT-4 Mini)

- **Node Details:**  

  - **Format Names**  
    - Type: LangChain OpenAI node  
    - Role: Cleans names by removing emojis, professional titles, punctuation after initials, while preserving accented characters and multipart names.  
    - Configuration:  
      - Model: gpt-4o-mini  
      - Max Tokens: 128  
      - System prompt instructs specific rules for name formatting (JSON output with firstName and lastName)  
    - Inputs: Receives each lead item from batch split node.  
    - Outputs: JSON with cleaned firstName and lastName for use in icebreaker generation.  
    - Connections: Outputs to "IBC V3" (icebreaker generator).  
    - Edge Cases: Potential for AI misinterpretation; malformed names might not be perfectly cleaned.  
    - Notes: Sticky note explains importance of formatting names to avoid robotic or unprofessional outputs.

---

#### 2.5 AI Icebreaker Generation

- **Overview:**  
  Uses GPT-4 to generate a unique, engaging, personalized icebreaker sentence for each lead based on cleaned names and lead summary data.

- **Nodes Involved:**  
  - IBC V3 (OpenAI GPT-4)

- **Node Details:**  

  - **IBC V3**  
    - Type: LangChain OpenAI node  
    - Role: Generates one personalized icebreaker sentence per lead using firstName, lastName, companyName, jobTitle, and summary fields.  
    - Configuration:  
      - Model: chatgpt-4o-latest (GPT-4)  
      - Options: topP=1, maxTokens=248, temperature=1 (for creative output)  
      - Messages:  
        - System prompt contains detailed instructions and examples to create a natural, confident, casual icebreaker sentence referencing a specific achievement or detail from the summary.  
        - Output strictly JSON format containing "icebreaker" field.  
    - Inputs: Uses cleaned names from "Format Names" node and additional data from the "Append or update row in sheet" node.  
    - Outputs: JSON with the personalized icebreaker sentence.  
    - Connections: Outputs to "Add leads to instantly" node.  
    - Edge Cases:  
      - AI generation could fail or produce invalid JSON, causing errors downstream.  
      - Rate limiting or API quota issues with OpenAI.  
    - Notes: Sticky note encourages testing and tuning of system prompt to optimize icebreaker quality.

---

#### 2.6 Lead Upload to Instantly and Notification

- **Overview:**  
  Adds each enriched lead with personalized icebreaker to an Instantly campaign via API and sends a Telegram notification upon workflow completion.

- **Nodes Involved:**  
  - Add leads to instantly (HTTP Request)  
  - Notify Master (Telegram)  

- **Node Details:**  

  - **Add leads to instantly**  
    - Type: HTTP Request  
    - Role: Calls Instantly API to add leads to a specified campaign with all required fields and custom variables, including the icebreaker.  
    - Configuration:  
      - URL: https://api.instantly.ai/api/v2/leads (Instatly API endpoint)  
      - Method: POST  
      - Body (JSON):  
        - campaign: [User to replace with actual Instantly campaign ID]  
        - email, first_name, last_name, company_name from respective node outputs  
        - Flags to skip duplicates and verify leads enabled  
        - custom_variables includes icebreaker text generated by AI  
      - Credentials: None indicated (assumed token in header or environment)  
      - Error handling: onError set to continue to avoid workflow stop in case of failure  
    - Connections: On success, loops back to "Loop Over Items" to process next lead batch.  
    - Edge Cases:  
      - API authentication errors if credentials missing or invalid  
      - Network errors, API rate limits  
      - Failure to add lead due to missing mandatory fields or invalid data  
    - Notes: Sticky note references Instantly API docs for bulk add leads and stresses importance of correct custom variable setup.

  - **Notify Master**  
    - Type: Telegram node  
    - Role: Sends a Telegram message notifying user of workflow completion.  
    - Configuration:  
      - Text: Static message "Hello Master, Your n8n Instantly add leads From EMAIL SCRAPE Workflow is completed."  
      - Credentials: Telegram bot credentials with chat configured to receive messages  
    - Connections: Output of "Limit" node triggers this notification.  
    - Edge Cases:  
      - Telegram API errors or misconfigured credentials  
      - Telegram chat/bot not setup properly to receive messages  
    - Notes: Sticky note provides link and guidance for Telegram bot creation and token retrieval.

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                          | Input Node(s)                    | Output Node(s)                 | Sticky Note                                                                                                                                    |
|---------------------------|---------------------------|----------------------------------------|---------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Starts the workflow manually            | -                               | Get row(s) in sheet            |                                                                                                                                                |
| Get row(s) in sheet       | Google Sheets             | Reads raw lead data from Google Sheets | When clicking ‘Execute workflow’ | If has all variables            | ## Read Sheets and Filter Variables<br>This workflow begins with a Google Sheets file that contains your scraped lead data. Checks if it has all the necessary variables to proceed and filters out ones that done.<br>Make sure you connect your google sheets API on google cloud console. |
| If has all variables      | If Condition              | Validates presence and format of fields | Get row(s) in sheet             | Append or update row in sheet  |                                                                                                                                                |
| Append or update row in sheet | Google Sheets             | Appends or updates filtered leads in new sheet | If has all variables (true)     | Loop Over Items                | ## Creating and populating new sheet within your Google sheet<br>We are grabbing all the rows that can be useful to us and attaching it to a new sheet within the same google sheet document we previously used this way we can be more organized |
| Loop Over Items           | SplitInBatches            | Splits leads into batches for processing | Append or update row in sheet   | Limit, Format Names            | ## Loop over each name<br>that we are pulling from pervious node.                                                                                 |
| Limit                     | Limit                     | Controls output rate to avoid flooding | Loop Over Items                 | Notify Master                  | ## Notify<br>When the loop is complete it will send a notification to telegram to notify you that your workflow is complete and you can start using the newly added campaign on instantly.<br>The Limit node is there because sometimes the "loop over items" node would output as much as it has looped which is problematic.<br>Also with Telegram make sure your able to create a seperate chat to receieve these notification there is a link here to create chat bots on the telegram ecosystem.<br>https://core.telegram.org/bots/tutorial#obtain-your-bot-token |
| Notify Master             | Telegram                  | Sends completion notification          | Limit                          | -                             | See Limit node sticky note                                                                                                                      |
| Format Names              | LangChain OpenAI (GPT-4 Mini) | Cleans and standardizes first and last names | Loop Over Items                | IBC V3                        | ## Format Names<br>In this node we focus on cleaning and standardizing the user's first and last name. This step improves the quality of the icebreaker by giving the AI a clean, natural name to work with.<br>Unformatted names look robotic and unprofessional. For example, you do not want the AI generating an opener like:<br>“Hey John E Riley III”<br>Proper formatting helps the icebreaker feel more personal and less automated. |
| IBC V3                   | LangChain OpenAI (GPT-4)  | Generates personalized icebreaker sentences | Format Names                  | Add leads to instantly         | ## Ice Breaker Generator<br>Using the person’s first name, last name, email, bio, summary, and company name, we will generate a personalized icebreaker for cold email outreach or linkedin direct message outreach. and inside the node is the system prompt I use for creating ice breakers Feel free to test differnt prompt for different results of ice breakers. |
| Add leads to instantly    | HTTP Request              | Adds leads with icebreaker to Instantly campaign | IBC V3                        | Loop Over Items               | ## Add leads to a campaign<br>This includes adding all required variables to the campaign. Refer to the documentation for the correct process of adding leads:<br>https://developer.instantly.ai/api/v2/lead/bulkaddleads<br>Provide first name, last name, email, and a description if needed. It’s important to create the custom variables as shown in the documentation, as this is where the icebreaker variable will be stored in Instantly. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create manual trigger node:**  
   - Add "Manual Trigger" node named `When clicking ‘Execute workflow’`.

2. **Add Google Sheets node to read leads:**  
   - Add Google Sheets node named `Get row(s) in sheet`.  
   - Set operation to read rows from sheet "Sheet3" within your Google Sheets document.  
   - Provide Google Sheets OAuth2 credentials.  
   - Connect `When clicking ‘Execute workflow’` output to this node input.

3. **Add "If" node to validate lead data completeness:**  
   - Add "If" node named `If has all variables`.  
   - Configure conditions (AND):  
     - firstName not empty  
     - lastName not empty  
     - summary not empty  
     - email not empty and matches regex pattern for valid email.  
   - Connect output of `Get row(s) in sheet` to this node.  
   - Connect "true" output to next node for filtered leads.

4. **Add Google Sheets node to append or update filtered leads:**  
   - Add Google Sheets node named `Append or update row in sheet`.  
   - Configure to write to sheet "Law firm test sheet" in the same Google Sheets document.  
   - Set operation to append or update using "firstName" as the matching column.  
   - Map columns: firstName, lastName, summary, email.  
   - Use same Google Sheets OAuth2 credentials.  
   - Connect "true" output of `If has all variables` to this node.

5. **Add SplitInBatches node to process leads in batches:**  
   - Add node `Loop Over Items` (SplitInBatches).  
   - Connect output of `Append or update row in sheet` to this node.

6. **Add Limit node to control output rate:**  
   - Add Limit node named `Limit`.  
   - Connect main output of `Loop Over Items` to this node.

7. **Add Telegram node for notification:**  
   - Add Telegram node named `Notify Master`.  
   - Configure with your Telegram Bot credentials and chat ID.  
   - Set message text: "Hello Master, Your n8n Instantly add leads From EMAIL SCRAPE Workflow is completed."  
   - Connect `Limit` node output to `Notify Master`.

8. **Add OpenAI LangChain node for name formatting:**  
   - Add LangChain OpenAI node named `Format Names`.  
   - Use model: GPT-4 Mini (`gpt-4o-mini`).  
   - Set max tokens to 128.  
   - Provide system prompt to clean and format names as per detailed instructions.  
   - Connect second output of `Loop Over Items` node to this node.

9. **Add OpenAI LangChain node for icebreaker generation:**  
   - Add LangChain OpenAI node named `IBC V3`.  
   - Use GPT-4 model (`chatgpt-4o-latest`).  
   - Configure temperature=1, maxTokens=248, topP=1.  
   - Provide system prompt that instructs generating a single, natural, confident icebreaker sentence referencing the lead’s firstName, lastName, company, jobTitle, and summary.  
   - Connect output of `Format Names` to this node.

10. **Add HTTP Request node to add leads to Instantly:**  
    - Add HTTP Request node named `Add leads to instantly`.  
    - Configure POST request to `https://api.instantly.ai/api/v2/leads`.  
    - Set raw JSON body with fields: campaign ID (replace placeholder), email, first_name, last_name, company_name, and custom_variables including the icebreaker text.  
    - Use appropriate authentication (API key/token) as required by Instantly.  
    - Set error handling to continue on error.  
    - Connect output of `IBC V3` to this node.

11. **Connect output of `Add leads to instantly` back to `Loop Over Items` to continue batch processing.**

12. **Ensure all credentials are configured:**  
    - Google Sheets OAuth2 credentials with access to the target spreadsheet.  
    - OpenAI API credentials with access to GPT-4 and GPT-4 Mini models.  
    - Telegram Bot API credentials with chat configured for notifications.  
    - Instantly API credentials or token for lead addition.

13. **Test the workflow manually by triggering the manual start node.**

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|----------------|
| Ensure Google Sheets API is enabled and OAuth2 credentials are valid to avoid authentication issues. | Workflow Google Sheets nodes and sticky note. |
| Telegram notifications require bot creation and chat setup; see https://core.telegram.org/bots/tutorial#obtain-your-bot-token | Notification block sticky note. |
| Instantly API documentation for adding leads: https://developer.instantly.ai/api/v2/lead/bulkaddleads | Add leads to Instantly sticky note. |
| Crafting an effective AI prompt is crucial for quality icebreakers; iterative testing recommended. | Icebreaker Generator sticky note. |
| Proper name formatting improves personalization and prevents robotic-sounding messages. | Format Names sticky note. |
| Workflow designed to handle batch processing to avoid API rate limits and performance issues. | Loop Over Items and Limit nodes notes. |
| The workflow depends on external APIs; network issues or quota limits can cause failures. Logging and error handling recommended. | General operational note. |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.