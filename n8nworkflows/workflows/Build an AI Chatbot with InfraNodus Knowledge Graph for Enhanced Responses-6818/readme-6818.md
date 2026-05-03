Build an AI Chatbot with InfraNodus Knowledge Graph for Enhanced Responses

https://n8nworkflows.xyz/workflows/build-an-ai-chatbot-with-infranodus-knowledge-graph-for-enhanced-responses-6818


# Build an AI Chatbot with InfraNodus Knowledge Graph for Enhanced Responses

### 1. Workflow Overview

This workflow implements a lightweight AI chatbot interface powered by InfraNodus, a knowledge graph platform. It is designed to process user input from a web form, query an InfraNodus knowledge graph to generate context-aware AI responses, and then display those responses back to the user through a form interface.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Captures user messages submitted through a web form.
- **1.2 AI Processing via InfraNodus:** Sends the user’s message to InfraNodus API with configured parameters to obtain a knowledge-graph-enhanced AI response.
- **1.3 Response Delivery:** Presents the AI-generated answer back to the user in a styled form response.

Additionally, two sticky notes provide supplementary guidance on setting up the knowledge base and sharing the chat publicly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures the user’s question or message submitted through a web form, acting as the entry point of the chatbot interaction.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Listens for form submissions and triggers the workflow when a user submits a message.  
    - Configuration:  
      - Webhook ID assigned (unique identifier for receiving HTTP POST requests)  
      - Form title: “What would you like to know?”  
      - One form field: textarea labeled “Your message” for user input  
    - Key expressions/variables: User input accessible via `$json['Your message']`  
    - Input connections: None (trigger node)  
    - Output connections: Connected to “AI Response” node  
    - Version: 2.2  
    - Edge cases:  
      - Network issues or webhook misconfiguration could prevent trigger firing  
      - User submitting empty or malformed input  
    - Sub-workflow: None

---

#### 2.2 AI Processing via InfraNodus

- **Overview:**  
  Sends the user’s message to InfraNodus API, which processes it with the configured knowledge graph to generate an enhanced AI response. This block handles authentication, request formatting, and API communication.

- **Nodes Involved:**  
  - AI Response

- **Node Details:**

  - **AI Response**  
    - Type: HTTP Request  
    - Role: Posts user input to InfraNodus API endpoint to retrieve an AI-generated response enriched by the knowledge graph.  
    - Configuration:  
      - Method: POST  
      - URL: `https://infranodus.com/api/v1/graphAndAdvice` with query parameters:  
        - `doNotSave=true` (prevents saving session data)  
        - `addStats=true` (requests inclusion of statistics)  
        - `optimize=develop` (optimization mode)  
        - `includeStatements=true` (include extracted statements in response)  
        - `includeGraphSummary=true` (include a summary of the graph)  
        - `includeGraph=false` (do not include full graph data)  
      - Body parameters (sent as JSON):  
        - `name`: InfraNodus graph name, fixed as `"eightos_system"` (this should be replaced with the actual knowledge base graph name)  
        - `requestMode`: `"response"` (specifies that a response is requested)  
        - `aiTopics`: `"true"` (enables topics extraction assistance)  
        - `prompt`: Dynamic expression: `={{ $json['Your message'] }}` (user’s message from the form)  
      - Authentication: HTTP Bearer Token (InfraNodus Expert credential)  
    - Input connections: From “On form submission” node  
    - Output connections: To “Form” node (for response presentation)  
    - Version: 4.2  
    - Edge cases:  
      - HTTP errors such as 401 Unauthorized if token invalid or expired  
      - Network timeouts or connectivity issues  
      - API rate limiting or quota exhaustion  
      - Missing or incorrect graph name leading to empty or irrelevant responses  
      - Expression errors if `$json['Your message']` is undefined or empty  
    - Sub-workflow: None

---

#### 2.3 Response Delivery

- **Overview:**  
  Delivers the AI-generated response back to the user through a styled form interface, completing the chatbot interaction cycle.

- **Nodes Involved:**  
  - Form

- **Node Details:**

  - **Form**  
    - Type: Form  
    - Role: Displays the AI’s response to the user in a clean, styled web form interface.  
    - Configuration:  
      - Webhook ID assigned (unique for presenting the form to users)  
      - Operation: Completion (used to show a completion message on form submit)  
      - Completion title: “Your response”  
      - Completion message: Dynamic expression: `={{ $json.aiAdvice[0].text }}`  
        - This extracts the first piece of AI advice text returned from InfraNodus API’s response JSON under the field `aiAdvice`  
      - Custom CSS: A comprehensive style block defining fonts, colors, spacings, borders, and shadows for a polished UI experience.  
    - Input connections: From “AI Response” node  
    - Output connections: None  
    - Version: 1  
    - Edge cases:  
      - If the API response does not include `aiAdvice` or it is empty, the completion message will be blank or undefined  
      - Styling may not render correctly if CSS is overridden or not supported by the client browser  
    - Sub-workflow: None

---

#### 2.4 Supplemental Nodes (Sticky Notes)

- **Overview:**  
  Provide inline documentation and setup instructions as visual notes within the workflow editor.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content:  
      ```
      ## Add your knowledge base

      ### 1. Create a graph in [InfraNodus](https://infranodus.com) with the data you want to use as a knowledge base

      ### 2. Add its name into the `name` field here
      ```  
    - Position: Top left area (guidance for knowledge base setup)  
    - Role: Instructions on creating and linking the graph knowledge base used by API

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content:  
      ```
      ## Publicly available chat

      ### Copy the `production` URL from here and share it the public
      ```  
    - Position: Top left area (near the other sticky note)  
    - Role: Instructions on how to share the deployed chatbot URL publicly

---

### 3. Summary Table

| Node Name         | Node Type              | Functional Role                      | Input Node(s)         | Output Node(s) | Sticky Note                                                                                     |
|-------------------|------------------------|------------------------------------|-----------------------|----------------|------------------------------------------------------------------------------------------------|
| On form submission | Form Trigger           | Capture user input from form       | None                  | AI Response    |                                                                                                |
| AI Response       | HTTP Request           | Query InfraNodus API for AI reply  | On form submission    | Form           |                                                                                                |
| Form              | Form                   | Display AI response to user        | AI Response           | None           |                                                                                                |
| Sticky Note       | Sticky Note            | Instructions on knowledge base     | None                  | None           | ## Add your knowledge base<br>### 1. Create a graph in [InfraNodus](https://infranodus.com) with the data you want to use as a knowledge base<br>### 2. Add its name into the `name` field here |
| Sticky Note1      | Sticky Note            | Instructions on sharing chat URL   | None                  | None           | ## Publicly available chat<br>### Copy the `production` URL from here and share it the public   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node:**  
   - Set node name: `On form submission`  
   - Configure webhook ID (auto-generated)  
   - Set Form title: `What would you like to know?`  
   - Add one form field:  
     - Field type: `Textarea`  
     - Label: `Your message`  
   - This node will trigger the workflow when the user submits their message.

3. **Add an HTTP Request node:**  
   - Set node name: `AI Response`  
   - Method: `POST`  
   - URL: `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&addStats=true&optimize=develop&includeStatements=true&includeGraphSummary=true&includeGraph=false`  
   - Authentication: HTTP Bearer Auth  
     - Create and assign credentials with InfraNodus API Bearer token (named e.g. `InfraNodus Expert`)  
   - Body parameters (JSON) with the following fields:  
     - `name`: Set to your InfraNodus knowledge graph name (e.g., `"eightos_system"`)  
     - `requestMode`: `"response"`  
     - `aiTopics`: `"true"`  
     - `prompt`: Use expression referencing input message: `{{$json["Your message"]}}`  
   - Ensure `Send Body` is enabled and parameters are sent as JSON.  
   - Connect the output of `On form submission` to this node’s input.

4. **Add a Form node:**  
   - Set node name: `Form`  
   - Configure webhook ID (auto-generated)  
   - Set operation: `completion`  
   - Set completion title: `Your response`  
   - Set completion message to expression: `{{$json.aiAdvice[0].text}}`  
     - This extracts the AI advice text from the API response.  
   - Paste the provided custom CSS (or create your own) for styling the form and response.  
   - Connect the output of `AI Response` to this node’s input.

5. **Add optional Sticky Note nodes for documentation:**  
   - Add one sticky note with instructions to create the InfraNodus graph and set the `name` parameter accordingly.  
   - Add another sticky note explaining how to share the public URL of the deployed chatbot.

6. **Activate the workflow when ready.**

7. **Test the chatbot by submitting messages to the form URL generated by the `On form submission` trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                          |
|----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| Create your knowledge base graph in InfraNodus at https://infranodus.com to power the AI responses with structured data.        | https://infranodus.com                   |
| The workflow uses InfraNodus API with bearer token authentication; ensure your token is valid and has appropriate permissions.    | InfraNodus API documentation (internal)|
| Customize the form’s CSS for branding or UX improvements as needed; CSS variables are defined for colors, fonts, and spacing.    | CSS customization within Form node      |
| Share the workflow’s public URL from the production environment to allow external users to interact with the chatbot.            | Workflow deployment environment          |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.