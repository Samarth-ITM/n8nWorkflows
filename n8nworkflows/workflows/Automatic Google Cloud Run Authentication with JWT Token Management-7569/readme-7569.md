Automatic Google Cloud Run Authentication with JWT Token Management

https://n8nworkflows.xyz/workflows/automatic-google-cloud-run-authentication-with-jwt-token-management-7569


# Automatic Google Cloud Run Authentication with JWT Token Management

### 1. Workflow Overview

This workflow automates authentication to Google Cloud Run services by managing JWT (JSON Web Token) tokens. It is designed to efficiently handle token reuse and refresh, ensuring secure and seamless API calls with minimal overhead.

**Target Use Cases:**

- Automating Google Cloud Run service authentication in integrations or API calls.
- Managing JWT tokens with automatic refresh before expiration.
- Minimizing quota usage by validating tokens locally before requesting new ones.
- Serving as a reusable sub-workflow in larger automation setups that require authenticated Google Cloud Run service access.

**Logical Blocks:**

- **1.1 Input Reception & Context Setup:** Receives inputs (`id_token`, `service_url`, `service_path`), sets variables, and combines context data.
- **1.2 Token Existence Check:** Determines if an existing JWT token (`id_token`) is provided.
- **1.3 JWT Decoding & Expiration Validation:** Decodes the token and checks if it is still valid considering a 5-minute buffer.
- **1.4 JWT Signing & Token Request:** Generates and signs a new JWT token if needed, then exchanges it for an access token via Google OAuth2.
- **1.5 Output Preparation:** Returns the valid or newly generated token along with service URL and path for downstream use.
- **1.6 Documentation & Notes:** Sticky notes providing explanations, instructions, and setup references.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Context Setup

**Overview:**  
This block receives input parameters and prepares the context for the token management logic by setting necessary variables and merging them.

**Nodes Involved:**  
- Start  
- Vars  
- Combine Context  

**Node Details:**

- **Start**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point, receiving optional inputs: `id_token`, `service_url`, `service_path`.  
  - *Configuration:* Defined input fields for these three parameters.  
  - *Connections:* Outputs to `Combine Context` and `Vars`.  
  - *Edge Cases:* Missing inputs may affect downstream logic; `service_url` is required for token audience.  

- **Vars**  
  - *Type:* Set  
  - *Role:* Sets static variables like `client_email` (must be replaced by user), and `token_uri` for OAuth2 token endpoint. Also passes through `service_url` from input.  
  - *Configuration:* Raw JSON with placeholders; client_email must be updated with Google Service Account email.  
  - *Connections:* Outputs to `Combine Context`.  
  - *Edge Cases:* Incorrect `client_email` or missing `token_uri` URL will cause failures during JWT signing or token requests.  

- **Combine Context**  
  - *Type:* Merge (combine mode)  
  - *Role:* Combines input parameters and variables into a single context object for downstream use.  
  - *Connections:* Inputs from `Start` and `Vars`; outputs to `If Token`.  
  - *Edge Cases:* Merge failure if inputs are malformed or missing.

---

#### 1.2 Token Existence Check

**Overview:**  
Checks if an existing JWT token (`id_token`) is provided to decide whether to reuse or generate a new one.

**Nodes Involved:**  
- If Token  

**Node Details:**

- **If Token**  
  - *Type:* If  
  - *Role:* Checks if `id_token` exists in the combined context JSON.  
  - *Configuration:* Condition tests existence of `id_token` string property.  
  - *Connections:* True branch to `Decode JWT` (validate token), False branch to `Sign New JWT` (generate new).  
  - *Edge Cases:* If `id_token` is malformed or empty string, false path triggers new token generation.

---

#### 1.3 JWT Decoding & Expiration Validation

**Overview:**  
Decodes the JWT token to extract claims and checks if the token is still valid with a 5-minute expiration buffer.

**Nodes Involved:**  
- Decode JWT  
- If  

**Node Details:**

- **Decode JWT**  
  - *Type:* JWT  
  - *Role:* Decodes JWT token payload without verifying signature to read claims such as expiration.  
  - *Configuration:* Takes `id_token` from JSON, using JWT credentials with RS256 private key.  
  - *Connections:* Outputs to `If` node which checks expiration.  
  - *Edge Cases:* Malformed tokens or invalid JWT structure cause decode failure. Credential misconfiguration can cause errors.  

- **If**  
  - *Type:* If  
  - *Role:* Checks if token expiration time (`exp`) minus 5 minutes (300,000 ms) is greater than current time (Date.now()).  
  - *Configuration:* Condition uses expression: `($json.payload.exp * 1000) - 300000 >= Date.now()`  
  - *Connections:* True branch outputs to `Raise Id` (reuse token), False branch outputs to `Sign New JWT` (refresh token).  
  - *Edge Cases:* Incorrect `exp` claim or time skew may cause premature refresh or reuse of expired tokens.

---

#### 1.4 JWT Signing & Token Request

**Overview:**  
Signs a new JWT token with claims needed for Google OAuth2, then requests a bearer token from Google’s token endpoint.

**Nodes Involved:**  
- Sign New JWT  
- Bearer Token Request  

**Node Details:**

- **Sign New JWT**  
  - *Type:* JWT  
  - *Role:* Creates a new JWT with claims including issuer, subject, audience, issued-at, expiration (1 hour), and target audience (service URL).  
  - *Configuration:* Uses JSON claims with dynamic expressions referencing variables like `client_email`, `token_uri`, `service_url`.  
  - *Connections:* Outputs signed JWT to `Bearer Token Request`.  
  - *Credential:* JWT Auth with private_key from Google Service Account JSON; algorithm RS256.  
  - *Edge Cases:* Missing or incorrect private key causes signing failure; incorrect claims cause token rejection later.

- **Bearer Token Request**  
  - *Type:* HTTP Request  
  - *Role:* Exchanges signed JWT for OAuth2 bearer token by POSTing to Google's token URI.  
  - *Configuration:* POST method with `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer` and `assertion` set to signed JWT token. Content type is `application/x-www-form-urlencoded`.  
  - *Connections:* Outputs to `Return Values` node.  
  - *Edge Cases:* Network errors, invalid assertion, or quota limits cause failure; response parsing errors possible.

---

#### 1.5 Output Preparation

**Overview:**  
Prepares the final output containing the valid JWT token and service endpoint details for use by other workflows or API calls.

**Nodes Involved:**  
- Raise Id  
- Return Values  

**Node Details:**

- **Raise Id**  
  - *Type:* Set  
  - *Role:* Passes along existing valid `id_token` from context for reuse.  
  - *Connections:* Outputs to `Return Values`.  
  - *Edge Cases:* Missing token causes empty output.

- **Return Values**  
  - *Type:* Set  
  - *Role:* Outputs final JSON with `id_token`, `service_url`, and `service_path`.  
  - *Configuration:* Pulls `id_token` from either newly generated or reused token, and service info from combined context.  
  - *Connections:* Terminal node.  
  - *Edge Cases:* Missing inputs cause incomplete output.

---

#### 1.6 Documentation & Notes

**Overview:**  
Multiple sticky notes provide detailed documentation, setup instructions, and feature explanations.

**Nodes Involved:**  
- Sticky Note1 to Sticky Note5  

**Node Details:**

- **Sticky Note1**  
  - Describes workflow purpose, input/output, and high-level flow.

- **Sticky Note2**  
  - Usage instructions as sub-workflow, token refresh logic, and testing notes.

- **Sticky Note3**  
  - Key features like automatic validation, error prevention, and efficient reuse.

- **Sticky Note4**  
  - Notes on variable configuration, specifically `client_email`.

- **Sticky Note5**  
  - Instructions for creating JWT credentials with private key setup.

- **Sticky Note (at top left)**  
  - Required Google Cloud Run and Service Account setup steps.  
  - Link to detailed Medium article:  
    https://medium.com/@marcocodes/build-a-secure-google-cloud-run-api-then-call-it-from-n8n-88c03291a95f

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                                  | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                          |
|---------------------|----------------------------|-------------------------------------------------|---------------------|----------------------|----------------------------------------------------------------------------------------------------|
| Start               | Execute Workflow Trigger   | Entry point receiving inputs                     |                     | Combine Context, Vars |                                                                                                    |
| Vars                | Set                        | Sets variables: client_email, token_uri, service_url | Start               | Combine Context       | #### Var Config - **client_email** - from `.json key`                                              |
| Combine Context     | Merge                      | Combines inputs and variables into context       | Start, Vars          | If Token              |                                                                                                    |
| If Token            | If                         | Checks existence of `id_token`                    | Combine Context      | Decode JWT (true), Sign New JWT (false) |                                                                                                    |
| Decode JWT          | JWT                        | Decodes token payload for claims                  | If Token             | If                    |                                                                                                    |
| If                  | If                         | Checks token expiration with 5-min buffer         | Decode JWT           | Raise Id (true), Sign New JWT (false) |                                                                                                    |
| Raise Id            | Set                        | Passes existing valid token                        | If                   | Return Values          |                                                                                                    |
| Sign New JWT        | JWT                        | Creates new JWT token with Google claims          | If Token, If (false) | Bearer Token Request   | ### Create a JWT Credential Use the **private_key** from your generated `.json key` file.          |
| Bearer Token Request | HTTP Request               | Exchanges JWT for OAuth2 bearer token              | Sign New JWT         | Return Values          |                                                                                                    |
| Return Values       | Set                        | Outputs token and service info                      | Raise Id, Bearer Token Request |                      |                                                                                                    |
| Sticky Note1        | Sticky Note                | Workflow description and purpose                    |                     |                      | ## Google Service Auth Workflow Purpose, How It Works, Output                                   |
| Sticky Note2        | Sticky Note                | Usage instructions and token logic                  |                     |                      | ## Using This Workflow Sub-workflow usage and token refresh logic                               |
| Sticky Note3        | Sticky Note                | Key features and error prevention                    |                     |                      | ## Key Features Automatic validation, Error prevention, Efficient reuse                         |
| Sticky Note4        | Sticky Note                | Variable configuration notes                          |                     |                      | #### Var Config - **client_email** - from `.json key`                                           |
| Sticky Note5        | Sticky Note                | JWT credential creation instructions                 |                     |                      | ### Create a JWT Credential Use the **private_key** from your generated `.json key` file.       |
| Sticky Note         | Sticky Note                | Google Cloud Run & Service Account setup instructions |                     |                      | ## Required Setup — Google Cloud Run & Service Account [Medium Article](https://medium.com/@marcocodes/build-a-secure-google-cloud-run-api-then-call-it-from-n8n-88c03291a95f) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Start node:**  
   - Type: Execute Workflow Trigger  
   - Configure inputs: `id_token`, `service_url`, `service_path` (all optional except `service_url` for testing).  

2. **Add a Set node named `Vars`:**  
   - Set mode: Raw JSON  
   - JSON:  
     ```json
     {
       "service_url": "{{ $json.service_url }}",
       "client_email": "_____INSERT_____",
       "token_uri": "https://oauth2.googleapis.com/token"
     }
     ```  
   - Replace `client_email` with your Google Service Account email.  

3. **Add a Merge node named `Combine Context`:**  
   - Mode: Combine  
   - Combine By: Combine All Inputs  
   - Connect inputs from **Start** and **Vars**.  

4. **Add an If node named `If Token`:**  
   - Condition: Check if `id_token` exists (string exists on `{{$json.id_token}}`).  
   - Connect input from **Combine Context**.  

5. **Add a JWT node named `Decode JWT`:**  
   - Operation: Decode  
   - Token: `{{$json.id_token}}`  
   - Credentials: JWT Auth with RS256 private key from your Google Service Account JSON.  
   - Connect **true** output of **If Token** to this node.  

6. **Add an If node named `If`:**  
   - Condition:  
     ```  
     (($json.payload.exp * 1000) - 300000) >= Date.now()  
     ```  
   - Connect input from **Decode JWT**.  

7. **Add a Set node named `Raise Id`:**  
   - Set `id_token` to `{{$json.id_token}}`.  
   - Connect **true** output of **If** to this node.  

8. **Add a JWT node named `Sign New JWT`:**  
   - Operation: Sign  
   - Use JSON claims:  
     ```json
     {
       "iss": "{{ $json.client_email }}",
       "sub": "{{ $json.client_email }}",
       "aud": "{{ $json.token_uri }}",
       "iat": {{ Math.floor(Date.now() / 1000) }},
       "exp": {{ Math.floor(Date.now() / 1000) + 3600 }},
       "target_audience": "{{ $json.service_url }}"
     }
     ```  
   - Credentials: JWT Auth with RS256 private key.  
   - Connect **false** output of **If Token** and **false** output of **If** to this node (merge these two outputs).  

9. **Add an HTTP Request node named `Bearer Token Request`:**  
   - Method: POST  
   - URL: `{{$('Vars').item.json.token_uri}}`  
   - Body (raw string):  
     ```
     grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion={{ $json.token }}
     ```  
   - Content-Type: `application/x-www-form-urlencoded`  
   - Connect input from **Sign New JWT**.  

10. **Add a Set node named `Return Values`:**  
    - Set fields:  
      - `id_token` = `{{$json.id_token}}` (from either `Raise Id` or `Bearer Token Request`)  
      - `service_url` = `{{$('Combine Context').item.json.service_url}}`  
      - `service_path` = `{{$('Combine Context').item.json.service_path}}`  
    - Connect inputs from **Raise Id** and **Bearer Token Request** outputs (merge).  

11. **Add Sticky Notes for documentation:**  
    - Include workflow purpose, usage instructions, key features, variable configuration, JWT credential setup, and required Google Cloud Run & Service Account steps with links as per original notes.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow handles Google Cloud Run authentication with automatic token refresh and reuse, preventing mid-execution token expiration.                                                   | Main workflow purpose                                                                                                   |
| Use 5-minute expiration buffer to avoid token expiry during workflow execution.                                                                                                          | Token refresh strategy                                                                                                  |
| JWT credentials require RS256 algorithm and private_key from Google Service Account JSON, including full PEM block.                                                                     | Credential setup                                                                                                        |
| Google Cloud Run service must be configured to require authentication; Service Account must have Cloud Run Invoker permission.                                                          | Setup prerequisites                                                                                                    |
| Detailed explanation and step-by-step guide available at: [Build a Secure Google Cloud Run API, Then Call It from n8n (Free Tier)](https://medium.com/@marcocodes/build-a-secure-google-cloud-run-api-then-call-it-from-n8n-88c03291a95f) | External blog article link                                                                                              |
| Workflow can be used as a sub-workflow; pass optional `id_token` to reuse tokens, or let it generate a fresh token if none provided.                                                   | Usage note                                                                                                             |
| Tokens are valid for 60 minutes; workflow refreshes tokens only when expired or near expiry to reduce API calls and quota usage.                                                       | Efficiency and quota optimization                                                                                      |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.