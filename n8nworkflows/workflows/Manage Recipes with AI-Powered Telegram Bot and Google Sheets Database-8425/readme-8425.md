Manage Recipes with AI-Powered Telegram Bot and Google Sheets Database

https://n8nworkflows.xyz/workflows/manage-recipes-with-ai-powered-telegram-bot-and-google-sheets-database-8425


# Manage Recipes with AI-Powered Telegram Bot and Google Sheets Database

### 1. Workflow Overview

This workflow, titled **"🍳 Culinary Companion — 30-Step Workflow (Telegram + Google Sheets + LLM)"**, is designed to manage recipes via an AI-powered Telegram bot integrated with a Google Sheets database. It facilitates recipe querying, creation, modification, and AI-assisted interaction, leveraging large language models (LLMs) to enhance user experience.

**Target Use Cases:**
- Users interact with a Telegram bot to browse recipe names, select recipes, and receive full recipe details.
- Users can create or modify recipes using natural language processed by AI agents.
- Recipe data is stored and managed in Google Sheets, acting as a central database.
- AI language models provide conversational capabilities and intelligent parsing of user inputs.

**Logical Blocks:**

- **1.1 Input Reception and Command Routing:** Handles incoming Telegram messages and routes commands appropriately.
- **1.2 Recipe Listing and Selection:** Reads recipe names from Google Sheets, formats them, sends lists, and processes user selection.
- **1.3 Recipe Detail Retrieval and Presentation:** Fetches full recipe details from Google Sheets and sends formatted recipes to users.
- **1.4 Recipe Creation and Modification Preparation:** Prepares data and context for creating or modifying recipes.
- **1.5 AI Interaction and Processing:** Uses LangChain agents connected to OpenAI and Google Gemini models to process AI-driven recipe management actions and conversations.
- **1.6 Google Sheets Read/Update Operations:** Tools to read from and update the recipes sheet as part of AI agent operations.
- **1.7 Messaging Responses:** Sends various Telegram messages including menus, recipe lists, full recipes, and AI-generated responses.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Command Routing

**Overview:**  
This block captures Telegram user messages via a trigger node and uses a switch node to route commands to different workflow paths based on user input.

**Nodes Involved:**  
- Telegram Trigger  
- Command Router  

**Node Details:**  
- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages to start workflow execution.  
  - Configuration: Default trigger listening on configured Telegram bot.  
  - Inputs: None (external trigger)  
  - Outputs: Connected to Command Router.  
  - Edge Cases: Telegram API downtime, invalid message formats, user blocking the bot.  

- **Command Router**  
  - Type: Switch  
  - Role: Routes the workflow based on parsed command or message content.  
  - Configuration: Switch on message text or command to select from multiple possible paths (menu, list, select, create, modify, search).  
  - Inputs: From Telegram Trigger.  
  - Outputs: Six outputs connected to different functional blocks.  
  - Edge Cases: Unrecognized commands, empty or malformed messages, unexpected user input.  

---

#### 1.2 Recipe Listing and Selection

**Overview:**  
Reads recipe names from Google Sheets, formats the list as a message, sends it via Telegram, and processes user selection for detailed lookup.

**Nodes Involved:**  
- Read Recipe Names (Google Sheets)  
- Format List Message (Function)  
- Telegram Send Recipe List  
- Extract Select Param (Function)  
- Lookup Recipe Row (Google Sheets)  

**Node Details:**  
- **Read Recipe Names**  
  - Type: Google Sheets  
  - Role: Reads the sheet containing recipe names to provide a list for users.  
  - Configuration: Reads a specific range or sheet with recipe names.  
  - Inputs: From Command Router (list command)  
  - Outputs: To Format List Message.  
  - Edge Cases: Sheet unavailable, permission denied, empty data range.  

- **Format List Message**  
  - Type: Function  
  - Role: Formats the list of recipe names into a user-friendly message string.  
  - Configuration: Uses JavaScript to concatenate recipe names with formatting.  
  - Inputs: From Read Recipe Names.  
  - Outputs: To Telegram Send Recipe List.  
  - Edge Cases: Empty input data, malformed sheet rows.  

- **Telegram Send Recipe List**  
  - Type: Telegram  
  - Role: Sends the formatted recipe list message to the user.  
  - Configuration: Uses chat ID from trigger context, sends text message.  
  - Inputs: From Format List Message.  
  - Outputs: None (end of this path)  
  - Edge Cases: Telegram send failures, invalid chat ID.  

- **Extract Select Param**  
  - Type: Function  
  - Role: Extracts the user selection parameter (e.g., recipe name or ID) from the Telegram message.  
  - Configuration: Parses message text to identify which recipe user selected.  
  - Inputs: From Command Router (select command)  
  - Outputs: To Lookup Recipe Row.  
  - Edge Cases: User input not matching any recipe, parsing errors.  

- **Lookup Recipe Row**  
  - Type: Google Sheets  
  - Role: Reads the full row data for the selected recipe from the sheet.  
  - Configuration: Uses extracted parameter to query the correct row.  
  - Inputs: From Extract Select Param.  
  - Outputs: To Format Full Recipe.  
  - Edge Cases: Recipe not found, sheet read errors.  

---

#### 1.3 Recipe Detail Retrieval and Presentation

**Overview:**  
Formats the full recipe information and sends it to the user via Telegram.

**Nodes Involved:**  
- Format Full Recipe (Function)  
- Telegram Send Full Recipe  

**Node Details:**  
- **Format Full Recipe**  
  - Type: Function  
  - Role: Converts the raw row data for a recipe into a nicely formatted message.  
  - Configuration: Generates multiline text with recipe title, ingredients, instructions, etc.  
  - Inputs: From Lookup Recipe Row.  
  - Outputs: To Telegram Send Full Recipe.  
  - Edge Cases: Missing fields, unexpected data formats.  

- **Telegram Send Full Recipe**  
  - Type: Telegram  
  - Role: Sends the formatted recipe details to the user.  
  - Configuration: Uses chat ID, sends text message.  
  - Inputs: From Format Full Recipe.  
  - Outputs: None (end of path)  
  - Edge Cases: Telegram API failures, invalid chat ID.  

---

#### 1.4 Recipe Creation and Modification Preparation

**Overview:**  
Prepares inputs and context for creating or modifying recipes by the user, directing flow towards AI processing.

**Nodes Involved:**  
- Prepare Recipe Creation (Function)  
- Prepare Recipe Modification (Function)  

**Node Details:**  
- **Prepare Recipe Creation**  
  - Type: Function  
  - Role: Sets up necessary parameters and context for creating a new recipe.  
  - Configuration: Extracts and structures user input for AI processing.  
  - Inputs: From Command Router (create command)  
  - Outputs: To Recipe AI Agent.  
  - Edge Cases: Insufficient input detail, malformed user request.  

- **Prepare Recipe Modification**  
  - Type: Function  
  - Role: Prepares data and context for modifying an existing recipe.  
  - Configuration: Parses user input for modification intent, recipe reference.  
  - Inputs: From Command Router (modify command)  
  - Outputs: To Recipe AI Agent.  
  - Edge Cases: Recipe not specified or ambiguous, input parsing errors.  

---

#### 1.5 AI Interaction and Processing

**Overview:**  
Handles AI-driven conversational logic and recipe management using LangChain agents connected with OpenAI and Google Gemini language models, including memory management and tool usage for Google Sheets integration.

**Nodes Involved:**  
- Recipe AI Agent (LangChain agent)  
- Google Gemini Chat Model  
- OpenAI Chat Model  
- Conversation Memory (Buffer Window)  
- Think (LangChain tool Think)  
- Read Recipes Sheet (Google Sheets Tool)  
- Update Recipes Sheet (Google Sheets Tool)  
- Extract Search Term (Function)  
- Telegram Send AI Response (Telegram)  

**Node Details:**  
- **Recipe AI Agent**  
  - Type: LangChain Agent  
  - Role: Central AI agent managing recipe-related NLP tasks, integrating memory and tools.  
  - Configuration: Uses AI language model nodes for processing, memory node for context, and Google Sheets tools for DB interaction.  
  - Inputs: From Prepare Recipe Creation, Prepare Recipe Modification, Extract Search Term, Think node.  
  - Outputs: To Telegram Send AI Response, acts as ai_tool for Google Sheets Tool nodes.  
  - Edge Cases: API rate limits, model timeouts, malformed AI responses.  

- **Google Gemini Chat Model**  
  - Type: LangChain Chat Model (Google Gemini)  
  - Role: Provides AI language model capabilities to the Recipe AI Agent.  
  - Configuration: Configured with Google Gemini credentials and parameters.  
  - Inputs: Connected as ai_languageModel to Recipe AI Agent.  
  - Outputs: None (used internally by agent).  
  - Edge Cases: Authentication errors, API limits.  

- **OpenAI Chat Model**  
  - Type: LangChain Chat Model (OpenAI)  
  - Role: Alternative or complementary AI language model for the agent.  
  - Configuration: Uses OpenAI credentials and parameters.  
  - Inputs: Connected as ai_languageModel to Recipe AI Agent.  
  - Outputs: None.  
  - Edge Cases: API quota, invalid API keys.  

- **Conversation Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context within agent sessions.  
  - Configuration: Configured with window size and persistence.  
  - Inputs: Linked as ai_memory to Recipe AI Agent.  
  - Outputs: None.  
  - Edge Cases: Memory overflow, context loss over long conversations.  

- **Think**  
  - Type: LangChain Tool Think  
  - Role: Sub-agent or tool node to perform internal reasoning or external calls before AI response.  
  - Configuration: Triggered within AI agent flow, outputs to Recipe AI Agent.  
  - Inputs: From Recipe AI Agent.  
  - Outputs: To Recipe AI Agent.  
  - Edge Cases: Execution timeout, tool failure.  

- **Read Recipes Sheet**  
  - Type: Google Sheets Tool  
  - Role: AI agent tool to read recipe data from Google Sheets on demand.  
  - Configuration: Configured with sheet and range info, authorized credentials.  
  - Inputs: From Recipe AI Agent (ai_tool).  
  - Outputs: To Recipe AI Agent.  
  - Edge Cases: Access issues, sheet not found.  

- **Update Recipes Sheet**  
  - Type: Google Sheets Tool  
  - Role: AI agent tool to update recipe data in Google Sheets.  
  - Configuration: Configured with target sheet, authorized credentials.  
  - Inputs: From Recipe AI Agent (ai_tool).  
  - Outputs: To Recipe AI Agent.  
  - Edge Cases: Write permission denied, data conflicts.  

- **Extract Search Term**  
  - Type: Function  
  - Role: Extracts search keywords from user input for AI agent processing.  
  - Configuration: Parsing text for relevant terms.  
  - Inputs: From Command Router (search command)  
  - Outputs: To Recipe AI Agent.  
  - Edge Cases: Ambiguous input, empty queries.  

- **Telegram Send AI Response**  
  - Type: Telegram  
  - Role: Sends AI-generated messages back to the Telegram user.  
  - Configuration: Uses chat ID from context.  
  - Inputs: From Recipe AI Agent.  
  - Outputs: None.  
  - Edge Cases: Message send failures, invalid context.  

---

#### 1.6 Google Sheets Read/Update Operations (Within AI Agent)

**Overview:**  
These nodes are integrated as tools to the AI agent to facilitate dynamic reading and updating of the Google Sheets recipe database during AI conversations.

**Nodes Involved:**  
- Read Recipes Sheet  
- Update Recipes Sheet  

**Node Details:**  
- Covered above in AI Interaction section.

---

#### 1.7 Messaging Responses

**Overview:**  
This block handles sending various Telegram messages: main menu, recipe lists, full recipes, and AI replies.

**Nodes Involved:**  
- Telegram Send Menu  
- Telegram Send Recipe List  
- Telegram Send Full Recipe  
- Telegram Send AI Response  

**Node Details:**  
- **Telegram Send Menu**  
  - Sends the main menu or help message to the user upon command routing.  

- **Telegram Send Recipe List**  
  - Sends formatted recipe name list (covered above).  

- **Telegram Send Full Recipe**  
  - Sends detailed recipe content (covered above).  

- **Telegram Send AI Response**  
  - Sends AI-generated replies (covered above).  

All configured with appropriate chat IDs and message content from upstream nodes.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                          | Input Node(s)          | Output Node(s)              | Sticky Note                          |
|------------------------|----------------------------------|----------------------------------------|-----------------------|-----------------------------|------------------------------------|
| Telegram Trigger       | Telegram Trigger                  | Entry point: receives Telegram messages | None                  | Command Router               |                                    |
| Command Router         | Switch                           | Routes commands to workflow blocks     | Telegram Trigger       | Telegram Send Menu, Read Recipe Names, Extract Select Param, Prepare Recipe Modification, Prepare Recipe Creation, Extract Search Term |                                    |
| Telegram Send Menu     | Telegram                         | Sends main menu to user                 | Command Router         | None                        |                                    |
| Read Recipe Names      | Google Sheets                    | Reads recipe names list from sheet     | Command Router         | Format List Message          |                                    |
| Format List Message    | Function                        | Formats recipe names list message      | Read Recipe Names      | Telegram Send Recipe List    |                                    |
| Telegram Send Recipe List | Telegram                       | Sends recipe list message               | Format List Message    | None                        |                                    |
| Extract Select Param   | Function                        | Parses user selection input             | Command Router         | Lookup Recipe Row            |                                    |
| Lookup Recipe Row      | Google Sheets                   | Retrieves full recipe details           | Extract Select Param   | Format Full Recipe           |                                    |
| Format Full Recipe     | Function                        | Formats full recipe for message         | Lookup Recipe Row      | Telegram Send Full Recipe    |                                    |
| Telegram Send Full Recipe | Telegram                      | Sends full recipe to user                | Format Full Recipe     | None                        |                                    |
| Prepare Recipe Creation | Function                       | Prepares data for new recipe creation   | Command Router         | Recipe AI Agent              |                                    |
| Prepare Recipe Modification | Function                   | Prepares data for modifying recipe      | Command Router         | Recipe AI Agent              |                                    |
| Extract Search Term    | Function                        | Extracts search keywords for AI         | Command Router         | Recipe AI Agent              |                                    |
| Recipe AI Agent        | LangChain Agent                 | AI logic for recipe management and conversation | Prepare Recipe Creation, Prepare Recipe Modification, Extract Search Term, Think, Read Recipes Sheet, Update Recipes Sheet, Conversation Memory, Google Gemini Chat Model, OpenAI Chat Model | Telegram Send AI Response          |                                    |
| Google Gemini Chat Model | LangChain Chat Model           | AI language model (Google Gemini)       | None                  | Recipe AI Agent (ai_languageModel) |                                    |
| OpenAI Chat Model      | LangChain Chat Model            | AI language model (OpenAI)               | None                  | Recipe AI Agent (ai_languageModel) |                                    |
| Conversation Memory    | LangChain Memory Buffer Window | Maintains conversation context          | None                  | Recipe AI Agent (ai_memory)  |                                    |
| Think                  | LangChain Tool Think            | AI sub-tool for internal reasoning      | Recipe AI Agent        | Recipe AI Agent              |                                    |
| Read Recipes Sheet     | Google Sheets Tool              | Reads recipe database sheet             | Recipe AI Agent (ai_tool) | Recipe AI Agent             |                                    |
| Update Recipes Sheet   | Google Sheets Tool              | Updates recipe database sheet           | Recipe AI Agent (ai_tool) | Recipe AI Agent             |                                    |
| Telegram Send AI Response | Telegram                     | Sends AI-generated reply                 | Recipe AI Agent        | None                        |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure credentials for your Telegram bot.  
   - Set to listen to all messages or commands as needed.  

2. **Add a Switch node named "Command Router"**  
   - Connect Telegram Trigger output to Command Router input.  
   - Configure switch conditions based on message commands or text (e.g., "/menu", "/list", "/select", "/create", "/modify", "/search").  
   - Create six outputs corresponding to these commands.  

3. **Telegram Send Menu node**  
   - Add Telegram node to send main menu message.  
   - Connect from Command Router output for menu command.  
   - Configure to use chat ID from Telegram Trigger context, set static or templated menu text.  

4. **Recipe Listing Path:**  
   - Add Google Sheets node "Read Recipe Names"  
     - Connect from Command Router list output.  
     - Configure Google Sheets credentials and specify sheet/range with recipe names.  
   - Add Function node "Format List Message"  
     - Connect from Read Recipe Names output.  
     - Write JavaScript to format recipe names into a readable list message.  
   - Add Telegram node "Telegram Send Recipe List"  
     - Connect from Format List Message output.  
     - Send formatted list to user.  

5. **Recipe Selection Path:**  
   - Add Function node "Extract Select Param"  
     - Connect from Command Router select output.  
     - Parse user message to extract recipe identifier.  
   - Add Google Sheets node "Lookup Recipe Row"  
     - Connect from Extract Select Param output.  
     - Configure to read full recipe row matching extracted parameter.  
   - Add Function node "Format Full Recipe"  
     - Connect from Lookup Recipe Row output.  
     - Format full recipe details for messaging.  
   - Add Telegram node "Telegram Send Full Recipe"  
     - Connect from Format Full Recipe output.  
     - Send detailed recipe to user.  

6. **Recipe Creation Path:**  
   - Add Function node "Prepare Recipe Creation"  
     - Connect from Command Router create output.  
     - Prepare structured data from user input for AI agent.  
   - Connect output to "Recipe AI Agent" node (LangChain agent).  

7. **Recipe Modification Path:**  
   - Add Function node "Prepare Recipe Modification"  
     - Connect from Command Router modify output.  
     - Prepare structured data for modification.  
   - Connect output to "Recipe AI Agent".  

8. **Search Path:**  
   - Add Function node "Extract Search Term"  
     - Connect from Command Router search output.  
     - Extract search keywords.  
   - Connect output to "Recipe AI Agent".  

9. **Add LangChain Agent node "Recipe AI Agent"**  
   - Configure with AI language models (connect OpenAI Chat Model and Google Gemini Chat Model nodes as ai_languageModel inputs).  
   - Add Conversation Memory node connected as ai_memory input.  
   - Add Google Sheets Tool nodes ("Read Recipes Sheet" and "Update Recipes Sheet") connected as ai_tool inputs.  
   - Connect outputs to Telegram node "Telegram Send AI Response".  

10. **Add Google Gemini Chat Model node**  
    - Configure with Google AI API credentials.  
    - Connect as ai_languageModel to Recipe AI Agent.  

11. **Add OpenAI Chat Model node**  
    - Configure with OpenAI API credentials.  
    - Connect as ai_languageModel to Recipe AI Agent.  

12. **Add Conversation Memory node**  
    - Set buffer window size and persistence settings.  
    - Connect to Recipe AI Agent as memory.  

13. **Add Think node**  
    - Connect as ai_tool from Recipe AI Agent.  
    - Connect Think node output back to Recipe AI Agent input.  

14. **Add Google Sheets Tool nodes**  
    - "Read Recipes Sheet" configured to read recipe data.  
    - "Update Recipes Sheet" configured to update recipe data.  
    - Both connected as ai_tool to Recipe AI Agent.  

15. **Add Telegram node "Telegram Send AI Response"**  
    - Connect from Recipe AI Agent main output.  
    - Sends AI-generated messages to user.  

16. **Credentials Setup:**  
    - Telegram Bot OAuth2 credentials.  
    - Google Sheets API credentials with read/write scopes.  
    - OpenAI API key.  
    - Google Gemini AI credentials.  

17. **Test and Validate:**  
    - Test each command path independently.  
    - Validate Google Sheets reading and writing.  
    - Ensure AI agent responses are coherent and relevant.  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                            |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| The workflow uses LangChain integration for AI agents, enabling advanced conversational and tool use.   | https://docs.n8n.io/integrations/builtin/ai/langchain/     |
| Google Sheets acts as the primary recipe database, requiring proper API credentials and sheet sharing.   | Google Sheets API documentation                             |
| Telegram Bot must be set up with BotFather and proper webhook configuration in n8n.                      | https://core.telegram.org/bots/api                         |
| For AI language models, ensure API keys are valid and rate limits considered to avoid failures.          | OpenAI and Google Gemini AI API docs                        |
| Memory buffer in LangChain is used to maintain context over conversations, enhancing user experience.   | LangChain conversation memory concepts                      |
| Workflow is modular: commands routed cleanly for extensibility and maintenance.                           | Best practice in n8n workflows                              |

---

*Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, a no-code automation tool. It fully complies with content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.*