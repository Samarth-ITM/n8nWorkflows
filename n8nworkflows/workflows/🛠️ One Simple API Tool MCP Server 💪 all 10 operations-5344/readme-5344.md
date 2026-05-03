🛠️ One Simple API Tool MCP Server 💪 all 10 operations

https://n8nworkflows.xyz/workflows/----one-simple-api-tool-mcp-server----all-10-operations-5344


# 🛠️ One Simple API Tool MCP Server 💪 all 10 operations

### 1. Workflow Overview

This workflow, titled "🛠️ One Simple API Tool MCP Server 💪 all 10 operations," serves as a centralized API server that exposes ten different utility operations accessible through a single webhook trigger. It is designed to handle various data processing and retrieval tasks, each implemented as a separate node, but all triggered and coordinated via one entry point.

The workflow logic is organized into the following main blocks:

- **1.1 Input Reception**  
  The workflow starts with a single webhook trigger node that listens for incoming API requests via the MCP (Multi-Channel Platform) trigger.

- **1.2 Operation Dispatch and Execution**  
  The main logic consists of ten independent operation nodes, each implementing a distinct API utility function exposed by the One Simple API Tool integration. These cover operations such as currency conversion, image metadata extraction, social media profile details retrieval, URL expansion, QR code generation, email validation, PDF generation, SEO data retrieval, and screenshot capture.

- **1.3 Documentation and Notes**  
  Several sticky notes are included to provide contextual information or visual separation between groups of operations.

The workflow is structured as a fan-out model from a single trigger to multiple independent operation nodes, allowing it to serve multiple API operations simultaneously depending on the request.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures incoming HTTP requests via a webhook, acting as the single entry point for the entire workflow. It listens for requests that specify which API operation to perform and passes data down to the corresponding operation node.

**Nodes Involved:**  
- One Simple API Tool MCP Server

**Node Details:**

- **One Simple API Tool MCP Server**  
  - Type: MCP Trigger (Webhook) node  
  - Role: Entry point; listens for incoming API calls routed through the MCP trigger webhook.  
  - Configuration: Uses a webhook with ID `a21c11a0-7ab0-47b0-bcfe-0c0640d4d5da` to receive external requests. No additional parameters configured, implying it expects the incoming payload to specify the operation and parameters.  
  - Input: External HTTP request  
  - Output: Data routed to all downstream operation nodes via "ai_tool" output connections  
  - Version: v1 of MCP Trigger node  
  - Possible Failures:  
    - Webhook connectivity issues  
    - Malformed input payloads causing downstream errors  
    - Unauthorized or unauthenticated requests if security is not configured externally  
  - Sub-workflow: None

#### 2.2 Operation Dispatch and Execution

**Overview:**  
This block contains the ten distinct utility operations implemented as separate One Simple API Tool nodes. Each node performs a specific task triggered by the MCP node. They run in parallel, each independently processing requests as dictated by the input data.

**Nodes Involved:**  
- Convert a value between currencies  
- Get image metadata from a URL  
- Get details about an Instagram profile  
- Get details about a Spotify artist  
- Expand a shortened URL  
- Generate a QR code utility  
- Validate an email address  
- Generate PDF  
- Get SEO Data  
- Screenshot

**Node Details:**

- **Convert a value between currencies**  
  - Type: One Simple API Tool node  
  - Role: Converts a monetary value from one currency to another based on input parameters.  
  - Configuration: Default, expects input fields such as amount, source currency, and target currency from the trigger data.  
  - Input: Data from MCP Trigger node  
  - Output: Converted currency value  
  - Possible Failures: Invalid currency codes, network errors, API quota exceeded

- **Get image metadata from a URL**  
  - Type: One Simple API Tool node  
  - Role: Retrieves metadata such as dimensions, format, or size from a given image URL.  
  - Configuration: Expects an image URL input.  
  - Input: Data from MCP Trigger node  
  - Output: Image metadata  
  - Possible Failures: Invalid URL, inaccessible image, unsupported formats

- **Get details about an Instagram profile**  
  - Type: One Simple API Tool node  
  - Role: Fetches public information about an Instagram profile by username or profile URL.  
  - Configuration: Expects Instagram profile identifier.  
  - Input: Data from MCP Trigger node  
  - Output: Profile details (followers, bio, posts, etc.)  
  - Possible Failures: Private or non-existent profiles, API limits

- **Get details about a Spotify artist**  
  - Type: One Simple API Tool node  
  - Role: Retrieves public information about a Spotify artist.  
  - Configuration: Expects artist name or Spotify ID.  
  - Input: Data from MCP Trigger node  
  - Output: Artist metadata (albums, genres, followers)  
  - Possible Failures: Invalid artist ID, API errors

- **Expand a shortened URL**  
  - Type: One Simple API Tool node  
  - Role: Resolves shortened URLs to their original full URLs.  
  - Configuration: Expects a shortened URL input.  
  - Input: Data from MCP Trigger node  
  - Output: Expanded URL  
  - Possible Failures: Invalid or expired shortened URLs, network timeouts

- **Generate a QR code utility**  
  - Type: One Simple API Tool node  
  - Role: Generates a QR code image from provided text or URL.  
  - Configuration: Expects text or URL input to encode into a QR code.  
  - Input: Data from MCP Trigger node  
  - Output: QR code image data or URL  
  - Possible Failures: Invalid input data, image generation errors

- **Validate an email address**  
  - Type: One Simple API Tool node  
  - Role: Checks the validity of an email address format and possibly its deliverability.  
  - Configuration: Expects an email address input.  
  - Input: Data from MCP Trigger node  
  - Output: Validation result (valid/invalid, reason)  
  - Possible Failures: Invalid email format, external validation service errors

- **Generate PDF**  
  - Type: One Simple API Tool node  
  - Role: Converts input content (HTML or text) into a PDF document.  
  - Configuration: Expects content to render.  
  - Input: Data from MCP Trigger node  
  - Output: PDF file or binary data  
  - Possible Failures: Invalid content format, generation errors

- **Get SEO Data**  
  - Type: One Simple API Tool node  
  - Role: Retrieves SEO-related data for a website URL (e.g., rankings, metadata).  
  - Configuration: Expects a URL input.  
  - Input: Data from MCP Trigger node  
  - Output: SEO metrics and metadata  
  - Possible Failures: Invalid URL, API quota limits

- **Screenshot**  
  - Type: One Simple API Tool node  
  - Role: Captures a screenshot of a webpage URL.  
  - Configuration: Expects a URL input.  
  - Input: Data from MCP Trigger node  
  - Output: Screenshot image data  
  - Possible Failures: Invalid URL, timeout, rendering issues

---

### 3. Summary Table

| Node Name                         | Node Type                    | Functional Role                     | Input Node(s)             | Output Node(s)                                  | Sticky Note                       |
|----------------------------------|------------------------------|-----------------------------------|---------------------------|------------------------------------------------|----------------------------------|
| Workflow Overview 0              | Sticky Note                  | Documentation / Overview          |                           |                                                |                                  |
| One Simple API Tool MCP Server   | MCP Trigger                  | Main webhook entry point          |                           | Convert a value between currencies, Get image metadata from a URL, Get details about an Instagram profile, Get details about a Spotify artist, Expand a shortened URL, Generate a QR code utility, Validate an email address, Generate PDF, Get SEO Data, Screenshot |                                  |
| Convert a value between currencies | One Simple API Tool          | Currency conversion operation     | One Simple API Tool MCP Server |                                                |                                  |
| Get image metadata from a URL    | One Simple API Tool          | Image metadata extraction         | One Simple API Tool MCP Server |                                                |                                  |
| Sticky Note 1                   | Sticky Note                  | Visual grouping for first operations |                           |                                                |                                  |
| Get details about an Instagram profile | One Simple API Tool          | Instagram profile data retrieval  | One Simple API Tool MCP Server |                                                |                                  |
| Get details about a Spotify artist | One Simple API Tool          | Spotify artist data retrieval     | One Simple API Tool MCP Server |                                                |                                  |
| Sticky Note 2                   | Sticky Note                  | Visual grouping for social media operations |                           |                                                |                                  |
| Expand a shortened URL           | One Simple API Tool          | URL expansion                    | One Simple API Tool MCP Server |                                                |                                  |
| Generate a QR code utility       | One Simple API Tool          | QR code generation                | One Simple API Tool MCP Server |                                                |                                  |
| Validate an email address        | One Simple API Tool          | Email validation                  | One Simple API Tool MCP Server |                                                |                                  |
| Sticky Note 3                   | Sticky Note                  | Visual grouping for utility operations |                           |                                                |                                  |
| Generate PDF                    | One Simple API Tool          | PDF generation                   | One Simple API Tool MCP Server |                                                |                                  |
| Get SEO Data                   | One Simple API Tool          | SEO data retrieval                | One Simple API Tool MCP Server |                                                |                                  |
| Screenshot                     | One Simple API Tool          | Webpage screenshot capture       | One Simple API Tool MCP Server |                                                |                                  |
| Sticky Note 4                   | Sticky Note                  | Visual grouping for final operations |                           |                                                |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add the MCP Trigger node**:
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`
   - Name: `One Simple API Tool MCP Server`
   - Configuration:  
     - Use default webhook settings or set a specific webhook ID if needed.  
     - No special parameters required.  
   - Purpose: Entry point to receive API requests.

3. **Add One Simple API Tool nodes for each operation** (ten nodes total):

   For each node below:

   - Node Type: `n8n-nodes-base.oneSimpleApiTool`
   - Name: As specified below
   - Credentials: Assign appropriate credentials if required by One Simple API Tool (e.g., API key)
   - Configuration: Leave default; the actual input parameters will be passed dynamically from the MCP Trigger

   Nodes to create:

   1. `Convert a value between currencies`  
   2. `Get image metadata from a URL`  
   3. `Get details about an Instagram profile`  
   4. `Get details about a Spotify artist`  
   5. `Expand a shortened URL`  
   6. `Generate a QR code utility`  
   7. `Validate an email address`  
   8. `Generate PDF`  
   9. `Get SEO Data`  
   10. `Screenshot`

4. **Connect the MCP Trigger node output to each One Simple API Tool node**:
   - Connect the output named `ai_tool` from the MCP Trigger to the input of each One Simple API Tool node.
   - This fan-out design allows the MCP trigger to route requests to any of these operations.

5. **Optional: Add Sticky Note nodes** at positions to visually group or document the operations:
   - Add sticky notes near node groups to separate or label logical groups.
   - Sticky notes are informational and do not affect execution.

6. **Set default values or constraints**:
   - For each One Simple API Tool node, ensure that input parameters (e.g., URLs, currencies, email addresses) are passed properly from the MCP Trigger request payload.
   - Configure error handling or validation as needed in the MCP trigger or downstream nodes, depending on use case.

7. **Activate the workflow** and deploy the webhook:
   - Make sure the webhook URL generated by the MCP Trigger is accessible to clients.
   - Test each operation by sending appropriate requests specifying the desired API operation and parameters.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|-----------------------------------------------------------------------------------------------|----------------------------------------|
| Workflow created and maintained by David Ashby <david.ashby.lds@gmail.com>.                   | Workflow ownership metadata             |
| Workflow uses the One Simple API Tool integration, which allows multiple utility operations via a single trigger. | Integration details                     |
| This design supports scalability by allowing easy addition of new API operations as separate nodes. | Architectural note                      |
| Sticky notes are used to organize and visually separate groups of operations for clarity.     | Visual documentation aid                |

---

**Disclaimer:** The text provided is exclusively generated from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.