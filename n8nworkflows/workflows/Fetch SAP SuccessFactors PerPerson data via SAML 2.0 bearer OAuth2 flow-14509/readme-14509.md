Fetch SAP SuccessFactors PerPerson data via SAML 2.0 bearer OAuth2 flow

https://n8nworkflows.xyz/workflows/fetch-sap-successfactors-perperson-data-via-saml-2-0-bearer-oauth2-flow-14509


# Fetch SAP SuccessFactors PerPerson data via SAML 2.0 bearer OAuth2 flow

## 1. Workflow Overview

This workflow retrieves **PerPerson** data from **SAP SuccessFactors OData v2** using the **SAML 2.0 bearer assertion OAuth2 flow**, then flattens nested employment records into a simple item-per-person-employment output.

It is designed for cases where:
- You need to call SuccessFactors APIs from n8n
- Standard n8n OAuth2 credentials are not suitable for this proprietary authentication flow
- You want a reproducible pattern for fetching person and employment data securely with a registered OAuth2 client and private key

### 1.1 Input and Configuration
The workflow starts manually and defines all required environment-specific values in a Set node, including API endpoints, tenant identifiers, OAuth client details, user identity, private key, and OData query parameters.

### 1.2 SAML Assertion and OAuth Token Retrieval
The workflow first requests a **signed SAML assertion** from the SuccessFactors identity provider endpoint, then exchanges that assertion for an **OAuth bearer access token**.

### 1.3 SuccessFactors Data Retrieval
Using the access token, the workflow calls the `/PerPerson` OData endpoint and requests selected person fields plus expanded employment navigation data.

### 1.4 Result Flattening
Because SuccessFactors OData responses are nested, the workflow transforms the payload into one n8n item per person-employment record, while still preserving persons who have no employment entries.

---

## 2. Block-by-Block Analysis

## 2.1 Block: Manual Start and Runtime Configuration

**Overview:**  
This block initializes the workflow and centralizes all values required for authentication and querying. It acts as the single source of truth for URLs, credentials, and OData parameters.

**Nodes Involved:**  
- When clicking 'Test workflow'
- Configuration

### Node Details

#### 1) When clicking 'Test workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for testing and ad hoc execution.
- **Configuration choices:**  
  No custom configuration. Execution begins only when the user clicks the test/run button in the editor.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Configuration`
- **Version-specific requirements:**  
  Type version `1`; standard manual trigger behavior.
- **Edge cases or potential failure types:**  
  - No runtime failure expected from the node itself
  - Not suitable for unattended production execution
- **Sub-workflow reference:**  
  None.

#### 2) Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines all static and operator-supplied configuration values as fields in the current item.
- **Configuration choices:**  
  The node creates the following fields:
  - `SF_API_BASE_URL`: Base OData v2 API URL
  - `SF_IDP_URL`: SAML assertion endpoint
  - `SF_TOKEN_URL`: OAuth token endpoint
  - `company_id`: SuccessFactors company/tenant ID
  - `client_id`: OAuth client API key
  - `user_id`: SuccessFactors user on whose behalf the token is issued
  - `private_key`: Base64 content of the private key from PEM, without header/footer/line breaks
  - `top`: OData `$top` limit, default `20`
  - `select`: OData `$select` fields, default `personIdExternal,perPersonUuid`
- **Key expressions or variables used:**  
  The node creates values later referenced as:
  - `$('Configuration').first().json.SF_IDP_URL`
  - `$('Configuration').first().json.SF_TOKEN_URL`
  - `$('Configuration').first().json.client_id`
  - `$('Configuration').first().json.user_id`
  - `$('Configuration').first().json.private_key`
  - `$('Configuration').first().json.company_id`
  - `$('Configuration').first().json.SF_API_BASE_URL`
  - `$('Configuration').first().json.top`
  - `$('Configuration').first().json.select`
- **Input and output connections:**  
  - Input: `When clicking 'Test workflow'`
  - Output: `Get SAML Assertion`
- **Version-specific requirements:**  
  Type version `3.4`; uses assignments-based Set node configuration.
- **Edge cases or potential failure types:**  
  - Invalid URLs cause downstream HTTP failures
  - Empty or malformed `private_key` causes assertion generation failure
  - Incorrect `client_id`, `company_id`, or `user_id` leads to authentication rejection
  - If `select` omits required fields expected by later processing, output may contain nulls
- **Sub-workflow reference:**  
  None.

---

## 2.2 Block: SAML Assertion and Bearer Token Acquisition

**Overview:**  
This block implements the proprietary SuccessFactors authentication flow. It first requests a SAML 2.0 assertion from `/oauth/idp`, then exchanges that assertion at `/oauth/token` for an access token.

**Nodes Involved:**  
- Get SAML Assertion
- Get Bearer Token

### Node Details

#### 3) Get SAML Assertion
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a form-encoded POST request to the SuccessFactors identity provider endpoint to obtain a signed SAML assertion.
- **Configuration choices:**  
  - Method: `POST`
  - Content type: `application/x-www-form-urlencoded`
  - URL comes from the configuration node
  - Body fields:
    - `client_id`
    - `user_id`
    - `token_url`
    - `private_key`
- **Key expressions or variables used:**  
  - URL: `={{ $('Configuration').first().json.SF_IDP_URL }}`
  - `client_id`: `={{ $('Configuration').first().json.client_id }}`
  - `user_id`: `={{ $('Configuration').first().json.user_id }}`
  - `token_url`: `={{ $('Configuration').first().json.SF_TOKEN_URL }}`
  - `private_key`: `={{ $('Configuration').first().json.private_key }}`
- **Input and output connections:**  
  - Input: `Configuration`
  - Output: `Get Bearer Token`
- **Version-specific requirements:**  
  Type version `4.3`; uses the newer HTTP Request node parameter structure.
- **Edge cases or potential failure types:**  
  - 400/401/403 if OAuth client registration is wrong or not authorized
  - Invalid or incorrectly pasted private key
  - `user_id` not permitted for the client or not valid in SuccessFactors
  - Network/DNS/TLS issues with the endpoint
  - If the response shape changes and does not include the expected assertion field used downstream
- **Sub-workflow reference:**  
  None.

#### 4) Get Bearer Token
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Exchanges the returned SAML assertion for an OAuth bearer token.
- **Configuration choices:**  
  - Method: `POST`
  - Content type: `application/x-www-form-urlencoded`
  - Body fields:
    - `company_id`
    - `client_id`
    - `grant_type = urn:ietf:params:oauth:grant-type:saml2-bearer`
    - `assertion = {{$json.data}}`
- **Key expressions or variables used:**  
  - URL: `={{ $('Configuration').first().json.SF_TOKEN_URL }}`
  - `company_id`: `={{ $('Configuration').first().json.company_id }}`
  - `client_id`: `={{ $('Configuration').first().json.client_id }}`
  - `assertion`: `={{ $json.data }}`
- **Input and output connections:**  
  - Input: `Get SAML Assertion`
  - Output: `Fetch PerPerson from SF`
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - If `Get SAML Assertion` does not return the assertion in `data`, token exchange fails
  - Expired or invalid assertion
  - Wrong tenant/company ID
  - Unsupported client or permission mismatch
  - Response may not include `access_token` if the request is rejected
- **Sub-workflow reference:**  
  None.

---

## 2.3 Block: PerPerson Retrieval from SuccessFactors

**Overview:**  
This block calls the SuccessFactors OData API with the retrieved bearer token. It requests a limited set of person fields and expands employment relationships to return nested employment data in one call.

**Nodes Involved:**  
- Fetch PerPerson from SF

### Node Details

#### 5) Fetch PerPerson from SF
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs the authenticated GET request against the `/PerPerson` OData v2 endpoint.
- **Configuration choices:**  
  - URL: `SF_API_BASE_URL + /PerPerson`
  - Sends query parameters:
    - `$format=json`
    - `$top=<configured top>`
    - `$select=<configured select>`
    - `$expand=employmentNav`
  - Sends header:
    - `Authorization: Bearer {{$json.access_token}}`
- **Key expressions or variables used:**  
  - URL: `={{ $('Configuration').first().json.SF_API_BASE_URL }}/PerPerson`
  - `$top`: `={{ $('Configuration').first().json.top }}`
  - `$select`: `={{ $('Configuration').first().json.select }}`
  - Authorization header: `=Bearer {{ $json.access_token }}`
- **Input and output connections:**  
  - Input: `Get Bearer Token`
  - Output: `Flatten Results`
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Missing or expired `access_token`
  - Insufficient permissions for the selected user
  - Invalid OData field names in `$select`
  - API paging not implemented beyond `$top`
  - Large payloads if `$expand=employmentNav` returns many child records
  - OData response shape assumed to follow v2 structure
- **Sub-workflow reference:**  
  None.

---

## 2.4 Block: Flattening Nested OData Results

**Overview:**  
This block converts the nested SuccessFactors OData response into simple n8n items. It ensures that each employment record becomes its own item and also keeps persons with no employment records.

**Nodes Involved:**  
- Flatten Results

### Node Details

#### 6) Flatten Results
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes JavaScript to unpack `d.results` and nested `employmentNav.results`, then emits normalized output items.
- **Configuration choices:**  
  The code:
  - Reads `d.results` from the incoming JSON
  - Iterates through each person
  - Reads `employmentNav.results`
  - If no employments exist, emits one item with employment fields set to `null`
  - Otherwise emits one item per employment row
- **Key expressions or variables used:**  
  Internally uses:
  - `$input.first().json?.d?.results ?? []`
  - `person.employmentNav?.results ?? []`
  Output fields:
  - `personIdExternal`
  - `perPersonUuid`
  - `empStartDate`
  - `empEndDate`
  - `userId`
- **Input and output connections:**  
  - Input: `Fetch PerPerson from SF`
  - Output: none shown; terminal node
- **Version-specific requirements:**  
  Type version `2`; compatible with JavaScript Code node execution.
- **Edge cases or potential failure types:**  
  - If the upstream response body is not JSON or differs from expected OData v2 structure, output becomes empty
  - If `employmentNav` is absent, it is treated as no employment
  - Only the first input item is processed via `$input.first()`, which is fine here because the previous HTTP node emits one item
  - Additional expanded structures are ignored unless the code is extended
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Test workflow' | Manual Trigger | Manual workflow entry point for testing |  | Configuration |  |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | ## SuccessFactors read Person & Employment — via SAML 2.0 Bearer Assertion based on OAuth 2 configuration. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | **Note:** SuccessFactors uses a proprietary **SAML 2.0 Bearer Assertion** flow — n8n's built-in OAuth2 credential does not work here. Basic Authentication is not recommended as a secure authentication mechanism. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | ### Flow |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 1. **Configuration** — URLs, credentials, private key |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 2. **Get SAML Assertion** → POST to `/oauth/idp` |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 3. **Get Bearer Token** → POST to `/oauth/token` |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 4. **Fetch PerPerson** → OData v2 GET with Bearer token |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 5. **Flatten Results** → one item per person-employment record |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | Example URLs can be found in configuration node. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | ### Setup |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 1. Register OAuth2 client in SF Admin → `Go to Manage OAuth2 Client` Applications. Generate a new certificate from there. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 2. `Register new client application` and provide an application name and URL which are easy to understand. Both can be virtual and must not exist. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 3. Optional step: you can limit the client registration to a single user, which permissions will be relevant for the API in SuccessFactors. I recommend keeping this empty, since the SAML Assertion requires to limit on a user too. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 4. Press `Generate X.509 Certificate` and provide at least a virtual Common Name (CN) and limit the validity of the certificate to your needs. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 5. When generated you got the necessary certificate and **API Key** which is our `client_id`. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 6. Export Certificate.pem from the OAuth 2 Client registration in SuccessFactors. Open with a text editor and copy base64 content between `-----BEGIN ENCRYPTED PRIVATE KEY-----` and `-----END ENCRYPTED PRIVATE KEY-----`. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | --> That is your `private_key`. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | 3. Fill in the Configuration node. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | ### Extend |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - Add `personalInfoNav`, `jobInfoNav` to `` |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - Loop + `` for full pagination |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - Replace Manual Trigger with Schedule Trigger |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | ### 1 — Configure |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - `SF_API_BASE_URL` — OData v2 base URL |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - `SF_IDP_URL` — assertion endpoint |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - `SF_TOKEN_URL` — token endpoint |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - `company_id` — SF tenant ID |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - `client_id` — API Key from OAuth2 registration |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - `user_id` — **Mandatory.** SF user the token is issued for. Data access is scoped to this user's permissions — use a service account. |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - `private_key` — base64 body from the exported `.pem` (no headers, no line breaks) |
| Configuration | Set | Stores endpoints, tenant values, OAuth client data, user identity, private key, and OData query parameters | When clicking 'Test workflow' | Get SAML Assertion | - `top` / `select` — OData query params |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | ## SuccessFactors read Person & Employment — via SAML 2.0 Bearer Assertion based on OAuth 2 configuration. |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | **Note:** SuccessFactors uses a proprietary **SAML 2.0 Bearer Assertion** flow — n8n's built-in OAuth2 credential does not work here. Basic Authentication is not recommended as a secure authentication mechanism. |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | ### Flow |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | 1. **Configuration** — URLs, credentials, private key |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | 2. **Get SAML Assertion** → POST to `/oauth/idp` |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | 3. **Get Bearer Token** → POST to `/oauth/token` |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | 4. **Fetch PerPerson** → OData v2 GET with Bearer token |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | 5. **Flatten Results** → one item per person-employment record |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | Example URLs can be found in configuration node. |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | ### 2 — Get Token |
| Get SAML Assertion | HTTP Request | Requests signed SAML assertion from SuccessFactors IdP endpoint | Configuration | Get Bearer Token | **Get SAML Assertion** — POSTs `client_id`, `user_id`, `token_url`, `private_key` to `/oauth/idp`. Returns a signed SAML 2.0 XML assertion. |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | ## SuccessFactors read Person & Employment — via SAML 2.0 Bearer Assertion based on OAuth 2 configuration. |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | **Note:** SuccessFactors uses a proprietary **SAML 2.0 Bearer Assertion** flow — n8n's built-in OAuth2 credential does not work here. Basic Authentication is not recommended as a secure authentication mechanism. |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | ### Flow |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | 1. **Configuration** — URLs, credentials, private key |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | 2. **Get SAML Assertion** → POST to `/oauth/idp` |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | 3. **Get Bearer Token** → POST to `/oauth/token` |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | 4. **Fetch PerPerson** → OData v2 GET with Bearer token |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | 5. **Flatten Results** → one item per person-employment record |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | Example URLs can be found in configuration node. |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | ### 2 — Get Token |
| Get Bearer Token | HTTP Request | Exchanges SAML assertion for OAuth bearer token | Get SAML Assertion | Fetch PerPerson from SF | **Get Bearer Token** — POSTs assertion + `company_id` + `client_id` to `/oauth/token` with `grant_type=urn:ietf:params:oauth:grant-type:saml2-bearer`. Returns `access_token`. |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | ## SuccessFactors read Person & Employment — via SAML 2.0 Bearer Assertion based on OAuth 2 configuration. |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | **Note:** SuccessFactors uses a proprietary **SAML 2.0 Bearer Assertion** flow — n8n's built-in OAuth2 credential does not work here. Basic Authentication is not recommended as a secure authentication mechanism. |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | ### Flow |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | 1. **Configuration** — URLs, credentials, private key |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | 2. **Get SAML Assertion** → POST to `/oauth/idp` |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | 3. **Get Bearer Token** → POST to `/oauth/token` |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | 4. **Fetch PerPerson** → OData v2 GET with Bearer token |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | 5. **Flatten Results** → one item per person-employment record |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | Example URLs can be found in configuration node. |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | ### 3 — Fetch & Flatten |
| Fetch PerPerson from SF | HTTP Request | Retrieves PerPerson OData data with expanded employment navigation | Get Bearer Token | Flatten Results | **Fetch PerPerson** — GET `/PerPerson?=employmentNav` with `Authorization: Bearer <token>`. |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | ## SuccessFactors read Person & Employment — via SAML 2.0 Bearer Assertion based on OAuth 2 configuration. |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | **Note:** SuccessFactors uses a proprietary **SAML 2.0 Bearer Assertion** flow — n8n's built-in OAuth2 credential does not work here. Basic Authentication is not recommended as a secure authentication mechanism. |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | ### Flow |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | 1. **Configuration** — URLs, credentials, private key |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | 2. **Get SAML Assertion** → POST to `/oauth/idp` |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | 3. **Get Bearer Token** → POST to `/oauth/token` |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | 4. **Fetch PerPerson** → OData v2 GET with Bearer token |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | 5. **Flatten Results** → one item per person-employment record |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | Example URLs can be found in configuration node. |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | ### 3 — Fetch & Flatten |
| Flatten Results | Code | Normalizes nested OData person/employment response into flat items | Fetch PerPerson from SF |  | **Flatten Results** — unpacks `d.results` and `employmentNav.results`. Outputs one item per person-employment combination. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Keep default configuration
   - Name it: **When clicking 'Test workflow'**

3. **Add a Set node**
   - Node type: **Set**
   - Name it: **Configuration**
   - Create these fields:
     1. `SF_API_BASE_URL` as string  
        Example: `https://apiXX.successfactors.com/odata/v2`
     2. `SF_IDP_URL` as string  
        Example: `https://apiXX.successfactors.com/oauth/idp`
     3. `SF_TOKEN_URL` as string  
        Example: `https://apiXX.successfactors.com/oauth/token`
     4. `company_id` as string  
        Your SuccessFactors tenant/company ID
     5. `client_id` as string  
        API key from SuccessFactors OAuth2 client registration
     6. `user_id` as string  
        The SuccessFactors user to impersonate/authenticate as
     7. `private_key` as string  
        The base64 private key body copied from the exported PEM, with no header/footer and no line breaks
     8. `top` as number  
        Set to `20`
     9. `select` as string  
        Set to `personIdExternal,perPersonUuid`
   - Keep standard Set options unless you need stricter field handling

4. **Connect Manual Trigger → Configuration**

5. **Add an HTTP Request node for SAML assertion**
   - Node type: **HTTP Request**
   - Name it: **Get SAML Assertion**
   - Configure:
     - Method: `POST`
     - URL: `={{ $('Configuration').first().json.SF_IDP_URL }}`
     - Send Body: enabled
     - Content Type: `Form URLencoded`
   - Add body parameters:
     - `client_id` = `={{ $('Configuration').first().json.client_id }}`
     - `user_id` = `={{ $('Configuration').first().json.user_id }}`
     - `token_url` = `={{ $('Configuration').first().json.SF_TOKEN_URL }}`
     - `private_key` = `={{ $('Configuration').first().json.private_key }}`
   - No separate n8n credential is used here; authentication is passed in the form body according to the SuccessFactors flow

6. **Connect Configuration → Get SAML Assertion**

7. **Add an HTTP Request node for bearer token exchange**
   - Node type: **HTTP Request**
   - Name it: **Get Bearer Token**
   - Configure:
     - Method: `POST`
     - URL: `={{ $('Configuration').first().json.SF_TOKEN_URL }}`
     - Send Body: enabled
     - Content Type: `Form URLencoded`
   - Add body parameters:
     - `company_id` = `={{ $('Configuration').first().json.company_id }}`
     - `client_id` = `={{ $('Configuration').first().json.client_id }}`
     - `grant_type` = `urn:ietf:params:oauth:grant-type:saml2-bearer`
     - `assertion` = `={{ $json.data }}`
   - This assumes the previous node returns the SAML assertion in a field named `data`

8. **Connect Get SAML Assertion → Get Bearer Token**

9. **Add an HTTP Request node for SuccessFactors data retrieval**
   - Node type: **HTTP Request**
   - Name it: **Fetch PerPerson from SF**
   - Configure:
     - Method: `GET` or leave default request method if GET is implied by empty body
     - URL: `={{ $('Configuration').first().json.SF_API_BASE_URL }}/PerPerson`
     - Send Query Parameters: enabled
     - Send Headers: enabled
   - Add query parameters:
     - `$format` = `json`
     - `$top` = `={{ $('Configuration').first().json.top }}`
     - `$select` = `={{ $('Configuration').first().json.select }}`
     - `$expand` = `employmentNav`
   - Add header:
     - `Authorization` = `=Bearer {{ $json.access_token }}`
   - This assumes the token response contains `access_token`

10. **Connect Get Bearer Token → Fetch PerPerson from SF**

11. **Add a Code node**
   - Node type: **Code**
   - Name it: **Flatten Results**
   - Language: JavaScript
   - Paste this logic conceptually:
     - Read `d.results` from the HTTP response
     - For each person:
       - Read `employmentNav.results`
       - If no employment exists, emit one item with null employment fields
       - Otherwise emit one item per employment record
   - Use this code:

```javascript
const persons = $input.first().json?.d?.results ?? [];
const output = [];

for (const person of persons) {
  const employments = person.employmentNav?.results ?? [];

  if (employments.length === 0) {
    output.push({
      json: {
        personIdExternal: person.personIdExternal ?? null,
        perPersonUuid:    person.perPersonUuid ?? null,
        empStartDate:     null,
        empEndDate:       null,
        userId:           null,
      }
    });
  } else {
    for (const emp of employments) {
      output.push({
        json: {
          personIdExternal: person.personIdExternal ?? null,
          perPersonUuid:    person.perPersonUuid ?? null,
          empStartDate:     emp.startDate ?? null,
          empEndDate:       emp.endDate ?? null,
          userId:           emp.userId ?? null,
        }
      });
    }
  }
}

return output;
```

12. **Connect Fetch PerPerson from SF → Flatten Results**

13. **Run the workflow manually**
   - Click **Test workflow**
   - Validate outputs in sequence:
     - SAML assertion returned from the IdP call
     - Access token returned from the token call
     - `d.results` returned from `/PerPerson`
     - Flattened normalized records in the final node

14. **Prepare SuccessFactors-side prerequisites**
   - In SuccessFactors Admin, go to **Manage OAuth2 Client Applications**
   - Register a new client application
   - Generate an X.509 certificate
   - Record the generated **API Key** as `client_id`
   - Export the certificate/private key material as indicated in the workflow note
   - Extract the base64 private key body only
   - Ensure the target `user_id` exists and has the required API permissions
   - Prefer a service account for stable access scope

15. **Optional production improvements**
   - Replace Manual Trigger with **Schedule Trigger**
   - Add pagination handling for datasets larger than `$top`
   - Extend `$select` and `$expand` if more entities are needed
   - Add error handling branches or retry logic for transient failures
   - Store secrets in credentials or external secret management rather than directly in the Set node

### Credential configuration note
This workflow does **not** use a built-in n8n OAuth2 credential because the SuccessFactors authentication mechanism here is a specialized SAML bearer flow. The private key and client values are submitted directly in requests. For security, move these values into environment variables or a secret store if deploying beyond simple testing.

### Sub-workflow setup
There are **no sub-workflows** and **no Execute Workflow nodes** in this workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SuccessFactors uses a proprietary SAML 2.0 Bearer Assertion flow, so n8n’s standard OAuth2 credential is not suitable for this pattern. | Workflow design note |
| Basic Authentication is not recommended as a secure authentication mechanism for this use case. | Security note |
| Register the OAuth2 client in SuccessFactors under Manage OAuth2 Client Applications. | SuccessFactors admin setup |
| The `user_id` is mandatory and determines the permission scope of accessible data. Use a service account where possible. | Authentication and authorization behavior |
| The private key must be pasted as base64 body only, without PEM header/footer lines and without line breaks. | Configuration requirement |
| Suggested extensions from the notes: add `personalInfoNav` and `jobInfoNav`, implement full pagination, replace Manual Trigger with Schedule Trigger. | Enhancement ideas |