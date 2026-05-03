User verification and login using Auth0

https://n8nworkflows.xyz/workflows/user-verification-and-login-using-auth0-2408


# User verification and login using Auth0

### 1. Workflow Overview

This workflow provides a user authentication and login mechanism using Auth0 as the identity provider. It targets developers and teams who want a straightforward login solution integrated with n8n, enabling users to authenticate via email (including Gmail) or other social providers by extending the authorization URL.

The workflow is logically divided into three main blocks:

- **1.1 Initialization and Login Request Handling:** Receives initial login webhook calls, sets application configuration, and redirects users to the Auth0 hosted login page.
- **1.2 Authorization Code Handling and Token Exchange:** Handles the callback after user login, verifies the presence of an authorization code, and exchanges it for an access token.
- **1.3 User Information Retrieval:** Uses the access token to fetch user profile details from Auth0.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Login Request Handling

- **Overview:**  
  This block manages the initial login request from the user. It sets the Auth0 application details such as domain, client ID, and server URL, then redirects the user to the Auth0-hosted login page.

- **Nodes Involved:**  
  - `/login` (Webhook)  
  - Set Application Details (Set)  
  - Open Auth Webpage (Respond to Webhook)  
  - Sticky Notes: Sticky Note1, Sticky Note3, Sticky Note7, Sticky Note

- **Node Details:**  

  - `/login`  
    - Type: Webhook  
    - Role: Entry point for login requests from users.  
    - Configuration: Listens on path `/login` with response mode set to wait for `Respond to Webhook` node.  
    - Inputs: External HTTP request.  
    - Outputs: Forwards to `Set Application Details`.  
    - Edge Cases: Incoming requests without proper parameters are handled downstream.  

  - Set Application Details  
    - Type: Set  
    - Role: Stores Auth0 app configuration parameters (`domain`, `client_id`, `my_server`) for reuse.  
    - Configuration: Parameters must be filled manually with values from the Auth0 application dashboard.  
    - Inputs: From `/login` webhook.  
    - Outputs: Connects to `Open Auth Webpage`.  
    - Edge Cases: Missing or incorrect values will cause authentication failures or redirection errors.  

  - Open Auth Webpage  
    - Type: Respond to Webhook  
    - Role: Redirects the user to the Auth0 authorization URL to start the login process.  
    - Configuration: Redirect URL dynamically constructed with parameters: `response_type=code`, scopes (`openid email profile image name`), `client_id`, and `redirect_uri` pointing to the `/receive-token` webhook.  
    - Inputs: From `Set Application Details`.  
    - Outputs: HTTP redirect response to the user.  
    - Edge Cases: Malformed URLs or missing parameters result in failed redirection.  
    - Notes: The sticky note suggests appending `&connection=github` to the URL to enable GitHub or other social logins.  

  - Sticky Notes (Informational)  
    - Provide setup instructions, usage steps, and tips for extending social login capabilities.  

#### 2.2 Authorization Code Handling and Token Exchange

- **Overview:**  
  This block processes the callback from Auth0 after user login. It verifies the presence of an authorization code, and if valid, exchanges it for an access token using Auth0's OAuth token endpoint.

- **Nodes Involved:**  
  - `/receive-token` (Webhook)  
  - If (Conditional)  
  - No Code Found (Stop and Error)  
  - Set Application Details1 (Set)  
  - Request Access Token (HTTP Request)  
  - Sticky Notes: Sticky Note4, Sticky Note5

- **Node Details:**  

  - `/receive-token`  
    - Type: Webhook  
    - Role: Receives Auth0's redirect callback containing the authorization code.  
    - Configuration: Listens on path `/receive-token` with response mode set to return the last node's output.  
    - Inputs: External HTTP request from Auth0 containing query parameters.  
    - Outputs: Passes data to the `If` node for code validation.  
    - Edge Cases: Missing or malformed query parameters can interrupt flow.  

  - If  
    - Type: If  
    - Role: Checks if the `code` query parameter exists in the incoming webhook data.  
    - Configuration: Condition tests for existence of `query.code` in input JSON.  
    - Inputs: From `/receive-token`.  
    - Outputs: Two branches:  
      - True: Proceed to `Set Application Details1`.  
      - False: Proceed to `No Code Found`.  
    - Edge Cases: Absence of `code` triggers error handling downstream.  

  - No Code Found  
    - Type: Stop and Error  
    - Role: Terminates workflow execution with an error message if no authorization code was found.  
    - Configuration: Static error message `"Couldn't get authorization code!"`.  
    - Inputs: From `If` node false branch.  
    - Outputs: None (ends execution).  

  - Set Application Details1  
    - Type: Set  
    - Role: Sets the Auth0 app details plus client secret required for token exchange.  
    - Configuration: Fields include `domain`, `client_id`, `my_server`, and `client_secret`. Values must be set manually from Auth0 dashboard.  
    - Inputs: From `If` node true branch.  
    - Outputs: Forwards to `Request Access Token`.  
    - Edge Cases: Missing or incorrect client secret causes token requests to fail.  

  - Request Access Token  
    - Type: HTTP Request  
    - Role: Sends POST request to Auth0 OAuth token endpoint to exchange the authorization code for an access token.  
    - Configuration:  
      - URL: `{{ $json.domain }}/oauth/token` (dynamic)  
      - Method: POST  
      - Headers: `content-type: application/x-www-form-urlencoded` (note: despite JSON body, header suggests form-encoded - may cause issues, potential caveat)  
      - Body: JSON containing grant_type, code, client_id, client_secret, redirect_uri, and audience.  
      - Sends both body and headers.  
    - Inputs: From `Set Application Details1`.  
    - Outputs: Passes token response to `Get Userinfo`.  
    - Edge Cases:  
      - Auth failures due to invalid client credentials or expired code.  
      - Network timeouts or endpoint errors.  
      - Header/body content-type mismatch might cause errors depending on Auth0 expectations.  

#### 2.3 User Information Retrieval

- **Overview:**  
  Using the access token obtained, this block queries the Auth0 userinfo endpoint to retrieve user profile data, confirming authentication success.

- **Nodes Involved:**  
  - Get Userinfo (HTTP Request)  
  - Sticky Note2

- **Node Details:**  

  - Get Userinfo  
    - Type: HTTP Request  
    - Role: Fetches user profile from Auth0 using the access token.  
    - Configuration:  
      - URL: `{{ $('Set Application Details1').item.json.domain }}/userinfo` (dynamic)  
      - Method: GET (default)  
      - Headers: Authorization Bearer token set dynamically from previous node's access_token.  
      - Sends headers only (no body).  
    - Inputs: From `Request Access Token`.  
    - Outputs: Provides user information JSON as the final webhook response to the client.  
    - Edge Cases:  
      - Invalid or expired access tokens cause 401 errors.  
      - Network or endpoint issues.  

  - Sticky Note2  
    - Describes this step as the final user login confirmation step, returning user data.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                          | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                           |
|-----------------------|---------------------|----------------------------------------|-----------------------|--------------------------|-----------------------------------------------------------------------------------------------------|
| /login                | Webhook             | Entry point for login requests          | External HTTP request  | Set Application Details   |                                                                                                     |
| Set Application Details| Set                 | Sets Auth0 app config params            | /login                | Open Auth Webpage         |                                                                                                     |
| Open Auth Webpage      | Respond to Webhook   | Redirects user to Auth0 login page      | Set Application Details| None (HTTP redirect)      | "This step will return the authentication page to the user and let him login using gmail or by creating a new account." |
| Sticky Note            | Sticky Note         | Info: Add &connection=github etc.       |                       |                          | "You can also add &connection=github to end of authorize URL in order to get user to login via Github, Facebook, etc" |
| /receive-token         | Webhook             | Receives Auth0 callback with code       | External HTTP request  | If                       |                                                                                                     |
| If                    | If                  | Checks presence of authorization code   | /receive-token         | Set Application Details1, No Code Found |                                                                                                     |
| No Code Found          | Stop and Error       | Stops workflow if no code found          | If (false branch)      | None                     |                                                                                                     |
| Set Application Details1| Set                 | Sets Auth0 app params + client secret   | If (true branch)       | Request Access Token      |                                                                                                     |
| Request Access Token   | HTTP Request        | Exchanges code for access token          | Set Application Details1| Get Userinfo             |                                                                                                     |
| Get Userinfo           | HTTP Request        | Retrieves user profile using access token| Request Access Token   | None (final response)     | "This step will return the authentication page to the user and let him login using gmail or by creating a new account." |
| Sticky Note1           | Sticky Note         | Setup instructions for Auth0 app         |                       |                          | "1. First, go to https://auth0.com and create a Single Page Application. ..." (setup instructions)   |
| Sticky Note2           | Sticky Note         | Describes the userinfo fetch step        |                       |                          | "This step will return the authentication page to the user and let him login using gmail or by creating a new account." |
| Sticky Note3           | Sticky Note         | Header for authentication step           |                       |                          | "Step 1: Authentication"                                                                             |
| Sticky Note4           | Sticky Note         | Header for access token retrieval step   |                       |                          | "Step 2: Get Access Token"                                                                           |
| Sticky Note5           | Sticky Note         | Explains token exchange step              |                       |                          | "If Step 1 was successful, Auth0 will automatically call Step 2 in its callback with a code..."       |
| Sticky Note7           | Sticky Note         | Instructions to fill app details and login URL |                       |                          | "2. Fill in Set Application Details and Set Application Details1\n3. Login from https://<n8n server address>/webhook/login!" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the `/login` Webhook Node:**  
   - Type: Webhook  
   - Path: `login`  
   - Response Mode: `Response Node` (wait for downstream response)  
   - Purpose: Entry point for user login requests.

2. **Create the Set Application Details Node:**  
   - Type: Set  
   - Add fields:  
     - `domain` (string): Your Auth0 domain URL, e.g., `https://your-domain.auth0.com`  
     - `client_id` (string): Your Auth0 client ID  
     - `my_server` (string): Your n8n server URL, e.g., `http://localhost:5678` or your public server address  
   - Connect `/login` webhook node output to this node.

3. **Create the Respond to Webhook Node Named Open Auth Webpage:**  
   - Type: Respond to Webhook  
   - Parameters:  
     - Respond with: `Redirect`  
     - Redirect URL:  
     ```  
     {{ $json.domain }}/authorize?response_type=code&scope=openid+email+profile+image+name&client_id={{ $json.client_id }}&redirect_uri={{ $json.my_server }}/webhook/receive-token  
     ```  
   - Connect `Set Application Details` node output to this node.

4. **Create the `/receive-token` Webhook Node:**  
   - Type: Webhook  
   - Path: `receive-token`  
   - Response Mode: `Last Node` (wait for final response)  
   - Purpose: Handle callback from Auth0 post-login.

5. **Create an If Node to Check for Authorization Code:**  
   - Condition: Check if `{{$json.query.code}}` exists (string exists operator)  
   - Connect `/receive-token` webhook node output to this node.

6. **Create the No Code Found Stop and Error Node:**  
   - Type: Stop and Error  
   - Error message: `"Couldn't get authorization code!"`  
   - Connect the `If` node false branch to this node.

7. **Create the Set Application Details1 Node:**  
   - Type: Set  
   - Add fields:  
     - `domain` (string): Your Auth0 domain URL  
     - `client_id` (string): Your Auth0 client ID  
     - `client_secret` (string): Your Auth0 client secret  
     - `my_server` (string): Your n8n server URL  
   - Connect the `If` node true branch to this node.

8. **Create the Request Access Token HTTP Request Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `={{ $json.domain }}/oauth/token`  
   - Headers:  
     - `content-type: application/x-www-form-urlencoded`  
   - Body Type: JSON (Note: Despite header, body is JSON formatted)  
   - Body Content:  
   ```json
   {
     "grant_type": "authorization_code",
     "code": "{{ $json.query.code }}",
     "client_id": "{{ $json.client_id }}",
     "client_secret": "{{ $json.client_secret }}",
     "redirect_uri": "{{ $json.my_server }}/webhook/receive-token",
     "audience": "{{ $json.domain }}/api/v2/"
   }
   ```  
   - Send Body: true  
   - Send Headers: true  
   - Connect `Set Application Details1` to this node.

9. **Create the Get Userinfo HTTP Request Node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $('Set Application Details1').item.json.domain }}/userinfo`  
   - Headers:  
     - `Authorization: Bearer {{ $json.access_token }}`  
   - Send Headers: true  
   - Connect `Request Access Token` node output to this node.

10. **Connect the `/receive-token` Webhook node output to the `If` node.**  
11. **Connect the `Get Userinfo` node output to the `/receive-token` webhook node's response.**

12. **Credentials Setup:**  
    - No explicit credentials node is required because all HTTP requests use direct parameters.  
    - Ensure that the Auth0 domain, client ID, and client secret are correctly set in the Set nodes.

13. **Testing:**  
    - Run the workflow active on your n8n instance.  
    - Visit `https://<your n8n server>/webhook/login` to initiate login.  
    - Complete login via Auth0 hosted page.  
    - Upon successful login, user info JSON will be returned by the `/receive-token` webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| To use this workflow, create a Single Page Application on https://auth0.com and configure allowed callback URLs to include your n8n webhook URLs.    | Setup instructions in Sticky Note1 and Sticky Note7 |
| Add `&connection=github` to the authorization URL query string to enable login via Github, Facebook, and other social providers.                      | Sticky Note (near Open Auth Webpage node)        |
| Replace `localhost` with your n8n server's public address if you are not running n8n locally.                                                          | Sticky Note1 content                              |
| The workflow supports login via any email address or Gmail by default through Auth0's UI.                                                              | Workflow description                              |
| For security, ensure your client_secret is kept private and properly configured in the Set Application Details1 node.                                  | General best practice                             |

---

This concludes the detailed analysis and reproduction instructions for the "Auth0 User Login" workflow. It provides a robust, extensible foundation for adding secure user authentication to n8n workflows with minimal setup.