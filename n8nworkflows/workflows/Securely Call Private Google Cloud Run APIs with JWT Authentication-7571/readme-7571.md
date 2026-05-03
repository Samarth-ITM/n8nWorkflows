Securely Call Private Google Cloud Run APIs with JWT Authentication

https://n8nworkflows.xyz/workflows/securely-call-private-google-cloud-run-apis-with-jwt-authentication-7571


# Securely Call Private Google Cloud Run APIs with JWT Authentication

### 1. Workflow Overview

This n8n workflow demonstrates how to securely call private Google Cloud Run APIs using JWT (JSON Web Token) authentication. It is designed for scenarios where a Cloud Run service is configured to require authentication, and a service account with appropriate permissions is used to generate an ID token for authorization.

The workflow is logically divided into the following blocks:

- **1.1 Input Configuration:** Define and set essential authentication and service parameters such as service URL, client email, and token URI.
- **1.2 JWT Creation:** Generate a signed JWT assertion using the service account credentials.
- **1.3 Token Exchange:** Exchange the JWT for a Google OAuth2 Bearer token (ID token) by making an HTTP request to Google’s OAuth2 token endpoint.
- **1.4 Authenticated API Request:** Use the obtained ID token to make an authenticated HTTP request to the target Cloud Run service.
- **1.5 Manual Trigger:** Initiate the workflow manually for test or operational purposes.
- **1.6 Documentation and Guidance:** Sticky notes provide contextual instructions, setup requirements, credential hints, and useful references.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Configuration

- **Overview:**  
  This block sets up the essential parameters required for authentication and API access. It defines the Cloud Run service URL, the Google service account email, and the OAuth2 token URI.

- **Nodes Involved:**  
  - Edit Fields  
  - Sticky Note (Var Config)

- **Node Details:**

  - **Edit Fields**  
    - *Type & Role:* Set node; defines raw JSON input parameters.  
    - *Configuration:* Static JSON object with keys:  
      - `service_url`: Placeholder for the Cloud Run service URL (must be replaced by the user).  
      - `client_email`: Placeholder for the service account email.  
      - `token_uri`: Fixed URL `https://oauth2.googleapis.com/token` (Google OAuth2 token endpoint).  
    - *Expressions:* None beyond static JSON.  
    - *Connections:* Input from Manual Trigger → Output to JWT node.  
    - *Edge Cases:* User must replace placeholders; failure to do so results in failed authentication or HTTP calls.  
    - *Version:* 3.4  

  - **Sticky Note (Var Config)**  
    - *Purpose:* Explains variables `service_url` and `client_email` and their source (service account JSON key).  
    - *Position:* Adjacent to Edit Fields node for context.

---

#### 2.2 JWT Creation

- **Overview:**  
  This block creates a signed JWT token using the service account’s private key, including claims required for Google Cloud authentication.

- **Nodes Involved:**  
  - JWT  
  - Sticky Note (Credential Instructions)

- **Node Details:**

  - **JWT**  
    - *Type & Role:* JWT node; signs a JWT with given claims.  
    - *Configuration:*  
      - Uses JSON input from previous node (`Edit Fields`).  
      - Claims JSON includes:  
        - `iss` (issuer): service account email  
        - `sub` (subject): service account email  
        - `aud` (audience): token URI  
        - `iat` (issued at): current UNIX time  
        - `exp` (expiration): current UNIX time + 3600 seconds (1 hour)  
        - `target_audience`: Cloud Run service URL  
      - Credential: JWT Auth credential using the service account’s private key (PEM format) with RS256 algorithm.  
    - *Expressions:* Uses JavaScript expressions to generate timestamps and insert JSON fields dynamically.  
    - *Input:* From Edit Fields node.  
    - *Output:* JWT token string in JSON field `token`.  
    - *Edge Cases:*  
      - Invalid or expired private key causes signing failure.  
      - Incorrect claim values lead to token rejection by Google.  
      - Time synchronization issues may cause `iat` or `exp` problems.  
    - *Version:* 1  

  - **Sticky Note (Credential Instructions)**  
    - *Purpose:* Instructs how to create the JWT credential in n8n, emphasizing the private key format and algorithm.  
    - *Position:* Near JWT node.

---

#### 2.3 Token Exchange

- **Overview:**  
  This block exchanges the signed JWT for an OAuth2 Bearer token (ID token) by making an HTTP POST request to Google's token URI.

- **Nodes Involved:**  
  - Bearer Token Request  
  - Sticky Note (Bearer Credential Instructions)

- **Node Details:**

  - **Bearer Token Request**  
    - *Type & Role:* HTTP Request node; performs POST to OAuth2 token endpoint.  
    - *Configuration:*  
      - URL: Token URI from `Edit Fields`.  
      - Method: POST.  
      - Body: Form-urlencoded string containing:  
        `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion={{ $json.token }}`  
      - Content type: `application/x-www-form-urlencoded`.  
      - Sends body as a string (not JSON).  
    - *Expressions:* Uses JWT token from previous node’s output.  
    - *Output:* Contains `id_token` field (Google ID token).  
    - *Edge Cases:*  
      - Invalid or malformed JWT causes token exchange failure.  
      - Network or auth errors.  
      - Rate limiting or Google API downtime.  
    - *Version:* 4.2  

  - **Sticky Note (Bearer Credential Instructions)**  
    - *Purpose:* Explains usage of the returned `id_token` as a Bearer token in subsequent requests.  
    - *Additional Info:* Reminds to append any required path to the base service URL.

---

#### 2.4 Authenticated API Request

- **Overview:**  
  This block makes the authenticated call to the Google Cloud Run service using the obtained ID token as Bearer authorization.

- **Nodes Involved:**  
  - Cloud Run Request  
  - Sticky Note (Request Instructions)

- **Node Details:**

  - **Cloud Run Request**  
    - *Type & Role:* HTTP Request node; performs authenticated GET or POST to Cloud Run service.  
    - *Configuration:*  
      - URL: Taken dynamically from `Edit Fields` node’s `service_url`.  
      - Authentication: HTTP Bearer Auth, using credential configured with the `id_token`.  
    - *Input:* From Bearer Token Request node.  
    - *Output:* Response payload from Cloud Run service.  
    - *Edge Cases:*  
      - Invalid or expired ID token causes 401 Unauthorized errors.  
      - Incorrect service URL or network issues cause request failures.  
      - Service-specific errors (e.g., 403 Forbidden if permissions insufficient).  
    - *Version:* 4.2  

  - **Sticky Note (Request Instructions)**  
    - *Purpose:* Reinforces that the `id_token` is used as Bearer token and that the service URL may need a path appended.

---

#### 2.5 Manual Trigger

- **Overview:**  
  Allows the user to start the workflow manually for testing or execution.

- **Nodes Involved:**  
  - Execute (Manual Trigger)

- **Node Details:**

  - **Execute**  
    - *Type & Role:* Manual Trigger node; initiates the workflow run.  
    - *Configuration:* None.  
    - *Output:* Connects to Edit Fields node to start the process.  
    - *Edge Cases:* None significant.  
    - *Version:* 1  

---

#### 2.6 Documentation and Setup Guidance

- **Overview:**  
  Sticky notes throughout the workflow provide setup instructions, credential creation tips, and useful external resources for configuring Google Cloud Run and service accounts properly.

- **Nodes Involved:**  
  - Sticky Note1 (Workflow Purpose & Summary)  
  - Sticky Note4 (Google Cloud Run & Service Account Setup)  

- **Details:**

  - **Sticky Note1**:  
    - Explains the workflow’s purpose: creating an ID token to authorize calls to Cloud Run.  
    - Outlines the flow: setting variables → signing JWT → exchanging token → calling service.  
    - Mentions output: payload from Cloud Run.  

  - **Sticky Note4**:  
    - Lists required Google Cloud setup steps:  
      - Enable Cloud Run service with authentication required.  
      - Create a service account with Cloud Run Invoker role.  
    - Provides a link to a detailed Medium article for more in-depth guidance:  
      [Build a Secure Google Cloud Run API, Then Call It from n8n (Free Tier)](https://medium.com/@marcocodes/build-a-secure-google-cloud-run-api-then-call-it-from-n8n-88c03291a95f)

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                          | Input Node(s)        | Output Node(s)        | Sticky Note                                                                                                                     |
|---------------------|--------------------|----------------------------------------|----------------------|-----------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Execute             | Manual Trigger     | Manually starts workflow                |                      | Edit Fields           |                                                                                                                                |
| Edit Fields         | Set                | Defines authentication and service parameters | Execute              | JWT                   | #### Var Config - **service_url** — your Cloud Run service URL (base URL)  - **client_email** — from `.json key`              |
| JWT                 | JWT                 | Creates signed JWT with service account claims | Edit Fields           | Bearer Token Request   | ### Create a JWT Credential Use the **private_key** from your generated `.json key` file. - **Key Type:** PEM Key - **Private Key:** `private_key` (full block) - **Algorithm:** RS256 |
| Bearer Token Request | HTTP Request        | Exchanges JWT for an OAuth2 Bearer token (ID token) | JWT                   | Cloud Run Request      | #### Create a Bearer Credential Use the `{{$json.id_token}}` value. Append desired `/path` to the service_url.                 |
| Cloud Run Request    | HTTP Request        | Calls the Cloud Run service with Bearer token authorization | Bearer Token Request  |                       | #### Create a Bearer Credential Use the `{{$json.id_token}}` value. Append desired `/path` to the service_url.                 |
| Sticky Note1        | Sticky Note         | Workflow purpose and summary            |                      |                       | ## Google Service Auth Workflow (Minimal) Purpose and flow explanation                                                        |
| Sticky Note4        | Sticky Note         | Google Cloud Run & Service Account setup instructions |                      |                       | ## Required Setup — Google Cloud Run & Service Account See Medium article: https://medium.com/@marcocodes/build-a-secure-google-cloud-run-api-then-call-it-from-n8n-88c03291a95f |
| Sticky Note         | Sticky Note         | Variable configuration instructions     |                      |                       | #### Var Config - **service_url** — your Cloud Run service URL (base URL)  - **client_email** — from `.json key`              |
| Sticky Note2        | Sticky Note         | Instructions for creating JWT credential |                      |                       | ### Create a JWT Credential Use the **private_key** from your generated `.json key` file. - **Key Type:** PEM Key - **Private Key:** `private_key` (full block) - **Algorithm:** RS256 |
| Sticky Note3        | Sticky Note         | Instructions for creating Bearer credential |                      |                       | #### Create a Bearer Credential Use the `{{$json.id_token}}` value. Append desired `/path` to the service_url.                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Execute`  
   - Purpose: To manually start the workflow.

2. **Create Set Node for Input Parameters**  
   - Name: `Edit Fields`  
   - Connect from `Execute` node.  
   - Set mode: `Raw JSON`.  
   - JSON Content:  
     ```json
     {
       "service_url": "_____INSERT_____",
       "client_email": "_____INSERT_____",
       "token_uri": "https://oauth2.googleapis.com/token"
     }
     ```  
   - Replace placeholders with your Cloud Run service URL and service account email.

3. **Create JWT Node**  
   - Name: `JWT`  
   - Connect from `Edit Fields` node.  
   - Use JSON input.  
   - Claims JSON:  
     ```js
     {
       "iss": "{{ $json.client_email }}",
       "sub": "{{ $json.client_email }}",
       "aud": "{{ $json.token_uri }}",
       "iat": {{ Math.floor(Date.now() / 1000) }},
       "exp": {{ Math.floor(Date.now() / 1000) + 3600 }},
       "target_audience": "{{ $json.service_url }}"
     }
     ```  
   - Credentials: Create or select a JWT Auth credential configured with your Google service account private key:  
     - Key type: PEM Key  
     - Private key: Paste the full private key block from the `.json` key file (including BEGIN/END lines).  
     - Algorithm: RS256.

4. **Create HTTP Request Node for Bearer Token Exchange**  
   - Name: `Bearer Token Request`  
   - Connect from `JWT` node.  
   - Method: POST  
   - URL: Use expression to get `token_uri` from `Edit Fields` node:  
     ```js
     {{$node["Edit Fields"].json["token_uri"]}}
     ```  
   - Body Content Type: `application/x-www-form-urlencoded`  
   - Send Body as string: Enabled  
   - Body:  
     ```js
     grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion={{ $json.token }}
     ```  
   - No credentials needed here because token URI is public.

5. **Create HTTP Request Node for Cloud Run API Call**  
   - Name: `Cloud Run Request`  
   - Connect from `Bearer Token Request` node.  
   - Method: GET (or POST depending on your API)  
   - URL: Use expression to get `service_url` from `Edit Fields` node:  
     ```js
     {{$node["Edit Fields"].json["service_url"]}}
     ```  
   - Authentication: HTTP Bearer Auth  
   - Credentials: Create a Bearer Auth credential in n8n where you will input the `id_token` received from the previous node manually or dynamically.  
     *Note:* Since the `id_token` value is dynamic, you must configure the node to use an expression for the token, or alternatively, you can use the "Authentication: None" option and set the header manually:  
     - Headers: `Authorization: Bearer {{$json.id_token}}`  
     This can be set in the HTTP headers section if dynamic credential usage is limited.

6. **Create Sticky Notes for Documentation**  
   - Add sticky notes near relevant nodes explaining:  
     - How to set the variables (`service_url`, `client_email`).  
     - How to create the JWT credential with private key and RS256 algorithm.  
     - How to use the returned `id_token` as Bearer token in the Cloud Run request.  
     - Setup instructions for Google Cloud Run and service account with Cloud Run Invoker permission.  
     - Link to external article:  
       https://medium.com/@marcocodes/build-a-secure-google-cloud-run-api-then-call-it-from-n8n-88c03291a95f

7. **Save and Test the Workflow**  
   - Run the `Execute` manual trigger.  
   - Verify that the JWT is signed correctly.  
   - Confirm Bearer token is received.  
   - Confirm Cloud Run service responds with expected payload.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                 | Context or Link                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Google Cloud Run service must be configured to require authentication and the service account must have Cloud Run Invoker role.                                            | Google Cloud Console setup                                                                                                          |
| JWT Auth credential in n8n must use your Google service account private key in PEM format, with RS256 algorithm.                                                             | n8n JWT Auth credential configuration                                                                                               |
| To understand the full setup and usage, refer to the detailed Medium article by the author:                                                                                  | https://medium.com/@marcocodes/build-a-secure-google-cloud-run-api-then-call-it-from-n8n-88c03291a95f                                |
| When constructing the Cloud Run request, append any necessary path segments to the base `service_url` as required by your API endpoint.                                    | Workflow sticky notes                                                                                                                |
| Ensure system clock is synchronized to avoid JWT expiration or invalid issuance time errors.                                                                                 | Best practice for JWT authentication                                                                                                |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow and fully complies with content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.