Find KlickTipp tags to remove by prefix

https://n8nworkflows.xyz/workflows/find-klicktipp-tags-to-remove-by-prefix-13664


# Find KlickTipp tags to remove by prefix

This technical reference document provides a detailed breakdown of the **"Find KlickTipp tags to remove by prefix"** workflow.

### 1. Workflow Overview
This workflow functions as a specialized utility (sub-workflow) designed to manage tag hygiene within KlickTipp. It identifies which tags currently assigned to a scope should be removed by comparing the existing state in KlickTipp against a "source of truth" provided as input.

The logic is centered around a **Prefix Scope** (e.g., `Zoho |`). By defining a prefix, the workflow ensures it only evaluates and suggests the removal of tags within that specific namespace, preventing the accidental deletion of unrelated manual tags.

#### Logical Blocks:
*   **1.1 Input Reception:** Receives the prefix and the list of tags that *should* exist.
*   **1.2 Data Preparation:** Transforms the "keep" list into fully qualified KlickTipp tag names and fetches all existing tags from the KlickTipp API.
*   **1.3 Difference Analysis:** Compares the two lists to find tags present in KlickTipp but missing from the "keep" list.
*   **1.4 Filtering & Aggregation:** Ensures only tags matching the prefix are considered and formats the final list into a single array for the output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** Captures the execution parameters.
*   **Nodes Involved:** `Input: Prefix + Tags to keep`
*   **Node Details:**
    *   **Type:** Execute Workflow Trigger.
    *   **Configuration:** Uses a JSON example for development containing `prefix` (string) and `setTags` (array of strings).
    *   **Variables:** `{{ $json.prefix }}`, `{{ $json.setTags }}`.
    *   **Connections:** Triggers both the KlickTipp API fetch and the local tag transformation logic.

#### 2.2 Data Preparation
*   **Overview:** Fetches the live tag list from KlickTipp and prepares the "source of truth" list by prepending the prefix.
*   **Nodes Involved:** `List all tags`, `Build prefixed tag names`, `Split prefixed tags`.
*   **Node Details:**
    *   **List all tags (KlickTipp Node):**
        *   **Role:** Retrieves all available tags from the connected KlickTipp account.
        *   **Failures:** Invalid API credentials or KlickTipp API downtime.
    *   **Build prefixed tag names (Set Node):**
        *   **Role:** Uses a JavaScript map expression to combine the prefix and each tag name.
        *   **Expression:** `($json.setTags || []).map(t => { const p = ($json.prefix ?? '').trim(); return p ? \`\${p} \${t}\` : t; })`
    *   **Split prefixed tags (Split Out Node):**
        *   **Role:** Converts the array of strings into individual n8n items for merging.

#### 2.3 Difference Analysis & Filtering
*   **Overview:** Identifies the discrepancy between existing tags and required tags.
*   **Nodes Involved:** `Match keep vs all tags`, `Keep only tags with prefix`.
*   **Node Details:**
    *   **Match keep vs all tags (Merge Node):**
        *   **Mode:** `Combine` / `Keep Non-Matches`.
        *   **Logic:** Compares KlickTipp's `value` field against the `prefixedSetTags` generated in the previous block. It outputs only items that exist in KlickTipp but **not** in the "keep" list.
    *   **Keep only tags with prefix (Filter Node):**
        *   **Role:** A safety layer. It ensures that the "non-matches" found are actually part of the managed prefix scope.
        *   **Condition:** `{{ $json.value }}` starts with `{{ $('Input: Prefix + Tags to keep').item.json.prefix }}`.

#### 2.4 Output Generation
*   **Overview:** Formats the findings into a clean response.
*   **Nodes Involved:** `Collect tags to remove`, `Set tagNamesToRemove`.
*   **Node Details:**
    *   **Collect tags to remove (Aggregate Node):**
        *   **Role:** Groups the individual tag items back into a single array.
    *   **Set tagNamesToRemove (Set Node):**
        *   **Role:** Standardizes the output key name to `tagNamesToRemove`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Input: Prefix + Tags to keep** | Execute Workflow Trigger | Entry point | None | List all tags, Build prefixed tag names | ## Input |
| **Build prefixed tag names** | Set | Format keep-list | Input: Prefix + Tags to keep | Split prefixed tags | ## Determine tags to remove |
| **List all tags** | KlickTipp | Fetch live tags | Input: Prefix + Tags to keep | Match keep vs all tags | ## Determine tags to remove |
| **Split prefixed tags** | Split Out | Itemize keep-list | Build prefixed tag names | Match keep vs all tags | ## Determine tags to remove |
| **Match keep vs all tags** | Merge | Identify differences | List all tags, Split prefixed tags | Keep only tags with prefix | ## Resolve tag differences |
| **Keep only tags with prefix** | Filter | Scope safety | Match keep vs all tags | Collect tags to remove | ## Select tags to remove |
| **Collect tags to remove** | Aggregate | Array formation | Keep only tags with prefix | Set tagNamesToRemove | ## Output: tags to remove |
| **Set tagNamesToRemove** | Set | Final renaming | Collect tags to remove | None | ## Output: tags to remove |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Add an **Execute Workflow Trigger**. Define a test JSON with `prefix` (e.g., `"Zoho |"`) and `setTags` (e.g., `["A", "B"]`).
2.  **API Fetch:** Add a **KlickTipp Node**. Set the action to "Get All Tags". Connect it to the Trigger.
3.  **String Processing:** 
    *   Add a **Set Node** (`Build prefixed tag names`) connected to the Trigger. 
    *   Create an array mapping the input `setTags` to include the `prefix` using the expression: `{{ $json.setTags.map(t => $json.prefix + ' ' + t) }}`.
4.  **Itemization:** Add a **Split Out Node** after the Set node to split the newly created array.
5.  **Comparison Logic:** 
    *   Add a **Merge Node**. 
    *   Connect Input 1 to the **KlickTipp Node**. 
    *   Connect Input 2 to the **Split Out Node**.
    *   Set Mode to `Combine`, Join Mode to `Keep Non-Matches`. Compare `value` (from KlickTipp) with your prefixed field.
6.  **Scope Filtering:** Add a **Filter Node**. Set the rule: `value` must start with the `prefix` value from the Trigger node.
7.  **Data Re-assembly:** 
    *   Add an **Aggregate Node** to collect all `value` fields into an array.
    *   Add a final **Set Node** to wrap this array into a property named `tagNamesToRemove`.
8.  **Credentials:** Ensure a valid KlickTipp API credential is selected in the KlickTipp node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Sub-workflow calculates removal list only | This workflow does **not** delete tags; it only identifies them. |
| Logic based on Tag Names | Comparison is performed on the string value, not Tag IDs. |
| KlickTipp API Requirements | Requires KlickTipp API access (username/password/API key). |