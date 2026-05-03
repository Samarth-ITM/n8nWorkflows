Collect Star Rating Feedback with Self-Contained HTML Forms and Data Tables

https://n8nworkflows.xyz/workflows/collect-star-rating-feedback-with-self-contained-html-forms-and-data-tables-9787


# Collect Star Rating Feedback with Self-Contained HTML Forms and Data Tables

### 1. Workflow Overview

This workflow, titled **Customer Feedback Collector**, is designed to collect star rating feedback (1 to 5 stars) alongside an optional message from customers via a fully self-contained HTML form. It is optimized for simplicity and autonomy, requiring no external services or credentials. The feedback data, including any URL query parameters, is saved into an n8n Data Table for convenient review and management.

The workflow handles two main HTTP endpoints for the feedback form:

- **GET /feedback**: Serves a styled, accessible HTML feedback form with dynamic content and theming.
- **POST /feedback**: Accepts form submissions, saves the feedback data along with any query parameters into a Data Table, and returns a thank-you confirmation page.

The workflow consists of the following logical blocks:

- **1.1 Form Setup and Theming:** Defines the form content, UI text, colors, and styling variables.
- **1.2 Feedback Form Delivery:** Handles GET requests to serve the feedback input HTML form.
- **1.3 Feedback Submission Processing:** Handles POST requests to receive submitted feedback, saves it to the Data Table, and responds with a confirmation page.
- **1.4 Data Persistence:** Maps and saves incoming feedback data and query parameters into a Data Table.
- **1.5 UI Confirmation Rendering:** Displays a thank-you page after successful feedback submission.

---

### 2. Block-by-Block Analysis

#### 1.1 Form Setup and Theming

**Overview:**  
This block initializes all configurable text labels, URLs, and styling variables used throughout the feedback form and confirmation pages. It centralizes the form’s content and visual theme for easy maintenance and customization.

**Nodes Involved:**  
- Form Configuration  
- Theme Configuration - Feedback Input  
- Theme Configuration - Feedback Submitted

**Node Details:**

- **Form Configuration**  
  - Type: Set  
  - Role: Defines form metadata including title, headers, button texts, POST URL, and query parameter handling expression.  
  - Key Expressions:  
    - `PostFeedbackLink`: URL for POST endpoint (e.g., https://sus-tech.app.n8n.cloud/webhook-test/feedback)  
    - `queryParams`: Dynamically builds query string from incoming request query parameters using expression, ensuring proper encoding and filtering empty values.  
  - Inputs: Receives original webhook request JSON with possible query parameters.  
  - Outputs: Supplies configuration JSON for downstream nodes.  
  - Edge Cases: Expression failure if query object is malformed or contains unsupported data types.

- **Theme Configuration - Feedback Input**  
  - Type: Set  
  - Role: Defines CSS custom properties for the feedback input form’s colors, typography, shadows, sizing, and star icons.  
  - Configuration: Raw JSON with color codes and CSS variables like `--bg`, `--star-active`, `--radius`, `--shadow`.  
  - Inputs: Receives output from Form Configuration.  
  - Outputs: Supplies theme JSON to the HTML form node.

- **Theme Configuration - Feedback Submitted**  
  - Type: Set  
  - Role: Defines CSS custom properties for the thank you confirmation page styling.  
  - Configuration: Raw JSON with colors and layout variables similar to input theme but simplified.  
  - Inputs: Receives data from the Save Feedback to Data Table node (triggered after POST submission).  
  - Outputs: Supplies theme JSON to confirmation HTML node.

---

#### 1.2 Feedback Form Delivery

**Overview:**  
This block handles GET requests to the `/feedback` webhook path, returning a fully styled HTML form that users can interact with to submit feedback. The form is dynamically populated with configuration and themed styles.

**Nodes Involved:**  
- Input Feedback  
- Form Configuration (reused)  
- Theme Configuration - Feedback Input (reused)  
- Feedback Input HTML  
- Respond with Feedback Input

**Node Details:**

- **Input Feedback**  
  - Type: Webhook (HTTP GET)  
  - Role: Entry point for GET requests at `/feedback`.  
  - Configuration: Path set to `feedback`, httpMethod defaults to GET, responseMode set to "responseNode".  
  - Outputs: Passes request data (including query) to Form Configuration node.  
  - Edge Cases: Unpublished webhook or wrong path results in 404; malformed query parameters could cause expression errors downstream.

- **Form Configuration**  
  - As described above, reused to provide form content and URL parameters dynamically based on query params.

- **Theme Configuration - Feedback Input**  
  - As described above, sets the CSS theme for the form.

- **Feedback Input HTML**  
  - Type: HTML  
  - Role: Renders the full HTML document for the feedback form.  
  - Configuration: HTML code uses embedded n8n expressions to inject form titles, descriptions, button labels, POST URL (including query parameters), and CSS variables for theming.  
  - Inputs: Receives form configuration and theme JSON.  
  - Outputs: Provides the final HTML string to the response node.  
  - Edge Cases: Expression evaluation errors if referenced JSON keys are missing.

- **Respond with Feedback Input**  
  - Type: Respond to Webhook  
  - Role: Sends the generated HTML form as the HTTP response to the GET request.  
  - Configuration: Responds with `text/html` body containing the HTML from the previous node.

---

#### 1.3 Feedback Submission Processing

**Overview:**  
This block listens for POST requests to the `/feedback` webhook, processes submitted rating and message fields, saves the data, and replies with a thank-you confirmation page.

**Nodes Involved:**  
- Post Feedback  
- Save Feedback to Data Table  
- Page Configuration  
- Theme Configuration - Feedback Submitted  
- Feedback Submitted HTML  
- Respond with Feedback Submitted

**Node Details:**

- **Post Feedback**  
  - Type: Webhook (HTTP POST)  
  - Role: Entry point for POST requests at `/feedback`.  
  - Configuration: Path set to `feedback`, method POST, responseMode "responseNode".  
  - Outputs: Passes incoming POST body (form fields) and query parameters to Save Feedback node.  
  - Edge Cases: Unpublished webhook, missing required fields (rating), or invalid payload format could cause failures.

- **Save Feedback to Data Table**  
  - Type: Data Table  
  - Role: Persists submitted feedback into an n8n Data Table named "Feedback".  
  - Configuration: Maps incoming `rating` (number), `message` (string), and serializes all query parameters as JSON string into `additionalInformation`.  
  - Inputs: Receives POST body and query JSON.  
  - Outputs: On success routes to Page Configuration node.  
  - Edge Cases: Data Table schema mismatch, permission issues, or network problems could prevent saving.

- **Page Configuration**  
  - Type: Set  
  - Role: Defines textual content for the confirmation page (title, header, description).  
  - Inputs: Triggered after data saving.  
  - Outputs: Supplies configuration for theming the confirmation page.

- **Theme Configuration - Feedback Submitted**  
  - As described above, provides CSS variables for the confirmation page styling.

- **Feedback Submitted HTML**  
  - Type: HTML  
  - Role: Renders a simple, styled thank-you confirmation HTML page.  
  - Inputs: Receives page configuration and theme JSON.  
  - Outputs: Supplies HTML string to response node.

- **Respond with Feedback Submitted**  
  - Type: Respond to Webhook  
  - Role: Sends the thank-you HTML page as the HTTP response to the POST request.

---

#### 1.4 Data Persistence

**Overview:**  
This block focuses on mapping and saving the submitted feedback data along with any query parameters to a designated n8n Data Table, ensuring structured, queryable storage.

**Nodes Involved:**  
- Save Feedback to Data Table

**Node Details:**

- **Save Feedback to Data Table**  
  - As described above, stores `rating` as number, `message` as string, and captures all query parameters as a JSON-formatted string `additionalInformation`.  
  - This ensures metadata like userId or source passed in URL query is preserved with each feedback entry.

---

#### 1.5 UI Confirmation Rendering

**Overview:**  
After successfully saving feedback, this block renders and returns a thank-you page to the user, confirming receipt of their input.

**Nodes Involved:**  
- Page Configuration  
- Theme Configuration - Feedback Submitted  
- Feedback Submitted HTML  
- Respond with Feedback Submitted

**Node Details:**

- These nodes have been detailed above in 1.3 Feedback Submission Processing.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                              | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                          |
|-------------------------------|------------------------|----------------------------------------------|---------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| Post Feedback                 | Webhook (POST)          | Receives submitted feedback POST requests    |                           | Save Feedback to Data Table  | ### How It Works - POST `/feedback` receives rating/message, saves to Data Table, returns confirmation. |
| Save Feedback to Data Table   | Data Table              | Saves feedback data and query params          | Post Feedback             | Page Configuration           | ### Setup - Confirm Data Table mapping (rating, message, additional info).                          |
| Page Configuration           | Set                     | Provides texts for thank-you confirmation page| Save Feedback to Data Table| Theme Configuration - Feedback Submitted |                                                                                                    |
| Theme Configuration - Feedback Submitted | Set           | Defines CSS theme for confirmation page       | Page Configuration        | Feedback Submitted HTML      | ### Setup - Optionally adjust theme configuration.                                                  |
| Feedback Submitted HTML       | HTML                    | Renders thank-you confirmation HTML           | Theme Configuration - Feedback Submitted | Respond with Feedback Submitted | ### Overview - Displays simple “Thank you” confirmation page.                                      |
| Respond with Feedback Submitted| Respond to Webhook     | Sends confirmation HTML response               | Feedback Submitted HTML   |                             |                                                                                                    |
| Input Feedback               | Webhook (GET)           | Serves feedback input HTML form                |                           | Form Configuration           | ### How It Works - GET `/feedback` renders styled HTML form.                                        |
| Form Configuration           | Set                     | Sets form texts, POST URL, and query params   | Input Feedback, Post Feedback | Theme Configuration - Feedback Input | ### Setup - Update title, text, and `PostFeedbackLink` (POST webhook URL).                           |
| Theme Configuration - Feedback Input | Set               | Defines CSS theme for feedback input form      | Form Configuration        | Feedback Input HTML          | ### Setup - Optionally adjust theme configuration.                                                  |
| Feedback Input HTML          | HTML                    | Renders feedback form HTML with embedded variables | Theme Configuration - Feedback Input | Respond with Feedback Input | ### Overview - Fully self-contained feedback form with star rating and optional message.           |
| Respond with Feedback Input  | Respond to Webhook      | Sends feedback form HTML as GET response       | Feedback Input HTML       |                             |                                                                                                    |
| Sticky Note1                 | Sticky Note             | Overview and high-level summary                 |                           |                             | Collects quick 1–5⭐ feedback + optional message. Saves results to Data Table including query params. Displays “Thank you” confirmation page. Fully self-contained, no external creds. |
| Sticky Note2                 | Sticky Note             | Setup instructions                              |                           |                             | Open Form Configuration → update title/text and POST URL. Adjust Theme Configuration. Confirm Data Table mapping. |
| Sticky Note3                 | Sticky Note             | Workflow operation explanation                   |                           |                             | GET `/feedback` renders form. POST `/feedback` saves data and confirms. Query params saved as JSON. |
| Sticky Note4                 | Sticky Note             | Troubleshooting and notes                        |                           |                             | Only submitted fields + query info stored. If failure: verify webhook publishing, POST URL, Data Table schema. Contact office@sus-tech.at |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Input Feedback"**  
   - Type: Webhook  
   - HTTP Method: GET  
   - Path: `feedback`  
   - Response Mode: `responseNode`  

2. **Create Set Node "Form Configuration"**  
   - Assign string parameters:  
     - Title: `"Simple Feedback Input"`  
     - Header: `"Rate your experience"`  
     - Description: `"Your rating helps us improve. Stars are required; message is optional."`  
     - ClearBtn: `"Clear"`  
     - SubmitBtn: `"Submit feedback"`  
     - PostFeedbackLink: set to your POST webhook URL (e.g., `"https://your-n8n-instance/webhook/feedback"`)  
     - queryParams: Use the expression below to build query string from `$json.query`:  
       ```
       ={{ $json.query && Object.keys($json.query).length
         ? '?' + Object.keys($json.query)
             .filter(k => $json.query[k] !== undefined && $json.query[k] !== null && $json.query[k] !== '')
             .map(k => encodeURIComponent(k) + '=' + encodeURIComponent(String($json.query[k])))
             .join('&')
         : '' }}
       ```
   - Connect Input Feedback → Form Configuration.

3. **Create Set Node "Theme Configuration - Feedback Input"**  
   - Mode: Raw JSON  
   - Paste the following JSON for CSS variables:  
     ```
     {
       "bg": "#f7f8fb",
       "card": "#ffffff",
       "text": "#1f2937",
       "muted": "#6b7280",
       "accent": "#3b82f6",
       "accent-contrast": "#ffffff",
       "border": "#e5e7eb",
       "star": "#cbd5e1",
       "star-active": "#f59e0b",
       "focus": "0 0 0 4px rgba(59, 130, 246, 0.25)",
       "radius": "16px",
       "shadow": "0 10px 25px rgba(2, 6, 23, 0.06), 0 4px 10px rgba(2, 6, 23, 0.04)",
       "maxw": "560px",
       "star-size": "28px"
     }
     ```
   - Connect Form Configuration → Theme Configuration - Feedback Input.

4. **Create HTML Node "Feedback Input HTML"**  
   - Paste the full HTML content of the feedback form (as provided in the workflow) into the HTML parameter.  
   - This HTML uses expressions to inject configuration values and CSS variables.  
   - Connect Theme Configuration - Feedback Input → Feedback Input HTML.

5. **Create Respond to Webhook Node "Respond with Feedback Input"**  
   - Respond with: Text  
   - Response Body: `={{ $json.html }}` (output of Feedback Input HTML)  
   - Connect Feedback Input HTML → Respond with Feedback Input.

6. **Create Webhook Node "Post Feedback"**  
   - HTTP Method: POST  
   - Path: `feedback`  
   - Response Mode: `responseNode`  

7. **Create Data Table Node "Save Feedback to Data Table"**  
   - Select or create a Data Table named "Feedback".  
   - Map columns as follows:  
     - `rating`: `={{ $json.body.rating }}` (number)  
     - `message`: `={{ $json.body.message }}` (string)  
     - `additionalInformation`: `={{ JSON.stringify($json.query) }}` (string)  
   - Connect Post Feedback → Save Feedback to Data Table.

8. **Create Set Node "Page Configuration"**  
   - Assign strings:  
     - Title: `"Thank You for Your Feedback"`  
     - Header: `"Thank You!"`  
     - Description: `"We appreciate your feedback — it helps us improve our service."`  
   - Connect Save Feedback to Data Table → Page Configuration.

9. **Create Set Node "Theme Configuration - Feedback Submitted"**  
   - Mode: Raw JSON  
   - Paste JSON for confirmation page CSS variables:  
     ```
     {
       "bg": "#f7f8fb",
       "card": "#ffffff",
       "muted": "#6b7280",
       "border": "#e5e7eb",
       "radius": "16px",
       "shadow": "0 10px 25px rgba(2, 6, 23, 0.06), 0 4px 10px rgba(2, 6, 23, 0.04)",
       "maxw": "560px"
     }
     ```
   - Connect Page Configuration → Theme Configuration - Feedback Submitted.

10. **Create HTML Node "Feedback Submitted HTML"**  
    - Paste the thank-you HTML content (as provided) into the HTML parameter.  
    - Connect Theme Configuration - Feedback Submitted → Feedback Submitted HTML.

11. **Create Respond to Webhook Node "Respond with Feedback Submitted"**  
    - Respond with: Text  
    - Response Body: `={{ $json.html }}` (output of Feedback Submitted HTML)  
    - Connect Feedback Submitted HTML → Respond with Feedback Submitted.

12. **Publish both webhook nodes ("Input Feedback" and "Post Feedback")** to make them available for HTTP requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                           |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------|
| Collects quick 1–5⭐ feedback + optional message. Saves results to an n8n Data Table including query params. Displays a simple “Thank you” confirmation page. Fully self-contained — no external services or credentials needed. | Workflow Overview, Sticky Note1           |
| Open Form Configuration node to update titles, descriptions, and POST endpoint URL. Optionally adjust theme colors and confirm Data Table schema mapping. | Setup instructions, Sticky Note2          |
| GET `/feedback` loads the HTML form. POST `/feedback` stores data and returns the thank-you page. Query parameters (e.g. userId, source) are saved with feedback as JSON. | Workflow Operation, Sticky Note3          |
| Only submitted fields plus query info are stored. On failure: ensure webhooks are published, POST URL is correct, and Data Table schema matches mapping. For help, contact office@sus-tech.at | Troubleshooting, Sticky Note4              |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.