Google Spreadsheet to HTML - Variant with JS Function

https://n8nworkflows.xyz/workflows/google-spreadsheet-to-html---variant-with-js-function-1757


# Google Spreadsheet to HTML - Variant with JS Function

### 1. Workflow Overview

This workflow converts data from a Google Spreadsheet into a fully customizable HTML table and serves it via an HTTP endpoint. It is designed for users who want to display spreadsheet data on a web page with a flexible HTML output format.  
The workflow is structured into these logical blocks:

- **1.1 Input Reception:** Listens for incoming HTTP requests via a webhook to trigger the workflow.
- **1.2 Data Retrieval:** Reads data from a specified Google Sheets document using the Google Sheets API.
- **1.3 HTML Construction:** Processes the spreadsheet data and generates a complete HTML page containing a styled table.
- **1.4 HTTP Response:** Returns the generated HTML back to the requester with appropriate content headers.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives the initial HTTP request that triggers the workflow. It acts as the entry point, exposing a public URL endpoint.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook  
    - Role: Receives incoming HTTP GET/POST requests to trigger the workflow.  
    - Configuration:  
      - Path: `bbcd9487-54f9-449d-8246-49f3f61f44fc` (unique endpoint identifier)  
      - Response Mode: `responseNode` (the actual HTTP response is handled by a downstream node)  
      - Options: default (no authentication or additional parameters)  
    - Connections: Outputs to "Read from Google Sheets" node.  
    - Edge Cases & Failures:  
      - Invalid HTTP methods (if restricted)  
      - Endpoint URL changes if workflow is re-imported or webhook reset  
      - Unauthenticated access is possible (no auth configured)  

#### 1.2 Data Retrieval

- **Overview:**  
  Reads the content of a specific Google Sheet document (including headers and data rows) via Google Sheets API using OAuth2 credentials.

- **Nodes Involved:**  
  - Read from Google Sheets

- **Node Details:**

  - **Read from Google Sheets**  
    - Type: Google Sheets  
    - Role: Retrieves rows from the specified Google Sheet.  
    - Configuration:  
      - Sheet ID: `1uFISwZJ1rzkOnOSNocX-_n-ASSAznWGdpcPK3_KCvVo` (spreadsheet identifier)  
      - Options: default, no range specified (reads the entire sheet)  
      - Credentials: Google Sheets OAuth2 (Tom's Google Sheets account)  
    - Connections: Outputs data to "Build HTML" node.  
    - Edge Cases & Failures:  
      - OAuth token expiration or revocation  
      - Sheet ID invalid or access permission errors  
      - Empty sheets or missing header row (may cause errors in the next block)  
      - Network/API timeouts or quota limits  

#### 1.3 HTML Construction

- **Overview:**  
  This block transforms the retrieved spreadsheet data into a complete HTML document. It generates a styled table with Bootstrap CSS and JS for responsiveness.

- **Nodes Involved:**  
  - Build HTML (Function node)

- **Node Details:**

  - **Build HTML**  
    - Type: Function  
    - Role: Constructs an HTML string representing the data as a Bootstrap-styled table.  
    - Configuration:  
      - Custom JavaScript code reads the first row keys as column headers.  
      - Builds a full HTML page string including `<html>`, `<head>`, and `<body>` tags.  
      - Uses Bootstrap 5.2.0 CSS and JS CDN links for styling.  
      - Maps spreadsheet rows to table rows `<tr>`, and columns to table data cells `<td>`.  
      - Returns the HTML in a JSON property named `html`.  
    - Key expressions/variables:  
      - `columns = Object.keys(items[0].json)` to get column headers dynamically  
      - Template literals for HTML construction  
    - Connections: Outputs to "Respond to Webhook" node.  
    - Edge Cases & Failures:  
      - Empty input array (no rows) will cause `items[0]` to be undefined → runtime error  
      - Missing or malformed data cells may render empty cells  
      - Large datasets may result in a very large HTML string affecting performance  
    - Version Requirements: Compatible with n8n Function node version 1+  

#### 1.4 HTTP Response

- **Overview:**  
  Sends the generated HTML content back to the client that triggered the webhook request, setting the correct content type for rendering in browsers.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends response data back to the HTTP client.  
    - Configuration:  
      - Response with: Text  
      - Response body: Expression `={{$json["html"]}}` to send the HTML string from the prior node.  
      - Response headers: Sets `Content-Type` to `text/html; charset=UTF-8` for proper HTML rendering.  
    - Connections: No output connections (terminal node).  
    - Edge Cases & Failures:  
      - If previous node does not provide `html` property, response will be empty or invalid  
      - Large HTML payloads might cause timeout or client-side loading delays  
      - HTTP errors or network interruptions not handled explicitly  

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role           | Input Node(s)         | Output Node(s)         | Sticky Note                                   |
|------------------------|---------------------|--------------------------|-----------------------|------------------------|-----------------------------------------------|
| Webhook                | Webhook             | Entry point, receives HTTP requests | —                     | Read from Google Sheets | To run: visit the URL provided by this webhook node. |
| Read from Google Sheets | Google Sheets       | Fetches spreadsheet data | Webhook               | Build HTML             | Ensure sheet ID is set correctly with header row present. |
| Build HTML             | Function            | Generates HTML page from data | Read from Google Sheets | Respond to Webhook     | Fully customizable HTML output using Bootstrap. |
| Respond to Webhook     | Respond to Webhook  | Sends HTML response to client | Build HTML            | —                      | Sets `Content-Type` header to `text/html`.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node:**  
   - Name: `Webhook`  
   - Path: Choose a unique path (e.g., `bbcd9487-54f9-449d-8246-49f3f61f44fc`)  
   - Response Mode: `responseNode` (to delegate response to another node)  
   - Leave other options default.  

2. **Create a Google Sheets node:**  
   - Name: `Read from Google Sheets`  
   - Operation: Read rows (default)  
   - Set the `Sheet ID` to your target Google Sheet's ID (found in your sheet URL).  
   - Credentials: Link your Google Sheets OAuth2 credentials (ensure valid access).  
   - No specific range needed if your sheet has a header row and data.  

3. **Connect Webhook output to Google Sheets input.**

4. **Create a Function node:**  
   - Name: `Build HTML`  
   - Paste the following JavaScript code into the Function Code area:

```js
const columns = Object.keys(items[0].json);

const html = `
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>HTML Table Example</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.2.0/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-gH2yIJqKdNHPEq0n4Mqa/HGKIhSkIHeL5AyhkYV8i59U5AR6csBvApHHNl/vI1Bx" crossorigin="anonymous">
  </head>
  <body>
    <div class="container">
      <div class="row">
        <div class="col">
          <h1>HTML Table Example</h1>
          <table class="table">
            <thead>
              <tr>
                ${columns.map(e => '<th scope="col">' + e + '</th>').join('\n')}
              </tr>
            </thead>
            <tbody>
              ${items.map(e => '<tr>' + columns.map(ee => '<td>' + e.json[ee] + '</td>').join('\n') + '</tr>').join('\n')}
            </tbody>
          </table>
        </div>
      </div>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.2.0/dist/js/bootstrap.bundle.min.js" integrity="sha384-A3rJD856KowSb7dwlZdYEkO39Gagi7vIsF0jrRAoQmDKKtQBHUuLZ9AsSv4jD4Xa" crossorigin="anonymous"></script>
  </body>
</html>
`;

return [{
  json: {
    html: html
  }
}];
```

5. **Connect Google Sheets output to Function node input.**

6. **Create a Respond to Webhook node:**  
   - Name: `Respond to Webhook`  
   - Response with: Text  
   - Response Body: Expression — use `{{$json["html"]}}` to send the generated HTML.  
   - Response Headers: Add a header entry:  
     - Name: `Content-Type`  
     - Value: `text/html; charset=UTF-8`  

7. **Connect Function node output to Respond to Webhook input.**

8. **Save and activate the workflow.**

9. **Test by visiting the Webhook URL in a browser:**  
   - The browser should display the HTML table generated from your Google Sheet data.  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                     |
|------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is a variant of the [Google Spreadsheet to HTML workflow](https://n8n.io/workflows/1756) with enhanced HTML customization via a Function node. | Original workflow link for comparison and deeper understanding.                                   |
| Bootstrap 5.2.0 CSS and JS are used via CDN for responsive table styling.    | https://getbootstrap.com/docs/5.2/getting-started/introduction/                                   |
| Make sure your Google Sheet includes a header row, as headers are dynamically extracted from the first row keys. | Critical for correct column detection and HTML table header generation.                            |
| When executing the workflow manually in n8n, use the test URL provided by the Webhook node to preview the output. | n8n execution environment detail.                                                                  |