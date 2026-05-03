Business Model Canvas AI-Powered Generator (LLM Flexible)

https://n8nworkflows.xyz/workflows/business-model-canvas-ai-powered-generator--llm-flexible--3663


# Business Model Canvas AI-Powered Generator (LLM Flexible)

### 1. Workflow Overview

This workflow, titled **Business Model Canvas AI-Powered Generator (LLM Flexible)**, automates the creation of a fully populated Business Model Canvas (BMC) based on a user’s business idea. It is designed for startup founders, business consultants, product teams, and agencies who want to quickly generate a structured, professional, and printable BMC in an A4 landscape HTML format.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the user’s business idea via a chat interface.
- **1.2 AI Section Generators:** Uses multiple AI agents (one per BMC section) to generate concise bullet points for each canvas section.
- **1.3 HTML Transformation:** Converts AI-generated text outputs into formatted HTML snippets.
- **1.4 Data Merging:** Combines all HTML snippets and the business title into a single structured HTML document.
- **1.5 Output Generation:** Converts the final HTML code into a downloadable `.html` file.

The workflow supports flexible integration with different Large Language Models (LLMs) such as Ollama or OpenAI by simply swapping the language model node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving a chat message containing the user's business idea. It prompts the user with a friendly greeting and requests detailed input about their business.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type:* Chat Trigger (LangChain)  
    - *Role:* Starts the workflow by capturing user input via chat.  
    - *Configuration:*  
      - Public webhook enabled for external access.  
      - Initial greeting message: "Hi there! 👋 Please tell me everything about your business, and I will help you create the business canvas."  
    - *Input/Output:*  
      - Input: Incoming chat message (business idea).  
      - Output: JSON containing `chatInput` with user’s business description.  
    - *Potential Failures:* Network issues, webhook misconfiguration, or user input errors (empty or irrelevant input).  
    - *Version:* 1.1  

---

#### 2.2 AI Section Generators

- **Overview:**  
  This block contains ten parallel AI agent nodes, each responsible for generating bullet points for a specific section of the Business Model Canvas based on the user’s input. Each agent uses a prompt tailored to its section and outputs a pipe-separated list of concise bullet points.

- **Nodes Involved:**  
  - Key Partners Generator  
  - Key Activities Generator  
  - Value Proposition Generator  
  - Customer Relationship Generator  
  - Customer Segments Generator  
  - Key Resources Generator  
  - Channels Generator  
  - Cost Structure Generator  
  - Revenue Streams Generator  
  - Title Generator

- **Node Details:**

  Each AI agent node shares the following characteristics:

  - *Type:* LangChain Agent (AI Agent Node)  
  - *Role:* Generate section-specific bullet points based on the business idea.  
  - *Configuration:*  
    - Prompt instructs the agent to act as a domain expert (e.g., strategic business analyst, marketing strategist).  
    - Output format strictly pipe-separated bullet points without headers or numbering.  
    - Each bullet point is concise (4-5 words, max 10 words).  
    - Input expression uses `{{ $json.chatInput }}` to inject the user's business idea dynamically.  
  - *Input/Output:*  
    - Input: Business idea text from chat trigger node.  
    - Output: Pipe-separated bullet points string in `output` JSON property.  
  - *Version:* 1.8 for agents, 1.1 for Title Generator.  
  - *Potential Failures:*  
    - AI model timeout or rate limits.  
    - Unexpected output format breaking downstream parsing.  
    - Authentication or credential errors with the LLM provider.  
  - *Sub-workflow:* None, but all use the same LLM node (Ollama Chat Model) for processing.

  Specific notes per node:

  - **Title Generator:** Generates a simple business name (max 5 words) from the idea, outputting only the name.

---

#### 2.3 HTML Transformation

- **Overview:**  
  Converts each AI-generated pipe-separated bullet points string into formatted HTML paragraphs for display on the canvas. This standardizes the output for easy merging and final rendering.

- **Nodes Involved:**  
  - Key Partners HTML Transformer  
  - Key Activities HTML Transformer  
  - Values proposition HTML Transformer  
  - Customer Relationship HTML Transformer  
  - Customer Segments HTML Transformer  
  - Key Resources HTML Transformer  
  - Channels HTML Transformer  
  - Cost Structure HTML Transformer  
  - Revenue streams HTML Transformer  
  - Code1 (Title cleanup)

- **Node Details:**

  Each transformer node:

  - *Type:* Code Node (JavaScript)  
  - *Role:* Parses the pipe-separated string, trims items, and wraps each bullet point in a `<p>• ...</p>` HTML tag (except Customer Relationship which omits the bullet).  
  - *Key Code Logic:*  
    - Split input string by pipe `|`.  
    - Trim whitespace from each item.  
    - Generate HTML paragraphs with bullet points.  
  - *Input/Output:*  
    - Input: AI agent output string from previous block.  
    - Output: JSON with a key named after the section (e.g., `key_partners`) containing the HTML snippet.  
  - *Potential Failures:*  
    - Input string missing or malformed (e.g., no pipe separators).  
    - Unexpected characters causing HTML rendering issues.  
  - *Version:* 2  

  - **Code1 Node:**  
    - Cleans up the title output by removing newline characters.  
    - Outputs a clean `title` string for final use.

---

#### 2.4 Data Merging

- **Overview:**  
  This block merges all transformed HTML snippets and the cleaned title into a single JSON object to be used for final HTML generation.

- **Nodes Involved:**  
  - Merge All Data

- **Node Details:**

  - *Type:* Merge Node  
  - *Role:* Combines outputs from all 10 HTML transformer nodes plus the title cleanup node into one data structure.  
  - *Configuration:*  
    - Set to accept 10 inputs (one per section plus title).  
    - Merges data by index to keep all parts aligned.  
  - *Input/Output:*  
    - Input: HTML snippets and title JSONs.  
    - Output: Single JSON object with all canvas sections and title.  
  - *Potential Failures:*  
    - Missing inputs if any upstream node fails.  
    - Data misalignment if inputs arrive out of order.  
  - *Version:* 3.1  

---

#### 2.5 Output Generation

- **Overview:**  
  Generates the final Business Model Canvas as a styled HTML document and converts it into a downloadable file.

- **Nodes Involved:**  
  - Turn to HTML  
  - HTML code to HTML file

- **Node Details:**

  - **Turn to HTML:**  
    - *Type:* Code Node  
    - *Role:* Constructs a full HTML document embedding all canvas sections into a styled A4 landscape layout.  
    - *Key Logic:*  
      - Uses template literals to inject title and all HTML snippets into a responsive table structure.  
      - Includes CSS for print-friendly formatting and professional styling.  
      - Ensures proper aspect ratio for A4 paper in landscape orientation.  
    - *Input:* Merged JSON with all sections and title.  
    - *Output:* JSON with `final_html` property containing full HTML code and `title` for filename use.  
    - *Potential Failures:*  
      - Missing or malformed HTML snippets causing broken layout.  
      - Large content causing browser rendering issues.  
    - *Version:* 2  

  - **HTML code to HTML file:**  
    - *Type:* Convert to File Node  
    - *Role:* Converts the HTML code string into a downloadable `.html` file with a dynamic filename based on the business title.  
    - *Configuration:*  
      - Filename template: `{{ $json.title }} BMC.html`  
      - Operation: toText (text file)  
    - *Input:* `final_html` property from previous node.  
    - *Output:* Binary file data ready for download.  
    - *Potential Failures:*  
      - File naming conflicts or invalid characters in title.  
      - File size limits or encoding issues.  
    - *Version:* 1.1  

---

#### 2.6 LLM Integration Node

- **Overview:**  
  Central language model node used by all AI agent nodes to process prompts and generate outputs.

- **Node Involved:**  
  - Ollama Chat Model

- **Node Details:**

  - *Type:* LangChain Ollama Chat Model Node  
  - *Role:* Executes all AI prompt completions using the Ollama LLaMA 3.1 model.  
  - *Configuration:*  
    - Model: `llama3.1:latest`  
    - Credentials: Ollama API key configured.  
  - *Input:* Prompts from AI agent nodes.  
  - *Output:* Text completions for each prompt.  
  - *Potential Failures:*  
    - API authentication errors.  
    - Model unavailability or latency.  
    - Rate limits.  
  - *Version:* 1  

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                           | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                                  |
|-------------------------------|--------------------------------|-----------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When chat message received     | Chat Trigger (LangChain)        | Capture user business idea input        | -                                | Key Partners Generator, Value Proposition Generator, Customer Relationship Generator, Customer Segments Generator, Key Resources Generator, Channels Generator, Cost Structure Generator, Revenue Streams Generator, Title Generator, Key Activities Generator |                                                                                                                              |
| Key Partners Generator         | LangChain Agent                 | Generate Key Partners bullet points     | When chat message received       | Key Partners HTML Transformer    |                                                                                                                              |
| Key Activities Generator       | LangChain Agent                 | Generate Key Activities bullet points   | When chat message received       | Key Activities HTML Transformer  |                                                                                                                              |
| Value Proposition Generator    | LangChain Agent                 | Generate Value Proposition bullet points| When chat message received       | Values proposition HTML Transformer |                                                                                                                              |
| Customer Relationship Generator| LangChain Agent                 | Generate Customer Relationship bullet points | When chat message received   | Customer Relationship HTML Transformer |                                                                                                                              |
| Customer Segments Generator    | LangChain Agent                 | Generate Customer Segments bullet points| When chat message received       | Customer Segments HTML Transformer |                                                                                                                              |
| Key Resources Generator        | LangChain Agent                 | Generate Key Resources bullet points    | When chat message received       | Key Resources HTML Transformer   |                                                                                                                              |
| Channels Generator             | LangChain Agent                 | Generate Channels bullet points         | When chat message received       | Channels HTML Transformer        |                                                                                                                              |
| Cost Structure Generator       | LangChain Agent                 | Generate Cost Structure bullet points   | When chat message received       | Cost Structure HTML Transformer  |                                                                                                                              |
| Revenue Streams Generator      | LangChain Agent                 | Generate Revenue Streams bullet points  | When chat message received       | Revenue streams HTML Transformer |                                                                                                                              |
| Title Generator               | LangChain Agent                 | Generate business title                  | When chat message received       | Code1                          |                                                                                                                              |
| Key Partners HTML Transformer  | Code Node                      | Format Key Partners output to HTML      | Key Partners Generator           | Merge All Data                  |                                                                                                                              |
| Key Activities HTML Transformer| Code Node                      | Format Key Activities output to HTML    | Key Activities Generator         | Merge All Data                  |                                                                                                                              |
| Values proposition HTML Transformer | Code Node                 | Format Value Proposition output to HTML | Value Proposition Generator      | Merge All Data                  |                                                                                                                              |
| Customer Relationship HTML Transformer | Code Node              | Format Customer Relationship output to HTML | Customer Relationship Generator | Merge All Data                  |                                                                                                                              |
| Customer Segments HTML Transformer | Code Node                  | Format Customer Segments output to HTML | Customer Segments Generator      | Merge All Data                  |                                                                                                                              |
| Key Resources HTML Transformer | Code Node                     | Format Key Resources output to HTML     | Key Resources Generator          | Merge All Data                  |                                                                                                                              |
| Channels HTML Transformer      | Code Node                     | Format Channels output to HTML          | Channels Generator               | Merge All Data                  |                                                                                                                              |
| Cost Structure HTML Transformer| Code Node                     | Format Cost Structure output to HTML    | Cost Structure Generator         | Merge All Data                  |                                                                                                                              |
| Revenue streams HTML Transformer | Code Node                   | Format Revenue Streams output to HTML   | Revenue Streams Generator        | Merge All Data                  |                                                                                                                              |
| Code1                         | Code Node                      | Clean business title string              | Title Generator                 | Merge All Data                  |                                                                                                                              |
| Merge All Data                | Merge Node                     | Merge all HTML snippets and title       | All HTML Transformer nodes + Code1 | Turn to HTML                  |                                                                                                                              |
| Turn to HTML                  | Code Node                      | Generate full HTML document for BMC     | Merge All Data                  | HTML code to HTML file          |                                                                                                                              |
| HTML code to HTML file        | Convert to File Node           | Convert HTML code to downloadable file  | Turn to HTML                   | -                              |                                                                                                                              |
| Ollama Chat Model             | LangChain Ollama Chat Model   | LLM backend for all AI agent nodes      | AI Agent nodes                 | AI Agent nodes                 | 🔁 Changeable LLM Model: This template is powered by Ollama LLM (LLaMA 3.1) by default but can be swapped easily with other LLMs. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **When chat message received** (LangChain Chat Trigger) node.  
   - Set it as a public webhook.  
   - Configure initial message: "Hi there! 👋 Please tell me everything about your business, and I will help you create the business canvas."

2. **Add LLM Node:**  
   - Add **Ollama Chat Model** node (LangChain Ollama Chat Model).  
   - Configure with model `llama3.1:latest`.  
   - Add Ollama API credentials.

3. **Create AI Agent Nodes (10 total):**  
   For each Business Model Canvas section, create a LangChain Agent node with the following details:

   - **Key Partners Generator:**  
     - Prompt: Strategic business analyst focused on key partners.  
     - Output format: Pipe-separated bullet points.  
     - Input expression: `{{ $json.chatInput }}`.

   - **Key Activities Generator:**  
     - Prompt: Strategic business analyst focused on key activities.  
     - Output format: Pipe-separated bullet points.

   - **Value Proposition Generator:**  
     - Prompt: Value-driven strategist for value proposition.  
     - Output format: Pipe-separated bullet points.

   - **Customer Relationship Generator:**  
     - Prompt: Customer relationship strategist.  
     - Output format: Pipe-separated bullet points.

   - **Customer Segments Generator:**  
     - Prompt: Segmentation expert.  
     - Output format: Pipe-separated bullet points.

   - **Key Resources Generator:**  
     - Prompt: Operational strategist for key resources.  
     - Output format: Pipe-separated bullet points.

   - **Channels Generator:**  
     - Prompt: Marketing strategist for channels.  
     - Output format: Pipe-separated bullet points.

   - **Cost Structure Generator:**  
     - Prompt: Finance strategist for cost structure.  
     - Output format: Pipe-separated bullet points.

   - **Revenue Streams Generator:**  
     - Prompt: Monetization expert for revenue streams.  
     - Output format: Pipe-separated bullet points.

   - **Title Generator:**  
     - Prompt: Generate simple business name (max 5 words).  
     - Output: Plain text title.

   - Connect all these agents to the **Ollama Chat Model** node as their language model.

4. **Create HTML Transformer Nodes (10 total):**  
   For each AI agent output, add a **Code Node** to convert pipe-separated strings into HTML paragraphs:

   - Use JavaScript to split by `|`, trim, and wrap each item in `<p>• ...</p>`.  
   - For Customer Relationship, omit the bullet symbol.  
   - For Title Generator, add a Code Node to remove newline characters.

5. **Add Merge Node:**  
   - Add a **Merge** node configured to accept 10 inputs.  
   - Connect all HTML Transformer nodes and the title cleanup node to this merge node.

6. **Add Final HTML Generation Node:**  
   - Add a **Code Node** named "Turn to HTML".  
   - Write JavaScript to assemble the full HTML document embedding all sections in a styled table with A4 landscape layout.  
   - Use the merged data to inject title and HTML snippets.

7. **Add Convert to File Node:**  
   - Add **Convert to File** node to convert the HTML string to a downloadable `.html` file.  
   - Set filename to `{{ $json.title }} BMC.html`.  
   - Configure operation as `toText`.  
   - Set source property to the HTML code property from the previous node.

8. **Connect Nodes:**  
   - Connect the chat trigger node to all AI agent nodes.  
   - Connect AI agent nodes to their respective HTML transformer nodes.  
   - Connect all HTML transformers and title cleanup node to the Merge node.  
   - Connect Merge node to the Turn to HTML node.  
   - Connect Turn to HTML node to the Convert to File node.

9. **Credential Setup:**  
   - Configure Ollama API credentials for the Ollama Chat Model node.  
   - If using another LLM, configure corresponding credentials and update AI agent nodes accordingly.

10. **Testing:**  
    - Start the workflow.  
    - Send a business idea via the chat interface.  
    - Wait for processing to complete.  
    - Access the final node to download the generated Business Model Canvas HTML file.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 🔁 Changeable LLM Model: This template is powered by Ollama LLM (LLaMA 3.1) by default but can be swapped easily with other LLMs.       | Sticky Note near LLM node                                                                           |
| How to get the output? After processing, download the canvas from the final node “HTML code to HTML file” by clicking the Download button. | Sticky Note near the final output node                                                              |
| For help customizing or feedback, contact: **sinamirshafiee@gmail.com**                                                                  | Sticky Note at the bottom of the workflow                                                           |
| The final HTML uses Google Fonts (Headland One) and includes print-friendly CSS for professional A4 landscape output.                   | Embedded in the Turn to HTML node                                                                    |
| Supports easy customization: change LLM, modify final HTML layout, add PDF export, email delivery, or CRM/webform triggers.             | Workflow description                                                                                |

---

This documentation provides a detailed, structured understanding of the Business Model Canvas AI-Powered Generator workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.