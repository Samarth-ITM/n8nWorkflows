Generate Shopify product images using Anthropic and deAPI

https://n8nworkflows.xyz/workflows/generate-shopify-product-images-using-anthropic-and-deapi-13704


# Generate Shopify product images using Anthropic and deAPI

# Workflow Reference: E-commerce Product Visual Generator

This document provides a technical breakdown of the n8n workflow designed to automate the generation of professional product imagery for Shopify stores using Anthropic's Claude and deAPI's image generation suite.

---

### 1. Workflow Overview

The **E-commerce Product Visual Generator** is an automated pipeline that monitors a Shopify store for new products and uses AI to generate high-quality marketing visuals. It solves the problem of manual photo editing by creating both a stylized "hero" image and a clean, transparent background PNG for every new listing.

#### Logical Blocks:
*   **1.1 Input Reception:** Listens for new product creation events in Shopify.
*   **1.2 Data Normalization:** Extracts and formats relevant product metadata (title, category, tags).
*   **1.3 AI Prompt Engineering:** Uses an AI Agent with a specialized tool to craft a high-performing image generation prompt.
*   **1.4 Image Synthesis:** Generates a professional product photo based on the AI prompt.
*   **1.5 Post-Processing:** Removes the background from the generated image to create a secondary asset.
*   **1.6 Store Update:** Uploads both the original and the transparent images back to the specific Shopify product.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** The entry point of the workflow, triggered by external store activity.
*   **Nodes Involved:** `Shopify Trigger`.
*   **Node Details:**
    *   **Type:** Shopify Trigger
    *   **Configuration:** Listens for the `products/create` topic.
    *   **Authentication:** Requires an Access Token with `read_products` scope.
    *   **Edge Cases:** Webhook failures if the Shopify URL is not HTTPS or if the app scopes are insufficient.

#### 2.2 Data Normalization
*   **Overview:** Isolates the necessary text fields from the Shopify JSON payload to provide context for the AI.
*   **Nodes Involved:** `Edit Fields`.
*   **Node Details:**
    *   **Type:** Set / Edit Fields (v3.4)
    *   **Key Expressions:** 
        *   `category`: `{{ $json.category.name }}`
        *   `tags`: `{{ $json.tags }}`
    *   **Function:** Passes through `title` and `body_html` while specifically mapping the category name and tags for clarity.

#### 2.3 AI Prompt Engineering
*   **Overview:** A sophisticated AI agent cluster that transforms raw product data into a professional photography prompt.
*   **Nodes Involved:** `AI Agent`, `Anthropic Chat Model`, `Image prompt booster in deAPI`, `Structured Output Parser`.
*   **Node Details:**
    *   **AI Agent:** Uses "Define" prompt type with a System Message acting as a "Product Photography Expert."
    *   **Anthropic Chat Model:** Configured for `claude-sonnet-4-5-20250929`.
    *   **deAPI Tool:** Invokes the `prompt` resource to "boost" the text for better image generation results.
    *   **Structured Output Parser:** Ensures the agent returns a valid JSON object: `{"boosted_prompt": "..."}`.
    *   **Edge Cases:** API limits on Anthropic or deAPI; halluncinations if product descriptions are empty.

#### 2.4 Image Synthesis & Post-Processing
*   **Overview:** Turns text into pixels and performs background removal.
*   **Nodes Involved:** `deAPI Generate Image`, `deAPI Remove Background`.
*   **Node Details:**
    *   **deAPI Generate Image:** Takes the `boosted_prompt` from the AI Agent and outputs a image URL.
    *   **deAPI Remove Background:** Takes the result of the previous node and processes it to create a transparent PNG.
    *   **Edge Cases:** Content moderation filters on image generation; processing timeouts for high-res images.

#### 2.5 Store Update
*   **Overview:** Syncs the new visuals back to the original Shopify product listing.
*   **Nodes Involved:** `Shopify Update Product`.
*   **Node Details:**
    *   **Type:** Shopify Node (Action: Update Product).
    *   **Configuration:** Target `productId` is derived from the initial Trigger.
    *   **Image Positions:** 
        *   **Position 1:** Styled hero image (`result_url` from deAPI Generate).
        *   **Position 2:** Transparent PNG (`result_url` from deAPI Remove Background).
    *   **Authentication:** Requires `write_products` and `write_files` scopes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Shopify Trigger** | Shopify Trigger | Webhook Entry | None | Edit Fields | 1. Shopify Product Trigger. Listens for `products/create` webhook events. |
| **Edit Fields** | Set | Data Extraction | Shopify Trigger | AI Agent | 2. Extract Product Data. Prepares category and tags for the AI Agent. |
| **AI Agent** | AI Agent | Logic Controller | Edit Fields | deAPI Generate Image | 3. AI-Powered Prompt Generation. Analyzes product data to create optimized prompts. |
| **Anthropic Chat Model** | Anthropic Model | Reasoning Engine | AI Agent | AI Agent | Part of 3. AI-Powered Prompt Generation. |
| **Image prompt booster** | deAPI Tool | Prompt Enhancer | AI Agent | AI Agent | Uses the deAPI Image Prompt Booster tool to create optimized prompts. |
| **Structured Output Parser**| Output Parser | Format Validator | AI Agent | AI Agent | Ensures consistent JSON output. |
| **deAPI Generate Image** | deAPI | Image Creation | AI Agent | deAPI Remove Background | 4. Generate Product Image. Generates a professional product image. |
| **deAPI Remove Background** | deAPI | Image Editing | deAPI Generate Image | Shopify Update Product | 5. Remove Background. Creates a transparent PNG version. |
| **Shopify Update Product** | Shopify | Data Sync | deAPI Remove Background | None | 6. Update Product Images. Uploads both images to Shopify. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Shopify Connection:** 
    *   Create a **Shopify Trigger** node. 
    *   Set Topic to `products/create`. 
    *   Authenticate using an Access Token with `read_products` scope.
2.  **Data Extraction:** 
    *   Add an **Edit Fields** node. 
    *   Keep `title` and `body_html`.
    *   Add two string assignments: `category` (map to `$json.category.name`) and `tags` (map to `$json.tags`).
3.  **AI Orchestration:**
    *   Add an **AI Agent** node. Set to "Define" prompt.
    *   Connect an **Anthropic Chat Model** node (select Claude 3.5 or 4.5 Sonnet).
    *   Connect a **deAPI Tool** node. Set resource to `prompt` and map the prompt parameter to `$fromAI`.
    *   Connect a **Structured Output Parser**. Define a schema with a single string property: `boosted_prompt`.
4.  **Generation Pipeline:**
    *   Add a **deAPI** node. Operation: `generate`. Set Prompt to `{{ $json.output.boosted_prompt }}`.
    *   Add another **deAPI** node. Operation: `removeBackground`. This automatically uses the output of the generation node.
5.  **Closing the Loop:**
    *   Add a **Shopify** node. Operation: `Update`.
    *   Map the Product ID from the Trigger node: `{{ $('Shopify Trigger').item.json.id }}`.
    *   In the "Images" field, add two items:
        *   URL 1: `{{ $('deAPI Generate Image').item.json.result_url }}`
        *   URL 2: `{{ $json.result_url }}` (from the Background Remover).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **deAPI Documentation** | [https://docs.deapi.ai](https://docs.deapi.ai) |
| **Shopify Trigger Guide** | [n8n Shopify Trigger Docs](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.shopifytrigger) |
| **Community Support** | [n8n Forum](https://community.n8n.io/) |
| **Support Discord** | [n8n Discord](https://discord.gg/n8n) |