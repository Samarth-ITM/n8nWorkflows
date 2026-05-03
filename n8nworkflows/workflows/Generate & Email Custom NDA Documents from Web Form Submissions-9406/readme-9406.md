Generate & Email Custom NDA Documents from Web Form Submissions

https://n8nworkflows.xyz/workflows/generate---email-custom-nda-documents-from-web-form-submissions-9406


# Generate & Email Custom NDA Documents from Web Form Submissions

### 1. Workflow Overview

This workflow automates the process of generating and emailing a customized Non-Disclosure Agreement (NDA) document based on data submitted via a web form on a landing page. It is designed for businesses seeking to streamline NDA requests and delivery without manual intervention.

Logical blocks:

- **1.1 Landing Page Delivery**: Hosts and serves the HTML landing page with the NDA request form.
- **1.2 Form Submission Reception**: Receives submitted form data from the landing page.
- **1.3 NDA Document Generation**: Creates a personalized NDA document in HTML format, then converts it to a DOCX file.
- **1.4 Email Dispatch**: Sends the generated NDA document as an email attachment to a predefined recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Landing Page Delivery

**Overview:**  
This block hosts the landing page form for users to request an NDA. It serves HTML content styled with Tailwind CSS and includes client-side JavaScript to submit form data to the backend webhook.

**Nodes Involved:**  
- Landingpage Endpoint  
- Set Form Endpoint  
- HTML for Landingpage  
- Respond to Webhook  
- Sticky Note1

**Node Details:**

- **Landingpage Endpoint**  
  - Type: Webhook (HTTP POST)  
  - Role: Entry point serving the landing page HTML. Path: `2f94bacb-f629-4053-a204-cab2ac8fd326`  
  - Config: Response mode set to use a response node (Respond to Webhook)  
  - Inputs: None (external HTTP request)  
  - Outputs: Set Form Endpoint  
  - Edge Cases: Invalid HTTP methods; missing response node may cause errors

- **Set Form Endpoint**  
  - Type: Set  
  - Role: Assigns the landing page’s form submission URL dynamically using the webhook URL from the workflow context.  
  - Config: Sets variable `FormEndpoint` to the webhook URL of the FormData Endpoint node.  
  - Inputs: Landingpage Endpoint  
  - Outputs: HTML for Landingpage  
  - Edge Cases: If webhook URL not resolved correctly, form submission URL will be invalid.

- **HTML for Landingpage**  
  - Type: HTML  
  - Role: Contains the full HTML of the landing page including the form, Tailwind CSS, and JavaScript for form submission.  
  - Config: Static HTML with embedded script that reads `{{ $json.FormEndpoint }}` to post data to the backend FormData Endpoint.  
  - Inputs: Set Form Endpoint (provides form endpoint URL)  
  - Outputs: Respond to Webhook  
  - Edge Cases: Malformed HTML could break UI; script errors prevent form submission; CORS issues on fetch if server not properly configured

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends the HTML content of the landing page as the HTTP response to the initial GET request.  
  - Inputs: HTML for Landingpage  
  - Outputs: None (HTTP response)  
  - Edge Cases: Response timeout or misconfiguration could lead to blank or partial page load

- **Sticky Note1**  
  - Content: "## Landingpage Server\nYou can access the landing page at the webhook address."  
  - Applies to: Entire Landing Page Delivery block

---

#### 1.2 Form Submission Reception

**Overview:**  
Receives the JSON-formatted form data submitted from the landing page and triggers NDA generation.

**Nodes Involved:**  
- FormData Endpoint  
- NDA (HTML Version)  
- Sticky Note

**Node Details:**

- **FormData Endpoint**  
  - Type: Webhook (HTTP POST)  
  - Role: Entry point for receiving submitted form data from the client-side form. Path: `2f94bacb-f629-4053-a204-cab2ac8fd326` (same path as Landingpage Endpoint but different webhookId)  
  - Config: Responds with a simple HTML snippet `<h1>Hello World</h1>` (likely to confirm submission)  
  - Inputs: None (external HTTP request)  
  - Outputs: NDA (HTML Version)  
  - Edge Cases: Wrong HTTP method or malformed JSON payload; network timeouts; missing required fields

- **NDA (HTML Version)**  
  - Type: HTML  
  - Role: Generates a personalized NDA document in HTML format by injecting form data into placeholders within a styled NDA contract template.  
  - Config: Uses expressions like `{{ $json.body.firstName }}` to insert submitted data dynamically. The current date is also injected. The NDA is styled for print/word processing.  
  - Inputs: FormData Endpoint  
  - Outputs: HTML to Docx  
  - Edge Cases: Missing form fields cause incomplete placeholders; invalid HTML structure may break conversion downstream

- **Sticky Note**  
  - Content: "## FormData Endpoint\nThe data for the form is received here from the landing page."  
  - Applies to: FormData Endpoint node

---

#### 1.3 NDA Document Generation

**Overview:**  
Converts the NDA HTML document into a DOCX file format suitable for emailing and printing.

**Nodes Involved:**  
- HTML to Docx  
- Send email

**Node Details:**

- **HTML to Docx**  
  - Type: Custom Node (`@custom-js/n8n-nodes-pdf-toolkit.Html2Docx`)  
  - Role: Transforms the HTML NDA document into a DOCX file.  
  - Config: Input is the HTML content from the NDA (HTML Version) node; uses external custom JavaScript API credential ("Coding Service") for conversion.  
  - Inputs: NDA (HTML Version)  
  - Outputs: Send email  
  - Version: Requires custom node plugin `@custom-js/n8n-nodes-pdf-toolkit` and configured custom API credentials.  
  - Edge Cases: Conversion errors if HTML contains unsupported elements; credential or API downtime; file size limits

---

#### 1.4 Email Dispatch

**Overview:**  
Sends the finalized NDA DOCX document by email to a predefined recipient using SMTP.

**Nodes Involved:**  
- Send email

**Node Details:**

- **Send email**  
  - Type: Email Send (SMTP)  
  - Role: Sends an email with the DOCX NDA document attached as binary data.  
  - Config:  
    - Subject: "NDA"  
    - Body Text: Simple plaintext message ("Hello, Here is the requested NDA form. Best Henrik")  
    - Recipient: `test@testmail.com` (hardcoded)  
    - Sender: `test@test.de` (hardcoded)  
    - Attachments: Uses binary data from previous node (converted DOCX)  
    - Credentials: Uses stored SMTP credential labeled "SMTP account"  
  - Inputs: HTML to Docx  
  - Outputs: None  
  - Edge Cases: SMTP authentication failure; network issues; attachment size limits; invalid recipient email address

---

### 3. Summary Table

| Node Name          | Node Type                    | Functional Role                    | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                   |
|--------------------|------------------------------|----------------------------------|-----------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Landingpage Endpoint| Webhook                      | Serves landing page HTML          | None                  | Set Form Endpoint      | ## Landingpage Server You can access the landing page at the webhook address.                |
| Set Form Endpoint   | Set                          | Sets form submission endpoint URL| Landingpage Endpoint   | HTML for Landingpage   | ## Landingpage Server You can access the landing page at the webhook address.                |
| HTML for Landingpage| HTML                         | Provides landing page HTML content| Set Form Endpoint     | Respond to Webhook     | ## Landingpage Server You can access the landing page at the webhook address.                |
| Respond to Webhook  | Respond to Webhook           | Sends landing page HTML response  | HTML for Landingpage   | None                  | ## Landingpage Server You can access the landing page at the webhook address.                |
| Sticky Note1        | Sticky Note                  | Info about landing page server    | None                  | None                  | ## Landingpage Server You can access the landing page at the webhook address.                |
| FormData Endpoint   | Webhook                      | Receives submitted form data      | None                  | NDA (HTML Version)     | ## FormData Endpoint The data for the form is received here from the landing page.           |
| NDA (HTML Version)  | HTML                         | Generates personalized NDA HTML   | FormData Endpoint      | HTML to Docx           | ## FormData Endpoint The data for the form is received here from the landing page.           |
| Sticky Note        | Sticky Note                  | Info about form data reception    | None                  | None                  | ## FormData Endpoint The data for the form is received here from the landing page.           |
| HTML to Docx        | Custom Node (Html2Docx)      | Converts NDA HTML to DOCX format  | NDA (HTML Version)     | Send email             |                                                                                              |
| Send email          | Email Send (SMTP)            | Sends NDA document by email       | HTML to Docx           | None                  |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Landingpage Endpoint**  
   - Type: Webhook  
   - HTTP Method: GET (default) or POST (configured)  
   - Path: `2f94bacb-f629-4053-a204-cab2ac8fd326`  
   - Response Mode: Select `Respond With Response Node`  
   - Save node

2. **Create Set Node: Set Form Endpoint**  
   - Add a field `FormEndpoint` with value expression `{{$json["webhookUrl"]}}` (takes webhook URL from Landingpage Endpoint)  
   - Connect Landingpage Endpoint → Set Form Endpoint

3. **Create HTML Node: HTML for Landingpage**  
   - Paste the full landing page HTML (provided in the workflow) into the HTML field  
   - Note: Ensure the form submission fetch URL uses `{{ $json.FormEndpoint }}` to dynamically fill the webhook URL for form submission  
   - Connect Set Form Endpoint → HTML for Landingpage

4. **Create Respond to Webhook Node**  
   - Configure to respond with text  
   - Response Body: `={{ $json.html }}` (use HTML from previous node)  
   - Connect HTML for Landingpage → Respond to Webhook

5. **Create Webhook Node: FormData Endpoint**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `2f94bacb-f629-4053-a204-cab2ac8fd326` (same as Landingpage Endpoint but unique webhook ID)  
   - Configure it to respond with a simple HTML string `<h1>Hello World</h1>` (optional)  
   - Save node

6. **Create HTML Node: NDA (HTML Version)**  
   - Paste the NDA HTML template with placeholders such as `{{ $json.body.firstName }}`, `{{ $json.body.lastName }}`, and `{{ $json.body.currentDate }}`  
   - Use JavaScript expression or workflow function to inject the current date in `currentDate` field before this node triggers or inline in the HTML  
   - Connect FormData Endpoint → NDA (HTML Version)

7. **Create Custom Node: HTML to Docx**  
   - Install and configure the `@custom-js/n8n-nodes-pdf-toolkit` package if not already available  
   - Configure credentials for the custom API service (e.g., "Coding Service")  
   - Set the HTML input to `={{ $json.html }}` from NDA (HTML Version) node  
   - Connect NDA (HTML Version) → HTML to Docx

8. **Create Email Send Node: Send email**  
   - Set From Email: `test@test.de` (or your valid sender email)  
   - Set To Email: `test@testmail.com` (or desired recipient)  
   - Subject: "NDA"  
   - Body Text: "Hello,\n\nHere is the requested NDA form.\n\nBest\nHenrik" (plain text)  
   - Attachments: Use binary data from HTML to Docx node to attach the generated DOCX file  
   - Configure SMTP credentials ("SMTP account") with valid SMTP server, username, and password  
   - Connect HTML to Docx → Send email

9. **Activate Workflow**  
   - Verify all nodes are connected as per above  
   - Activate workflow to enable webhooks  

10. **Test**  
    - Access the Landingpage Endpoint URL in a browser  
    - Fill and submit the NDA request form  
    - Verify email is received with correct NDA document attached  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Landing page uses Tailwind CSS for styling and includes client-side JavaScript to handle form submission.     | Embedded in HTML for Landingpage node                                                                    |
| NDA template is styled for print; placeholders use n8n expressions to insert user data dynamically.            | NDA (HTML Version) node                                                                                   |
| Custom Node `Html2Docx` requires external API service credential ("Coding Service").                           | Credential configuration needed for HTML to Docx node                                                    |
| SMTP credentials must be configured with valid SMTP provider details to enable email sending.                  | Send email node                                                                                           |
| The landing page and form webhook share the same URL path but have distinct webhook IDs internally in n8n.    | Important for routing different HTTP methods and processing in n8n                                       |
| Form submission expects JSON body with fields: firstName, lastName, address, houseNo, postalCode, city, country | From the client-side JavaScript fetch call in landing page HTML                                          |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.