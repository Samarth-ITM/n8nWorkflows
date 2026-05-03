Generate styled PDF or DOCX business reports from Markdown with Autype

https://n8nworkflows.xyz/workflows/generate-styled-pdf-or-docx-business-reports-from-markdown-with-autype-13777


# Generate styled PDF or DOCX business reports from Markdown with Autype

This document provides a technical overview and configuration guide for the **Generate Styled Markdown Report with Image Upload** workflow in n8n.

### 1. Workflow Overview
The primary purpose of this workflow is to automate the generation of professional, branded business reports in PDF or DOCX format. It leverages **Extended Markdown** via the Autype integration to transform raw data and styling parameters into high-quality documents.

The logic is divided into three functional segments:
1.  **Configuration & Branding:** Defining the visual identity (colors, fonts, logos).
2.  **Asset Management:** Fetching external images and staging them in a temporary secure environment.
3.  **Document Rendering:** Combining Markdown content, dynamic charts, and the staged assets into a final styled file.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration & Input
This block initializes the workflow and establishes the variables used for branding and content.
*   **Nodes Involved:** `Run Workflow`, `Company Settings`.
*   **Node Details:**
    *   **Run Workflow (Manual Trigger):** Initiates the process.
    *   **Company Settings (Set):** Defines the "Theme" of the report. It assigns strings for `companyName`, `primaryColor` (Hex), `secondaryColor`, `fontFamily` (e.g., Arial), and URLs for the logo and a placeholder report image.
    *   **Potential Failures:** Incorrect Hex codes or broken image URLs in the configuration will cause rendering issues later.

#### 2.2 Asset Preparation
Before rendering, any local or external images used *inside* the report body must be processed.
*   **Nodes Involved:** `Download Report Image`, `Upload Temp Image`.
*   **Node Details:**
    *   **Download Report Image (HTTP Request):** Fetches the report image defined in the previous block. It is configured to output a "File" (binary data).
    *   **Upload Temp Image (Autype):** Takes the binary file from the HTTP request and uploads it to Autype's temporary storage (valid for 24 hours). 
    *   **Key Output:** It returns a `refPath` (e.g., `/temp-image/XYZ`). This internal reference is used in the Markdown instead of a public URL to ensure the image renders correctly in the PDF/DOCX engine.
    *   **Edge Cases:** If the image URL is 404, the HTTP request will fail.

#### 2.3 Document Generation
The final stage where the report structure is built and styled.
*   **Nodes Involved:** `Render Markdown Report`.
*   **Node Details:**
    *   **Render Markdown Report (Autype):** Uses the `renderMarkdown` operation.
    *   **Markdown Content:** A complex expression combining static text, variables from "Company Settings", and the `refPath` from the upload node. It includes:
        *   Page breaks (`---page---`)
        *   Image alignment and sizing syntax (`{width=520 align=center}`)
        *   Dynamic Tables (Financial metrics)
        *   Interactive Charts (Bar and Line charts using the `:::chart` syntax)
    *   **Styling (Defaults JSON):** A JSON object that defines the global look, including font sizes, line heights, header/footer configuration (with page numbering), and chart color palettes using the primary/secondary colors defined in Block 1.
    *   **Output:** Generates a binary file (PDF by default) available for download or further processing (e.g., email or cloud storage).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Run Workflow** | Manual Trigger | Entry Point | None | Company Settings | |
| **Company Settings** | Set | Global Variables | Run Workflow | Download Report Image | 1. Company Settings: Brand colors, font, logo URL, and report image URL. |
| **Download Report Image** | HTTP Request | Asset Fetching | Company Settings | Upload Temp Image | 2. Mock Image Download & Image Upload: Download an image → upload to Autype as temp image. |
| **Upload Temp Image** | Autype | Temporary Storage | Download Report Image | Render Markdown Report | 2. Mock Image Download & Image Upload: Download an image → upload to Autype as temp image. |
| **Render Markdown Report** | Autype | Doc Generation | Upload Temp Image | None | 3. Render: Markdown content + defaults JSON → Autype renders PDF/DOCX. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Prerequisites:** 
    *   Install the `n8n-nodes-autype` community node (Settings > Community Nodes).
    *   Create an account at [app.autype.com](https://app.autype.com) and generate an API Key.
    *   Add an "Autype API" credential in your n8n instance.

2.  **Define Variables:** Add a **Set** node. Create string assignments for:
    *   `companyName`, `primaryColor`, `textColor`, `fontFamily`, `companyLogoUrl`, and `reportImageUrl`.

3.  **Fetch Assets:** Add an **HTTP Request** node. 
    *   Method: `GET`. 
    *   URL: `{{ $json.reportImageUrl }}`. 
    *   Response Format: `File`.

4.  **Upload to Staging:** Add an **Autype** node. 
    *   Resource: `File`, Operation: `Upload Image`.
    *   Connect the input to the HTTP Request node.

5.  **Generate Report:** Add another **Autype** node. 
    *   Operation: `Render Markdown`.
    *   **Content Field:** Construct your Markdown. To use the uploaded image, use: `![Alt Text]({{ $json.refPath }})`. 
    *   **Additional Fields > Defaults:** Create a JSON object to map your variables (e.g., `fontFamily: {{ $('Company Settings').first().json.fontFamily }}`).
    *   **Download Output:** Toggle this to **ON** to receive the final file.

6.  **Connect & Test:** Link the nodes in the order described in the Summary Table and click "Execute Workflow".

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Community Node Required** | This workflow requires `n8n-nodes-autype` on a self-hosted n8n instance. |
| **Extended Markdown Specs** | Details on chart and layout syntax can be found in the [Autype Documentation](https://docs.autype.com). |
| **Temporary Files** | The `Upload Temp Image` node creates assets that automatically expire after 24 hours. |
| **API Key Setup** | Get your API key at [app.autype.com](https://app.autype.com). |