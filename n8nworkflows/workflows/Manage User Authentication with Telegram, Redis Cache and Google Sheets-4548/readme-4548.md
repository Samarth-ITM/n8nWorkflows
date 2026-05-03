Manage User Authentication with Telegram, Redis Cache and Google Sheets

https://n8nworkflows.xyz/workflows/manage-user-authentication-with-telegram--redis-cache-and-google-sheets-4548


# Manage User Authentication with Telegram, Redis Cache and Google Sheets

### 1. Workflow Overview

This workflow implements a **user authentication and management system** integrating **Telegram**, **Redis caching**, and **Google Sheets** as a user database. Its primary purpose is to handle login requests from Telegram users, verify their existence and status in a Google Sheets user registry, utilize Redis for caching to improve performance, and update or create user entries as needed.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receiving authentication attempts from Telegram messages or external workflow triggers.
- **1.2 User Cache Verification:** Checking Redis cache for existing user data to avoid redundant lookups.
- **1.3 User Lookup and Creation:** Searching Google Sheets for user records, deciding if the user is new, and creating new users if necessary.
- **1.4 User ID Retrieval and Merging:** Extracting and consolidating user ID information from different branches for downstream processing.
- **1.5 Data Preparation and Merging:** Setting and merging user-related fields to format the data correctly for cache updates or responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block handles the initial receipt of user authentication requests either directly from Telegram messages or from another workflow trigger, establishing the workflow's entry points.

- **Nodes Involved:**  
  - Get Message  
  - When Executed by Another Workflow

- **Node Details:**

  - **Get Message**  
    - *Type:* Telegram Trigger  
    - *Role:* Listens for incoming Telegram messages to initiate login attempts.  
    - *Configuration:* Uses default webhook settings to receive messages.  
    - *Inputs:* External Telegram messages.  
    - *Outputs:* Passes message data downstream for processing.  
    - *Edge Cases:* Telegram API downtime, webhook misconfiguration, message format inconsistencies.

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Allows triggering this login workflow programmatically from other workflows, facilitating integration.  
    - *Configuration:* Default.  
    - *Inputs:* Calls from other workflows.  
    - *Outputs:* Starts user data retrieval.  
    - *Edge Cases:* Missing or malformed input data, execution permission issues.

---

#### 2.2 User Cache Verification

- **Overview:**  
  This block checks if the user data is already cached in Redis to speed up authentication without querying Google Sheets unnecessarily.

- **Nodes Involved:**  
  - Merge  
  - Find Cached User  
  - Is Cached

- **Node Details:**

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Combines data streams from Telegram and external workflow triggers for unified processing.  
    - *Configuration:* Default.  
    - *Inputs:* Data from "Edit Fields" (Telegram) and "Edit Fields2" (external trigger).  
    - *Outputs:* Passes merged data downstream.  
    - *Edge Cases:* Data format mismatches causing merge failures.

  - **Find Cached User**  
    - *Type:* Redis  
    - *Role:* Queries Redis cache for user data keyed by user identifier.  
    - *Configuration:* Connects to Redis instance; uses user ID or Telegram ID as key.  
    - *Inputs:* Merged user data.  
    - *Outputs:* User data if found in cache.  
    - *Edge Cases:* Redis connection issues, key not found, data expiration.

  - **Is Cached**  
    - *Type:* Switch  
    - *Role:* Branches workflow based on cache hit or miss.  
    - *Configuration:* Evaluates if Redis query returned user data.  
    - *Inputs:* Output from "Find Cached User".  
    - *Outputs:*  
      - True: To "Find User" (Google Sheets lookup).  
      - False: To "UserId" (bypass cache).  
    - *Edge Cases:* Expression errors in condition, ambiguous data.

---

#### 2.3 User Lookup and Creation

- **Overview:**  
  When cache misses occur, this block looks up user details in Google Sheets, determines if the user is new, and creates user entries as necessary.

- **Nodes Involved:**  
  - Find User  
  - Is New User  
  - Cache User  
  - Get UserId  
  - Create User

- **Node Details:**

  - **Find User**  
    - *Type:* Google Sheets  
    - *Role:* Searches user records in Google Sheets by user identifier.  
    - *Configuration:* Reads the user sheet, filters by user ID or Telegram ID.  
    - *Inputs:* From "Is Cached" switch (cache miss path).  
    - *Outputs:* User record data.  
    - *Edge Cases:* Sheet access permission errors, data format inconsistencies.

  - **Is New User**  
    - *Type:* Switch  
    - *Role:* Determines if the user exists in Google Sheets (new or existing).  
    - *Configuration:* Checks if "Find User" returns empty or record found.  
    - *Inputs:* User data from "Find User".  
    - *Outputs:*  
      - True (new user): To "Cache User" (cache new user) branch.  
      - False (existing user): To "Get UserId" branch.  
    - *Edge Cases:* Logical errors in condition leading to wrong branch.

  - **Cache User**  
    - *Type:* Redis  
    - *Role:* Caches the newly found or created user data in Redis for future requests.  
    - *Configuration:* Writes user info keyed by user ID or Telegram ID.  
    - *Inputs:* User data from "Is New User" (new user path).  
    - *Outputs:* Passes to "UserId" node.  
    - *Edge Cases:* Redis write failures, data serialization issues.

  - **Get UserId**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts or formats user ID from user data for downstream use.  
    - *Configuration:* Custom JavaScript code to parse user info.  
    - *Inputs:* Existing user data from "Is New User" (existing user path).  
    - *Outputs:* Passes processed user ID to "Create User".  
    - *Edge Cases:* Code execution errors, unexpected data structures.

  - **Create User**  
    - *Type:* Google Sheets  
    - *Role:* Inserts new user entries into Google Sheets if user does not exist.  
    - *Configuration:* Appends new row with user details.  
    - *Inputs:* User ID data from "Get UserId".  
    - *Outputs:* Passes data to "UserId" node.  
    - *Edge Cases:* Sheet permission errors, row insertion conflicts.

---

#### 2.4 User ID Retrieval and Merging

- **Overview:**  
  This block merges user ID data from different sources (cache or newly created/found in Google Sheets) to unify downstream handling.

- **Nodes Involved:**  
  - UserId

- **Node Details:**

  - **UserId**  
    - *Type:* Merge  
    - *Role:* Consolidates user ID outputs from "Cache User" and "Create User" nodes.  
    - *Configuration:* Default merge settings.  
    - *Inputs:* Outputs from both cache and creation branches.  
    - *Outputs:* Unified user ID data downstream.  
    - *Edge Cases:* Merge conflicts or empty data.

---

#### 2.5 Data Preparation and Merging

- **Overview:**  
  This block sets and prepares user-related fields for caching or further processing and merges data streams from different input paths.

- **Nodes Involved:**  
  - Get User  
  - Edit Fields  
  - Edit Fields2  
  - Merge

- **Node Details:**

  - **Get User**  
    - *Type:* Google Sheets  
    - *Role:* Retrieves user details from Google Sheets when triggered externally.  
    - *Configuration:* Reads user data for specified user IDs.  
    - *Inputs:* Triggered by "When Executed by Another Workflow" node.  
    - *Outputs:* Passes user data to "Edit Fields2".  
    - *Edge Cases:* Sheet access issues, invalid parameters.

  - **Edit Fields**  
    - *Type:* Set  
    - *Role:* Mutates or formats incoming Telegram message data to prepare for caching or lookup.  
    - *Configuration:* Sets or modifies key fields like user ID, username.  
    - *Inputs:* From "Get Message".  
    - *Outputs:* Passes to "Merge".  
    - *Edge Cases:* Misconfigured field mappings causing data loss.

  - **Edit Fields2**  
    - *Type:* Set  
    - *Role:* Similar to "Edit Fields" but processes data from external workflow triggers.  
    - *Configuration:* Sets or formats fields for consistency.  
    - *Inputs:* From "Get User".  
    - *Outputs:* Passes to "Merge".  
    - *Edge Cases:* Similar risks to "Edit Fields".

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Combines the prepared data from Telegram messages and external workflow sources for cache querying.  
    - *Configuration:* Default.  
    - *Inputs:* From "Edit Fields" and "Edit Fields2".  
    - *Outputs:* Passes merged data downstream to "Find Cached User".  
    - *Edge Cases:* Data conflicts or missing fields.

---

### 3. Summary Table

| Node Name                | Node Type                | Functional Role                                          | Input Node(s)                          | Output Node(s)                        | Sticky Note                                                                                         |
|--------------------------|--------------------------|----------------------------------------------------------|--------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------|
| Get Message              | Telegram Trigger         | Receives Telegram messages to start login process        | —                                    | Edit Fields                        |                                                                                                   |
| When Executed by Another Workflow | Execute Workflow Trigger | Allows external workflows to trigger login process       | —                                    | Get User                         |                                                                                                   |
| Edit Fields              | Set                      | Prepares Telegram message data for processing            | Get Message                         | Merge                            |                                                                                                   |
| Edit Fields2             | Set                      | Prepares external workflow input data                     | Get User                           | Merge                            |                                                                                                   |
| Merge                    | Merge                    | Combines Telegram and external workflow data              | Edit Fields, Edit Fields2           | Find Cached User                 |                                                                                                   |
| Find Cached User         | Redis                    | Queries Redis cache for user data                          | Merge                              | Is Cached                       |                                                                                                   |
| Is Cached                | Switch                   | Branches based on cache hit or miss                        | Find Cached User                   | Find User (if miss), UserId (if hit) |                                                                                                   |
| Find User                | Google Sheets            | Looks up user in Google Sheets                             | Is Cached (cache miss branch)       | Is New User                    |                                                                                                   |
| Is New User              | Switch                   | Determines if user exists or is new                        | Find User                         | Cache User (new user), Get UserId (existing user) |                                                                                                   |
| Cache User               | Redis                    | Caches new user data                                       | Is New User (true path)              | UserId                          |                                                                                                   |
| Get UserId               | Code                     | Extracts user ID from user data                            | Is New User (false path)             | Create User                    |                                                                                                   |
| Create User              | Google Sheets            | Creates new user entry                                     | Get UserId                        | UserId                          |                                                                                                   |
| UserId                   | Merge                    | Merges user ID data from cache and creation branches      | Cache User, Create User             | —                               |                                                                                                   |
| Get User                 | Google Sheets            | Retrieves user details when triggered externally          | When Executed by Another Workflow   | Edit Fields2                   |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Telegram Trigger node ("Get Message")**  
   - Type: Telegram Trigger  
   - Configure webhook to receive messages from your Telegram bot.  
   - No parameters required beyond default.  

2. **Create the Execute Workflow Trigger node ("When Executed by Another Workflow")**  
   - Type: Execute Workflow Trigger  
   - Default configuration to allow external triggers.  

3. **Create "Edit Fields" Set node**  
   - Type: Set  
   - Configure to extract and set necessary user fields from Telegram message, e.g., user ID, username.  
   - Connect input from "Get Message".  

4. **Create "Get User" Google Sheets node**  
   - Type: Google Sheets (Read operation)  
   - Configure credentials for Google Sheets access.  
   - Setup to read user information by passed user identifiers.  
   - Connect input from "When Executed by Another Workflow".  

5. **Create "Edit Fields2" Set node**  
   - Type: Set  
   - Configure similarly to "Edit Fields" but based on data from "Get User".  
   - Connect input from "Get User".  

6. **Create "Merge" node**  
   - Type: Merge  
   - Configure to merge incoming data from "Edit Fields" and "Edit Fields2".  
   - Connect both "Edit Fields" and "Edit Fields2" outputs to this node.  

7. **Create "Find Cached User" Redis node**  
   - Type: Redis  
   - Configure Redis credentials and connection settings.  
   - Setup to query user data with user ID as key.  
   - Connect input from "Merge".  

8. **Create "Is Cached" Switch node**  
   - Type: Switch  
   - Configure condition to check if Redis query returned valid user data (e.g., if data exists).  
   - Connect input from "Find Cached User".  

9. **Create "Find User" Google Sheets node**  
   - Type: Google Sheets (Read operation)  
   - Configure to search Google Sheets for user by user ID if cache miss.  
   - Connect from "Is Cached" (cache miss output).  

10. **Create "Is New User" Switch node**  
    - Type: Switch  
    - Configure to check if "Find User" returned empty (new user) or existing record.  
    - Connect input from "Find User".  

11. **Create "Cache User" Redis node**  
    - Type: Redis  
    - Configure to cache new user data in Redis.  
    - Connect input from "Is New User" (new user path).  

12. **Create "Get UserId" Code node**  
    - Type: Code (JavaScript)  
    - Write code to extract user ID from user data.  
    - Connect input from "Is New User" (existing user path).  

13. **Create "Create User" Google Sheets node**  
    - Type: Google Sheets (Append operation)  
    - Configure to append new user data to Google Sheets.  
    - Connect input from "Get UserId".  

14. **Create "UserId" Merge node**  
    - Type: Merge  
    - Configure to merge outputs from "Cache User" and "Create User".  
    - Connect inputs from both these nodes.  

15. **Connect "UserId" output as needed for downstream processing or responses.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                       |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| Workflow integrates Telegram, Redis, and Google Sheets to efficiently manage user authentication.   | Project description.                                                  |
| Redis caching is used to optimize performance and reduce Google Sheets calls.                        | Redis node documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.redis/ |
| Google Sheets nodes require OAuth2 credentials with proper spreadsheet access permissions.          | Google Sheets node docs: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleSheets/ |
| Telegram webhook must be correctly configured to receive bot messages.                              | Telegram bot API docs: https://core.telegram.org/bots/api           |

---

This document fully describes the "Login System" workflow, enabling understanding, modification, and reproduction without referring back to the JSON source.