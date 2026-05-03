Evaluation Metric: Summarization

https://n8nworkflows.xyz/workflows/evaluation-metric--summarization-4428


# Evaluation Metric: Summarization

### 1. Workflow Overview

This workflow is designed to evaluate the quality of AI-generated summaries based on transcript data, specifically focusing on summarization accuracy and faithfulness. It targets scenarios where AI models produce summaries from video transcripts (e.g., YouTube videos) stored remotely, such as in Google Drive. The evaluation metric assesses how well the summaries reflect the source content without hallucination or omission.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Triggering:** Receives evaluation triggers and inputs, such as dataset rows or webhook calls containing Google Drive URLs pointing to transcript files.
- **1.2 Transcript Retrieval and Preparation:** Downloads the transcript file from Google Drive, extracts text content, and prepares it for summarization.
- **1.3 AI Summarization Generation:** Uses AI language models (OpenAI GPT-4.1-mini) to generate a summary of the transcript.
- **1.4 Evaluation of the Summary:** Evaluates the AI-generated summary against the transcript using a Google Gemini model to assign a quality rating based on defined criteria.
- **1.5 Output Handling and Metrics Recording:** Stores evaluation results and metrics back into a Google Sheet for record-keeping and further analysis.
- **1.6 Control and Conditional Flow:** Checks if the evaluation should proceed and handles responses accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

**Overview:**  
This block handles the initiation of the workflow by receiving triggers when new data is available for evaluation, either from a dataset row (Google Sheets) or via a webhook containing a Google Drive URL to a transcript file.

**Nodes Involved:**  
- When fetching a dataset row  
- Webhook  
- Get Gdrive URL  

**Node Details:**

- **When fetching a dataset row**  
  - *Type:* Evaluation Trigger (n8n-nodes-base.evaluationTrigger)  
  - *Role:* Automatically triggers the workflow when a new row is fetched from a specified Google Sheet.  
  - *Configuration:* Connected to a Google Sheet with ID `1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y`, sheet tab `82338773` ("Summarization").  
  - *Input/Output:* Outputs row data to downstream nodes.  
  - *Edge cases:* Google Sheets API rate limits or authentication errors may occur.

- **Webhook**  
  - *Type:* Webhook (n8n-nodes-base.webhook)  
  - *Role:* Receives external HTTP requests with data, such as a Google Drive URL.  
  - *Configuration:* Path set to a unique ID `088cd101-9dbc-46f2-899a-d2d91c15c1e5`.  
  - *Input/Output:* Passes the incoming HTTP body to the next node.  
  - *Edge cases:* Unauthorized or malformed requests.

- **Get Gdrive URL**  
  - *Type:* Set (n8n-nodes-base.set)  
  - *Role:* Extracts the Google Drive URL from the webhook payload and stores it as `input` in JSON for downstream use.  
  - *Configuration:* Sets `input` field to `{{$json.body.gdrive_url}}`.  
  - *Input/Output:* Receives webhook data; outputs formatted JSON.  
  - *Edge cases:* Missing or invalid URL in the request payload.

---

#### 2.2 Transcript Retrieval and Preparation

**Overview:**  
This block downloads the transcript file from Google Drive using the provided URL, extracts the textual content, and prepares it for summarization.

**Nodes Involved:**  
- Download Transcript  
- Extract from File  

**Node Details:**

- **Download Transcript**  
  - *Type:* Google Drive (n8n-nodes-base.googleDrive)  
  - *Role:* Downloads the transcript file referenced by the Google Drive file ID.  
  - *Configuration:* File ID is dynamically set from the `input` JSON (extracted Google Drive file URL).  
  - *Credentials:* Uses OAuth2 credentials for Google Drive API access.  
  - *Input/Output:* Receives `input` with file ID, outputs file binary data.  
  - *Edge cases:* File not found, permission denied, or download failures.

- **Extract from File**  
  - *Type:* Extract from File (n8n-nodes-base.extractFromFile)  
  - *Role:* Extracts text content from the downloaded file (e.g., transcript text).  
  - *Configuration:* Operation set to "text" extraction.  
  - *Input/Output:* Receives file binary, outputs text data under `data`.  
  - *Edge cases:* Unsupported file formats, extraction errors.

---

#### 2.3 AI Summarization Generation

**Overview:**  
This block sends the extracted transcript text to an AI language model to generate a summary of the top highlights.

**Nodes Involved:**  
- Summarise Agent  
- OpenAI Chat Model  

**Node Details:**

- **Summarise Agent**  
  - *Type:* Chain LLM (LangChain)  
  - *Role:* Sends the transcript text as input to the chained OpenAI model to generate a summary.  
  - *Configuration:* Prompt message: “Summarise the top 5 highlights of this video using the provided transcript.” Uses the `data` field from the extracted text.  
  - *Input/Output:* Inputs transcript text, outputs summary text.  
  - *Edge cases:* API timeouts, model errors, token limits.

- **OpenAI Chat Model**  
  - *Type:* OpenAI Chat Model (LangChain)  
  - *Role:* The actual OpenAI GPT-4.1-mini LLM used for summarization.  
  - *Configuration:* Model set to `gpt-4.1-mini`.  
  - *Credentials:* Requires OpenAI API key configured.  
  - *Input/Output:* Receives prompt from Summarise Agent, outputs generated summary.  
  - *Edge cases:* API rate limits, authentication failures, model unavailability.

---

#### 2.4 Evaluation of the Summary

**Overview:**  
This block evaluates the AI-generated summary against the transcript using a Google Gemini model, assigning a rating with detailed reasoning based on predefined evaluation criteria and rubric.

**Nodes Involved:**  
- Is Evaluating?  
- Evaluate Summarisation  
- LLM (Google Gemini)  
- Output (Structured Output Parser)  

**Node Details:**

- **Is Evaluating?**  
  - *Type:* Evaluation (n8n-nodes-base.evaluation)  
  - *Role:* Checks if the workflow is in evaluation mode before proceeding.  
  - *Configuration:* Operation set to `checkIfEvaluating`.  
  - *Input/Output:* Receives summary, conditionally passes to evaluation or responds to user.  
  - *Edge cases:* Workflow misconfiguration or evaluation state errors.

- **Evaluate Summarisation**  
  - *Type:* Chain LLM (LangChain)  
  - *Role:* Core evaluator chain that inputs the transcript and AI summary, runs evaluation prompt to score the summary.  
  - *Configuration:* Uses a detailed evaluation prompt instructing to assess instruction following, groundedness, conciseness, and fluency, with a numeric rating and explanation.  
  - *Input/Output:* Inputs transcript and summary texts; outputs structured evaluation JSON including `rating` and `reason`.  
  - *Edge cases:* Model hallucination, prompt parsing errors.

- **LLM (Google Gemini)**  
  - *Type:* Google Gemini (PaLM) Chat Model (LangChain)  
  - *Role:* Language model used for evaluation scoring.  
  - *Configuration:* Model set to `models/gemini-2.0-flash`.  
  - *Credentials:* Uses Google PaLM API credentials.  
  - *Input/Output:* Receives evaluation prompt, outputs rating and reasoning.  
  - *Edge cases:* API limits, authentication, latency.

- **Output**  
  - *Type:* Structured Output Parser (LangChain)  
  - *Role:* Parses evaluation model’s raw output into structured JSON fields (`rating`, `reason`).  
  - *Configuration:* JSON schema defines fields:  
    ```json
    {
      "rating": 1,
      "reason": "Tell me the reason for being lonely"
    }
    ```  
  - *Input/Output:* Inputs raw text from LLM, outputs structured JSON.  
  - *Edge cases:* Parsing errors if output format deviates.

---

#### 2.5 Output Handling and Metrics Recording

**Overview:**  
This block updates the Google Sheet with evaluation scores and reasons, then sets numeric metrics for further processing or reporting.

**Nodes Involved:**  
- Set Outputs  
- Set Metrics  
- Respond to User  

**Node Details:**

- **Set Outputs**  
  - *Type:* Evaluation (n8n-nodes-base.evaluation)  
  - *Role:* Writes evaluation outputs (`score`, `score_reason`) back into the Google Sheet in the same row used as input.  
  - *Configuration:* Maps `score` to `{{$json.output.score}}` and `score_reason` to `{{$json.output.reason}}`. Uses same Google Sheet and tab as input.  
  - *Credentials:* Google Sheets OAuth2 credentials.  
  - *Edge cases:* Google Sheets write permission errors, concurrency issues.

- **Set Metrics**  
  - *Type:* Evaluation (n8n-nodes-base.evaluation)  
  - *Role:* Sets numeric evaluation metrics internally for the workflow or external reporting.  
  - *Configuration:* Assigns metric `score` with value from `{{$json.output.rating}}` (numeric rating from evaluation).  
  - *Edge cases:* Metric assignment failures.

- **Respond to User**  
  - *Type:* NoOp (n8n-nodes-base.noOp)  
  - *Role:* Placeholder node to handle cases where evaluation is not performed.  
  - *Configuration:* No operation; used for workflow branching.  
  - *Edge cases:* None.

---

### 3. Summary Table

| Node Name              | Node Type                              | Functional Role                            | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                                                        |
|------------------------|--------------------------------------|--------------------------------------------|---------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When fetching a dataset row | Evaluation Trigger (n8n-nodes-base.evaluationTrigger) | Triggers workflow on new dataset row         | —                         | Download Transcript        | See Sticky Note3: Demonstrates summarization evaluation workflow using transcript and scoring approach.                          |
| Webhook                | Webhook (n8n-nodes-base.webhook)     | Receives external HTTP input with GDrive URL | —                         | Get Gdrive URL             | See Sticky Note1: Setup your AI workflow to use evaluations and input transcript file URLs from Google Drive.                   |
| Get Gdrive URL         | Set (n8n-nodes-base.set)              | Extracts Google Drive URL from webhook payload | Webhook                   | Download Transcript        | See Sticky Note1                                                                                                                  |
| Download Transcript    | Google Drive (n8n-nodes-base.googleDrive) | Downloads transcript file from Google Drive   | When fetching a dataset row, Get Gdrive URL | Extract from File         | See Sticky Note1                                                                                                                  |
| Extract from File      | Extract from File (n8n-nodes-base.extractFromFile) | Extracts text content from downloaded file    | Download Transcript        | Summarise Agent            | See Sticky Note1                                                                                                                  |
| Summarise Agent        | Chain LLM (LangChain)                 | Generates summary from transcript text         | Extract from File, OpenAI Chat Model | Is Evaluating?             | See Sticky Note2: Evaluates if AI summary is grounded in source material.                                                        |
| OpenAI Chat Model      | OpenAI Chat Model (LangChain)         | Provides GPT-4.1-mini model for summarization | Summarise Agent           | Summarise Agent            | See Sticky Note2                                                                                                                  |
| Is Evaluating?         | Evaluation (n8n-nodes-base.evaluation) | Checks evaluation mode status                   | Summarise Agent            | Evaluate Summarisation, Respond to User | See Sticky Note2                                                                                                                  |
| Evaluate Summarisation | Chain LLM (LangChain)                 | Evaluates summary quality vs. transcript using Gemini model | Is Evaluating?             | Set Outputs                | See Sticky Note2                                                                                                                  |
| LLM                    | Google Gemini (PaLM) Chat Model (LangChain) | Language model used for evaluation scoring      | Evaluate Summarisation     | Evaluate Summarisation     | See Sticky Note2                                                                                                                  |
| Output                 | Structured Output Parser (LangChain) | Parses evaluation model's textual output to JSON | Evaluate Summarisation     | Evaluate Summarisation     | See Sticky Note2                                                                                                                  |
| Set Outputs            | Evaluation (n8n-nodes-base.evaluation) | Writes evaluation score and reason to Google Sheet | Evaluate Summarisation     | Set Metrics                | See Sticky Note2                                                                                                                  |
| Set Metrics            | Evaluation (n8n-nodes-base.evaluation) | Sets numeric evaluation metrics internally       | Set Outputs                | —                         | See Sticky Note2                                                                                                                  |
| Respond to User        | NoOp (n8n-nodes-base.noOp)            | Handles workflow branch when not evaluating      | Is Evaluating?             | —                         | See Sticky Note2                                                                                                                  |
| Sticky Note1           | Sticky Note                          | Setup instructions and overview                 | —                         | —                         | Setup AI workflow with Evaluations, input transcript from Google Drive URL.                                                     |
| Sticky Note2           | Sticky Note                          | Explanation of summarization evaluation metric   | —                         | —                         | Evaluates AI summary grounding and alignment with transcript using scoring rubric.                                              |
| Sticky Note3           | Sticky Note                          | Introductory notes on workflow, scoring approach | —                         | —                         | Explains the evaluation approach adapted from Google Vertex AI metrics template and provides helpful links.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node - “When fetching a dataset row”**  
   - Type: Evaluation Trigger  
   - Configure Google Sheets OAuth2 credentials.  
   - Set Sheet ID to `1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y` and Sheet Name/Tab to `82338773` ("Summarization").  

2. **Create Webhook Node - “Webhook”**  
   - Type: Webhook  
   - Set webhook path to `088cd101-9dbc-46f2-899a-d2d91c15c1e5`.  
   - No authentication required unless desired for security.  

3. **Create Set Node - “Get Gdrive URL”**  
   - Type: Set  
   - Configure to assign a new field `input` with value `{{$json.body.gdrive_url}}`.  
   - Connect “Webhook” node output to this node.  

4. **Create Google Drive Node - “Download Transcript”**  
   - Type: Google Drive  
   - Operation: Download  
   - Set File ID to `{{$json.input}}` (dynamic from previous node).  
   - Configure Google Drive OAuth2 credentials.  
   - Connect outputs from “When fetching a dataset row” and “Get Gdrive URL” to this node (both trigger transcript download from either source).  

5. **Create Extract from File Node - “Extract from File”**  
   - Type: Extract from File  
   - Operation: Text  
   - Connect “Download Transcript” output to this node.  

6. **Create Chain LLM Node - “OpenAI Chat Model”**  
   - Type: LangChain OpenAI Chat Model  
   - Model: Set to `gpt-4.1-mini`.  
   - Configure OpenAI API Key credentials.  

7. **Create Chain LLM Node - “Summarise Agent”**  
   - Type: LangChain Chain LLM  
   - Prompt message: “Summarise the top 5 highlights of this video using the provided transcript.”  
   - Input text: `{{$json.data}}` from “Extract from File”.  
   - Connect “OpenAI Chat Model” as the AI language model node inside this chain.  
   - Connect “Extract from File” output to this node input.  

8. **Create Evaluation Node - “Is Evaluating?”**  
   - Type: Evaluation  
   - Operation: Check if evaluating mode is active.  
   - Connect output of “Summarise Agent” to this node.  

9. **Create Chain LLM Node - “LLM” (Google Gemini)**  
   - Type: LangChain Google Gemini (PaLM) Chat Model  
   - Model: `models/gemini-2.0-flash`.  
   - Configure Google PaLM API credentials.  

10. **Create Chain LLM Node - “Evaluate Summarisation”**  
    - Type: LangChain Chain LLM  
    - Configure prompt with detailed instructions for evaluation: include transcript and AI summary, evaluation criteria, rubric, and step-by-step rating explanation.  
    - Enable structured output parsing with JSON schema: `{ "rating": 1, "reason": "Explain reason" }`.  
    - Use “LLM” node as the model inside this chain.  
    - Connect “Is Evaluating?” node output (evaluation path) to this node.  

11. **Create Evaluation Node - “Set Outputs”**  
    - Type: Evaluation  
    - Configure to write outputs `score` and `score_reason` back to the Google Sheet (`1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y` tab `82338773`).  
    - Map `score` to `{{$json.output.score}}` and `score_reason` to `{{$json.output.reason}}`.  
    - Connect output of “Evaluate Summarisation” node to this node.  

12. **Create Evaluation Node - “Set Metrics”**  
    - Type: Evaluation  
    - Operation: Set Metrics  
    - Assign metric named `score` with value `{{$json.output.rating}}` (numeric rating).  
    - Connect output of “Set Outputs” node to this node.  

13. **Create NoOp Node - “Respond to User”**  
    - Type: NoOp  
    - Connect “Is Evaluating?” node’s non-evaluation branch to this node for graceful handling of non-evaluation cases.  

14. **Connect Workflow Nodes**  
    - “When fetching a dataset row” → “Download Transcript”  
    - “Webhook” → “Get Gdrive URL” → “Download Transcript”  
    - “Download Transcript” → “Extract from File” → “Summarise Agent”  
    - “Summarise Agent” → “Is Evaluating?”  
      - Evaluation branch → “Evaluate Summarisation” → “Set Outputs” → “Set Metrics”  
      - Non-evaluation branch → “Respond to User”  

15. **Add Sticky Notes** (optional for documentation inside n8n)  
    - Add notes with the content from Sticky Note1, Sticky Note2, and Sticky Note3 for user guidance and context.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Setup your AI workflow to use Evaluations and input transcript file URLs from Google Drive. | [n8n Evaluations Trigger Documentation](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) |
| Explanation of summarization evaluation metric focusing on AI summary grounding and alignment with transcript. | [n8n Evaluations Node Documentation](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluation) |
| Workflow adapted from Google Vertex AI generative AI metrics template for summarization quality. | [Google Vertex AI Metrics Templates](https://cloud.google.com/vertex-ai/generative-ai/docs/models/metrics-templates#pointwise_summarization_quality) |
| Join n8n community for help and examples. | [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/) |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.