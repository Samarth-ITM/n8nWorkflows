Food Image Analysis for Calorie Estimation with Vision AI and Telegram

https://n8nworkflows.xyz/workflows/food-image-analysis-for-calorie-estimation-with-vision-ai-and-telegram-9619


# Food Image Analysis for Calorie Estimation with Vision AI and Telegram

### 1. Workflow Overview

This workflow automates the analysis of food images sent via Telegram to estimate ingredient details and calorie content using Vision AI combined with a large language model (LLM). It is designed for users who want quick nutritional insights from food photos, supporting logging, sharing, or dietary feedback.

The workflow consists of these logical blocks:

- **1.1 Input Reception**  
  Receives a food image message from Telegram and downloads the image file.

- **1.2 AI Processing: Ingredient & Calorie Inference**  
  Uses an AI agent node powered by a vision-capable LLM to analyze the image and generate a structured JSON output detailing the dish name, ingredients (with amounts and calories), total calories, and a nutrition evaluation.

- **1.3 Structured Output Parsing**  
  Validates and parses the AI agent’s JSON output against a strict schema to ensure data integrity.

- **1.4 Email Formatting & Delivery**  
  Formats the parsed data into a styled HTML email report and sends it via Gmail to a predefined email address.

- **1.5 Error Handling & Notes**  
  Includes sticky notes throughout the workflow documenting prerequisites, expected input/output, error handling strategies, and extensibility options.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new Telegram messages containing photos, downloads the image for further processing, and passes the image file ID to the AI processing block.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Messaging trigger node for Telegram  
    - Configurations:  
      - Listens for incoming "message" updates  
      - Downloads media files automatically (`download` enabled)  
      - Uses a Telegram API credential with OAuth2 setup  
    - Key Expressions:  
      - Extracts `file_id` from `$json.message.photo[0].file_id` for the image URL reference  
    - Connections: Output connects to AI Agent node  
    - Possible Failures:  
      - Telegram API auth errors or rate limits  
      - Missing or unsupported media types in incoming messages  
      - Download failures for large or unsupported images

#### 2.2 AI Processing: Ingredient & Calorie Inference

- **Overview:**  
  This block sends the image reference to a vision-enabled AI agent that analyzes the food image, infers ingredients and calorie information, and produces a strict JSON output.

- **Nodes Involved:**  
  - AI Agent - 食材分析 (Ingredient Analysis AI Agent)  
  - OpenRouter Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: Language model node (OpenRouter gateway to OpenAI GPT-5 Mini)  
    - Configuration:  
      - Model: `openai/gpt-5-mini` (vision-capable)  
      - Temperature: 0.3 (low for stable output)  
    - Credentials: OpenRouter API key  
    - Role: Provides LLM capability for AI Agent  
    - Connects as AI language model input to AI Agent node  
    - Failure Modes: API rate limits, network timeouts, model unavailability

  - **AI Agent - 食材分析**  
    - Type: LangChain Agent Node with Vision AI and Prompt Definition  
    - Configuration:  
      - System message instructs to analyze the food image (`{{ $json.message.photo[0].file_id }}` references image file ID from Telegram)  
      - Prompt requests JSON output with fields: dishName, ingredients (name, amount, calories), totalCalories, nutritionEvaluation  
      - Output parser enabled to enforce JSON-only response  
    - Input: Receives image ID from Telegram Trigger, LLM model from OpenRouter  
    - Output: JSON structured data passed to next node  
    - Failure Modes:  
      - Model misunderstanding or incomplete output  
      - JSON parse failures if AI output is malformed  
      - Vision model limits on image size or quality

  - **Structured Output Parser**  
    - Type: LangChain structured output parser  
    - Configuration:  
      - Validates AI output against a strict JSON schema defining required fields and types (dishName, ingredients array, totalCalories, nutritionEvaluation)  
    - Input: Raw AI output JSON  
    - Output: Parsed and validated JSON or error if schema mismatch  
    - Failure Modes: Schema validation errors, missing fields, incorrect data types

#### 2.3 Email Formatting & Delivery

- **Overview:**  
  This block converts the structured AI output into a polished HTML email and sends it to a preset Gmail address.

- **Nodes Involved:**  
  - Format for Gmail (Code node)  
  - Send a message1 (Gmail node)

- **Node Details:**

  - **Format for Gmail**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Extracts the parsed AI analysis result  
      - Validates presence of required fields to avoid errors  
      - Builds a styled HTML email with:  
        - Dish name as main heading  
        - Total estimated calories highlighted  
        - Detailed ingredients table (name, amount in g, calories)  
        - Nutrition evaluation summary box  
      - Returns subject and HTML body for email node  
    - Input: Parsed JSON from Structured Output Parser  
    - Output: JSON object with `subject` and `htmlBody` fields  
    - Failure modes: Missing or malformed AI output causing fallback error message

  - **Send a message1**  
    - Type: Gmail node (OAuth2 authenticated)  
    - Configuration:  
      - Sends email to fixed recipient `t.minamig20@gmail.com`  
      - Subject and HTML body set from previous node output  
      - Attribution disabled (no signature appended)  
    - Credentials: Gmail OAuth2 account configured  
    - Failure modes: Gmail API auth failure, quota limits, network issues

#### 2.4 Error Handling & Documentation Notes

- **Overview:**  
  Multiple sticky notes throughout the workflow provide documentation on prerequisites, input formats, model prompt policies, output schema, testing steps, error handling, security, delivery options, and extensibility.

- **Nodes Involved:**  
  - Sticky 1 through Sticky 10 (Sticky Note nodes)

- **Node Details:**

  - These nodes only contain informational text to assist users in understanding and maintaining the workflow.  
  - Topics include:  
    - Workflow purpose and input/output expectations  
    - Credential and environment variable requirements  
    - Model usage policies and prompt design  
    - JSON schema validation details  
    - Error handling strategies including retries on API limits  
    - Security best practices for handling PII and API keys  
    - Alternative delivery options (e.g., Telegram reply, HTTP response)  
    - Suggestions for extending functionality (multilingual, unit normalization)  
  - These notes are not executable and do not affect workflow logic but provide crucial context.

---

### 3. Summary Table

| Node Name                | Node Type                              | Functional Role                         | Input Node(s)          | Output Node(s)            | Sticky Note                                              |
|--------------------------|--------------------------------------|---------------------------------------|------------------------|---------------------------|----------------------------------------------------------|
| Telegram Trigger         | n8n-nodes-base.telegramTrigger       | Receives Telegram image messages      | —                      | AI Agent - 食材分析       | Sticky 1, Sticky 2, Sticky 3                             |
| OpenRouter Chat Model    | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides LLM for AI agent             | —                      | AI Agent - 食材分析 (ai_languageModel) | Sticky 4                                                  |
| AI Agent - 食材分析       | @n8n/n8n-nodes-langchain.agent       | Analyzes image and infers ingredients | Telegram Trigger, OpenRouter Chat Model | Format for Gmail           | Sticky 4, Sticky 5, Sticky 7                              |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Validates AI JSON output               | AI Agent - 食材分析 (ai_outputParser) | AI Agent - 食材分析 (ai_outputParser response) | Sticky 5                                                  |
| Format for Gmail         | n8n-nodes-base.code                  | Formats HTML email from parsed data   | AI Agent - 食材分析     | Send a message1           | Sticky 6                                                  |
| Send a message1          | n8n-nodes-base.gmail                  | Sends email report via Gmail          | Format for Gmail        | —                         | Sticky 8, Sticky 9                                       |
| Sticky 1: Overview       | n8n-nodes-base.stickyNote             | Documentation overview                 | —                      | —                         | —                                                        |
| Sticky 2: Prerequisites  | n8n-nodes-base.stickyNote             | Docs: credentials and environment     | —                      | —                         | —                                                        |
| Sticky 3: Input Format   | n8n-nodes-base.stickyNote             | Docs: expected input JSON format       | —                      | —                         | —                                                        |
| Sticky 4: Model & Prompt | n8n-nodes-base.stickyNote             | Docs: model and prompt usage           | —                      | —                         | —                                                        |
| Sticky 5: Output Schema  | n8n-nodes-base.stickyNote             | Docs: JSON schema and validation       | —                      | —                         | —                                                        |
| Sticky 6: Test Steps     | n8n-nodes-base.stickyNote             | Docs: testing instructions             | —                      | —                         | —                                                        |
| Sticky 7: Errors & Limits| n8n-nodes-base.stickyNote             | Docs: error handling and limits        | —                      | —                         | —                                                        |
| Sticky 8: Security       | n8n-nodes-base.stickyNote             | Docs: security & PII management        | —                      | —                         | —                                                        |
| Sticky 9: Delivery       | n8n-nodes-base.stickyNote             | Docs: alternative delivery methods     | —                      | —                         | —                                                        |
| Sticky 10: Extensibility | n8n-nodes-base.stickyNote             | Docs: future extension ideas           | —                      | —                         | —                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Node Type: `telegramTrigger`  
   - Parameters:  
     - Updates: `message`  
     - Additional Fields: Enable `download` to true  
   - Credentials: Connect to Telegram API via OAuth2 credentials  
   - Position: Starting node  
   - Purpose: Listens for incoming Telegram messages with photos, automatically downloads image files.

2. **Create OpenRouter Chat Model Node**  
   - Node Type: `lmChatOpenRouter` under LangChain nodes  
   - Parameters:  
     - Model: `openai/gpt-5-mini`  
     - Temperature: `0.3` (low for stable output)  
   - Credentials: OpenRouter API key (configured in n8n credentials)  
   - Connect: No input; output connects to AI Agent's AI language model input.

3. **Create AI Agent Node ("AI Agent - 食材分析")**  
   - Node Type: `langchain.agent`  
   - Parameters:  
     - Prompt: Japanese instructions to analyze the provided food image URL (image file ID from Telegram message)  
     - System Message: Detailed instructions for JSON-only output with dishName, ingredients (name, amount in g, calories), totalCalories, and nutritionEvaluation  
     - Output Parser: Enable with strict JSON schema matching expected output  
   - Inputs:  
     - Connect main input from Telegram Trigger node (for image file ID)  
     - Connect AI language model input from OpenRouter Chat Model node  
   - Outputs: Pass parsed JSON result to the next node.

4. **Create Structured Output Parser Node**  
   - Node Type: `langchain.outputParserStructured`  
   - Parameters:  
     - Schema Type: Manual  
     - Input Schema: JSON schema with required fields: dishName (string), ingredients (array with name, amount, calories), totalCalories (number), nutritionEvaluation (string)  
   - Connect input from AI Agent node's output parser output.

5. **Create Code Node ("Format for Gmail")**  
   - Node Type: `code`  
   - Parameters:  
     - JavaScript code that:  
       - Extracts AI analysis JSON from input  
       - Validates presence of required fields to avoid errors  
       - Builds an HTML email with styling including dish name, total calories, ingredients table, nutrition evaluation, and footer  
       - Outputs JSON with `subject` and `htmlBody` fields for email  
   - Connect input from AI Agent node’s main output.

6. **Create Gmail Node ("Send a message1")**  
   - Node Type: `gmail`  
   - Parameters:  
     - Recipient Email: `t.minamig20@gmail.com` (hardcoded; modify as needed)  
     - Subject: Expression referencing previous node’s `subject` field  
     - Message: Expression referencing previous node’s `htmlBody` field  
     - Options: Disable attribution (no signature appended)  
   - Credentials: Gmail OAuth2 account configured in n8n  
   - Connect input from "Format for Gmail" node.

7. **Link Nodes According to Logic:**  
   - Telegram Trigger → AI Agent (main)  
   - OpenRouter Chat Model → AI Agent (ai_languageModel)  
   - AI Agent (ai_outputParser) → Structured Output Parser  
   - AI Agent (main) → Format for Gmail  
   - Format for Gmail → Send a message1 (Gmail node)

8. **Create Sticky Note Nodes for Documentation:**  
   - Add sticky notes as per content in the original workflow for overview, prerequisites, input format, prompt policy, output schema, test instructions, error handling, security, delivery options, and extensibility.  
   - Place near corresponding functional nodes for contextual clarity.

9. **Set Workflow Settings:**  
   - Timezone: Asia/Tokyo  
   - Enable execution data saving for success and errors (all stages)  
   - Execution order: version 1

10. **Credential Setup:**  
    - Telegram API OAuth2 credentials (Telegram Bot token)  
    - OpenRouter API key credential for LLM calls  
    - Gmail OAuth2 credential for sending emails

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires a vision-capable LLM model such as GPT-4o or OpenAI GPT-5 mini variant with vision input.| See Sticky 2 and Sticky 4 for model and prompt details.                                         |
| API keys and tokens must be stored securely in n8n Credentials or environment variables; never hardcode in nodes.| Sticky 8 details security best practices.                                                       |
| For Telegram image references, the workflow uses the `file_id` to fetch the image; ensure bot permissions allow file downloads.| Sticky 3 explains input format and normalization details.                                       |
| Retry logic and exponential backoff on 429 or 5xx errors recommended for production robustness.                 | Sticky 7 covers error handling and limits.                                                     |
| Email formatting uses inline CSS for compatibility with major email clients and includes a summary and detailed table.| Code node "Format for Gmail" contains full HTML template.                                      |
| Alternative delivery options (Telegram replies, Slack messages, HTTP responses) can be added for flexibility.    | Sticky 9 outlines alternative result delivery strategies.                                      |
| Extend the workflow by adding multilingual output, unit normalization, or evidence-based nutrition comments.    | See Sticky 10 for extensibility ideas.                                                         |
| Test the workflow by sending a sample Telegram image message and verifying the structured JSON and email output.| Sticky 6 provides quick test steps.                                                            |

---

**Disclaimer:** The provided analysis and documentation are based exclusively on an n8n automated workflow. All data and processes respect applicable content policies and contain no illegal or sensitive information.