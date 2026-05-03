Create AI coloring book pages from Google Sheets and save to Google Drive with Stable Diffusion

https://n8nworkflows.xyz/workflows/create-ai-coloring-book-pages-from-google-sheets-and-save-to-google-drive-with-stable-diffusion-15034


# Create AI coloring book pages from Google Sheets and save to Google Drive with Stable Diffusion

# Workflow Documentation: AI Coloring Book Page Generator

### 1. Workflow Overview
The purpose of this workflow is to automate the generation of high-quality, printable black-and-white coloring book pages based on themes stored in a Google Sheet. It leverages Stable Diffusion (via Hugging Face) to create multiple variations of an image for a given theme, saves these images to Google Drive, and logs structured marketing metadata (titles, descriptions, keywords) back into a Google Sheet for commercial use.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Triggering the workflow and retrieving settings and themes from Google Sheets.
- **1.2 Task Filtering & Batching:** Identifying "pending" tasks and managing the flow of data to prevent API overloading.
- **1.3 Prompt Engineering & Variation Generation:** Constructing a standardized AI prompt and creating multiple unique variations for each theme.
- **1.4 AI Generation & Storage:** Generating the actual images via Hugging Face and uploading the binary files to Google Drive.
- **1.5 Metadata Processing & Logging:** Generating SEO-friendly text for the images and recording the final results in a tracking sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
**Overview:** This block initializes the workflow and gathers the necessary "global" settings and the specific themes to be processed.
- **Nodes Involved:** `Run Manually`, `Run Weekly Coloring Generation`, `Coloring Page Config`, `Get Themes`, `Merge Settings and Themes`.
- **Node Details:**
    - **Run Manually / Run Weekly Coloring Generation:** Entry points for the workflow. One allows for testing; the other schedules periodic generation.
    - **Coloring Page Config (Google Sheets):** Fetches global parameters (e.g., `image_count`) from the "Settings" tab.
    - **Get Themes (Google Sheets):** Fetches the list of coloring themes from the "ColoringThemes" tab.
    - **Merge Settings and Themes (Merge):** Combines the configuration settings with the theme data using "Combine by Position" to ensure every theme row has access to the global settings.

#### 2.2 Task Filtering & Batching
**Overview:** This block ensures that only unprocessed themes are handled and manages the rate of execution.
- **Nodes Involved:** `Filter Pending Coloring Tasks`, `Split In Batches`.
- **Node Details:**
    - **Filter Pending Coloring Tasks (IF):** Checks if the `status` column in the sheet equals `"pending"`. Only "True" paths proceed.
    - **Split In Batches:** Regulates the flow of data to avoid hitting API rate limits or crashing the system during large-scale generation.

#### 2.3 Prompt Engineering & Variation Generation
**Overview:** Transforms raw theme data into a technical prompt for Stable Diffusion and expands a single theme into multiple unique image requests.
- **Nodes Involved:** `Build Prompt`, `Generate Coloring Variations`.
- **Node Details:**
    - **Build Prompt (Set):** Creates a structured string containing requirements (Line art, white background, bold outlines, no shading) combined with theme, style, keywords, and size.
    - **Generate Coloring Variations (Code):** A JavaScript node that reads the `image_count` from settings. It loops to create $X$ number of items, adding a "Variation" suffix to the prompt to ensure the AI produces different compositions for the same theme. It also generates a unique filename for Google Drive.

#### 2.4 AI Generation & Storage
**Overview:** Executes the image generation and saves the resulting binary file to the cloud.
- **Nodes Involved:** `Wait`, `Generate Image (huggingface456)`, `Save Coloring Image to Drive`.
- **Node Details:**
    - **Wait:** Implements an 8-second delay between requests to satisfy Hugging Face API rate limits.
    - **Generate Image (HTTP Request):** Sends a POST request to the Stable Diffusion XL model. Configuration is set to return a "File" response format.
    - **Save Coloring Image to Drive (Google Drive):** Uploads the binary image. The filename is dynamically constructed using the theme and a timestamp to avoid overwriting.

#### 2.5 Metadata Processing & Logging
**Overview:** Creates marketing-ready text for the generated image and logs the final link and data.
- **Nodes Involved:** `Prepare Output Metadata`, `Log Results to Google Sheets`.
- **Node Details:**
    - **Prepare Output Metadata (Code):** Generates a professional Title, Subtitle, Description, and a comma-separated list of Keywords based on the theme.
    - **Log Results to Google Sheets (Google Sheets):** Appends the metadata and the Google Drive `webViewLink` to the "Result" tab of the master sheet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Manually | Manual Trigger | Manual Start | None | Config / Get Themes | AI Coloring Page Generator (Overall Description) |
| Run Weekly... | Schedule Trigger | Scheduled Start | None | Config / Get Themes | AI Coloring Page Generator (Overall Description) |
| Coloring Page Config | Google Sheets | Fetch Global Settings | Triggers | Merge Settings... | Prompt Input & Configuration |
| Get Themes | Google Sheets | Fetch Theme List | Triggers | Merge Settings... | Prompt Input & Configuration |
| Merge Settings... | Merge | Combine Data | Config / Themes | Filter Pending... | Prompt Input & Configuration |
| Filter Pending... | IF | Status Check | Merge Settings... | Split In Batches | Filter Pending Coloring Tasks |
| Split In Batches | Split In Batches | Loop Controller | Filter / Log Results | Build Prompt | Filter Pending Coloring Tasks |
| Build Prompt | Set | Prompt Construction | Split In Batches | Generate Variations | Filter Pending Coloring Tasks |
| Generate Coloring Variations | Code | Create Multi-images | Build Prompt | Wait | Filter Pending Coloring Tasks |
| Wait | Wait | Rate Limiting | Generate Variations | Generate Image | Save Outputs and Log Results |
| Generate Image... | HTTP Request | Image Creation | Wait | Save to Drive | Save Outputs and Log Results |
| Save Coloring Image... | Google Drive | Cloud Storage | Generate Image | Prepare Metadata | Save Outputs and Log Results |
| Prepare Output Metadata | Code | SEO Text Generation | Save to Drive | Log Results | Save Outputs and Log Results |
| Log Results... | Google Sheets | Final Data Logging | Prepare Metadata | Split In Batches | Save Outputs and Log Results |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Data Source Setup
1. Create a Google Sheet with three tabs: `Settings` (columns: `image_count`), `ColoringThemes` (columns: `theme`, `style`, `keyword`, `Size`, `status`), and `Result` (columns: `theme`, `title`, `subtitle`, `description`, `keywords`, `pdf_link`).

#### Step 2: Input & Filtering
2. Add a **Manual Trigger** and a **Schedule Trigger**.
3. Add two **Google Sheets** nodes: one to read the `Settings` tab and one to read the `ColoringThemes` tab.
4. Connect both to a **Merge** node set to `Combine by Position`.
5. Add an **IF** node to check if `status` is equal to `pending`.

#### Step 3: The Processing Loop
6. Add a **Split In Batches** node.
7. Add a **Set** node to create the `prompt` variable. Use the template: *"create a clean black and white coloring page... Theme: {{theme}} Style: {{style}}..."*
8. Add a **Code** node to create variations. Use a `for` loop based on `image_count` to output multiple items with a `Variation X` suffix in the prompt.

#### Step 4: AI Generation & Storage
9. Add a **Wait** node set to 8 seconds.
10. Add an **HTTP Request** node:
    - **Method:** POST
    - **URL:** `https://router.huggingface.co/hf-inference/models/stabilityai/stable-diffusion-xl-base-1.0`
    - **Authentication:** HuggingFace API Key.
    - **Body:** `{"inputs": "{{$json.prompt}}"}`
    - **Response Format:** File.
11. Add a **Google Drive** node (Upload) and map the filename to `{{$json.theme}}_{{$now}}.png`.

#### Step 5: Finalization
12. Add a **Code** node to generate the marketing strings (Title, Subtitle, Description) using template literals based on the `theme`.
13. Add a **Google Sheets** node set to `Append` to the `Result` tab.
14. Close the loop by connecting the `Log Results` node back to the **Split In Batches** node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| The workflow uses an externalized configuration strategy, allowing non-technical users to change image counts and themes via Google Sheets without opening n8n. | Architectural Design |
| Stable Diffusion XL is used via the Hugging Face Inference API for consistency in line art. | Technology Stack |
| Rate limiting (Wait node) is critical to avoid 429 Too Many Requests errors from Hugging Face. | Stability Tip |