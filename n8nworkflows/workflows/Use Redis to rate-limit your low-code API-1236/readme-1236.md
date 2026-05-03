Use Redis to rate-limit your low-code API

https://n8nworkflows.xyz/workflows/use-redis-to-rate-limit-your-low-code-api-1236


# Use Redis to rate-limit your low-code API

### 1. Workflow Overview

This workflow demonstrates how to implement API rate limiting using Redis within an n8n low-code automation environment. The primary purpose is to control the number of API requests a user can make based on their unique API key, enforcing limits both per minute and per hour. The workflow identifies users through their API key sent in the request header, increments counters in Redis accordingly, and either allows access to data (from Airtable) or returns a rate-limit exceeded message.

The workflow logic is structured into these main blocks:

- **1.1 Input Reception:** Receives incoming API requests via a webhook, extracting the API key from request headers.
- **1.2 Per-minute Rate Limiting:** Uses Redis to increment and check request counts per minute.
- **1.3 Per-hour Rate Limiting:** Uses Redis to increment and check request counts per hour.
- **1.4 Data Fetching:** Retrieves data from Airtable when rate limits are not exceeded.
- **1.5 Response Formatting:** Formats the final output message or error message for the API response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Receives incoming HTTP requests via webhook, authenticates them, and extracts the API key from headers. Sets Redis keys incorporating time components to differentiate rate limits by minute and hour.

- **Nodes Involved:**  
  - Webhook1  
  - Set  
  - Set2

- **Node Details:**

  - **Webhook1**  
    - Type: Webhook  
    - Role: Entry point for incoming API requests. Authenticates requests using HTTP header authentication.  
    - Configuration: Path set to a unique ID; authentication enabled via headerAuth with predefined credentials.  
    - Input: HTTP request  
    - Output: JSON object containing request headers and data  
    - Edge Cases: Missing or invalid API key; authentication failure returns 401 Unauthorized.  
    - Notes: This node initiates the workflow.

  - **Set**  
    - Type: Set  
    - Role: Constructs a Redis key string for per-minute rate limiting by combining the API key from headers with current hour and minute.  
    - Configuration: Sets a field `apiKey` with expression `{{$json["headers"]["x-api-key"] + '-' + new Date().getHours() + '-' + new Date().getMinutes()}}`.  
    - Input: Webhook1 output  
    - Output: JSON with constructed per-minute Redis key  
    - Edge Cases: If `x-api-key` header missing, key will be malformed; could cause Redis errors downstream.

  - **Set2**  
    - Type: Set  
    - Role: Constructs a Redis key string for per-hour rate limiting by combining the API key with current hour only.  
    - Configuration: Sets `apiKey` as `{{$node['Webhook1'].json["headers"]["x-api-key"] + '-' + new Date().getHours()}}`.  
    - Input: Downstream of Redis per-minute check (via Per minute node)  
    - Output: JSON with constructed per-hour Redis key  
    - Edge Cases: Similar to Set node, missing header causes malformed keys.

---

#### 1.2 Per-minute Rate Limiting

- **Overview:**  
  Increments the per-minute request count in Redis and checks whether the user has exceeded the per-minute limit of 10 requests.

- **Nodes Involved:**  
  - Redis  
  - Per minute  
  - Set2 (also involved here as it is connected downstream)  
  - Set3

- **Node Details:**

  - **Redis**  
    - Type: Redis  
    - Role: Increments the Redis key corresponding to the per-minute limit; sets key expiration to 3600 seconds (1 hour) to reset counters.  
    - Configuration:  
      - Key: `{{$json["apiKey"]}}` (from Set node)  
      - Operation: `incr` (increment)  
      - TTL: 3600 seconds  
      - Expire: true  
    - Input: JSON with per-minute key from Set node  
    - Output: Incremented count for this minute’s key  
    - Edge Cases: Redis connectivity issues; key expiration misconfiguration; key collision if constructed key malformed.

  - **Per minute (IF node)**  
    - Type: If  
    - Role: Compares the incremented count with limit of 10 requests per minute; routes flow accordingly.  
    - Configuration:  
      - Condition: Check if Redis count (`{{$json[$node["Set"].json["apiKey"]]}}`) ≤ 10  
      - True output: Proceed to per-hour check (Set2 node)  
      - False output: Rate limit exceeded path (Set3)  
    - Input: Redis node output  
    - Output: Conditional routing  
    - Edge Cases: Expression evaluation errors if keys missing.

  - **Set2**  
    - As above, constructs per-hour key for further rate limiting.

  - **Set3**  
    - Type: Set  
    - Role: Prepares error message JSON `"You exceeded your limit"` for per-minute limit breach.  
    - Configuration: Sets `message` field with fixed string.  
    - Input: From Per minute node’s false branch  
    - Output: JSON error message  
    - Edge Cases: None significant.

---

#### 1.3 Per-hour Rate Limiting

- **Overview:**  
  Increments the per-hour Redis key counter and checks if the user has exceeded the hourly limit of 60 requests.

- **Nodes Involved:**  
  - Redis1  
  - Per hour  
  - Airtable  
  - Set1

- **Node Details:**

  - **Redis1**  
    - Type: Redis  
    - Role: Increment the Redis key for the per-hour rate limit without TTL (no expiration set explicitly here).  
    - Configuration:  
      - Key: `{{$json["apiKey"]}}` (from Set2 node)  
      - Operation: `incr`  
      - TTL: not set (no expiration).  
    - Input: JSON with per-hour Redis key from Set2  
    - Output: Incremented per-hour count  
    - Edge Cases: Redis errors; lack of TTL may cause stale counts if not cleaned.

  - **Per hour (IF node)**  
    - Type: If  
    - Role: Checks if hourly count is ≤ 60.  
    - Configuration:  
      - Condition: `{{$json[$node["Set2"].json["apiKey"]]}} ≤ 60`  
      - True branch: Proceed to data fetching (Airtable)  
      - False branch: Rate limit exceeded path (Set1)  
    - Input: Redis1 output  
    - Output: Conditional routing  
    - Edge Cases: Expression errors if keys missing.

  - **Airtable**  
    - Type: Airtable node  
    - Role: Queries the "Pokemon" table to retrieve data when rate limits are not exceeded.  
    - Configuration:  
      - Operation: List records  
      - Table: "Pokemon"  
      - No filters/conditions  
    - Input: True branch from Per hour node  
    - Output: List of records  
    - Edge Cases: API key/auth errors for Airtable; empty table; request timeouts.

  - **Set1**  
    - Type: Set  
    - Role: Constructs error message JSON `"You exceeded your limit"` for per-hour limit breach.  
    - Configuration: Sets `message` field with fixed string.  
    - Input: False branch from Per hour node  
    - Output: JSON error message  
    - Edge Cases: None significant.

---

#### 1.4 Response Formatting

- **Overview:**  
  Formats the final response message including the rate limit consumption and data fetched from Airtable.

- **Nodes Involved:**  
  - Function

- **Node Details:**

  - **Function**  
    - Type: Function (JavaScript)  
    - Role: Constructs a JSON response that contains:  
      - A message showing the limit consumed (current hourly count)  
      - An array of objects with `name` and `url` fields extracted from the Airtable records  
    - Configuration:  
      - Reads hourly count from `Redis1` node output using dynamic key from `Set2` node's `apiKey`  
      - Maps Airtable items to simplified objects with name and url  
      - Returns combined JSON  
    - Input: Airtable node output and Redis1 count (indirect by connection)  
    - Output: JSON response for the API caller  
    - Edge Cases: Null or missing Airtable data; key not found in Redis output; malformed input data.

---

### 3. Summary Table

| Node Name  | Node Type          | Functional Role                  | Input Node(s)      | Output Node(s)       | Sticky Note                                  |
|------------|--------------------|--------------------------------|--------------------|----------------------|----------------------------------------------|
| Webhook1   | Webhook            | Receives API requests and auth | -                  | Set                  |                                              |
| Set        | Set                | Constructs per-minute Redis key | Webhook1           | Redis                |                                              |
| Redis      | Redis              | Increments per-minute count     | Set                | Per minute            |                                              |
| Per minute | If                 | Checks per-minute limit ≤ 10    | Redis              | Set2, Set3            |                                              |
| Set2       | Set                | Constructs per-hour Redis key   | Per minute (true)  | Redis1                |                                              |
| Redis1     | Redis              | Increments per-hour count       | Set2               | Per hour              |                                              |
| Per hour   | If                 | Checks per-hour limit ≤ 60      | Redis1              | Airtable, Set1        |                                              |
| Airtable   | Airtable           | Fetches data if within limits   | Per hour (true)    | Function              |                                              |
| Set1       | Set                | Prepares error message (hourly) | Per hour (false)   | -                     |                                              |
| Set3       | Set                | Prepares error message (minute) | Per minute (false) | -                     |                                              |
| Function   | Function           | Formats final API response      | Airtable            | -                     |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node ("Webhook1")**  
   - Type: Webhook  
   - Path: Unique string (e.g., `a3167ed7-98d2-422c-bfe2-e3ba599d19f1`)  
   - Authentication: HTTP Header Auth using predefined credentials  
   - Purpose: Receive API requests and authenticate.

2. **Create Set node ("Set")**  
   - Type: Set  
   - Add String field `apiKey`  
   - Expression: `{{$json["headers"]["x-api-key"] + '-' + new Date().getHours() + '-' + new Date().getMinutes()}}`  
   - Connect Webhook1 → Set.

3. **Create Redis node ("Redis")**  
   - Type: Redis  
   - Operation: Increment (`incr`)  
   - Key: `{{$json["apiKey"]}}` (from Set)  
   - TTL: 3600 seconds (1 hour)  
   - Expire: Enabled  
   - Connect Set → Redis.

4. **Create If node ("Per minute")**  
   - Type: If  
   - Condition: Number check  
   - Value1: `{{$json[$node["Set"].json["apiKey"]]}}`  
   - Operation: smaller or equal  
   - Value2: 10  
   - Connect Redis → Per minute.

5. **Create Set node ("Set3")**  
   - Type: Set  
   - Add String field `message` with value `"You exceeded your limit"`  
   - Connect Per minute (false) → Set3.

6. **Create Set node ("Set2")**  
   - Type: Set  
   - Add String field `apiKey`  
   - Expression: `{{$node["Webhook1"].json["headers"]["x-api-key"] + '-' + new Date().getHours()}}`  
   - Connect Per minute (true) → Set2.

7. **Create Redis node ("Redis1")**  
   - Type: Redis  
   - Operation: Increment (`incr`)  
   - Key: `{{$json["apiKey"]}}` (from Set2)  
   - TTL: Not set (default)  
   - Connect Set2 → Redis1.

8. **Create If node ("Per hour")**  
   - Type: If  
   - Condition: Number check  
   - Value1: `{{$json[$node["Set2"].json["apiKey"]]}}`  
   - Operation: smaller or equal  
   - Value2: 60  
   - Connect Redis1 → Per hour.

9. **Create Set node ("Set1")**  
   - Type: Set  
   - Add String field `message` with value `"You exceeded your limit"`  
   - Connect Per hour (false) → Set1.

10. **Create Airtable node ("Airtable")**  
    - Type: Airtable  
    - Operation: List  
    - Table: "Pokemon"  
    - Connect Per hour (true) → Airtable.  
    - Configure Airtable credentials accordingly.

11. **Create Function node ("Function")**  
    - Type: Function  
    - Code:  
      ```javascript
      const limit = `Limit consumed: ` + $node['Redis1'].json[$node["Set2"].json["apiKey"]];
      return [
        {
          json: {
            message: limit,
            body: items.map(item => {
              const name = item.json.fields.name;
              const url = item.json.fields.url;
              return { name, url };
            }),
          },
        },
      ];
      ```  
    - Connect Airtable → Function.

12. **Configure Webhook Response**  
    - Set the webhook node's response mode to "lastNode" so the response is sent from the last executed node (Function or error message Sets).

13. **Credential Setup**  
    - Configure and assign Redis credentials for Redis and Redis1 nodes (e.g., Redis Cloud credentials).  
    - Configure Airtable credentials for the Airtable node.  
    - Configure Header Authentication credentials for Webhook1 node.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                              |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Duplicate this Airtable base to test the workflow: https://airtable.com/shraudfG9XAvqkBpF       | Airtable data source for Pokémon data                         |
| Workflow screenshot referenced (fileId:538) is used for visualization in the original context. | Useful for UI layout understanding (not included here)        |
| Rate limits are enforced with Redis keys combining API key and time components to segregate counts | Ensures accurate per-minute and per-hour rate limiting logic |

---

This documentation enables full comprehension, replication, and modification of the API rate limiting workflow leveraging Redis counters and Airtable data retrieval within n8n.