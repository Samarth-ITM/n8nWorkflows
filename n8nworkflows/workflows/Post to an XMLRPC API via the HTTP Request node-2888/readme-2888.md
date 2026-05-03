Post to an XMLRPC API via the HTTP Request node

https://n8nworkflows.xyz/workflows/post-to-an-xmlrpc-api-via-the-http-request-node-2888


# Post to an XMLRPC API via the HTTP Request node

### 1. Workflow Overview

This workflow demonstrates how to interact with XML-RPC APIs using the generic HTTP Request node in n8n, specifically by posting content to a WordPress blog. It serves as a practical example and workaround when dedicated n8n integrations (like the WordPress node) are unavailable or malfunctioning.

The workflow is logically divided into three main blocks:

- **1.1 Settings Initialization:** Defines and sets all necessary configuration parameters such as the WordPress blog URL, user credentials, and post content.
- **1.2 XML Payload Preparation and Request Sending:** Constructs a properly escaped XML payload for the XML-RPC API call and sends it via an HTTP POST request.
- **1.3 Response Handling and Conditional Outcome:** Parses the XML response into JSON and uses conditional logic to determine if the post was successful, routing the flow accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Settings Initialization

- **Overview:**  
  This block collects and sets all user-configurable parameters required for the XML-RPC request, including WordPress URL, username, application password, and the content to be published.

- **Nodes Involved:**  
  - ManualTrigger  
  - Settings  
  - Sticky Note (Settings)

- **Node Details:**

  - **ManualTrigger**  
    - *Type & Role:* Trigger node to manually start the workflow.  
    - *Configuration:* Default manual trigger with no parameters.  
    - *Connections:* Output to the Settings node.  
    - *Failure Modes:* None typical; manual initiation.  
    - *Version:* 1.

  - **Settings**  
    - *Type & Role:* Set node used to define static variables for the workflow.  
    - *Configuration:* Assigns string values for:  
      - `wordpressUrl` (e.g., "YOURBLOG.wordpress.com")  
      - `wordpressUsername` (e.g., "YourUserName")  
      - `wordpressApplicationPassword` (your app-specific password)  
      - `contentTitle` (post title)  
      - `contentText` (post content)  
    - *Connections:* Input from ManualTrigger; output to PrepareXML node.  
    - *Failure Modes:* Misconfiguration or missing credentials will cause downstream failures.  
    - *Version:* 3.4.

  - **Sticky Note (Settings)**  
    - *Role:* Visual annotation grouping the Settings block.  
    - *Content:* "## Settings"  
    - *Position:* Near the Settings and ManualTrigger nodes.

---

#### 2.2 XML Payload Preparation and Request Sending

- **Overview:**  
  This block prepares the XML payload with proper escaping of special characters, then sends the HTTP POST request to the WordPress XML-RPC endpoint.

- **Nodes Involved:**  
  - PrepareXML (Code node)  
  - PostRequest (HTTP Request node)  
  - Sticky Note (Request Sending)

- **Node Details:**

  - **PrepareXML**  
    - *Type & Role:* Code node that builds the XML request body dynamically.  
    - *Configuration:*  
      - Runs once per item.  
      - Extracts variables from input JSON (wordpressUsername, wordpressApplicationPassword, contentTitle, contentText).  
      - Defines a helper function `escapeXml` to escape XML special characters (`<`, `>`, `&`, `'`, `"`).  
      - Constructs the XML payload string for the `wp.newPost` method call with parameters: blogId (0), username, password, post title, post content, and published status (1).  
      - Adds the XML string as a new field `xmlRequestBody` in the item JSON.  
    - *Key Expressions:* Uses template literals and escaped variables for XML construction.  
    - *Connections:* Input from Settings; output to PostRequest.  
    - *Failure Modes:*  
      - Errors in escaping or malformed XML can cause request failure.  
      - Missing or invalid credentials will cause authentication failures downstream.  
    - *Version:* 2.

  - **PostRequest**  
    - *Type & Role:* HTTP Request node sending the XML-RPC POST request.  
    - *Configuration:*  
      - URL: `https://{{ wordpressUrl }}/xmlrpc.php` (dynamically from Settings).  
      - Method: POST.  
      - Body: Raw content set to the XML payload from `xmlRequestBody`.  
      - Headers: `Content-Type: text/xml`.  
      - Sends headers and raw body as `text/xml`.  
    - *Connections:* Input from PrepareXML; output to HandleResponse.  
    - *Failure Modes:*  
      - Network errors, timeouts, or incorrect URL.  
      - Authentication failure if credentials are invalid.  
      - HTTP errors (4xx, 5xx).  
    - *Version:* 4.2.

  - **Sticky Note (Request Sending)**  
    - *Role:* Visual annotation for this block.  
    - *Content:* "## Request Sending"  
    - *Position:* Near PrepareXML and PostRequest nodes.

---

#### 2.3 Response Handling and Conditional Outcome

- **Overview:**  
  This block parses the XML response from the WordPress server into JSON, then uses a conditional node to check if the post was successful. Based on the result, it routes to success or error handling nodes.

- **Nodes Involved:**  
  - HandleResponse (XML node)  
  - IsSuccessful (If node)  
  - Success (NoOp node)  
  - Error (NoOp node)  
  - Sticky Note2 (Response Handling)

- **Node Details:**

  - **HandleResponse**  
    - *Type & Role:* XML node that converts the XML response into JSON format for easier processing.  
    - *Configuration:* Default XML to JSON conversion.  
    - *Connections:* Input from PostRequest; output to IsSuccessful.  
    - *Failure Modes:*  
      - Malformed XML response may cause parsing errors.  
    - *Version:* 1.

  - **IsSuccessful**  
    - *Type & Role:* Conditional (If) node that checks if the XML-RPC call succeeded.  
    - *Configuration:*  
      - Condition: Checks if the JSON path `$json.methodResponse.params.param.value` exists.  
      - If exists → success path; else → error path.  
      - Uses loose type validation and case-sensitive string existence check.  
    - *Connections:* Input from HandleResponse; outputs to Success (true) and Error (false).  
    - *Failure Modes:*  
      - Unexpected response structure may cause false negatives.  
    - *Version:* 2.2.

  - **Success**  
    - *Type & Role:* NoOp node representing successful completion.  
    - *Configuration:* None.  
    - *Connections:* Input from IsSuccessful (true).  
    - *Failure Modes:* None.

  - **Error**  
    - *Type & Role:* NoOp node representing failure outcome.  
    - *Configuration:* None.  
    - *Connections:* Input from IsSuccessful (false).  
    - *Failure Modes:* None.

  - **Sticky Note2 (Response Handling)**  
    - *Role:* Visual annotation for this block.  
    - *Content:* "## Response Handling"  
    - *Position:* Near HandleResponse, IsSuccessful, Success, and Error nodes.

---

### 3. Summary Table

| Node Name     | Node Type          | Functional Role                  | Input Node(s)       | Output Node(s)          | Sticky Note                      |
|---------------|--------------------|--------------------------------|---------------------|-------------------------|---------------------------------|
| ManualTrigger | Manual Trigger     | Start workflow manually         | -                   | Settings                | ## Settings                     |
| Settings      | Set                | Define WordPress and post data  | ManualTrigger       | PrepareXML              | ## Settings                     |
| PrepareXML    | Code               | Build and escape XML payload    | Settings            | PostRequest             | ## Request Sending              |
| PostRequest   | HTTP Request       | Send XML-RPC POST request       | PrepareXML          | HandleResponse          | ## Request Sending              |
| HandleResponse| XML                | Convert XML response to JSON    | PostRequest         | IsSuccessful            | ## Response Handling            |
| IsSuccessful  | If                 | Check if post was successful    | HandleResponse      | Success, Error          | ## Response Handling            |
| Success       | NoOp               | Success path                    | IsSuccessful (true) | -                       | ## Response Handling            |
| Error         | NoOp               | Error path                      | IsSuccessful (false)| -                       | ## Response Handling            |
| Sticky Note   | Sticky Note        | Visual annotation for Settings  | -                   | -                       | ## Settings                     |
| Sticky Note2  | Sticky Note        | Visual annotation for Response  | -                   | -                       | ## Response Handling            |
| Sticky Note5  | Sticky Note        | Visual annotation for Request   | -                   | -                       | ## Request Sending              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow.  
   - No special configuration needed.

2. **Create Set Node Named "Settings"**  
   - Type: Set  
   - Connect input from Manual Trigger.  
   - Add the following string parameters:  
     - `wordpressUrl`: e.g., `"YOURBLOG.wordpress.com"`  
     - `wordpressUsername`: e.g., `"YourUserName"`  
     - `wordpressApplicationPassword`: your WordPress app password  
     - `contentTitle`: e.g., `"This is a demo title"`  
     - `contentText`: e.g., `"This is the main text."`  
   - Ensure all values are set as strings.

3. **Create Code Node Named "PrepareXML"**  
   - Type: Code  
   - Connect input from Settings node.  
   - Set mode to "Run Once For Each Item".  
   - Paste the following JavaScript code (adapted for n8n environment):

```javascript
const input = $json;

const username = input.wordpressUsername;
const password = input.wordpressApplicationPassword;
const title = input.contentTitle;
const text = input.contentText;

const blogId = 0;
const published = 1; // 0 = draft, 1 = published

function escapeXml(unsafe) {
  return unsafe.replace(/[<>&'"]/g, (c) => {
    switch (c) {
      case '<': return '&lt;';
      case '>': return '&gt;';
      case '&': return '&amp;';
      case '\'': return '&apos;';
      case '"': return '&quot;';
      default: return c;
    }
  });
}

const titleEscaped = escapeXml(title);
const textEscaped = escapeXml(text);

const xmlData = `<?xml version="1.0"?>
<methodCall>
  <methodName>wp.newPost</methodName>
  <params>
    <param>
      <value><string>${blogId}</string></value>
    </param>
    <param>
      <value><string>${username}</string></value>
    </param>
    <param>
      <value><string>${password}</string></value>
    </param>
    <param>
      <value>
        <struct>
          <member>
            <name>post_title</name>
            <value><string>${titleEscaped}</string></value>
          </member>
          <member>
            <name>post_content</name>
            <value><string>${textEscaped}</string></value>
          </member>
        </struct>
      </value>
    </param>
    <param>
      <value><boolean>${published}</boolean></value>
    </param>
  </params>
</methodCall>`;

$input.item.json.xmlRequestBody = xmlData;

return $input.item;
```

4. **Create HTTP Request Node Named "PostRequest"**  
   - Type: HTTP Request  
   - Connect input from PrepareXML.  
   - Set URL to: `https://{{ $json.wordpressUrl }}/xmlrpc.php` (use expression editor).  
   - Method: POST  
   - Body Content Type: Raw  
   - Raw Content Type: `text/xml`  
   - Body: Use expression to pass `{{ $json.xmlRequestBody }}`  
   - Headers: Add header `Content-Type: text/xml`  
   - Enable "Send Headers" and "Send Body".

5. **Create XML Node Named "HandleResponse"**  
   - Type: XML  
   - Connect input from PostRequest.  
   - Use default settings to convert XML response to JSON.

6. **Create If Node Named "IsSuccessful"**  
   - Type: If  
   - Connect input from HandleResponse.  
   - Configure condition:  
     - Check if the JSON path `$json.methodResponse.params.param.value` exists (use expression).  
     - Condition type: "Exists" (string operation).  
     - Case sensitive: true.  
     - Loose type validation enabled.  
   - True output: Success node  
   - False output: Error node

7. **Create NoOp Node Named "Success"**  
   - Type: NoOp  
   - Connect input from IsSuccessful (true path).

8. **Create NoOp Node Named "Error"**  
   - Type: NoOp  
   - Connect input from IsSuccessful (false path).

9. **Add Sticky Notes for Clarity (Optional)**  
   - Near Settings and ManualTrigger: "## Settings"  
   - Near PrepareXML and PostRequest: "## Request Sending"  
   - Near HandleResponse, IsSuccessful, Success, and Error: "## Response Handling"

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Ensure your WordPress user has an [application-specific password](https://wordpress.com/support/security/two-step-authentication/application-specific-passwords/) for XML-RPC authentication. | WordPress security documentation on app passwords.                                                      |
| This workflow is a workaround for when the dedicated WordPress node in n8n is unavailable or broken.      | Useful for maintaining WordPress integration continuity.                                                |
| XML escaping is critical to avoid malformed requests and injection issues; the helper function handles this. | Important for XML-RPC payload construction.                                                             |
| The XML-RPC endpoint URL is always `https://<your-blog-url>/xmlrpc.php`.                                  | Standard WordPress XML-RPC endpoint.                                                                     |

---

This documentation enables users and automation agents to understand, reproduce, and modify the workflow confidently, anticipating potential failure points and integration nuances.