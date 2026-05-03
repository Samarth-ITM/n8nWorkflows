SharePoint List Fetch with OAuth Token

https://n8nworkflows.xyz/workflows/sharepoint-list-fetch-with-oauth-token-2527


# SharePoint List Fetch with OAuth Token

### 1. Workflow Overview

This workflow automates the retrieval of data from a SharePoint list using OAuth 2.0 token-based authentication. It is designed for teams who want to securely connect to SharePoint Online via Microsoft Azure Active Directory credentials and pull SharePoint list items directly into n8n.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Initialization:** Periodically triggers the workflow and initializes necessary variables such as tenant ID.
- **1.2 OAuth Token Generation:** Authenticates with Microsoft Azure AD OAuth endpoint to generate an access token.
- **1.3 SharePoint List Data Fetch:** Uses the acquired OAuth token to query a specific SharePoint list using the SharePoint REST API.
- **1.4 Security Advisory:** Annotations reminding users about security best practices for handling sensitive OAuth credentials.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Initialization

- **Overview:**  
This block triggers the workflow on a schedule and sets the `tenant_id` which is required for OAuth token generation.

- **Nodes Involved:**  
  - Schedule Trigger  
  - setTenant

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow execution on a recurring, periodic basis (the exact interval is user-configurable).  
    - *Configuration:* Uses a time interval trigger, default interval unspecified (empty array in JSON).  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Connects to `setTenant`.  
    - *Edge Cases:* Misconfiguration of interval can lead to no triggers or too frequent triggering.  
    - *Version:* 1.2  

  - **setTenant**  
    - *Type:* Set  
    - *Role:* Defines the `tenant_id` variable which is essential for OAuth token request URL construction.  
    - *Configuration:* Sets the string variable `tenant_id` with a placeholder value (empty by default - must be replaced by user).  
    - *Inputs:* Receives trigger data from `Schedule Trigger`.  
    - *Outputs:* Connects to `Generate OAuth Token`.  
    - *Edge Cases:* Empty or incorrect `tenant_id` will cause OAuth token request failures.  
    - *Version:* 3.4  

#### 1.2 OAuth Token Generation

- **Overview:**  
This block obtains an OAuth access token from the Microsoft Azure AD OAuth endpoint using client credentials. The token is needed to authenticate subsequent SharePoint API calls.

- **Nodes Involved:**  
  - Generate OAuth Token

- **Node Details:**

  - **Generate OAuth Token**  
    - *Type:* HTTP Request  
    - *Role:* Sends a POST request to the Microsoft OAuth endpoint to retrieve an access token using client credentials flow.  
    - *Configuration:*  
      - URL dynamically constructed using `tenant_id` from previous node: `https://accounts.accesscontrol.windows.net/{{ $json.tenant_id }}/tokens/oAuth/2`  
      - Method: POST  
      - Body parameters:  
        - `grant_type`: `client_credentials`  
        - `client_id`: must be replaced by user’s Azure AD Application ID  
        - `client_secret`: must be replaced by user’s Azure AD Application Secret  
        - `resource`: URL of the SharePoint domain (e.g. `https://your-sharepoint-domain.sharepoint.com`)  
    - *Inputs:* Receives data containing `tenant_id` from `setTenant`.  
    - *Outputs:* Access token response passed to `Fetch SharePoint List`.  
    - *Key Expressions:* Uses double curly braces to resolve variables.  
    - *Edge Cases:*  
      - Incorrect tenant_id, client_id or client_secret lead to authentication errors (401 Unauthorized).  
      - Network or timeout errors possible.  
      - Expired or revoked credentials cause failures.  
    - *Version:* 2  

#### 1.3 SharePoint List Data Fetch

- **Overview:**  
Using the OAuth token from the previous step, this block queries a specific SharePoint list to retrieve its items.

- **Nodes Involved:**  
  - Fetch SharePoint List

- **Node Details:**

  - **Fetch SharePoint List**  
    - *Type:* HTTP Request  
    - *Role:* Sends a GET request to SharePoint REST API to fetch list items.  
    - *Configuration:*  
      - URL: `https://{your-sharepoint-domain}.sharepoint.com/_api/web/lists/getbytitle('YourListTitle')/items` (user must replace placeholders)  
      - Headers:  
        - `Accept`: `application/json;odata=nometadata` (minimal metadata for response)  
        - `Content-Type`: `application/json;odata=verbose`  
        - `Authorization`: `Bearer {{Token}}` — the OAuth token obtained from the previous node; note that the token variable name should match the actual token field returned by the OAuth call (may require expression adjustment).  
    - *Inputs:* Receives OAuth token from `Generate OAuth Token`.  
    - *Outputs:* Outputs SharePoint list data in JSON format.  
    - *Edge Cases:*  
      - Token missing or invalid leads to 401 Unauthorized errors.  
      - Incorrect SharePoint domain or list title causes 404 Not Found or other API errors.  
      - API throttling or rate limits may cause delays or failures.  
    - *Version:* 2  

#### 1.4 Security Advisory (Sticky Note)

- **Overview:**  
Provides best practices and security warnings regarding sensitive credential management.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Role:* Communicates to users not to hardcode sensitive credentials like `tenant_id`, `client_id`, and `client_secret`. Advises use of secure vaults such as HashiCorp Vault or Google Cloud Secret Manager.  
    - *Configuration:* Contains markdown text; no inputs or outputs.  
    - *Edge Cases:* Not applicable.  
    - *Version:* 1  

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                      | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                 |
|----------------------|--------------------|------------------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger   | Periodically starts the workflow   | —                     | setTenant             |                                                                                             |
| setTenant            | Set                | Sets tenant_id variable             | Schedule Trigger      | Generate OAuth Token  |                                                                                             |
| Generate OAuth Token | HTTP Request       | Generates OAuth access token        | setTenant             | Fetch SharePoint List |                                                                                             |
| Fetch SharePoint List| HTTP Request       | Fetches SharePoint list items       | Generate OAuth Token  | —                     |                                                                                             |
| Sticky Note          | Sticky Note        | Security reminder on credentials    | —                     | —                     | ## Never expose or hard code below values tenant_id, client_id, client_secret. Always save these either in secure vault like hashicorp or GCP Secret Manager. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set desired interval (e.g., every hour, daily). Leave default or configure as needed.  
   - No credentials required.  

2. **Create a Set node named `setTenant`:**  
   - Type: Set  
   - Add a string field named `tenant_id`.  
   - Set its value to your Azure AD Tenant ID (e.g., `yourtenant.onmicrosoft.com` or tenant GUID).  
   - Connect Schedule Trigger → setTenant.  

3. **Create an HTTP Request node named `Generate OAuth Token`:**  
   - Type: HTTP Request  
   - HTTP Method: POST  
   - URL: `https://accounts.accesscontrol.windows.net/{{ $json.tenant_id }}/tokens/oAuth/2` (use expression editor to insert `tenant_id` from previous node)  
   - Body Parameters (form-urlencoded or JSON depending on API):  
     - `grant_type`: `client_credentials`  
     - `client_id`: Your Azure AD Application (client) ID (replace placeholder)  
     - `client_secret`: Your Azure AD Application Secret (replace placeholder)  
     - `resource`: `https://{your-sharepoint-domain}.sharepoint.com` (replace with your actual SharePoint Online domain URL)  
   - Connect setTenant → Generate OAuth Token.  
   - No special credentials, but ensure secrets are stored securely (e.g., in environment variables or n8n credentials).  

4. **Create an HTTP Request node named `Fetch SharePoint List`:**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://{your-sharepoint-domain}.sharepoint.com/_api/web/lists/getbytitle('YourListTitle')/items` (replace placeholders accordingly)  
   - Headers:  
     - `Accept`: `application/json;odata=nometadata`  
     - `Content-Type`: `application/json;odata=verbose`  
     - `Authorization`: Use an expression to set `Bearer {{ $json.access_token }}` or equivalent depending on the token field returned by the OAuth call.  
   - Connect Generate OAuth Token → Fetch SharePoint List.  

5. **Create a Sticky Note for security best practices:**  
   - Add a Sticky Note node with the content:  
     ```
     ## Never expose or hard code below values  
     **tenant_id,client_id,client_secret**  
     Always save these either in secure vault like hashicorp or GCP Secret Manager.
     ```  
   - Position it prominently.  

6. **Credentials Setup:**  
   - Store `client_id` and `client_secret` securely as environment variables or use n8n credentials management system for OAuth2 client credentials.  
   - Replace placeholders in nodes with correct values or use n8n variables referencing credentials.

7. **Test Workflow:**  
   - Run the workflow manually for the first time.  
   - Check the OAuth token response and ensure the token is correctly passed to the SharePoint API node.  
   - Verify SharePoint list data is returned as expected.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Never expose or hardcode sensitive OAuth credentials; use secure vaults such as HashiCorp or GCP SM | Security best practice reminder included as sticky note in the workflow                             |
| Replace placeholder values `{tenant_id}`, `{client_id}`, `{client_secret}`, and SharePoint URLs      | Setup steps in workflow description; critical to ensure proper API authentication and endpoint URLs |
| SharePoint REST API documentation: https://learn.microsoft.com/en-us/sharepoint/dev/sp-add-ins/get-to-know-the-sharepoint-rest-service | Microsoft official documentation for SharePoint API                                               |
| Microsoft OAuth 2.0 Client Credentials Flow: https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow | For understanding OAuth token generation with client credentials                                  |

---

This document fully covers the structure, purpose, nodes, and reproduction steps for the "SharePoint List Fetch with OAuth Token" workflow in n8n, allowing users and automation agents to understand, modify, and recreate the workflow securely and reliably.