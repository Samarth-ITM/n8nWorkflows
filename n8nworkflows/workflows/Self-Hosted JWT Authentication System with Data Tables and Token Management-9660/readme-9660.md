Self-Hosted JWT Authentication System with Data Tables and Token Management

https://n8nworkflows.xyz/workflows/self-hosted-jwt-authentication-system-with-data-tables-and-token-management-9660


# Self-Hosted JWT Authentication System with Data Tables and Token Management

---

## 1. Workflow Overview

This workflow implements a **Self-Hosted JWT Authentication System** with user registration, login, JWT access token verification, refresh token management, and secure password handling. It is designed to serve as a standalone authentication backend for applications requiring token-based user authentication.

The workflow logically divides into four main blocks:

- **1.1 Registration Flow:** Handles new user sign-up, including input validation, uniqueness checks for username/email, password salting and hashing, and storing user records.
- **1.2 Login Flow:** Processes user login requests by verifying credentials, generating JWT access and refresh tokens, storing refresh tokens securely, and returning tokens to the client.
- **1.3 Verify Token Flow:** Accepts an access token, parses and validates it, verifies its signature and expiry, and returns token validity status.
- **1.4 Refresh Token Flow:** Accepts a refresh token, verifies its signature and expiry, checks the token's presence in the database, and issues a new access token if valid.

Additionally, the workflow uses **n8n Data Tables** to store user information and refresh tokens, and employs cryptographic operations such as SHA-512 password hashing with salts and HMAC-SHA256 signatures for JWT tokens.

---

## 2. Block-by-Block Analysis

### 2.1 Registration Flow

**Overview:**  
Manages new user registrations by validating inputs, checking for username and email uniqueness, generating a salt, hashing the password with the salt, and storing user data securely. Ensures uniqueness before hashing to optimize resource use.

**Nodes Involved:**  
- Registration Webhook  
- Parse Register Request  
- Validate Registration Request  
- Get User By Username  
- If Username Is Available  
- Get User By Email  
- If Email Is Available  
- Generate Salt  
- Format Password & Salt  
- Hash Password  
- Format User Data  
- Create User  
- Registration Successful  
- Error Registration Response  
- Username Taken Error  
- Email Taken Error  
- Registration Request Invalid Error  
- Sticky Notes (Registration Flow, Password Security, Table Setup, Critical Security, Troubleshooting, Test Sequence, Customization Points)

**Node Details:**

- **Registration Webhook**  
  - Type: Webhook (HTTP POST endpoint)  
  - Role: Entry point for user registration requests (`/register-user`)  
  - Receives JSON body with `email`, `username`, `password`  
  - Outputs to parse register request node  
  - Failure: Missing or malformed request body  

- **Parse Register Request**  
  - Type: Set  
  - Extracts `email`, `username`, `password` from request body  
  - Outputs properties for validation  
  - Failure: None expected here  

- **Validate Registration Request**  
  - Type: Code  
  - Checks presence of `email`, `username`, `password`  
  - Validates password length (minimum 8 characters)  
  - Trims and normalizes email and username  
  - Throws error if invalid, caught later for error response  
  - Failure: Missing fields, short password  

- **Get User By Username**  
  - Type: DataTable (get)  
  - Queries `users` table by username  
  - Failure: Database errors, no user found returns empty  

- **If Username Is Available**  
  - Type: If  
  - Checks if username query result is empty (available) or not (taken)  
  - Routes accordingly to error or next step  

- **Get User By Email**  
  - Type: DataTable (get)  
  - Queries `users` table by email  
  - Failure: Database errors, no user found returns empty  

- **If Email Is Available**  
  - Type: If  
  - Checks if email query result is empty (available) or not (taken)  
  - Routes accordingly to error or next step  

- **Generate Salt**  
  - Type: Crypto (generate)  
  - Generates a random salt in hexadecimal format  
  - Used to salt the password before hashing  

- **Format Password & Salt**  
  - Type: Code  
  - Concatenates plaintext password and generated salt into `passwordWithSalt` for hashing  

- **Hash Password**  
  - Type: Crypto (SHA512 hash)  
  - Hashes the concatenated password+salt string  
  - Outputs the hashed password  

- **Format User Data**  
  - Type: Code  
  - Formats user data object for storage, combining salt and hash as `"salt:hash"` string for `password_hash`  
  - Includes email and username  

- **Create User**  
  - Type: DataTable (create)  
  - Inserts new user row into `users` table  
  - Failure: Duplicate keys, DB connection issues  

- **Registration Successful**  
  - Type: RespondToWebhook  
  - Returns HTTP 200 success with newly created user info  

- **Error Registration Response**  
  - Type: RespondToWebhook  
  - Returns HTTP 500 for internal errors  

- **Username Taken Error**  
  - Type: RespondToWebhook  
  - Returns HTTP 400 with message about username already taken  

- **Email Taken Error**  
  - Type: RespondToWebhook  
  - Returns HTTP 400 with message about email already registered  

- **Registration Request Invalid Error**  
  - Type: RespondToWebhook  
  - Returns HTTP 400 with validation error messages  

- **Sticky Notes**  
  - Provide detailed explanations about registration flow, password security (use of salt and SHA-512), data table schema, security considerations, troubleshooting tips, and testing instructions.  

---

### 2.2 Login Flow

**Overview:**  
Handles login requests by validating inputs, retrieving user record by email, verifying password hash using stored salt, generating JWT access and refresh tokens, storing refresh tokens securely, and responding with tokens.

**Nodes Involved:**  
- Login Webhook  
- Process login webhook (Set)  
- SET ACCESS AND REFRESH SECRET (Set)  
- Verify Input (Code)  
- Get User (DataTable get)  
- If User Exists (If)  
- Extract Salt & Hash (Code)  
- Hash Input Password (Crypto SHA512)  
- Verify Password (Code)  
- Create JWT Payload (Code)  
- Sign Access Token (Crypto HMAC SHA256)  
- Sign Refresh Token (Crypto HMAC SHA256)  
- Merge JWT Tokens (Merge)  
- Format JWT Tokens (Code)  
- Hash Refresh Token for Storage (Crypto SHA256)  
- Code in JavaScript (Code)  
- Update User Refresh Token (DataTable update)  
- Store Refresh Token (DataTable upsert)  
- Merge (Merge chooseBranch)  
- Login Successful (RespondToWebhook)  
- Login Credentials Invalid Response (RespondToWebhook)  
- User Not Found (RespondToWebhook)  
- Bad Request (RespondToWebhook)  
- Sticky Notes (Login Flow, Two Token System, Token Verification, etc.)

**Node Details:**

- **Login Webhook**  
  - Type: Webhook (HTTP POST `/login`)  
  - Receives JSON with `email` and `password`  

- **Process login webhook**  
  - Type: Set  
  - Extracts `email` and `password` from request body  

- **SET ACCESS AND REFRESH SECRET**  
  - Type: Set  
  - Stores access and refresh secret keys (empty by default, user must configure)  
  - Used in token signing and verification  

- **Verify Input**  
  - Type: Code  
  - Validates presence of email and password, normalizes email  

- **Get User**  
  - Type: DataTable (get)  
  - Queries `users` table by email  

- **If User Exists**  
  - Type: If  
  - Checks if user record was found  

- **Extract Salt & Hash**  
  - Type: Code  
  - Splits stored `password_hash` into salt and hash  
  - Combines input password with salt for hashing  

- **Hash Input Password**  
  - Type: Crypto (SHA512)  
  - Hashes input password concatenated with salt  

- **Verify Password**  
  - Type: Code  
  - Compares input hash with stored hash  
  - Throws error if mismatch  

- **Create JWT Payload**  
  - Type: Code  
  - Creates JWT payloads for access (15 min expiry) and refresh token (7 days expiry)  
  - Encodes header and payload to base64url  
  - Prepares signature input strings  

- **Sign Access Token & Sign Refresh Token**  
  - Type: Crypto (HMAC SHA256)  
  - Signs payloads with respective secret keys to produce token signatures  

- **Merge JWT Tokens**  
  - Type: Merge (combine)  
  - Combines signed access and refresh token data  

- **Format JWT Tokens**  
  - Type: Code  
  - Constructs full JWT tokens by combining header, payload, and base64url-encoded signature  

- **Hash Refresh Token for Storage**  
  - Type: Crypto (SHA256)  
  - Hashes refresh token before storing for security  

- **Code in JavaScript**  
  - Prepares refresh token DB entry including user id, token hash, expiry timestamps, and tokens  

- **Update User Refresh Token**  
  - Type: DataTable (update)  
  - Updates the user record with latest refresh token (optional)  

- **Store Refresh Token**  
  - Type: DataTable (upsert)  
  - Inserts or updates refresh token record in `refresh_tokens` table  

- **Merge**  
  - Type: Merge (chooseBranch)  
  - Combines token storage and user update results  

- **Login Successful**  
  - Type: RespondToWebhook  
  - Returns HTTP 200 with access token, refresh token, and user info  

- **Login Credentials Invalid Response**  
  - Type: RespondToWebhook  
  - Returns HTTP 400 when credentials do not match  

- **User Not Found**  
  - Type: RespondToWebhook  
  - Returns HTTP 401 when user email not found  

- **Bad Request**  
  - Type: RespondToWebhook  
  - Returns HTTP 400 for input validation errors  

- **Sticky Notes**  
  - Explain the login flow, two-token system advantages, token verification process, and password security details.  

---

### 2.3 Verify Token Flow

**Overview:**  
Accepts an access token, parses it, verifies expiration and token type, recreates the HMAC signature to check integrity, and returns token validity status.

**Nodes Involved:**  
- Verify Access Token (Webhook)  
- Process Verify Token Webhook (Set)  
- Parse JWT (Code)  
- SET ACCESS AND REFRESH SECRET2 (Set)  
- Verify HMAC Signature (Crypto HMAC SHA256)  
- Compare Signatures (Code)  
- Access Token Valid (RespondToWebhook)  
- Access Token Invalid (RespondToWebhook)  
- Sticky Notes (Token Verification)

**Node Details:**

- **Verify Access Token**  
  - Type: Webhook (HTTP POST `/verify-token`)  
  - Receives JSON with `access_token`  

- **Process Verify Token Webhook**  
  - Type: Set  
  - Extracts access token from request body  

- **Parse JWT**  
  - Type: Code  
  - Splits JWT into header, payload, and signature  
  - Decodes payload, checks expiration and token type (must not be refresh)  
  - Throws error if invalid format or expired  

- **SET ACCESS AND REFRESH SECRET2**  
  - Type: Set  
  - Loads secret keys for signature verification  

- **Verify HMAC Signature**  
  - Type: Crypto (HMAC SHA256)  
  - Recreates expected signature using secret key and signature input  

- **Compare Signatures**  
  - Type: Code  
  - Converts provided signature from base64url to base64 and compares with expected signature  
  - Returns valid true or throws error for mismatch  

- **Access Token Valid**  
  - Type: RespondToWebhook  
  - Returns success JSON if token is valid  

- **Access Token Invalid**  
  - Type: RespondToWebhook  
  - Returns HTTP 403 Access Denied if token invalid  

- **Sticky Notes**  
  - Provide step-by-step token verification explanation and troubleshooting tips  

---

### 2.4 Refresh Token Flow

**Overview:**  
Handles requests to refresh access tokens using refresh tokens. It verifies refresh token format, signature, and expiration, checks token presence in database after hashing, and issues a new access token if valid.

**Nodes Involved:**  
- Refresh Access Token (Webhook)  
- Process Refresh Token (Set)  
- SET ACCESS AND REFRESH SECRET1 (Set)  
- Parse Refresh Token (Code)  
- Verify Signature (Crypto HMAC SHA256)  
- Compare Refresh Token Signature (Code)  
- Hash Refresh Token For DB Lookup (Crypto SHA256)  
- If Refresh Token Exists (DataTable get)  
- If Refresh Token Is Valid (If)  
- Create Access Token Payload (Code)  
- Sign New Access Token (Crypto HMAC SHA256)  
- Format Access Token JWT (Code)  
- Return New Access Token (RespondToWebhook)  
- Session Expired (RespondToWebhook)  
- Sticky Notes (Refresh Token Flow, Defense in Depth)

**Node Details:**

- **Refresh Access Token**  
  - Type: Webhook (HTTP POST `/refresh`)  
  - Receives JSON with `refresh_token`  

- **Process Refresh Token**  
  - Type: Set  
  - Extracts refresh token from request body  

- **SET ACCESS AND REFRESH SECRET1**  
  - Type: Set  
  - Loads secret keys for verification and signing  

- **Parse Refresh Token**  
  - Type: Code  
  - Splits refresh token JWT  
  - Decodes payload and checks expiration  
  - Throws error if expired  

- **Verify Signature**  
  - Type: Crypto (HMAC SHA256)  
  - Recreates expected signature for refresh token  

- **Compare Refresh Token Signature**  
  - Type: Code  
  - Compares provided and expected signatures (base64url/base64 conversion)  

- **Hash Refresh Token For DB Lookup**  
  - Type: Crypto (SHA256)  
  - Hashes full refresh token for secure database lookup  

- **If Refresh Token Exists**  
  - Type: DataTable (get)  
  - Checks if hashed token exists in `refresh_tokens` table for given user_id  

- **If Refresh Token Is Valid**  
  - Type: If  
  - Branches on token existence and validity  

- **Create Access Token Payload**  
  - Type: Code  
  - Creates new access token payload with 15-minute expiry  

- **Sign New Access Token**  
  - Type: Crypto (HMAC SHA256)  
  - Signs new access token payload  

- **Format Access Token JWT**  
  - Type: Code  
  - Constructs new JWT access token string  

- **Return New Access Token**  
  - Type: RespondToWebhook  
  - Returns new access token in JSON with success status  

- **Session Expired**  
  - Type: RespondToWebhook  
  - Returns HTTP 403 with session expired message if invalid  

- **Sticky Notes**  
  - Explain refresh token lifecycle, database lookup for revocation, defense in depth strategy (hashed tokens in DB), and troubleshooting  

---

## 3. Summary Table

| Node Name                     | Node Type               | Functional Role                      | Input Node(s)                             | Output Node(s)                                  | Sticky Note                                                                                           |
|-------------------------------|-------------------------|------------------------------------|------------------------------------------|------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Registration Webhook          | Webhook                 | Entry point for new user registration | -                                        | Parse Register Request                          | ## Registration Flow                                                                                |
| Parse Register Request        | Set                     | Extracts registration fields       | Registration Webhook                     | Validate Registration Request                   |                                                                                                     |
| Validate Registration Request | Code                    | Validates registration input       | Parse Register Request                   | Get User By Username, Registration Request Invalid Error |                                                                                                     |
| Get User By Username          | DataTable (get)          | Checks username uniqueness          | Validate Registration Request            | If Username Is Available                         |                                                                                                     |
| If Username Is Available      | If                      | Branches on username availability  | Get User By Username                     | Get User By Email, Username Taken Error         |                                                                                                     |
| Get User By Email             | DataTable (get)          | Checks email uniqueness             | If Username Is Available                 | If Email Is Available                            |                                                                                                     |
| If Email Is Available         | If                      | Branches on email availability     | Get User By Email                        | Generate Salt, Email Taken Error                 |                                                                                                     |
| Generate Salt                | Crypto (generate)         | Creates random salt for password    | If Email Is Available                    | Format Password & Salt                           | ## 🔐 PASSWORD SECURITY                                                                             |
| Format Password & Salt       | Code                     | Concatenates password + salt        | Generate Salt                           | Hash Password                                   |                                                                                                     |
| Hash Password                | Crypto (SHA512)           | Hashes salted password              | Format Password & Salt                   | Format User Data                                |                                                                                                     |
| Format User Data             | Code                     | Prepares user data object for storage | Hash Password                          | Create User                                     |                                                                                                     |
| Create User                 | DataTable (create)        | Inserts new user record             | Format User Data                        | Registration Successful, Error Registration Response |                                                                                                     |
| Registration Successful     | RespondToWebhook          | Responds with success               | Create User                            | -                                              |                                                                                                     |
| Error Registration Response | RespondToWebhook          | Responds with error                 | Create User                            | -                                              |                                                                                                     |
| Username Taken Error        | RespondToWebhook          | Returns username taken error        | If Username Is Available                | -                                              |                                                                                                     |
| Email Taken Error           | RespondToWebhook          | Returns email taken error           | If Email Is Available                   | -                                              |                                                                                                     |
| Registration Request Invalid Error | RespondToWebhook    | Returns validation error            | Validate Registration Request           | -                                              |                                                                                                     |
| Login Webhook               | Webhook                   | Entry point for login requests      | -                                        | Process login webhook                           | ## 🔑 LOGIN FLOW                                                                                     |
| Process login webhook       | Set                       | Extracts login credentials          | Login Webhook                          | SET ACCESS AND REFRESH SECRET                    |                                                                                                     |
| SET ACCESS AND REFRESH SECRET | Set                     | Sets secret keys                    | Process login webhook                   | Verify Input                                    |                                                                                                     |
| Verify Input                | Code                      | Validates login input               | SET ACCESS AND REFRESH SECRET           | Get User, Bad Request                           |                                                                                                     |
| Get User                   | DataTable (get)            | Retrieves user by email             | Verify Input                          | If User Exists                                  |                                                                                                     |
| If User Exists             | If                        | Checks user existence               | Get User                              | Extract Salt & Hash, User Not Found             |                                                                                                     |
| Extract Salt & Hash        | Code                      | Extracts salt and hash from stored password | If User Exists                      | Hash Input Password                             |                                                                                                     |
| Hash Input Password        | Crypto (SHA512)            | Hashes input password + salt        | Extract Salt & Hash                    | Verify Password                                 |                                                                                                     |
| Verify Password            | Code                      | Validates input password hash       | Hash Input Password                   | Create JWT Payload, Login Credentials Invalid Response |                                                                                                     |
| Create JWT Payload         | Code                      | Builds JWT payloads for tokens      | Verify Password                      | Sign Access Token, Sign Refresh Token           | ## 🎫 TWO TOKEN SYSTEM                                                                               |
| Sign Access Token          | Crypto (HMAC SHA256)       | Signs access token                  | Create JWT Payload                   | Merge JWT Tokens                                |                                                                                                     |
| Sign Refresh Token         | Crypto (HMAC SHA256)       | Signs refresh token                 | Create JWT Payload                   | Merge JWT Tokens                                |                                                                                                     |
| Merge JWT Tokens           | Merge (combine)            | Combines signed tokens              | Sign Access Token, Sign Refresh Token | Format JWT Tokens                               |                                                                                                     |
| Format JWT Tokens          | Code                      | Constructs full JWT strings         | Merge JWT Tokens                    | Hash Refresh Token for Storage                   |                                                                                                     |
| Hash Refresh Token for Storage | Crypto (SHA256)         | Hashes refresh token for DB storage | Format JWT Tokens                  | Code in JavaScript                              | ## 🔒 DEFENSE IN DEPTH                                                                              |
| Code in JavaScript         | Code                      | Prepares DB data for refresh token  | Hash Refresh Token for Storage         | Update User Refresh Token, Store Refresh Token  |                                                                                                     |
| Update User Refresh Token  | DataTable (update)          | Updates user record with refresh token | Code in JavaScript               | Merge                                           |                                                                                                     |
| Store Refresh Token        | DataTable (upsert)          | Inserts/updates refresh token record | Code in JavaScript                | Merge                                           |                                                                                                     |
| Merge                     | Merge (chooseBranch)        | Combines update and insert results  | Update User Refresh Token, Store Refresh Token | Login Successful                              |                                                                                                     |
| Login Successful          | RespondToWebhook            | Returns tokens and user info        | Merge                               | -                                              |                                                                                                     |
| Login Credentials Invalid Response | RespondToWebhook     | Returns invalid credentials error   | Verify Password                   | -                                              |                                                                                                     |
| User Not Found            | RespondToWebhook            | Returns user not found error        | If User Exists                     | -                                              |                                                                                                     |
| Bad Request              | RespondToWebhook            | Returns bad request error            | Verify Input                      | -                                              |                                                                                                     |
| Verify Access Token       | Webhook                    | Endpoint to verify access tokens    | -                                | Process Verify Token Webhook                     | ✅ TOKEN VERIFICATION                                                                             |
| Process Verify Token Webhook | Set                      | Extracts access_token from request  | Verify Access Token             | Parse JWT                                       |                                                                                                     |
| Parse JWT                | Code                       | Parses and validates JWT token      | Process Verify Token Webhook     | SET ACCESS AND REFRESH SECRET2                    |                                                                                                     |
| SET ACCESS AND REFRESH SECRET2 | Set                   | Loads secret keys for verification  | Parse JWT                      | Verify HMAC Signature                            |                                                                                                     |
| Verify HMAC Signature     | Crypto (HMAC SHA256)        | Recreates expected signature        | SET ACCESS AND REFRESH SECRET2  | Compare Signatures                               |                                                                                                     |
| Compare Signatures        | Code                       | Compares provided and expected signatures | Verify HMAC Signature         | Access Token Valid, Access Token Invalid         |                                                                                                     |
| Access Token Valid        | RespondToWebhook            | Returns valid token response         | Compare Signatures              | -                                              |                                                                                                     |
| Access Token Invalid      | RespondToWebhook            | Returns invalid token response       | Compare Signatures              | -                                              |                                                                                                     |
| Refresh Access Token      | Webhook                    | Endpoint to refresh access tokens   | -                                | Process Refresh Token                            | ## 🔄 REFRESH PROCESS                                                                             |
| Process Refresh Token     | Set                        | Extracts refresh_token from request | Refresh Access Token            | SET ACCESS AND REFRESH SECRET1                    |                                                                                                     |
| SET ACCESS AND REFRESH SECRET1 | Set                   | Loads secret keys for refresh token verification | Process Refresh Token          | Parse Refresh Token                              |                                                                                                     |
| Parse Refresh Token       | Code                       | Parses and validates refresh token  | SET ACCESS AND REFRESH SECRET1  | Verify Signature                                 |                                                                                                     |
| Verify Signature         | Crypto (HMAC SHA256)        | Recreates expected refresh token signature | Parse Refresh Token            | Compare Refresh Token Signature                   |                                                                                                     |
| Compare Refresh Token Signature | Code                   | Compares provided and expected refresh token signatures | Verify Signature           | Hash Refresh Token For DB Lookup                  |                                                                                                     |
| Hash Refresh Token For DB Lookup | Crypto (SHA256)         | Hashes refresh token for DB lookup  | Compare Refresh Token Signature | If Refresh Token Exists                           |                                                                                                     |
| If Refresh Token Exists  | DataTable (get)             | Checks token presence in DB          | Hash Refresh Token For DB Lookup | If Refresh Token Is Valid                         |                                                                                                     |
| If Refresh Token Is Valid | If                         | Branches on token validity           | If Refresh Token Exists          | Create Access Token Payload, Session Expired     |                                                                                                     |
| Create Access Token Payload | Code                      | Builds new access token payload      | If Refresh Token Is Valid        | Sign New Access Token                             |                                                                                                     |
| Sign New Access Token     | Crypto (HMAC SHA256)        | Signs new access token               | Create Access Token Payload      | Format Access Token JWT                           |                                                                                                     |
| Format Access Token JWT   | Code                       | Constructs new access token string   | Sign New Access Token            | Return New Access Token                           |                                                                                                     |
| Return New Access Token   | RespondToWebhook            | Returns new access token to client   | Format Access Token JWT          | -                                              |                                                                                                     |
| Session Expired          | RespondToWebhook            | Returns session expired error        | If Refresh Token Is Valid        | -                                              |                                                                                                     |

---

## 4. Reproducing the Workflow from Scratch

1. **Set up Data Tables:**  
   - Create a `users` table with columns:  
     - `email` (string)  
     - `username` (string)  
     - `password_hash` (string, format `salt:hash`)  
     - `refresh_token` (string, optional)  
   - Create a `refresh_tokens` table with columns:  
     - `user_id` (number)  
     - `token_hash` (string, SHA256 hash of refresh token)  
     - `expires_at` (datetime)

2. **Create Registration Webhook:**  
   - HTTP Method: POST  
   - Path: `/register-user`  
   - Response Mode: Response Node

3. **Parse Register Request (Set Node):**  
   - Extract `email`, `username`, `password` from webhook body  

4. **Validate Registration Request (Code Node):**  
   - Check all fields present  
   - Password length ≥ 8  
   - Normalize email (lowercase, trim) and username (trim)  
   - Throw error if invalid  

5. **Get User By Username (DataTable Get):**  
   - Filter by `username`  

6. **If Username Is Available (If Node):**  
   - Condition: username query result is empty  

7. **Get User By Email (DataTable Get):**  
   - Filter by `email`  

8. **If Email Is Available (If Node):**  
   - Condition: email query result is empty  

9. **Generate Salt (Crypto Node):**  
   - Action: Generate random hex string  

10. **Format Password & Salt (Code Node):**  
    - Concatenate password + salt  

11. **Hash Password (Crypto Node):**  
    - SHA512 hash of password+salt  

12. **Format User Data (Code Node):**  
    - Format `password_hash` as `"salt:hash"`  
    - Include `email` and `username`  

13. **Create User (DataTable Create):**  
    - Insert new row into `users` table  

14. **Respond Registration Successful (RespondToWebhook Node):**  
    - HTTP 200 with user data  

15. **Error Handlers (RespondToWebhook Nodes):**  
    - For username/email taken, registration errors  

16. **Create Login Webhook:**  
    - HTTP POST `/login`  
    - Response Mode: Response Node  

17. **Process login webhook (Set Node):**  
    - Extract `email` and `password` from body  

18. **Set Secret Keys (Set Node):**  
    - Set `ACCESS_SECRET` and `REFRESH_SECRET` (ideally use n8n Variables)  

19. **Verify Input (Code Node):**  
    - Validate presence of email/password  
    - Normalize email  

20. **Get User (DataTable Get):**  
    - Filter by email  

21. **If User Exists (If Node):**  
    - Check if user found  

22. **Extract Salt & Hash (Code Node):**  
    - Split stored password hash into salt and hash  
    - Append input password + salt  

23. **Hash Input Password (Crypto Node):**  
    - SHA512 hash of input password + salt  

24. **Verify Password (Code Node):**  
    - Compare input hash with stored hash  
    - Throw error if mismatch  

25. **Create JWT Payload (Code Node):**  
    - Create access token payload (15 min expiry)  
    - Create refresh token payload (7 days expiry)  
    - Encode header and payload base64url  
    - Prepare signature input strings  

26. **Sign Access Token (Crypto Node):**  
    - HMAC SHA256 sign access token payload with `ACCESS_SECRET`  

27. **Sign Refresh Token (Crypto Node):**  
    - HMAC SHA256 sign refresh token payload with `REFRESH_SECRET`  

28. **Merge JWT Tokens (Merge Node):**  
    - Combine signed token data  

29. **Format JWT Tokens (Code Node):**  
    - Construct JWT strings: `<header>.<payload>.<signature>` (signature base64url encoded)  

30. **Hash Refresh Token for Storage (Crypto Node):**  
    - SHA256 hash refresh token string for DB storage  

31. **Prepare DB Entry (Code Node):**  
    - Include user_id, token hash, expiry, tokens  

32. **Update User Refresh Token (DataTable Update):**  
    - Update `refresh_token` field (optional)  

33. **Store Refresh Token (DataTable Upsert):**  
    - Insert or update token in `refresh_tokens` table  

34. **Merge (Merge Node):**  
    - Combine DB update and insert results  

35. **Respond Login Successful (RespondToWebhook Node):**  
    - Return access token, refresh token, user info  

36. **Error Responses (RespondToWebhook Nodes):**  
    - User not found, invalid credentials, bad request  

37. **Create Verify Token Webhook:**  
    - HTTP POST `/verify-token`  

38. **Process Verify Token Webhook (Set Node):**  
    - Extract `access_token` from request body  

39. **Parse JWT (Code Node):**  
    - Split JWT, decode payload, check expiration and type  

40. **Set Secret Keys (Set Node):**  
    - Load secret keys for verification  

41. **Verify HMAC Signature (Crypto Node):**  
    - Recreate expected signature for access token  

42. **Compare Signatures (Code Node):**  
    - Compare provided and expected signature  

43. **Respond Access Token Valid/Invalid (RespondToWebhook Nodes):**  
    - Return token validity status  

44. **Create Refresh Token Webhook:**  
    - HTTP POST `/refresh`  

45. **Process Refresh Token (Set Node):**  
    - Extract `refresh_token` from request body  

46. **Set Secret Keys (Set Node):**  
    - Load secret keys for refresh token verification  

47. **Parse Refresh Token (Code Node):**  
    - Split JWT, decode payload, check expiration  

48. **Verify Signature (Crypto Node):**  
    - Recreate expected signature for refresh token  

49. **Compare Refresh Token Signature (Code Node):**  
    - Compare provided and expected signature  

50. **Hash Refresh Token For DB Lookup (Crypto Node):**  
    - Hash full refresh token string  

51. **If Refresh Token Exists (DataTable Get):**  
    - Check existence in DB by hash and user_id  

52. **If Refresh Token Is Valid (If Node):**  
    - Branch based on token presence  

53. **Create Access Token Payload (Code Node):**  
    - Prepare new access token payload (15 min)  

54. **Sign New Access Token (Crypto Node):**  
    - Sign new access token with secret  

55. **Format Access Token JWT (Code Node):**  
    - Construct JWT string for new access token  

56. **Return New Access Token (RespondToWebhook Node):**  
    - Respond with new access token  

57. **Session Expired (RespondToWebhook Node):**  
    - Respond with session expired error  

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| ## Login Flow<br>Validates input → Finds user → Verifies password → Generates tokens → Stores refresh token → Returns tokens. Password verification extracts salt from stored "salt:hash", hashes input password with same salt, and compares hashes (never plain text!).                              | Sticky Note1, Sticky Note6                                                                             |
| ## Registration Flow<br>Checks username → Checks email → Creates user. Duplicate prevention before hashing saves resources. Password stored as "salt:hash".                                                                                                                                          | Sticky Note, Sticky Note4                                                                              |
| ## Password Security<br>Random salt generated per user, password + salt combined, hashed with SHA-512 (irreversible). Salt ensures unique hashes even for identical passwords.                                                                                                                       | Sticky Note5                                                                                          |
| ## Two Token System<br>Access token (15 min lifespan) used for API requests, not stored in DB. Refresh token (7 days lifespan) used to get new access tokens, stored in DB for revocation. Both signed with HMAC-SHA256.                                                                               | Sticky Note7                                                                                          |
| ✅ Token Verification<br>Parse token, check expiration, recreate signature, and compare. Invalid signature means tampering; expired tokens are rejected.                                                                                                                                             | Sticky Note8                                                                                          |
| ## Refresh Process<br>Client sends refresh token, verify signature, check expiration, lookup in DB (hashed), generate new access token. Database lookup enables token revocation and theft detection.                                                                                               | Sticky Note9                                                                                          |
| ## Defense in Depth<br>Refresh tokens hashed before storing. If DB leaks, attackers cannot use tokens directly and must crack SHA-256 hashes.                                                                                                                                                        | Sticky Note10                                                                                         |
| ## Critical Security<br>Use two secret keys: ACCESS_SECRET and REFRESH_SECRET. Keys must be identical across login, verify, and refresh workflows; mismatches cause signature failures.                                                                                                             | Sticky Note11                                                                                        |
| ## Troubleshooting<br>Common errors include "Invalid signature" (secret keys mismatch), "Token not found in DB" (hashing issues), "Cannot read property" (node reference errors), login failures (password hash format), and refresh failures (token storage issues).                                | Sticky Note12                                                                                        |
| ## Test Sequence<br>Register new user → Login → Copy tokens → Verify access token → Wait or test refresh → Use refresh token for new access token → Check error responses for troubleshooting.                                                                                                      | Sticky Note13                                                                                        |
| # Workflow Overview And Setup<br>Summary of all flows: Registration, Login, Verify Token, Refresh Token, and client flow.                                                                                                                                                                            | Sticky Note35                                                                                        |
| ## Table Setup<br>Use n8n Data Tables feature for users and refresh_tokens with specified columns. Access tokens are stateless and not stored.                                                                                                                                                      | Sticky Note14                                                                                        |
| ⚙️ Customization Points<br>Change token lifespan by adjusting expiration timestamps, update hash algorithms (must be consistent), add fields to JWT payload as needed, and revoke tokens by deleting or invalidating refresh tokens in DB.                                                         | Sticky Note15                                                                                        |
| ## Variables Recommendation<br>It is recommended to use n8n Variables for secret keys instead of Set nodes to ensure secrets are consistent and secure across workflows.                                                                                                                           | Sticky Note33, Sticky Note34                                                                          |
| ## Branding and Credits<br>This workflow is a comprehensive example to add authentication to any application using n8n. For more details, visit the official n8n documentation or community forums.                                                                                                | https://docs.n8n.io/                                                                                   |

---

**Disclaimer:**  
This document is generated exclusively from an n8n workflow automation export. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---