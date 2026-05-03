Generate Recipes from Fridge Photos using GPT-4 Vision & Telegram

https://n8nworkflows.xyz/workflows/generate-recipes-from-fridge-photos-using-gpt-4-vision---telegram-9558


# Generate Recipes from Fridge Photos using GPT-4 Vision & Telegram

### 1. Workflow Overview

This workflow enables users to generate cooking recipes from either images of their fridge contents or text lists of ingredients sent via Telegram. It leverages GPT-4 Vision capabilities for image analysis and GPT language models for recipe creation, supporting Japanese and international cuisine with structured, detailed outputs.

The workflow is logically divided into four primary blocks:

- **1.1 User Input via Telegram**  
  Listens for Telegram messages containing either a photo or text ingredient list.

- **1.2 Ingredient Extraction (Dual Path)**  
  Processes the input based on type: applies AI vision to photos to identify ingredients or directly uses text input ingredients.

- **1.3 AI Recipe Generation**  
  Uses the standardized ingredient list to prompt an AI agent that generates detailed, structured recipe suggestions.

- **1.4 Format & Send Response**  
  Formats the AI-generated recipe data into a user-friendly message and sends it back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input via Telegram

- **Overview:**  
  This block triggers the workflow when a user sends a message to the Telegram bot. It determines whether the input is text or an image to route processing accordingly.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Check Input Type  

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger node  
    - *Role:* Starts workflow upon receiving a Telegram message (text or photo).  
    - *Configuration:* Listens for "message" updates; downloads media if present.  
    - *Credentials:* Telegram OAuth2 API credentials.  
    - *Connections:* Outputs to Check Input Type.  
    - *Failures:* Possible issues include Telegram API rate limits or webhook misconfiguration.

  - **Check Input Type**  
    - *Type:* IF node  
    - *Role:* Checks if the incoming Telegram message contains text.  
    - *Configuration:* Condition checks if the property `message.text` exists; if yes, routes to text processing, else assumes image input.  
    - *Connections:*  
      - True branch (text input) → Get Text Ingredients  
      - False branch (image input) → AI Vision Agent  
    - *Failures:* If message format changes or is malformed, condition evaluation may fail.

---

#### 2.2 Ingredient Extraction (Dual Path)

- **Overview:**  
  Converts input into a standardized ingredients list. Two parallel paths handle image-based and text-based inputs, converging on a common format.

- **Nodes Involved:**  
  - AI Vision Agent  
  - OpenAI Vision Model (used internally by AI Vision Agent)  
  - Ingredient Parser  
  - Format Ingredients  
  - Get Text Ingredients  

- **Node Details:**

  - **AI Vision Agent**  
    - *Type:* Langchain Agent node  
    - *Role:* Uses GPT-4 Vision to analyze an image URL and identify visible ingredients.  
    - *Configuration:*  
      - System message positions the AI as a food ingredient expert.  
      - Prompt includes the Telegram image message ID as the URL (assumed to be accessible).  
      - Output parser enabled for structured results.  
    - *Connected LLM:* OpenAI Vision Model.  
    - *Connections:* Output to Ingredient Parser.  
    - *Failures:*  
      - Image URL might be inaccessible or invalid.  
      - Vision model API errors or timeouts.  
      - Misidentification of ingredients.

  - **OpenAI Vision Model**  
    - *Type:* Language Model node (OpenRouter GPT-4 Vision Preview)  
    - *Role:* Actual LLM processing for AI Vision Agent.  
    - *Configuration:* Temperature set to 0.3 for stable outputs.  
    - *Credentials:* OpenRouter API key.  
    - *Connections:* Output to AI Vision Agent.  
    - *Failures:* API quota, network issues.

  - **Ingredient Parser**  
    - *Type:* Langchain Output Parser  
    - *Role:* Parses AI Vision Agent’s output into a JSON object containing an array of ingredient strings.  
    - *Configuration:* Manual JSON schema requiring an `ingredients` array of strings.  
    - *Connections:* Back to AI Vision Agent for output parsing.  
    - *Failures:* Parsing errors if AI output deviates from expected schema.

  - **Format Ingredients**  
    - *Type:* Set node  
    - *Role:* Converts the array of ingredients into a comma-separated string `ingredients_text`.  
    - *Configuration:* Assignment uses expression joining ingredient array with commas.  
    - *Connections:* Output to Recipe Generator.  
    - *Failures:* Empty or malformed ingredient array.

  - **Get Text Ingredients**  
    - *Type:* Set node  
    - *Role:* For text inputs, directly assigns the raw message text as `ingredients_text`.  
    - *Connections:* Output to Recipe Generator.  
    - *Failures:* If text is empty or non-ingredient data is provided.

---

#### 2.3 AI Recipe Generation

- **Overview:**  
  Takes the standardized ingredient list and prompts an AI agent to create three detailed recipes, each with structured metadata and stepwise instructions.

- **Nodes Involved:**  
  - Recipe Generator  
  - OpenAI Recipe Model (used internally by Recipe Generator)  
  - Recipe Parser  

- **Node Details:**

  - **Recipe Generator**  
    - *Type:* Langchain Agent node  
    - *Role:* Acts as a professional chef AI, generating recipes from `ingredients_text`.  
    - *Configuration:*  
      - System message instructs to produce detailed recipes with cooking time, difficulty, instructions, and includes both Japanese and international cuisine.  
      - Prompt dynamically inserts the ingredient list.  
      - Output parser enabled for structured response.  
    - *Connected LLM:* OpenAI Recipe Model.  
    - *Connections:* Output to Recipe Parser and then Format Response.  
    - *Failures:* Model errors, insufficient ingredient detail, or malformed prompt.

  - **OpenAI Recipe Model**  
    - *Type:* Language Model node (OpenRouter GPT-4o-mini)  
    - *Role:* Processes recipe generation prompt.  
    - *Configuration:* Temperature 0.7 for creative output.  
    - *Credentials:* OpenRouter API key.  
    - *Connections:* Output to Recipe Generator.  
    - *Failures:* API limits, network issues.

  - **Recipe Parser**  
    - *Type:* Langchain Output Parser  
    - *Role:* Parses the AI-generated recipes into a structured JSON array of recipe objects, each with fields: name, cuisine, difficulty, cookingTime, servings, ingredients, instructions.  
    - *Configuration:* Manual JSON schema enforces strict recipe object structure.  
    - *Connections:* Output to Recipe Generator for final use.  
    - *Failures:* Parsing errors if AI output deviates from schema.

---

#### 2.4 Format & Send Response

- **Overview:**  
  Formats the parsed recipes into a readable, emoji-enhanced message and sends it back to the user via Telegram.

- **Nodes Involved:**  
  - Format Response  
  - Send a text message  

- **Node Details:**

  - **Format Response**  
    - *Type:* Set node  
    - *Role:* Uses a JavaScript expression to transform the recipe JSON into a multi-line text message containing recipe details with emojis and formatting.  
    - *Configuration:* Expression loops through each recipe, formats metadata and instructions with line breaks and emojis.  
    - *Connections:* Output to Send a text message.  
    - *Failures:* Expression errors if structured data is missing or malformed.

  - **Send a text message**  
    - *Type:* Telegram node  
    - *Role:* Sends the final formatted recipe list back to the Telegram chat.  
    - *Configuration:* Uses chat ID from trigger; disables attribution footer.  
    - *Credentials:* Telegram OAuth2 API credentials.  
    - *Failures:* Telegram API errors, invalid chat ID.

---

### 3. Summary Table

| Node Name             | Node Type                                   | Functional Role                            | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                              |
|-----------------------|---------------------------------------------|--------------------------------------------|-------------------------|-------------------------|---------------------------------------------------------------------------------------------------------|
| Telegram Trigger      | Telegram Trigger                            | Start workflow on Telegram message          | —                       | Check Input Type         | Phase 1: User Input via Telegram - Starts workflow on message/image.                                    |
| Check Input Type      | IF Node                                    | Routes input to image or text processing    | Telegram Trigger         | AI Vision Agent (false branch), Get Text Ingredients (true branch) | Phase 1: User Input via Telegram - Checks message type for routing.                                     |
| AI Vision Agent       | Langchain Agent                            | Analyze image to identify ingredients       | Check Input Type (false) | Ingredient Parser        | Phase 2: Ingredient Extraction - Image path AI analysis.                                                |
| OpenAI Vision Model   | Language Model (OpenRouter GPT-4 Vision)  | LLM for image analysis                       | AI Vision Agent          | AI Vision Agent         | Phase 2: Ingredient Extraction - Vision model for image understanding.                                 |
| Ingredient Parser     | Langchain Output Parser                     | Parse vision AI output into structured list | AI Vision Agent          | Format Ingredients       | Phase 2: Ingredient Extraction - Parsing AI output.                                                    |
| Format Ingredients    | Set Node                                   | Convert ingredients array to CSV string     | Ingredient Parser        | Recipe Generator         | Phase 2: Ingredient Extraction - Formatting ingredients for next step.                                 |
| Get Text Ingredients  | Set Node                                   | Extract ingredients text from message       | Check Input Type (true)  | Recipe Generator         | Phase 2: Ingredient Extraction - Direct text input path.                                              |
| Recipe Generator      | Langchain Agent                            | Generate detailed recipes from ingredients  | Format Ingredients, Get Text Ingredients | Recipe Parser, Format Response | Phase 3: AI Recipe Generation - Main recipe creation AI.                                                |
| OpenAI Recipe Model   | Language Model (OpenRouter GPT-4o-mini)   | LLM for recipe generation                    | Recipe Generator         | Recipe Generator         | Phase 3: AI Recipe Generation - Recipe model for creativity.                                          |
| Recipe Parser        | Langchain Output Parser                     | Parse recipe AI output into structured JSON | Recipe Generator         | Recipe Generator         | Phase 3: AI Recipe Generation - Ensures structured recipe data.                                       |
| Format Response       | Set Node                                   | Format parsed recipes into user-friendly text | Recipe Generator         | Send a text message      | Phase 4: Format & Send Response - Formatting final message with emojis and structure.                 |
| Send a text message   | Telegram Node                              | Send final recipe text back to Telegram user | Format Response          | —                       | Phase 4: Format & Send Response - Sends the response message.                                         |
| Sticky Note           | Sticky Note                                | Documentation for Phase 1                    | —                       | —                       | Phase 1: User Input via Telegram - Explains trigger and input type check.                              |
| Sticky Note1          | Sticky Note                                | Documentation for Phase 2                    | —                       | —                       | Phase 2: Ingredient Extraction - Explains dual path image/text extraction.                            |
| Sticky Note2          | Sticky Note                                | Documentation for Phase 3                    | —                       | —                       | Phase 3: AI Recipe Generation - Explains AI recipe creation and parsing.                              |
| Sticky Note3          | Sticky Note                                | Documentation for Phase 4                    | —                       | —                       | Phase 4: Format & Send Response - Explains formatting and response sending.                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates, enable media download.  
   - Credentials: Configure Telegram API OAuth2 credentials.  
   - Position: Start of workflow.

2. **Add Check Input Type IF node:**  
   - Type: IF node  
   - Condition: Check if `message.text` field exists (string not exists = false branch means image).  
   - Connect Telegram Trigger main output to Check Input Type input.

3. **Set up AI Vision Agent:**  
   - Type: Langchain Agent node  
   - Prompt:  
     ```
     Please analyze this food image and identify all visible ingredients. List them clearly.

     Image URL: {{ $json.body.events[0].message.id }}
     ```  
   - System Message: "You are a food ingredient identification expert. Analyze food images and identify all visible ingredients accurately. Return a structured list of ingredients."  
   - Enable output parser.  
   - Connect IF node false branch (image input) to AI Vision Agent.

4. **Configure OpenAI Vision Model node:**  
   - Type: Language Model (OpenRouter GPT-4 Vision Preview)  
   - Temperature: 0.3  
   - Credentials: OpenRouter API key.  
   - Connect AI Vision Agent’s ai_languageModel input to OpenAI Vision Model output.

5. **Add Ingredient Parser node:**  
   - Type: Langchain Output Parser  
   - Input schema:  
     ```json
     {
       "type": "object",
       "properties": {
         "ingredients": {
           "type": "array",
           "items": { "type": "string" },
           "description": "List of identified ingredients"
         }
       },
       "required": ["ingredients"]
     }
     ```  
   - Connect AI Vision Agent’s ai_outputParser output to Ingredient Parser input.

6. **Add Format Ingredients node:**  
   - Type: Set node  
   - Create field `ingredients_text` with expression:  
     `={{ $json.output.ingredients.join(', ') }}`  
   - Connect Ingredient Parser output to Format Ingredients.

7. **Add Get Text Ingredients node:**  
   - Type: Set node  
   - Create field `ingredients_text` with expression:  
     `={{ $json.message.text }}`  
   - Connect IF node true branch (text input) to Get Text Ingredients.

8. **Add Recipe Generator node:**  
   - Type: Langchain Agent node  
   - Prompt:  
     ```
     Based on these ingredients: {{ $json.ingredients_text }}

     Please suggest 3 delicious recipes that can be made with these ingredients. Consider both Japanese and international cuisine options. Text is Japanese.
     ```  
   - System Message:  
     "You are a professional chef and recipe expert. Create detailed, easy-to-follow recipes based on available ingredients. Include both traditional Japanese dishes and international cuisine. Always provide cooking time, difficulty level, and step-by-step instructions."  
   - Enable output parser.  
   - Connect Format Ingredients and Get Text Ingredients outputs to Recipe Generator input.

9. **Configure OpenAI Recipe Model node:**  
   - Type: Language Model (OpenRouter GPT-4o-mini)  
   - Temperature: 0.7  
   - Credentials: OpenRouter API key.  
   - Connect Recipe Generator’s ai_languageModel input to OpenAI Recipe Model output.

10. **Add Recipe Parser node:**  
    - Type: Langchain Output Parser  
    - Input schema:  
      ```json
      {
        "type": "object",
        "properties": {
          "recipes": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "name": { "type": "string" },
                "cuisine": { "type": "string" },
                "difficulty": { "type": "string" },
                "cookingTime": { "type": "string" },
                "servings": { "type": "number" },
                "ingredients": { "type": "array", "items": { "type": "string" } },
                "instructions": { "type": "array", "items": { "type": "string" } }
              },
              "required": ["name", "cuisine", "difficulty", "cookingTime", "servings", "ingredients", "instructions"]
            }
          }
        },
        "required": ["recipes"]
      }
      ```  
    - Connect Recipe Generator’s ai_outputParser output to Recipe Parser input.

11. **Add Format Response node:**  
    - Type: Set node  
    - Create field `message` with JavaScript expression:  
      ```javascript
      return $json.output.recipes.map((r, index) => {
        return (
          `📖 レシピ${index + 1}：${r.name}\n` +
          `🌐 ジャンル：${r.cuisine}\n` +
          `⏱ 調理時間：${r.cookingTime}\n` +
          `🎚 難易度：${r.difficulty}\n` +
          `👥 目安人数：${r.servings}人分\n\n` +
          `🛒 材料:\n${r.ingredients.map(i => `- ${i}`).join("\n")}\n\n` +
          `👣 手順:\n${r.instructions.map((step, i) => `${i + 1}. ${step}`).join("\n")}`
        );
      }).join("\n\n------------------------\n\n");
      ```  
    - Connect Recipe Generator output to Format Response.

12. **Add Send a Text Message node:**  
    - Type: Telegram node  
    - Parameters:  
      - Text: `={{ $json.message }}`  
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
      - Disable attribution footer.  
    - Credentials: Telegram API OAuth2.  
    - Connect Format Response output to Send a text message input.

13. **Set workflow timezone to Asia/Tokyo and enable saving executions and errors for debugging.**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                  |
|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow supports both image and text inputs from Telegram users for flexible ingredient input.                      | Workflow design                                                  |
| Uses GPT-4 Vision Preview via OpenRouter to analyze images for ingredients.                                          | OpenRouter API, GPT-4 Vision Preview                            |
| Recipe generation prompt and system message are tailored to produce Japanese and international cuisine recipes.     | Recipe Generator prompt                                          |
| Output parsers enforce structured JSON for predictable downstream processing and formatting.                         | Langchain Output Parser nodes                                    |
| Final response includes emojis and Japanese text formatting for user-friendly presentation.                          | Format Response node                                             |
| Telegram credentials must have appropriate permissions to receive messages and send replies with media.             | Telegram API setup                                              |
| Workflow timezone set to Asia/Tokyo to align with expected user base time.                                           | Workflow settings                                               |
| Sticky notes in workflow provide phase explanations and can be referenced for quick understanding.                  | Workflow documentation                                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.