Automated Zalo OA Token Management with OAuth and Webhook Integration

https://n8nworkflows.xyz/workflows/automated-zalo-oa-token-management-with-oauth-and-webhook-integration-8675


# Automated Zalo OA Token Management with OAuth and Webhook Integration

### 1. Workflow Overview

This workflow automates the management of Zalo Official Account (OA) OAuth tokens by maintaining a fresh access token through scheduled refreshes and manual reset capabilities. It uses n8n’s global Workflow Static Data to cache tokens, ensuring availability for downstream integrations without repeated authentication calls. The workflow also exposes a webhook endpoint to provide the current cached token on-demand.

The workflow contains three main logical blocks:

- **1.1 Scheduled Token Refresh:** Automatically refreshes the access token every 12 hours.
- **1.2 Manual Reset and Re-seed:** Allows manual clearing and reseeding of the token cache via a secured webhook.
- **1.3 Token Retrieval Webhook:** Provides a simple webhook to fetch the current cached access token.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Token Refresh

**Overview:**  
This block periodically refreshes the Zalo OA access token using stored credentials and updates the cached token data with new tokens and expiry times.

**Nodes Involved:**  
- Schedule Trigger  
- Set Refresh Token and App ID  
- Load to Static Data  
- Refresh Token (Zalo v4)  
- Store to SD & Pass token

**Node Details:**

- **Schedule Trigger**  
  - *Type & Role:* Schedule Trigger; initiates the flow every 12 hours.  
  - *Configuration:* Runs at fixed 12-hour intervals.  
  - *Input/Output:* No input; outputs trigger signal to next node.  
  - *Failures:* None typical; consider workflow paused or scheduling errors.

- **Set Refresh Token and App ID**  
  - *Type & Role:* Set node; assigns static credentials (refresh token, app ID, secret key) used for token refresh requests.  
  - *Configuration:* Hardcoded placeholder values for `refresh_token`, `app_id`, and `secret_key`.  
  - *Expressions:* None; values are static strings.  
  - *Input:* Trigger from Schedule Trigger or manual reset path.  
  - *Output:* JSON with credentials for following nodes.  
  - *Failures:* Credentials missing or incorrect cause API request failure.

- **Load to Static Data**  
  - *Type & Role:* Code node; loads or initializes cached tokens in Workflow Static Data and determines if refresh is needed.  
  - *Configuration:*  
    - Reads global static data under `zalo` key.  
    - Applies a 90-second early refresh buffer to avoid token expiry during use.  
    - If no refresh token cached, seeds it from input JSON (initial run).  
  - *Key Logic:* Flags `needs_refresh` boolean for downstream conditional use.  
  - *Input:* Credentials JSON.  
  - *Output:* JSON with token status and expiry info.  
  - *Failures:* Expression errors if input malformed; static data access issues are rare.

- **Refresh Token (Zalo v4)**  
  - *Type & Role:* HTTP Request; calls Zalo OAuth v4 endpoint to exchange refresh token for new access token.  
  - *Configuration:*  
    - POST to `https://oauth.zaloapp.com/v4/oa/access_token`  
    - Content-Type: `application/x-www-form-urlencoded`  
    - Body parameters: `refresh_token`, `app_id`, `grant_type=refresh_token`.  
    - Header: `secret_key` from Set Refresh Token and App ID node.  
  - *Input:* Tokens and credentials from prior node.  
  - *Output:* JSON response with new access token, refresh token, and expiry info.  
  - *Failures:*  
    - HTTP errors (4xx/5xx), expired or invalid refresh token, network timeouts, or incorrect secret key.  
    - Must handle response parsing safely.

- **Store to SD & Pass token**  
  - *Type & Role:* Code node; persists new tokens and expiry timestamps into Workflow Static Data and outputs refreshed token.  
  - *Configuration:*  
    - Updates global static data keys: `access_token`, `refresh_token`, `access_expires_at`, `refresh_expires_at`.  
    - Returns current refreshed token info for downstream use.  
  - *Input:* HTTP response from Refresh Token node.  
  - *Output:* JSON with refreshed token and expiry timestamp.  
  - *Failures:* Static data write permission issues (rare), invalid data structure.

---

#### 1.2 Manual Reset and Re-seed

**Overview:**  
Allows manual clearing of cached tokens and reseeding of credentials via a secured webhook, useful for testing or credential rotation.

**Nodes Involved:**  
- Execute_Node (Webhook)  
- Clean Zalo Static Data  
- Set Refresh Token and App ID (shared with scheduled flow)  
- Load to Static Data (shared)  
- Refresh Token (Zalo v4) (shared)  
- Store to SD & Pass token (shared)

**Node Details:**

- **Execute_Node (Webhook)**  
  - *Type & Role:* Webhook; manual POST endpoint to trigger token cache reset and reseed.  
  - *Configuration:*  
    - Path: auto-generated unique ID string.  
    - HTTP method: POST.  
    - No explicit authentication configured (recommend securing in production).  
  - *Input:* External POST request triggers flow.  
  - *Output:* Triggers Clear Static Data node.  
  - *Failures:* Unauthorized access if unsecured; malformed requests.

- **Clean Zalo Static Data**  
  - *Type & Role:* Code node; clears cached Zalo tokens from Workflow Static Data.  
  - *Configuration:* Deletes `zalo` key from global static data.  
  - *Input:* Triggered by Execute_Node.  
  - *Output:* JSON confirming clearance.  
  - *Failures:* Static data access errors (rare).

- **Set Refresh Token and App ID**  
  - *Same as in scheduled flow.* Used here to provide credentials after clearing cache.

- **Load to Static Data, Refresh Token (Zalo v4), Store to SD & Pass token**  
  - *Same nodes as scheduled flow.*  
  - Re-initialize and refresh tokens based on fresh credentials.

---

#### 1.3 Token Retrieval Webhook

**Overview:**  
Provides an HTTP POST webhook endpoint to retrieve the currently cached Zalo OA access token and expiry without triggering a refresh.

**Nodes Involved:**  
- Webhook (zalo-intergration-v1)  
- Load Access Token

**Node Details:**

- **Webhook (zalo-intergration-v1)**  
  - *Type & Role:* Webhook; exposes a lightweight POST endpoint for token retrieval.  
  - *Configuration:*  
    - Path: `zalo-intergration-v1`  
    - HTTP method: POST.  
    - No authentication configured (recommend securing in production).  
  - *Input:* External POST request.  
  - *Output:* Triggers Load Access Token node.  
  - *Failures:* Unauthorized access if unsecured.

- **Load Access Token**  
  - *Type & Role:* Code node; reads cached access token, refresh token, and expiry from Workflow Static Data and returns them.  
  - *Configuration:* Reads from global `zalo` static data key.  
  - *Input:* Triggered by webhook.  
  - *Output:* JSON with current tokens and expiry.  
  - *Failures:* Returns nulls if no token cached.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                                               | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                         |
|---------------------------|-------------------------|---------------------------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger         | Triggers scheduled token refresh every 12 hours               |                             | Set Refresh Token and App ID | # Zalo OA Token Auto-Refresh (n8n) - Runs the auto-refresh flow on a fixed interval (every 12h)    |
| Set Refresh Token and App ID | Set                    | Provides refresh token, app ID, secret key                     | Schedule Trigger, Clean Zalo Static Data | Load to Static Data          | Tip: In production, reference environment variables instead of hardcoding.                        |
| Load to Static Data       | Code                    | Loads / seeds cached tokens, checks if refresh needed          | Set Refresh Token and App ID | Refresh Token (Zalo v4)      | Applies 90s early refresh buffer; flags if a refresh is needed.                                  |
| Refresh Token (Zalo v4)   | HTTP Request            | Exchanges refresh token for new access token                   | Load to Static Data          | Store to SD & Pass token     | Calls Zalo OAuth v4 token endpoint; requires valid credentials.                                  |
| Store to SD & Pass token  | Code                    | Stores refreshed tokens and expiry timestamps in static data  | Refresh Token (Zalo v4)      |                             | Persists tokens globally for workflow reuse.                                                     |
| Execute_Node              | Webhook                 | Manual trigger webhook to reset token cache                    |                             | Clean Zalo Static Data       | Securing this webhook is recommended for production.                                            |
| Clean Zalo Static Data    | Code                    | Clears cached tokens from workflow static data                 | Execute_Node                | Set Refresh Token and App ID | Prepares for token reseeding on manual reset.                                                   |
| Webhook (zalo-intergration-v1) | Webhook             | Endpoint to fetch cached token without refreshing              |                             | Load Access Token            | Useful for other workflows to retrieve current token.                                           |
| Load Access Token         | Code                    | Reads cached token and expiry, returns to webhook caller       | Webhook (zalo-intergration-v1) |                             | Returns nulls if no token cached yet.                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to every 12 hours.

2. **Create a Set node named `Set Refresh Token and App ID`**  
   - Add fields:  
     - `refresh_token`: string, initial refresh token value (replace `"refresh_token_initial_refesh_token"`).  
     - `app_id`: string, your Zalo Official Account App ID.  
     - `secret_key`: string, your Zalo app secret key.  
   - Connect Schedule Trigger → Set Refresh Token and App ID.

3. **Create a Code node named `Load to Static Data`**  
   - Paste JavaScript code that:  
     - Reads global static data at `zalo`.  
     - If no refresh token cached, seeds from input JSON.  
     - Checks if access token is present and not expired (with 90s buffer).  
     - Outputs `needs_refresh` flag and current tokens.  
   - Connect Set Refresh Token and App ID → Load to Static Data.

4. **Create an HTTP Request node named `Refresh Token (Zalo v4)`**  
   - Method: POST  
   - URL: `https://oauth.zaloapp.com/v4/oa/access_token`  
   - Content-Type: `application/x-www-form-urlencoded`  
   - Body parameters:  
     - `refresh_token` = `{{$json["refresh_token"]}}`  
     - `app_id` = `{{$json["app_id"]}}`  
     - `grant_type` = `refresh_token`  
   - Header parameter:  
     - `secret_key` = expression referencing `Set Refresh Token and App ID` node output  
   - Connect Load to Static Data → Refresh Token (Zalo v4).

5. **Create a Code node named `Store to SD & Pass token`**  
   - Paste JavaScript code that:  
     - Updates global static data `zalo` with new `access_token`, `refresh_token`, and expiry timestamps using response data.  
     - Returns current refreshed token info.  
   - Connect Refresh Token (Zalo v4) → Store to SD & Pass token.

6. **Create a Webhook node named `Execute_Node` for manual reset**  
   - HTTP method: POST  
   - Path: generate a unique string (e.g., UUID) for security.  
   - This node acts as a manual trigger.  

7. **Create a Code node named `Clean Zalo Static Data`**  
   - Paste JavaScript code to delete the `zalo` key from global static data.  
   - Connect Execute_Node → Clean Zalo Static Data.

8. **Connect Clean Zalo Static Data → Set Refresh Token and App ID**  
   - This resumes the token re-seeding and refresh flow.

9. **Create a Webhook node named `Webhook (zalo-intergration-v1)` for token retrieval**  
   - HTTP method: POST  
   - Path: `zalo-intergration-v1`.  
   - This endpoint returns cached token info on demand.

10. **Create a Code node named `Load Access Token`**  
    - Paste JavaScript code that reads the global static data `zalo` object and returns `access_token`, `access_expires_at`, and `refresh_token` (or null if not set).  
    - Connect Webhook (zalo-intergration-v1) → Load Access Token.

11. **Secure both webhook nodes (`Execute_Node` and `Webhook (zalo-intergration-v1)`)**  
    - Add IP allowlisting, secret tokens, or OAuth2 authentication as needed to prevent unauthorized access.

12. **Environment Variables (Optional but Recommended)**  
    - Replace hardcoded values in `Set Refresh Token and App ID` with environment variables using expressions like `{{$env["REFRESH_TOKEN"]}}`.

13. **Test the workflow by:**  
    - Triggering the Schedule Trigger node manually.  
    - Calling the manual reset webhook.  
    - Calling the token retrieval webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The token cache is scoped only to this workflow; other workflows should call this one or the webhook to retrieve tokens. | Workflow design note.                                                                              |
| An early refresh buffer of 90 seconds helps avoid token expiry during active API calls.               | Implemented in `Load to Static Data` node.                                                        |
| Manual reset webhook should be secured in production, e.g., with IP allowlist or secret authentication. | Security best practice.                                                                            |
| Consider moving sensitive values (refresh token, app ID, secret key) to environment variables for safer management. | Security best practice.                                                                            |
| Optional optimization: add an IF node after `Load to Static Data` to bypass refresh if token still valid. | Efficiency improvement suggestion.                                                                |
| Workflow static data is global and persistent but not shared cross-instance or cross-environment.    | Important consideration for multi-instance deployments.                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.