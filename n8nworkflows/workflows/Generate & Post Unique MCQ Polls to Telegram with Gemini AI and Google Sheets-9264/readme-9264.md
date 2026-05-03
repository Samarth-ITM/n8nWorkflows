Generate & Post Unique MCQ Polls to Telegram with Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/generate---post-unique-mcq-polls-to-telegram-with-gemini-ai-and-google-sheets-9264


# Generate & Post Unique MCQ Polls to Telegram with Gemini AI and Google Sheets

### 1. Workflow Overview

This n8n workflow automates the creation, storage, and publishing of unique multiple-choice quiz questions (MCQs) to a Telegram channel or group, leveraging AI (Google Gemini) for content generation and Google Sheets for data management.

It is designed primarily for educational content creators preparing quizzes for competitive exams like BPSC, UPSC, SSC, etc., who want to automate MCQ generation and Telegram poll posting.

The workflow is logically divided into two main interconnected blocks:

- **1.1 Content Generation & Storage (Scheduled Trigger Path):**  
  Automatically generates a new, unique MCQ in English and Hindi using AI. It checks existing quiz data in Google Sheets to avoid duplicates, then appends or updates the generated question into the sheet for review.

- **1.2 Publishing & Status Update (Event-Driven Path):**  
  Triggered by row updates in Google Sheets, it scans for approved but unposted quiz questions, posts them as Telegram polls, then marks them as posted in the sheet. Finally, it signals the generation block to create the next question, maintaining a continuous quiz publishing cycle.

---

### 2. Block-by-Block Analysis

#### 2.1 Content Generation & Storage (Scheduled Trigger Path)

**Overview:**  
This block runs every 3 hours, invoking an AI agent that creates a brand new MCQ on a random General Studies topic relevant to the BPSC syllabus. It uses existing sheet data as memory to ensure question uniqueness and stores the new MCQ in Google Sheets.

**Nodes Involved:**  
- Schedule Trigger  
- AI Agent  
- Google Gemini Chat Model  
- Simple Memory  
- Get row(s) in sheet in Google Sheets  
- Append or update row in sheet in Google Sheets1

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the content generation process every 3 hours automatically.  
  - *Configuration:* Interval set to 3 hours.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Triggers AI Agent node.  
  - *Edge Cases:* Workflow will not trigger if n8n is offline or schedule is misconfigured.

- **AI Agent**  
  - *Type:* Langchain Agent Node  
  - *Role:* Generates a new MCQ with options, correct answer, and explanation in English and Hindi.  
  - *Configuration:*  
    - Prompt instructs to create one unique MCQ in English and Hindi, aligned to BPSC syllabus, avoiding repeats.  
    - Uses system message defining AI’s expert persona and output format (question, options, correct answer, explanation, Hindi translation).  
  - *Expressions:* Uses sheet data as memory input to avoid duplicating questions.  
  - *Inputs:* Trigger from Schedule Trigger, memory context from Simple Memory, and existing questions via Get row(s) in sheet in Google Sheets.  
  - *Outputs:* Produces MCQ data for storage.  
  - *Version Specifics:* Requires Langchain integration with Google Gemini model.  
  - *Failure Modes:* API limits, AI model errors, or malformed prompts may cause failures.

- **Google Gemini Chat Model**  
  - *Type:* Langchain Language Model Node  
  - *Role:* Provides the underlying AI language model (Gemini 2.5 Pro) for generating text.  
  - *Configuration:* Specifies model "models/gemini-2.5-pro".  
  - *Inputs/Outputs:* Connected as AI language model provider for AI Agent node.  
  - *Failure Modes:* Authentication errors, quota limits, network timeouts.

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains session memory keyed by execution ID to provide context/history for AI generation.  
  - *Configuration:* Uses custom sessionKey based on execution ID.  
  - *Inputs:* AI Agent outputs.  
  - *Outputs:* Memory data back to AI Agent for iterative context.  
  - *Edge Cases:* Memory overflow or session key conflicts.

- **Get row(s) in sheet in Google Sheets**  
  - *Type:* Google Sheets Tool Node  
  - *Role:* Retrieves existing quiz questions to avoid duplication by filtering on the 'Question' column.  
  - *Configuration:* Filters on question column using AI-generated values to check uniqueness.  
  - *Inputs:* Receives requests from AI Agent node.  
  - *Outputs:* Provides existing questions as input to AI Agent.  
  - *Credentials:* Requires configured Google Sheets credentials.  
  - *Failures:* API quota, incorrect sheet ID/GID, or network issues.

- **Append or update row in sheet in Google Sheets1**  
  - *Type:* Google Sheets Tool Node  
  - *Role:* Appends new MCQ data or updates existing rows in the Google Sheet with AI-generated quiz content.  
  - *Configuration:* Maps AI output fields (Quiz no, Subject, Question, Options A-D, Explanation, Correct Answer, Approval Status, Posted on Telegram) to sheet columns. Uses 'Quiz no' as matching column for updating.  
  - *Inputs:* AI Agent output.  
  - *Outputs:* Updates Google Sheet with new question data.  
  - *Credentials:* Requires Google Sheets credentials.  
  - *Edge Cases:* Row conflicts, write permission issues, malformed data.

---

#### 2.2 Publishing & Status Update (Event-Driven Path)

**Overview:**  
This block listens for changes in the Google Sheet indicating new or approved quiz questions, then posts them as polls on Telegram. After successful posting, it marks the question as posted and triggers the generation of the next question.

**Nodes Involved:**  
- Google Sheets Trigger  
- Read Quiz Data  
- Check New Quiz Added (If node)  
- Send Telegram Poll (HTTP Request)  
- Update Quiz Status1 (Google Sheets)  
- Aggregate

**Node Details:**

- **Google Sheets Trigger**  
  - *Type:* Google Sheets Trigger Node  
  - *Role:* Listens for row updates in specific columns ('Approval Status' and 'Posted on Telegram') to trigger downstream logic.  
  - *Configuration:* Watches the specified columns in the designated sheet and document ID. Polls every minute.  
  - *Outputs:* Triggers Read Quiz Data node upon detected change.  
  - *Failures:* Google API errors, incorrect credentials.

- **Read Quiz Data**  
  - *Type:* Google Sheets Node  
  - *Role:* Reads all quiz rows from Google Sheets to obtain current quiz data for evaluation.  
  - *Configuration:* Reads entire sheet specified by document ID and sheet name (GID).  
  - *Outputs:* Provides quiz data to Check New Quiz Added node.  
  - *Edge Cases:* Large sheet size may cause performance delays.

- **Check New Quiz Added (If Node)**  
  - *Type:* If Node  
  - *Role:* Determines if a quiz row is ready to be posted by checking:  
    - 'Approval Status' contains "Complete"  
    - 'Posted on Telegram' is not "✅"  
  - *Outputs:*  
    - True branch: sends data to Send Telegram Poll node.  
    - False branch: sends data to Aggregate node (to trigger next generation).  
  - *Edge Cases:* Case sensitivity in string checks, missing fields.

- **Send Telegram Poll (HTTP Request)**  
  - *Type:* HTTP Request Node  
  - *Role:* Sends the quiz question as a poll to Telegram using Telegram Bot API `sendPoll` method.  
  - *Configuration:*  
    - POST request to Telegram Bot API.  
    - Query parameters include question, options array (A-D), correct answer, and chat ID.  
    - Uses environment variables for `<TELEGRAM_BOT_TOKEN>` and `<TELEGRAM_CHAT_ID>`.  
  - *Inputs:* Quiz data from Check New Quiz Added node.  
  - *Outputs:* Provides Telegram API response to Update Quiz Status1 node.  
  - *Failures:* Invalid bot token, chat ID, API rate limits, network failures.

- **Update Quiz Status1 (Google Sheets)**  
  - *Type:* Google Sheets Node  
  - *Role:* Updates the row matching the quiz question, setting 'Posted on Telegram' to "✅" and optionally updating question text.  
  - *Configuration:* Matches rows by 'Question' column and updates 'Posted on Telegram' column.  
  - *Inputs:* Response from Send Telegram Poll node.  
  - *Outputs:* Confirms update, finalizes posting cycle.  
  - *Failures:* Write permission issues, mismatched rows.

- **Aggregate**  
  - *Type:* Aggregate Node  
  - *Role:* Aggregates workflow state and triggers AI Agent node again to generate the next MCQ, maintaining continuous operation.  
  - *Inputs:* False branch from Check New Quiz Added node (when no new quiz to post).  
  - *Outputs:* Triggers AI Agent node to start content generation loop.  
  - *Edge Cases:* Prevent infinite loops or excessive API calls.

---

### 3. Summary Table

| Node Name                     | Node Type                    | Functional Role                             | Input Node(s)         | Output Node(s)           | Sticky Note                                               |
|-------------------------------|------------------------------|---------------------------------------------|-----------------------|--------------------------|-----------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger              | Triggers AI content generation every 3 hrs | None                  | AI Agent                 | Workflow 1: Content Generation & Storage (Scheduled run every 3 hours) |
| AI Agent                     | Langchain Agent              | Generates unique MCQ in English & Hindi     | Schedule Trigger, Simple Memory, Get row(s) in sheet | Append or update row in sheet in Google Sheets1 | Automated MCQ Generation and Telegram Poll Posting Workflow (see detailed note) |
| Google Gemini Chat Model      | Langchain Language Model     | Provides Gemini AI model for text generation | AI Agent               | AI Agent                 |                                                           |
| Simple Memory                | Langchain Memory Buffer      | Maintains session memory for AI context     | AI Agent               | AI Agent                 |                                                           |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool          | Reads existing questions to avoid duplicates | AI Agent               | AI Agent                 |                                                           |
| Append or update row in sheet in Google Sheets1 | Google Sheets Tool          | Stores new MCQ data in Google Sheets         | AI Agent               | None                     |                                                           |
| Google Sheets Trigger         | Google Sheets Trigger        | Triggers on sheet row updates                | None                   | Read Quiz Data           | Workflow 2: Publishing & Status Update (Triggered by new row in Google Sheet) |
| Read Quiz Data               | Google Sheets                | Reads all quiz rows for evaluation           | Google Sheets Trigger  | Check New Quiz Added     |                                                           |
| Check New Quiz Added          | If Node                     | Checks if quiz is approved & unposted        | Read Quiz Data         | Send Telegram Poll, Aggregate |                                                           |
| Send Telegram Poll            | HTTP Request                | Posts quiz as Telegram poll                   | Check New Quiz Added   | Update Quiz Status1      | Calls Telegram sendPoll. Set TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID as env vars. |
| Update Quiz Status1           | Google Sheets               | Marks quiz row as posted                       | Send Telegram Poll     | None                     |                                                           |
| Aggregate                    | Aggregate                   | Triggers AI generation loop again            | Check New Quiz Added (False branch) | AI Agent                 |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set interval to every 3 hours.  
   - This node starts the content generation loop.

2. **Create a Google Gemini Chat Model node:**  
   - Set modelName to `"models/gemini-2.5-pro"`.  
   - Configure with Google Gemini credentials.

3. **Create a Simple Memory node:**  
   - Use type `memoryBufferWindow`.  
   - Set `sessionKey` as an expression `{{$execution.id}}` and `sessionIdType` to `customKey`.  
   - This node provides conversational memory context for the AI.

4. **Create a Get row(s) in sheet in Google Sheets node:**  
   - Connect Google Sheets credentials.  
   - Configure to read from your quiz sheet (provide Document ID and Sheet GID).  
   - Use filter on the "Question" column to check existing questions for uniqueness.

5. **Create an AI Agent node:**  
   - Set node type to Langchain Agent.  
   - Configure `text` parameter with prompt to generate a unique MCQ in English and Hindi, with options, correct answer, and explanation.  
   - Set system message instructing AI on exam relevance, uniqueness, format, and language.  
   - Connect inputs: Schedule Trigger (main trigger), Google Gemini Chat Model (AI language model), Simple Memory (memory), and Get row(s) in sheet (AI tool).  
   - Ensure credentials for AI and Google Gemini are set.

6. **Create an Append or update row in sheet in Google Sheets node:**  
   - Connect Google Sheets credentials.  
   - Configure Document ID and Sheet GID for your quiz sheet.  
   - Map AI output fields to columns: Quiz no, Subject, Question, Options A-D, Explanation, Correct Answer, Approval Status, Posted on Telegram.  
   - Use "Quiz no" as matching column for updates.  
   - Connect AI Agent output as input here.

7. **Connect nodes in order:**  
   - Schedule Trigger → AI Agent (main)  
   - Google Gemini Chat Model → AI Agent (languageModel)  
   - Simple Memory → AI Agent (memory)  
   - Get row(s) in sheet → AI Agent (ai_tool)  
   - AI Agent → Append or update row in sheet

8. **Create a Google Sheets Trigger node:**  
   - Connect Google Sheets credentials.  
   - Set event to `rowUpdate` on columns `Approval Status` and `Posted on Telegram`.  
   - Set polling interval to every minute.  
   - Configure Document ID and Sheet GID of your quiz sheet.

9. **Create a Read Quiz Data node:**  
   - Connect Google Sheets credentials.  
   - Set to read all rows from the quiz sheet.

10. **Create an If node named Check New Quiz Added:**  
    - Condition 1: Check if `Approval Status` contains "Complete".  
    - Condition 2: Check if `Posted on Telegram` is not equal to "✅".  
    - If both true, output to Send Telegram Poll; else output to Aggregate.

11. **Create a Send Telegram Poll node (HTTP Request):**  
    - Method: POST  
    - URL: `https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/sendPoll` (replace `<TELEGRAM_BOT_TOKEN>` with your environment variable or actual token).  
    - Query Parameters:  
      - `question`: set from `{{$json.Question}}`  
      - `options`: JSON array of options from columns A-D  
      - `Correct Answer`: from `{{$json['Correct Answer']}}`  
      - `chat_id`: your Telegram chat ID (replace `<TELEGRAM_CHAT_ID>`).  
    - Header: Content-Type: application/json  
    - Connect input from Check New Quiz Added (true branch).

12. **Create an Update Quiz Status1 node (Google Sheets):**  
    - Connect Google Sheets credentials.  
    - Configure to update the row matching the `Question` column.  
    - Set `Posted on Telegram` column value to "✅".  
    - Connect input from Send Telegram Poll output.

13. **Create an Aggregate node:**  
    - Connect input from Check New Quiz Added (false branch).  
    - Connect output to AI Agent node to loop generation.

14. **Connect nodes in order:**  
    - Google Sheets Trigger → Read Quiz Data  
    - Read Quiz Data → Check New Quiz Added  
    - Check New Quiz Added (true) → Send Telegram Poll → Update Quiz Status1  
    - Check New Quiz Added (false) → Aggregate → AI Agent (to restart generation)

15. **Set environment variables or securely store credentials:**  
    - Google Sheets API credentials for all Google Sheets nodes.  
    - Google Gemini AI credentials for Gemini Chat Model.  
    - Telegram Bot Token and Chat ID as environment variables or credentials manager entries.  
    - Ensure no sensitive info is hardcoded in the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for educators and coaching institutes looking to automate daily quiz or poll publishing on Telegram channels/groups. It supports bilingual quiz generation (English and Hindi) and is specifically tailored towards BPSC General Studies syllabus but can be adapted for other exams.                                                                                                                                                                                          | Workflow description and intended audience.                                                                                      |
| The Telegram Bot API requires the bot token and chat ID; ensure these are securely stored as environment variables (`TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`) and not hardcoded.                                                                                                                                                                                                                                                                                                                 | Telegram API documentation: https://core.telegram.org/bots/api#sendpoll                                                           |
| AI Agent uses Google Gemini 2.5 Pro model; ensure you have access and credentials configured for Google Gemini in n8n. The prompt and system message are carefully crafted to produce exam-relevant, unique MCQs with concise explanations and bilingual output.                                                                                                                                                                                                                                  | Google Gemini AI docs: https://developers.generativeai.google/                                                                    |
| Google Sheets columns required include: Quiz no, Subject, Question, Option A, Option B, Option C, Option D, Correct Answer, Explanation, Approval Status, Posted on Telegram. Approval Status is used to control which questions are published; Posted on Telegram marks published questions.                                                                                                                                                                                                      | Google Sheets setup instructions within workflow notes.                                                                           |
| Sticky note inside the workflow provides a detailed explanation of how the two parts (generation and publishing) work together for continuous quiz supply. It also includes setup instructions for credentials and environment variables.                                                                                                                                                                                                                                                       | Visible as sticky notes inside the workflow editor.                                                                               |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.