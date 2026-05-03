Secure GitHub Webhooks with HMAC256 Signature Validation

https://n8nworkflows.xyz/workflows/secure-github-webhooks-with-hmac256-signature-validation-8906


# Secure GitHub Webhooks with HMAC256 Signature Validation

---

### 1. Workflow Overview

This workflow secures GitHub webhook requests by validating their HMAC256 signatures to ensure authenticity and data integrity. It targets developers and DevOps engineers who want to protect webhook endpoints from tampering or unauthorized access.

The workflow’s logic is grouped into three main blocks:

- **1.1 Receive GitHub Webhook**: Listens for incoming GitHub webhook POST requests.
- **1.2 HMAC256 Signature Validation**: Computes an HMAC256 hash of the webhook payload using a secret and compares it against the signature GitHub provides in the request header.
- **1.3 Response and Further Processing**: Returns appropriate HTTP response codes based on validation results, optionally stops the workflow on failure, or continues with additional GitHub API calls for further automation.

---

### 2. Block-by-Block Analysis

#### 1.1 Receive GitHub Webhook

**Overview:**  
This block accepts incoming POST webhook requests from GitHub on a configured endpoint path and serves as the workflow’s entry point.

**Nodes Involved:**  
- `GitHub Webhook`

**Node Details:**  
- **GitHub Webhook**  
  - *Type:* Webhook  
  - *Role:* Receives POST requests from GitHub at path `/github-test`.  
  - *Configuration:* HTTP method set to POST; response mode configured to be handled by downstream response nodes rather than immediate response.  
  - *Expressions:* None.  
  - *Inputs:* External HTTP request from GitHub.  
  - *Outputs:* Passes webhook payload and headers to next node (`Compute HMAC256`).  
  - *Edge cases:*  
    - Request not matching HTTP method or path will not trigger workflow.  
    - Malformed requests will still be processed but may fail validation later.  
  - *Version:* 2.1  

---

#### 1.2 HMAC256 Signature Validation

**Overview:**  
This block computes the HMAC256 hash of the webhook body using a secret, compares it against the signature sent by GitHub in the `x-hub-signature-256` header, and determines if the request is authentic.

**Nodes Involved:**  
- `Compute HMAC256`  
- `Validate HMAC256`

**Node Details:**  
- **Compute HMAC256**  
  - *Type:* Crypto (HMAC)  
  - *Role:* Calculates the HMAC256 hash of the webhook payload using a configured secret.  
  - *Configuration:*  
    - Hash type: SHA256  
    - Action: HMAC  
    - Value: JSON stringified webhook body (`{{ JSON.stringify($json.body) }}`)  
    - Secret: Set to the GitHub webhook secret (`your_github_secret_here` — must be replaced with actual secret)  
    - Output property: `signature-256`  
  - *Expressions:* Uses n8n expression to capture the request body as string and to set secret.  
  - *Inputs:* Webhook payload and headers from `GitHub Webhook` node.  
  - *Outputs:* Passes computed signature in `$json['signature-256']` to next node.  
  - *Edge cases:*  
    - Secret mismatch will cause signature mismatch.  
    - Empty or malformed payload may produce incorrect signature.  
  - *Version:* 1  

- **Validate HMAC256**  
  - *Type:* If  
  - *Role:* Compares computed HMAC256 signature to the signature GitHub sent in the `x-hub-signature-256` header.  
  - *Configuration:*  
    - Condition: Checks if computed signature (`$json['signature-256']`) equals the signature extracted from header (`$json.headers['x-hub-signature-256'].split('=').pop()`).  
    - Case sensitive, strict string comparison.  
  - *Expressions:* Extracts signature from header by splitting on '=' and taking the last element.  
  - *Inputs:* Output from `Compute HMAC256`.  
  - *Outputs:*  
    - True branch: signature matches → send 200 OK response.  
    - False branch: signature mismatch → send 401 Unauthorized response.  
  - *Edge cases:*  
    - Missing or malformed `x-hub-signature-256` header causes failure.  
    - Expression errors if header is absent or not formatted as expected.  
  - *Version:* 2.2  

---

#### 1.3 Response and Further Processing

**Overview:**  
This block sends the appropriate HTTP response back to GitHub based on signature validation and either continues with further GitHub API processing or stops with an error.

**Nodes Involved:**  
- `Respond 200 OK`  
- `Get the profile of a repository`  
- `Respond 401 Unauthorized`  
- `Stop and Error`

**Node Details:**  
- **Respond 200 OK**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends HTTP 200 OK status without response body to acknowledge valid webhook.  
  - *Configuration:* Response code set to 200, no data in response body.  
  - *Inputs:* True branch from `Validate HMAC256`.  
  - *Outputs:* Passes control to `Get the profile of a repository` node.  
  - *Edge cases:* None specific, but failure to respond may cause GitHub retries.  
  - *Version:* 1.4  

- **Get the profile of a repository**  
  - *Type:* GitHub  
  - *Role:* Demonstrates next step after validation; retrieves repository profile using GitHub API.  
  - *Configuration:*  
    - Operation: getProfile on resource `repository`.  
    - Owner and repository parameters dynamically extracted from webhook payload URLs.  
    - Uses GitHub API credentials configured in n8n.  
  - *Expressions:* Owner and repository URLs extracted via expressions from webhook JSON body.  
  - *Inputs:* Output from `Respond 200 OK`.  
  - *Outputs:* Final node; no connected downstream nodes.  
  - *Edge cases:*  
    - Invalid or missing GitHub credentials cause authentication errors.  
    - Malformed repository URLs cause API errors.  
  - *Version:* 1.1  

- **Respond 401 Unauthorized**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends HTTP 401 Unauthorized status without response body to reject invalid webhook requests.  
  - *Configuration:* Response code set to 401, no data in response body.  
  - *Inputs:* False branch from `Validate HMAC256`.  
  - *Outputs:* Passes control to `Stop and Error`.  
  - *Edge cases:* None specific.  
  - *Version:* 1.4  

- **Stop and Error**  
  - *Type:* Stop and Error  
  - *Role:* Stops the workflow execution and generates an error with a descriptive message about the signature mismatch.  
  - *Configuration:* Custom error message explaining potential secret mismatch.  
  - *Inputs:* Output from `Respond 401 Unauthorized`.  
  - *Outputs:* None (workflow ends here).  
  - *Edge cases:* Workflow termination prevents further processing.  
  - *Version:* 1  

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                              | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                              |
|----------------------------|---------------------------|----------------------------------------------|-------------------------|------------------------------|---------------------------------------------------------------------------------------------------------|
| GitHub Webhook             | Webhook                   | Receive incoming GitHub webhook POST request | (External HTTP Request)  | Compute HMAC256              | See sticky note about validating GitHub webhook signatures using HMAC256                                |
| Compute HMAC256            | Crypto (HMAC SHA256)      | Compute HMAC256 hash of webhook body          | GitHub Webhook           | Validate HMAC256             | Explains HMAC256 computation and GitHub signature header validation details with useful links          |
| Validate HMAC256           | If                        | Compare computed signature with GitHub header | Compute HMAC256          | Respond 200 OK, Respond 401 Unauthorized |                                                                                                         |
| Respond 200 OK             | Respond to Webhook        | Return HTTP 200 OK for valid signature        | Validate HMAC256 (true)  | Get the profile of a repository | Explains acknowledging valid webhook with 200 OK                                                       |
| Get the profile of a repository | GitHub API             | Fetch repository profile after validation     | Respond 200 OK           | (none)                      | Suggests replacing this node with custom business logic after validation                                |
| Respond 401 Unauthorized   | Respond to Webhook        | Return HTTP 401 Unauthorized for invalid signature | Validate HMAC256 (false) | Stop and Error              | Explains sending 401 Unauthorized to GitHub on invalid signature                                       |
| Stop and Error             | Stop and Error            | Stop workflow and report signature mismatch   | Respond 401 Unauthorized | (none)                      | Optional node to stop workflow and raise error for unauthorized webhook calls                          |
| Sticky Note6               | Sticky Note               | Documentation on HMAC256 computation           | (none)                  | (none)                      | Detailed explanation and official GitHub docs link on webhook validation                               |
| Sticky Note                | Sticky Note               | Documentation on HTTP response codes           | (none)                  | (none)                      | Explains use of 200 OK and 401 Unauthorized response nodes                                            |
| Sticky Note1               | Sticky Note               | Documentation on continuing workflow post-validation | (none)                  | (none)                      | Suggests replacing placeholder with business logic after validation                                   |
| Sticky Note7               | Sticky Note               | Overall workflow explanation, use cases, warnings | (none)                  | (none)                      | Comprehensive overview, use cases, warnings, and customization tips with external contact links        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Name: `GitHub Webhook`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `github-test`  
   - Response Mode: `Response Node` (to defer response)  

2. **Create a Crypto Node for HMAC256 Computation**  
   - Name: `Compute HMAC256`  
   - Type: Crypto  
   - Type of Hash: SHA256  
   - Action: HMAC  
   - Value: Use expression `{{ JSON.stringify($json.body) }}` to stringify webhook payload  
   - Secret: Set to your GitHub webhook secret (replace `your_github_secret_here`)  
   - Output field name: `signature-256`  
   - Connect `GitHub Webhook` output to this node’s input  

3. **Create an If Node to Validate Signature**  
   - Name: `Validate HMAC256`  
   - Type: If  
   - Condition: String equals  
     - Left: `{{ $json['signature-256'] }}`  
     - Right: `{{ $json.headers['x-hub-signature-256'].split('=').pop() }}`  
   - Case Sensitive: true  
   - Connect `Compute HMAC256` output to this node’s input  

4. **Create a Respond to Webhook Node for 200 OK**  
   - Name: `Respond 200 OK`  
   - Type: Respond to Webhook  
   - Response Code: 200  
   - Respond With: No Data  
   - Connect `Validate HMAC256` “true” output to this node’s input  

5. **Create a Respond to Webhook Node for 401 Unauthorized**  
   - Name: `Respond 401 Unauthorized`  
   - Type: Respond to Webhook  
   - Response Code: 401  
   - Respond With: No Data  
   - Connect `Validate HMAC256` “false” output to this node’s input  

6. **Create a Stop and Error Node**  
   - Name: `Stop and Error`  
   - Type: Stop and Error  
   - Error Message:  
     `"HMAC256 signature doesn't match provided signature. Make sure that the GitHub webhook secret is identical to the secret stored in the 'Compute HMAC256' node."`  
   - Connect `Respond 401 Unauthorized` output to this node’s input  

7. **Create a GitHub Node for Further Processing**  
   - Name: `Get the profile of a repository`  
   - Type: GitHub  
   - Credentials: Select your configured GitHub account credentials  
   - Resource: Repository  
   - Operation: Get Profile  
   - Owner: Use expression `{{ $json.body.repository.owner.html_url }}` with mode set to URL  
   - Repository: Use expression `{{ $json.body.repository.html_url }}` with mode set to URL  
   - Connect `Respond 200 OK` output to this node’s input  

8. **Review and Update Secret**  
   - Replace the placeholder secret in `Compute HMAC256` with the exact secret configured in your GitHub webhook settings.  
   - Ensure the secret is kept confidential and not committed to source control.  

9. **Add Optional Sticky Notes for Documentation**  
   - Add notes explaining each block and node to provide inline documentation and usage instructions.  

10. **Activate Workflow and Test**  
   - Activate the workflow.  
   - Configure a GitHub webhook pointing to your n8n webhook URL at path `/github-test` with the same secret.  
   - Send test webhook events from GitHub and verify correct responses and processing.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow validates GitHub webhook authenticity using HMAC256 signature validation.                                           | Core workflow purpose                                                                                   |
| Read more about validating GitHub webhook deliveries: https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries | Official GitHub documentation on webhook signature validation                                         |
| n8n Crypto node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.crypto/#hmac-parameters     | Details on HMAC computation parameters in n8n                                                        |
| Warning: The webhook secret is stored in plaintext within this workflow. Avoid committing it to public repositories.         | Security best practice                                                                                   |
| For help, reach out on LinkedIn: https://www.linkedin.com/in/ytkaczyk/ or ask in the n8n community forum: https://community.n8n.io/ | Support channels                                                                                        |
| Replace the `Get the profile of a repository` node with your own business logic after validating webhook authenticity.       | Guidance on workflow customization                                                                      |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.

---