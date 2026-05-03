List properties instantly with UploadToURL, OpenAI Vision, WordPress and Airtable

https://n8nworkflows.xyz/workflows/list-properties-instantly-with-uploadtourl--openai-vision--wordpress-and-airtable-13635


# List properties instantly with UploadToURL, OpenAI Vision, WordPress and Airtable

# Workflow Reference: Instant Property Listing with UploadToURL, OpenAI Vision, WordPress, and Airtable

## 1. Workflow Overview

This workflow automates the transition from a raw property photo to a professional real estate listing. It is designed for real estate agents who need an "upload-and-forget" pipeline. The workflow receives a photo (via binary upload or URL) and address metadata, hosts the image on a CDN, uses AI to generate an MLS-compliant description based on visual analysis, and simultaneously publishes the data to WordPress (as a draft) and Airtable (as a record).

The logic is organized into three main functional blocks:
- **1.1 Intake & Analysis:** Validates input, uploads the image to a public CDN, and performs GPT-4o Vision analysis.
- **1.2 Parallel Publishing:** Simultaneously creates a WordPress draft post with Gutenberg blocks and an Airtable MLS record.
- **1.3 Synchronization & Notification:** Merges results from both platforms and notifies the agent via Telegram with direct links.

---

## 2. Block-by-Block Analysis

### 1.1 Intake & Analysis
**Overview:** This block handles the entry point, data validation, and the core AI "vision" processing.
- **Nodes Involved:** Webhook, Validate Payload, Has Remote URL?, Upload to URL (Remote/Binary), Extract CDN URL, GPT-4o Vision - MLS Analysis, Parse Vision & Build Payloads.
- **Node Details:**
    - **Webhook:** Entry point for property photos. Configured for `POST` requests.
    - **Validate Payload (Code):** Ensures `listingId` and `address` are present. Normalizes prices and generates a structured filename for the CDN.
    - **Has Remote URL? (If):** Checks if the input is a file URL or a binary upload to route to the correct upload node.
    - **Upload to URL (Community Node):** Uses `n8n-nodes-uploadtourl` to host the image and get a public CDN link. Requires API credentials.
    - **Extract CDN URL (Code):** Normalizes the response from the CDN provider to ensure a consistent HTTPS URL.
    - **GPT-4o Vision (OpenAI):** Analyzes the image. Uses a system prompt to act as an MLS specialist. Returns JSON containing room type, condition score (1-10), feature tags, and a professional description.
    - **Parse Vision (Code):** Extracts the AI's JSON and pre-builds the HTML/Gutenberg blocks for WordPress and the field mapping for Airtable.

### 1.2 Parallel Publishing
**Overview:** This block executes two external integrations at once to minimize processing time.
- **Nodes Involved:** WordPress - Create Draft Post, Airtable - Create MLS Record.
- **Node Details:**
    - **WordPress (HTTP Request):** Uses the WP REST API to create a draft. It pushes the pre-built `wpContent` and maps property metadata (listing ID, price, agent) into custom fields (post meta).
    - **Airtable (Airtable Node):** Creates a record in the specified Base/Table. Uses the "Auto Map" feature to inject the `airtableRecord` object created in the previous block.

### 1.3 Synchronization & Notification
**Overview:** Collects IDs from the published platforms and alerts the user.
- **Nodes Involved:** Merge Platform Results, Build Unified Response, Telegram - Agent Confirmation, Respond to Webhook.
- **Node Details:**
    - **Merge (Merge):** Set to `combine` mode. It waits until both the WordPress and Airtable nodes successfully complete before continuing.
    - **Build Unified Response (Code):** Constructs the final URLs for the WordPress editor and the Airtable record using environment variables.
    - **Telegram (Telegram):** Sends a rich-text Markdown message to the agent featuring the property summary, AI analysis results, and direct links to the generated listings.
    - **Respond to Webhook:** Returns a `201 Created` status to the original requester with the full structured payload.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - Receive Property Photo | Webhook | Entry Point | (None) | Validate Payload | 1 — Upload & Vision analysis |
| Validate Payload | Code | Data Sanitization | Webhook | Has Remote URL? | 1 — Upload & Vision analysis |
| Has Remote URL? | If | Route Logic | Validate Payload | Upload to URL (Remote/Binary) | 1 — Upload & Vision analysis |
| Upload to URL - Remote | UploadToURL | Image Hosting | Has Remote URL? | Extract CDN URL | 1 — Upload & Vision analysis |
| Upload to URL - Binary | UploadToURL | Image Hosting | Has Remote URL? | Extract CDN URL | 1 — Upload & Vision analysis |
| Extract CDN URL | Code | URL Normalization | Upload to URL nodes | GPT-4o Vision | 1 — Upload & Vision analysis |
| GPT-4o Vision - MLS Analysis | OpenAI | Image Intelligence | Extract CDN URL | Parse Vision & Build Payloads | 1 — Upload & Vision analysis |
| Parse Vision & Build Payloads | Code | Payload Prep | GPT-4o Vision | WordPress, Airtable | 1 — Upload & Vision analysis |
| WordPress - Create Draft Post | HTTP Request | Platform Publish | Parse Vision | Merge Platform Results | 2 — Parallel publish |
| Airtable - Create MLS Record | Airtable | Platform Publish | Parse Vision | Merge Platform Results | 2 — Parallel publish |
| Merge Platform Results | Merge | Sync Gate | WordPress, Airtable | Build Unified Response | 3 — Merge & notify |
| Build Unified Response | Code | Result Assembly | Merge | Telegram | 3 — Merge & notify |
| Telegram - Agent Confirmation | Telegram | Notification | Build Unified Response | Respond to Webhook | 3 — Merge & notify |
| Respond to Webhook | Respond to Webhook | API Completion | Telegram | (None) | 3 — Merge & notify |

---

## 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    - Install the `n8n-nodes-uploadtourl` community node.
    - Define Global Variables/Variables: `WP_BASE_URL`, `AIRTABLE_BASE_ID`, `AIRTABLE_TABLE_NAME`, `TELEGRAM_CHAT_ID`.
2.  **Intake Configuration:**
    - Create a **Webhook** node (POST) with a path like `/property-photo-publish`.
    - Add a **Code** node to validate `listingId` and `address`. Use regex to sanitize the address for filenames.
    - Add an **If** node to check for `fileUrl`.
3.  **Hosting & AI:**
    - Connect **UploadToURL** nodes (one for binary, one for remote URL).
    - Add a **Code** node to extract the direct link from the upload response.
    - Add an **OpenAI** node (GPT-4o Vision). Use a prompt that enforces a JSON response including `roomType`, `mlsDescription`, and `conditionScore`.
    - Add a **Code** node to transform the AI JSON into WordPress HTML (Gutenberg comments) and an Airtable-compatible object.
4.  **Publishing:**
    - Create an **HTTP Request** node for WordPress. Set Method to `POST`, URL to `{{$vars.WP_BASE_URL}}/wp-json/wp/v2/posts`. Use Basic Auth or Application Passwords.
    - Create an **Airtable** node. Set Operation to `Create`. Map the columns using the expression `={{ $json.airtableRecord }}`.
5.  **Closing the Loop:**
    - Connect both WordPress and Airtable nodes to a **Merge** node (Mode: Combine).
    - Add a **Code** node to generate the `wpEditUrl` and `airtableUrl`.
    - Add a **Telegram** node using Markdown to format the summary and links.
    - Finish with a **Respond to Webhook** node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires Community Node: n8n-nodes-uploadtourl | [npm link](https://www.npmjs.com/package/n8n-nodes-uploadtourl) |
| Uses GPT-4o Vision for visual analysis | Requires OpenAI API Key with Vision access |
| Designed for WordPress REST API | Uses Gutenberg block markers for rich formatting |
| Variables required: WP_BASE_URL, AIRTABLE_BASE_ID, TELEGRAM_CHAT_ID | Setup these in n8n Settings > Variables |