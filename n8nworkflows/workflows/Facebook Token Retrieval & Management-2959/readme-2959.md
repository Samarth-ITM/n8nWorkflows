Facebook Token Retrieval & Management

https://n8nworkflows.xyz/workflows/facebook-token-retrieval---management-2959


# Facebook Token Retrieval & Management

### 1. Workflow Overview

This workflow automates the retrieval and management of Facebook access tokens, specifically designed to obtain user short-lived tokens, exchange them for long-lived tokens, and facilitate the retrieval of Facebook Page tokens for posting and engagement purposes. It is targeted at developers, social media managers, and n8n users who need to integrate Facebook API authentication and token management into their applications or automation processes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Configuration Setup**: Receives incoming HTTP requests via a webhook and sets up configuration parameters.
- **1.2 Conditional Routing**: Determines whether the incoming request contains an authorization code or requires redirection to Facebook’s OAuth dialog.
- **1.3 Token Retrieval and Exchange**: Exchanges the authorization code for a short-lived token, then converts it into a long-lived token.
- **1.4 Response Handling**: Returns the long-lived token to the user or redirects to the Facebook OAuth URL.
- **1.5 Instructional Guidance**: Provides detailed instructions and notes for setup, scope customization, and token usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Configuration Setup

**Overview:**  
This block receives incoming HTTP requests from Facebook OAuth redirects and sets up essential configuration parameters such as app ID, app secret, and redirect URI.

**Nodes Involved:**  
- Webhook  
- Config

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Entry point for receiving Facebook OAuth redirect requests containing authorization codes.  
  - Configuration: Path set to `facebook-login`, response mode set to `responseNode` to allow dynamic responses.  
  - Inputs: External HTTP requests from Facebook OAuth flow.  
  - Outputs: Passes request data to the Config node.  
  - Edge Cases: Missing or malformed query parameters; invalid HTTP methods; webhook URL must be correctly configured in Facebook App settings.

- **Config**  
  - Type: Set  
  - Role: Defines static configuration parameters needed for OAuth requests.  
  - Configuration: Sets three string parameters:  
    - `fb_redirect_uri` = `https://localhost:5678/webhook/facebook-login`  
    - `app_id` = `12345678900000` (placeholder, must be replaced)  
    - `app_secret` = `aaabbbcccceedb6f9c91993c871` (placeholder, must be replaced)  
  - Inputs: Receives data from Webhook node.  
  - Outputs: Passes configuration and webhook data to the If node.  
  - Edge Cases: Incorrect or outdated app credentials will cause token retrieval failures.

---

#### 2.2 Conditional Routing

**Overview:**  
Determines if the incoming request contains a valid Facebook OAuth authorization code to proceed with token exchange or if it should redirect the user to Facebook’s OAuth dialog for login.

**Nodes Involved:**  
- If

**Node Details:**

- **If**  
  - Type: If  
  - Role: Checks if the query parameter `code` exists and is not empty, and not equal to the string "login".  
  - Configuration: Condition uses expression `={{ $('Webhook').first().json.query.code }}` with operator `notEmpty` and excludes the value "login".  
  - Inputs: Receives data from Config node.  
  - Outputs:  
    - True branch: Proceeds to Short-Lived Token retrieval.  
    - False branch: Proceeds to Redirect URL generation.  
  - Edge Cases: Missing or invalid `code` parameter; unexpected query values; case sensitivity is enabled.

---

#### 2.3 Token Retrieval and Exchange

**Overview:**  
Handles the OAuth token exchange process: first obtaining a short-lived token using the authorization code, then exchanging it for a long-lived token valid for approximately 60 days.

**Nodes Involved:**  
- Short-Lived Token  
- Long-Lived Token

**Node Details:**

- **Short-Lived Token**  
  - Type: HTTP Request  
  - Role: Sends a GET request to Facebook’s OAuth endpoint to exchange the authorization code for a short-lived access token.  
  - Configuration:  
    - URL: `https://graph.facebook.com/v21.0/oauth/access_token`  
    - Query parameters:  
      - `client_id` from Config node  
      - `redirect_uri` from Config node  
      - `client_secret` from Config node  
      - `code` from Webhook query parameter  
    - Retry on failure enabled with 5-second intervals.  
    - On error: continues workflow with error output (to avoid halting).  
  - Inputs: True branch from If node.  
  - Outputs: Passes short-lived token JSON to Long-Lived Token node.  
  - Edge Cases: Invalid or expired authorization code; network timeouts; Facebook API rate limits; malformed responses.

- **Long-Lived Token**  
  - Type: HTTP Request  
  - Role: Exchanges the short-lived token for a long-lived token using Facebook’s token exchange endpoint.  
  - Configuration:  
    - URL: `https://graph.facebook.com/v21.0/oauth/access_token`  
    - Query parameters:  
      - `grant_type` = `fb_exchange_token`  
      - `client_id` from Config node  
      - `client_secret` from Config node  
      - `fb_exchange_token` from Short-Lived Token node’s `access_token` field  
    - Retry on failure enabled with 5-second intervals.  
  - Inputs: Output from Short-Lived Token node.  
  - Outputs: Passes long-lived token JSON to Token response node.  
  - Edge Cases: Invalid short-lived token; network issues; API changes; token expiration.

---

#### 2.4 Response Handling

**Overview:**  
Delivers the final output to the user: either redirects to Facebook’s OAuth dialog URL or returns the long-lived token as plain text.

**Nodes Involved:**  
- Redirect URL  
- Redirect  
- Token

**Node Details:**

- **Redirect URL**  
  - Type: Code  
  - Role: Generates the Facebook OAuth authorization URL with required scopes and a random state parameter.  
  - Configuration:  
    - Reads `app_id`, `fb_redirect_uri` from Config node.  
    - Defines an array `correctScopes` with Facebook permissions such as `publish_video`, `pages_manage_posts`, etc.  
    - Generates a random `state` string for CSRF protection.  
    - Constructs the OAuth URL with parameters: client_id, redirect_uri, scope (comma-separated), response_type=code, and state.  
  - Inputs: False branch from If node or error branch from Short-Lived Token node.  
  - Outputs: Passes `authUrl` to Redirect node.  
  - Edge Cases: Incorrect scopes; invalid redirect URI; state parameter mismatch.

- **Redirect**  
  - Type: Respond to Webhook  
  - Role: Redirects the user’s browser to the generated Facebook OAuth URL.  
  - Configuration: Uses the `authUrl` from Redirect URL node as the redirect target.  
  - Inputs: Output from Redirect URL node.  
  - Outputs: Ends workflow with HTTP redirect response.  
  - Edge Cases: Redirect URL must be valid and accessible; browser compatibility.

- **Token**  
  - Type: Respond to Webhook  
  - Role: Returns the long-lived access token as a plain text HTTP response.  
  - Configuration: Responds with the `access_token` field from Long-Lived Token node’s JSON output.  
  - Inputs: Output from Long-Lived Token node.  
  - Outputs: Ends workflow with token response.  
  - Edge Cases: Token exposure risks; consider securing or storing tokens instead of direct display.

---

#### 2.5 Instructional Guidance

**Overview:**  
Provides detailed instructions, setup notes, and guidance for users to configure the Facebook app, webhook, scopes, and token usage.

**Nodes Involved:**  
- Sticky Note (Instructions to Get Token)  
- Sticky Note1 (Config Editing and Facebook App Setup)  
- Sticky Note2 (Guide to Editing Scopes)  
- Sticky Note3 (Optional Database Storage Note)

**Node Details:**

- **Sticky Note (Instructions to Get a 2-Month Facebook Token)**  
  - Type: Sticky Note  
  - Role: Contains step-by-step instructions for activating the Facebook app, accessing the webhook URL, checking token status, and retrieving page tokens.  
  - Content Highlights:  
    - Activate Facebook app in developer portal.  
    - Use tunneling tools like Ngrok or Cloudflare Tunnel for localhost webhook access.  
    - Use Facebook Access Token Debugger to verify tokens.  
    - Instructions on retrieving Page Tokens for posting.  
    - Link to GitHub repository for workflow JSON.  
  - Inputs/Outputs: None (informational only).

- **Sticky Note1 (Config Editing and Facebook App Setup)**  
  - Type: Sticky Note  
  - Role: Advises on editing key parameters (`fb_redirect_uri`, `app_id`, `app_secret`) and configuring Facebook Login settings in the Facebook App dashboard.  
  - Content Highlights:  
    - Add webhook URL to Valid OAuth Redirect URIs in Facebook App.  
    - Link to Facebook App settings page.  
  - Inputs/Outputs: None.

- **Sticky Note2 (Guide to Editing Scopes)**  
  - Type: Sticky Note  
  - Role: Explains how to customize the `correctScopes` array in the Redirect URL node to add, edit, or remove Facebook permissions.  
  - Content Highlights:  
    - Code examples for modifying scopes.  
    - Link to official Facebook Permissions documentation.  
  - Inputs/Outputs: None.

- **Sticky Note3 (Optional Database Storage Note)**  
  - Type: Sticky Note  
  - Role: Suggests extending the workflow by adding nodes to save tokens into a database instead of displaying them.  
  - Inputs/Outputs: None.

---

### 3. Summary Table

| Node Name         | Node Type              | Functional Role                         | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                                  |
|-------------------|------------------------|---------------------------------------|---------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook           | Webhook                | Receives OAuth redirect requests      | External HTTP       | Config                   |                                                                                                              |
| Config            | Set                    | Sets app credentials and redirect URI | Webhook             | If                       | Sticky Note1: Edit fb_redirect_uri, app_id, app_secret; configure Facebook Login settings in Facebook App.   |
| If                | If                     | Routes based on presence of auth code | Config              | Short-Lived Token, Redirect URL |                                                                                                              |
| Short-Lived Token  | HTTP Request           | Exchanges code for short-lived token  | If (true)           | Long-Lived Token, Redirect URL (on error) | Sticky Note3: Token displayed by default; can add DB storage node.                                           |
| Long-Lived Token   | HTTP Request           | Exchanges short-lived token for long-lived token | Short-Lived Token    | Token                    |                                                                                                              |
| Token             | Respond to Webhook     | Returns long-lived token to user      | Long-Lived Token     | None                     |                                                                                                              |
| Redirect URL      | Code                   | Generates Facebook OAuth URL           | If (false), Short-Lived Token (error) | Redirect                  | Sticky Note2: Guide to editing scopes; link to Facebook Permissions documentation.                           |
| Redirect          | Respond to Webhook     | Redirects user to Facebook OAuth URL  | Redirect URL        | None                     |                                                                                                              |
| Sticky Note       | Sticky Note            | Instructions for token retrieval and usage | None                | None                     | Sticky Note: Detailed instructions for app activation, token checking, page token retrieval, GitHub link.    |
| Sticky Note1      | Sticky Note            | Config editing and Facebook App setup | None                | None                     |                                                                                                              |
| Sticky Note2      | Sticky Note            | Guide to editing OAuth scopes          | None                | None                     |                                                                                                              |
| Sticky Note3      | Sticky Note            | Optional database storage suggestion   | None                | None                     |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Name: `Webhook`  
   - Type: Webhook  
   - Parameters:  
     - Path: `facebook-login`  
     - Response Mode: `responseNode`  
   - Purpose: Entry point to receive Facebook OAuth redirect requests.

2. **Create a Set Node for Configuration**  
   - Name: `Config`  
   - Type: Set  
   - Parameters:  
     - Add string fields:  
       - `fb_redirect_uri` = `https://localhost:5678/webhook/facebook-login` (adjust as needed)  
       - `app_id` = Your Facebook App ID  
       - `app_secret` = Your Facebook App Secret  
   - Connect `Webhook` output to `Config` input.

3. **Create an If Node for Conditional Routing**  
   - Name: `If`  
   - Type: If  
   - Parameters:  
     - Condition: Check if `{{$node["Webhook"].json.query.code}}` is not empty and not equal to `"login"`  
   - Connect `Config` output to `If` input.

4. **Create HTTP Request Node to Get Short-Lived Token**  
   - Name: `Short-Lived Token`  
   - Type: HTTP Request  
   - Parameters:  
     - HTTP Method: GET  
     - URL: `https://graph.facebook.com/v21.0/oauth/access_token`  
     - Query Parameters:  
       - `client_id` = `{{$node["Config"].json.app_id}}`  
       - `redirect_uri` = `{{$node["Config"].json.fb_redirect_uri}}`  
       - `client_secret` = `{{$node["Config"].json.app_secret}}`  
       - `code` = `{{$node["Webhook"].json.query.code}}`  
     - Retry on Fail: Enabled, wait 5000ms between tries  
     - On Error: Continue with error output  
   - Connect `If` node’s true output to this node.

5. **Create HTTP Request Node to Get Long-Lived Token**  
   - Name: `Long-Lived Token`  
   - Type: HTTP Request  
   - Parameters:  
     - HTTP Method: GET  
     - URL: `https://graph.facebook.com/v21.0/oauth/access_token`  
     - Query Parameters:  
       - `grant_type` = `fb_exchange_token`  
       - `client_id` = `{{$node["Config"].json.app_id}}`  
       - `client_secret` = `{{$node["Config"].json.app_secret}}`  
       - `fb_exchange_token` = `{{$node["Short-Lived Token"].json.access_token}}`  
     - Retry on Fail: Enabled, wait 5000ms between tries  
   - Connect `Short-Lived Token` output to this node.

6. **Create Respond to Webhook Node to Return Token**  
   - Name: `Token`  
   - Type: Respond to Webhook  
   - Parameters:  
     - Respond With: Text  
     - Response Body: `{{$node["Long-Lived Token"].json.access_token}}`  
   - Connect `Long-Lived Token` output to this node.

7. **Create Code Node to Generate Redirect URL**  
   - Name: `Redirect URL`  
   - Type: Code  
   - Parameters:  
     - JavaScript code:  
       ```javascript
       const config = $node["Config"].json;

       const generateRandomState = () => {
         return Math.random().toString(36).substring(2, 15) +
                Math.random().toString(36).substring(2, 15);
       }
       const state = generateRandomState();

       const correctScopes = [
         'publish_video',
         'pages_show_list',
         'business_management',
         'pages_read_engagement',
         'pages_read_user_content',
         'pages_manage_metadata',
         'pages_manage_posts',
         'pages_manage_engagement',
       ];

       const authUrl = `https://www.facebook.com/v22.0/dialog/oauth?client_id=${config.app_id}&redirect_uri=${config.fb_redirect_uri}&scope=${correctScopes.join(',')}&response_type=code&state=${state}`;

       return [{ authUrl }];
       ```
   - Connect `If` node’s false output and `Short-Lived Token` node’s error output to this node.

8. **Create Respond to Webhook Node for Redirect**  
   - Name: `Redirect`  
   - Type: Respond to Webhook  
   - Parameters:  
     - Respond With: Redirect  
     - Redirect URL: `{{$json.authUrl}}`  
   - Connect `Redirect URL` output to this node.

9. **Add Sticky Notes for Instructions and Guidance**  
   - Create four Sticky Note nodes with the content as described in section 2.5.  
   - Position them for user reference; they do not connect to the workflow logic.

10. **Configure Credentials**  
    - Set up Facebook App credentials in n8n’s credential manager or directly in the Config node as strings.  
    - Ensure the Facebook App has the OAuth redirect URI configured to match `fb_redirect_uri`.  
    - Use tunneling tools (Ngrok, Cloudflare Tunnel) if running locally.

11. **Activate the Workflow**  
    - Test by visiting the webhook URL with `?type=login` to initiate the OAuth flow.  
    - Follow the Facebook login and authorization steps.  
    - Verify token retrieval and display.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Run n8n with a tunnel (Ngrok, Cloudflare Tunnel) to expose localhost webhook URL for Facebook OAuth redirects.                 | Sticky Note instructions; required for local development.                                            |
| Facebook Access Token Debugger helps verify token validity, expiration, and permissions.                                        | https://developers.facebook.com/tools/debug/accesstoken/                                             |
| Facebook Permissions documentation provides a comprehensive list of available OAuth scopes.                                    | https://developers.facebook.com/docs/permissions/                                                    |
| To post on Facebook Pages, obtain Page Tokens via the `/me/accounts` endpoint using the User Token.                            | Explained in Sticky Note instructions.                                                                |
| GitHub repository with workflow JSON and updates: https://github.com/luuhung93/n8n-json                                        | Provided in Sticky Note for reference and updates.                                                    |
| Consider securing tokens by storing them in a database instead of displaying them directly to avoid security risks.            | Suggested in Sticky Note3.                                                                             |
| Ensure Facebook App is activated (status switched to Active) in the developer portal before testing OAuth flows.               | Sticky Note instructions.                                                                              |

---

This documentation provides a thorough understanding of the Facebook Token Retrieval & Management workflow, enabling users and automation agents to reproduce, modify, and troubleshoot the process effectively.