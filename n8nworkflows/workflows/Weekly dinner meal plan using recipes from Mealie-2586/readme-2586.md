Weekly dinner meal plan using recipes from Mealie

https://n8nworkflows.xyz/workflows/weekly-dinner-meal-plan-using-recipes-from-mealie-2586


# Weekly dinner meal plan using recipes from Mealie

### 1. Workflow Overview

This workflow automates the creation of a weekly dinner meal plan by selecting random recipes from a Mealie recipe management system. It is designed to run on a scheduled basis (by default, every Friday at 8 PM) and generates a meal plan starting a configurable number of days into the future. The workflow supports optional filtering by recipe category from Mealie.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Configuration Setup:** Schedule trigger or manual trigger initiates the workflow, setting configuration parameters such as number of recipes, offset days, category, and Mealie instance URL.

- **1.2 Recipe Retrieval:** Fetches the list of available recipes from the Mealie API, optionally filtered by category.

- **1.3 Random Recipe Selection:** Executes custom logic to randomly select a specified number of unique recipes from the retrieved list and assigns each recipe to a specific date for the meal plan.

- **1.4 Meal Plan Creation:** Sends the constructed meal plan back to Mealie via API to create the meal plan entries.

- **1.5 Documentation & Setup Notes:** Sticky notes provide user guidance and important setup instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Configuration Setup

- **Overview:**  
  Initiates the workflow either on schedule (weekly Friday 8 PM) or manually for testing. Sets up configuration parameters that control the workflow behavior.

- **Nodes Involved:**  
  - Friday 8pm (Schedule Trigger)  
  - When clicking "Test workflow" (Manual Trigger)  
  - Config (Set Node)

- **Node Details:**

  - **Friday 8pm**  
    - Type: Schedule Trigger  
    - Configuration: Runs weekly on Fridays at 8 PM (20:00)  
    - Inputs: None (trigger)  
    - Outputs: Connects to `Config` node  
    - Edge Cases: If the system time zone is misconfigured, the trigger time may be off; ensure server time zone matches expectations.

  - **When clicking "Test workflow"**  
    - Type: Manual Trigger  
    - Configuration: Triggered manually by user via UI for testing purposes  
    - Inputs: None  
    - Outputs: Connects to `Config` node  
    - Edge Cases: Manual trigger bypasses schedule; useful for debugging and quick tests.

  - **Config**  
    - Type: Set  
    - Configuration: Defines key parameters:  
      - `numberOfRecipes` (number): How many recipes to select for the plan (default 5)  
      - `offsetPlanDays` (number): Days from today to start the meal plan (default 3)  
      - `mealieCategoryId` (string): Category ID to filter recipes (default set to a specific UUID; can be blank for all recipes)  
      - `mealieBaseUrl` (string): Base URL of the Mealie instance (default local IP with port)  
    - Inputs: Receives trigger input  
    - Outputs: Passes config to `Get Recipes` node  
    - Edge Cases: Invalid or missing category ID may cause empty recipe list. Incorrect base URL or unreachable Mealie API will cause HTTP errors.

---

#### 1.2 Recipe Retrieval

- **Overview:**  
  Retrieves recipes from the Mealie API based on the configured category and base URL.

- **Nodes Involved:**  
  - Get Recipes (HTTP Request)

- **Node Details:**

  - **Get Recipes**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET  
      - URL: `{{ $json.mealieBaseUrl }}/api/recipes`  
      - Query Parameters:  
        - `perPage=100` (limits results to 100 recipes)  
        - `categories={{ $json.mealieCategoryId }}` (filters by category if set)  
      - Authentication: Uses a custom HTTP header credential containing Mealie API token  
      - Response format: JSON  
    - Inputs: Receives config JSON with base URL and category ID  
    - Outputs: Passes received recipe list JSON to `Generate Random Items`  
    - Edge Cases:  
      - If category ID is invalid or empty, API may return all recipes or none.  
      - API errors like 401 Unauthorized if token invalid or missing.  
      - Timeout or network errors if Mealie server unreachable.  
      - Pagination limited to 100 recipes; if more exist, they won't be retrieved.

---

#### 1.3 Random Recipe Selection

- **Overview:**  
  Custom JavaScript logic randomly selects unique recipes from the retrieved list and assigns each a date starting from offset days in the future, producing an array of meal plan entries.

- **Nodes Involved:**  
  - Generate Random Items (Code)

- **Node Details:**

  - **Generate Random Items**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Reads config values: `numberOfRecipes`, `offsetPlanDays`  
      - Reads recipes from input JSON `items` array  
      - Randomly selects recipes without duplicates until reaching desired number  
      - Assigns each selected recipe a date: current date + offset + index  
      - Outputs array of objects with:  
        - `date` (ISO string YYYY-MM-DD)  
        - `entryType` (fixed string "dinner")  
        - `recipeId` and `name` from selected recipes  
    - Inputs: Receives recipes list from `Get Recipes` node  
    - Outputs: Passes meal plan array to `Create Meal Plan` node  
    - Edge Cases:  
      - If requested `numberOfRecipes` exceeds available recipes, the loop may run indefinitely or until all are selected; should be handled gracefully.  
      - If recipe list is empty, results will be empty and subsequent API calls may fail.  
      - Date calculations assume server time zone; discrepancies can occur if server time zone differs from Mealie.

---

#### 1.4 Meal Plan Creation

- **Overview:**  
  Sends the constructed meal plan to the Mealie API to create the scheduled dinner entries.

- **Nodes Involved:**  
  - Create Meal Plan (HTTP Request)

- **Node Details:**

  - **Create Meal Plan**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `{{ $('Config').first().json.mealieBaseUrl }}/api/households/mealplans`  
      - Body: JSON array of meal plan entries generated by the previous node  
      - Authentication: Same HTTP header credential for Mealie API token  
      - Expects JSON response  
    - Inputs: Receives meal plan array from `Generate Random Items`  
    - Outputs: None configured (terminal node)  
    - Edge Cases:  
      - API errors if payload malformed or token invalid  
      - If Mealie API changes, may require updates to URL or payload format  
      - Network errors or timeouts  
      - Duplicate or overlapping meal plans if workflow runs multiple times without cleanup

---

#### 1.5 Documentation & Setup Notes

- **Overview:**  
  Provides inline documentation and instructions for users to configure and understand the workflow.

- **Nodes Involved:**  
  - Sticky Note (multiple)

- **Node Details:**

  - **Sticky Note (Trigger)**  
    - Content: "Set the trigger to run when you like"  
    - Role: Reminder for scheduling.

  - **Sticky Note1 (Config Instructions)**  
    - Content: Instructions to update base URL, number of recipes, offset days, and category ID.

  - **Sticky Note2 (Setup Instructions)**  
    - Content: Guidance on setting up Mealie API credentials, applying credentials to HTTP nodes, and configuring schedule and config.

  - **Sticky Note3 (Workflow Logic Summary)**  
    - Content: Summary of workflow steps: get recipes, randomly pick, create meal plans.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                       |
|-------------------------|---------------------|----------------------------------------|---------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| Friday 8pm              | Schedule Trigger    | Initiates workflow weekly on Friday 8 PM | None                      | Config                  | Set the trigger to run when you like                                                            |
| When clicking "Test workflow" | Manual Trigger     | Manual start for testing workflow       | None                      | Config                  |                                                                                                 |
| Config                  | Set                 | Sets workflow configuration parameters | Friday 8pm, Manual Trigger | Get Recipes             | Update this Config: base URL, number of recipes, offset days, category ID                        |
| Get Recipes             | HTTP Request        | Retrieves recipes from Mealie API       | Config                    | Generate Random Items    | Set up a credential for your Mealie API token; apply it to HTTP Request nodes                   |
| Generate Random Items    | Code                 | Selects random unique recipes & assigns dates | Get Recipes               | Create Meal Plan         | Workflow logic: get recipes, randomly pick, create meal plans                                   |
| Create Meal Plan        | HTTP Request        | Sends meal plan to Mealie API           | Generate Random Items      | None                    | Set up a credential for your Mealie API token; apply it to HTTP Request nodes                   |
| Sticky Note             | Sticky Note          | Documentation and setup notes            | None                      | None                    | See details in section 2.5                                                                       |
| Sticky Note1            | Sticky Note          | Documentation and setup notes            | None                      | None                    | See details in section 2.5                                                                       |
| Sticky Note2            | Sticky Note          | Documentation and setup notes            | None                      | None                    | See details in section 2.5                                                                       |
| Sticky Note3            | Sticky Note          | Documentation and setup notes            | None                      | None                    | See details in section 2.5                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **Schedule Trigger** node named `Friday 8pm`:
     - Set to run every week on Friday (`triggerAtDay`=5) at 20:00 (8 PM).
   
   - Add a **Manual Trigger** node named `When clicking "Test workflow"` for manual runs.

2. **Create Config Node:**

   - Add a **Set** node named `Config`.
   - Add the following fields with default values:
     - `numberOfRecipes` (Number): 5
     - `offsetPlanDays` (Number): 3
     - `mealieCategoryId` (String): `<category UUID or empty for all>`
     - `mealieBaseUrl` (String): e.g. `http://192.168.1.5:9925`
   - Connect both trigger nodes (`Friday 8pm` and `When clicking "Test workflow"`) to this node's input.

3. **Create HTTP Request to Get Recipes:**

   - Add an **HTTP Request** node named `Get Recipes`.
   - Set method to GET.
   - URL: `{{ $json.mealieBaseUrl }}/api/recipes`
   - Add query parameters:
     - `perPage`: 100
     - `categories`: `{{ $json.mealieCategoryId }}`
   - Set authentication to HTTP Header Auth with your Mealie API token credential.
   - Connect `Config` node to this node.

4. **Create Code Node for Random Selection:**

   - Add a **Code** node named `Generate Random Items`.
   - Paste the following JavaScript code:

     ```javascript
     const numberOfRecipes = $('Config').first().json.numberOfRecipes;
     const offsetPlanDays = $('Config').first().json.offsetPlanDays;
     const items = $input.first().json.items;

     let planFirstDate = new Date();
     planFirstDate.setDate(planFirstDate.getDate() + offsetPlanDays);

     const recipeList = [];
     const randomNums = [];
     let currentItem = 0;

     while (recipeList.length < numberOfRecipes) {
       const randomNum = Math.floor(Math.random() * Math.floor(items.length));

       if (!randomNums.includes(randomNum)) {
         const thisRecipe = items[randomNum];

         const newDate = new Date(planFirstDate);
         newDate.setDate(planFirstDate.getDate() + currentItem);

         const planDate = [
           newDate.getFullYear(),
           ('0' + (newDate.getMonth() + 1)).slice(-2),
           ('0' + newDate.getDate()).slice(-2)
         ].join('-');

         const planDay = {
           "date": planDate,
           "entryType": "dinner",
           "recipeId": thisRecipe.id,
           "name": thisRecipe.name
         };

         currentItem++;
         recipeList.push(planDay);
         randomNums.push(randomNum);
       }
     }

     return recipeList;
     ```
   - Connect `Get Recipes` node to this node.

5. **Create HTTP Request to Post Meal Plan:**

   - Add an **HTTP Request** node named `Create Meal Plan`.
   - Set method to POST.
   - URL: `{{ $('Config').first().json.mealieBaseUrl }}/api/households/mealplans`
   - Body content type: JSON
   - Add to the request body the JSON output from `Generate Random Items`.
   - Use the same HTTP Header Auth credential as before.
   - Connect `Generate Random Items` node to this node.

6. **Add Sticky Notes (Optional):**

   - Add sticky notes with the setup instructions and workflow overview for documentation.

7. **Credential Setup:**

   - Create an HTTP Header Authentication credential containing your Mealie API token.
   - Assign this credential to both HTTP Request nodes (`Get Recipes` and `Create Meal Plan`).

8. **Final Connections Check:**

   - Ensure both triggers connect to `Config`.
   - `Config` connects to `Get Recipes`.
   - `Get Recipes` connects to `Generate Random Items`.
   - `Generate Random Items` connects to `Create Meal Plan`.

9. **Testing:**

   - Run the workflow manually with the manual trigger or wait for the schedule.
   - Confirm that recipes are fetched, random selection works, and meal plan is created in Mealie.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Setup requires a valid Mealie API token and base URL of your Mealie instance.                    | Mealie documentation or your local instance setup                                                 |
| Schedule trigger can be adjusted to any desired frequency/time to automate meal plan generation. | n8n scheduling rules documentation                                                                |
| The workflow handles up to 100 recipes per API call; for larger recipe sets, consider pagination. | Mealie API documentation on pagination                                                            |
| For troubleshooting, use manual trigger to test workflow runs immediately.                       | n8n UI manual trigger functionality                                                               |
| The code node ensures no duplicate recipes in the generated plan.                               | Custom JavaScript logic within `Generate Random Items` node                                       |
| To expand functionality, consider adding error handling nodes or notifications on failure.      | n8n error workflow and notification nodes                                                        |
| This workflow was shared by the n8n community and exemplifies API integration with Mealie.       | https://mealie.io/ (official Mealie website)                                                      |

---

This detailed reference document fully captures the workflow’s architecture, node configurations, and operational logic, enabling both human users and automated systems to understand, reproduce, and extend it confidently.