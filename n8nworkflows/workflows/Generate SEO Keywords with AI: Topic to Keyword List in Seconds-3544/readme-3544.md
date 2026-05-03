Generate SEO Keywords with AI: Topic to Keyword List in Seconds

https://n8nworkflows.xyz/workflows/generate-seo-keywords-with-ai--topic-to-keyword-list-in-seconds-3544


# Generate SEO Keywords with AI: Topic to Keyword List in Seconds

### 1. Workflow Overview

This workflow, titled **"Generate SEO Keywords with AI: Topic to Keyword List in Seconds"**, is designed to automate the generation of SEO keyword lists using AI. It targets marketers, SEO specialists, and content creators who want to streamline keyword research by leveraging AI language models to produce relevant, high-potential keywords based on user inputs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user inputs via a form trigger and prepares data for processing.
- **1.2 Data Aggregation:** Consolidates input data into a structured format suitable for AI processing.
- **1.3 AI Processing:** Uses an AI language model to generate keyword suggestions based on the aggregated data.
- **1.4 Result Formatting:** Extracts and formats the AI-generated keywords into a clean, usable output.
- **1.5 Result Delivery:** Sends the formatted keyword list to the user’s email inbox.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures the user’s input parameters for keyword generation via a web form and initializes the data structure for downstream processing.

- **Nodes Involved:**  
  - Input Form  
  - Set Data from Form

- **Node Details:**

  - **Input Form**  
    - *Type & Role:* Form Trigger node; entry point for user input.  
    - *Configuration:* Listens for HTTP POST requests containing form data with fields such as Topic, Search Intent, and Keyword Type.  
    - *Expressions/Variables:* Captures raw form data from the webhook payload.  
    - *Connections:* Output connected to "Set Data from Form".  
    - *Edge Cases:* Missing or malformed form data; webhook connectivity issues.  
    - *Version:* 2.2  

  - **Set Data from Form**  
    - *Type & Role:* Set node; initializes and structures input data for aggregation.  
    - *Configuration:* Maps form fields into named variables or JSON structure for consistent downstream use.  
    - *Expressions/Variables:* Uses expressions to extract form fields from the Input Form node output.  
    - *Connections:* Input from "Input Form"; output to "Aggregate Data Points for AI Keyword Agent".  
    - *Edge Cases:* Expression failures if expected form fields are missing or renamed.  
    - *Version:* 3.4  

#### 1.2 Data Aggregation

- **Overview:**  
  Aggregates and consolidates the structured input data into a single data point or object to be passed to the AI agent.

- **Nodes Involved:**  
  - Aggregate Data Points for AI Keyword Agent

- **Node Details:**

  - **Aggregate Data Points for AI Keyword Agent**  
    - *Type & Role:* Aggregate node; combines multiple input items into one.  
    - *Configuration:* Aggregates all incoming data into a single JSON object or array to simplify AI input.  
    - *Expressions/Variables:* Operates on the output of "Set Data from Form".  
    - *Connections:* Input from "Set Data from Form"; output to "AI Keyword Agent".  
    - *Edge Cases:* Empty input data; aggregation misconfiguration leading to malformed AI input.  
    - *Version:* 1  

#### 1.3 AI Processing

- **Overview:**  
  This block uses an AI language model to generate a list of SEO keywords based on the aggregated input data.

- **Nodes Involved:**  
  - Select your Chat Model  
  - AI Keyword Agent

- **Node Details:**

  - **Select your Chat Model**  
    - *Type & Role:* Language Model Chat node (Groq provider); selects and configures the AI model to be used.  
    - *Configuration:* Connected to user’s Groq account or other LLM provider credentials; allows selection of model such as OpenAI, Claude AI, or Llama.  
    - *Expressions/Variables:* None by default; credentials and model selection configured in node parameters.  
    - *Connections:* Output to "AI Keyword Agent" via ai_languageModel connection.  
    - *Edge Cases:* Authentication failures; API rate limits; model unavailability.  
    - *Version:* 1  

  - **AI Keyword Agent**  
    - *Type & Role:* Langchain Agent node; orchestrates AI prompt execution and response handling.  
    - *Configuration:* Uses the aggregated input data to construct prompts following SEO best practices for keyword generation.  
    - *Expressions/Variables:* Receives aggregated data as input; outputs AI-generated keyword list.  
    - *Connections:* Input from "Aggregate Data Points for AI Keyword Agent"; AI model input from "Select your Chat Model"; output to "Extract and Format".  
    - *Edge Cases:* AI response timeouts; malformed AI responses; prompt injection risks.  
    - *Version:* 1.8  

#### 1.4 Result Formatting

- **Overview:**  
  Processes the raw AI output to extract and format the keyword list into a clean, readable format suitable for email delivery.

- **Nodes Involved:**  
  - Extract and Format

- **Node Details:**

  - **Extract and Format**  
    - *Type & Role:* Code node; executes custom JavaScript to parse and format AI output.  
    - *Configuration:* Contains code to parse AI response JSON or text, extract keywords, and format them as a list or table.  
    - *Expressions/Variables:* Uses input data from "AI Keyword Agent"; outputs formatted string or HTML.  
    - *Connections:* Input from "AI Keyword Agent"; output to "Send Result".  
    - *Edge Cases:* Parsing errors if AI output format changes; empty or unexpected AI responses.  
    - *Version:* 2  

#### 1.5 Result Delivery

- **Overview:**  
  Sends the formatted keyword list to the user’s email address.

- **Nodes Involved:**  
  - Send Result

- **Node Details:**

  - **Send Result**  
    - *Type & Role:* Gmail node; sends email with keyword list.  
    - *Configuration:* Configured with Gmail OAuth2 credentials; "sendTo" parameter set to user’s email address; email body contains formatted keywords.  
    - *Expressions/Variables:* Uses formatted output from "Extract and Format" as email content.  
    - *Connections:* Input from "Extract and Format".  
    - *Edge Cases:* Authentication failures; email delivery failures; invalid recipient address.  
    - *Version:* 2.1  

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                  | Input Node(s)                      | Output Node(s)                     | Sticky Note                        |
|-----------------------------------|----------------------------------|--------------------------------|----------------------------------|----------------------------------|----------------------------------|
| Input Form                        | Form Trigger                     | Captures user input via form   |                                  | Set Data from Form                |                                  |
| Set Data from Form                | Set                              | Structures form data           | Input Form                       | Aggregate Data Points for AI Keyword Agent |                                  |
| Aggregate Data Points for AI Keyword Agent | Aggregate                      | Aggregates input data          | Set Data from Form               | AI Keyword Agent                 |                                  |
| Select your Chat Model            | Langchain LM Chat (Groq)          | Selects AI language model      |                                  | AI Keyword Agent (ai_languageModel) |                                  |
| AI Keyword Agent                 | Langchain Agent                   | Generates keywords using AI    | Aggregate Data Points for AI Keyword Agent, Select your Chat Model | Extract and Format              |                                  |
| Extract and Format               | Code                             | Parses and formats AI output   | AI Keyword Agent                 | Send Result                     |                                  |
| Send Result                     | Gmail                            | Sends keyword list via email   | Extract and Format               |                                  |                                  |
| Sticky Note                     | Sticky Note                      | Annotation                    |                                  |                                  |                                  |
| Sticky Note1                    | Sticky Note                      | Annotation                    |                                  |                                  |                                  |
| Sticky Note2                    | Sticky Note                      | Annotation                    |                                  |                                  |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Input Form" node**  
   - Type: Form Trigger  
   - Configure webhook to accept POST requests with fields: Topic, Search Intent, Keyword Type  
   - Position: Left side as entry point  

2. **Create "Set Data from Form" node**  
   - Type: Set  
   - Map form fields from "Input Form" output to named variables (e.g., `topic`, `searchIntent`, `keywordType`)  
   - Connect "Input Form" output to this node's input  

3. **Create "Aggregate Data Points for AI Keyword Agent" node**  
   - Type: Aggregate  
   - Configure to aggregate all incoming data into a single JSON object  
   - Connect "Set Data from Form" output to this node's input  

4. **Create "Select your Chat Model" node**  
   - Type: Langchain LM Chat (Groq)  
   - Add credentials for your AI provider (Groq, OpenAI, Claude AI, etc.)  
   - Select desired AI model  
   - Position near AI Keyword Agent node  

5. **Create "AI Keyword Agent" node**  
   - Type: Langchain Agent  
   - Configure prompt template to generate SEO keywords based on input data (topic, search intent, keyword type)  
   - Connect "Aggregate Data Points for AI Keyword Agent" main output to this node's main input  
   - Connect "Select your Chat Model" output to this node's `ai_languageModel` input  

6. **Create "Extract and Format" node**  
   - Type: Code  
   - Write JavaScript code to parse AI response and format keywords into a clean list or HTML table  
   - Connect "AI Keyword Agent" output to this node's input  

7. **Create "Send Result" node**  
   - Type: Gmail  
   - Configure with Gmail OAuth2 credentials or preferred email service credentials  
   - Set "sendTo" parameter to recipient email address  
   - Use formatted output from "Extract and Format" as email body  
   - Connect "Extract and Format" output to this node's input  

8. **Add Sticky Notes** (optional)  
   - Add notes for documentation or instructions as needed  

9. **Test the workflow**  
   - Trigger the form with sample inputs  
   - Verify email delivery of keyword list  

10. **Activate the workflow**  
    - Once testing is successful, activate for production use  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow template is designed to save time on SEO keyword research by automating keyword generation with AI. | Workflow description and setup instructions provided in the initial documentation.              |
| Supports multiple AI providers: Groq, OpenAI, Claude AI, Llama, etc.                             | Allows flexibility in AI model selection via the "Select your Chat Model" node.                  |
| Email delivery can be replaced with database or spreadsheet nodes for alternative keyword storage. | Customization tip for adapting workflow to different output requirements.                        |
| Modify AI prompts in "AI Keyword Agent" to tailor keyword generation strategies.                 | Enables advanced users to refine keyword output quality and relevance.                          |
| Add filtering or integration nodes to extend functionality (e.g., competition analysis).         | Workflow is a modular starting point for broader SEO automation pipelines.                      |
| ![979shots_so.jpg](fileId:1109)                                                                 | Visual branding or illustrative image included in the original workflow documentation.          |

---

This structured documentation provides a comprehensive understanding of the workflow’s design, node configurations, and operational flow, enabling users and automation agents to reproduce, modify, and troubleshoot the workflow effectively.