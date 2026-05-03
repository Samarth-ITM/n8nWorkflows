Create a Complete User Authentication System with PostgreSQL & Webhooks

https://n8nworkflows.xyz/workflows/create-a-complete-user-authentication-system-with-postgresql---webhooks-10040


# Create a Complete User Authentication System with PostgreSQL & Webhooks

### 1. Workflow Overview

This workflow implements a **Complete User Authentication System** integrating PostgreSQL as the backend database and n8n webhooks as the API entry point. It supports three core functionalities:

- **User Signup:** Registers new users with bcrypt-hashed passwords.
- **User Login:** Authenticates users by verifying credentials in a case-insensitive manner.
- **Forgot Password:** Resets and returns a new random password for users who forgot theirs.

The workflow logically divides into these blocks:

- **1.1 Input Reception:** A webhook node listens for incoming POST requests containing authentication actions.
- **1.2 Request Routing:** A switch node routes the request based on the `path` property (`signup`, `signin`, or `forgot`).
- **1.3 Signup Processing:** Inserts a new user into the PostgreSQL `users` table with hashed password.
- **1.4 Login Verification:** Queries the database to validate credentials and uses an IF node to confirm success.
- **1.5 Forgot Password Handling:** Updates the user's password with a new random string.
- **1.6 Response Generation:** Sends back a structured JSON response indicating success or failure for each operation.
- **1.7 Documentation and Setup Notes:** Sticky notes provide detailed explanations, usage instructions, and prerequisites for the database setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives all incoming authentication requests via a POST webhook at `/webhook/auth`.
- **Nodes Involved:**  
  - `Webhook`  
  - `Webhook Doc` (Sticky Note)  
  - `Overview` (Sticky Note)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point for all auth requests.  
    - Config: HTTP POST method, path set to `auth`, response mode configured to use a response node downstream.  
    - Inputs: HTTP POST request JSON payload with keys: `path`, `email`, `password`, `name`.  
    - Outputs: JSON data forwarded to Router node.  
    - Failure Modes: Invalid HTTP method, missing fields in payload, webhook misconfiguration.  
    - Version: v2.1

  - **Webhook Doc**  
    - Type: Sticky Note  
    - Role: Documents webhook usage, expected JSON input format, and routing logic based on the `path` field.

  - **Overview**  
    - Type: Sticky Note  
    - Role: High-level description of workflow purpose, prerequisites, and usage example.

#### 2.2 Request Routing

- **Overview:** Routes incoming requests to corresponding processing blocks based on the `path`: `signup`, `signin`, or `forgot`.
- **Nodes Involved:**  
  - `Router` (Switch node)  
  - `Switch Doc` (Sticky Note)

- **Node Details:**

  - **Router**  
    - Type: Switch  
    - Role: Checks the `path` field in the JSON payload (case-sensitive string).  
    - Config: Matches exact strings: `signup`, `signin`, `forgot`.  
    - Inputs: JSON from Webhook node.  
    - Outputs:  
      - Output 1 → `signup` flow  
      - Output 2 → `signin` flow  
      - Output 3 → `forgot` flow  
      - Output 0 (default) → no action  
    - Failure Modes: Missing or invalid `path` value, case-sensitivity issues.  
    - Version: v1

  - **Switch Doc**  
    - Type: Sticky Note  
    - Role: Explains routing logic and default behavior for invalid paths.

#### 2.3 Signup Processing

- **Overview:** Inserts a new user record into PostgreSQL with bcrypt-hashed password.
- **Nodes Involved:**  
  - `Signup` (PostgreSQL node)  
  - `Signup Doc` (Sticky Note)

- **Node Details:**

  - **Signup**  
    - Type: PostgreSQL  
    - Role: Executes an SQL INSERT query to add a new user with hashed password.  
    - Config:  
      - Query uses `crypt` with `gen_salt('bf')` (bcrypt) to hash password.  
      - Inserts `full_name`, `email`, `password_hash`.  
      - Returns `id`, `full_name`, `email`, `created_at`.  
    - Inputs: JSON containing `name`, `email`, `password`.  
    - Outputs: Inserted user data on success.  
    - Credential: Uses PostgreSQL credential named "Project: Edubot".  
    - Failure Modes: Email uniqueness violation, DB connection issues, SQL errors.  
    - Version: v2.6

  - **Signup Doc**  
    - Type: Sticky Note  
    - Role: Documents SQL query, purpose, output, and notes on password hashing and email uniqueness.

#### 2.4 Login Verification

- **Overview:** Validates user credentials by comparing input password with stored bcrypt hash, case-insensitive email matching.
- **Nodes Involved:**  
  - `Login` (PostgreSQL node)  
  - `Check Login` (IF node)  
  - `Login Doc` (Sticky Note)  
  - `IF Doc` (Sticky Note)

- **Node Details:**

  - **Login**  
    - Type: PostgreSQL  
    - Role: Runs SQL SELECT to fetch user fields and verify password with `crypt`.  
    - Config:  
      - `WHERE LOWER(email) = LOWER(input_email)` for case-insensitive email match.  
      - Returns `id`, `full_name`, `email`, and a boolean `isPasswordMatch`.  
    - Inputs: JSON with `email`, `password`.  
    - Outputs: User data with password match boolean.  
    - Credential: "Project: Edubot" PostgreSQL.  
    - Failure Modes: No user found, password mismatch, DB errors.  
    - Version: v2.6

  - **Check Login**  
    - Type: IF  
    - Role: Checks if `id` exists and `isPasswordMatch` is `true`.  
    - Config: Boolean condition that evaluates to true only if both criteria are met.  
    - Inputs: Output of Login node.  
    - Outputs:  
      - True path → successful login response.  
      - False path → error response (invalid credentials).  
    - Failure Modes: Expression errors, missing fields.  
    - Version: v2.2

  - **Login Doc**  
    - Type: Sticky Note  
    - Role: Explains SQL query, case-insensitive email check, and output fields.

  - **IF Doc**  
    - Type: Sticky Note  
    - Role: Explains validation logic and conditional response routing.

#### 2.5 Forgot Password Handling

- **Overview:** Resets user password to a new random 8-character string and returns it.
- **Nodes Involved:**  
  - `Reset Password` (PostgreSQL node)  
  - `Forgot Password Doc` (Sticky Note)

- **Node Details:**

  - **Reset Password**  
    - Type: PostgreSQL  
    - Role: Performs an UPDATE on the `users` table to set a new bcrypt-hashed password generated randomly.  
    - Config: Uses a CTE (`WITH new_pass AS (...)`) to generate a random 8-character password using `substring(md5(random()::text) from 1 for 8)`.  
    - Updates password hash with bcrypt and returns email and new plain password.  
    - Inputs: JSON with `email`.  
    - Outputs: Email and new password fields.  
    - Credential: "Project: Edubot" PostgreSQL.  
    - Failure Modes: No user found with that email, DB errors.  
    - Version: v2.6

  - **Forgot Password Doc**  
    - Type: Sticky Note  
    - Role: Explains SQL logic, case-insensitive email lookup, and output structure.

#### 2.6 Response Generation

- **Overview:** Sends back a uniform JSON response indicating success or failure and relevant data.
- **Nodes Involved:**  
  - `Respond` (Respond to Webhook node)  
  - `Response Doc` (Sticky Note)

- **Node Details:**

  - **Respond**  
    - Type: Respond to Webhook  
    - Role: Returns JSON to the client based on upstream node output.  
    - Config: Default settings, linked to receive output from all main process nodes.  
    - Inputs: Success or error data from Signup, Login (true or false), or Reset Password nodes.  
    - Outputs: HTTP JSON response with fields:  
      - `status`: "success" or "error"  
      - `message`: descriptive string  
      - `data`: object with query results or empty.  
    - Failure Modes: Response generation failure, invalid JSON format.  
    - Version: v1.4

  - **Response Doc**  
    - Type: Sticky Note  
    - Role: Describes response JSON format and usage notes.

#### 2.7 Database Setup & Documentation

- **Overview:** Provides instructions for setting up PostgreSQL/Supabase database schema and extensions.
- **Nodes Involved:**  
  - `Database Setup` (Sticky Note)

- **Node Details:**

  - **Database Setup**  
    - Type: Sticky Note  
    - Role: Detailed step-by-step for creating Supabase project, enabling required extensions (`uuid-ossp`, `pgcrypto`), creating `users` table with unique email, and adding credentials to n8n.  
    - Contains example SQL for table creation.

---

### 3. Summary Table

| Node Name       | Node Type               | Functional Role                            | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                                         |
|-----------------|-------------------------|-------------------------------------------|---------------------|------------------------|---------------------------------------------------------------------------------------------------------------------|
| Overview        | Sticky Note             | Workflow purpose, prereqs, usage example  | —                   | —                      | Explains purpose, features, usage, prerequisites, example input/output JSON                                         |
| Webhook Doc     | Sticky Note             | Documents webhook input and routing       | —                   | —                      | Explains webhook POST endpoint, input JSON structure, routing logic                                                |
| Webhook         | Webhook                 | Receives auth requests via POST            | —                   | Router                 |                                                                                                                     |
| Switch Doc      | Sticky Note             | Documents Router node routing logic        | —                   | —                      | Describes routing rules based on `path` field                                                                      |
| Router          | Switch                  | Routes request to signup/login/forgot     | Webhook             | Signup, Login, Reset Password |                                                                                                                     |
| Signup Doc      | Sticky Note             | Documents Signup SQL and logic             | —                   | —                      | Details INSERT SQL, bcrypt usage, output, unique email requirement                                                 |
| Signup          | PostgreSQL              | Inserts new user with hashed password      | Router (signup output) | Respond                 |                                                                                                                     |
| Login Doc       | Sticky Note             | Documents Login SQL and logic              | —                   | —                      | Explains case-insensitive email check and password verification                                                   |
| Login           | PostgreSQL              | Validates user credentials                  | Router (signin output) | Check Login             |                                                                                                                     |
| IF Doc          | Sticky Note             | Explains login validation condition        | —                   | —                      | Details IF node condition on `id` and `isPasswordMatch`                                                            |
| Check Login     | IF                      | Determines login success/failure            | Login                | Respond (true & false)  |                                                                                                                     |
| Forgot Password Doc | Sticky Note          | Documents password reset SQL and logic     | —                   | —                      | Explains random password generation and update query                                                               |
| Reset Password  | PostgreSQL              | Resets password to random string            | Router (forgot output) | Respond                 |                                                                                                                     |
| Response Doc    | Sticky Note             | Documents response JSON format              | —                   | —                      | Describes success/error response structure                                                                          |
| Respond         | Respond to Webhook      | Sends JSON response to client               | Signup, Check Login, Reset Password | —                      |                                                                                                                     |
| Database Setup  | Sticky Note             | Guides PostgreSQL/Supabase setup            | —                   | —                      | Setup instructions for database, extensions, table creation, and credentials                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Webhook node:**
   - Name: `Webhook`
   - HTTP Method: POST
   - Path: `auth`
   - Response Mode: `Response Node`
   - This will be the entry point for all auth requests.

3. **Add a Switch node:**
   - Name: `Router`
   - Input: connect from `Webhook` node.
   - Property to check: `path` from the incoming JSON.
   - Rules (string matching):  
     - `signup` → Output 1  
     - `signin` → Output 2  
     - `forgot` → Output 3  
   - Default output: 0 (no action)

4. **Create Signup block:**
   - Add a PostgreSQL node named `Signup`.
   - Credentials: Set PostgreSQL credentials with access to your database.
   - Query:
     ```sql
     INSERT INTO users (full_name, email, password_hash)
     VALUES (
       '{{$json["name"]}}',
       '{{$json["email"]}}',
       crypt('{{$json["password"]}}', gen_salt('bf'))
     )
     RETURNING id, full_name, email, created_at;
     ```
   - Connect Router output 1 to Signup input.

5. **Create Login block:**
   - Add a PostgreSQL node named `Login`.
   - Credentials: same PostgreSQL credentials.
   - Query:
     ```sql
     SELECT
       id,
       full_name,
       email,
       (password_hash = crypt('{{$json["password"]}}', password_hash)) AS "isPasswordMatch"
     FROM users
     WHERE LOWER(email) = LOWER('{{$json["email"]}}');
     ```
   - Connect Router output 2 to Login input.

   - Add an IF node named `Check Login`.
   - Connect Login output to Check Login input.
   - Condition: Boolean expression:  
     `{{$json["id"] !== undefined && $json["isPasswordMatch"] === true}}` equals `true`.
   - True output → successful login path.
   - False output → invalid credentials path.

6. **Create Forgot Password block:**
   - Add a PostgreSQL node named `Reset Password`.
   - Credentials: same PostgreSQL credentials.
   - Query:
     ```sql
     WITH new_pass AS (
       SELECT substring(md5(random()::text) from 1 for 8) AS plain_password
     )
     UPDATE users
     SET password_hash = crypt(new_pass.plain_password, gen_salt('bf'))
     FROM new_pass
     WHERE LOWER(email) = LOWER('{{$json["email"]}}')
     RETURNING email, new_pass.plain_password AS newPassword;
     ```
   - Connect Router output 3 to Reset Password input.

7. **Add a Respond to Webhook node named `Respond`:**
   - Connect outputs from:  
     - Signup node  
     - Check Login node (both true and false outputs)  
     - Reset Password node  
   - Configure to return JSON responses in the format:
     ```json
     {
       "status": "success" | "error",
       "message": "string",
       "data": { ... }
     }
     ```
   - Implement logic in the workflow or via expressions to set `status` and `message` accordingly:  
     - Signup success → success status with user data  
     - Login success → success status with user data  
     - Login failure → error status with message "Invalid credentials"  
     - Forgot password success → success status with new password  
     - Any errors → error status with error message

8. **Setup PostgreSQL database:**
   - Create a Supabase project or PostgreSQL database.
   - Enable extensions:
     ```sql
     CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
     CREATE EXTENSION IF NOT EXISTS "pgcrypto";
     ```
   - Create `users` table:
     ```sql
     CREATE TABLE users (
       id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
       full_name text NOT NULL,
       email text UNIQUE NOT NULL,
       password_hash text NOT NULL,
       created_at timestamptz DEFAULT now()
     );
     ```
   - Add PostgreSQL credentials in n8n with appropriate permissions.

9. **Test the workflow:**
   - Send POST requests to `/webhook/auth` with JSON payloads like:
     ```json
     {
       "path": "signup",
       "email": "user@example.com",
       "password": "pass123",
       "name": "John Doe"
     }
     ```
   - Test with other paths `signin` and `forgot`.
   - Confirm expected JSON responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| PostgreSQL must have `uuid-ossp` and `pgcrypto` extensions enabled for UUID generation and bcrypt password hashing.                                           | Database setup instructions in sticky note.        |
| Supabase is recommended for easy PostgreSQL hosting and management.                                                                                           | https://supabase.com                                |
| The workflow handles all auth actions via a single webhook endpoint `/webhook/auth` using the `path` field for routing.                                     | Documentation in `Webhook Doc` sticky note.         |
| Passwords are hashed securely using bcrypt (`gen_salt('bf')`), and login checks password correctness via PostgreSQL's `crypt` function.                      | Security best practices documented in Signup/Login Docs. |
| Email uniqueness is enforced by a unique constraint in the `users` table; signup will fail with duplicate emails.                                             | Database schema notes.                              |
| The forgot password flow returns the new password in the response JSON; additional logic can be added to email the new password to the user securely.         | Suggested extension for production systems.         |
| Case-insensitive email lookup prevents user errors related to email casing during login and password reset.                                                  | Documented in Login and Forgot Password Docs.       |
| The response node returns a consistent JSON structure with status, message, and data fields for all flows.                                                    | Response formatting notes.                          |

---

This completes the detailed reference documentation of the "Authentication Panel (Signup/Login/Forgot Password)" n8n workflow, enabling understanding, reproduction, and adaptation for user authentication systems leveraging PostgreSQL and webhooks.