Generate images from Airtable records with Layerre and Canva

https://n8nworkflows.xyz/workflows/generate-images-from-airtable-records-with-layerre-and-canva-13881


# Generate images from Airtable records with Layerre and Canva

# 1. Workflow Overview

This workflow generates images from Airtable records by using Layerre to render Canva-based template variants, then writes the generated image URL back into Airtable.

Typical use cases include:
- Generating personalized lead images
- Creating event visuals from structured Airtable data
- Producing product cards or marketing assets at scale

The workflow is structured into four main logical blocks:

## 1.1 Trigger and One-Time Template Initialization
The workflow starts manually. It includes an optional one-time Layerre action that converts a Canva design into a reusable Layerre template.

## 1.2 Airtable Record Retrieval
After the template exists, the workflow fetches records from an Airtable base/table. Each record acts as one input item for image generation.

## 1.3 Variant Rendering with Layerre
For each Airtable record, the workflow creates one Layerre variant by mapping Airtable fields to specific Canva layers.

## 1.4 Airtable Record Update
Once Layerre returns the generated image URL, the workflow updates the original Airtable record with that URL.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and One-Time Template Initialization

### Overview
This block provides the workflow entry point and an optional setup step for creating a Layerre template from a Canva design. The template creation is intended to be run once, then disabled for normal production runs.

### Nodes Involved
- `When clicking "Test"`
- `Create Template from Canva`

### Node Details

#### When clicking "Test"`
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry node for test and ad hoc execution.
- **Configuration choices:** No parameters are configured. It simply starts the workflow when manually executed.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No input
  - Output → `Create Template from Canva`
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - No functional error on its own
  - Only usable for manual runs; not suitable for unattended automation
- **Sub-workflow reference:** None.

#### Create Template from Canva
- **Type and technical role:** `n8n-nodes-layerre.layerre`; Layerre node used here to create a template from a Canva design URL.
- **Configuration choices:**
  - Node is currently **disabled**
  - `canvaUrl` is empty in the provided workflow and must be filled before use
  - Intended as a one-time setup node
- **Key expressions or variables used:** None in current configuration.
- **Input and output connections:**
  - Input ← `When clicking "Test"`
  - Output → `List Airtable records`
- **Version-specific requirements:** Type version 1 of the Layerre node.
- **Edge cases or potential failure types:**
  - Missing or invalid Layerre credentials
  - Empty `canvaUrl`
  - Invalid or inaccessible Canva design URL
  - Layer parsing issues if the Canva design is unsupported or malformed
  - If the node remains disabled, n8n will skip it; depending on workflow behavior, downstream execution should be validated in the editor when rebuilding
- **Sub-workflow reference:** None.

---

## 2.2 Airtable Record Retrieval

### Overview
This block reads source records from Airtable. Each returned Airtable record becomes one item to process in the image-generation phase.

### Nodes Involved
- `List Airtable records`

### Node Details

#### List Airtable records
- **Type and technical role:** `n8n-nodes-base.airtable`; Airtable search/read node used to fetch records.
- **Configuration choices:**
  - Operation: `search`
  - Base: selected from Airtable resources
  - Table: selected from Airtable resources
  - No additional options configured
- **Key expressions or variables used:** None in current configuration.
- **Input and output connections:**
  - Input ← `Create Template from Canva`
  - Output → `Create a variant`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**
  - Missing or invalid Airtable Personal Access Token
  - Base or table not selected
  - Permission issues on the Airtable base
  - Empty result set, which means no variants will be generated
  - If later expressions expect fields like `Name` or `Image url`, records missing those fields may break downstream mapping or create incomplete variants
- **Sub-workflow reference:** None.

---

## 2.3 Variant Rendering with Layerre

### Overview
This block transforms each Airtable record into a Layerre image variant. It maps Airtable fields onto Canva layer overrides, such as text and image layers.

### Nodes Involved
- `Create a variant`

### Node Details

#### Create a variant
- **Type and technical role:** `n8n-nodes-layerre.layerre`; Layerre node used to create a rendered variant from an existing template.
- **Configuration choices:**
  - Resource: `variant`
  - `templateId`: empty in the provided workflow and must be filled with the template ID created in Step 1
  - Overrides include two layer mappings:
    1. A text override using `={{ $json.fields.Name }}`
    2. An image override using `={{ $json.fields['Image url'] }}`
  - Both `layerId` values are empty and must be set to the correct Canva/Layerre layer identifiers
  - `variantDimensions` left empty
  - `requestOptions` empty
- **Key expressions or variables used:**
  - `={{ $json.fields.Name }}`
  - `={{ $json.fields['Image url'] }}`
- **Input and output connections:**
  - Input ← `List Airtable records`
  - Output → `Update Airtable record`
- **Version-specific requirements:** Type version 1 of the Layerre node.
- **Edge cases or potential failure types:**
  - Missing or invalid Layerre credentials
  - Empty `templateId`
  - Missing `layerId` values in overrides
  - Field name mismatches in Airtable, such as `Name` or `Image url` not existing
  - Null or invalid source image URLs
  - Layer type mismatch, e.g. applying text to an image layer or image URL to a text layer
  - API failures or rendering timeouts
  - Output may not contain `url` if rendering fails or returns an unexpected payload
- **Sub-workflow reference:** None.

---

## 2.4 Airtable Record Update

### Overview
This block writes the generated image URL back to the same Airtable record that produced the variant. It uses the original Airtable record ID and the Layerre output URL.

### Nodes Involved
- `Update Airtable record`

### Node Details

#### Update Airtable record
- **Type and technical role:** `n8n-nodes-base.airtable`; Airtable update node for writing data back to an existing record.
- **Configuration choices:**
  - Operation: `update`
  - Base: selected from Airtable resources
  - Table: selected from Airtable resources
  - Mapping mode: define fields below
  - Matching column: `id`
  - Mapped fields:
    - `id` = `={{ $('List Airtable records').item.json.id }}`
    - `Generated url` = `={{ $json.url }}`
- **Key expressions or variables used:**
  - `={{ $('List Airtable records').item.json.id }}`
  - `={{ $json.url }}`
- **Input and output connections:**
  - Input ← `Create a variant`
  - No downstream output
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**
  - Missing or invalid Airtable credentials
  - Base/table mismatch with source records
  - `Generated url` field not existing in Airtable
  - Expression resolution failure if item linking to `List Airtable records` is broken
  - Missing or empty `$json.url` from Layerre response
  - Record update failure if Airtable record ID is invalid or inaccessible
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking "Test" | Manual Trigger | Manual workflow entry point |  | Create Template from Canva |  |
| Create Template from Canva | Layerre | One-time creation of a Layerre template from a Canva design | When clicking "Test" | List Airtable records | ## Step 1: Create Template (one-time)<br>Create a Layerre template from your Canva design. Run this node once with your Canva URL, then disable it and paste the template ID into the Create Variant node. |
| List Airtable records | Airtable | Fetch source records from Airtable for variant generation | Create Template from Canva | Create a variant | ## Step 2: List Airtable records<br>1. Add Airtable credentials (Personal Access Token).<br>2. Select your base and table.<br>3. Optionally use **Filter By Formula** to limit records (e.g. only rows where the output URL is empty).<br>4. Each record needs an **id**; the Update step uses it to write back to the correct row. |
| Create a variant | Layerre | Render one image variant per Airtable record using template overrides | List Airtable records | Update Airtable record | ## Step 3: Create Variant per record<br>Set **Template ID** (from Step 1). Map Airtable fields to your Canva layers using `$json.fields['Field Name']` for field values. Each list item becomes one variant; the record **id** is passed through for the Update step. |
| Update Airtable record | Airtable | Write the generated image URL back into Airtable | Create a variant |  | ## Step 4: Update Airtable with image URL<br>Write the rendered image URL back to the same record. Use the record **id** from the List step (e.g. `$('List Airtable records').item.json.id`) and the image URL from the variant output (e.g. `$json.url`) so each row is updated correctly. |
| Overview | Sticky Note | Documentation/comment node |  |  | ## Airtable → Layerre → Airtable<br><br>Generate images from Airtable records (e.g. leads, events, products) and write the rendered image URL back into your base.<br><br>### How it works<br>1. **Create Template**: One-time – create a Layerre template from your Canva design (run the Create Template node once, then disable it and set the template ID in the Create Variant node).<br>2. **List Airtable records**: Read records from your base and table.<br>3. **Create Variant**: For each record, create one image variant; map Airtable fields to your Canva layers.<br>4. **Update Airtable**: Write the rendered image URL back to the same record.<br><br>### Prerequisites<br>- [Layerre account](https://layerre.com) with API key<br>- Canva design with the layers you want to customize<br>- Airtable base with a table and a column for the output URL<br><br>### Customization<br>- Map your field names in Create Variant (e.g. `$json.fields.Name`, `$json.fields['Image url']`).<br>- In Update, use the record **id** from the List step and the image URL from the variant output (e.g. `$json.url`).<br>- Add a Filter or Schedule trigger to run only for new or updated records.<br><br>### Resources<br>- [Layerre Documentation](https://layerre.com/docs)<br>- [Layerre n8n Node](https://github.com/layerre/n8n-nodes-layerre) |
| Step 1 Instructions | Sticky Note | Documentation/comment node |  |  | ## Step 1: Create Template (one-time)<br><br>Create a Layerre template from your Canva design. Run this node once with your Canva URL, then disable it and paste the template ID into the Create Variant node. |
| Step 2 Instructions | Sticky Note | Documentation/comment node |  |  | ## Step 2: List Airtable records<br><br>1. Add Airtable credentials (Personal Access Token).<br>2. Select your base and table.<br>3. Optionally use **Filter By Formula** to limit records (e.g. only rows where the output URL is empty).<br>4. Each record needs an **id**; the Update step uses it to write back to the correct row. |
| Step 3 Instructions | Sticky Note | Documentation/comment node |  |  | ## Step 3: Create Variant per record<br><br>Set **Template ID** (from Step 1). Map Airtable fields to your Canva layers using `$json.fields['Field Name']` for field values. Each list item becomes one variant; the record **id** is passed through for the Update step. |
| Step 4 Instructions | Sticky Note | Documentation/comment node |  |  | ## Step 4: Update Airtable with image URL<br><br>Write the rendered image URL back to the same record. Use the record **id** from the List step (e.g. `$('List Airtable records').item.json.id`) and the image URL from the variant output (e.g. `$json.url`) so each row is updated correctly. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Generate images from Airtable records with Layerre`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Keep default configuration.
   - Rename it to: `When clicking "Test"`.

3. **Add the first documentation sticky note**
   - Add a Sticky Note node named `Overview`.
   - Paste this content:
     - Airtable → Layerre → Airtable
     - Explain that the workflow creates images from Airtable records and writes generated URLs back
     - Include prerequisites:
       - Layerre account and API key
       - Canva design
       - Airtable base/table with output URL field
     - Include resource links:
       - `https://layerre.com/docs`
       - `https://github.com/layerre/n8n-nodes-layerre`

4. **Add the one-time Layerre template creation node**
   - Node type: `Layerre`
   - Rename it to: `Create Template from Canva`
   - Configure it for template creation from a Canva URL.
   - Enter the Canva design URL in the `canvaUrl` field.
   - Add Layerre credentials:
     - Configure the Layerre API key in n8n credentials
   - After successfully creating the template once, copy the returned template ID.
   - Disable this node for future runs.

5. **Connect the trigger to the template creation node**
   - `When clicking "Test"` → `Create Template from Canva`

6. **Add a sticky note for Step 1**
   - Content should state that this is a one-time step:
     - Run once
     - Create a template from Canva
     - Then disable the node
     - Paste the template ID into the variant node

7. **Add the Airtable record listing node**
   - Node type: `Airtable`
   - Rename it to: `List Airtable records`
   - Set operation to: `Search`
   - Select your Airtable credentials:
     - Use a Personal Access Token with access to the target base/table
   - Select:
     - Base
     - Table
   - Optionally configure search/filter options, especially if you only want rows where the generated URL is empty.

8. **Connect the template creation node to the Airtable list node**
   - `Create Template from Canva` → `List Airtable records`

9. **Add a sticky note for Step 2**
   - Mention:
     - Add Airtable credentials
     - Select base and table
     - Optionally use Filter By Formula
     - Ensure the record `id` is available for updates

10. **Add the Layerre variant creation node**
    - Node type: `Layerre`
    - Rename it to: `Create a variant`
    - Configure:
      - Resource: `variant`
      - Template ID: paste the template ID created in Step 4
    - Add override entries for each Canva layer you want to customize.

11. **Configure the first override in Create a variant**
    - Set the target `layerId` to the text layer in the Layerre template
    - Configure a text override with:
      - `={{ $json.fields.Name }}`
    - This reads the Airtable field `Name`

12. **Configure the second override in Create a variant**
    - Set the target `layerId` to the image layer in the Layerre template
    - Configure an image URL override with:
      - `={{ $json.fields['Image url'] }}`
    - This reads the Airtable field named `Image url`

13. **Review optional Layerre settings**
    - Leave `variantDimensions` empty unless you need specific output dimensions
    - Leave request options empty unless required by Layerre API behavior
    - Confirm credentials are attached

14. **Connect Airtable listing to variant creation**
    - `List Airtable records` → `Create a variant`

15. **Add a sticky note for Step 3**
    - Mention:
      - Set template ID
      - Map Airtable fields with expressions such as `$json.fields['Field Name']`
      - One record becomes one variant
      - Preserve the Airtable record ID for the update step

16. **Prepare Airtable to receive the generated URL**
    - In Airtable, create a field named `Generated url`
   - Ensure it is writable by the token being used.

17. **Add the Airtable update node**
   - Node type: `Airtable`
   - Rename it to: `Update Airtable record`
   - Set operation to: `Update`
   - Select the same base and table as in the list node

18. **Configure the update mapping**
   - Use “define below” style mapping
   - Add field mappings:
     - `id` = `={{ $('List Airtable records').item.json.id }}`
     - `Generated url` = `={{ $json.url }}`
   - Use `id` as the matching field/column

19. **Connect variant creation to Airtable update**
   - `Create a variant` → `Update Airtable record`

20. **Add a sticky note for Step 4**
   - Mention:
     - Write back to the same Airtable record
     - Use the original record ID from the list node
     - Use the generated URL from the variant node output

21. **Test the one-time template creation**
   - Enable `Create Template from Canva`
   - Fill the Canva URL
   - Run the workflow manually
   - Copy the returned template ID

22. **Finalize production configuration**
   - Paste the template ID into `Create a variant`
   - Fill all required `layerId` values in the override list
   - Disable `Create Template from Canva`

23. **Run an end-to-end test**
   - Execute the workflow
   - Confirm that:
     - Airtable records are listed
     - One Layerre variant is created per record
     - The response contains a usable `url`
     - Airtable records are updated in the `Generated url` field

24. **Optional hardening**
   - Add filters before variant creation to skip records with missing `Name` or `Image url`
   - Add a scheduled trigger instead of a manual trigger for automation
   - Add Airtable search filters so only records without generated URLs are processed

### Credential Configuration

#### Layerre credentials
- Create Layerre credentials in n8n using your Layerre API key.
- Attach them to:
  - `Create Template from Canva`
  - `Create a variant`

#### Airtable credentials
- Create Airtable credentials in n8n using a Personal Access Token.
- Ensure access to:
  - The selected base
  - The selected table
  - Read/write permissions for the output field

### Sub-workflow setup
- This workflow does **not** use any Execute Workflow / sub-workflow nodes.
- No child workflow setup is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Layerre account with API key is required before using the Layerre nodes. | https://layerre.com |
| Layerre API and product documentation. | https://layerre.com/docs |
| Layerre n8n node source and reference. | https://github.com/layerre/n8n-nodes-layerre |
| The Canva design must contain the layers you intend to override through Layerre. | Canva design preparation |
| The Airtable table should include a writable output field such as `Generated url`. | Airtable schema requirement |
| Suggested customization: add a Filter or Schedule trigger so only new or updated records are processed. | Operational improvement |

## Additional implementation notes

- The provided workflow contains only a **manual trigger**. It is not fully automated unless you replace or supplement it with a Schedule Trigger, Airtable Trigger, or another event source.
- In the provided JSON, the Layerre nodes are missing actual credentials and required values such as:
  - Canva URL
  - Template ID
  - Override layer IDs
- In the provided JSON, both Airtable nodes are also missing selected base/table values and credentials.
- The workflow assumes that Layerre returns a top-level `url` field after variant creation. If your Layerre node version returns a different payload shape, adjust the update expression accordingly.
- The expression `$('List Airtable records').item.json.id` relies on item linking across node executions. If the workflow is modified with merge/split logic later, validate that paired items still resolve correctly.