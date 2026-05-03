Practice Job Interviews with Voice-based Google Gemini AI Interviewer

https://n8nworkflows.xyz/workflows/practice-job-interviews-with-voice-based-google-gemini-ai-interviewer-6614


# Practice Job Interviews with Voice-based Google Gemini AI Interviewer

---

### 1. Workflow Overview

This workflow, titled **"Practice Job Interviews with Voice-based Google Gemini AI Interviewer"**, is designed to provide a dynamic AI-powered job interview experience via voice interaction. It targets users preparing for technical job interviews by simulating a professional recruiter who asks tailored, context-aware questions based on the candidate's resume, job description, and previous interactions.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Receives incoming HTTP POST requests from a frontend UI that captures user data such as resume text, job description, and conversation history.

- **1.2 Initial vs. Follow-up Question Determination:** Uses conditional logic to differentiate between the first interview question and subsequent follow-ups.

- **1.3 Prompt Construction:** Generates customized prompts for the AI based on the interview stage—either an initial question prompt or a follow-up question prompt incorporating conversation history.

- **1.4 AI Processing (Google Gemini Chat Model):** Sends the constructed prompt to the Google Gemini Large Language Model (LLM) to generate an appropriate interview question.

- **1.5 Response Preparation and Delivery:** Formats the AI’s output and updated conversation history into a JSON response, which is sent back to the frontend UI.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming POST requests containing the candidate’s resume, job description, and conversation data. It serves as the workflow’s entry point.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - *Type:* HTTP Webhook  
    - *Role:* Receives POST requests from the frontend UI.  
    - *Configuration:*  
      - Path: `df27691d-564e-4207-8d9f-26a239f7a410` (unique webhook endpoint)  
      - HTTP Method: POST  
      - Response Mode: Respond via "Respond to Webhook" node (deferred response)  
    - *Input/Output:*  
      - Input: External HTTP POST request containing JSON body with keys like `resumeText`, `jobDescription`, `jobTitle`, `conversationHistory`, `lastQuestion`, and `answer`.  
      - Output: Forwards received JSON to the next node (`If`).  
    - *Edge Cases:*  
      - Missing or malformed JSON body could cause failures downstream.  
      - Unauthorized or unexpected requests are not explicitly handled here.

---

#### 2.2 Initial vs. Follow-up Question Determination

- **Overview:**  
  Determines if the incoming request is the start of a new interview or a continuation, based on whether `conversationHistory` is empty.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - *Type:* Conditional node  
    - *Role:* Checks if the `conversationHistory` field in the request body is empty.  
    - *Configuration:*  
      - Condition: Checks if `{{ $json.body.conversationHistory }}` is empty (string empty check, case-sensitive, strict type).  
      - If true (empty): path for initial question prompt.  
      - Else (non-empty): path for follow-up question prompt.  
    - *Input/Output:*  
      - Input: JSON from Webhook node.  
      - Output: Branch 1 (true) to "Create Initial Prompt" node; Branch 2 (false) to "Create Follow-up Prompt" node.  
    - *Edge Cases:*  
      - If `conversationHistory` field is missing (not just empty), may evaluate as empty or cause expression errors depending on data.  
      - String-based empty check requires strict data format.

---

#### 2.3 Prompt Construction

- **Overview:**  
  Builds the AI prompt for the Google Gemini model, either for the initial interview question or a follow-up question based on conversation history.

- **Nodes Involved:**  
  - Create Initial Prompt  
  - Create Follow-up Prompt  
  - Merge

- **Node Details:**

  - **Create Initial Prompt**  
    - *Type:* Set node  
    - *Role:* Constructs the initial AI prompt using candidate resume, job description, and job title.  
    - *Configuration:*  
      - Sets a string `prompt` with embedded expressions:  
        - Injects `jobTitle`, `jobDescription`, and `resumeText` from the request body.  
        - Instruction to ask a single impactful opening question without filler.  
    - *Input/Output:*  
      - Input: From If node (empty conversation history path).  
      - Output: Passes prompt to "Merge" node.  
    - *Edge Cases:*  
      - Missing or malformed `resumeText` or `jobDescription` may result in incomplete prompt.

  - **Create Follow-up Prompt**  
    - *Type:* Set node  
    - *Role:* Constructs a follow-up AI prompt including conversation history, last question, and user answer.  
    - *Configuration:*  
      - Sets string `prompt` embedding:  
        - `jobTitle`, `conversationHistory`, `lastQuestion`, and `answer` fields from request.  
        - Instructions to ask the next logical follow-up question without repetition or filler.  
    - *Input/Output:*  
      - Input: From If node (non-empty conversation history path).  
      - Output: Passes prompt to "Merge" node.  
    - *Edge Cases:*  
      - If any of the fields (`conversationHistory`, `lastQuestion`, `answer`) are missing or malformed, prompt clarity may suffer.

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines the two possible prompt paths into a single stream for AI processing.  
    - *Configuration:* Default merge (no special options).  
    - *Input/Output:*  
      - Inputs: Receives from both "Create Initial Prompt" (main input 0) and "Create Follow-up Prompt" (main input 1).  
      - Output: Passes merged prompt data to "Basic LLM Chain" node.  
    - *Edge Cases:*  
      - Unexpected or simultaneous inputs on both paths could cause race conditions, but here only one path is active per execution.

---

#### 2.4 AI Processing (Google Gemini Chat Model)

- **Overview:**  
  Sends the constructed prompt to the Google Gemini LLM to generate a relevant interview question.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Basic LLM Chain

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type:* Language Model (Google Gemini via LangChain integration)  
    - *Role:* Invokes the Google Gemini API to generate AI responses.  
    - *Configuration:*  
      - Uses stored Google Gemini API credential (`Google Gemini(PaLM) Api account`).  
      - No custom parameters provided; defaults used.  
    - *Input/Output:*  
      - Input: Receives prompt from "Merge" node via chaining with "Basic LLM Chain".  
      - Output: Provides raw AI-generated text to "Basic LLM Chain".  
    - *Edge Cases:*  
      - API errors: authentication failure, rate limits, network timeouts.  
      - Unexpected model responses or malformed output.

  - **Basic LLM Chain**  
    - *Type:* LangChain LLM Chain node  
    - *Role:* Processes the prompt text and manages interaction with the AI model node.  
    - *Configuration:*  
      - Accepts `prompt` text from previous nodes.  
      - Outputs the AI-generated answer as `text` in JSON.  
    - *Input/Output:*  
      - Input: Merged prompt from "Merge" node; AI model via "Google Gemini Chat Model".  
      - Output: Passes AI response text to "Prepare Response Data" node.  
    - *Edge Cases:*  
      - Expression errors if prompt text is missing or malformed.  
      - Latency due to API call duration.

---

#### 2.5 Response Preparation and Delivery

- **Overview:**  
  Formats AI output and conversation data into a structured JSON response and sends it back to the frontend.

- **Nodes Involved:**  
  - Prepare Response Data  
  - Respond to Webhook

- **Node Details:**

  - **Prepare Response Data**  
    - *Type:* Set node  
    - *Role:* Creates response JSON with the new AI question and updates conversation history.  
    - *Configuration:*  
      - Assigns:  
        - `newQuestion`: the AI-generated text from "Basic LLM Chain".  
        - `updatedHistory`: concatenates previous conversation history (if any) with the new AI question, formatted with "AI:" and "User:" labels for frontend parsing.  
    - *Input/Output:*  
      - Input: JSON from "Basic LLM Chain" and original request from "Webhook".  
      - Output: Passes structured JSON to "Respond to Webhook".  
    - *Edge Cases:*  
      - If conversation history is undefined, initializes as empty string to avoid errors.  
      - Formatting errors may impact frontend parsing.

  - **Respond to Webhook**  
    - *Type:* Respond to Webhook node  
    - *Role:* Sends the JSON response back to the caller of the webhook.  
    - *Configuration:*  
      - Responds with JSON: keys `question` and `conversationHistory`.  
      - Uses data from "Prepare Response Data".  
    - *Input/Output:*  
      - Input: JSON with keys `newQuestion` and `updatedHistory`.  
      - Output: HTTP response to original POST request.  
    - *Edge Cases:*  
      - Network issues could prevent delivery.  
      - Malformed JSON could cause frontend errors.

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                                | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                                                                                                                                   |
|-----------------------|--------------------------------|-----------------------------------------------|-----------------------|------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook               | Webhook                       | Entry point; receives POST request from UI    | —                     | If                     | START HERE: This workflow starts when it receives a POST request from the frontend UI. It checks if it's the first question or a follow-up.                                                                                   |
| If                    | If                           | Determines if first or follow-up question     | Webhook                | Create Initial Prompt, Create Follow-up Prompt | START HERE: This workflow starts when it receives a POST request from the frontend UI. It checks if it's the first question or a follow-up.                                                                                   |
| Create Initial Prompt  | Set                          | Builds initial interview question prompt      | If (true branch)       | Merge                  | 1. First Question: If it's the start of the interview, this node creates the initial prompt for the AI, using the resume text and job description sent from the UI.                                                            |
| Create Follow-up Prompt| Set                          | Builds follow-up interview question prompt    | If (false branch)      | Merge                  | 2. Follow-up Question: For all subsequent turns, this node creates a different prompt that includes the conversation history, asking the AI for a logical follow-up question.                                                   |
| Merge                 | Merge                        | Combines initial and follow-up prompts        | Create Initial Prompt, Create Follow-up Prompt | Basic LLM Chain        |                                                                                                                                                                                                                               |
| Google Gemini Chat Model| Language Model (Google Gemini) | Generates AI question using Google Gemini LLM| Basic LLM Chain (ai_languageModel input) | Basic LLM Chain (ai_languageModel output) | 3. AI Brain (Google Gemini): The prompt is sent to the Google Gemini model to generate the interview question. Remember to add your own Google Gemini API credential here!                                                    |
| Basic LLM Chain        | LangChain LLM Chain          | Manages prompt execution and AI response      | Merge, Google Gemini Chat Model | Prepare Response Data   |                                                                                                                                                                                                                               |
| Prepare Response Data  | Set                          | Formats AI response and updates conversation  | Basic LLM Chain        | Respond to Webhook      | 4. Send Response: These nodes prepare the AI's response and the updated conversation history, sending it all back to the frontend UI in a clean JSON format.                                                                    |
| Respond to Webhook     | Respond to Webhook           | Sends JSON response back to frontend UI       | Prepare Response Data  | —                      | 4. Send Response: These nodes prepare the AI's response and the updated conversation history, sending it all back to the frontend UI in a clean JSON format.                                                                    |
| Sticky Note            | Sticky Note                  | Documentation and instructions                 | —                     | —                      | START HERE: This workflow starts when it receives a POST request from the frontend UI. It checks if it's the first question or a follow-up.                                                                                   |
| Sticky Note1           | Sticky Note                  | Documentation on first question prompt         | —                     | —                      | 1. First Question: If it's the start of the interview, this node creates the initial prompt for the AI, using the resume text and job description sent from the UI.                                                            |
| Sticky Note3           | Sticky Note                  | Documentation on follow-up question prompt     | —                     | —                      | 2. Follow-up Question: For all subsequent turns, this node creates a different prompt that includes the conversation history, asking the AI for a logical follow-up question.                                                   |
| Sticky Note2           | Sticky Note                  | Documentation on AI model usage                 | —                     | —                      | 3. AI Brain (Google Gemini): The prompt is sent to the Google Gemini model to generate the interview question. Remember to add your own Google Gemini API credential here!                                                    |
| Sticky Note4           | Sticky Note                  | Documentation on response preparation           | —                     | —                      | 4. Send Response: These nodes prepare the AI's response and the updated conversation history, sending it all back to the frontend UI in a clean JSON format.                                                                    |
| Sticky Note5           | Sticky Note                  | Full workflow explanation, usage instructions  | —                     | —                      | What does this workflow do? This workflow acts as the backend "brain" for a sophisticated AI Voice Interviewer. It receives a user's resume text and a target job description, then uses a Large Language Model (LLM) to conduct a realistic interview. <br> This template is designed to work with a simple HTML frontend that handles voice-to-text and text-to-speech. <br> What services does this workflow use? Google Gemini: This is the LLM used to generate intelligent interview questions. You can swap it out for others like OpenAI. <br> What credentials do you need? Google Gemini API Key from Google AI Studio. <br> Usage: Set up frontend, add Gemini credential, activate workflow, connect frontend via webhook URL, start interviewing! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Parameters:  
     - HTTP Method: POST  
     - Path: Use a unique string (e.g., `df27691d-564e-4207-8d9f-26a239f7a410`)  
     - Response Mode: `Response Node` (to defer response)  
   - Purpose: Receive interview requests from the frontend UI.

2. **Add If Node to Determine Interview Stage**  
   - Type: If  
   - Condition: Check if `{{$json.body.conversationHistory}}` is empty (string empty check with strict type validation)  
   - True (empty): Route to initial question prompt creation  
   - False (non-empty): Route to follow-up prompt creation

3. **Create Initial Prompt Node (Set Node)**  
   - Type: Set  
   - Assign variable `prompt` as string with embedded expressions:  
     ```
     You are an expert technical recruiter named 'Alex'. You are conducting an interview for the position of '{{ $json.body.jobTitle }}'. Your task is to conduct a professional interview based *only* on the provided resume and job description.

     Here is the job description:
     """
     {{ $json.body.jobDescription }}
     """

     Here is the candidate's resume text:
     """
     {{ $json.body.resumeText }}
     """

     Ask the single, most impactful opening question. Ask only one question. Do not add any conversational filler.
     ```
   - Input: From If node (true branch)

4. **Create Follow-up Prompt Node (Set Node)**  
   - Type: Set  
   - Assign variable `prompt` as string with embedded expressions:  
     ```
     You are an expert technical recruiter named 'Alex', continuing an interview for '{{ $json.body.jobTitle }}'. The conversation history is below. Ask the next logical follow-up question based on the user's last answer. Do not repeat questions. Ask only one question.  CONVERSATION HISTORY: {{ $json.body.conversationHistory }}AI: {{ $json.body.lastQuestion }} User: {{ $json.body.answer }}
     ```
   - Input: From If node (false branch)

5. **Add Merge Node**  
   - Type: Merge  
   - Inputs: Connect outputs from both "Create Initial Prompt" and "Create Follow-up Prompt" nodes  
   - Purpose: Combine prompt streams into one for AI processing.

6. **Add Google Gemini Chat Model Node**  
   - Type: Language Model (Google Gemini via LangChain)  
   - Credentials: Set up and link Google Gemini API credential here (Google AI Studio key)  
   - Input: Connect from "Merge" node (via LangChain chain with "Basic LLM Chain")  
   - Purpose: Generate interview question based on prompt.

7. **Add Basic LLM Chain Node**  
   - Type: LangChain LLM Chain  
   - Configuration: Use default settings; accepts `prompt` and links to Google Gemini node as AI model  
   - Input: Connect from "Merge" node (main input) and Google Gemini node (ai_languageModel input)  
   - Output: AI-generated text under `text` field

8. **Add Prepare Response Data Node (Set Node)**  
   - Type: Set  
   - Assign variables:  
     - `newQuestion` = `={{ $('Basic LLM Chain').item.json.text }}`  
     - `updatedHistory` = `={{ $('Webhook').item.json.body.conversationHistory || '' }}AI: {{ $json.newQuestion }}\nUser:`  
   - Input: From "Basic LLM Chain" node

9. **Add Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Configuration:  
     - Respond With: JSON  
     - Response Body:  
       ```
       {
         "question": {{$json.newQuestion}},
         "conversationHistory": {{$json.updatedHistory}}
       }
       ```  
   - Input: From "Prepare Response Data" node

10. **Connect Nodes Appropriately:**  
    - Webhook → If  
    - If (true) → Create Initial Prompt → Merge (input 0)  
    - If (false) → Create Follow-up Prompt → Merge (input 1)  
    - Merge → Basic LLM Chain (main input)  
    - Google Gemini Chat Model → Basic LLM Chain (ai_languageModel input)  
    - Basic LLM Chain → Prepare Response Data → Respond to Webhook

11. **Credential Setup:**  
    - Create or import Google Gemini API credential in n8n credentials section  
    - Link credential to "Google Gemini Chat Model" node

12. **Activate Workflow:**  
    - Save and activate the workflow  
    - Copy Webhook production URL and configure frontend to send POST requests accordingly

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow acts as the backend "brain" for a sophisticated AI Voice Interviewer using Google Gemini LLM. It requires a separate frontend to handle voice-to-text and text-to-speech operations.                                                  | Sticky Note5 content                                                                                                 |
| Google Gemini API Key is required; available for free from Google AI Studio. You can replace the AI model with alternatives like OpenAI if desired.                                                                                                | Sticky Note5 content                                                                                                 |
| Frontend code and setup instructions are available in the referenced GitHub repository (not included here). The webhook URL from this workflow must be pasted into the frontend's configuration to enable communication.                            | Sticky Note5 content                                                                                                 |
| The workflow is designed to maintain conversation history and generate context-aware follow-up questions to simulate realistic interview dialogues.                                                                                                 | Sticky Note5 content                                                                                                 |
| For troubleshooting API errors, verify credentials, network connectivity, and observe rate limits of the Google Gemini API.                                                                                                                        | General best practice                                                                                               |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available.

---