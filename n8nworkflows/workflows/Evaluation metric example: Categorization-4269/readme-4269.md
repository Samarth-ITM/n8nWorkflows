Evaluation metric example: Categorization

https://n8nworkflows.xyz/workflows/evaluation-metric-example--categorization-4269


# Evaluation metric example: Categorization

### 1. Workflow Overview

This workflow demonstrates how to evaluate an AI-driven categorization and prioritization model applied to customer support tickets. The main goal is to classify each support ticket into predefined categories and priorities, then compare the AI-generated results against expected labels from a test dataset to calculate evaluation metrics.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming support ticket data via webhook or iterates over a test dataset.
- **1.2 AI Processing:** Uses a LangChain AI agent powered by OpenAI GPT-4o-mini to classify tickets into categories and priorities.
- **1.3 Evaluation Control:** Determines if the workflow is running in evaluation mode (test dataset) or live mode (webhook).
- **1.4 Metric Calculation:** Compares AI output against expected values and sets metric results for evaluation.
- **1.5 Response Handling:** Returns output to the webhook caller if applicable.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block handles the intake of support ticket data either from a live webhook or from a pre-existing dataset (Google Sheets). It prepares the data in the required format for the AI agent.

- **Nodes Involved:**  
  - Webhook  
  - When fetching a dataset row  
  - Match webhook format

- **Node Details:**

  1. **Webhook**  
     - *Type & Role:* HTTP webhook node to receive live incoming ticket data (subject and body).  
     - *Configuration:* Path set to a UUID-based unique string for webhook identification. No authentication configured.  
     - *Input/Output:* Receives HTTP POST/GET requests; outputs JSON with support ticket fields.  
     - *Failures:* Possible issues if webhook URL is not publicly accessible or if payload is malformed.

  2. **When fetching a dataset row**  
     - *Type & Role:* Evaluation trigger node iterating over rows in a Google Sheets document containing test support tickets with expected labels.  
     - *Configuration:* Points to a specific Google Sheets URL and sheet ID; uses OAuth2 credentials for access.  
     - *Input/Output:* Outputs each row as JSON with ticket fields and expected category/priority.  
     - *Failures:* OAuth token expiration, network errors, or sheet permission issues.

  3. **Match webhook format**  
     - *Type & Role:* Set node that transforms dataset row data into a format matching live webhook input structure.  
     - *Configuration:* Sets `query.subject` and `query.body` fields by serializing JSON from the current dataset row.  
     - *Input/Output:* Accepts dataset row input; outputs in a webhook-like structured JSON.  
     - *Failures:* Expression evaluation errors if expected fields are missing or null.

---

#### 2.2 AI Processing

- **Overview:**  
  This block runs the AI classification model that receives the subject and body of a ticket and returns a predicted category and priority in JSON format.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  1. **AI Agent**  
     - *Type & Role:* LangChain agent node orchestrating the prompt and model interaction.  
     - *Configuration:*  
       - Input text constructed from subject and body fields.  
       - System message defines classification instructions, categories, priorities, and output JSON format.  
       - Returns intermediate steps for debug/traceability.  
     - *Input/Output:* Receives formatted ticket text; outputs structured JSON with `category` and `priority`.  
     - *Expressions:* Uses `={{ $json.subject }}` and `={{ $json.body }}` to inject ticket content.  
     - *Failures:* API errors, prompt formatting issues, rate limits, or unexpected AI outputs.

  2. **OpenAI Chat Model**  
     - *Type & Role:* Language model node using GPT-4o-mini to generate classification from the prompt prepared by AI Agent.  
     - *Configuration:* Model set explicitly to "gpt-4o-mini".  
     - *Credentials:* Uses stored OpenAI API key credential.  
     - *Input/Output:* Receives prompt text; returns raw AI completion.  
     - *Failures:* Auth errors, quota exceeded, network timeouts.

  3. **Structured Output Parser**  
     - *Type & Role:* Parses AI output string to JSON based on an example schema expecting `category` and `priority` keys.  
     - *Configuration:* JSON schema example provided to guide parsing.  
     - *Input/Output:* Takes raw AI text; outputs parsed JSON object.  
     - *Failures:* Parsing errors if AI output format deviates from expected.

---

#### 2.3 Evaluation Control

- **Overview:**  
  Determines whether the workflow is running in evaluation mode (processing dataset rows) or live mode (processing webhook input), directing the flow accordingly.

- **Nodes Involved:**  
  - Evaluating?

- **Node Details:**

  1. **Evaluating?**  
     - *Type & Role:* Evaluation node checking if the workflow is in evaluation mode.  
     - *Configuration:* Operation set to `checkIfEvaluating`.  
     - *Input/Output:* Receives AI Agent output; outputs to two branches: evaluation or direct webhook response.  
     - *Failures:* Configuration errors leading to incorrect branching.

---

#### 2.4 Metric Calculation

- **Overview:**  
  Compares AI-predicted category and priority against expected values from the dataset and sets numeric metrics for evaluation.

- **Nodes Involved:**  
  - Check categorization  
  - Set metrics

- **Node Details:**

  1. **Check categorization**  
     - *Type & Role:* Set node that creates boolean flags indicating if category and priority match expected values.  
     - *Configuration:*  
       - Evaluates equality between AI output and dataset expected fields for category and priority.  
     - *Expressions:*  
       - `category_match = $json.output.category == $('When fetching a dataset row').item.json.expected_category`  
       - `priority_match = $json.output.priority == $('When fetching a dataset row').item.json.expected_priority`  
     - *Input/Output:* Receives AI output JSON and dataset expected labels; outputs booleans.  
     - *Failures:* Null or missing fields causing expression evaluation errors.

  2. **Set metrics**  
     - *Type & Role:* Evaluation node that converts boolean matches into numeric metrics for reporting.  
     - *Configuration:*  
       - Converts `category_match` and `priority_match` booleans to numbers (0 or 1).  
       - Sets these as evaluation metrics.  
     - *Input/Output:* Accepts booleans; outputs metrics.  
     - *Failures:* Type conversion errors if input is not boolean.

---

#### 2.5 Response Handling

- **Overview:**  
  Sends a response back to the caller when the workflow operates in live webhook mode.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  1. **Respond to Webhook**  
     - *Type & Role:* Sends HTTP response to the original webhook request.  
     - *Configuration:* Default response options, returning the AI classification result.  
     - *Input/Output:* Takes AI output JSON; returns HTTP response.  
     - *Failures:* Timeout or connection issues if webhook request is lost.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                        | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                              |
|-------------------------|----------------------------------|-------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Sticky Note1            | Sticky Note                      | Reminder to check category/priority |                            |                            | Check whether the category/priority matches the expected one in the dataset                            |
| Sticky Note3            | Sticky Note                      | Workflow overview and instructions  |                            |                            | ## How it works\nThis template shows how to calculate a workflow evaluation metric: **whether a category matches the expected one**.\n\nThe workflow takes support tickets and generates a category and priority, which is then compared with the correct answers in the dataset.\n\nYou can find more information on workflow evaluation [here](https://docs.n8n.io/advanced-ai/evaluations/overview), and other metric examples [here](https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/#2-calculate-metrics). |
| Sticky Note4            | Sticky Note                      | Dataset source description          |                            |                            | Read in [this test dataset](https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=294497137#gid=294497137) of support tickets |
| Webhook                 | Webhook                         | Receive live support tickets        |                            | AI Agent                   |                                                                                                        |
| When fetching a dataset row | Evaluation Trigger              | Iterate test dataset rows           |                            | Match webhook format       |                                                                                                        |
| Match webhook format    | Set                             | Format dataset row like webhook     | When fetching a dataset row | AI Agent                   |                                                                                                        |
| AI Agent                | LangChain Agent                 | Classify ticket category & priority | Webhook / Match webhook format | Evaluating?                |                                                                                                        |
| OpenAI Chat Model       | LangChain LM Chat OpenAI        | Language model for classification   | AI Agent (ai_languageModel) | AI Agent (ai_languageModel)|                                                                                                        |
| Structured Output Parser| LangChain Output Parser Structured | Parse AI output JSON                 | AI Agent (ai_outputParser)  | AI Agent                   |                                                                                                        |
| Evaluating?             | Evaluation                     | Branch flow based on evaluation mode | AI Agent                   | Check categorization / Respond to Webhook |                                                                                                  |
| Check categorization    | Set                             | Compare AI output with expected labels | Evaluating?                | Set metrics                |                                                                                                        |
| Set metrics             | Evaluation                     | Set numeric evaluation metrics      | Check categorization         |                            |                                                                                                        |
| Respond to Webhook      | Respond to Webhook              | Return classification to caller     | Evaluating?                 |                            |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook  
   - Parameters: Set path to a unique identifier (e.g., UUID)  
   - Purpose: Receive live incoming support tickets with at least `subject` and `body` fields.

2. **Create an Evaluation Trigger Node**  
   - Type: Evaluation Trigger  
   - Parameters:  
     - Document ID and Sheet Name set to the Google Sheets URL and sheet containing the test dataset.  
   - Credentials: Configure Google Sheets OAuth2 credentials with appropriate access.  
   - Purpose: Iterate over each row in the test dataset containing ticket text and expected labels.

3. **Create a Set Node (Match webhook format)**  
   - Type: Set  
   - Parameters: Use raw JSON mode to create an object matching webhook input format, using expressions:  
     ```json
     {
       "headers": {},
       "params": {},
       "query": {
         "subject": {{ $json.subject.toJsonString() }},
         "body": {{ $json.body.toJsonString() }}
       },
       "body": {},
       "executionMode": "test"
     }
     ```  
   - Purpose: Normalize dataset rows to the same format as live webhook input.

4. **Create the AI Agent Node**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text: `"Subject: {{ $json.subject }}\nBody: {{ $json.body }}"`  
     - System Message: Provide classification instructions including categories and priorities, example input/output, and output JSON format.  
     - Return intermediate steps: Enabled  
     - Prompt Type: Define  
     - Output Parser: Enabled with JSON schema example for `category` and `priority`.  
   - Purpose: Generate category and priority classification from ticket text.

5. **Create the OpenAI Chat Model Node**  
   - Type: LangChain LM Chat OpenAI  
   - Parameters:  
     - Model: `gpt-4o-mini`  
   - Credentials: Configure OpenAI API key credentials.  
   - Purpose: Underlying language model to power the AI Agent.

6. **Link OpenAI Chat Model to AI Agent**  
   - Connect OpenAI Chat Model’s `ai_languageModel` output to AI Agent’s corresponding input.

7. **Create Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Parameters: Provide JSON schema example for parsing AI output.  
   - Connect its output to AI Agent’s `ai_outputParser` input.

8. **Create Evaluation Node ‘Evaluating?’**  
   - Type: Evaluation  
   - Parameters: Operation set to `checkIfEvaluating`.  
   - Connect AI Agent output to this node.

9. **Create Set Node ‘Check categorization’**  
   - Type: Set  
   - Parameters:  
     - Create two boolean fields:  
       - `category_match` = compare AI output category to expected category from dataset row  
       - `priority_match` = compare AI output priority to expected priority from dataset row  
   - Expressions example:  
     ```js
     category_match = $json.output.category == $('When fetching a dataset row').item.json.expected_category
     priority_match = $json.output.priority == $('When fetching a dataset row').item.json.expected_priority
     ```
   - Connect ‘Evaluating?’ true branch to this node.

10. **Create Evaluation Node ‘Set metrics’**  
    - Type: Evaluation  
    - Parameters:  
      - Set metrics converting booleans to numbers:  
        - `category_match` = `category_match.toNumber()`  
        - `priority_match` = `priority_match.toNumber()`  
    - Connect ‘Check categorization’ output to this node.

11. **Create Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Parameters: Default  
    - Connect ‘Evaluating?’ false branch to this node.

12. **Connect Workflow Start Nodes**  
    - For live mode: Connect Webhook node output to AI Agent input.  
    - For evaluation mode: Connect Evaluation Trigger node output to ‘Match webhook format’ node, then to AI Agent.

13. **Add Sticky Notes**  
    - Add descriptive sticky notes with provided content near relevant nodes for clarity.

14. **Configure Credentials**  
    - OpenAI API key credential must be set up and linked to the OpenAI Chat Model node.  
    - Google Sheets OAuth2 credentials must be set up and linked to the Evaluation Trigger node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This template illustrates evaluating AI workflow metrics by checking if predicted categories match expected ones from a dataset.   | Workflow description and use case                                                                           |
| More information on workflow evaluation and other metric examples is available at:                                                   | https://docs.n8n.io/advanced-ai/evaluations/overview and https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/#2-calculate-metrics |
| The test dataset used is publicly available as a Google Sheets document of support tickets with expected categories and priorities. | https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=294497137#gid=294497137 |

---

**Disclaimer:** The provided text is generated exclusively from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All data processed are legal and publicly available.