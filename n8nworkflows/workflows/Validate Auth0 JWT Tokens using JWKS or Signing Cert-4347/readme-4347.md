Validate Auth0 JWT Tokens using JWKS or Signing Cert

https://n8nworkflows.xyz/workflows/validate-auth0-jwt-tokens-using-jwks-or-signing-cert-4347


# Validate Auth0 JWT Tokens using JWKS or Signing Cert

### 1. Workflow Overview

This workflow is designed to validate Auth0 JWT (JSON Web Tokens) tokens received via HTTP requests to n8n webhooks. It supports two distinct validation methods:  
- **1.1 Validation via JWKS URI:** Uses the JWKS (JSON Web Key Set) endpoint provided by Auth0 to verify tokens dynamically.  
- **1.2 Validation via Public Signing Certificate:** Uses a static signing certificate to verify tokens.  

The workflow acts as an API gateway that accepts requests bearing Auth0 JWT tokens in the Authorization header, validates the token using one of the two methods, and then responds with either a success or an unauthorized error response based on validation results.

Logical Blocks:  
- **1.1 Input Reception:** Two webhook nodes receive incoming requests on the same path but are used to demonstrate the two validation methods separately.  
- **1.2 JWKS Token Validation:** Code node using `jwks-rsa` and `jsonwebtoken` libraries to validate tokens against the JWKS URI.  
- **1.3 Public Certificate Token Validation:** Code node verifying tokens using a statically embedded public signing certificate.  
- **1.4 Response Handling:** Nodes that send back HTTP 200 OK or 401 Unauthorized responses based on token validity.  
- **1.5 Auxiliary Nodes:** NoOp nodes used to route the flow after validation, sticky notes for documentation and instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives HTTP requests on a specified webhook path. Supports two parallel webhook nodes for testing each validation method independently.

**Nodes Involved:**  
- `Webhook`  
- `Webhook1`

**Node Details:**  
- **Webhook / Webhook1**  
  - Type: Webhook  
  - Role: Entry point for incoming HTTP requests containing JWT tokens in the Authorization header.  
  - Configuration: Both configured with the same webhook path (`6b1e6a3d-9b6a-4b11-8d18-759b4073e651`).  
  - Inputs: External HTTP requests.  
  - Outputs: Forward requests to respective validation code nodes (`Webhook` → `Using Public Cert`; `Webhook1` → `Using JWK-RSA`).  
  - Edge Cases: Misconfiguration of webhook path or missing Authorization header in requests may cause failures downstream.

---

#### 1.2 JWKS Token Validation

**Overview:**  
Validates the JWT token by dynamically fetching the signing keys from the JWKS URI provided by Auth0. Uses the `jwks-rsa` Node.js library integrated inside a Code node.

**Nodes Involved:**  
- `Using JWK-RSA`  
- `Continue with Request`  
- `401 Unauthorized`  
- `200 OK`

**Node Details:**  
- **Using JWK-RSA**  
  - Type: Code (JavaScript)  
  - Role: Validates JWT token using `jsonwebtoken` and `jwks-rsa` libraries.  
  - Configuration:  
    - JWKS URI: `https://dev-abcdef.us.auth0.com/.well-known/jwks.json` (must be customized per Auth0 application).  
    - Expected Audience: `https://dev-abcdef.us.auth0.com/api/v2/`  
    - Issuer: `https://dev-abcdef.us.auth0.com/`  
    - Algorithm: RS256  
  - Key Expressions:  
    - Extracts token from `Authorization` header with `const [_, token] = $json.headers.Authorization.split(' ');`  
    - Uses asynchronous `verifyToken(token)` with `jwt.verify()` and dynamically obtains signing key via `getKey()`.  
  - Input: JSON object with HTTP headers.  
  - Output: On success, outputs original input extended with decoded JWT payload under `jwtPayload`.  
  - Error Handling: `onError` set to `continueErrorOutput` to catch invalid tokens and route to 401 response node.  
  - Edge Cases:  
    - Missing or malformed Authorization header results in errors.  
    - Network issues contacting JWKS URI could cause timeouts or errors.  
    - Invalid token or token with unrecognized kid triggers rejection.  
  - Requirements: Requires self-hosted n8n with `jwk-rsa` package installed and environment variable `NODE_FUNCTION_ALLOW_EXTERNAL=*` set.

- **Continue with Request**  
  - Type: NoOp  
  - Role: Passes validated requests forward to success response.  
  - Connections: Receives from success output of `Using JWK-RSA`, outputs to `200 OK`.  

- **401 Unauthorized**  
  - Type: RespondToWebhook  
  - Role: Returns HTTP 401 Unauthorized JSON response for invalid tokens detected by `Using JWK-RSA`.  
  - Response: `{ "error": 401, "message": "Unauthorized" }`

- **200 OK**  
  - Type: RespondToWebhook  
  - Role: Returns HTTP 200 OK JSON response indicating successful validation.  
  - Response: `{ "error": null }`

---

#### 1.3 Public Certificate Token Validation

**Overview:**  
Validates JWT tokens by verifying signature against a static Auth0 public signing certificate embedded in the node.

**Nodes Involved:**  
- `Using Public Cert`  
- `Continue with Request1`  
- `401 Unauthorized1`  
- `200 OK1`

**Node Details:**  
- **Using Public Cert**  
  - Type: Code (JavaScript)  
  - Role: Validates JWT token using `jsonwebtoken` library and static PEM-formatted public certificate.  
  - Configuration:  
    - Audience: `https://dev-abcdef.us.auth0.com/api/v2/`  
    - Issuer: `https://dev-abcdef.us.auth0.com/`  
    - Algorithm: RS256  
    - Public Certificate: Multi-line string placeholder `-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----` (must be replaced with actual certificate).  
  - Key Expressions:  
    - Extracts token from Authorization header similarly with `const [_, token] = $json.headers.Authorization.split(' ');`  
    - Attempts verification with `jwt.verify(token, cert, config)`.  
  - Error Handling: Throws an error if verification fails; error output configured to continue and route to 401 response.  
  - Edge Cases:  
    - Missing or malformed Authorization header.  
    - Certificate not matching token signature.  
    - Expired or otherwise invalid tokens.  
  - Requirements: Self-hosted n8n with environment variable `NODE_FUNCTION_ALLOW_EXTERNAL=*` set.

- **Continue with Request1**  
  - Type: NoOp  
  - Role: Passes validated requests forward to success response for cert-based validation branch.  
  - Connections: Success output of `Using Public Cert` to `200 OK1`.

- **401 Unauthorized1**  
  - Type: RespondToWebhook  
  - Role: Sends HTTP 401 Unauthorized JSON response for invalid tokens in cert validation branch.  

- **200 OK1**  
  - Type: RespondToWebhook  
  - Role: Sends HTTP 200 OK JSON response indicating successful validation in cert validation branch.

---

#### 1.4 Auxiliary and Documentation Nodes

**Nodes Involved:**  
- `Sticky Note`  
- `Sticky Note1`  
- `Sticky Note2`  
- `Sticky Note3`

**Node Details:**  
- Sticky notes provide detailed instructions on setup, usage, environment requirements, and important caveats including:  
  - How to use JWKS validation method (installing `jwk-rsa`, setting env var, configuring JWKS URI).  
  - How to use public cert validation (embedding cert, setting env var).  
  - Explanation of why Auth0 tokens require this approach due to RS256 and lack of private key access.  
  - Self-hosted n8n requirement due to need for external Node.js dependencies.  
  - Links to community support channels (Discord, Forum).  

---

### 3. Summary Table

| Node Name            | Node Type            | Functional Role                          | Input Node(s)      | Output Node(s)           | Sticky Note                                |
|----------------------|----------------------|----------------------------------------|--------------------|--------------------------|--------------------------------------------|
| Webhook              | Webhook              | Entry point for requests (Cert branch) | None               | Using Public Cert        |                                            |
| Webhook1             | Webhook              | Entry point for requests (JWKS branch) | None               | Using JWK-RSA            |                                            |
| Using JWK-RSA        | Code                 | Validate token using JWKS URI           | Webhook1           | Continue with Request, 401 Unauthorized | See Sticky Note (JWKS instructions)        |
| 401 Unauthorized     | RespondToWebhook     | Respond 401 for invalid JWKS tokens     | Using JWK-RSA      | None                     |                                            |
| Continue with Request | NoOp                  | Pass valid JWKS token requests          | Using JWK-RSA      | 200 OK                   |                                            |
| 200 OK               | RespondToWebhook     | Respond 200 for valid JWKS tokens       | Continue with Request | None                   |                                            |
| Using Public Cert     | Code                 | Validate token using static signing cert | Webhook           | Continue with Request1, 401 Unauthorized1 | See Sticky Note1 (Cert instructions)       |
| 401 Unauthorized1    | RespondToWebhook     | Respond 401 for invalid cert tokens     | Using Public Cert  | None                     |                                            |
| Continue with Request1 | NoOp                  | Pass valid cert token requests          | Using Public Cert  | 200 OK1                  |                                            |
| 200 OK1              | RespondToWebhook     | Respond 200 for valid cert tokens       | Continue with Request1 | None                   |                                            |
| Sticky Note          | Sticky Note          | JWKS validation instructions            | None               | None                     | JWKS usage details and setup instructions  |
| Sticky Note1         | Sticky Note          | Public cert validation instructions     | None               | None                     | Cert usage details and setup instructions  |
| Sticky Note2         | Sticky Note          | General overview and usage instructions | None               | None                     | Full workflow explanation and links        |
| Sticky Note3         | Sticky Note          | Self-hosted n8n environment requirement | None               | None                     | Notes on dependency installation and n8n edition |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node `Webhook`**  
   - Type: Webhook  
   - Path: `6b1e6a3d-9b6a-4b11-8d18-759b4073e651`  
   - Purpose: Entry for Public Cert validation branch  
   - Leave options default.

2. **Create Webhook Node `Webhook1`**  
   - Type: Webhook  
   - Path: same as above (`6b1e6a3d-9b6a-4b11-8d18-759b4073e651`)  
   - Purpose: Entry for JWKS validation branch.

3. **Create Code Node `Using JWK-RSA`**  
   - Type: Code (JavaScript)  
   - Execute for each item: Yes (mode: runOnceForEachItem)  
   - Paste the following logic (adapted):  
     - Import `jsonwebtoken` and `jwks-rsa` libraries.  
     - Configure `jwksClient` with your Auth0 JWKS URI (replace placeholder).  
     - Define `config` with your audience, issuer, and algorithms.  
     - Extract Bearer token from `Authorization` header.  
     - Use asynchronous `jwt.verify` with dynamic key fetching from JWKS.  
     - On success, return original input extended with decoded token payload.  
   - Set `onError` to `continueErrorOutput`.  
   - Connect input from `Webhook1`.

4. **Create RespondToWebhook Node `401 Unauthorized`**  
   - HTTP Response Code: 401  
   - Response Type: JSON  
   - Response Body: `{"error": 401, "message": "Unauthorized"}`  
   - Connect error output of `Using JWK-RSA` here.

5. **Create NoOp Node `Continue with Request`**  
   - No parameters needed.  
   - Connect success output of `Using JWK-RSA` here.

6. **Create RespondToWebhook Node `200 OK`**  
   - HTTP Response Code: 200  
   - Response Type: JSON  
   - Response Body: `{"error": null}`  
   - Connect output of `Continue with Request` here.

7. **Create Code Node `Using Public Cert`**  
   - Type: Code (JavaScript)  
   - Execute for each item: Yes  
   - Paste logic:  
     - Import `jsonwebtoken`.  
     - Configure verification options (audience, issuer, algorithms).  
     - Embed your Auth0 public signing certificate as PEM string (replace placeholder).  
     - Extract Bearer token from `Authorization` header.  
     - Call `jwt.verify(token, cert, config)` inside try/catch block.  
     - Return original input with decoded payload on success.  
     - Throw error on failure.  
   - Set `onError` to `continueErrorOutput`.  
   - Connect input from `Webhook`.

8. **Create RespondToWebhook Node `401 Unauthorized1`**  
   - Same settings as `401 Unauthorized`.  
   - Connect error output of `Using Public Cert` here.

9. **Create NoOp Node `Continue with Request1`**  
   - Connect success output of `Using Public Cert` here.

10. **Create RespondToWebhook Node `200 OK1`**  
    - Same settings as `200 OK`.  
    - Connect output of `Continue with Request1` here.

11. **Add Sticky Note Nodes** with content describing:  
    - JWKS validation instructions and requirements.  
    - Public Cert validation instructions and requirements.  
    - Full overview of the workflow.  
    - Self-hosted n8n and dependency installation notes.

12. **Environment Setup:**  
    - Install `jwk-rsa` globally in your self-hosted n8n environment: `npm i -g jwk-rsa`  
    - Set environment variable `NODE_FUNCTION_ALLOW_EXTERNAL=*` to allow external library usage in Code nodes.  
    - Replace placeholders in JWKS URI, audience, issuer, and public certificate with your Auth0 tenant values.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow requires self-hosted n8n due to the need for third-party Node.js libraries (`jsonwebtoken`, `jwks-rsa`) not available on n8n cloud.                                                                                                                                                                                                                                   | Sticky Note3                                                                                             |
| Install third-party dependency for JWKS validation: `npm i -g jwk-rsa`                                                                                                                                                                                                                                                                                                               | Sticky Note                                                                                              |
| Set environment variable `NODE_FUNCTION_ALLOW_EXTERNAL=*` to allow Code nodes to use external libraries.                                                                                                                                                                                                                                                                             | Sticky Note, Sticky Note1                                                                                 |
| Auth0's JWKS URI can be found in the Auth0 dashboard under Applications > Settings > Advanced Settings > Endpoints > JSON Web Key Set URL. The signing certificate can be found under Certificates in the same section.                                                                                                                                                              | Sticky Note, Sticky Note1                                                                                 |
| Auth0 tokens use RS256 algorithm, which requires validation via public keys rather than shared secrets. This workflow addresses this limitation by validating tokens after webhook reception.                                                                                                                                                                                        | Sticky Note2                                                                                              |
| For community help and troubleshooting, join the n8n Discord: https://discord.com/invite/XPKeKXeB7d or post on the n8n Forum: https://community.n8n.io/                                                                                                                                                                                                                            | Sticky Note2                                                                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, strictly respecting current content policies and containing no illegal or protected material. All processed data is legal and public.