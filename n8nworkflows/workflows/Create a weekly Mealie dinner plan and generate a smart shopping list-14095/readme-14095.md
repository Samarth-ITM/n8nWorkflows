Create a weekly Mealie dinner plan and generate a smart shopping list

https://n8nworkflows.xyz/workflows/create-a-weekly-mealie-dinner-plan-and-generate-a-smart-shopping-list-14095


# Create a weekly Mealie dinner plan and generate a smart shopping list

# Workflow Analysis: Weekly Mealie Dinner Plan & Shopping List

## 1. Workflow Overview
This workflow automates the process of dinner planning and grocery procurement using the **Mealie** self-hosted recipe manager. It follows a cycle of generating a random weekly plan, allowing the user to review and modify it via an interactive email, and finally aggregating the required ingredients into a Mealie shopping list.

The logic is divided into three primary functional blocks:
- **1.1 Schedule and Meal Generation:** Handles the timing and the creation of a random 7-day meal plan in Mealie.
- **1.2 Meal Plan Email Workflow:** Fetches the generated plan, formats it into a professional HTML email, and waits for user feedback (removals).
- **1.3 Shopping List Creation:** Processes the final approved meals, retrieves detailed recipe data, and populates a shopping list in Mealie.

---

## 2. Block-by-Block Analysis

### 2.1 Schedule and Meal Generation
**Overview:** This block triggers the automation and populates the Mealie calendar with random dinner entries for the upcoming week.

- **Nodes Involved:** 
    - `Weekly Schedule Trigger`
    - `Generate Upcoming Week`
    - `Generate Random Meal Plan`
- **Node Details:**
    - **Weekly Schedule Trigger** (Schedule Trigger): Triggers the workflow once per week.
    - **Generate Upcoming Week** (Code): A JavaScript node that calculates the next 7 days from the current date. 
        - *Output:* An array of dates and day names.
    - **Generate Random Meal Plan** (HTTP Request): Sends a `POST` request to Mealie's `/api/households/mealplans/random` endpoint.
        - *Configuration:* Uses `date` and `entryType: dinner` from the previous node.
        - *Auth:* Mealie Bearer Token.
        - *Failure point:* Potential timeout if the Mealie instance is unreachable or the API token expires.

### 2.2 Meal Plan Email Workflow
**Overview:** This block transforms raw API data into a user-friendly email and handles the "Human-in-the-loop" approval process.

- **Nodes Involved:**
    - `Fetch Current Week Meal Plans`
    - `Prepare Meal Plan Email Data`
    - `Send Meal Plan Email`
    - `Normalize User Response`
    - `Check for Removals`
- **Node Details:**
    - **Fetch Current Week Meal Plans** (HTTP Request): GETs the meal plan for the current date range.
    - **Prepare Meal Plan Email Data** (Code): Complex JS node that generates a stylized HTML table of meals and defines a JSON form for the Gmail response.
    - **Send Meal Plan Email** (Gmail): Sends the HTML email and uses the `sendAndWait` operation. This creates a wait state until the user submits the form.
    - **Normalize User Response** (Code): Parses the form data returned from Gmail to identify which meals the user marked as "Remove from meal plan".
    - **Check for Removals** (If): A conditional branch. If the `removals` array is empty, it proceeds to shopping list creation; otherwise, it triggers the deletion logic.

### 2.3 Shopping List Creation
**Overview:** This block converts the approved meal plan into a tangible shopping list by extracting ingredients from each recipe.

- **Nodes Involved:**
    - `Split Removals Array`
    - `Delete Random Meal Plan`
    - `Fetch Recipe By Slug`
    - `Create Shopping List in Mealie`
    - `Normalize Recipe Data`
    - `Add Ingredients To Shopping List`
- **Node Details:**
    - **Split Removals Array** (Split Out): Turns the list of meals to remove into individual items for processing.
    - **Delete Random Meal Plan** (HTTP Request): Sends a `DELETE` request to Mealie to remove unwanted meals.
    - **Fetch Recipe By Slug** (HTTP Request): Retrieves the full recipe details (including ingredients) for each meal in the plan.
    - **Create Shopping List in Mealie** (HTTP Request): Creates a new shopping list container named "Shopping List Week of [Date]".
    - **Normalize Recipe Data** (Code): Maps the detailed recipe JSON into the specific format required by Mealie's shopping list API (e.g., `recipeId`, `recipeIncrementQuantity`).
    - **Add Ingredients To Shopping List** (HTTP Request): `POST` request to the specific shopping list ID to add the normalized ingredients.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Schedule Trigger | Schedule Trigger | Workflow Entry | - | Generate Upcoming Week | Schedule and meal generation |
| Generate Upcoming Week | Code | Date Calculation | Weekly Schedule Trigger | Generate Random Meal Plan | Schedule and meal generation |
| Generate Random Meal Plan | HTTP Request | API Meal Creation | Generate Upcoming Week / Delete Random Meal Plan | Fetch Current Week Meal Plans | Schedule and meal generation |
| Fetch Current Week Meal Plans | HTTP Request | API Data Retrieval | Generate Random Meal Plan | Prepare Meal Plan Email Data | Meal plan email workflow |
| Prepare Meal Plan Email Data | Code | HTML/Form Formatting | Fetch Current Week Meal Plans | Send Meal Plan Email | Meal plan email workflow |
| Send Meal Plan Email | Gmail | User Interaction | Prepare Meal Plan Email Data | Normalize User Response | Meal plan email workflow |
| Normalize User Response | Code | Data Parsing | Send Meal Plan Email | Check for Removals | Meal plan email workflow |
| Check for Removals | If | Logic Branching | Normalize User Response | Fetch Recipe By Slug / Split Removals Array | Meal plan email workflow |
| Split Removals Array | Split Out | Item Iteration | Check for Removals | Delete Random Meal Plan | Shopping list creation |
| Delete Random Meal Plan | HTTP Request | API Data Deletion | Split Removals Array | Generate Random Meal Plan | Shopping list creation |
| Fetch Recipe By Slug | HTTP Request | Detail Retrieval | Check for Removals | Create Shopping List in Mealie | Shopping list creation |
| Create Shopping List in Mealie | HTTP Request | List Initialization | Fetch Recipe By Slug | Normalize Recipe Data | Shopping list creation |
| Normalize Recipe Data | Code | Data Mapping | Create Shopping List in Mealie | Add Ingredients To Shopping List | Shopping list creation |
| Add Ingredients To Shopping List | HTTP Request | Final API Injection | Normalize Recipe Data | - | Shopping list creation |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Initialization & Trigger
1. Create a **Schedule Trigger** node set to a weekly interval.
2. Add a **Code** node (`Generate Upcoming Week`) to generate an array of the next 7 dates.

### Step 2: Mealie Plan Generation
1. Create an **HTTP Request** node (`Generate Random Meal Plan`).
    - Method: `POST`
    - URL: `http://<your-mealie-ip>:9925/api/households/mealplans/random`
    - Body: JSON containing `date` (from Code node) and `entryType: dinner`.
    - Credentials: HTTP Bearer Auth (Mealie Token).

### Step 3: User Approval Loop
1. Create an **HTTP Request** node to fetch the meal plan for the current week (`/api/households/mealplans`).
2. Create a **Code** node to build the HTML body and the `fields` array for the email.
3. Create a **Gmail** node:
    - Operation: `sendAndWait`
    - Configure the `message` to use the HTML from the previous node.
    - Configure `jsonOutput` to capture the form fields.
4. Create a **Code** node to parse the Gmail response and extract the IDs of recipes to be removed.
5. Create an **If** node to check if the `removals` array is empty.

### Step 4: Plan Adjustment (Optional Path)
1. Connect the `False` (removals found) branch of the If node to a **Split Out** node to isolate each removal.
2. Connect to an **HTTP Request** node using the `DELETE` method against `/api/households/mealplans/{{id}}`.
3. Loop this back to the **Generate Random Meal Plan** node to refill the gap.

### Step 5: Shopping List Finalization (True Path)
1. Connect the `True` (no removals) branch to an **HTTP Request** node to fetch full recipe details using the recipe slug.
2. Create an **HTTP Request** node to create a new shopping list (`POST /api/households/shopping/lists`).
3. Add a **Code** node to format the recipe ingredients into the structure: `recipeId`, `recipeIncrementQuantity`, and `recipeIngredients`.
4. Final **HTTP Request** node: `POST` to `/api/households/shopping/lists/{{list_id}}/recipe` with the normalized JSON body.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Setup requires Mealie API credentials and a configured Gmail OAuth2 connection. | Technical Requirements |
| The "Generate Upcoming Week" logic can be modified to start on specific days (e.g., every Sunday). | Customization |
| Ensure the Mealie IP address is correctly replaced in all HTTP nodes. | Configuration |