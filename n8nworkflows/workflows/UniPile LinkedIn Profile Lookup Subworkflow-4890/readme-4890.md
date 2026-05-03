UniPile LinkedIn Profile Lookup Subworkflow

https://n8nworkflows.xyz/workflows/unipile-linkedin-profile-lookup-subworkflow-4890


# UniPile LinkedIn Profile Lookup Subworkflow

### 1. Workflow Overview

This workflow, titled **UniPile LinkedIn Profile Lookup Subworkflow**, is designed to enrich LinkedIn profile data based on an input identifier (usually a LinkedIn user or organization ID). It receives a sender ID from another workflow and attempts to fetch detailed LinkedIn user data via the Unipile API. If the user data is not found, it falls back to searching for organization data tied to the same ID. If neither user nor organization data is found, it outputs a not-found message.

The workflow is logically divided into three main functional blocks:

- **1.1 Input Reception:** Receives execution trigger and input parameters from an external workflow.
- **1.2 User Data Retrieval and Processing:** Queries the Unipile API for LinkedIn user data and processes the response.
- **1.3 Organization Data Retrieval and Processing / Fallback:** If user data is unavailable, queries organization data and processes it; if none found, outputs a "not found" object.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when called by another workflow and extracts the `sender` input parameter, which is the LinkedIn identifier to look up.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Configuration: Configured to receive one input parameter named `sender` from the calling workflow.  
    - Key expressions: Accesses `sender` value via `$json.sender`.  
    - Input connections: Trigger node (no input).  
    - Output connections: Connects to "Get Linkedin User Data from Unipile".  
    - Edge cases: Missing or invalid `sender` input may cause downstream HTTP requests to fail or return empty data.  
    - Version: 1.1  

#### 2.2 User Data Retrieval and Processing

- **Overview:**  
  This block attempts to retrieve LinkedIn user data from the Unipile API using the sender ID. If successful, it maps relevant user fields into a structured object and groups them for output. If the user is not found (error or empty result), it triggers the organization data lookup block.

- **Nodes Involved:**  
  - Get Linkedin User Data from Unipile  
  - Set User Data from Unipile  
  - Group in one object - User

- **Node Details:**  

  - **Get Linkedin User Data from Unipile**  
    - Type: HTTP Request  
    - Configuration:  
      - GET request to `https://api9.unipile.com:13976/api/v1/users/{{ $json.sender }}`  
      - Query parameter `account_id` set to a fixed value `-oWmWRkASjKGUZadYcjcyg`  
      - Header includes `accept: application/json`  
      - Authenticates using HTTP Header Auth via stored credential "unipile angel"  
      - On error: Continue execution and send error output downstream (allow fallback)  
    - Input: Receives `sender` from previous node  
    - Output: JSON response expected to contain user fields (e.g., first_name, last_name, headline, websites, follower_count, etc.)  
    - Edge cases:  
      - HTTP errors or timeouts  
      - Empty or malformed JSON responses  
      - Authorization failures if credentials expire or are invalid  
    - Version: 4.2  

  - **Set User Data from Unipile**  
    - Type: Set Node  
    - Configuration: Assigns specific fields from the HTTP response JSON to new variables with clearer naming: first name, last name, headline, websites, follower counts, location, profile picture URL, influencer and premium flags, shared connections count, and marks the `type` as "user".  
    - Key expressions: Uses expressions like `={{ $json.first_name }}` to safely map data.  
    - Input: Output of HTTP request node  
    - Output: Structured user data object  
    - Edge cases: Input data missing some fields results in empty or null assignments.  
    - Version: 3.4  

  - **Group in one object - User**  
    - Type: Aggregate  
    - Configuration: Aggregates all input data items into a single object under the field `linkedinprofile` for output.  
    - Input: Output of the Set node  
    - Output: Single aggregated JSON object with user profile data under `linkedinprofile`  
    - Edge cases: Empty input will result in empty aggregation.  
    - Version: 1  

#### 2.3 Organization Data Retrieval and Processing / Fallback

- **Overview:**  
  If user data retrieval fails (e.g., user not found), this block attempts to retrieve LinkedIn organization (company) data using the same sender ID. It processes and maps organization fields similarly, then aggregates the data. If no organization data is found either, it outputs a default "unable to find LinkedIn Data" message.

- **Nodes Involved:**  
  - Get Linkedin Org Data from Unipile  
  - Set Linkedin Org Data from Unipile  
  - Group in one object - Org  
  - Set unable to find data object

- **Node Details:**  

  - **Get Linkedin Org Data from Unipile**  
    - Type: HTTP Request  
    - Configuration:  
      - GET request to `https://api9.unipile.com:13976/api/v1/linkedin/company/{{ $json.sender }}`  
      - Query parameter `account_id` as above  
      - Header: `accept: application/json`  
      - Auth via HTTP Header Auth credential "unipile angel"  
      - On error: Continue and send error output downstream (allow fallback)  
    - Input: Receives `sender` from prior node or fallback  
    - Output: JSON response with organization/company fields (e.g., name, description, website, followers_count, employee_count_range, locations, hashtags, logo)  
    - Edge cases: Same as user HTTP request node (timeouts, authorization, empty responses)  
    - Version: 4.2  

  - **Set Linkedin Org Data from Unipile**  
    - Type: Set Node  
    - Configuration: Maps organization fields into structured variables:  
      - `first_name` set to organization name  
      - `last_name` empty string  
      - `headline` set to description  
      - `websites` set to website URL (string)  
      - `follower_count` set to followers_count  
      - `employee_count` from employee_count_range.from  
      - `location` constructed from first entry in locations array (concatenated street, city, area, postal code, country)  
      - `hashtags` array from JSON  
      - `profile_picture_url` set to logo URL  
      - `type` set to "organization"  
    - Key expressions: Complex expression for location concatenation:  
      ```javascript
      {{$json.locations[0].street.join(' ')}}, {{$json.locations[0].city}} {{$json.locations[0].area}}, {{$json.locations[0].postalCode}} {{$json.locations[0].country}}.
      ```  
    - Input: Output of Org HTTP Request  
    - Output: Structured organization profile object  
    - Edge cases: Missing location array or fields may cause undefined or errors; fallback to empty or partial strings advisable.  
    - Version: 3.4  

  - **Group in one object - Org**  
    - Type: Aggregate  
    - Configuration: Aggregates all incoming items into one object with field `linkedinprofile`  
    - Input: Output of Set Linkedin Org Data node  
    - Output: Single aggregated organization profile object  
    - Edge cases: Empty input leads to empty aggregation.  
    - Version: 1  

  - **Set unable to find data object**  
    - Type: Set Node  
    - Configuration: Sets a single field `linkedinprofile` with string value `"Unable to find LinkedIn Data"`  
    - Input: Triggered only if Org HTTP Request fails or returns no data  
    - Output: Object signaling no data found  
    - Edge cases: None, except being the final fallback.  
    - Version: 3.4  

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                         | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                           |
|-------------------------------|----------------------------------|---------------------------------------|--------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry point; receives `sender` input | None                           | Get Linkedin User Data from Unipile |                                                                                                     |
| Get Linkedin User Data from Unipile | HTTP Request                     | Fetch user LinkedIn data from Unipile API | When Executed by Another Workflow | Set User Data from Unipile, Get Linkedin Org Data from Unipile |                                                                                                     |
| Set User Data from Unipile       | Set                              | Map and structure user data fields    | Get Linkedin User Data from Unipile | Group in one object - User          |                                                                                                     |
| Group in one object - User       | Aggregate                        | Aggregate user data into one object   | Set User Data from Unipile      | None                              |                                                                                                     |
| Get Linkedin Org Data from Unipile | HTTP Request                     | Fetch organization LinkedIn data if user not found | Get Linkedin User Data from Unipile (on error) | Set Linkedin Org Data from Unipile, Set unable to find data object |                                                                                                     |
| Set Linkedin Org Data from Unipile | Set                              | Map and structure organization data fields | Get Linkedin Org Data from Unipile | Group in one object - Org           |                                                                                                     |
| Group in one object - Org        | Aggregate                        | Aggregate organization data into one object | Set Linkedin Org Data from Unipile | None                              |                                                                                                     |
| Set unable to find data object   | Set                              | Output fallback message if no data found | Get Linkedin Org Data from Unipile (on error) | None                              |                                                                                                     |
| Sticky Note8                    | Sticky Note                     | Explains workflow logic and purpose  | None                           | None                              | ![unipile](https://uploads.n8n.io/templates/unipile.png)  \n## Enrich LinkedIn Data with User or Org Data  \nLinkedIn messages arrive from one of two object types, users or organizations. This workflow extracts the user data and passes it along. If the user is not found, then the Organization endpoint is searched instead. If it's found, it send that, otherwise it sends a not found object. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and name it "UniPile LinkedIn Profile Lookup Subworkflow".**

2. **Add node: "When Executed by Another Workflow" (Trigger)**
   - Type: Execute Workflow Trigger  
   - Configure parameters to accept input named `sender`.  
   - Position: leftmost, top.  
   - No credentials needed.  

3. **Add node: "Get Linkedin User Data from Unipile" (HTTP Request)**
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api9.unipile.com:13976/api/v1/users/{{ $json.sender }}`  
   - Query Parameters:  
     - `account_id` = `-oWmWRkASjKGUZadYcjcyg`  
   - Headers:  
     - `accept` = `application/json`  
   - Authentication: HTTP Header Auth using credential "unipile angel" (configure this credential in n8n with your API key).  
   - Error Handling: On error, continue execution (to allow fallback).  
   - Connect input from "When Executed by Another Workflow".  

4. **Add node: "Set User Data from Unipile" (Set)**
   - Type: Set  
   - Assign fields:  
     - `first_name`: `={{ $json.first_name }}`  
     - `last_name`: `={{ $json.last_name }}`  
     - `headline`: `={{ $json.headline }}`  
     - `websites`: `={{ $json.websites }}` (as array)  
     - `follower_count`: `={{ $json.follower_count }}` (number)  
     - `connections_count`: `={{ $json.connections_count }}` (number)  
     - `location`: `={{ $json.location }}`  
     - `profile_picture_url`: `={{ $json.profile_picture_url }}`  
     - `is_influencer`: `={{ $json.is_influencer }}` (boolean)  
     - `is_premium`: `={{ $json.is_premium }}` (boolean)  
     - `shared_connections_count`: `={{ $json.shared_connections_count }}` (number)  
     - `type`: `"user"` (string literal)  
   - Connect input from "Get Linkedin User Data from Unipile".  

5. **Add node: "Group in one object - User" (Aggregate)**
   - Type: Aggregate  
   - Aggregate all item data into field `linkedinprofile`.  
   - Connect input from "Set User Data from Unipile".  

6. **Add node: "Get Linkedin Org Data from Unipile" (HTTP Request)**
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api9.unipile.com:13976/api/v1/linkedin/company/{{ $json.sender }}`  
   - Query Parameters:  
     - `account_id` = `-oWmWRkASjKGUZadYcjcyg`  
   - Headers:  
     - `accept` = `application/json`  
   - Authentication: HTTP Header Auth using credential "unipile angel"  
   - Error Handling: On error continue execution to allow fallback.  
   - Connect input from "Get Linkedin User Data from Unipile" (error output branch).  

7. **Add node: "Set Linkedin Org Data from Unipile" (Set)**
   - Type: Set  
   - Assign fields:  
     - `first_name`: `={{ $json.name }}`  
     - `last_name`: `=""` (empty string)  
     - `headline`: `={{ $json.description }}`  
     - `websites`: `={{ $json.website }}` (string)  
     - `follower_count`: `={{ $json.followers_count }}` (number)  
     - `employee_count`: `={{ $json.employee_count_range.from }}` (number)  
     - `location`:  
       ```
       ={{$json.locations[0].street.join(' ')}}, {{$json.locations[0].city}} {{$json.locations[0].area}}, {{$json.locations[0].postalCode}} {{$json.locations[0].country}}.
       ```  
     - `hashtags`: `={{ $json.hashtags }}` (array)  
     - `profile_picture_url`: `={{ $json.logo }}`  
     - `type`: `"organization"` (string literal)  
   - Connect input from "Get Linkedin Org Data from Unipile".  

8. **Add node: "Group in one object - Org" (Aggregate)**
   - Type: Aggregate  
   - Aggregate all item data into field `linkedinprofile`.  
   - Connect input from "Set Linkedin Org Data from Unipile".  

9. **Add node: "Set unable to find data object" (Set)**
   - Type: Set  
   - Assign field:  
     - `linkedinprofile`: `"Unable to find LinkedIn Data"` (string literal)  
   - Connect input from "Get Linkedin Org Data from Unipile" error output branch (fallback if org data is also not found).  

10. **Connect output branches accordingly:**
    - "Get Linkedin User Data from Unipile" main output → "Set User Data from Unipile"  
    - "Get Linkedin User Data from Unipile" error output → "Get Linkedin Org Data from Unipile"  
    - "Get Linkedin Org Data from Unipile" main output → "Set Linkedin Org Data from Unipile"  
    - "Get Linkedin Org Data from Unipile" error output → "Set unable to find data object"  
    - "Set User Data from Unipile" → "Group in one object - User"  
    - "Set Linkedin Org Data from Unipile" → "Group in one object - Org"  

11. **Credential Setup:**
    - Create HTTP Header Auth credential in n8n named "unipile angel" containing the API key or token required by Unipile API.  

12. **Test the workflow by triggering it with a sample `sender` ID.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| ![unipile](https://uploads.n8n.io/templates/unipile.png)  \nEnrich LinkedIn Data with User or Org Data. LinkedIn messages may come from users or orgs. | Sticky Note in workflow explains the high-level logic and purpose of the workflow.             |
| The workflow uses the Unipile API endpoints for user and organization LinkedIn data lookup. Ensure API access rights and quotas are respected.         | API documentation and usage policies should be checked with Unipile (https://unipile.com).     |
| When handling location data for organizations, the workflow assumes the first location entry exists and contains certain fields; missing data may cause errors. | Consider adding additional checks or defaults if locations array or fields are absent in responses. |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.