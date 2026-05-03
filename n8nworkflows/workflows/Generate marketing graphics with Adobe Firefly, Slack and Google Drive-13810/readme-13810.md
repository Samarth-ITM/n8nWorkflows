Generate marketing graphics with Adobe Firefly, Slack and Google Drive

https://n8nworkflows.xyz/workflows/generate-marketing-graphics-with-adobe-firefly--slack-and-google-drive-13810


# Generate marketing graphics with Adobe Firefly, Slack and Google Drive

# Reference Document: n8n Adobe Firefly — Marketing Graphics Generator

## 1. Workflow Overview
This workflow automates the creation of branded marketing assets using the **Adobe Firefly v3 API**. It transforms a simple text prompt into four production-ready graphic variations optimized for specific social media or advertising platforms. By integrating **Slack** and **Google Drive**, it ensures the creative team is instantly notified and a permanent record of the asset manifest is stored.

The logic is organized into four main functional blocks:
*   **1.1 Input & Sanitization:** Receiving the campaign brief via webhook and setting credentials.
*   **1.2 Prompt Engineering:** Dynamically generating 4 prompt variations based on platform dimensions and brand style.
*   **1.3 Image Generation:** Executing the API call to Adobe Firefly.
*   **1.4 Asset Delivery:** Parsing results, saving a JSON manifest to Google Drive, and notifying Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Input & Sanitization
**Overview:** Captures the incoming request data and initializes the environment variables required for Adobe and external integrations.
*   **Nodes Involved:** `Receive Campaign Brief`, `Set Adobe Credentials`.
*   **Node Details:**
    *   **Receive Campaign Brief (Webhook):** Listens for a `POST` request. It is configured to respond via a specific node later in the flow to allow for asynchronous processing completion.
    *   **Set Adobe Credentials (Set):** Normalizes input fields (brand, prompt, style, platform, etc.) with default values. It also acts as a secure vault for `ADOBE_CLIENT_ID`, `ADOBE_CLIENT_SECRET`, `ADOBE_ORG_ID`, and other integration tokens.

### 2.2 Prompt Engineering
**Overview:** Translates the user's high-level idea into technically optimized instructions for the AI model.
*   **Nodes Involved:** `Build Firefly Prompts`, `Wait For Data`.
*   **Node Details:**
    *   **Build Firefly Prompts (Code):** A JavaScript-based node that calculates image dimensions based on the target platform (e.g., Instagram 1080x1080, LinkedIn 1200x627). It creates four variations: a hero image, an environmental wide shot, a close-up detail, and an abstract creative direction.
    *   **Wait For Data (Wait):** Pauses execution for 25 seconds. This is often used to manage rate limits or ensure API availability during heavy processing.

### 2.3 Image Generation
**Overview:** Communicates with Adobe’s commercially safe generative AI.
*   **Nodes Involved:** `Call Firefly Generate API`.
*   **Node Details:**
    *   **Call Firefly Generate API (HTTP Request):** Sends a `POST` request to `https://firefly-api.adobe.io/v3/images/generate`. 
    *   **Configuration:** Uses custom headers for Adobe’s specific auth requirements (`x-api-key`, `x-gw-ims-org-id`, and Bearer token). It sends the generated JSON payload from the previous step.
    *   **Edge Cases:** Timeout is set to 60 seconds to accommodate high-resolution image generation time.

### 2.4 Asset Delivery
**Overview:** Processes the API output and distributes it to the team and storage.
*   **Nodes Involved:** `Parse Asset Results`, `Save Manifest to Drive`, `Notify Slack Team`, `Return Asset Package`.
*   **Node Details:**
    *   **Parse Asset Results (Code):** Extracts image URLs from the Adobe response. It includes a fallback mechanism to generate demo URLs if the API returns no results, preventing workflow crashes.
    *   **Save Manifest to Drive (HTTP Request):** Uses a multipart/related POST to upload a JSON file containing the job metadata and image links to a specific Google Drive folder.
    *   **Notify Slack Team (HTTP Request):** Sends a rich-formatted block message to a Slack Webhook, displaying brand info, prompts, and clickable links to the four image variations.
    *   **Return Asset Package (Respond to Webhook):** Closes the initial HTTP request from the user with a 200 OK status and the full JSON metadata of the generated images.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Campaign Brief | Webhook | Entry point for campaign data | (None) | Set Adobe Credentials | # n8n Adobe Firefly AI Automation... |
| Set Adobe Credentials | Set | Configures API keys & defaults | Receive Campaign Brief | Build Firefly Prompts | # n8n Adobe Firefly AI Automation... |
| Build Firefly Prompts | Code | Generates 4 prompt variations | Set Adobe Credentials | Wait For Data | ## 1. Builds 4 optimized Firefly prompt variations |
| Wait For Data | Wait | Delays execution | Build Firefly Prompts | Call Firefly Generate API | ## 2. Parses the response, extracts all image URLs... |
| Call Firefly Generate API | HTTP Request | Firefly v3 API Call | Wait For Data | Parse Asset Results | ## 2. Parses the response, extracts all image URLs... |
| Parse Asset Results | Code | Formats API response | Call Firefly Generate API | Save Manifest to Drive, Notify Slack Team | ## 2. Parses the response, extracts all image URLs... |
| Save Manifest to Drive | HTTP Request | Google Drive Backup | Parse Asset Results | Return Asset Package | ## 3. Notification |
| Notify Slack Team | HTTP Request | Slack Alerts | Parse Asset Results | Return Asset Package | ## 3. Notification |
| Return Asset Package | Respond to Webhook | Final API Response | Save Manifest, Notify Slack | (None) | ## 3. Notification |

---

## 4. Reproducing the Workflow from Scratch

1.  **Webhook Setup:** Create a Webhook node set to `POST` on path `firefly-graphics`. Set "Response Mode" to "When Last Node Finishes" (or Response Node).
2.  **Configuration:** Add a **Set** node. Define strings for `brand`, `prompt`, `style`, `platform`, `campaign`, `mood`, and your Adobe/Slack/Google credentials/IDs.
3.  **Logic Logic (JS):** Add a **Code** node. Create a mapping for `platformSizes`. Write logic to concatenate the `prompt` with `brand`, `mood`, and `style`. Create an array of 4 variants and construct the `fireflyPayload` object.
4.  **Interval:** Add a **Wait** node set to 25 seconds.
5.  **Adobe API Integration:** 
    *   Add an **HTTP Request** node. 
    *   URL: `https://firefly-api.adobe.io/v3/images/generate`
    *   Method: `POST`. 
    *   Headers: `x-api-key`, `Authorization` (Bearer), `x-gw-ims-org-id`.
    *   Body: Use the expression `{{ $json.fireflyPayload }}`.
6.  **Data Extraction:** Add another **Code** node to iterate through the `outputs` array from the Adobe response and extract the `image.url` into a clean manifest object.
7.  **Storage:** 
    *   Add an **HTTP Request** node for Google Drive. 
    *   Use URL: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart`.
    *   Set Content-Type to `multipart/related; boundary=ff_boundary`.
8.  **Communication:** Add an **HTTP Request** node for Slack. Configure the `blocks` JSON to include fields for Campaign, Brand, and the list of generated URLs.
9.  **Response:** Add a **Respond to Webhook** node to return the `manifestJson` to the original caller.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Adobe Developer Console** | Setup required at [developer.adobe.com](https://developer.adobe.com) |
| **Adobe Firefly Safety** | Trained exclusively on licensed Adobe Stock content for commercial safety. |
| **Supported Platforms** | Instagram, Facebook, LinkedIn, Google Ads, Email, Twitter, YouTube. |
| **Prompt Engineering** | Automatically adds negative prompts to prevent low-quality outputs (e.g., "blurry, distorted faces"). |