Resolve and create KlickTipp tags from names with optional prefixes

https://n8nworkflows.xyz/workflows/resolve-and-create-klicktipp-tags-from-names-with-optional-prefixes-13699


# Resolve and create KlickTipp tags from names with optional prefixes

This document provides a comprehensive technical analysis of the **KlickTipp Tag Manager** workflow. This utility is designed to resolve existing tags and automatically create missing ones in KlickTipp, optionally applying a prefix for standardized namespacing.

---

### 1. Workflow Overview
The workflow functions as a reusable sub-workflow (utility) that ensures a specific set of tags exists in a KlickTipp account. It eliminates manual tag creation and prevents duplicate errors by checking the current tag database before attempting to create new entries.

**Logic Groups:**
- **1.1 Input Reception:** Receives an array of base tag names and an optional prefix.
- **1.2 Preparation:** Normalizes strings and handles the concatenation of the prefix and tag names.
- **1.3 Resolution & Comparison:** Fetches existing tags from KlickTipp and compares them against the requested list.
- **1.4 Creation Logic:** Triggers the creation of any tags identified as missing.
- **1.5 Consolidation:** Aggregates both previously existing and newly created tags into a single clean output array.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Preparation
**Overview:** Receives data from a parent workflow and prepares the strings for comparison.
- **Nodes Involved:** `Input: Prefix + tag names`, `Split tagNames into items`, `Map tagNames -> value`.
- **Node Details:**
    - **Input: Prefix + tag names (Execute Workflow Trigger):** Defines the interface. Expects `tagNames` (Array) and `prefix` (String).
    - **Split tagNames into items (Split Out):** Converts the input array into individual items for per-tag processing.
    - **Map tagNames -> value (Set):** Uses a JavaScript expression to combine the prefix and tag name. It handles spacing logic (e.g., ensuring a space exists between the prefix and the tag if not already present).
- **Edge Cases:** If `prefix` is null or empty, the node returns only the base tag name.

#### 2.2 Resolve & Detect Missing Tags
**Overview:** Compares the desired tag list against the actual tags currently in KlickTipp.
- **Nodes Involved:** `Get tag list`, `Find existing tags`, `Find tags to create`.
- **Node Details:**
    - **Get tag list (KlickTipp):** Fetches all existing tags from the configured KlickTipp account.
    - **Find existing tags (Merge):** Uses "Combine" mode to find the intersection between requested tags and existing KlickTipp tags based on the `value` field.
    - **Find tags to create (Merge):** Uses "Combine" mode with `keepNonMatches` for `input2`. This isolates the tags that were requested but do not exist in the KlickTipp list.
- **Failure Types:** Authentication errors with KlickTipp API; API rate limiting if the account has thousands of tags.

#### 2.3 Creation & Formatting
**Overview:** Executes API calls for new tags and formats the results.
- **Nodes Involved:** `Create new tag`, `Set the created tag`.
- **Node Details:**
    - **Create new tag (KlickTipp):** Performs the `create` operation for each item passed from the "Find tags to create" node.
    - **Set the created tag (Set):** Standardizes the output of the newly created tag to match the structure of the "existing" tags (id and value), ensuring they can be merged seamlessly.

#### 2.4 Aggregation
**Overview:** Combines all items back into a single response.
- **Nodes Involved:** `Combine existing & new tags`, `Extract tag name`, `Aggregate tag names`.
- **Node Details:**
    - **Combine existing & new tags (Merge):** Merges the stream of existing tags and the stream of newly created tags.
    - **Aggregate tag names (Aggregate):** Groups all individual tag items back into a single array named `tags`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Input: Prefix + tag names** | Trigger | Entry Point | (None) | Get tag list, Split tagNames into items | ## Input |
| **Split tagNames into items** | Split Out | Data Normalization | Input Trigger | Map tagNames -> value | ## 1. Preparation |
| **Map tagNames -> value** | Set | String Formatting | Split tagNames... | Find existing tags, Find tags to create | ## 1. Preparation |
| **Get tag list** | KlickTipp | Data Retrieval | Input Trigger | Find existing tags, Find tags to create | ## 2.1. Resolve existing tags |
| **Find existing tags** | Merge | Filtering | Get tag list, Map tagNames... | Combine existing & new tags | ## 2.1. Resolve existing tags |
| **Find tags to create** | Merge | Filtering | Get tag list, Map tagNames... | Create new tag | ## 2.2. Detect and create missing tags |
| **Create new tag** | KlickTipp | Action | Find tags to create | Set the created tag | ## 2.2. Detect and create missing tags |
| **Set the created tag** | Set | Transformation | Create new tag | Combine existing & new tags | ## 2.2. Detect and create missing tags |
| **Combine existing & new tags** | Merge | Data Union | Find existing tags, Set the created tag | Extract tag name | ## 3. Combine & return tag names |
| **Extract tag name** | Set | Cleaning | Combine... | Aggregate tag names | ## 3. Combine & return tag names |
| **Aggregate tag names** | Aggregate | Final Collection | Extract tag name | (Output) | ## 3. Combine & return tag names |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create an **Execute Workflow Trigger**. Set an example JSON with a `prefix` (string) and `tagNames` (array of strings).
2.  **KlickTipp Fetch:** Add a **KlickTipp Node**, select "Tag" as the resource and "Get All" (Get tag list).
3.  **Data Splitting:** Add a **Split Out Node**. Set "Field to Split Out" to `tagNames`.
4.  **Prefix Logic:** Add a **Set Node** after the Split Out. Create a field `value` using this expression:
    ```javascript
    {{ (() => {
      const prefix = ($('Input: Prefix + tag names').item.json.prefix || '').trim();
      const tag = String($json.tagNames || '').trim();
      if (!prefix) return tag;
      return prefix.endsWith(' ') ? prefix + tag : prefix + ' ' + tag;
    })() }}
    ```
5.  **Comparison (Existing):** Add a **Merge Node**. Mode: `Combine`. Connection 1: KlickTipp "Get tag list". Connection 2: Prefix logic "Set node". Set "Fields to Match" to `value`.
6.  **Comparison (Missing):** Add a **Merge Node**. Mode: `Combine`. Connection 1: KlickTipp "Get tag list". Connection 2: Prefix logic "Set node". Set "Fields to Match" to `value`. **Crucial:** Set "Output Data From" to `input2` and "Include" to `Non-matches`.
7.  **Tag Creation:** Add a **KlickTipp Node** after the "Missing" Merge node. Operation: `Create`. Name: `{{ $json.value }}`.
8.  **Output Mapping:** Add a **Set Node** after "Create" to map the new `id` and `value`.
9.  **Final Union:** Add a **Merge Node** (Append mode) to join the results of Step 5 and Step 8.
10. **Aggregation:** Add an **Aggregate Node** to collect all `value` fields into a final array named `tags`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Functionality** | This workflow prevents the "Tag already exists" error in KlickTipp by pre-validating names. |
| **Namespace Support** | Designed for source-specific tagging (e.g., "Shopify | Customer"). |
| **Sub-workflow calling** | Pass JSON: `{"prefix": "Source | ", "tagNames": ["Tag1", "Tag2"]}`. |
| **KlickTipp API** | Requires a valid KlickTipp API key and Username. |