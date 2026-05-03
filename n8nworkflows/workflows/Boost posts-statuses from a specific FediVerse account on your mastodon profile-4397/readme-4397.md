Boost posts/statuses from a specific FediVerse account on your mastodon profile

https://n8nworkflows.xyz/workflows/boost-posts-statuses-from-a-specific-fediverse-account-on-your-mastodon-profile-4397


# Boost posts/statuses from a specific FediVerse account on your mastodon profile

### 1. Workflow Overview

This workflow automates the process of boosting (reblogging) posts from a specific Mastodon account on your Mastodon profile. It targets users who want to automatically promote content from a Federated social media account (Fediverse) by reposting recent statuses on their own profile.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Starts the workflow execution at a fixed daily time.
- **1.2 Fetch Recent Statuses:** Retrieves all statuses from a specified Mastodon account.
- **1.3 Filter Recent Posts:** Filters statuses to keep only those created today.
- **1.4 Boost Filtered Statuses:** Automatically boosts (reblogs) the filtered statuses on your Mastodon profile.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
This block initiates the workflow execution every day at 23:59 hours, ensuring a daily refresh and promotion of posts.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  

  - **Schedule Trigger**  
    - **Type:** Schedule Trigger node  
    - **Role:** Starts the workflow on a time interval  
    - **Configuration:** Trigger set to daily at 23:59 (11:59 PM)  
    - **Key Expressions:** None; uses fixed static time configuration  
    - **Input:** None (trigger node)  
    - **Output:** Triggers the next node "Get statuses from threads.net"  
    - **Version Requirements:** v1.2 or higher recommended for stable scheduling  
    - **Potential Failures:** None typical unless n8n scheduler service is down  

#### 2.2 Fetch Recent Statuses

- **Overview:**  
Retrieves all statuses from a specific Mastodon account using the Mastodon API.

- **Nodes Involved:**  
  - Get statuses from threads.net

- **Node Details:**  

  - **Get statuses from threads.net**  
    - **Type:** HTTP Request node  
    - **Role:** Fetches statuses from Mastodon API endpoint  
    - **Configuration:**  
      - URL: `https://mastodon.social/api/v1/accounts/<ACCOUNT_ID>/statuses`  
      - Method: GET (default)  
      - Authentication: None specified (public endpoint assumed)  
      - Options: Default HTTP options, no special headers or parameters  
    - **Key Expressions:** None dynamic, static URL except for `<ACCOUNT_ID>` placeholder to be replaced with the target user's Mastodon account ID  
    - **Input:** Trigger from Schedule Trigger node  
    - **Output:** JSON array of statuses, each containing metadata including `id`, `created_at`, etc.  
    - **Version Requirements:** v4.2 or higher for improved HTTP request capabilities  
    - **Potential Failures:**  
      - HTTP errors (404 if invalid account, 429 rate limiting, 500 server errors)  
      - Network timeouts  
      - Invalid or missing `<ACCOUNT_ID>` causing failed API calls  

#### 2.3 Filter Recent Posts

- **Overview:**  
Filters the retrieved statuses to include only those posted today (based on UTC date).

- **Nodes Involved:**  
  - Filter today

- **Node Details:**  

  - **Filter today**  
    - **Type:** Filter node  
    - **Role:** Applies date filter on statuses to pass only today's posts  
    - **Configuration:**  
      - Condition: Keep items where `created_at` is on or after today's date (ISO string, date part only)  
      - Expression used:  
        ```javascript
        $json.created_at >= new Date().toISOString().split('T')[0]
        ```  
      - Comparison is strict and case-sensitive for date validation  
    - **Input:** JSON array of statuses from 'Get statuses from threads.net'  
    - **Output:** Filtered subset of statuses created today  
    - **Version Requirements:** v2.2 or higher for advanced filter condition support  
    - **Potential Failures:**  
      - Date parsing errors if `created_at` field is missing or malformed  
      - Timezone differences causing edge cases (e.g., posts near midnight UTC)  

#### 2.4 Boost Filtered Statuses

- **Overview:**  
Posts a reblog (boost) request for each filtered status to the Mastodon API, effectively resharing the content on your own profile.

- **Nodes Involved:**  
  - Boost statuses

- **Node Details:**  

  - **Boost statuses**  
    - **Type:** HTTP Request node  
    - **Role:** Sends POST requests to Mastodon API to boost each status  
    - **Configuration:**  
      - URL Template:  
        ```
        https://mastodon.social/api/v1/statuses/{{ $json.id }}/reblog
        ```  
      - Method: POST  
      - Authentication: HTTP Header with Bearer token  
      - Credential: Mastodon access token stored in `httpHeaderAuth` credentials  
    - **Key Expressions:** Uses dynamic expression to insert status `id` from filtered JSON items into the URL path  
    - **Input:** Filtered statuses from Filter today node  
    - **Output:** Response from Mastodon API confirming reblog success or failure per status  
    - **Version Requirements:** v1 or higher, standard HTTP Request node  
    - **Potential Failures:**  
      - Authentication errors (invalid or expired token)  
      - API rate limiting or server errors  
      - Attempting to boost a status already boosted (may cause 400 or 409 errors)  
      - Network or timeout issues  

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                        | Input Node(s)               | Output Node(s)           | Sticky Note                                  |
|-----------------------------|--------------------|-------------------------------------|-----------------------------|--------------------------|----------------------------------------------|
| Schedule Trigger             | Schedule Trigger   | Initiates workflow at 23:59 daily   | None                        | Get statuses from threads.net |                                              |
| Get statuses from threads.net | HTTP Request       | Fetches statuses from Mastodon API  | Schedule Trigger            | Filter today             |                                              |
| Filter today                | Filter             | Filters statuses to keep today's    | Get statuses from threads.net | Boost statuses           |                                              |
| Boost statuses             | HTTP Request       | Boosts each filtered status on Mastodon | Filter today               | None                     | Requires Mastodon access token credential for authorization |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it “Post Threads to Mastodon”.**

2. **Add a Schedule Trigger node:**  
   - Set the trigger interval to run daily at 23:59 (11:59 PM).  
   - No input parameters needed.  

3. **Add an HTTP Request node named “Get statuses from threads.net”:**  
   - Set method to GET.  
   - Set URL to `https://mastodon.social/api/v1/accounts/<ACCOUNT_ID>/statuses` replacing `<ACCOUNT_ID>` with the target Mastodon account ID.  
   - No authentication required if the account statuses are public.  
   - Connect Schedule Trigger output to this node's input.

4. **Add a Filter node named “Filter today”:**  
   - Configure to filter where `created_at` field is on or after today's date.  
   - Use expression for the right value: `={{ new Date().toISOString().split('T')[0] }}`  
   - Left value: `{{$json["created_at"]}}`  
   - Condition operator: `afterOrEquals`  
   - Connect “Get statuses from threads.net” output to this node's input.

5. **Add an HTTP Request node named “Boost statuses”:**  
   - Set method to POST.  
   - Set URL to `https://mastodon.social/api/v1/statuses/{{ $json.id }}/reblog`  
   - Set authentication to HTTP Header Authentication with a Mastodon access token credential.  
     - Configure credentials in n8n: Add a new HTTP Header Auth credential with header `Authorization: Bearer <YOUR_ACCESS_TOKEN>`  
   - Connect “Filter today” output to this node's input.

6. **Save the workflow and test:**  
   - Run manually or wait for scheduled time.  
   - Verify in Mastodon that statuses from the target account posted today have been boosted.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                         |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow uses Mastodon’s official REST API for accessing statuses and boosting posts.                  | Mastodon API Docs: https://docs.joinmastodon.org/api/  |
| Ensure your Mastodon access token has the necessary scopes for boosting posts (`write:statuses`).          | OAuth2 Scope Requirements for Mastodon                  |
| Timezone considerations: Filtering by UTC date may exclude posts depending on local time zones.            | Adjust filter logic if local timezone is preferred     |
| Replace `<ACCOUNT_ID>` with the numeric ID of the Mastodon account, not the username or handle.             | Retrieve account ID via Mastodon API `/accounts/lookup` endpoint |
| To avoid boosting duplicate posts, consider adding logic to track boosted status IDs externally.           | Custom enhancement suggestion                            |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.