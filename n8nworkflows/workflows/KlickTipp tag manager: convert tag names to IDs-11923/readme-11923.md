KlickTipp tag manager: convert tag names to IDs

https://n8nworkflows.xyz/workflows/klicktipp-tag-manager--convert-tag-names-to-ids-11923


# KlickTipp tag manager: convert tag names to IDs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** KlickTipp tag manager: convert tag names to IDs  
**Internal name (JSON):** Get or create tag IDs in KlickTipp from an array of tag names

**Purpose:**  
This workflow receives an input array of KlickTipp tag names (`tagNames[]`), resolves which tags already exist in KlickTipp, creates any missing tags, and returns a single aggregated array of tag IDs (`tagIds[]`). It is designed to be reused as a sub-workflow (“get or create tags”) from multiple parent automations.

**Primary use cases**
- Ensure tags exist before tagging contacts in KlickTipp.
- Normalize user-provided tag names into IDs required by other KlickTipp operations.
- Centralize “create if missing” behavior to avoid duplicating logic.

### Logical blocks
1. **Input & Preparation**
   - Receives `tagNames[]`, splits into items, maps each name into a consistent comparison field (`value`).
2. **Resolve existing vs missing**
   - Loads all tags from KlickTipp.
   - Matches input tag names to existing tags and identifies non-matching tag names to create.
3. **Create missing tags + Return IDs**
   - Creates missing tags.
   - Combines existing + newly created tags.
   - Extracts IDs and aggregates them into `tagIds[]`.

---

## 2. Block-by-Block Analysis

### 2.1 Block: Input & Preparation

**Overview:**  
Accepts an array of tag names and converts it into individual items with a standardized field (`value`) used later for matching against KlickTipp tag objects.

**Nodes involved:**
- Input: Tag names
- Split tagNames into items
- Map tagNames -> value

#### Node: **Input: Tag names**
- **Type / role:** `Execute Workflow Trigger` (sub-workflow entry point)
- **Configuration (interpreted):**
  - Input source: JSON example (used for testing and documentation)
  - Example payload:
    - `tagNames`: array of strings
- **Key variables/fields:** expects `$json.tagNames` (array)
- **Connections:**
  - Output → **Get tag list**
  - Output → **Split tagNames into items**
- **Version notes:** typeVersion `1.1`
- **Edge cases / failures:**
  - If `tagNames` is missing or not an array, downstream split/mapping will not behave as intended (may produce zero items or expression errors).
  - Empty array results in no tag creation and likely an empty aggregation downstream.

#### Node: **Split tagNames into items**
- **Type / role:** `Split Out` (turn array into multiple items)
- **Configuration (interpreted):**
  - Field to split: `tagNames`
  - Produces one item per element in `tagNames`
- **Connections:**
  - Input ← **Input: Tag names**
  - Output → **Map tagNames -> value**
- **Version notes:** typeVersion `1`
- **Edge cases / failures:**
  - Non-array `tagNames` can cause unexpected output (single item or failure depending on runtime behavior).

#### Node: **Map tagNames -> value**
- **Type / role:** `Set` (normalize field name for later matching)
- **Configuration (interpreted):**
  - Adds/sets field `value` as string: `={{ $json.tagNames }}`
  - Keeps other fields by default (no explicit “keep only set” configured)
- **Key expressions:**
  - `value = {{ $json.tagNames }}`
- **Connections:**
  - Input ← **Split tagNames into items**
  - Output → **Find existing tags** (input 2 / index 1)
  - Output → **Find tags to create** (input 2 / index 1)
- **Version notes:** typeVersion `3.4`
- **Edge cases / failures:**
  - If an item’s `tagNames` value is `null`/empty string, later matching and tag creation may create undesired empty/invalid tags unless KlickTipp rejects them.

---

### 2.2 Block: Resolve existing vs missing tags

**Overview:**  
Fetches all existing KlickTipp tags and compares them to the requested tag names. One branch keeps matches (existing tags). The other isolates non-matches (tags to create).

**Nodes involved:**
- Get tag list
- Find existing tags
- Find tags to create

#### Node: **Get tag list**
- **Type / role:** `KlickTipp` node (API call to list tags)
- **Configuration (interpreted):**
  - Operation not explicitly shown in parameters (defaults to a “list/get tags” operation for this node type/version).
  - Uses KlickTipp API credentials.
- **Credentials:**
  - `klickTippApi` credential required (API access)
- **Connections:**
  - Input ← **Input: Tag names**
  - Output → **Find existing tags** (input 1 / index 0)
  - Output → **Find tags to create** (input 1 / index 0)
- **Version notes:** typeVersion `3`
- **Edge cases / failures:**
  - Credential/auth errors (invalid API key/session).
  - API downtime/timeouts.
  - Large tag lists may impact runtime; comparison is item-based and can grow execution time.

#### Node: **Find existing tags**
- **Type / role:** `Merge` (match requested names to existing tags)
- **Configuration (interpreted):**
  - Mode: “Combine”
  - Match field (string): `value`
  - Intended behavior: produce matched pairs where an existing tag matches an input `value` (tag name).
  - **Important:** With “combine” merges, the exact result depends on how KlickTipp returns tag fields. This workflow assumes the KlickTipp tag list items have a field named `value` that corresponds to tag name, or that n8n’s KlickTipp node outputs a compatible field for matching.
- **Connections:**
  - Input 1 ← **Get tag list**
  - Input 2 ← **Map tagNames -> value**
  - Output → **Combine existing & new tags** (index 0)
- **Version notes:** typeVersion `3.2`
- **Edge cases / failures:**
  - If KlickTipp returns tag name under a different field (e.g., `name` instead of `value`), matching will fail and all tags will be treated as missing.
  - Duplicate tag names in input can yield duplicate matched items (and duplicates in final `tagIds[]`).
  - Case sensitivity/whitespace differences may prevent matches and cause unintended tag creation.

#### Node: **Find tags to create**
- **Type / role:** `Merge` (detect which requested names do *not* exist)
- **Configuration (interpreted):**
  - Mode: “Combine”
  - Join mode: “keepNonMatches”
  - Output data from: `input2` (the requested tags stream)
  - Match field (string): `value`
  - Intended behavior: compare existing tag list (input 1) vs requested names (input 2) and emit only those requested items that have no match in existing tags.
- **Connections:**
  - Input 1 ← **Get tag list**
  - Input 2 ← **Map tagNames -> value**
  - Output → **Create new tag**
- **Version notes:** typeVersion `3.2`
- **Edge cases / failures:**
  - Same field-name mismatch risk as above (all tags could be flagged “missing”).
  - If KlickTipp allows tags differing only by case, this flow might create near-duplicates unless you normalize names first.

---

### 2.3 Block: Create missing tags + combine + return IDs

**Overview:**  
Creates missing tags in KlickTipp, merges them with already-existing tags, extracts their IDs, and aggregates all IDs into a single `tagIds[]` array output.

**Nodes involved:**
- Create new tag
- Combine existing & new tags
- Extract only tag IDs
- Aggregate tag IDs

#### Node: **Create new tag**
- **Type / role:** `KlickTipp` node (API call to create tag)
- **Configuration (interpreted):**
  - Operation: `create`
  - Tag name: `={{ $json.value }}`
- **Credentials:**
  - `klickTippApi` credential required
- **Connections:**
  - Input ← **Find tags to create**
  - Output → **Combine existing & new tags** (index 1)
- **Version notes:** typeVersion `3`
- **Edge cases / failures:**
  - If tag already exists (race condition or mismatch in earlier compare), API may reject creation or create duplicates depending on KlickTipp behavior.
  - Invalid/empty names can cause API validation errors.
  - Rate limits if many tags are created at once.

#### Node: **Combine existing & new tags**
- **Type / role:** `Merge` (union of tags that exist and tags just created)
- **Configuration (interpreted):**
  - Default merge settings (parameters empty in JSON); in n8n, this typically means it will merge the two incoming streams into one output stream (behavior depends on default mode; often “append”/“combine” style).
- **Connections:**
  - Input 1 ← **Find existing tags**
  - Input 2 ← **Create new tag**
  - Output → **Extract only tag IDs**
- **Version notes:** typeVersion `3.2`
- **Edge cases / failures:**
  - If one branch produces no items, ensure merge mode still forwards items from the other branch (default behavior usually does, but depends on mode).
  - Potential duplication if an item appears in both streams (shouldn’t happen logically, but can in mismatch scenarios).

#### Node: **Extract only tag IDs**
- **Type / role:** `Set` (map tag object → numeric ID field for aggregation)
- **Configuration (interpreted):**
  - Sets `tagIds` as number: `={{ $json.id }}`
- **Key expressions:**
  - `tagIds = {{ $json.id }}`
- **Connections:**
  - Input ← **Combine existing & new tags**
  - Output → **Aggregate tag IDs**
- **Version notes:** typeVersion `3.4`
- **Edge cases / failures:**
  - If KlickTipp returns ID under a different key than `id`, output will be `null` and aggregation will be wrong.
  - If `id` is a string, coercion to number may fail or produce unexpected results.

#### Node: **Aggregate tag IDs**
- **Type / role:** `Aggregate` (collect all `tagIds` values into an array)
- **Configuration (interpreted):**
  - Aggregates field: `tagIds`
  - Produces a single item with `tagIds: [ ... ]`
- **Connections:**
  - Input ← **Extract only tag IDs**
  - Output: none (final result)
- **Version notes:** typeVersion `1`
- **Edge cases / failures:**
  - If upstream emits zero items, result may be empty output (or no item) depending on aggregate behavior/version.
  - If upstream contains null/undefined `tagIds`, aggregation may include nulls.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Input: Tag names | Execute Workflow Trigger | Sub-workflow entry point; receives `tagNames[]` | — | Get tag list; Split tagNames into items | ## Input |
| Get tag list | KlickTipp | Fetch all tags from KlickTipp for matching | Input: Tag names | Find existing tags; Find tags to create | ## 1. Preparation |
| Split tagNames into items | Split Out | Turn `tagNames[]` into one item per tag name | Input: Tag names | Map tagNames -> value | ## 1. Preparation |
| Map tagNames -> value | Set | Normalize each tag name into field `value` | Split tagNames into items | Find existing tags; Find tags to create | ## 1. Preparation |
| Find existing tags | Merge | Match requested tags with existing KlickTipp tags | Get tag list; Map tagNames -> value | Combine existing & new tags | ## 2.1. Resolve existing tags |
| Find tags to create | Merge | Output requested tags that have no match (need creation) | Get tag list; Map tagNames -> value | Create new tag | ## 2.2. Detect and create missing tags |
| Create new tag | KlickTipp | Create missing tags in KlickTipp | Find tags to create | Combine existing & new tags | ## 2.2. Detect and create missing tags |
| Combine existing & new tags | Merge | Union of existing tags + newly created tags | Find existing tags; Create new tag | Extract only tag IDs | ## 3. Combine & return tag IDs |
| Extract only tag IDs | Set | Map each tag object to `tagIds = id` | Combine existing & new tags | Aggregate tag IDs | ## 3. Combine & return tag IDs |
| Aggregate tag IDs  | Aggregate | Aggregate all IDs into `tagIds[]` | Extract only tag IDs | — | ## 3. Combine & return tag IDs |
| Sticky Note | Sticky Note | Comment block | — | — | ## 3. Combine & return tag IDs |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## 2.2. Detect and create missing tags |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## 2.1. Resolve existing tags |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## 1. Preparation |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## Input |
| Sticky Note5 | Sticky Note | Comment block (intro + setup + I/O examples) | — | — | ## Introduction… (contains setup, call example, output example, testing notes) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Get or create tag IDs in KlickTipp from an array of tag names** (or your preferred name).

2. **Add the entry node (sub-workflow trigger)**
   - Add node: **Execute Workflow Trigger**
   - Name: **Input: Tag names**
   - Configure a JSON example for testing:
     ```json
     { "tagNames": ["Tag name 1", "Tag name 2", "Tag name 3"] }
     ```

3. **Add KlickTipp “Get tag list”**
   - Add node: **KlickTipp**
   - Name: **Get tag list**
   - Configure credentials: create/select **KlickTipp API** credentials (the workflow uses `klickTippApi`).
   - Set operation to the node’s “list/get tags” equivalent (depends on your KlickTipp node UI/version).
   - Connect: **Input: Tag names → Get tag list**

4. **Split the input array**
   - Add node: **Split Out**
   - Name: **Split tagNames into items**
   - Field to split out: `tagNames`
   - Connect: **Input: Tag names → Split tagNames into items**

5. **Map each tag name to `value`**
   - Add node: **Set**
   - Name: **Map tagNames -> value**
   - Add assignment:
     - Field: `value` (type: string)
     - Value expression: `{{ $json.tagNames }}`
   - Connect: **Split tagNames into items → Map tagNames -> value**

6. **Find existing tags (match)**
   - Add node: **Merge**
   - Name: **Find existing tags**
   - Mode: **Combine**
   - Match field (string): `value`
   - Connect:
     - **Get tag list → Find existing tags** (into Input 1)
     - **Map tagNames -> value → Find existing tags** (into Input 2)

7. **Find tags to create (non-matches)**
   - Add node: **Merge**
   - Name: **Find tags to create**
   - Mode: **Combine**
   - Join mode: **Keep Non-Matches**
   - Output data from: **Input 2**
   - Match field (string): `value`
   - Connect:
     - **Get tag list → Find tags to create** (into Input 1)
     - **Map tagNames -> value → Find tags to create** (into Input 2)

8. **Create missing tags**
   - Add node: **KlickTipp**
   - Name: **Create new tag**
   - Operation: **Create** (tag)
   - Name field (tag name): expression `{{ $json.value }}`
   - Credentials: same KlickTipp API credential
   - Connect: **Find tags to create → Create new tag**

9. **Combine existing + new tags**
   - Add node: **Merge**
   - Name: **Combine existing & new tags**
   - Use a merge mode that outputs items from both inputs (matching the default behavior in your n8n version; commonly “Append” or equivalent if required by UI).
   - Connect:
     - **Find existing tags → Combine existing & new tags** (Input 1)
     - **Create new tag → Combine existing & new tags** (Input 2)

10. **Extract IDs**
   - Add node: **Set**
   - Name: **Extract only tag IDs**
   - Add assignment:
     - Field: `tagIds` (type: number)
     - Value expression: `{{ $json.id }}`
   - Connect: **Combine existing & new tags → Extract only tag IDs**

11. **Aggregate IDs into an array**
   - Add node: **Aggregate**
   - Name: **Aggregate tag IDs**
   - Configure “Fields to aggregate” to include `tagIds`
   - Connect: **Extract only tag IDs → Aggregate tag IDs**
   - The output should be a single item like:
     ```json
     { "tagIds": [123, 456, 789] }
     ```

12. **(Optional) Add sticky notes / documentation blocks**
   - Add sticky notes with headings:
     - “## Input”
     - “## 1. Preparation”
     - “## 2.1. Resolve existing tags”
     - “## 2.2. Detect and create missing tags”
     - “## 3. Combine & return tag IDs”
   - Add the longer introduction note if you want the embedded usage instructions.

**Sub-workflow usage expectation**
- **Input to this workflow (from parent via Execute Workflow node):**
  ```json
  { "tagNames": ["Tag A", "Tag B"] }
  ```
- **Output returned to parent:**
  ```json
  { "tagIds": [111111, 222222] }
  ```

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow receives an array of tag names, checks which tags already exist in KlickTipp, creates the missing ones, and returns a unified array of tag IDs — so you can reuse the same “get or create tags” logic across multiple automations. | Sticky note “Introduction” |
| Setup: Configure KlickTipp credentials in the KlickTipp nodes. | Sticky note “Setup Instructions” |
| How to call via parent workflow: use Execute Sub-workflow / Execute Workflow and pass `{ "tagNames": [...] }`. | Sticky note “How to call this sub-workflow” |
| Output: `{ "tagIds": [123456789, ...] }` | Sticky note “Output” |
| Testing suggestions: mix existing/new names; confirm tags appear in KlickTipp; use `tagIds[]` in parent workflow. | Sticky note “Testing” |