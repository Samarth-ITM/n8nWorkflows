Generate an SEO PDF report from HTML with Gotenberg and Claude Opus 4.6

https://n8nworkflows.xyz/workflows/generate-an-seo-pdf-report-from-html-with-gotenberg-and-claude-opus-4-6-13758


# Generate an SEO PDF report from HTML with Gotenberg and Claude Opus 4.6

# Reference Document: Generate an SEO PDF Report from HTML with Gotenberg and Claude Opus 4.6

## 1. Workflow Overview
This workflow automates the generation of a professional SEO audit report in PDF format. It starts with a user-provided URL, extracts the website's content, uses an advanced AI model (Anthropic Claude) to analyze SEO performance, and finally leverages a Gotenberg instance to transform the structured HTML analysis into a high-quality PDF document.

### Logical Blocks
*   **1.1 Input Reception:** Captures the target URL via an n8n Form.
*   **1.2 Data Extraction:** Fetches the raw HTML content of the provided website.
*   **1.3 AI Analysis:** Cleans the HTML and uses Claude Opus 4.6 to perform a structured SEO audit, outputting semantic HTML.
*   **1.4 File Preparation:** Converts the AI's text output into a physical `.html` file.
*   **1.5 PDF Generation:** Sends the HTML file to a Gotenberg service for conversion to PDF.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception
*   **Overview:** Provides the user interface for initiating the workflow.
*   **Nodes Involved:** `On form submission`
*   **Node Details:**
    *   **Type:** Form Trigger
    *   **Configuration:** 
        *   Title: "Automatic SEO Report"
        *   Field: `url` (Label: "What the URL you want to analyze?", Required: True)
    *   **Input/Output:** Starts the process; outputs the URL string.

### 2.2 Data Extraction
*   **Overview:** Retrieves the source code of the target website.
*   **Nodes Involved:** `Extracting HTML from URL`
*   **Node Details:**
    *   **Type:** HTTP Request
    *   **Configuration:** Method `GET`, URL dynamically mapped to `{{ $json.url }}`.
    *   **Edge Cases:** May fail if the website has anti-scraping measures, requires JS rendering, or is behind a firewall (403/404 errors).

### 2.3 AI Analysis
*   **Overview:** Processes the raw HTML and generates the audit report content.
*   **Nodes Involved:** `AI Agent`, `Anthropic Chat Model`
*   **Node Details:**
    *   **AI Agent:**
        *   **Input Expression:** Uses a complex regex to sanitize the HTML (removes scripts, styles, noscripts, SVGs, comments, and most attributes) to reduce token usage.
        *   **System Message:** Instructs the agent to act as a "Senior SEO Analyst" and follow specific structural guidelines (Score, Title Tag, Meta, Headings, Content, Technical, Recommendations).
        *   **Output Constraint:** Strictly requests raw HTML tags for PDF compatibility.
    *   **Anthropic Chat Model:**
        *   **Model:** `claude-opus-4-6`
        *   **Credential:** `Anthropic API`
    *   **Potential Failure:** AI context window limits if the website HTML is excessively large even after sanitization.

### 2.4 File Preparation
*   **Overview:** Prepares the generated text for the PDF engine.
*   **Nodes Involved:** `Convert to File`
*   **Node Details:**
    *   **Type:** Convert to File
    *   **Configuration:** Operation: "To Text"; Source Property: `output` (from AI Agent); File Name: `index.html`.
    *   **Role:** Transforms the string into a binary file object recognized by the next HTTP node.

### 2.5 PDF Generation
*   **Overview:** Communicates with Gotenberg to finalize the document.
*   **Nodes Involved:** `Using Gotenberg`
*   **Node Details:**
    *   **Type:** HTTP Request
    *   **Configuration:**
        *   URL: `https://demo.gotenberg.dev/forms/chromium/convert/html`
        *   Method: `POST`
        *   Body Content Type: `multipart-form-data`
        *   Parameters: Name: `files`, Type: `formBinaryData`, Input Data Field Name: `data`.
    *   **Note:** The demo URL is for testing only. Production environments should use a self-hosted Docker instance.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On form submission | Form Trigger | User Input | None | Extracting HTML from URL | 1. Passing the URL for analysis |
| Extracting HTML from URL | HTTP Request | Scrape Website | On form submission | AI Agent | 2. Extracting HTML from URL |
| AI Agent | AI Agent | SEO Reasoning | Extracting HTML from URL | Convert to File | 2. Generating the HTML of the SEO report with an AI Agent |
| Anthropic Chat Model | Anthropic Model | LLM Provider | None | AI Agent | |
| Convert to File | Convert to File | Text to Binary | AI Agent | Using Gotenberg | 3. Generating the HTML file for gotenberg |
| Using Gotenberg | HTTP Request | PDF Generation | Convert to File | None | 4. Using gotenberg to convert HTML to PDF. [Plus detailed Gotenberg setup notes]. |

---

## 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create an **n8n Form Trigger** node. Add one required text field named `url`.
2.  **Scraping Setup:** Add an **HTTP Request** node. Set the URL to `{{ $json.url }}`. This will output the website HTML as a string in the data property.
3.  **AI Logic:**
    *   Add an **AI Agent** node. Set the prompt to include a regex-sanitized version of the input HTML to save tokens.
    *   Connect an **Anthropic Chat Model** node to the AI Agent. Choose `claude-opus-4-6` and provide your API Key.
    *   In the Agent's "System Message," define the SEO report structure and insist on a raw `<html>` output with print-friendly CSS.
4.  **File Conversion:** Add a **Convert to File** node. Set the operation to "To Text" and use the expression `{{ $json.output }}` as the source. Name the file `index.html`.
5.  **PDF Conversion:**
    *   Add an **HTTP Request** node.
    *   Set Method to `POST` and URL to your Gotenberg instance (e.g., `https://demo.gotenberg.dev/forms/chromium/convert/html`).
    *   Select `Send Body` -> `multipart-form-data`.
    *   Add a Body Parameter: Name: `files`, Type: `formBinaryData`, value: `data`.
6.  **Connections:** Link nodes in this order: Form -> HTTP (Scrape) -> AI Agent -> Convert to File -> HTTP (Gotenberg).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Video Walkthrough** | [YouTube Video](https://www.youtube.com/watch?v=gl1zdyvqHiQ) |
| **Gotenberg Docs** | [Gotenberg Documentation](https://gotenberg.dev/docs/routes#convert-with-chromium) |
| **Author Contact** | [Marcelo Miranda - LinkedIn](https://www.linkedin.com/in/marceloamiranda) |
| **PDF Best Practices** | [PDF Noodle - GitHub](https://github.com/pdfnoodle/pdf-best-practices) |
| **Production Warning** | The demo Gotenberg URL is public and rate-limited. Use Docker `gotenberg/gotenberg:8` for production. |