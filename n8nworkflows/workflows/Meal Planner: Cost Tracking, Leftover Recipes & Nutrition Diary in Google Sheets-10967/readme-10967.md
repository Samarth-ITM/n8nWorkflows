Meal Planner: Cost Tracking, Leftover Recipes & Nutrition Diary in Google Sheets

https://n8nworkflows.xyz/workflows/meal-planner--cost-tracking--leftover-recipes---nutrition-diary-in-google-sheets-10967


# Meal Planner: Cost Tracking, Leftover Recipes & Nutrition Diary in Google Sheets

### 1. Workflow Overview

This workflow is a comprehensive Meal Planner assistant designed to automate cost tracking, leftover ingredient management, and nutritional diary creation using Google Sheets and AI-driven text analysis. It targets users who want to streamline meal planning and food budgeting while minimizing waste and promoting nutritional awareness. Additionally, it generates social media posts for sharing nutrition-related insights.

The logical flow is divided into the following blocks:

- **1.1 Input Reception:** Receives new meal/menu data via a webhook.
- **1.2 Menu Analysis & Cost Tracking:** Uses an AI agent to parse the menu input, extracting detailed ingredient and cost information; appends this data to a Google Sheet.
- **1.3 Leftover Ingredient Processing:** Identifies leftover ingredients and suggests recipes to use them; logs these suggestions into a Google Sheet.
- **1.4 Nutrition Diary Generation:** Creates a diary-style nutritional entry based on the menu; appends it to a Google Sheet.
- **1.5 Social Media Post Creation & Sending:** Converts the nutrition diary into an engaging social media post and sends it as a direct message via Twitter.

The workflow integrates multiple AI agents using OpenRouter language models and relies on Google Sheets for data storage and Twitter for social posting.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives new meal or menu data via a webhook, triggering the entire workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook node (n8n-nodes-base.webhook)  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Custom path configured by user (e.g., "your-webhook-path")  
      - Accepts incoming requests with new menu details as JSON payloads.  
    - Input: External HTTP POST request  
    - Output: JSON body of the request passed downstream  
    - Edge Cases:  
      - Invalid or missing payload may cause failure downstream.  
      - Unauthorized external calls if webhook URL is exposed.  
    - Version: 2.1  
    - Sticky Note: Describes need to copy the webhook URL and send menu item input to it.

---

#### 1.2 Menu Analysis & Cost Tracking

- **Overview:**  
Analyzes the input menu item using an AI agent to extract ingredients, costs, unit prices, quantities, and leftover information, then appends this structured data into the "Recipe" Google Sheet.

- **Nodes Involved:**  
  - OpenRouter Chat Model (Main language model for Menu Agent)  
  - Structured Output Parser  
  - Menu Agent (Langchain agent)  
  - Append row in sheet (for Recipe List)

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: Language model node (OpenRouter via Langchain integration)  
    - Role: Provides AI language model capabilities for text analysis  
    - Configuration: Default options, connected as AI model for Menu Agent  
    - Edge Cases: API key missing or invalid, network timeout, rate limits  
    - Version: 1

  - **Structured Output Parser**  
    - Type: Output parser node (Langchain structured output parser)  
    - Role: Parses AI response into a structured JSON format matching the schema for ingredients and costs  
    - Configuration: Uses a JSON schema example defining keys like Date, Item, Ingredients, Ingredient Cost, Unit Price, Quantity, Total, Cost, Leftover Ingredients  
    - Input: AI response text from Menu Agent  
    - Output: Structured JSON array with ingredient cost details  
    - Edge Cases: Parsing errors if AI output deviates from schema, malformed JSON, empty responses  
    - Version: 1.3

  - **Menu Agent**  
    - Type: Langchain Agent node  
    - Role: Core AI agent interpreting the menu input and extracting detailed ingredient and cost info  
    - Configuration:  
      - Input text: Raw menu JSON from webhook  
      - System message instructs the agent to act as a housekeeper extracting ingredient and cost details, ignoring seasoning cost per use, providing unit prices and totals with units  
      - Uses Structured Output Parser for output  
    - Input: Webhook JSON body  
    - Output: Structured ingredient cost JSON  
    - Edge Cases: Misinterpretation of input text, incomplete data extraction, API errors  
    - Version: 3

  - **Append row in sheet**  
    - Type: Google Sheets node (append operation)  
    - Role: Inserts each ingredient and cost record as a new row into the "Recipe" sheet  
    - Configuration:  
      - Document ID: User-supplied Google Sheet ID (e.g., "Recipe List")  
      - Sheet Name: "Recipe" (レシピ)  
      - Columns mapped from parsed output fields such as Date, Item, Ingredients, Ingredient Cost, Unit Price, Quantity, Total, Cost, Leftover Ingredients  
      - Cell format: USER_ENTERED  
    - Input: Structured JSON from Menu Agent output  
    - Output: Append confirmation  
    - Edge Cases: Credential issues, incorrect sheet name, quota limits  
    - Version: 4.7  
    - Sticky Note: Details about Google Sheets setup and required columns.

---

#### 1.3 Leftover Ingredient Processing

- **Overview:**  
Analyzes leftover ingredients from the menu and suggests three recipes using them; appends these suggestions to the "leftovers" Google Sheet.

- **Nodes Involved:**  
  - OpenRouter Chat Model3 (AI model for Leftovers Agent)  
  - Leftovers Agent (Langchain agent)  
  - Append row in sheet2

- **Node Details:**

  - **OpenRouter Chat Model3**  
    - Type: Language model node  
    - Role: Provides AI model for Leftovers Agent  
    - Configuration: Default options  
    - Version: 1

  - **Leftovers Agent**  
    - Type: Langchain Agent node  
    - Role: AI agent specializing in identifying leftover ingredients and suggesting three recipes without cooking instructions  
    - Configuration:  
      - Input text: Output JSON from Menu Agent (ingredient list)  
      - System message instructs focus on leftover ingredients and recipe suggestions only  
    - Input: Parsed ingredient/cost list from Menu Agent  
    - Output: Text listing leftover ingredients and suggested recipes  
    - Edge Cases: Ambiguous leftovers, incomplete ingredient mapping, API errors  
    - Version: 3

  - **Append row in sheet2**  
    - Type: Google Sheets node (append operation)  
    - Role: Appends leftover recipes and ingredient suggestions to the "leftovers" sheet  
    - Configuration:  
      - Document ID: Same Google Sheet as before  
      - Sheet Name: "leftovers"  
      - Columns: Date (from Menu Agent output), Ingredients (from Leftovers Agent output)  
    - Input: Leftovers Agent output text  
    - Edge Cases: Credential issues, sheet name mismatches  
    - Version: 4.7

---

#### 1.4 Nutrition Diary Generation

- **Overview:**  
Generates a diary-style entry with dietary advice and nutritional information based on the meal; appends this to the "diary" Google Sheet.

- **Nodes Involved:**  
  - OpenRouter Chat Model2 (AI model for Nutritionist Agent)  
  - Nutritionist Agent (Langchain agent)  
  - Append row in sheet1

- **Node Details:**

  - **OpenRouter Chat Model2**  
    - Type: Language model node  
    - Role: Provides AI model for Nutritionist Agent  
    - Configuration: Default options  
    - Version: 1

  - **Nutritionist Agent**  
    - Type: Langchain Agent node  
    - Role: AI nutritionist specializing in dietary advice and nutrient recommendations in diary format  
    - Configuration:  
      - Input text: Output from Menu Agent (structured menu data)  
      - System message instructs creation of diary-style dietary advice  
    - Input: Menu Agent output  
    - Output: Nutritional diary text  
    - Edge Cases: Incomplete data, irrelevant advice, API errors  
    - Version: 3

  - **Append row in sheet1**  
    - Type: Google Sheets node (append operation)  
    - Role: Inserts nutritional diary entry into the "diary" sheet  
    - Configuration:  
      - Document ID: Same Google Sheet  
      - Sheet Name: "diary"  
      - Columns: Diary (nutritional advice text)  
    - Input: Nutritionist Agent output  
    - Edge Cases: Credentials, sheet access, quota issues  
    - Version: 4.7

---

#### 1.5 Social Media Post Creation & Sending

- **Overview:**  
Transforms the nutritional diary entry into an engaging social media post (specifically for X, formerly Twitter) and sends it as a direct message to a specified user.

- **Nodes Involved:**  
  - OpenRouter Chat Model1 (AI model for Post Agent)  
  - Post Agent (Langchain agent)  
  - Create Direct Message (Twitter node)

- **Node Details:**

  - **OpenRouter Chat Model1**  
    - Type: Language model node  
    - Role: Provides AI model for crafting social media posts  
    - Configuration: Default options  
    - Version: 1

  - **Post Agent**  
    - Type: Langchain Agent node  
    - Role: AI agent that converts the nutritional diary into a compelling Twitter/X post  
    - Configuration:  
      - Input text: Diary content from Nutritionist Agent (field "Diary")  
      - System message instructs to create engaging social media content  
    - Input: Nutritionist Agent output  
    - Output: Social media post text  
    - Edge Cases: Post length limits, inappropriate content, API failures  
    - Version: 3

  - **Create Direct Message**  
    - Type: Twitter node (n8n-nodes-base.twitter)  
    - Role: Sends the social media post as a direct message to a specified user  
    - Configuration:  
      - Resource: directMessage  
      - Text: AI-generated social media post  
      - User: Twitter username configured by user (recipient of the DM)  
    - Input: Post Agent output  
    - Edge Cases: Twitter API rate limits, invalid credentials, DM permissions  
    - Version: 2  
    - Sticky Note: Twitter credential setup and user configuration details.

---

### 3. Summary Table

| Node Name              | Node Type                                 | Functional Role                           | Input Node(s)      | Output Node(s)             | Sticky Note                                                                                                                                                                                                                                             |
|------------------------|------------------------------------------|------------------------------------------|--------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                | n8n-nodes-base.webhook                    | Receives menu input via HTTP POST        | -                  | Menu Agent                 | The workflow starts with a Webhook. Copy the webhook URL from the "Webhook" node. You will send your menu item input to this URL.                                                                                                                     |
| OpenRouter Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides AI model for Menu Agent         | -                  | Menu Agent (ai_languageModel) |                                                                                                                                                                                                                                                         |
| Structured Output Parser| @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI response into structured JSON  | Menu Agent         | Menu Agent                 |                                                                                                                                                                                                                                                         |
| Menu Agent             | @n8n/n8n-nodes-langchain.agent           | Extracts ingredient and cost info        | Webhook, OpenRouter Chat Model, Structured Output Parser | Append row in sheet       |                                                                                                                                                                                                                                                         |
| Append row in sheet    | n8n-nodes-base.googleSheets               | Appends ingredient/cost data to Recipe sheet | Menu Agent          | Leftovers Agent            | Google Sheets integration instructions: must set up credentials, create sheets "Recipe", "leftovers", and "diary" with specified columns. Replace Document ID and ensure correct sheet names.                                                         |
| OpenRouter Chat Model3 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides AI model for Leftovers Agent    | -                  | Leftovers Agent (ai_languageModel) |                                                                                                                                                                                                                                                         |
| Leftovers Agent        | @n8n/n8n-nodes-langchain.agent           | Suggests recipes for leftover ingredients | Append row in sheet | Append row in sheet2       |                                                                                                                                                                                                                                                         |
| Append row in sheet2   | n8n-nodes-base.googleSheets               | Appends leftover recipes to leftovers sheet | Leftovers Agent      | Nutritionist Agent         |                                                                                                                                                                                                                                                         |
| OpenRouter Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides AI model for Nutritionist Agent | -                  | Nutritionist Agent (ai_languageModel) |                                                                                                                                                                                                                                                         |
| Nutritionist Agent     | @n8n/n8n-nodes-langchain.agent           | Creates nutritional diary entry          | Append row in sheet2 | Append row in sheet1       |                                                                                                                                                                                                                                                         |
| Append row in sheet1   | n8n-nodes-base.googleSheets               | Appends diary entry to diary sheet       | Nutritionist Agent  | Post Agent                 |                                                                                                                                                                                                                                                         |
| OpenRouter Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides AI model for Post Agent          | -                  | Post Agent (ai_languageModel) |                                                                                                                                                                                                                                                         |
| Post Agent             | @n8n/n8n-nodes-langchain.agent           | Creates social media post from diary     | Append row in sheet1 | Create Direct Message      |                                                                                                                                                                                                                                                         |
| Create Direct Message  | n8n-nodes-base.twitter                    | Sends social media post as Twitter DM    | Post Agent          | -                          | Twitter Integration: Set up Twitter credentials. Specify the user (username) for the direct message recipient.                                                                                                                                           |
| Sticky Note            | n8n-nodes-base.stickyNote                 | Documentation                            | -                  | -                          | Explains overall workflow functions: cost tracking, leftovers, nutrition diary, social media posting; prerequisites include n8n, Google Sheets, OpenRouter API key, Twitter developer account.                                                         |
| Sticky Note1           | n8n-nodes-base.stickyNote                 | Documentation for Google Sheets setup    | -                  | -                          | Details Google Sheets setup: document with sheets "Recipe", "leftovers", "diary"; required columns and mapping instructions; replace Document ID and sheet names in append nodes.                                                                       |
| Sticky Note2           | n8n-nodes-base.stickyNote                 | Documentation for webhook setup           | -                  | -                          | Explains webhook trigger usage and URL copying.                                                                                                                                                                                                         |
| Sticky Note4           | n8n-nodes-base.stickyNote                 | Documentation for Twitter integration     | -                  | -                          | Explains Twitter credential setup and user configuration for DM sending.                                                                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: e.g., "your-webhook-path" (customizable)  
   - Purpose: Entry point for new menu data.  

2. **Add OpenRouter Chat Model Node (Menu Agent AI model)**  
   - Type: Langchain OpenRouter Chat Model  
   - Default parameters  
   - Purpose: Provide AI capabilities for menu parsing.  

3. **Add Structured Output Parser Node**  
   - Type: Langchain Structured Output Parser  
   - Configure JSON schema example to parse ingredients and costs, with fields: Date, Item, Ingredients, Ingredient Cost, Unit Price, Quantity, Total, Cost, Leftover Ingredients.  
   - Purpose: Convert AI text into structured JSON.  

4. **Add Menu Agent Node**  
   - Type: Langchain Agent  
   - Input Text: `={{ $json.body }}` (webhook payload)  
   - System Message:  
     "You are a housekeeper. From the searched content, output the necessary ingredients and costs. You do not need to show the cost per use for seasonings. Costs should be shown as both the total amount for one shopping trip and the unit price. Include units for ingredients and prices."  
   - Enable output parser, link to Structured Output Parser.  
   - Connect AI model input to OpenRouter Chat Model node (step 2).  

5. **Add Google Sheets Node to Append Recipe Data (Append row in sheet)**  
   - Type: Google Sheets (append operation)  
   - Document ID: Your Google Sheet ID (e.g., "Recipe List")  
   - Sheet Name: "Recipe" (レシピ)  
   - Map columns from Menu Agent parsed output fields accordingly (Date, Item, Ingredients, Ingredient Cost, Unit Price, Quantity, Total, Cost, Leftover Ingredients)  
   - Cell format: USER_ENTERED  
   - Connect input from Menu Agent output.  

6. **Add OpenRouter Chat Model Node (Leftovers Agent AI model)**  
   - Type: Langchain OpenRouter Chat Model  
   - Default parameters  

7. **Add Leftovers Agent Node**  
   - Type: Langchain Agent  
   - Input Text: `={{ $('Menu Agent').item.json.output }}` (output from Menu Agent)  
   - System Message:  
     "You are a specialist in utilizing unused ingredients effectively. List leftover ingredients. Suggest three recipes that can be cooked with the remaining ingredients. You do not need to describe the cooking method."  
   - Connect AI model input to OpenRouter Chat Model (step 6).  

8. **Add Google Sheets Node to Append Leftover Recipes (Append row in sheet2)**  
   - Type: Google Sheets (append operation)  
   - Document ID: Same Google Sheet ID as above  
   - Sheet Name: "leftovers"  
   - Map columns: Date (from Menu Agent output), Ingredients (from Leftovers Agent output)  
   - Connect input from Leftovers Agent output.  

9. **Add OpenRouter Chat Model Node (Nutritionist Agent AI model)**  
   - Type: Langchain OpenRouter Chat Model  
   - Default parameters  

10. **Add Nutritionist Agent Node**  
    - Type: Langchain Agent  
    - Input Text: `={{ $('Menu Agent').item.json.output }}`  
    - System Message:  
      "You are a nutritionist specialist. Create a diary-style entry with dietary advice and what nutrients should be consumed."  
    - Connect AI model input to OpenRouter Chat Model (step 9).  

11. **Add Google Sheets Node to Append Nutrition Diary (Append row in sheet1)**  
    - Type: Google Sheets (append operation)  
    - Document ID: Same Google Sheet ID  
    - Sheet Name: "diary"  
    - Map column: Diary (from Nutritionist Agent output)  
    - Connect input from Nutritionist Agent output.  

12. **Add OpenRouter Chat Model Node (Post Agent AI model)**  
    - Type: Langchain OpenRouter Chat Model  
    - Default parameters  

13. **Add Post Agent Node**  
    - Type: Langchain Agent  
    - Input Text: `={{ $json['Diary'] }}` (diary text from Nutritionist Agent)  
    - System Message:  
      "You are an X (Twitter) poster. Create a compelling post to attract viewers."  
    - Connect AI model input to OpenRouter Chat Model (step 12).  

14. **Add Twitter Node to Create Direct Message**  
    - Type: Twitter (Direct Message)  
    - Text: `={{ $json.output }}` (from Post Agent)  
    - User: Specify the recipient username (your Twitter handle or test user)  
    - Configure Twitter OAuth2 credentials appropriately.  
    - Connect input from Post Agent output.  

15. **Connect Workflow Nodes in Sequence:**  
    - Webhook → Menu Agent (with OpenRouter Chat Model and Structured Output Parser) → Append row in sheet (Recipe) → Leftovers Agent (with OpenRouter Chat Model3) → Append row in sheet2 (leftovers) → Nutritionist Agent (with OpenRouter Chat Model2) → Append row in sheet1 (diary) → Post Agent (with OpenRouter Chat Model1) → Create Direct Message (Twitter)  

16. **Set up Credentials:**  
    - Google Sheets credential with access to the specified Google Sheet document  
    - OpenRouter API key credential for all Langchain AI model nodes  
    - Twitter OAuth2 credential with permission to send Direct Messages  

17. **Create Google Sheet Document:**  
    - Create a Google Sheet with three sheets:  
      - "Recipe" with columns: Date, Item, Ingredients, Ingredient Cost, Unit Price, Quantity, Total, Cost, Leftover Ingredients  
      - "leftovers" with columns: Date, Ingredients  
      - "diary" with column: Diary  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This advanced workflow acts as a comprehensive culinary assistant performing cost tracking, leftover recipe suggestions, nutritional diary entries, and social media posting. Prerequisites include n8n, Google Sheets, OpenRouter API, Twitter developer access. | Sticky Note at workflow start (overall explanation)                                             |
| Google Sheets setup requires a single document with three sheets named "Recipe", "leftovers", and "diary" with specified columns. Replace document IDs in append nodes accordingly.                                                      | Sticky Note explaining Google Sheets integration                                                |
| The workflow triggers via a Webhook; users must send menu input JSON to this webhook URL to start the process.                                                                                                                         | Sticky Note about webhook trigger                                                               |
| Twitter integration requires setting up Twitter OAuth2 credentials in n8n and specifying the user to receive direct messages.                                                                                                         | Sticky Note explaining Twitter integration                                                      |