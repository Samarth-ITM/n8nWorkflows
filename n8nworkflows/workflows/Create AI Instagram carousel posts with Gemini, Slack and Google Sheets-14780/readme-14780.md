Create AI Instagram carousel posts with Gemini, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/create-ai-instagram-carousel-posts-with-gemini--slack-and-google-sheets-14780


# Create AI Instagram carousel posts with Gemini, Slack and Google Sheets

# Technical Analysis: AI Instagram Carousel Generator

This document provides a comprehensive technical breakdown of the n8n workflow designed to automate the creation of high-fidelity Instagram carousel posts using Google Gemini, Slack, and Google Sheets.

---

### 1. Workflow Overview

The primary purpose of this workflow is to autonomously research, write, design, and deliver a 6-slide Instagram carousel on a daily basis. It ensures content uniqueness by maintaining a history of previous topics in a Google Sheet and leverages a two-step AI process: one for copywriting (textual logic) and another for visual generation (image creation).

The logic is divided into three primary functional blocks:
- **1.1 Data & Context:** Handles scheduling and deduplication by retrieving previously used topics.
- **1.2 The Creative Engine:** Uses LLMs to generate a narrative structure and a JavaScript bridge to transform that structure into precise visual prompts.
- **1.3 Rendering & Delivery:** Generates the actual images, packages them for Slack, and logs the final topic back to the database.

---

### 2. Block-by-Block Analysis

#### 2.1 Data & Context
**Overview:** This block triggers the process and ensures the AI does not repeat topics already covered in previous posts.

- **Nodes Involved:** `Daily at Noon`, `Get row(s) in sheet`, `Code in JavaScript`.
- **Node Details:**
    - **Daily at Noon (Schedule Trigger):** Triggers the workflow every day at 12:00 PM.
    - **Get row(s) in sheet (Google Sheets):** Fetches all existing rows from a specific spreadsheet.
        - *Configuration:* Targeted at a specific Document ID and Sheet Name.
        - *Failure Risk:* Authentication errors or missing sheet permissions.
    - **Code in JavaScript (Code):** Aggregates the "Topic" column from all retrieved rows into a single comma-separated string.
        - *Key Expression:* Maps `i.json.Topic` and joins them.
        - *Output:* A single object `{ history: "Topic A, Topic B..." }`.

#### 2.2 The Creative Engine
**Overview:** This block transforms a high-level niche requirement into a structured 6-slide script and corresponding image design specifications.

- **Nodes Involved:** `Generate Carousel Content`, `Build Slide Image Prompts`.
- **Node Details:**
    - **Generate Carousel Content (Google Gemini):** Acts as the lead strategist.
        - *Configuration:* Model `gemini-3-flash-preview`, Temperature `0.9`.
        - *Logic:* Uses a complex prompt to generate a JSON object containing a topic, a 6-slide array (Cover $\rightarrow$ Fact $\rightarrow$ Reality $\rightarrow$ Why it Matters $\rightarrow$ Pro Tip $\rightarrow$ CTA), and a caption.
        - *Dynamic Input:* Injects `{{ $json.history }}` to avoid duplicates.
        - *Edge Case:* AI might return markdown code fences (```json) which can break standard JSON parsing.
    - **Build Slide Image Prompts (Code):** A robust JavaScript bridge that parses the AI's response and maps it to visual prompts.
        - *Logic:* It includes a "regex cleaner" to remove markdown fences. It defines a `brandBase` (Black background, white text, neon accents) and an `avatar` description.
        - *Output:* Transforms the 1 JSON object into 6 separate items, each containing a specific `imagePrompt` tailored to the slide type (Cover, CTA, or Fact).

#### 2.3 Rendering & Delivery
**Overview:** Converts textual prompts into images and delivers them to the team for review.

- **Nodes Involved:** `Generate Slide Images`, `Prepare Slack Package`, `Upload Slides to Slack`, `Append row in sheet`.
- **Node Details:**
    - **Generate Slide Images (Google Gemini):** Generates the visual assets.
        - *Configuration:* Model `gemini-3-pro-image-preview` (Image resource).
        - *Input:* The `imagePrompt` generated in the previous step.
    - **Prepare Slack Package (Code):** Formats the metadata for Slack.
        - *Logic:* Creates a friendly comment for the first slide (including the caption) and simple labels for subsequent slides.
        - *Output:* Assigns a `fileName` (e.g., `carousel_slide_01.png`).
    - **Upload Slides to Slack (Slack):** Uploads the binary image files to a designated channel.
        - *Configuration:* OAuth2 authentication; uses `initialComment` to post the caption.
    - **Append row in sheet (Google Sheets):** Closes the loop.
        - *Logic:* Appends the new `topic` to the Google Sheet so it is excluded from tomorrow's generation.
        - *Execution:* Set to `executeOnce: true` to prevent 6 duplicate entries (since there are 6 slides).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily at Noon | Schedule Trigger | Workflow Entry | - | Get row(s) in sheet | - |
| Get row(s) in sheet | Google Sheets | Retrieve History | Daily at Noon | Code in JavaScript | 🔍 01: DATA & CONTEXT |
| Code in JavaScript | Code | Format History String | Get row(s) in sheet | Generate Carousel Content | 🔍 01: DATA & CONTEXT |
| Generate Carousel Content | Google Gemini | Copywriting/Ideation | Code in JavaScript | Build Slide Image Prompts | ✍️ 02: THE CREATIVE ENGINE |
| Build Slide Image Prompts | Code | Design Logic/Prompting | Generate Carousel Content | Generate Slide Images | ✍️ 02: THE CREATIVE ENGINE / JSON Parsing Warning |
| Generate Slide Images | Google Gemini | Image Generation | Build Slide Image Prompts | Prepare Slack Package | 🚀 03: RENDERING & DELIVERY |
| Prepare Slack Package | Code | Formatting Delivery | Generate Slide Images | Upload Slides to Slack | 🚀 03: RENDERING & DELIVERY |
| Upload Slides to Slack | Slack | Delivery to Team | Prepare Slack Package | Append row in sheet | 🚀 03: RENDERING & DELIVERY |
| Append row in sheet | Google Sheets | Update History | Upload Slides to Slack | - | 🚀 03: RENDERING & DELIVERY |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Environment Setup
1. Create a Google Sheet with a single column header named `Topic`.
2. Configure credentials for:
    - **Google Sheets** (Service Account or OAuth2).
    - **Google Gemini** (API Key).
    - **Slack** (OAuth2).

#### Step 2: Trigger and Memory
1. Add a **Schedule Trigger** set to daily at 12:00.
2. Connect a **Google Sheets** node (Get Many Rows) to read the "Topic" column.
3. Add a **Code Node** to join all "Topic" values into a single string variable called `history`.

#### Step 3: AI Content Generation
1. Add a **Google Gemini** node. Configure it with the `gemini-3-flash-preview` model.
2. Paste the prompt instructing it to output a 6-slide JSON structure. Use the `history` variable in the "Negative Context" section of the prompt.
3. Enable **JSON Output** in the node settings.

#### Step 4: Design Bridge (The Logic)
1. Add a **Code Node** with the JavaScript provided in the JSON.
2. Ensure the code contains the `brandBase` and `avatar` strings to maintain visual consistency.
3. This node must split the single AI response into 6 individual items.

#### Step 5: Image Production
1. Add a **Google Gemini** node set to the `image` resource using the `gemini-3-pro-image-preview` model.
2. Map the prompt input to `{{ $json.imagePrompt }}`.

#### Step 6: Delivery and Logging
1. Add a **Code Node** to generate `slackComment` and `fileName` for each item.
2. Add a **Slack Node** (Upload File) using the binary data from Gemini. Map the `initialComment` and `fileName`.
3. Add a **Google Sheets Node** (Append Row) to save the `topic` back to the sheet. **Crucial:** Set the "Execute Once" toggle to `true`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Visual Customization** | Modify `brandBase` and `avatar` variables in the "Build Slide Image Prompts" node to change branding. |
| **Robustness** | The regex in the Code node is essential to prevent failures when Gemini returns markdown fences. |
| **Scale** | To change posting frequency, modify the "Daily at Noon" trigger. |
| **Required Header** | The Google Sheet MUST have a column exactly named `Topic`. |