Learn Secure Webhook APIs with Authentication and Supabase Integration

https://n8nworkflows.xyz/workflows/learn-secure-webhook-apis-with-authentication-and-supabase-integration-8258


# Learn Secure Webhook APIs with Authentication and Supabase Integration

### 1. Workflow Overview

This workflow demonstrates how to implement secure webhook APIs with various HTTP methods, authentication, and integration with a Supabase database. It targets use cases involving CRUD (Create, Read, Update, Delete) operations via webhooks, showing how to receive web requests, process data, interact with a Supabase table, and respond appropriately.

The workflow is logically divided into the following blocks:

- **1.1 Webhook Entry Points:** Multiple webhook nodes handling different HTTP methods (GET, POST, PATCH, DELETE, PUT, HEAD) on the same path but with distinct configurations.
- **1.2 Data Processing and Transformation:** Nodes that prepare or adjust incoming data from webhook payloads before database operations.
- **1.3 Supabase Database Operations:** Nodes that perform CRUD operations on the "demo_contacts" table in Supabase.
- **1.4 Webhook Responses:** Nodes that send back data or acknowledgments to the webhook callers after processing.
- **1.5 Documentation and Explanation Notes:** Sticky notes providing context, explanations, and helpful resources about webhooks, authentication, and response types.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Entry Points

**Overview:**  
This block exposes multiple HTTP endpoints (webhooks) on the same URL path to handle different HTTP methods (GET, POST, PATCH, DELETE, PUT, HEAD). Each webhook is configured to receive requests with different semantics and respond accordingly.

**Nodes Involved:**  
- Webhook (GET)  
- Webhook1 (POST)  
- Webhook3 (PATCH)  
- Webhook4 (DELETE)  
- Webhook2 (PUT)  
- Webhook5 (HEAD)

**Node Details:**

- **Webhook (GET)**
  - Type: Webhook node, entry point for HTTP GET requests.
  - Config: Path “07aaa04d-6c73-416f-82e2-1e6ededeacc4”, default GET method.
  - Response Mode: Respond using a response node downstream.
  - Connections: Output to “Get a row” node.
  - Edge Cases: Missing query parameters (e.g., email), unauthorized access if auth is configured.

- **Webhook1 (POST)**
  - Type: Webhook node for HTTP POST requests.
  - Config: Same path, POST method explicitly set.
  - Response Mode: Respond using downstream node.
  - Connections: Output to “Edit Fields” node.
  - Edge Cases: Malformed JSON bodies, missing required fields.

- **Webhook3 (PATCH)**
  - Type: Webhook node for HTTP PATCH requests.
  - Config: Same path, PATCH method, with “Allowed Origins” set to "*" (CORS open), response mode “streaming”.
  - Connections: Output to “Edit Fields1” node.
  - Edge Cases: Partial updates without required fields, CORS related issues if front-end misconfigured.

- **Webhook4 (DELETE)**
  - Type: Webhook node for HTTP DELETE requests.
  - Config: Same path, DELETE method.
  - Response Mode: Default (responseNode).
  - Connections: Output to “Delete a row” node.
  - Edge Cases: Missing or invalid id parameter in query, repeated deletes.

- **Webhook2 (PUT)**
  - Type: Webhook node for HTTP PUT requests.
  - Config: Same path, PUT method.
  - No downstream connections (possibly incomplete or for demonstration).
  - Edge Cases: Full replacement data missing or malformed.

- **Webhook5 (HEAD)**
  - Type: Webhook node for HTTP HEAD requests.
  - Config: Same path, HEAD method.
  - No downstream connections (likely for header checks).
  - Edge Cases: No response body by design; may be used for health checks.

---

#### 1.2 Data Processing and Transformation

**Overview:**  
Nodes that manipulate or set the data received from webhooks before sending it to Supabase for database operations.

**Nodes Involved:**  
- Edit Fields  
- Edit Fields1

**Node Details:**

- **Edit Fields**
  - Type: Set node.
  - Role: Converts the incoming webhook POST body into JSON output suitable for Supabase insertion.
  - Configuration: Raw mode, outputs JSON from `$json.body` (the POST payload).
  - Input: From Webhook1 (POST).
  - Output: To “Create a row” node.
  - Edge Cases: Invalid JSON bodies or unexpected payload structures.

- **Edit Fields1**
  - Type: Set node.
  - Role: Similar to Edit Fields but for PATCH requests, prepares partial update data.
  - Configuration: Raw mode, outputs JSON from `$json.body`.
  - Input: From Webhook3 (PATCH).
  - Output: To “Update a row” node.
  - Edge Cases: Partial data missing required fields for update.

---

#### 1.3 Supabase Database Operations

**Overview:**  
Nodes interacting with Supabase to perform CRUD operations on the “demo_contacts” table based on the webhook input.

**Nodes Involved:**  
- Get a row  
- Create a row  
- Update a row  
- Delete a row

**Node Details:**

- **Get a row**
  - Type: Supabase node.
  - Operation: Get row(s) from “demo_contacts” table.
  - Filter: Uses query parameter `email` from the webhook GET request to filter rows.
  - Credentials: Supabase API credentials configured.
  - Input: From Webhook (GET).
  - Output: To “Respond to Webhook”.
  - Edge Cases: Email not found, connection or auth errors to Supabase.

- **Create a row**
  - Type: Supabase node.
  - Operation: Insert new row into “demo_contacts”.
  - Data: Automatically maps input JSON fields except `id`.
  - Credentials: Same Supabase API.
  - Input: From “Edit Fields” (POST processing).
  - Output: To “Respond to Webhook1”.
  - Edge Cases: Duplicate entries, invalid data violating DB constraints.

- **Update a row**
  - Type: Supabase node.
  - Operation: Update row in “demo_contacts” matching `id` from input JSON.
  - Filter: By `id` field equality.
  - Data: Auto maps input JSON fields.
  - Credentials: Supabase API.
  - Input: From “Edit Fields1” (PATCH processing).
  - Output: To “Respond to Webhook2”.
  - Edge Cases: Non-existent `id`, partial update conflicts, invalid data.

- **Delete a row**
  - Type: Supabase node.
  - Operation: Delete row matching `id` passed as query parameter.
  - Filter: By `id` equality.
  - Credentials: Supabase API.
  - Input: From Webhook4 (DELETE).
  - Output: No response node connected downstream (implicit or default response).
  - Edge Cases: Missing or invalid `id`, deleting non-existent rows.

---

#### 1.4 Webhook Responses

**Overview:**  
Nodes that send responses back to the webhook callers after Supabase operations complete.

**Nodes Involved:**  
- Respond to Webhook  
- Respond to Webhook1  
- Respond to Webhook2

**Node Details:**

- **Respond to Webhook**
  - Type: Respond to Webhook node.
  - Purpose: Sends response data after retrieving a row (GET).
  - Configuration: Responds with all incoming items.
  - Input: From “Get a row”.
  - Output: None (terminates the chain).
  - Edge Cases: Empty results, response formatting errors.

- **Respond to Webhook1**
  - Type: Respond to Webhook node.
  - Purpose: Sends response data after creating a row (POST).
  - Configuration: Responds with all incoming items.
  - Input: From “Create a row”.
  - Output: None.
  - Edge Cases: Could fail if create operation failed silently.

- **Respond to Webhook2**
  - Type: Respond to Webhook node.
  - Purpose: Sends response data after updating a row (PATCH).
  - Configuration: Responds with all incoming items.
  - Input: From “Update a row”.
  - Output: None.
  - Edge Cases: Update failure or empty response.

---

#### 1.5 Documentation and Explanation Notes

**Overview:**  
Sticky notes that provide users with detailed explanations about webhooks, HTTP methods, authentication options, response types, and useful external resources.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6  
- Sticky Note7  
- Sticky Note8  
- Sticky Note10  
- Sticky Note11  
- Sticky Note12

**Node Details:**

- **Sticky Note** (top-left)
  - Content: Explains what a webhook is in n8n, security options, response modes.
- **Sticky Note1 to Sticky Note6**
  - Each defines one HTTP method (GET, POST, PUT, PATCH, DELETE, HEAD) and their typical use cases.
- **Sticky Note7**
  - Details webhook authentication types: Basic, Header, JWT, plus IP whitelist and CORS.
- **Sticky Note8**
  - Describes webhook response timing options in n8n.
- **Sticky Note10**
  - Credits and author information with links.
- **Sticky Note11**
  - Embedded video link to “n8n Webhooks 101 | Secure Them the Right Way”.
- **Sticky Note12**
  - Blog post link: “n8n Webhooks: A Beginner’s Guide (with Security Built-In)”.

---

### 3. Summary Table

| Node Name       | Node Type            | Functional Role                     | Input Node(s)    | Output Node(s)      | Sticky Note                                                                                                    |
|-----------------|----------------------|-----------------------------------|------------------|---------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook         | Webhook              | Receives GET webhook requests     | -                | Get a row           | ## GET → “Retrieve data without making changes. Think queries or health checks.”                               |
| Webhook1        | Webhook              | Receives POST webhook requests    | -                | Edit Fields         | ## POST → “Send new data/events. Most webhooks from apps use POST.”                                           |
| Webhook2        | Webhook              | Receives PUT webhook requests     | -                | -                   | ## PUT → “Replace a whole resource with new data. Idempotent.”                                                |
| Webhook3        | Webhook              | Receives PATCH webhook requests   | -                | Edit Fields1        | ## PATCH → “Update part of a resource. Send only the fields that changed.”                                     |
| Webhook4        | Webhook              | Receives DELETE webhook requests  | -                | Delete a row        | ## DELETE → “Remove a resource. Repeating the call has the same result.”                                       |
| Webhook5        | Webhook              | Receives HEAD webhook requests    | -                | -                   | ## HEAD → “Like GET but no body — used for checks/headers only.”                                              |
| Get a row       | Supabase             | Retrieves row by email             | Webhook          | Respond to Webhook   |                                                                                                               |
| Edit Fields     | Set                  | Prepares POST data for create     | Webhook1         | Create a row        |                                                                                                               |
| Create a row    | Supabase             | Inserts new row in Supabase       | Edit Fields      | Respond to Webhook1  |                                                                                                               |
| Respond to Webhook | Respond to Webhook  | Sends GET response                | Get a row        | -                   |                                                                                                               |
| Respond to Webhook1 | Respond to Webhook | Sends POST response               | Create a row     | -                   |                                                                                                               |
| Edit Fields1    | Set                  | Prepares PATCH data for update    | Webhook3         | Update a row        |                                                                                                               |
| Update a row    | Supabase             | Updates row in Supabase           | Edit Fields1     | Respond to Webhook2  |                                                                                                               |
| Respond to Webhook2 | Respond to Webhook  | Sends PATCH response              | Update a row     | -                   |                                                                                                               |
| Delete a row    | Supabase             | Deletes row by id                 | Webhook4         | -                   |                                                                                                               |
| Sticky Note     | Sticky Note           | Explanation of webhooks           | -                | -                   | Explains n8n webhooks, security, and response modes.                                                         |
| Sticky Note1    | Sticky Note           | HTTP GET explanation              | -                | -                   | ## GET → “Retrieve data without making changes. Think queries or health checks.”                               |
| Sticky Note2    | Sticky Note           | HTTP POST explanation             | -                | -                   | ## POST → “Send new data/events. Most webhooks from apps use POST.”                                           |
| Sticky Note3    | Sticky Note           | HTTP PUT explanation              | -                | -                   | ## PUT → “Replace a whole resource with new data. Idempotent.”                                                |
| Sticky Note4    | Sticky Note           | HTTP PATCH explanation            | -                | -                   | ## PATCH → “Update part of a resource. Send only the fields that changed.”                                     |
| Sticky Note5    | Sticky Note           | HTTP DELETE explanation           | -                | -                   | ## DELETE → “Remove a resource. Repeating the call has the same result.”                                       |
| Sticky Note6    | Sticky Note           | HTTP HEAD explanation             | -                | -                   | ## HEAD → “Like GET but no body — used for checks/headers only.”                                              |
| Sticky Note7    | Sticky Note           | Webhook Auth types explanation    | -                | -                   | Describes Basic, Header, JWT auth, IP whitelist, CORS.                                                        |
| Sticky Note8    | Sticky Note           | Webhook response types explanation| -                | -                   | Explains immediate, last node finish, and Respond to Webhook response modes.                                  |
| Sticky Note10   | Sticky Note           | Author credits and support links  | -                | -                   | Built by Wayne Simpson at nocodecreative.io, coffee link included.                                            |
| Sticky Note11   | Sticky Note           | Video link                       | -                | -                   | Link to video: “n8n Webhooks 101 | Secure Them the Right Way” Youtube video.                                  |
| Sticky Note12   | Sticky Note           | Blog post link                   | -                | -                   | Blog post: “n8n Webhooks: A Beginner’s Guide (with Security Built-In)” at nocodecreative.io blog.              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes for Each HTTP Method:**

   - Add a **Webhook** node, name "Webhook"  
     - Path: `07aaa04d-6c73-416f-82e2-1e6ededeacc4`  
     - HTTP Method: GET (default)  
     - Response Mode: Use response node downstream  
   
   - Add **Webhook1**, name it "Webhook1"  
     - Same path  
     - HTTP Method: POST  
     - Response Mode: Use response node downstream  
   
   - Add **Webhook3**, name it "Webhook3"  
     - Same path  
     - HTTP Method: PATCH  
     - Response Mode: Streaming  
     - Allowed Origins: `*` (for CORS)  
   
   - Add **Webhook4**, name it "Webhook4"  
     - Same path  
     - HTTP Method: DELETE  
     - Response Mode: Use response node downstream  
   
   - Add **Webhook2**, name it "Webhook2"  
     - Same path  
     - HTTP Method: PUT  
   
   - Add **Webhook5**, name it "Webhook5"  
     - Same path  
     - HTTP Method: HEAD  

2. **Setup Supabase Credentials:**

   - Configure Supabase API credentials with access to your Supabase project.
   - Ensure you have a table named `demo_contacts` with at least fields including `id`, `email`, and other contact info.

3. **Build GET Flow:**

   - Connect "Webhook" output to a **Supabase** node named "Get a row"  
     - Operation: Get  
     - Table: `demo_contacts`  
     - Filter: `email` equals `{{$json.query.email}}`  
   - Connect "Get a row" output to a **Respond to Webhook** node  
     - Respond with all incoming items

4. **Build POST Flow:**

   - Connect "Webhook1" output to a **Set** node named "Edit Fields"  
     - Mode: Raw  
     - JSON output: `{{$json.body}}` (extract POST body)  
   - Connect "Edit Fields" output to a **Supabase** node named "Create a row"  
     - Operation: Insert  
     - Table: `demo_contacts`  
     - Data to send: Auto map input data (ignore `id`)  
   - Connect "Create a row" output to a **Respond to Webhook** node named "Respond to Webhook1"  
     - Respond with all incoming items

5. **Build PATCH Flow:**

   - Connect "Webhook3" output to a **Set** node named "Edit Fields1"  
     - Mode: Raw  
     - JSON output: `{{$json.body}}` (extract PATCH body)  
   - Connect "Edit Fields1" output to a **Supabase** node named "Update a row"  
     - Operation: Update  
     - Table: `demo_contacts`  
     - Filter: `id` equals `{{$json.id}}`  
     - Data to send: Auto map input data  
   - Connect "Update a row" output to a **Respond to Webhook** node named "Respond to Webhook2"  
     - Respond with all incoming items

6. **Build DELETE Flow:**

   - Connect "Webhook4" output to a **Supabase** node named "Delete a row"  
     - Operation: Delete  
     - Table: `demo_contacts`  
     - Filter: `id` equals `{{$json.query.id}}`  
   - Optionally add a Respond to Webhook node here to confirm deletion.

7. **Optional PUT and HEAD Flows:**

   - Webhook2 (PUT) and Webhook5 (HEAD) can be configured similarly if needed; currently unconnected.

8. **Add Sticky Notes for Documentation:**

   - Add sticky notes with detailed explanations about webhooks, HTTP methods, auth types, response modes, and links to video and blog posts.
   - Position for clarity near relevant nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                               |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| What is a webhook (in n8n)? Explanation of webhooks, security features, and response modes.                                       | Sticky Note top-left in workflow                                                                              |
| HTTP methods explained (GET, POST, PUT, PATCH, DELETE, HEAD) with typical use cases for each.                                      | Sticky Notes 1 to 6                                                                                           |
| Webhook Authentication Types in n8n: Basic, Header, JWT, with IP Whitelist and CORS options.                                       | Sticky Note7                                                                                                  |
| Webhook Response Types in n8n: immediately, when last node finishes, Respond to Webhook node.                                      | Sticky Note8                                                                                                  |
| Author credit: Wayne Simpson at nocodecreative.io with a coffee donation link.                                                    | Sticky Note10                                                                                                 |
| Video: “n8n Webhooks 101 | Secure Them the Right Way” on YouTube.                                                                  | Sticky Note11 — [Watch Video](https://www.youtube.com/watch?v=o6F36xsiuBk)                                    |
| Blog post: “n8n Webhooks: A Beginner’s Guide (with Security Built-In)” on nocodecreative.io blog.                                 | Sticky Note12 — [Read Blog](https://blog.nocodecreative.io/n8n-webhooks-a-beginners-guide-with-security-built-in/) |

---

This comprehensive documentation enables users and automation agents to understand, reproduce, and extend the workflow with clear knowledge of how webhook methods, security, data processing, and Supabase integration are orchestrated.