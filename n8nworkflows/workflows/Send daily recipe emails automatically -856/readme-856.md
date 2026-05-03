Send daily recipe emails automatically 

https://n8nworkflows.xyz/workflows/send-daily-recipe-emails-automatically--856


# Send daily recipe emails automatically 

### 1. Workflow Overview

This workflow, titled **“Send daily recipe emails automatically”**, is designed to provide users with daily recipe suggestions tailored to their dietary preferences and constraints. It automatically fetches recipes from the Edamam Recipe Search API based on user-defined criteria, formats those recipes into an email, and sends it out once daily at a scheduled time.

The workflow is structured into the following logical blocks:

- **1.1 Scheduling Trigger**: Runs the workflow automatically every day at a preset hour.
- **1.2 User Criteria Definition & Adjustment**: Defines user preferences and dynamically adjusts certain parameters like diet and health filters if set to random.
- **1.3 Recipe Data Retrieval**: Queries the Edamam API twice — first to get the total recipe count available, then to retrieve detailed recipe data for a subset.
- **1.4 Recipe Selection & Formatting**: Selects a set of recipes based on counts, formats the recipe details into an HTML email body.
- **1.5 Email Dispatch**: Sends the formatted recipe email to the user’s configured email address.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling Trigger

- **Overview:**  
  Initiates the workflow daily at a fixed hour (10:00 AM) to automate the recipe email distribution without manual intervention.

- **Nodes Involved:**  
  - Cron

- **Node Details:**  
  - **Cron**  
    - Type: Trigger node that fires on a schedule.  
    - Configuration: Set to trigger once daily at hour 10 (10:00 AM local time).  
    - Input: None (trigger node).  
    - Output: Triggers downstream nodes once per day.  
    - Edge Cases: If n8n is offline at trigger time, the workflow will miss that day’s run. No retry mechanism in the node itself.  
    - Version: Compatible with all current n8n versions.  

---

#### 2.2 User Criteria Definition & Adjustment

- **Overview:**  
  Collects and sets the user-defined criteria for recipe searching including number of recipes, calorie limits, diet, health filters, and the search term. Also dynamically replaces random diet or health options with actual random selections from predefined lists.

- **Nodes Involved:**  
  - Search Criteria  
  - Set Query Values

- **Node Details:**  
  - **Search Criteria**  
    - Type: Set node used to define static or default parameters for the search.  
    - Configuration:  
      - RecipeCount: 3 (default number of recipes to retrieve)  
      - IngredientCount: 5 (maximum ingredients per recipe)  
      - CaloriesMin: blank (can be customized)  
      - CaloriesMax: 1500  
      - TimeMin: blank  
      - TimeMax: 30 (max preparation time in minutes)  
      - Diet: "balanced" (default diet option)  
      - Health: "random" (will be replaced dynamically)  
      - SearchItem: "chicken" (default search term)  
      - AppID and AppKey: placeholders for Edamam API credentials  
    - Input: Trigger data (empty payload) from Cron node.  
    - Output: JSON object with user criteria.  
    - Edge Cases: Missing or invalid API credentials will cause downstream API calls to fail.  
    - Version: No specific requirements.  

  - **Set Query Values**  
    - Type: Function node that computes additional search parameters.  
    - Configuration Logic:  
      - Combines CaloriesMin and CaloriesMax into a calorie range string (e.g., "0-1500").  
      - Combines TimeMin and TimeMax into a time range string.  
      - If Diet is “random”, selects randomly from a fixed diet list.  
      - If Health is “random”, selects randomly from a fixed health option list.  
    - Key expressions: uses JavaScript to modify the incoming JSON data with new values.  
    - Input: Output from Search Criteria node.  
    - Output: Modified JSON with updated Diet, Health, calories, and time fields.  
    - Edge Cases: If CaloriesMin or TimeMin is blank, resulting range strings might be incomplete or malformed, potentially causing API query issues. Random selection is uniform but without seed (non-repeatable randomness).  
    - Version: Compatible with standard n8n function node environment.  

---

#### 2.3 Recipe Data Retrieval

- **Overview:**  
  Queries the Edamam API twice: first to get the total number of recipes matching criteria, then to fetch a subset of recipes based on randomized pagination.

- **Nodes Involved:**  
  - Retrieve Recipe Counts  
  - Set Counts  
  - Set Recipe ID Values  
  - Retrieve Recipes

- **Node Details:**  
  - **Retrieve Recipe Counts**  
    - Type: HTTP Request node querying Edamam API.  
    - Configuration:  
      - URL: https://api.edamam.com/search  
      - Query parameters:  
        - q: search term (from Set Query Values)  
        - app_id, app_key: Edamam credentials  
        - ingr: max ingredient count  
        - diet, calories, time: filtered as per user criteria  
        - from=1, to=2 (only fetching first two results to get metadata including count)  
    - Input: Output from Set Query Values node.  
    - Output: JSON response including `count` (total recipes found).  
    - Edge Cases: API rate limits, invalid credentials, network errors, or malformed query parameters can cause failures.  
    - Version: Requires HTTPS support and valid Edamam API keys.  

  - **Set Counts**  
    - Type: Set node to store recipe counts for use in the next steps.  
    - Configuration:  
      - RecipeCount: total count from Retrieve Recipe Counts response.  
      - ReturnCount: user-defined number of recipes to return (from Search Criteria).  
    - Input: Output from Retrieve Recipe Counts.  
    - Output: JSON object with updated counts.  
    - Edge Cases: If Retrieve Recipe Counts fails or returns zero count, logic downstream may fail or produce empty emails.  

  - **Set Recipe ID Values**  
    - Type: Function node that sets random starting point for recipe retrieval (`from`) and calculates `to` value based on requested number of recipes.  
    - Configuration Logic:  
      - `from` is a random integer between 1 and user-requested RecipeCount.  
      - `to` is `from` + ReturnCount, to define the window of recipes to fetch.  
    - Input: Output from Set Counts node.  
    - Output: JSON updated with pagination indices.  
    - Edge Cases: If `from` + `ReturnCount` exceeds the total number of available recipes, API might return fewer results than requested. Randomized `from` can sometimes lead to empty responses if the range is invalid.  

  - **Retrieve Recipes**  
    - Type: HTTP Request node that fetches detailed recipes from Edamam based on pagination.  
    - Configuration:  
      - URL and query parameters similar to Retrieve Recipe Counts, but with dynamic `from` and `to` values from Set Recipe ID Values.  
      - Uses search criteria and filters from earlier nodes.  
    - Input: Output from Set Recipe ID Values.  
    - Output: Detailed recipe data (hits array).  
    - Edge Cases: Similar to Retrieve Recipe Counts, plus possible empty results if pagination is invalid or no recipes match criteria.  

---

#### 2.4 Recipe Selection & Formatting

- **Overview:**  
  Converts the retrieved recipe data into an HTML formatted email body listing clickable recipe links.

- **Nodes Involved:**  
  - Create Email Body in HTML

- **Node Details:**  
  - **Create Email Body in HTML**  
    - Type: Function node that processes the retrieved recipes.  
    - Configuration Logic:  
      - Extracts the `hits` array from the Edamam API response.  
      - Iterates over recipes to create an unordered list of clickable links with recipe titles.  
      - Sets the generated HTML string as `emailBody` property.  
    - Input: Output from Retrieve Recipes node.  
    - Output: JSON object with `emailBody` as HTML string.  
    - Edge Cases: If `hits` is empty or missing, the email body may be empty or malformed. Robustness depends on API response integrity.  

---

#### 2.5 Email Dispatch

- **Overview:**  
  Sends the generated recipe email to the user via configured SMTP credentials.

- **Nodes Involved:**  
  - Send Recipes

- **Node Details:**  
  - **Send Recipes**  
    - Type: Email Send node.  
    - Configuration:  
      - Subject line dynamically composed using recipe count, diet, health, calories max, and time max values for clarity.  
      - HTML body set from Create Email Body in HTML node’s output.  
      - To and From email addresses must be configured by the user.  
      - SMTP credentials selected from stored Gmail credentials.  
    - Input: Output from Create Email Body in HTML node.  
    - Output: Email send status.  
    - Edge Cases: SMTP authentication failures, invalid email addresses, or network issues can cause email sending to fail. Proper error handling recommended.  
    - Version: Requires SMTP credential setup in n8n, compatible with Gmail OAuth2 or SMTP.  

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                          | Input Node(s)             | Output Node(s)            | Sticky Note                                    |
|------------------------|-----------------------|----------------------------------------|---------------------------|---------------------------|------------------------------------------------|
| Cron                   | Cron Trigger          | Initiates workflow daily at 10:00 AM   | —                         | Search Criteria           |                                                |
| Search Criteria        | Set                   | Defines user search parameters          | Cron                      | Set Query Values          | See workflow description for customizable criteria. |
| Set Query Values       | Function              | Adjusts ranges and randomizes options   | Search Criteria            | Retrieve Recipe Counts    |                                                |
| Retrieve Recipe Counts | HTTP Request          | Queries Edamam for total recipe count   | Set Query Values           | Set Counts                | Requires valid Edamam API credentials.          |
| Set Counts             | Set                   | Sets total and return recipe counts     | Retrieve Recipe Counts     | Set Recipe ID Values      |                                                |
| Set Recipe ID Values   | Function              | Chooses random recipe pagination window | Set Counts                 | Retrieve Recipes          |                                                |
| Retrieve Recipes       | HTTP Request          | Fetches detailed recipes from Edamam    | Set Recipe ID Values       | Create Email Body in HTML |                                                |
| Create Email Body in HTML | Function           | Formats recipes into HTML email body    | Retrieve Recipes           | Send Recipes              |                                                |
| Send Recipes           | Email Send            | Sends recipe email via SMTP              | Create Email Body in HTML  | —                         | Configure SMTP credentials and email addresses.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Cron” node:**  
   - Type: Cron  
   - Set trigger to run once daily at hour 10 (10:00 AM).  
   - No credentials needed.

2. **Create “Search Criteria” node:**  
   - Type: Set  
   - Add fields:  
     - RecipeCount (number): default 3  
     - IngredientCount (number): default 5  
     - CaloriesMin (number): leave blank or set as desired  
     - CaloriesMax (number): default 1500  
     - TimeMin (number): leave blank or set as desired  
     - TimeMax (number): default 30  
     - Diet (string): default "balanced"  
     - Health (string): default "random"  
     - SearchItem (string): default "chicken"  
     - AppID (string): Enter your Edamam AppID  
     - AppKey (string): Enter your Edamam AppKey  
   - Connect “Cron” node output to this node.

3. **Create “Set Query Values” node:**  
   - Type: Function  
   - Paste the JavaScript code to:  
     - Combine CaloriesMin and CaloriesMax into a calorie range string.  
     - Combine TimeMin and TimeMax into a time range string.  
     - Randomly select Diet if set to “random”.  
     - Randomly select Health if set to “random”.  
   - Connect “Search Criteria” node output here.

4. **Create “Retrieve Recipe Counts” node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://api.edamam.com/search  
   - Query Parameters:  
     - q: `={{$node["Set Query Values"].json["SearchItem"]}}`  
     - app_id: `={{$node["Set Query Values"].json["AppID"]}}`  
     - app_key: `={{$node["Set Query Values"].json["AppKey"]}}`  
     - ingr: `={{$node["Set Query Values"].json["IngredientCount"]}}`  
     - diet: `={{$node["Set Query Values"].json["Diet"]}}`  
     - calories: `={{$node["Set Query Values"].json["calories"]}}`  
     - time: `={{$node["Set Query Values"].json["time"]}}`  
     - from: 1  
     - to: 2  
   - Connect “Set Query Values” node output here.

5. **Create “Set Counts” node:**  
   - Type: Set  
   - Add fields:  
     - RecipeCount (number): `={{$node["Retrieve Recipe Counts"].json["count"]}}`  
     - ReturnCount (number): `={{$node["Search Criteria"].json["RecipeCount"]}}`  
   - Enable “Keep Only Set” option to discard other data.  
   - Connect “Retrieve Recipe Counts” node output here.

6. **Create “Set Recipe ID Values” node:**  
   - Type: Function  
   - Paste JavaScript code that:  
     - Randomly selects `from` between 1 and RecipeCount  
     - Calculates `to` as `from + ReturnCount`  
   - Connect “Set Counts” node output here.

7. **Create “Retrieve Recipes” node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://api.edamam.com/search  
   - Query Parameters:  
     - q: `={{$node["Search Criteria"].json["SearchItem"]}}`  
     - app_id: `={{$node["Search Criteria"].json["AppID"]}}`  
     - app_key: `={{$node["Search Criteria"].json["AppKey"]}}`  
     - from: `={{$node["Set Recipe ID Values"].json["from"]}}`  
     - to: `={{$node["Set Recipe ID Values"].json["to"]}}`  
     - ingr: `={{$node["Search Criteria"].json["IngredientCount"]}}`  
     - diet: `={{$node["Search Criteria"].json["Diet"]}}`  
     - calories: `={{$node["Set Query Values"].json["calories"]}}`  
     - time: `={{$node["Set Query Values"].json["time"]}}`  
   - Connect “Set Recipe ID Values” node output here.

8. **Create “Create Email Body in HTML” node:**  
   - Type: Function  
   - Paste JavaScript code that:  
     - Extracts recipes from `hits` array.  
     - Builds an HTML list of recipe links using `recipe.label` and `recipe.shareAs`.  
     - Sets `emailBody` field on output JSON.  
   - Connect “Retrieve Recipes” node output here.

9. **Create “Send Recipes” node:**  
   - Type: Email Send  
   - Configure SMTP credentials (e.g., Gmail OAuth2 or SMTP).  
   - Set:  
     - To Email: Your target email address.  
     - From Email: Your sending email address.  
     - Subject: Use expression to compose with RecipeCount, Diet, Health, CaloriesMax, TimeMax values from “Set Query Values” node.  
     - HTML: Use expression to set from `emailBody` field from “Create Email Body in HTML” node.  
   - Connect “Create Email Body in HTML” node output here.

10. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow requires an Edamam Recipe Search API key. Sign up at https://developer.edamam.com/edamam-recipe-api to obtain your AppID and AppKey.                                                                             | Edamam API Documentation                         |
| Configure SMTP credentials in n8n for email delivery, e.g., Gmail SMTP with OAuth2 or app password.                                                                                                                            | n8n Email Credentials Setup Guide                 |
| Customize your recipe preferences in the "Search Criteria" node to tailor diet, health filters, calorie range, and ingredient counts.                                                                                        | Workflow customization instructions (above)      |
| The “random” options for Diet and Health enable daily variety but may occasionally lead to zero results depending on filters used; consider adjusting constraints if emails come empty.                                        | Workflow behavior note                            |
| This workflow sends emails once daily at 10 AM; adjust the "Cron" node to change frequency or time.                                                                                                                            | Scheduling adjustment                             |

---

This structured documentation should enable advanced users and automation agents to fully understand, reproduce, and customize the “Send daily recipe emails automatically” workflow effectively.