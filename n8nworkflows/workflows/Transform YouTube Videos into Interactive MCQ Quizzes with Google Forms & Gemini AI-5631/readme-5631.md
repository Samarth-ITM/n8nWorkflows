Transform YouTube Videos into Interactive MCQ Quizzes with Google Forms & Gemini AI

https://n8nworkflows.xyz/workflows/transform-youtube-videos-into-interactive-mcq-quizzes-with-google-forms---gemini-ai-5631


# Transform YouTube Videos into Interactive MCQ Quizzes with Google Forms & Gemini AI

---

### 1. Workflow Overview

This workflow automates the process of transforming YouTube videos into interactive multiple-choice question (MCQ) quizzes hosted on Google Forms, utilizing Google Gemini AI to generate questions. It targets educators, trainers, and content creators who want to quickly convert video content into engaging quizzes without manual question crafting.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Accepts user input via an n8n form trigger, capturing YouTube URL, form name, email, and the desired number of questions.
  
- **1.2 Input Validation and Prompt Setup:** Validates the number of questions, prepares a prompt for the Gemini AI model specifying the MCQ generation task.
  
- **1.3 AI Processing with Google Gemini:** Sends the prompt and YouTube video URL to Google Gemini's generative language model to generate quiz questions with answers.
  
- **1.4 Error Handling and Outcome Definition:** Checks if the AI response contains errors, defines output fields accordingly, or routes to error output.
  
- **1.5 JSON Extraction & Parsing:** Extracts and converts the AI-generated text into structured JSON containing questions, options, and answers.
  
- **1.6 Google Form Creation:** Creates a new Google Form using Google Forms API with the user-provided form name.
  
- **1.7 Questions Preparation for Google Forms API:** Transforms the extracted JSON questions into the specific request format required by the Google Forms batchUpdate API.
  
- **1.8 Quiz Questions Upload:** Sends the prepared questions to Google Forms API to populate the form as a quiz with grading settings.
  
- **1.9 Post-Creation Handling:** Redirects the user to the created Google Form or displays error messages in case of failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Collects essential user inputs for quiz generation through a web form.
- **Nodes Involved:**  
  - Input YouTube URL
  - filter
  - Sticky Note (Input description)

- **Node Details:**

  - **Input YouTube URL**  
    - Type: Form Trigger  
    - Role: Entry point; gathers 'Form Name', 'Email', 'YouTube URL', and 'Number of Questions'.  
    - Configuration:  
      - Form title: "Quize Form YT Videos"  
      - Form description specifies limits (max 90 questions, up to 50-minute video)  
      - Required fields: Form Name, YouTube URL, Number of Questions  
      - Webhook ID for external triggering  
    - Inputs: None (trigger)  
    - Outputs: Connects to 'filter' node  
    - Edge Cases: Invalid URLs, missing required fields, number of questions exceeding limit  
    - Sticky Note: "Take input (YouTube URL, Form name, Number of questions) by n8n form trigger."

  - **filter**  
    - Type: If  
    - Role: Ensures 'Number of Questions' is less than 91 to comply with limits  
    - Configuration: Checks if $json['Number of Questions'] < 91  
    - Inputs: From 'Input YouTube URL'  
    - Outputs: Passes valid inputs to 'Set Prompt and Model', invalid to 'Error Output'  
    - Edge Cases: Number of questions missing or non-numeric, exceeding 90

#### 2.2 Input Validation and Prompt Setup

- **Overview:** Sets up the prompt string for the AI model and designates the model version to use.
- **Nodes Involved:**  
  - Set Prompt and Model  
  - Sticky Note (Prompt and model)

- **Node Details:**

  - **Set Prompt and Model**  
    - Type: Set  
    - Role: Defines the prompt to generate MCQs and selects the Gemini model version  
    - Configuration:  
      - prompt: "Generate {{ $json['Number of Questions'] }} MCQ (Multiple Choice Question) from the video with answer, every question will have 3 options and return those question as json format."  
      - model: "gemini-2.5-flash"  
      - Includes all other input fields  
    - Inputs: From 'filter'  
    - Outputs: To 'HTTP Request to Gemini'  
    - Edge Cases: Empty or malformed 'Number of Questions' field

#### 2.3 AI Processing with Google Gemini

- **Overview:** Sends the prompt and YouTube URL to Google Gemini AI to generate quiz questions.
- **Nodes Involved:**  
  - HTTP Request to Gemini  
  - If: Was an error detected?  
  - Set Fields: Define Outcome  
  - Extract JSON  
  - Sticky Notes (AI request and JSON conversion)

- **Node Details:**

  - **HTTP Request to Gemini**  
    - Type: HTTP Request  
    - Role: Calls Google Gemini generative language API to produce MCQs based on video content  
    - Configuration:  
      - POST to Gemini endpoint with API key (placeholder "API_KEY")  
      - JSON body includes prompt text and file_uri (YouTube URL)  
      - Timeout set very high to accommodate video processing  
      - Content-Type: application/json  
    - Inputs: From 'Set Prompt and Model'  
    - Outputs: Passes response to 'If: Was an error detected?'  
    - Edge Cases: API key invalid, timeout, rate limits, malformed response  
    - Error handling: Continues on error but routes to error output downstream if detected

  - **If: Was an error detected?**  
    - Type: If  
    - Role: Checks if AI response contains text content (presence of generated answer)  
    - Configuration: Tests existence of $json.candidates[0].content.parts[0].text  
    - Inputs: From 'HTTP Request to Gemini'  
    - Outputs: Valid to 'Set Fields: Define Outcome', error to 'Error Output'  
    - Edge Cases: Empty or error responses from API

  - **Set Fields: Define Outcome**  
    - Type: Set  
    - Role: Extracts and sets key fields like AI-generated answer, token counts, and model version, excluding raw candidates and usage metadata  
    - Inputs: From 'If: Was an error detected?'  
    - Outputs: To 'Extract JSON'  
    - Edge Cases: Missing fields in response JSON

  - **Extract JSON**  
    - Type: Chain LLM (LangChain)  
    - Role: Extracts structured JSON from the AI-generated free text using a defined JSON schema example for questions and answers  
    - Configuration:  
      - JSON schema example includes array "Questions" with question text, options, and answer  
    - Inputs: From 'Set Fields: Define Outcome'  
    - Outputs: To 'Create a Google Form' or 'Error Output'  
    - Edge Cases: Parsing failure, unexpected format in AI output

#### 2.4 Google Form Creation

- **Overview:** Creates a new Google Form to host the quiz using Google Forms API.
- **Nodes Involved:**  
  - Create a Google Form  
  - Sticky Note (Form creation and JSON conversion)

- **Node Details:**

  - **Create a Google Form**  
    - Type: HTTP Request  
    - Role: Issues POST request to Google Forms API to create form with title and document title from user input  
    - Configuration:  
      - URL: https://forms.googleapis.com/v1/forms  
      - Method: POST  
      - JSON body sets 'info.title' and 'info.documentTitle' using form name and timestamp  
      - Auth: OAuth2 with Google Forms credentials  
      - Content-Type: application/json  
    - Inputs: From 'Extract JSON'  
    - Outputs: To 'Prepare Questions for API call' or 'Error Output'  
    - Edge Cases: Authentication failure, quota limits, invalid form data

#### 2.5 Preparing Questions for Google Forms API

- **Overview:** Transforms AI quiz questions into the specific batchUpdate request format required to populate the Google Form quiz.
- **Nodes Involved:**  
  - Prepare Questions for API call  
  - Sticky Note (Preparing JSON for Google Forms)

- **Node Details:**

  - **Prepare Questions for API call**  
    - Type: Code (JavaScript)  
    - Role: Iterates over extracted questions, builds an array of request objects for Google Forms batchUpdate  
    - Configuration:  
      - Reads questions array from 'Extract JSON' output  
      - For each question, creates options array (3 options)  
      - Sets grading with correct answer, point value 1, custom feedback for right/wrong answers  
      - Shuffle options enabled  
      - Sets question insertion order by index  
    - Inputs: From 'Create a Google Form'  
    - Outputs: To 'Create MCQ Quizzes'  
    - Edge Cases: Missing options, malformed data, empty questions array

#### 2.6 Quiz Questions Upload and Completion

- **Overview:** Uploads the questions to Google Form and manages redirection or error display.
- **Nodes Involved:**  
  - Create MCQ Quizzes  
  - Loop Over Items  
  - Redirect to Google Form  
  - Error Output  
  - Sticky Notes (Redirect and error handling)

- **Node Details:**

  - **Create MCQ Quizzes**  
    - Type: HTTP Request  
    - Role: Sends batchUpdate request to Google Forms API with prepared question requests to add quiz items  
    - Configuration:  
      - URL includes formId from created form  
      - Method: POST  
      - JSON body contains "requests" array from previous code node  
      - Auth: OAuth2 Google Forms  
      - Retries enabled (max 5) for robustness  
    - Inputs: From 'Prepare Questions for API call'  
    - Outputs: On success to 'Loop Over Items', on failure to 'Error Output'  
    - Edge Cases: API rate limits, partial failures, malformed request

  - **Loop Over Items**  
    - Type: Split in Batches  
    - Role: Splits the output for sequential processing or to handle multiple items (used here for managing flow after quiz creation)  
    - Inputs: From 'Create MCQ Quizzes'  
    - Outputs: On first branch to 'Redirect to Google Form', else to 'Create MCQ Quizzes' (loop back)  
    - Edge Cases: Empty batches, unexpected input

  - **Redirect to Google Form**  
    - Type: Form Completion (webhook)  
    - Role: Redirects user to the responder URI of the created Google Form for quiz attempt  
    - Configuration:  
      - Redirect URL dynamically set from 'Create a Google Form' node's form responder URI  
      - Respond with HTTP redirect  
    - Inputs: From 'Loop Over Items'  
    - Outputs: None (end)  
    - Edge Cases: Missing form URI, redirect failures

  - **Error Output**  
    - Type: Form Completion (webhook)  
    - Role: Displays error message if any step fails (invalid input, API error, etc.)  
    - Configuration:  
      - Title: "Something went wrong!"  
      - Message: "Please check your Youtube URL OR the number of questions."  
    - Inputs: Multiple error branches from earlier nodes  
    - Outputs: None (end)  
    - Sticky Note: "Default output for error handling."

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                           | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                  |
|-------------------------|-------------------------|-----------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Input YouTube URL       | Form Trigger            | Collects user input (YouTube URL, etc.) | None                       | filter                      | https://youtu.be/cZQPDLgPtNg?si=Ze6zOu5oTpF8QMq0; Take input (YouTube URL, Form name, Number of questions) by n8n form trigger. |
| filter                  | If                      | Validates number of questions < 91      | Input YouTube URL          | Set Prompt and Model / Error Output |                                                                                              |
| Set Prompt and Model    | Set                     | Sets AI prompt and model                 | filter                     | HTTP Request to Gemini       | Send "HTTP Request" to "Gemini-2.5-flash" for creating MCQ Quizzes.                          |
| HTTP Request to Gemini  | HTTP Request            | Calls Google Gemini AI to generate MCQs | Set Prompt and Model       | If: Was an error detected? / Error Output |                                                                                              |
| If: Was an error detected? | If                    | Checks if AI generated content exists   | HTTP Request to Gemini     | Set Fields: Define Outcome / Error Output |                                                                                              |
| Set Fields: Define Outcome | Set                   | Extracts key fields from AI response    | If: Was an error detected? | Extract JSON                |                                                                                              |
| Extract JSON            | Chain LLM (LangChain)   | Parses AI-generated text into JSON      | Set Fields: Define Outcome | Create a Google Form / Error Output | Convert Gemini Output (MCQ Questions and their answers) into json format and creating a Google Form. |
| Create a Google Form    | HTTP Request            | Creates a new Google Form                | Extract JSON               | Prepare Questions for API call / Error Output |                                                                                              |
| Prepare Questions for API call | Code              | Prepares quiz questions for Google Forms API | Create a Google Form       | Create MCQ Quizzes          | Prepare the json to match with the structure of Google forms api for creating the quizzes and redirect to the Google form. |
| Create MCQ Quizzes      | HTTP Request            | Uploads questions to Google Form         | Prepare Questions for API call | Loop Over Items / Error Output |                                                                                              |
| Loop Over Items         | Split In Batches        | Controls flow for post-upload steps     | Create MCQ Quizzes         | Redirect to Google Form / Create MCQ Quizzes |                                                                                              |
| Redirect to Google Form | Form Completion (Webhook) | Redirects user to completed Google Form | Loop Over Items            | None                       |                                                                                              |
| Error Output            | Form Completion (Webhook) | Displays error message on failure        | Multiple error branches    | None                       | Default output for error handling.                                                           |
| Sticky Note             | Sticky Note             | Notes and explanations                   | N/A                        | N/A                        | Multiple sticky notes distributed per logical block as described above                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form with fields:  
     - Form Name (text, required)  
     - Email (email, optional)  
     - YouTube URL (text, required)  
     - Number of Questions (number, required)  
   - Set webhook ID or leave for automatic generation  
   - Add form description noting limits (max 90 questions, max video length 50 minutes)  
   - Name node "Input YouTube URL"

2. **Add If Node "filter"**  
   - Check if `Number of Questions` < 91  
   - If true, connect to next step  
   - If false, connect to error output node (created later)

3. **Add Set Node "Set Prompt and Model"**  
   - Assign fields:  
     - `prompt`: "Generate {{ $json['Number of Questions'] }} MCQ (Multiple Choice Question) from the video with answer, every question will have 3 options and return those question as json format."  
     - `model`: "gemini-2.5-flash"  
   - Include all other incoming fields for context  
   - Connect input from 'filter' node (true branch)

4. **Add HTTP Request Node "HTTP Request to Gemini"**  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/{{ $json.model }}:generateContent?key=API_KEY` (replace API_KEY with your Google API key)  
   - Headers: Content-Type: application/json  
   - JSON Body:  
     ```json
     {
       "contents": [
         {
           "parts": [
             { "text": {{ JSON.stringify($json.prompt) }} },
             { "file_data": { "file_uri": "{{ $json['Youtube URL'] }}" } }
           ]
         }
       ]
     }
     ```  
   - Set timeout to a large value (e.g., 100000000 ms)  
   - Connect input from 'Set Prompt and Model'

5. **Add If Node "If: Was an error detected?"**  
   - Condition: Check if `$json.candidates[0].content.parts[0].text` exists  
   - True branch connects to next step  
   - False branch connects to error output

6. **Add Set Node "Set Fields: Define Outcome"**  
   - Assign fields:  
     - `answerAIGenerated`: Set to first candidate text part  
     - `promptTokenCount`, `candidatesTokenCount`, `totalTokenCount`, `modelVersionUsed`: Extract from usageMetadata or fallback error messages  
   - Exclude raw candidates and usageMetadata fields from output  
   - Connect input from true branch of error check

7. **Add Chain LLM Node "Extract JSON"**  
   - Use LangChain output parser to extract JSON from AI text  
   - Provide JSON schema example for MCQ questions:  
     ```json
     {
       "Questions": [
         {
           "question": "Example question?",
           "options": ["Option1", "Option2", "Option3"],
           "answer": "CorrectOption"
         }
       ]
     }
     ```  
   - Connect input from 'Set Fields: Define Outcome'

8. **Add HTTP Request Node "Create a Google Form"**  
   - Method: POST  
   - URL: https://forms.googleapis.com/v1/forms  
   - Auth: OAuth2 (Google Forms API credentials)  
   - Headers: Content-Type: application/json  
   - Body:  
     ```json
     {
       "info": {
         "title": "{{ $node[\"Input YouTube URL\"].json['Form Name'] }}",
         "documentTitle": "{{ $node[\"Input YouTube URL\"].json['Form Name'] }}-{{ $now }}"
       }
     }
     ```  
   - Connect input from 'Extract JSON'

9. **Add Code Node "Prepare Questions for API call"**  
   - JavaScript code to iterate over extracted questions and prepare batchUpdate requests compatible with Google Forms API, including quiz settings and grading  
   - Refer to provided code logic in analysis  
   - Connect input from 'Create a Google Form'

10. **Add HTTP Request Node "Create MCQ Quizzes"**  
    - Method: POST  
    - URL: https://forms.googleapis.com/v1/forms/{{ formId }}:batchUpdate  
    - Auth: OAuth2 (Google Forms API)  
    - Headers: Content-Type: application/json  
    - Body:  
      ```json
      {
        "requests": {{ JSON.stringify($json.data) }}
      }
      ```  
    - Enable retry on failure (max 5 attempts)  
    - Connect input from 'Prepare Questions for API call'

11. **Add Split In Batches Node "Loop Over Items"**  
    - Input: output from 'Create MCQ Quizzes'  
    - Configure default batching (1 item at a time)  
    - Connect first output to 'Redirect to Google Form'  
    - Connect second output back to 'Create MCQ Quizzes' for loop processing (if necessary)

12. **Add Form Completion Node "Redirect to Google Form"**  
    - Webhook node to complete form response and redirect user  
    - Redirect URL set dynamically from created form's responderUri  
    - Connect input from 'Loop Over Items'

13. **Add Form Completion Node "Error Output"**  
    - Webhook node to display error message  
    - Title: "Something went wrong!"  
    - Message: "Please check your Youtube URL OR the number of questions."  
    - Connect all error branches from previous nodes (filter fail, API errors, parsing failures, etc.)

14. **Connect all nodes following the logical flow** as per the analysis connections.

15. **Configure Credentials:**
    - Google Gemini API key in 'HTTP Request to Gemini' URL (replace placeholder)  
    - Google OAuth2 credentials for Google Forms API in 'Create a Google Form' and 'Create MCQ Quizzes' nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow input form URL example: https://youtu.be/cZQPDLgPtNg?si=Ze6zOu5oTpF8QMq0                                            | Provided in 'Input YouTube URL' sticky note                                                    |
| Limitations: Maximum 90 questions, video length up to 50 minutes                                                             | Specified in form description and filter node                                                  |
| Google Gemini AI model used: "gemini-2.5-flash"                                                                              | Model selection in prompt setup                                                                |
| For Google Forms API OAuth2 setup, ensure scopes include forms and forms.responses.write                                     | Google Forms API documentation: https://developers.google.com/forms/api/guides/overview         |
| The quiz questions support option shuffling and custom feedback on correct/incorrect answers                                  | Implemented in the code node preparing API calls                                               |
| Error handling routes users to a completion webhook with an informative message                                              | Provides graceful fallback for API or input errors                                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---