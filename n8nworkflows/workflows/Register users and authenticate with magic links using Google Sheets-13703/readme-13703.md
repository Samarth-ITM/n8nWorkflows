Register users and authenticate with magic links using Google Sheets

https://n8nworkflows.xyz/workflows/register-users-and-authenticate-with-magic-links-using-google-sheets-13703


# Register users and authenticate with magic links using Google Sheets

This document provides a technical breakdown and reconstruction guide for the **"Register users and authenticate with magic links using Google Sheets"** workflow. This system leverages Google Sheets as a database and session store to provide a complete authentication cycle, including registration, credential delivery, magic link authentication, and session management.

---

### 1. Workflow Overview

This workflow implements a lightweight, serverless authentication portal. It is designed for n8n users who need a private area or member-only portal without a dedicated backend database.

**Logical Blocks:**
*   **1.1 User Registration:** Collects email via an n8n Form, generates a random password and a one-time token, stores them in Google Sheets, and emails the user.
*   **1.2 Manual Login:** A form-based login that validates credentials against Google Sheets and redirects the user to the Magic Link authentication endpoint.
*   **1.3 Magic Link Authentication (/auth):** A webhook endpoint that consumes a one-time token, generates a 64-character session ID (`sid`), stores it in the sheet, and sets a secure `HttpOnly` cookie on the user's browser.
*   **1.4 Protected Profile Page (/profile):** A webhook endpoint that reads the `sid` cookie, validates it against the sheet, and renders a personalized HTML page or redirects to login if the session is invalid.

---

### 2. Block-by-Block Analysis

#### 2.1 Registration Block
This block handles the initial onboarding of new users.
*   **Nodes Involved:** `On form register`, `Create password and token`, `Append or update row in sheet`, `Send username/password`, `Notice register success`.
*   **Node Details:**
    *   **On form register (Form Trigger):** Configured at path `/register`. Collects a required "Email" field.
    *   **Create password and token (Code):** Uses JavaScript to generate a 12-character alphanumeric password and a 32-character random token. Outputs `email`, `password`, and `token`.
    *   **Append or update row (Google Sheets):** Connects to a specific Spreadsheet ID. Uses `username` as the key to prevent duplicate entries.
    *   **Send username/password (Gmail):** Sends an HTML email containing the generated credentials and a magic link: `https://YOUR_DOMAIN/webhook/auth?token={{token}}`.

#### 2.2 Login Block
Handles users who prefer to log in manually via username and password.
*   **Nodes Involved:** `On form login`, `Check username/password`, `If login success`, `If has token in DB`, `Redirect to auth url`, `Generate token`, `Add token to DB`.
*   **Node Details:**
    *   **Check username/password (Google Sheets):** Filters the sheet using two conditions: `username` AND `password`.
    *   **Logic Gate:** If successful, it checks if the user already has a pending `token`. If so, it redirects to `/auth`. If not, it generates a new token first.
    *   **Redirect nodes:** These are Form nodes set to "Completion" mode with the "Redirect" response type.

#### 2.3 Authentication Block (The Magic Link Logic)
This is the core security mechanism that converts a temporary token into a persistent session.
*   **Nodes Involved:** `Auth`, `If has token`, `Search by token`, `If token valid`, `Prepare sid`, `Update sid`, `Redirect to profile and set cookies`.
*   **Node Details:**
    *   **Auth (Webhook):** Listens for GET requests on the `/auth` path.
    *   **Prepare sid (Code):** Generates a high-entropy 64-character session ID using `Date.now()` and `Math.random()`.
    *   **Update sid (Google Sheets):** Updates the row matching the token's user. **Crucial:** It clears the `token` field (making it one-time use) and sets the `sid` field.
    *   **Redirect to profile (Respond to Webhook):** Sends a 302 redirect to `/webhook/profile`. Includes a `Set-Cookie` header: `sid={{sid}}; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=604800`.

#### 2.4 Profile Block
The protected resource that verifies the user's session.
*   **Nodes Involved:** `Profile`, `Check sid cookies header`, `Search sid`, `If has data with sid`, `Prepare html`, `Show profile page`.
*   **Node Details:**
    *   **Check sid cookies header (Code):** A utility script that parses the `headers.cookie` string to extract the `sid` value.
    *   **Search sid (Google Sheets):** Queries the sheet for a row where the `sid` column matches the cookie.
    *   **Show profile page (Respond to Webhook):** Returns a text/html response. If the session is valid, it greets the user; if not, it redirects them back to `/form/login`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On form register | Form Trigger | Input Entry | - | Create password and token | ## 0. Register: Collect email, generate credentials... |
| Create password and token | Code | Logic/Gen | On form register | Append or update row... | ## 0. Register: Collect email, generate credentials... |
| Append or update row... | Google Sheets | Data Store | Create password... | Send username/password | ## 0. Register: Collect email, generate credentials... |
| Send username/password | Gmail | Notification | Append or update... | Notice register success | ## 0. Register: Collect email, generate credentials... |
| On form login | Form Trigger | Input Entry | - | Check username/password | ## 1. Login: Validate username/password... |
| Check username/password | Google Sheets | Validation | On form login | If login success | ## 1. Login: Validate username/password... |
| Auth | Webhook | Auth Entry | - | If has token | ## 2. Auth: Validate token, generate sid... |
| Search by token | Google Sheets | Validation | If has token | If token valid | ## 2. Auth: Validate token, generate sid... |
| Prepare sid | Code | Logic/Gen | If token valid | Update sid, Redirect to profile... | ## 2. Auth: Validate token, generate sid... |
| Update sid | Google Sheets | Data Update | Prepare sid | - | ## 2. Auth: Validate token, generate sid... |
| Redirect to profile... | Respond to Webhook | Session Set | Prepare sid | - | ## 2. Auth: Validate token, generate sid... |
| Profile | Webhook | Protected Page | - | Check sid cookies header | ## 3. Profile: Read sid cookie, render page... |
| Check sid cookies header| Code | Cookie Parser | Profile | Search sid | ## 3. Profile: Read sid cookie, render page... |
| Search sid | Google Sheets | Session Check | Check sid cookies...| If has data with sid | ## 3. Profile: Read sid cookie, render page... |
| Show profile page | Respond to Webhook | Content Delivery| Prepare html | - | ## 3. Profile: Read sid cookie, render page... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Database Setup (Google Sheets)
1. Create a Google Sheet with the following headers in the first row: `username`, `password`, `role`, `token`, `sid`.
2. Rename the tab to `Sheet2` (or adjust the node configurations accordingly).

#### Step 2: Registration Flow
1. Create a **Form Trigger** node. Path: `register`. Field: "Email" (Required).
2. Add a **Code** node (Run once per item). Use JS to generate a random password and a 32-char token.
3. Add a **Google Sheets** node. Operation: `Append or Update`. Mapping: `username` (from Email).
4. Add a **Gmail** node. Set the body to include the login link: `https://<YOUR_INSTANCE>/webhook/auth?token={{$json.token}}`.

#### Step 3: Login & Magic Link Generation
1. Create a **Form Trigger** node. Path: `login`. Fields: `username` and `password` (type: password).
2. Add a **Google Sheets** node. Operation: `Get Many`. Filter: `username == {{form.username}}` AND `password == {{form.password}}`.
3. Add an **If** node to check if a row was found.
4. If found, use another **If** node to check if the `token` column is empty.
5. If empty, generate a token via **Code** and update the sheet. Redirect the user to `/webhook/auth?token=...`.

#### Step 4: Authentication & Cookie Handling
1. Create a **Webhook** node. Path: `auth`. Method: `GET`. Response Mode: `Response to Webhook Node`.
2. Add a **Google Sheets** node to search for the user by the `token` query parameter.
3. Add a **Code** node to generate a 64-char `sid`.
4. Add a **Google Sheets** node. Operation: `Update`. Match by `username`. Set `sid = new_sid` and `token = ""` (empty string to invalidate it).
5. Add a **Respond to Webhook** node. Status: `302`.
   - Header `Location`: `/webhook/profile`
   - Header `Set-Cookie`: `sid={{$json.sid}}; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=604800`

#### Step 5: Profile Verification
1. Create a **Webhook** node. Path: `profile`.
2. Add a **Code** node to parse `sid` from `headers.cookie`.
3. Add a **Google Sheets** node to search for a user matching that `sid`.
4. Add an **If** node. If valid, use **Respond to Webhook** to show HTML. If invalid, **Respond to Webhook** with a redirect to `/form/login`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Security Note** | Ensure your n8n instance uses **HTTPS**. The `Secure` flag on the cookie will prevent it from being sent over unencrypted connections. |
| **Cookie Persistence** | The `Max-Age=604800` parameter sets the session duration to 7 days. |
| **Logic Consistency** | Always replace `YOUR_DOMAIN` and `YOUR_SPREADSHEET_ID` with your actual values before testing. |
| **Magic Link Expiry** | By default, this workflow makes tokens "one-time use" by clearing them upon `/auth` access, but doesn't implement a time-based expiry (e.g., 15 mins). |