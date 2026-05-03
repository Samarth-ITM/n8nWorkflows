Generate Food Recipes from Gmail & Form Requests with Ollama & Llama 3.2

https://n8nworkflows.xyz/workflows/generate-food-recipes-from-gmail---form-requests-with-ollama---llama-3-2-5871


# Generate Food Recipes from Gmail & Form Requests with Ollama & Llama 3.2

### 1. Workflow Overview

This workflow automates the generation and delivery of customized food recipes upon receiving user requests either via Gmail or a web form. It leverages advanced AI language models (Ollama and Llama 3.2) to interpret user input, create detailed recipes, format them for readability, and send the finished recipes back to the users via email.

**Logical Blocks:**

- **1.1 Input Reception:** Captures recipe requests from either Gmail emails or a web form submission.
- **1.2 AI Recipe Generation:** Processes user queries using AI models to generate detailed and engaging recipes.
- **1.3 Recipe Formatting:** Converts raw AI output into well-structured HTML suitable for email presentation.
- **1.4 Recipe Delivery:** Sends the formatted recipe back to the user’s email address.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming recipe requests either via Gmail or a web form. It triggers the workflow when a new email or form submission is detected.

- **Nodes Involved:**  
  - Recipe Request - Gmail  
  - Recipe Request - Web Form

- **Node Details:**

  - **Recipe Request - Gmail**  
    - *Type:* Gmail Trigger  
    - *Role:* Watches Gmail inbox for new emails to trigger recipe generation.  
    - *Configuration:* Polls every few minutes (configured to poll periodically). No specific filters applied.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Provides email data including email snippet and recipient address.  
    - *Edge cases:* Possible authentication errors if OAuth token expires; delays if Gmail API rate limits are hit; malformed emails missing expected content.  
    - *Credentials:* OAuth2 Gmail account with appropriate scopes.

  - **Recipe Request - Web Form**  
    - *Type:* Form Trigger  
    - *Role:* Listens for web form submissions titled "🍳 Ask the AI Chef!" capturing user queries.  
    - *Configuration:* One required text field for user’s recipe question with placeholder examples.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Sends form data including user’s recipe question.  
    - *Edge cases:* Missing or empty user input; webhook connectivity issues; duplicate form submissions.

---

#### 1.2 AI Recipe Generation

- **Overview:**  
  This block interprets the user’s query and generates a complete recipe using specialized AI language models, combining the power of two nodes for enhanced processing.

- **Nodes Involved:**  
  - Ollama Recipe Generator (LangChain Agent)  
  - Llama 3.2 - Chef Model (LM Chat Ollama)

- **Node Details:**

  - **Ollama Recipe Generator**  
    - *Type:* LangChain Agent Node  
    - *Role:* Receives user input and executes a prompt designed to generate detailed recipes including title, ingredients, instructions, special add-ons, suggested pairings, and chef’s notes.  
    - *Configuration:* Uses a custom prompt that instructs the AI to produce visually appealing, emoji-enhanced recipes formatted in Markdown without bullet points or symbols except numbering steps.  
    - *Key Expressions:*  
      - Uses `{{ $json.snippet }}` to insert the user’s query from Gmail or form input.  
      - Placeholder text for ingredients, recipe details, and add-ons suggests possible expansion or integration points.  
    - *Inputs:* Receives data from Gmail and web form triggers; also connected downstream from Llama 3.2.  
    - *Outputs:* Produces raw Markdown recipe text.  
    - *Edge cases:* AI may misinterpret ambiguous inputs; incomplete or incompatible ingredient data; API or model unavailability.  
    - *Version Specifics:* Uses LangChain agent with promptType "define" for controlled output formatting.

  - **Llama 3.2 - Chef Model**  
    - *Type:* LM Chat Ollama Node  
    - *Role:* Acts as a conversational AI model supporting the recipe generation process. Feeds output into the Ollama agent node for final recipe creation.  
    - *Configuration:* Uses latest Llama 3.2-16000 model with default options.  
    - *Inputs:* Receives input from triggers.  
    - *Outputs:* Feeds into the Ollama Recipe Generator node.  
    - *Credentials:* Requires Ollama API credentials.  
    - *Edge cases:* API call failures; model load issues; response delays.

---

#### 1.3 Recipe Formatting

- **Overview:**  
  This block converts the raw Markdown recipe output from the AI into user-friendly HTML with clear sections and inline styling to enhance readability in emails.

- **Nodes Involved:**  
  - Format Recipe Output (Code Node)

- **Node Details:**

  - **Format Recipe Output**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Parses the AI-generated Markdown text, extracts key recipe sections (title, ingredients, instructions, etc.), and formats them into styled HTML with line breaks and bolded section headers.  
    - *Configuration:* Custom JavaScript code processes the raw output; uses regex to extract sections; replaces Markdown line breaks with HTML `<br>` tags; uses inline CSS for font and spacing.  
    - *Inputs:* Receives raw recipe text from Ollama Recipe Generator node.  
    - *Outputs:* Produces formatted HTML string for email body.  
    - *Edge cases:* Unexpected or malformed AI output may cause extraction failures; missing sections will be omitted gracefully.

---

#### 1.4 Recipe Delivery

- **Overview:**  
  This block sends the formatted recipe back to the user’s Gmail address extracted from the initial request.

- **Nodes Involved:**  
  - Send Recipe via Email (Gmail Node)

- **Node Details:**

  - **Send Recipe via Email**  
    - *Type:* Gmail Node (Send Email)  
    - *Role:* Emails the formatted recipe HTML content to the user’s email address.  
    - *Configuration:*  
      - Recipient address dynamically set from the original Gmail trigger’s "To" field.  
      - Email subject: "🔥 Ready to Cook? Here’s Your Recipe from AI Chef".  
      - Message body: contains the formatted HTML recipe.  
      - Attribution appended is disabled.  
      - Executes once per input.  
    - *Inputs:* Receives formatted HTML from the previous node.  
    - *Outputs:* Email send status.  
    - *Credentials:* Uses the same Gmail OAuth2 credential as the trigger.  
    - *Edge cases:* Email sending failures due to SMTP limits or auth errors; malformed recipient addresses; network issues.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                         | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                   |
|--------------------------|----------------------------------|---------------------------------------|-------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Recipe Request - Gmail    | Gmail Trigger                    | Triggers workflow on new email request| None                          | Ollama Recipe Generator      | Starts when user sends a Gmail or submits a form with recipe request                          |
| Recipe Request - Web Form | Form Trigger                    | Triggers workflow on form submission  | None                          | Ollama Recipe Generator      | Starts when user sends a Gmail or submits a form with recipe request                          |
| Llama 3.2 - Chef Model   | LM Chat Ollama                  | AI language model assistant for recipes| Recipe Request - Gmail, Web Form | Ollama Recipe Generator      | Understands user query and generates recipe                                                   |
| Ollama Recipe Generator  | LangChain Agent                 | Generates recipe text based on user input | Recipe Request - Gmail, Web Form, Llama 3.2 | Format Recipe Output         | Understands user query and generates recipe                                                   |
| Format Recipe Output      | Code Node (JavaScript)          | Formats raw AI output into HTML       | Ollama Recipe Generator       | Send Recipe via Email        | Formats the recipe (e.g., ingredients, steps)                                                |
| Send Recipe via Email     | Gmail Node                     | Sends the formatted recipe email      | Format Recipe Output          | None                        | Sends formatted recipe back to the user                                                      |
| Sticky Note              | Sticky Note                    | Workflow annotation                   | None                          | None                        | Starts when user sends a Gmail or submits a form with recipe request                          |
| Sticky Note1             | Sticky Note                    | Workflow annotation                   | None                          | None                        | Understands user query and generates recipe                                                   |
| Sticky Note2             | Sticky Note                    | Workflow annotation                   | None                          | None                        | Formats the recipe (e.g., ingredients, steps)                                                |
| Sticky Note3             | Sticky Note                    | Workflow annotation                   | None                          | None                        | Sends formatted recipe back to the user                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Node Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail with read access to inbox.  
   - Set to poll every X minutes (default or preferred interval).  
   - Leave filters empty to trigger on all new emails.  
   - Name: "Recipe Request - Gmail".

2. **Create Form Trigger Node**  
   - Node Type: Form Trigger  
   - Configure webhook with unique ID.  
   - Set form title: "🍳 Ask the AI Chef!"  
   - Add one required text field labeled "Hey Dude, Tell Me fast." with placeholder text: `"What is a pizza recipe?","How do I make butter chicken?"`  
   - Add form description: "✨ Type any food name, ingredient, or dish, and our AI Chef will give you the perfect recipe in seconds!"  
   - Name: "Recipe Request - Web Form".

3. **Create Llama 3.2 - Chef Model Node**  
   - Node Type: LM Chat Ollama  
   - Model: "llama3.2-16000:latest"  
   - Leave options as default.  
   - Set Ollama API credentials (ensure valid API key configured).  
   - Name: "Llama 3.2 - Chef Model".  
   - Connect inputs from both "Recipe Request - Gmail" and "Recipe Request - Web Form".

4. **Create Ollama Recipe Generator Node**  
   - Node Type: LangChain Agent  
   - Prompt Type: "define"  
   - Paste the detailed recipe generation prompt provided, ensuring to incorporate the `${json.snippet}` expression to dynamically use the user’s question.  
   - Leave options default.  
   - Name: "Ollama Recipe Generator".  
   - Connect inputs from:  
     - "Recipe Request - Gmail"  
     - "Recipe Request - Web Form"  
     - "Llama 3.2 - Chef Model" (under ai_languageModel input).  

5. **Create Format Recipe Output Node**  
   - Node Type: Code (JavaScript)  
   - Paste the JavaScript code that processes raw AI output into HTML format with regex for section extraction and line breaks.  
   - Name: "Format Recipe Output".  
   - Connect input from "Ollama Recipe Generator".

6. **Create Send Recipe via Email Node**  
   - Node Type: Gmail (Send Email)  
   - Configure OAuth2 Gmail credentials (can be same as trigger).  
   - Set "Send To" address dynamically as `={{ $('Recipe Request - Gmail').item.json.To }}` to send back to original requester.  
   - Set subject: "🔥 Ready to Cook? Here’s Your Recipe from AI Chef".  
   - Set email message body to `={{ $json.formattedHtml }}` to use formatted HTML from previous node.  
   - Disable attribution.  
   - Set to execute once per input.  
   - Name: "Send Recipe via Email".  
   - Connect input from "Format Recipe Output".

7. **Add Sticky Notes** (optional for documentation)  
   - Add notes near each block describing purpose, e.g., trigger start, AI processing, formatting, email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow initiates on either Gmail or web form input, allowing versatile user engagement methods.       | Sticky Note near "Recipe Request" nodes.                                                                     |
| Recipe generation prompt is carefully crafted to produce visually appealing, emoji-rich, markdown output | Prompt inside "Ollama Recipe Generator" node configuration.                                                  |
| Email formatting uses inline CSS and HTML for better compatibility across email clients.                | Code node "Format Recipe Output" with regex-based section extraction.                                        |
| Ensure Gmail OAuth2 credentials have proper scopes for reading and sending emails to avoid auth errors. | Gmail nodes require OAuth2 setup with Gmail API scopes.                                                      |
| Ollama and Llama models require API credentials; ensure proper subscription and quota management.       | Credentials must be set up for Ollama API usage in LLM nodes.                                                |
| Consider error handling for API timeouts and malformed inputs for robust workflow operation.            | Gmail and AI nodes can fail due to network or auth issues; code node handles missing sections gracefully.    |

---

**Disclaimer:** The text provided is exclusively generated from an automated workflow created with n8n, respecting all content policies and handling only legal and public data.