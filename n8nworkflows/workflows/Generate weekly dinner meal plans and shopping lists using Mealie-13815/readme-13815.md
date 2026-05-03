Generate weekly dinner meal plans and shopping lists using Mealie

https://n8nworkflows.xyz/workflows/generate-weekly-dinner-meal-plans-and-shopping-lists-using-mealie-13815


# Generate weekly dinner meal plans and shopping lists using Mealie

# Workflow Analysis: Generate Weekly Dinner Meal Plans and Shopping Lists using Mealie

This workflow automates the process of generating a 7-day dinner plan and a corresponding shopping list within a self-hosted Mealie instance. It eliminates manual planning by randomly selecting recipes for the upcoming week and aggregating all necessary ingredients into a newly created shopping list.

---

### 1. Workflow Overview

The workflow is designed for users who want a "hands-off" approach to weekly meal prep. It follows a linear logical progression:
*   **1.1 Temporal Initialization:** Generates a list of dates for the next 7 days.
*   **1.2 Meal Generation:** For each date, it requests Mealie to pick a random "dinner" and assign it to the calendar.
*   **1.3 Data Retrieval:** Fetches the full details (ingredients/servings) for each selected recipe.
*   **1.4 List Preparation:** Creates a single unified shopping list named for that specific week.
*   **1.5 Data Normalization & Sync:** Cleans the recipe data and pushes all ingredients into the created Mealie shopping list.

---

### 2. Block-by-Block Analysis

#### 2.1 Timeline & Generation
**Overview:** This block establishes the timeframe and populates the Mealie calendar with random selections.
*   **Nodes Involved:** `Schedule Trigger`, `Generate Next 7 Days`, `Generate Random Meal in Mealie`.
*   **Node Details:**
    *   **Schedule Trigger:** Fires once per week.
    *   **Generate Next 7 Days (Code):** Uses JavaScript to create an array of 7 ISO-formatted dates starting from today.
    *   **Generate Random Meal in Mealie (HTTP Request):** 
        *   **Method:** POST to `/api/households/mealplans/random`.
        *   **Config:** Passes the generated date and sets `entryType` to "dinner".
        *   **Edge Case:** Requires the Mealie instance to have enough recipes to satisfy a random selection; may fail if the database is empty or the network is unreachable.

#### 2.2 Recipe Detail Retrieval
**Overview:** Obtains the specific requirements (ingredients) for the meals selected in the previous step.
*   **Nodes Involved:** `Get Recipe From Mealie By Slug`.
*   **Node Details:**
    *   **HTTP Request:** Performs a GET request using the recipe slug returned by the previous node.
    *   **Variable:** `{{ $json.recipe.slug }}`.
    *   **Edge Case:** If a recipe is deleted between the generation and fetch steps, this node will return a 404 error.

#### 2.3 Shopping List Management
**Overview:** Creates a target container for the week's groceries.
*   **Nodes Involved:** `Create New Shopping List in Mealie`.
*   **Node Details:**
    *   **HTTP Request:** POST to `/api/households/shopping/lists`.
    *   **Configuration:** Named dynamically: `"Shopping List Week of {{ $('Generate Next 7 Days').item.json.date }}"`.
    *   **Special Setting:** **Execute Once** is enabled to ensure only one list is created for the entire batch of 7 recipes.

#### 2.4 Data Normalization & Injection
**Overview:** Reformats raw recipe data into Mealie's expected schema for shopping lists and performs the final upload.
*   **Nodes Involved:** `Normalize Mealie Recipe Data`, `Add Recipe(s) Ingredients To Shopping List in Mealie`.
*   **Node Details:**
    *   **Normalize Mealie Recipe Data (Code):** Aggregates all recipe results from previous steps. It maps `recipeId`, `recipeServings`, and a detailed `recipeIngredients` array (quantity, unit, food, etc.).
    *   **Add Recipe(s) Ingredients... (HTTP Request):** 
        *   **Method:** POST to `/api/households/shopping/lists/{list_id}/recipe`.
        *   **Input:** Receives the normalized array from the Code node.
        *   **Requirement:** Relies on the ID generated in the "Create New Shopping List" node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule Trigger | Workflow entry point | None | Generate Next 7 Days | |
| Generate Next 7 Days | Code | Date array generation | Schedule Trigger | Generate Random Meal... | |
| Generate Random Meal in Mealie | HTTP Request | Create meal plan entry | Generate Next 7 Days | Get Recipe From Mealie... | 1. Generate Dates for the Full Week and Request Random Dinners |
| Get Recipe From Mealie By Slug | HTTP Request | Fetch recipe details | Generate Random Meal... | Create New Shopping List... | 2. Fetch Full Recipe Details from Mealie |
| Create New Shopping List in Mealie | HTTP Request | Create empty list | Get Recipe From Mealie... | Normalize Mealie Recipe Data | 3. Create a New Weekly Shopping List in Mealie |
| Normalize Mealie Recipe Data | Code | Format ingredient data | Create New Shopping List... | Add Recipe(s) Ingredients... | 4. Normalize Recipe Data and Add Ingredients to the Shopping List |
| Add Recipe(s) Ingredients To Shopping List in Mealie | HTTP Request | Upload ingredients | Normalize Mealie Recipe Data | None | 4. Normalize Recipe Data and Add Ingredients to the Shopping List |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger:** Add a **Schedule Trigger** set to repeat every 1 week.
2.  **Date Logic:** Add a **Code** node ("Generate Next 7 Days"). Use a loop to generate dates for $i=0$ to $i=6$ and return them as an array of objects containing `date` (YYYY-MM-DD).
3.  **Meal Plan:** Add an **HTTP Request** node.
    *   **Method:** POST
    *   **URL:** `http://<YOUR_MEALIE_IP>/api/households/mealplans/random`
    *   **Auth:** Header Auth (Bearer Token).
    *   **Body:** JSON with `date` (from Code node) and `entryType: "dinner"`.
4.  **Fetch Details:** Add an **HTTP Request** node.
    *   **Method:** GET
    *   **URL:** `http://<YOUR_MEALIE_IP>/api/recipes/{{ $json.recipe.slug }}`.
5.  **Create List:** Add an **HTTP Request** node.
    *   **Method:** POST to `/api/households/shopping/lists`.
    *   **Settings:** Enable **Execute Once**.
    *   **Body:** `{"name": "Shopping List Week of <Date>"}`.
6.  **Normalization:** Add a **Code** node. Use `const recipes = $('Get Recipe From Mealie By Slug').all();` to collect all previous items. Map them to a structure containing `recipeId` and the `recipeIngredients` array.
7.  **Final Sync:** Add an **HTTP Request** node.
    *   **Method:** POST
    *   **URL:** Use the ID from step 5: `/api/households/shopping/lists/{{$node["Create New Shopping List"].json["id"]}}/recipe`.
    *   **Body:** Set to the output of the Normalization Code node.
8.  **Credentials:** Create a "Header Auth" credential in n8n with the Name `Authorization` and Value `Bearer <YOUR_MEALIE_API_TOKEN>`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires a running Mealie instance (Self-hosted) | Pre-requisite for all API calls |
| Join the community for support | [n8n Forum](https://community.n8n.io/) / [Discord](https://discord.com/invite/XPKeKXeB7d) |
| Official Mealie API Documentation | Check your instance at `/docs` for endpoint verification |
| Update IP addresses in all HTTP nodes | Critical: Change `<mealie ip address>` to your actual server IP |