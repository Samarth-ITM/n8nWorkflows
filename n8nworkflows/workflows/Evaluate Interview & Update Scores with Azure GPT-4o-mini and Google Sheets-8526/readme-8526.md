Evaluate Interview & Update Scores with Azure GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/evaluate-interview---update-scores-with-azure-gpt-4o-mini-and-google-sheets-8526


# Evaluate Interview & Update Scores with Azure GPT-4o-mini and Google Sheets

### 1. Workflow Overview

This workflow automates the evaluation of candidate questionnaire responses by integrating AI analysis with existing candidate profile data stored in Google Sheets. It continuously monitors new questionnaire submissions, processes responses with Azure’s GPT-4o-mini model via a Langchain agent, parses and scores the AI output, retrieves candidate profile information, calculates combined final scores, and updates the candidate database accordingly.

**Target Use Cases:**  
- Automated candidate screening for recruitment teams  
- Real-time evaluation of questionnaire responses for roles requiring domain-specific knowledge  
- Combining AI-driven assessment with existing resume scores for holistic candidate evaluation

**Logical Blocks:**

- **1.1 Input Reception and Trigger:** Detects new questionnaire form submissions in Google Sheets.
- **1.2 AI Processing:** Formats candidate answers and sends them to Azure GPT-4o-mini for evaluation.
- **1.3 AI Output Parsing:** Converts AI textual output into structured JSON data.
- **1.4 Candidate Profile Lookup:** Retrieves existing candidate data from a separate Google Sheets database.
- **1.5 Score Calculation:** Combines AI questionnaire scores with existing candidate scores.
- **1.6 Database Update:** Appends or updates candidate records with new scores and evaluation data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Trigger

- **Overview:**  
  Continuously monitors the Google Sheets document containing questionnaire responses and triggers the workflow immediately upon detection of new submissions.

- **Nodes Involved:**  
  - Monitor New Questionnaire Responses

- **Node Details:**

  - **Monitor New Questionnaire Responses**  
    - *Type:* Google Sheets Trigger  
    - *Role:* Initiates workflow on new form submissions  
    - *Configuration:*  
      - Polls every minute for new rows in sheet "Form Responses 1" of spreadsheet "BD Questionarie"  
      - Uses OAuth2 credentials for Google Sheets Trigger  
    - *Input/Output:* No input; outputs newly detected rows  
    - *Edge Cases:*  
      - Possible delays or missed triggers if polling is interrupted  
      - Authentication or API quota errors with Google Sheets  
    - *Sticky Note:* Emphasizes real-time monitoring and reliability (Sticky Note1)  

#### 1.2 AI Processing

- **Overview:**  
  Prepares candidate answers from the questionnaire, sends them through a Langchain AI agent that uses Azure GPT-4o-mini, and requests the model to evaluate responses based on knowledge, problem-solving, and communication clarity.

- **Nodes Involved:**  
  - AI Questionnaire Evaluator  
  - Azure OpenAI GPT-4 Model

- **Node Details:**

  - **AI Questionnaire Evaluator**  
    - *Type:* Langchain Agent  
    - *Role:* Formats prompt with candidate answers and sends to AI language model  
    - *Configuration:*  
      - Prompt includes role, candidate questions, and answers injected via expressions referencing Google Sheets columns  
      - System message instructs AI to evaluate and score answers (0-30 scale) and return JSON output with score and key takeaways  
    - *Expressions:* Uses `{{ $json["<Question Column Name>"] }}` to inject candidate answers dynamically  
    - *Input:* Outputs from Google Sheets Trigger  
    - *Output:* Raw AI response text  
    - *Edge Cases:*  
      - AI response might not be valid JSON  
      - API timeouts or rate limits on Azure OpenAI  
    - *Sub-workflow:* Uses Azure OpenAI GPT-4 Model node as language model  

  - **Azure OpenAI GPT-4 Model**  
    - *Type:* Langchain Azure OpenAI Chat Model  
    - *Role:* Executes GPT-4o-mini chat completions for AI Questionnaire Evaluator  
    - *Configuration:*  
      - Model: gpt-4o-mini (lightweight GPT-4 variant)  
      - Azure OpenAI credentials configured  
    - *Input:* Prompt from AI Questionnaire Evaluator  
    - *Output:* AI-generated evaluation text  
    - *Edge Cases:*  
      - Azure service outages or credential expiration  
      - Model limitations affecting output quality  
    - *Sticky Note:* Details model choice and capabilities (Sticky Note3)  
    - *Sticky Note:* Describes AI evaluation engine function (Sticky Note2)  

#### 1.3 AI Output Parsing

- **Overview:**  
  Cleans the AI text output by removing markdown code fences, attempts to parse it as JSON, and handles parsing errors gracefully to ensure downstream nodes receive consistent data.

- **Nodes Involved:**  
  - Parse AI Evaluation Results

- **Node Details:**

  - **Parse AI Evaluation Results**  
    - *Type:* Code (JavaScript)  
    - *Role:* Cleans and parses AI JSON output  
    - *Configuration:*  
      - Removes ```json and ``` markdown fences  
      - Parses JSON; if parsing fails, returns error object containing raw text  
    - *Input:* Raw AI response from AI Questionnaire Evaluator  
    - *Output:* Parsed JSON object with keys: `questionnaire_score` and `key_takeaways` or error details  
    - *Edge Cases:*  
      - Malformed AI responses causing JSON parse failure  
      - Empty or missing AI output  
    - *Sticky Note:* Explains parsing logic and error handling (Sticky Note4)  

#### 1.4 Candidate Profile Lookup

- **Overview:**  
  Retrieves existing candidate data including previous scores and profile details from a central Google Sheets database to integrate with new questionnaire results.

- **Nodes Involved:**  
  - Lookup Candidate Profile Data

- **Node Details:**

  - **Lookup Candidate Profile Data**  
    - *Type:* Google Sheets  
    - *Role:* Reads candidate profile data from "Resume store" spreadsheet, "Sheet2"  
    - *Configuration:*  
      - Uses OAuth2 credentials for Google Sheets  
      - Reads all columns relevant to candidate profile and scores  
    - *Input:* Parsed AI evaluation JSON  
    - *Output:* Candidate profile data matched by candidate identifier (implicit via downstream matching)  
    - *Edge Cases:*  
      - Data mismatch or missing profile for candidate  
      - Google Sheets API quota or permission issues  
    - *Sticky Note:* Describes purpose and data retrieved (Sticky Note5)  

#### 1.5 Score Calculation

- **Overview:**  
  Combines the AI questionnaire score with existing candidate scores to produce a final comprehensive evaluation score.

- **Nodes Involved:**  
  - Calculate Combined Scores

- **Node Details:**

  - **Calculate Combined Scores**  
    - *Type:* Set  
    - *Role:* Creates/updates fields for final score and questionnaire score  
    - *Configuration:*  
      - Calculates final score as sum of existing score and AI questionnaire score  
      - Assigns `final score` and `Questionarie Score` fields for later database update  
    - *Input:* Candidate profile data and parsed AI results  
    - *Output:* Updated JSON including combined scores  
    - *Expressions:*  
      - `final score = $json.Score + $('Parse AI Evaluation Results').item.json.questionnaire_score`  
      - `Questionarie Score = $('Parse AI Evaluation Results').item.json.questionnaire_score`  
    - *Edge Cases:*  
      - Missing or non-numeric existing scores causing calculation errors  
    - *Sticky Note:* Describes scoring logic and output fields (Sticky Note6)  

#### 1.6 Database Update

- **Overview:**  
  Appends or updates candidate records in the Google Sheets "Resume store" with new questionnaire scores and combined final scores, preserving existing data.

- **Nodes Involved:**  
  - Update Candidate Database

- **Node Details:**

  - **Update Candidate Database**  
    - *Type:* Google Sheets  
    - *Role:* Writes updated candidate evaluation data back to "Resume store" spreadsheet, "Sheet2"  
    - *Configuration:*  
      - Operation: Append or update matching on "Name" column  
      - Writes fields: Name, Final score, Questionarie Score  
      - Maintains all other profile fields intact  
    - *Input:* Calculated combined scores and candidate profile data  
    - *Output:* Confirmation of database write operation  
    - *Edge Cases:*  
      - Duplicate names causing ambiguous updates  
      - Write permission or API quota issues  
    - *Sticky Note:* Details update method and data integrity (Sticky Note7)  

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                                  | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                   |
|--------------------------------|----------------------------------|-------------------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Monitor New Questionnaire Responses | Google Sheets Trigger             | Triggers workflow on new questionnaire submissions | -                                | AI Questionnaire Evaluator       | Monitors new responses in real-time; polls every minute; reliable trigger from Google Sheets (Sticky Note1)  |
| AI Questionnaire Evaluator      | Langchain Agent                  | Sends candidate answers to AI for evaluation    | Monitor New Questionnaire Responses | Parse AI Evaluation Results      | Evaluates candidate answers with Azure GPT-4o-mini; outputs JSON score and key takeaways (Sticky Note2)      |
| Azure OpenAI GPT-4 Model        | Langchain Azure OpenAI Chat Model | Provides GPT-4o-mini model for AI evaluation    | AI Questionnaire Evaluator (AI Model) | AI Questionnaire Evaluator (AI Model) | Efficient GPT-4 variant; Azure OpenAI service integration; optimized for evaluation (Sticky Note3)           |
| Parse AI Evaluation Results     | Code (JavaScript)                | Parses and cleans AI JSON output                 | AI Questionnaire Evaluator       | Lookup Candidate Profile Data    | Cleans code fences; parses JSON safely; handles errors gracefully (Sticky Note4)                             |
| Lookup Candidate Profile Data   | Google Sheets                   | Retrieves candidate profile and previous scores | Parse AI Evaluation Results       | Calculate Combined Scores        | Fetches candidate info from central database; links questionnaire to profile (Sticky Note5)                 |
| Calculate Combined Scores       | Set                             | Combines AI and existing scores                  | Lookup Candidate Profile Data     | Update Candidate Database        | Adds questionnaire score to existing score to compute final score (Sticky Note6)                             |
| Update Candidate Database       | Google Sheets                   | Updates candidate records with new scores        | Calculate Combined Scores         | -                               | Appends or updates records by name; maintains data integrity (Sticky Note7)                                 |
| Sticky Note1                   | Sticky Note                    | Comment block                                    | -                                | -                               | Describes trigger node monitoring and reliability                                                           |
| Sticky Note2                   | Sticky Note                    | Comment block                                    | -                                | -                               | Describes AI evaluation engine and scoring criteria                                                          |
| Sticky Note3                   | Sticky Note                    | Comment block                                    | -                                | -                               | Details Azure GPT-4o-mini model configuration                                                                |
| Sticky Note4                   | Sticky Note                    | Comment block                                    | -                                | -                               | Explains AI output parsing and error handling                                                                |
| Sticky Note5                   | Sticky Note                    | Comment block                                    | -                                | -                               | Describes candidate profile data retrieval                                                                   |
| Sticky Note6                   | Sticky Note                    | Comment block                                    | -                                | -                               | Explains score calculation logic                                                                              |
| Sticky Note7                   | Sticky Note                    | Comment block                                    | -                                | -                               | Details database update process                                                                                |
| Sticky Note8                   | Sticky Note                    | Workflow overview and benefits                   | -                                | -                               | Summarizes entire workflow purpose and benefits                                                              |
| Sticky Note9                   | Sticky Note                    | Configuration and maintenance notes              | -                                | -                               | Provides question details, scoring weights, monitoring frequency, and maintenance tasks                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node: "Monitor New Questionnaire Responses"**  
   - Type: Google Sheets Trigger  
   - Set to poll the spreadsheet "BD Questionarie" (ID: 1SFFWWqefQ6tcaEK0_Ovy9nVgyxS4LQPvno1i8fi39MY)  
   - Sheet: "Form Responses 1" (gid: 3745099)  
   - Polling: Every minute  
   - Credentials: Configure OAuth2 for Google Sheets Trigger  
   - Purpose: Trigger on new questionnaire submissions  

2. **Create Langchain Agent Node: "AI Questionnaire Evaluator"**  
   - Type: Langchain Agent  
   - Input: Use Google Sheets Trigger output  
   - Prompt template:  
     ```
     Role:
     Questions and Candidate Answers:
     Q1: What is Business development according to you
     A1: {{ $json["What is BD according to you?"] }}
     Q2: What is SWOT analysis
     A2: {{ $json["What is SWOT analysis"] }}

     Please evaluate the above responses based on knowledge depth, problem-solving ability, and communication clarity.
     ```  
   - System message:  
     ```
     You are an AI evaluator for candidate screening. 
     You will be given:
     - A set of role-specific questions.
     - A candidate's answers.

     Your job:
     1. Evaluate the quality of the answers against the intent of each question.
     2. Score the questionnaire (0–30 by default, or adjust scale if specified).
     3. Provide key takeaways (strengths, weaknesses, red flags, or standout qualities).

     Output ONLY in JSON with this structure:
     {
       "questionnaire_score": <number>,
       "key_takeaways": "<summary of candidate performance>"
     }
     ```  
   - Connect the node to a language model via AI Model input  

3. **Create Langchain Azure OpenAI Chat Model Node: "Azure OpenAI GPT-4 Model"**  
   - Type: Langchain Azure OpenAI Chat Model  
   - Model: gpt-4o-mini  
   - Credentials: Configure Azure OpenAI API credentials  
   - Connect as AI language model for the "AI Questionnaire Evaluator" node  

4. **Create Code Node: "Parse AI Evaluation Results"**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     return items.map(item => {
       let text = item.json.output;
       text = text.replace(/```json|```/g, "").trim();
       let parsed = {};
       try {
         parsed = JSON.parse(text);
       } catch (err) {
         parsed = { error: "Failed to parse JSON", raw: text };
       }
       return { json: parsed };
     });
     ```  
   - Input: Output from AI Questionnaire Evaluator  
   - Purpose: Clean and parse AI JSON output  

5. **Create Google Sheets Node: "Lookup Candidate Profile Data"**  
   - Type: Google Sheets (Read)  
   - Spreadsheet: "Resume store" (ID: 1JlXxy90s0we_IqErHyvomrJSijb8pd4H91hOUCH6xCA)  
   - Sheet: "Sheet2" (gid: 1424038785)  
   - Credentials: OAuth2 Google Sheets credentials  
   - Purpose: Retrieve candidate profile and previous scores for integration  
   - Input: Parsed AI evaluation JSON  

6. **Create Set Node: "Calculate Combined Scores"**  
   - Type: Set  
   - Assign the following fields:  
     - `final score` = existing `$json.Score` + AI questionnaire score (`$('Parse AI Evaluation Results').item.json.questionnaire_score`)  
     - `Questionarie Score` = AI questionnaire score  
   - Purpose: Combine existing candidate scores with new questionnaire evaluation  

7. **Create Google Sheets Node: "Update Candidate Database"**  
   - Type: Google Sheets (Append or Update)  
   - Spreadsheet: "Resume store" (same as lookup)  
   - Sheet: "Sheet2"  
   - Matching Columns: "Name"  
   - Columns to update: Name, Final score, Questionarie Score (plus any existing fields preserved)  
   - Credentials: OAuth2 Google Sheets credentials  
   - Purpose: Update candidate records with new scores  
   - Input: Output from "Calculate Combined Scores" node  

8. **Connect Nodes in Sequence:**  
   - Monitor New Questionnaire Responses → AI Questionnaire Evaluator → Parse AI Evaluation Results → Lookup Candidate Profile Data → Calculate Combined Scores → Update Candidate Database  
   - Connect Azure OpenAI GPT-4 Model as AI language model to AI Questionnaire Evaluator  

9. **Configure Credentials:**  
   - Google Sheets Trigger OAuth2 for trigger node  
   - Google Sheets OAuth2 for read/write nodes  
   - Azure OpenAI API credentials for GPT-4 model node  

10. **Test Workflow:**  
    - Verify trigger activates on new form submission  
    - Confirm AI evaluation returns valid JSON score and takeaways  
    - Check correct parsing and error handling for AI output  
    - Validate profile lookup and score calculation accuracy  
    - Confirm database updates correctly without overwriting unrelated fields  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automatically evaluates candidate questionnaire responses integrating AI scoring with existing resume scores for a comprehensive assessment.                                                                            | Overall workflow purpose (Sticky Note8)                                                            |
| AI evaluation criteria include knowledge depth (40%), problem-solving (30%), and communication clarity (30%), with a scoring range of 0-30 points per questionnaire.                                                              | Configuration details (Sticky Note9)                                                               |
| Questions currently evaluated: "What is Business Development according to you?" and "What is SWOT analysis?" These can be customized depending on role requirements.                                                             | Questionnaire customization (Sticky Note9)                                                        |
| Real-time evaluation with polling every minute ensures prompt processing of candidate submissions.                                                                                                                                 | Monitoring setting (Sticky Note9)                                                                  |
| Use consistent and up-to-date Azure OpenAI credentials and Google Sheets OAuth2 credentials to avoid authentication failures.                                                                                                      | Credential best practices                                                                           |
| Backup Google Sheets data regularly to prevent data loss.                                                                                                                                                                          | Maintenance advice (Sticky Note9)                                                                  |
| For best AI output quality, periodically review the prompt and evaluation criteria to adjust scoring logic and instructions.                                                                                                      | Maintenance advice (Sticky Note9)                                                                  |

---

*Disclaimer:* The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The process adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.