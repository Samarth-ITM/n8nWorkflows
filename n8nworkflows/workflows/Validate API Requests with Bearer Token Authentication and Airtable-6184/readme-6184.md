Validate API Requests with Bearer Token Authentication and Airtable

https://n8nworkflows.xyz/workflows/validate-api-requests-with-bearer-token-authentication-and-airtable-6184


# Validate API Requests with Bearer Token Authentication and Airtable

### 1. Workflow Overview

This workflow implements a secure API endpoint that validates incoming HTTP requests using Bearer Token authentication, with token and job data stored and validated against Airtable records. It is designed for scenarios where API consumers must authenticate using tokens, and access is restricted based on token validity, expiry, and ownership of specific jobs.

The workflow consists of the following logical blocks:

- **1.1 HTTP Request Reception and Method Handling:** Receives API requests on a webhook path and routes them based on HTTP method.
- **1.2 Request Validation:** Validates the presence and format of the Bearer token and required query parameters.
- **1.3 Token Lookup and Validation:** Queries Airtable for the token record, checks token existence and active status, including expiration.
- **1.4 Job Lookup and Authorization:** Finds the requested job record in Airtable and verifies that the token owner is authorized to access it.
- **1.5 Response Formatting and Delivery:** Formats the job data into a standardized JSON structure and responds to the API caller.
- **1.6 Error Handling:** Returns appropriate HTTP error responses for invalid tokens, expired tokens, unauthorized access, missing jobs, or unsupported HTTP methods.

---

### 2. Block-by-Block Analysis

#### 2.1 HTTP Request Reception and Method Handling

**Overview:**  
This block handles incoming API requests on the `/test-jobs` webhook path and directs them according to HTTP method. Only GET requests are allowed; other methods return a 405 error.

**Nodes Involved:**  
- `GET jobs` (Webhook)  
- `Other methods` (Webhook)  
- `405 Error` (RespondToWebhook)  
- `Sticky Note1`

**Node Details:**  

- **GET jobs**  
  - *Type:* Webhook  
  - *Role:* Entry point for valid GET requests to `/test-jobs`.  
  - *Configuration:* Accepts GET method only, response mode set to respond via response nodes downstream.  
  - *Inputs:* External HTTP requests.  
  - *Outputs:* Connects to the Validator node.  
  - *Edge cases:* Request with missing or invalid query parameters; handled downstream.

- **Other methods**  
  - *Type:* Webhook  
  - *Role:* Entry point for all other HTTP methods (POST, DELETE, HEAD, PATCH, PUT) on `/test-jobs`.  
  - *Configuration:* Accepts specified methods except GET, response mode to response node.  
  - *Outputs:* Connects to `405 Error` node for all method requests.  
  - *Edge cases:* Any unsupported HTTP method triggers a 405 response.

- **405 Error**  
  - *Type:* RespondToWebhook  
  - *Role:* Returns HTTP 405 Method Not Allowed with JSON body prompting use of GET.  
  - *Configuration:* Response code 405, fixed JSON error message.  
  - *Connected from:* `Other methods` node.

- **Sticky Note1**  
  - *Content:* "## HTTP Method handler"  
  - *Context:* Describes this block’s function to handle HTTP methods.

---

#### 2.2 Request Validation

**Overview:**  
Validates the incoming request’s Authorization header and query parameters. Ensures Authorization header exists, starts with "Bearer ", and that exactly one query parameter `job_id` is provided.

**Nodes Involved:**  
- `Validator` (Code)  
- `Valid` (If)  
- `Unauthorized` (RespondToWebhook)

**Node Details:**  

- **Validator**  
  - *Type:* Code  
  - *Role:* Executes JavaScript logic to validate headers and query parameters.  
  - *Configuration:*  
    - Checks presence of Authorization header.  
    - Checks header starts with "Bearer ".  
    - Checks query includes only `job_id`.  
  - *Key Expressions:* Uses `$json.headers.authorization` and `$json.query.job_id`.  
  - *Outputs:* Returns `{success: true}` if validation passes; otherwise returns `{success: false, reason: <error>}`.  
  - *Edge cases:* Missing or malformed header, missing or extra query parameters.

- **Valid**  
  - *Type:* If  
  - *Role:* Checks if validation was successful (`success == true`).  
  - *Connections:*  
    - True: Proceeds to `Get token` node.  
    - False: Sends to `Unauthorized` node.

- **Unauthorized**  
  - *Type:* RespondToWebhook  
  - *Role:* Returns HTTP 401 Unauthorized with a JSON error containing the validation failure reason.  
  - *Configuration:* Response code 401, body uses expression `{{$json.reason}}`.

---

#### 2.3 Token Lookup and Validation

**Overview:**  
Fetches the token record from Airtable based on the token string provided in the Authorization header. Checks if the token exists and is marked as active. If active, proceeds to job lookup; otherwise, returns error responses.

**Nodes Involved:**  
- `Get token` (Airtable)  
- `Token Exists` (If)  
- `Active` (If)  
- `invalid token` (RespondToWebhook)  
- `expired` (RespondToWebhook)  
- `Sticky Note2`

**Node Details:**  

- **Get token**  
  - *Type:* Airtable  
  - *Role:* Searches the "Tokens" table for a record matching the Bearer token string (extracted from header).  
  - *Configuration:* Uses filter formula `{token id} = "<token>"` where `<token>` is dynamically extracted from the Authorization header.  
  - *Credentials:* Uses configured Airtable API credentials.  
  - *Outputs:* Returns matching token record(s).  
  - *Edge cases:* No matching token found.

- **Token Exists**  
  - *Type:* If  
  - *Role:* Checks if Airtable returned a valid token record (`id` exists).  
  - *Connections:*  
    - True: Proceeds to `Active`.  
    - False: Proceeds to `invalid token`.

- **Active**  
  - *Type:* If  
  - *Role:* Checks if the token's `Is Active` field is true.  
  - *Outputs:*  
    - True: Proceeds to `Find job`.  
    - False: Proceeds to `expired`.

- **invalid token**  
  - *Type:* RespondToWebhook  
  - *Role:* Returns HTTP 400 Bad Request with JSON error "Invalid token".  
  - *Configuration:* Response code 400.

- **expired**  
  - *Type:* RespondToWebhook  
  - *Role:* Returns HTTP 401 Unauthorized with JSON error "Token is expired".  
  - *Configuration:* Response code 401.

- **Sticky Note2**  
  - *Content:* "## Database Example\nClone this [Airtable Base](https://airtable.com/appbw5TEhn8xIxxXR/shrN8ve4dfJIXjcAm)"  
  - *Context:* Provides link to example Airtable base used for tokens.

---

#### 2.4 Job Lookup and Authorization

**Overview:**  
Queries Airtable for the job record matching the requested `job_id`. Checks if the job exists and verifies that the token owner is authorized (matches job’s user). If authorized, formats data; otherwise, returns unauthorized or not found errors.

**Nodes Involved:**  
- `Find job` (Airtable)  
- `Job found?` (If)  
- `Owner?` (If)  
- `job not found` (RespondToWebhook)  
- `unauthorized` (RespondToWebhook)

**Node Details:**  

- **Find job**  
  - *Type:* Airtable  
  - *Role:* Searches the "Jobs" table for a record with the given `job_id` from the query parameter.  
  - *Credentials:* Same Airtable credentials as token lookup.  
  - *Outputs:* Returns matching job record(s).  
  - *Edge cases:* No job found.

- **Job found?**  
  - *Type:* If  
  - *Role:* Checks if a job record was found (`id` exists).  
  - *Connections:*  
    - True: Proceeds to `Owner?`.  
    - False: Proceeds to `job not found`.

- **Owner?**  
  - *Type:* If  
  - *Role:* Compares token's `Issued To` user with job's `Users` field to verify ownership/authorization.  
  - *Connections:*  
    - True: Proceeds to `format job`.  
    - False: Proceeds to `unauthorized`.

- **job not found**  
  - *Type:* RespondToWebhook  
  - *Role:* Returns HTTP 404 Not Found with JSON error "Job not found."  
  - *Configuration:* Response code 404.

- **unauthorized**  
  - *Type:* RespondToWebhook  
  - *Role:* Returns HTTP 401 Unauthorized with JSON error "Unauthorized. You don't have the permission to access this job."  
  - *Configuration:* Response code 401.

---

#### 2.5 Response Formatting and Delivery

**Overview:**  
Transforms the retrieved job record into a clean, standardized JSON structure for output and returns it with HTTP 200 OK.

**Nodes Involved:**  
- `format job` (Code)  
- `Return data` (RespondToWebhook)

**Node Details:**  

- **format job**  
  - *Type:* Code  
  - *Role:* Maps Airtable job record fields to a simplified object with specific keys (`id`, `created_time`, `job_title`, etc.).  
  - *Configuration:* Custom JavaScript iterates over a key mapping to rename and include fields.  
  - *Outputs:* JSON object with `job` array containing formatted job info.

- **Return data**  
  - *Type:* RespondToWebhook  
  - *Role:* Returns the formatted job information with HTTP 200 status.  
  - *Configuration:* Response code 200, body set to the `job` array from previous node.

---

#### 2.6 Error Handling and Testing Utilities

**Overview:**  
Additional nodes provide testing utilities and documentation.

**Nodes Involved:**  
- `When clicking ‘Execute workflow’` (ManualTrigger)  
- `Make a request` (HTTPRequest)  
- `Sticky Note` (Manual Test Note)  
- `Sticky Note3` (Documentation)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - *Type:* ManualTrigger  
  - *Role:* Facilitates manual testing by triggering an HTTP request node.  

- **Make a request**  
  - *Type:* HTTPRequest  
  - *Role:* Sends a GET request to the webhook with sample `Authorization` header and `job_id` query parameter for testing.  
  - *Configuration:* URL points to local webhook endpoint, includes query and header parameters.

- **Sticky Note**  
  - *Content:* "## Test the request"  
  - *Context:* Describes test trigger and request nodes.

- **Sticky Note3**  
  - *Content:* Extensive documentation describing the purpose and architecture of the workflow, including benefits and usage notes.  
  - *Context:* Provides background and rationale for the Bearer Token Validation workflow with Airtable.  
  - *Link:* [Creator Instagram](https://n8n.io/creators/islamnazmi/)

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                            | Input Node(s)                                 | Output Node(s)                        | Sticky Note                                               |
|-------------------------|---------------------|-------------------------------------------|----------------------------------------------|-------------------------------------|-----------------------------------------------------------|
| GET jobs                | Webhook             | Entry point for GET requests               | External HTTP request                         | Validator                           |                                                           |
| Other methods           | Webhook             | Entry point for non-GET methods            | External HTTP request                         | 405 Error                          |                                                           |
| 405 Error               | RespondToWebhook    | Returns 405 Method Not Allowed error       | Other methods                                |                                     |                                                           |
| Validator               | Code                | Validates Authorization header & query    | GET jobs                                     | Valid                              |                                                           |
| Valid                   | If                  | Checks if validation passed                 | Validator                                    | Get token / Unauthorized           |                                                           |
| Unauthorized            | RespondToWebhook    | Returns 401 Unauthorized with reason       | Valid (false)                                |                                     |                                                           |
| Get token               | Airtable            | Looks up Bearer token in Airtable          | Valid (true)                                 | Token Exists                      |                                                           |
| Token Exists            | If                  | Checks if token record exists               | Get token                                    | Active / invalid token             |                                                           |
| Active                  | If                  | Checks if token is active                    | Token Exists                                 | Find job / expired                 |                                                           |
| invalid token           | RespondToWebhook    | Returns 400 Invalid token error             | Token Exists (false)                          |                                     |                                                           |
| expired                 | RespondToWebhook    | Returns 401 Token expired error             | Active (false)                               |                                     |                                                           |
| Find job                | Airtable            | Looks up job by job_id in Airtable          | Active (true)                                | Job found?                        |                                                           |
| Job found?              | If                  | Checks if job record exists                  | Find job                                     | Owner? / job not found             |                                                           |
| job not found           | RespondToWebhook    | Returns 404 Job not found error             | Job found? (false)                           |                                     |                                                           |
| Owner?                  | If                  | Checks if token owner matches job user      | Job found? (true)                            | format job / unauthorized          |                                                           |
| unauthorized            | RespondToWebhook    | Returns 401 Unauthorized access error      | Owner? (false)                              |                                     |                                                           |
| format job              | Code                | Formats job record for response              | Owner? (true)                               | Return data                      |                                                           |
| Return data             | RespondToWebhook    | Returns 200 OK with formatted job data      | format job                                  |                                     |                                                           |
| When clicking ‘Execute workflow’ | ManualTrigger | Manual trigger for testing                  |                                              | Make a request                   |                                                           |
| Make a request          | HTTPRequest         | Sends test GET request with token and job_id | When clicking ‘Execute workflow’            |                                     |                                                           |
| Sticky Note             | StickyNote          | Test request description                     |                                              |                                     | ## Test the request                                        |
| Sticky Note1            | StickyNote          | HTTP Method handler description              |                                              |                                     | ## HTTP Method handler                                    |
| Sticky Note2            | StickyNote          | Airtable base example link                    |                                              |                                     | ## Database Example\nClone this [Airtable Base](https://airtable.com/appbw5TEhn8xIxxXR/shrN8ve4dfJIXjcAm) |
| Sticky Note3            | StickyNote          | Workflow documentation and overview          |                                              |                                     | ## Bearer Token Validation... [Creator](https://n8n.io/creators/islamnazmi/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes for HTTP Methods:**

   - Create a Webhook node named `GET jobs`:
     - Path: `test-jobs`
     - HTTP Method: `GET`
     - Response Mode: `Response Node`

   - Create a Webhook node named `Other methods`:
     - Path: `test-jobs`
     - HTTP Methods: `POST, DELETE, HEAD, PATCH, PUT`
     - Response Mode: `Response Node`

2. **Create RespondToWebhook for 405 Error:**

   - Create a node `405 Error`:
     - Response Code: 405
     - Respond With: JSON
     - Response Body: `{ "error": "Use GET request instead" }`
   - Connect outputs of `Other methods` to `405 Error` (all outputs).

3. **Create Request Validation Code Node:**

   - Create a `Validator` node (Code):
     - Use JavaScript code to check:
       - Presence of `Authorization` header.
       - Header starts with "Bearer ".
       - Query contains exactly one parameter: `job_id`.
     - Return JSON with `{success: true}` if valid, else `{success: false, reason: <message>}`.
   - Connect `GET jobs` output to `Validator`.

4. **Create If Node to Check Validation Result:**

   - Create an `If` node `Valid`:
     - Condition: `$json.success` equals `true`.
   - Connect `Validator` output to `Valid`.

5. **Create Unauthorized Respond Node:**

   - Create `Unauthorized` RespondToWebhook node:
     - Response Code: 401
     - Response Body: `{"error": "{{ $json.reason }}"}` (expression)
   - Connect `Valid` false output to `Unauthorized`.

6. **Create Airtable Lookup Node for Token:**

   - Create `Get token` Airtable node:
     - Operation: Search records
     - Base: Connect to your Airtable base (e.g., "Testing Bearer Auth")
     - Table: Tokens
     - Filter Formula: `{token id} = "{{ $json.headers.authorization.replace('Bearer ', '') }}"`
     - Credentials: Airtable API key configured for your base.
   - Connect `Valid` true output to `Get token`.

7. **Create If Node to Check Token Existence:**

   - Create `Token Exists` If node:
     - Condition: Check if `id` field exists in `Get token` output.
   - Connect `Get token` output to `Token Exists`.

8. **Create RespondToWebhook for Invalid Token:**

   - Create `invalid token` node:
     - Response Code: 400
     - Response Body: `{ "success": false, "error": "Invalid token" }`
   - Connect `Token Exists` false output to `invalid token`.

9. **Create If Node to Check Token Active Status:**

   - Create `Active` If node:
     - Condition: Check if token field `Is Active` is `true`.
   - Connect `Token Exists` true output to `Active`.

10. **Create RespondToWebhook for Expired Token:**

    - Create `expired` node:
      - Response Code: 401
      - Response Body: `{ "success": false, "error": "Token is expired" }`
    - Connect `Active` false output to `expired`.

11. **Create Airtable Lookup Node for Job:**

    - Create `Find job` Airtable node:
      - Operation: Search records
      - Base: Same Airtable base
      - Table: Jobs
      - Filter Formula: `{job id} = "{{ $json.query.job_id }}"`
      - Credentials: Same Airtable API key
    - Connect `Active` true output to `Find job`.

12. **Create If Node to Check Job Existence:**

    - Create `Job found?` If node:
      - Condition: Check if `id` exists in `Find job` output.
    - Connect `Find job` output to `Job found?`.

13. **Create RespondToWebhook for Job Not Found:**

    - Create `job not found` node:
      - Response Code: 404
      - Response Body: `{"success": false, "error": "Job not found."}`
    - Connect `Job found?` false output to `job not found`.

14. **Create If Node to Check Ownership:**

    - Create `Owner?` If node:
      - Condition: Compare token's `Issued To[0]` with job's `Users[0]` field.
      - Use expression:
        - Left: `{{$node["Get token"].item.json["Issued To"][0]}}`
        - Right: `{{$node["Find job"].item.json.Users[0]}}`
    - Connect `Job found?` true output to `Owner?`.

15. **Create RespondToWebhook for Unauthorized Access:**

    - Create `unauthorized` node:
      - Response Code: 401
      - Response Body: `{ "success": false, "error": "Unauthorized. You don't have the permission to access this job." }`
    - Connect `Owner?` false output to `unauthorized`.

16. **Create Code Node to Format Job Data:**

    - Create `format job` Code node:
      - JavaScript to map Airtable fields to simplified keys:
        ```
        const record = $json;
        const keyMapping = {
            "id": "id",
            "createdTime": "created_time",
            "Job Title": "job_title",
            "Location": "location",
            "Job Type": "job_type",
            "Company Name": "company_name",
            "Number of Applicants": "applicants",
            "Salary Midpoint": "salary"
        };
        let jobObject = { success: true };
        for (let key in keyMapping) {
            if (record[key] !== undefined) {
                jobObject[keyMapping[key]] = record[key];
            }
        }
        return [{ json: { job: [jobObject] } }];
        ```
    - Connect `Owner?` true output to `format job`.

17. **Create RespondToWebhook to Return Data:**

    - Create `Return data` node:
      - Response Code: 200
      - Response Body: `={{ $json.job }}`
    - Connect `format job` output to `Return data`.

18. **(Optional) Setup Manual Trigger and Test Request:**

    - Create `When clicking ‘Execute workflow’` ManualTrigger node.

    - Create `Make a request` HTTPRequest node:
      - URL: `https://localhost:8080/webhook/test-jobs`
      - Method: GET
      - Query Parameters: `job_id` with a sample value, e.g. `"recfCIKgmo9gZUCjj"`
      - Header Parameters: Authorization: `Bearer abc123`
    - Connect ManualTrigger to HTTPRequest.
    - This setup allows manual workflow execution for testing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| ## Bearer Token Validation<br><br>This n8n template helps you manage and validate tokens easily using n8n as backend and Airtable as token store.<br>- Stores tokens with expiry/status<br>- Validates incoming tokens<br>- Rejects invalid/expired tokens<br>- Can be extended for token management<br><br>Note: Simplified auth example.<br><br>Why use this?<br>- No full backend needed<br>- Modular and ready for SaaS workflows<br><br>Enjoy building secure automations with n8n + Airtable! 🚀<br><br>Built by: [Nazmy](https://n8n.io/creators/islamnazmi/) | Sticky Note3 node content                                                                        |
| ## Database Example<br>Clone this [Airtable Base](https://airtable.com/appbw5TEhn8xIxxXR/shrN8ve4dfJIXjcAm)                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note2 node content                                                                        |
| ## HTTP Method handler                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note1 node content                                                                        |
| ## Test the request                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note node content                                                                        |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.