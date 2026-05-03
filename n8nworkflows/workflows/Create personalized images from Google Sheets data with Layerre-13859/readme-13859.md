Create personalized images from Google Sheets data with Layerre

https://n8nworkflows.xyz/workflows/create-personalized-images-from-google-sheets-data-with-layerre-13859


# Create personalized images from Google Sheets data with Layerre

# Workflow Reference: Create Personalized Images from Google Sheets Data with Layerre

This document provides a technical breakdown of the n8n workflow designed to automate mass image personalization. By integrating Google Sheets with Layerre (a Canva-compatible automation engine), this workflow transforms rows of spreadsheet data into unique, high-quality images.

---

### 1. Workflow Overview
The workflow is designed to bridge the gap between structured data (spreadsheets) and visual content (Canva designs). It functions by converting a Canva URL into a reusable template and then iterating through data rows to generate custom image variants.

**Logical Blocks:**
- **1.1 Input & Initialization:** Manual trigger to start the process.
- **1.2 Template Preparation:** Converting a Canva design into a Layerre template.
- **1.3 Data Retrieval:** Fetching customization values (text and image URLs) from Google Sheets.
- **1.4 Image Generation:** Mapping spreadsheet data to specific Canva layers to produce final image assets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Initialization
- **Overview:** The entry point for the workflow, allowing for manual testing and execution.
- **Nodes Involved:** `When clicking "Test workflow"`
- **Node Details:**
    - **Type:** Manual Trigger
    - **Configuration:** No specific parameters required.
    - **Output Connection:** Connects directly to the Template Creation block.

#### 2.2 Template Preparation
- **Overview:** Converts a static Canva design into a dynamic Layerre template. This step is typically performed once per design.
- **Nodes Involved:** `Create Template from Canva`
- **Node Details:**
    - **Type:** Layerre Node
    - **Configuration:** Requires a Canva URL (set to "Anyone with the link can view" in Canva).
    - **Credentials:** Layerre API Key.
    - **Key Expressions:** The output `id` of this node is used later to identify which template to use for variants.
    - **Edge Cases:** Invalid Canva URLs or restricted permissions on the Canva side will cause a failure.

#### 2.3 Data Retrieval
- **Overview:** Pulls the personalization data from a designated Google Sheet. Each row in the sheet represents a unique image variant.
- **Nodes Involved:** `Get Rows from Google Sheet`
- **Node Details:**
    - **Type:** Google Sheets Node
    - **Configuration:** Uses the `Get Row(s)` operation. It points to a specific Document ID and Sheet Name.
    - **Credentials:** Google OAuth2 or Service Account.
    - **Inputs:** Triggered after the template is confirmed.
    - **Outputs:** Returns a list of objects, where each object contains the keys (e.g., `Text`, `Image`) found in the sheet headers.

#### 2.4 Image Generation
- **Overview:** The core logic where data meets design. It loops through the spreadsheet rows and instructs Layerre to swap layer content.
- **Nodes Involved:** `Create Variant with Custom Data`
- **Node Details:**
    - **Type:** Layerre Node (Resource: Variant)
    - **Configuration:** 
        - **Template ID:** Mapped dynamically via expression `{{ $('Create Template from Canva').item.json.id }}`.
        - **Overrides:** Maps Layer IDs to spreadsheet columns.
    - **Key Expressions:** 
        - `{{ $json.Text }}` → Mapped to the text layer ID.
        - `{{ $json.Image }}` → Mapped to the image layer ID (as an image URL).
    - **Edge Cases:** If Layer IDs in Layerre do not match the IDs specified in the node configuration, the overrides will fail to apply.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking "Test workflow" | Manual Trigger | Entry point | None | Create Template from Canva | |
| Create Template from Canva | Layerre | Template Setup | When clicking "Test workflow" | Get Rows from Google Sheet | Step 1: Create Template. Share Canva design and paste URL. |
| Get Rows from Google Sheet | Google Sheets | Data Fetching | Create Template from Canva | Create Variant with Custom Data | Step 2: Get Rows. Each data row becomes one variant. |
| Create Variant with Custom Data | Layerre | Image Creation | Get Rows from Google Sheet | None | Step 3: Generate Variants. Map sheet columns to Canva layers. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** 
    - Ensure you have a **Layerre API Key** and **Google Sheets Credentials** (OAuth2 or Service Account) configured in n8n.
    - Prepare a Canva design and set sharing to "Anyone with the link can view."

2.  **Trigger:**
    - Add a **Manual Trigger** node.

3.  **Template Node:**
    - Add a **Layerre** node.
    - Set Action to `Create Template` (Canva). 
    - Paste your **Canva URL** into the parameter field.
    - Connect the Manual Trigger to this node.

4.  **Data Node:**
    - Add a **Google Sheets** node.
    - Set Action to `Get Row(s)`.
    - Select your spreadsheet and the specific sheet (e.g., "Sheet1").
    - Connect the `Create Template` node to this node.

5.  **Variant Generation Node:**
    - Add another **Layerre** node.
    - Set Resource to `Variant` and Action to `Create`.
    - In the **Template ID** field, use the expression: `{{ $('Create Template from Canva').item.json.id }}`.
    - Under **Overrides**, add an item for Text:
        - Layer ID: [Your Canva Text Layer ID]
        - Value: `{{ $json.Text }}`
    - Add an item for Image:
        - Layer ID: [Your Canva Image Layer ID]
        - Value: `{{ $json.Image }}`
    - Connect the `Google Sheets` node to this node.

6.  **Finalization:**
    - Execute the workflow. The output of the final node will contain the URLs for the newly generated, personalized images.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Layerre Official Documentation | [https://layerre.com/docs](https://layerre.com/docs) |
| Layerre n8n Node Github | [https://github.com/layerre/n8n-nodes-layerre](https://github.com/layerre/n8n-nodes-layerre) |
| Public Google Sheets Example | [Google Sheets Link](https://docs.google.com/spreadsheets/d/1x8X_qZaoXkrIKGVlhCC25du8ht42UKinRPtmjThpTSE/edit?usp=sharing) |
| Scaling Tip | You can deactivate the "Create Template" node once the template ID is generated and hardcode the ID in Step 3 to save execution time. |