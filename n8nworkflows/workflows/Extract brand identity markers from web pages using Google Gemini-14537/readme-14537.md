Extract brand identity markers from web pages using Google Gemini

https://n8nworkflows.xyz/workflows/extract-brand-identity-markers-from-web-pages-using-google-gemini-14537


# Extract brand identity markers from web pages using Google Gemini

# Workflow Documentation: Extract Brand Identity Markers from Web Pages

## 1. Workflow Overview
This workflow automates the extraction of a brand's verbal identity and communication style from a given website URL. It uses Google Gemini AI to analyze raw HTML/text and transform it into a structured "Style Matrix" (JSON), which is then rendered into a professional HTML dashboard for the user.

The workflow is designed for marketing agencies or brand managers who need to quantify a brand's tone, lexicon, and syntax to ensure consistency across new content.

### Logical Blocks
- **1.1 Input & URL Validation:** Captures the target URL via a form and ensures the protocol (`http://` or `https://`) is correctly formatted.
- **1.2 Content Retrieval:** Fetches the raw content of the web page via an HTTP request.
- **1.3 AI Identity Extraction:** Processes the raw text using Google Gemini to extract specific linguistic markers into a rigid JSON schema.
- **1.4 Data Parsing & Formatting:** Cleans the AI response and maps the nested JSON data.
- **1.5 Visual Presentation:** Generates a styled HTML report and delivers it back to the user via the n8n Form completion page.

---

## 2. Block-by-Block Analysis

### 2.1 Input & URL Validation
**Overview:** This block handles the user entry point and sanitizes the URL to prevent HTTP request failures.
- **Nodes Involved:** `URL Form`, `Check URL`, `Add Missing Protocol`.
- **Node Details:**
    - **URL Form (Form Trigger):** Provides a UI for the user to input a "URL".
    - **Check URL (Switch):** Evaluates the input string. It routes to different paths if the URL starts with `http://`, `https://`, or has no protocol.
    - **Add Missing Protocol (Set):** If no protocol is found, it prepends `https://` to the URL to ensure the subsequent request is valid.
    - **Edge Cases:** Users entering non-URL text may cause the subsequent HTTP request to fail.

### 2.2 Content Retrieval
**Overview:** Retrieves the raw source code/text of the specified webpage.
- **Nodes Involved:** `HTTP Request`.
- **Node Details:**
    - **HTTP Request (HTTP Request):** Performs a GET request to the validated URL.
    - **Configuration:** `allowUnauthorizedCerts` is enabled to avoid blocking requests to sites with SSL issues.
    - **Failure Types:** 404 Not Found, 403 Forbidden (Bot blocking), or DNS timeouts.

### 2.3 AI Identity Extraction
**Overview:** The core intelligence layer that transforms raw website text into a quantified brand matrix.
- **Nodes Involved:** `Extract Vault A`.
- **Node Details:**
    - **Extract Vault A (Google Gemini):** Uses the `gemini-3-flash-preview` model.
    - **Prompt Logic:** It acts as the "Litopy Extraction Engine." It focuses on three pillars: Lexicon Mining (Power vs. Forbidden words), Syntax Calculus (Sentence length/complexity), and Stance Detection (Relationship model).
    - **Output Constraint:** Strictly forced to output JSON matching the `vault_a_style_matrix` schema.
    - **Input:** Passes the `data` (HTML/Text) from the HTTP Request node.

### 2.4 Data Parsing & Formatting
**Overview:** Extracts the specific text content from the Gemini response object and prepares the nested structure for the HTML template.
- **Nodes Involved:** `Extract AI Response`, `Extract Nested Data`.
- **Node Details:**
    - **Extract AI Response (Set):** Isolates the text result from the complex Gemini response object: `{{ $json.candidates[0].content.parts[0].text }}`.
    - **Extract Nested Data (Set):** Maps specific JSON paths (e.g., `lexicon_constraints.power_words`) to distinct variables to make them easily accessible for the HTML generator.

### 2.5 Visual Presentation
**Overview:** Converts the structured data into a human-readable, branded report.
- **Nodes Involved:** `HTML`, `Form`.
- **Node Details:**
    - **HTML (HTML):** A large template containing CSS and HTML. It uses expressions to map the JSON arrays (like `power_words`) into HTML `<span>` tags and calculates percentage widths for progress bars based on the `active_voice_ratio` and `emotional_temperature`.
    - **Form (Form):** The final node that displays the generated HTML to the user as the completion screen of the initial form trigger.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| URL Form | Form Trigger | User Input | - | Check URL | Extract Brand Identity Markers from Web Page... |
| Check URL | Switch | URL Validation | URL Form | HTTP Request, Add Missing Protocol | Retrieve the Web Page Content |
| Add Missing Protocol | Set | Protocol Sanitization | Check URL | HTTP Request | Retrieve the Web Page Content |
| HTTP Request | HTTP Request | Web Page Fetching | Check URL, Add Missing Protocol | Extract Vault A | Retrieve the Web Page Content |
| Extract Vault A | Google Gemini | Brand Logic Extraction | HTTP Request | Extract AI Response | Brand Identity Marker Extraction |
| Extract AI Response | Set | AI Response Cleaning | Extract Vault A | Extract Nested Data | Extract nested Data |
| Extract Nested Data | Set | Variable Mapping | Extract AI Response | HTML | Extract nested Data |
| HTML | HTML | Report Generation | Extract Nested Data | Form | Extract nested Data |
| Form | Form | Final Display | HTML | - | - |

---

## 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    - Create a **Form Trigger** node. Add one text field labeled "URL".
2.  **URL Logic:**
    - Add a **Switch** node. Create three outputs:
        - Route 1: `URL` starts with `http://`
        - Route 2: `URL` starts with `https://`
        - Route 3: Default (No protocol).
    - Connect Route 3 to a **Set** node that updates the `URL` variable to: `https://{{ $json.URL }}`.
3.  **Data Fetching:**
    - Create an **HTTP Request** node. Set the URL to the variable provided by the previous steps. Enable "Allow Unauthorized Certificates" in options.
4.  **AI Integration:**
    - Add a **Google Gemini** node.
    - **Model:** Select `gemini-3-flash-preview`.
    - **Prompt:** Use the "System Instruction" to define the `vault_a_style_matrix` JSON schema (Lexicon, Syntax, Stance, Formatting).
    - **Input:** Pass the output of the HTTP Request node as the "Input Text".
    - **Credentials:** Connect your Google Gemini (PaLM) API key.
5.  **Data Post-Processing:**
    - Add a **Set** node to extract the AI's text output using the expression: `{{ $json.candidates[0].content.parts[0].text }}`.
    - Add a second **Set** node to explicitly map the nested JSON fields (Power words, Forbidden words, etc.) to make them accessible for the template.
6.  **UI Generation:**
    - Add an **HTML** node. Paste the CSS/HTML template. Use the `.map()` function in expressions to iterate over the power/forbidden word arrays.
    - Add a **Form** node set to "Completion" operation. Set the response text to the output of the HTML node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Community Support | [n8n Discord](https://discord.com/invite/XPKeKXeB7d) |
| Community Forum | [n8n Forum](https://community.n8n.io/) |
| Workflow Creator | blakewise.com |
| LLM Requirement | Requires a valid Google Gemini API Key |