Crypto Exchange Listing & Delisting Alerts to Telegram, X, and Discord

https://n8nworkflows.xyz/workflows/crypto-exchange-listing---delisting-alerts-to-telegram--x--and-discord-8236


# Crypto Exchange Listing & Delisting Alerts to Telegram, X, and Discord

### 1. Workflow Overview

This workflow monitors cryptocurrency exchange listing and delisting events and sends alerts about these changes to multiple social and messaging platforms: Telegram, Twitter (X), and Discord. It periodically fetches the latest listings and delistings data from external APIs, compares them with stored records in Supabase to detect changes, and then publishes formatted alerts for new events.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger & Data Fetching:** Periodically triggers the workflow and concurrently fetches listing and delisting data via HTTP requests.
- **1.2 Data Processing & Filtering:** Splits the fetched data into individual items, limits processing for performance, and retrieves corresponding stored entries from Supabase to check for new or updated events.
- **1.3 Conditional Filtering:** Uses conditional logic to determine if each item is new or updated and thus requires alerting.
- **1.4 Alert Publishing:** Sends alerts about detected listings and delistings to Telegram, Twitter, and Discord channels.
- **1.5 Data Persistence:** Stores new or updated event IDs back into Supabase for future reference and to prevent duplicate alerts.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Fetching

**Overview:**  
This block initiates the workflow on a defined schedule and fetches the latest listings and delistings data from external APIs.

**Nodes Involved:**  
- Schedule Trigger  
- Http request (Delistings)  
- HTTP Request (listings)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow at regular intervals (default schedule not specified, likely periodic).  
  - Configuration: No parameters shown; likely triggers every defined time period.  
  - Inputs: None  
  - Outputs: Triggers two parallel HTTP request nodes.  
  - Edge Cases: Scheduling misconfiguration or downtime could disrupt data update frequency.

- **Http request (Delistings)**  
  - Type: HTTP Request  
  - Role: Fetches delisting information from a REST API.  
  - Configuration: API endpoint and request details are configured but not shown explicitly; assumed to return a list of delisting events.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Raw delisting data to "Split Items" node.  
  - Edge Cases: Network failures, API rate limits, unexpected response formats.

- **HTTP Request (listings)**  
  - Type: HTTP Request  
  - Role: Fetches listing information from a REST API.  
  - Configuration: API endpoint and request details configured similarly to delistings.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Raw listing data to "Split Items1" node.  
  - Edge Cases: Same as above.

---

#### 2.2 Data Processing & Filtering

**Overview:**  
Splits the bulk data into individual items, limits the number of processed items for performance, and checks against stored Supabase records to see if each event is new or already processed.

**Nodes Involved:**  
- Split Items  
- Limit  
- Get a row  
- If  
- Split Items1  
- Limit1  
- Get a row1  
- If1

**Node Details:**

- **Split Items / Split Items1**  
  - Type: Code  
  - Role: Splits the array of listings or delistings into individual items for processing one-by-one.  
  - Configuration: Custom JavaScript code (not shown) that likely iterates over the API response array to output separate items.  
  - Inputs: Data from respective HTTP Request nodes.  
  - Outputs: Individual items forwarded to Limit nodes.  
  - Edge Cases: Malformed data arrays or empty responses.

- **Limit / Limit1**  
  - Type: Limit  
  - Role: Restricts the number of items processed downstream to avoid overloading.  
  - Configuration: Default limit; number not shown but typically used to prevent excessive API calls or processing.  
  - Inputs: Individual items from Split Items nodes.  
  - Outputs: Limited number of items passed to Supabase queries.  
  - Edge Cases: If limit is too low, some data might be skipped; too high could cause performance issues.

- **Get a row / Get a row1**  
  - Type: Supabase  
  - Role: Queries Supabase database to check if the current item’s ID or unique identifier already exists (indicating it was previously processed).  
  - Configuration: Query configured to fetch row by unique identifier for listings or delistings.  
  - Inputs: Limited individual items.  
  - Outputs: Result passed to conditional "If" nodes.  
  - Edge Cases: Database connectivity or authentication issues; missing data or schema changes.

- **If / If1**  
  - Type: If  
  - Role: Checks if the Supabase query found an existing row.  
  - Configuration: Condition likely checks if the row is null or undefined to determine if the item is new.  
  - Inputs: Supabase query results.  
  - Outputs: True branch (new item) leads to alert nodes; false branch ignored.  
  - Edge Cases: Expression failures if data is malformed; false positives or negatives if IDs are inconsistent.

---

#### 2.3 Alert Publishing

**Overview:**  
For each newly detected listing or delisting, the workflow sends notifications to Telegram, Twitter, and Discord to inform subscribers of the event.

**Nodes Involved:**  
- Send Delisting Alert To Telegram  
- Send Listing Alert To Telegram  
- Create Tweet Delisting  
- Create Tweet Listing  
- Send a message(Delistings)  
- Send a message (new listings)

**Node Details:**

- **Send Delisting Alert To Telegram / Send Listing Alert To Telegram**  
  - Type: Telegram  
  - Role: Sends formatted alert messages about listings or delistings to a Telegram channel or group.  
  - Configuration: Uses Telegram bot credentials and webhooks; message formatting not shown but assumed informative.  
  - Inputs: True outputs from respective If nodes.  
  - Outputs: None (end nodes).  
  - Edge Cases: Telegram API rate limits, invalid bot tokens, chat permissions.

- **Create Tweet Delisting / Create Tweet Listing**  
  - Type: Twitter  
  - Role: Posts tweets announcing new delistings or listings.  
  - Configuration: Uses OAuth2 credentials; message content likely dynamically constructed from item data.  
  - Inputs: True outputs from respective If nodes.  
  - Outputs: None (end nodes).  
  - Edge Cases: Twitter API limits, revoked credentials, content size limits.

- **Send a message(Delistings) / Send a message (new listings)**  
  - Type: Discord  
  - Role: Sends messages to Discord channels about new delistings or listings.  
  - Configuration: Uses Discord webhook URLs; messages formatted for clarity.  
  - Inputs: True outputs from respective If nodes.  
  - Outputs: Passes to Supabase storage nodes.  
  - Edge Cases: Webhook URL invalidation, rate limits, Discord service outages.

---

#### 2.4 Data Persistence

**Overview:**  
Stores the IDs of newly alerted listings and delistings back in Supabase to ensure they are not alerted multiple times in subsequent runs.

**Nodes Involved:**  
- Store ID for future Updates  
- Store ID for future updates1

**Node Details:**

- **Store ID for future Updates / Store ID for future updates1**  
  - Type: Supabase  
  - Role: Inserts or updates records in Supabase database with the event IDs after alerts are sent.  
  - Configuration: Database table and columns configured to store IDs and possibly timestamps.  
  - Inputs: From Discord message nodes (indicating alerts were successfully sent).  
  - Outputs: None (end nodes).  
  - Edge Cases: Database write failures, permission issues, data integrity constraints.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                          | Input Node(s)                    | Output Node(s)                                   | Sticky Note                                           |
|----------------------------|---------------------|----------------------------------------|---------------------------------|-------------------------------------------------|-------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger     | Initiates workflow on schedule         | None                            | Http request (Delistings), HTTP Request (listings) |                                                       |
| Http request (Delistings)  | HTTP Request        | Fetch delisting data                    | Schedule Trigger                | Split Items                                      |                                                       |
| HTTP Request (listings)    | HTTP Request        | Fetch listing data                      | Schedule Trigger                | Split Items1                                     |                                                       |
| Split Items                | Code                | Split delisting data array              | Http request (Delistings)       | Limit                                           |                                                       |
| Split Items1               | Code                | Split listing data array                | HTTP Request (listings)         | Limit1                                          |                                                       |
| Limit                     | Limit               | Limit number of delisting items         | Split Items                    | Get a row                                       |                                                       |
| Limit1                    | Limit               | Limit number of listing items           | Split Items1                   | Get a row1                                      |                                                       |
| Get a row                 | Supabase            | Check delisting ID existence            | Limit                         | If                                              |                                                       |
| Get a row1                | Supabase            | Check listing ID existence               | Limit1                        | If1                                             |                                                       |
| If                        | If                  | Condition: Is delisting new?             | Get a row                     | Send a message(Delistings), Send Delisting Alert To Telegram, Create Tweet Delisting |                                                       |
| If1                       | If                  | Condition: Is listing new?                | Get a row1                    | Send a message (new listings), Send Listing Alert To Telegram, Create Tweet Listing |                                                       |
| Send a message(Delistings) | Discord             | Send delisting alert to Discord          | If                           | Store ID for future Updates                      |                                                       |
| Send a message (new listings) | Discord           | Send listing alert to Discord             | If1                          | Store ID for future updates1                     |                                                       |
| Send Delisting Alert To Telegram | Telegram        | Send delisting alert to Telegram          | If                           | None                                            |                                                       |
| Send Listing Alert To Telegram | Telegram          | Send listing alert to Telegram             | If1                          | None                                            |                                                       |
| Create Tweet Delisting     | Twitter             | Tweet about delisting                     | If                           | None                                            |                                                       |
| Create Tweet Listing       | Twitter             | Tweet about listing                       | If1                          | None                                            |                                                       |
| Store ID for future Updates | Supabase            | Store delisting IDs for future checks    | Send a message(Delistings)    | None                                            |                                                       |
| Store ID for future updates1 | Supabase           | Store listing IDs for future checks       | Send a message (new listings) | None                                            |                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to run at desired intervals (e.g., every 10 minutes).

2. **Create two HTTP Request nodes:**  
   - Names: "Http request (Delistings)" and "HTTP Request (listings)"  
   - Configure each with the appropriate API endpoint URLs to fetch delisting and listing data respectively.  
   - Set method to GET, configure authentication if required.

3. **Connect Schedule Trigger output to both HTTP Request nodes in parallel.**

4. **Create two Code nodes:**  
   - Names: "Split Items" (for delistings) and "Split Items1" (for listings)  
   - Insert JavaScript code to split the incoming JSON array into individual items.  
   - Connect "Http request (Delistings)" to "Split Items" and "HTTP Request (listings)" to "Split Items1".

5. **Add two Limit nodes:**  
   - Names: "Limit" and "Limit1"  
   - Configure to limit the number of items processed (e.g., 10 or 20).  
   - Connect "Split Items" to "Limit" and "Split Items1" to "Limit1".

6. **Add two Supabase nodes:**  
   - Names: "Get a row" and "Get a row1"  
   - Configure with Supabase credentials.  
   - Set up to query a table (e.g., "alerts") filtering by the unique identifier of each item (e.g., coin ID).  
   - Connect "Limit" to "Get a row" and "Limit1" to "Get a row1".

7. **Add two If nodes:**  
   - Names: "If" and "If1"  
   - Configure each to check if the Supabase query returned a row:  
     - Condition: if no row exists, treat as new event (true branch).  
   - Connect "Get a row" to "If" and "Get a row1" to "If1".

8. **Add alert sending nodes for delistings:**  
   - Telegram node: "Send Delisting Alert To Telegram"  
   - Twitter node: "Create Tweet Delisting"  
   - Discord node: "Send a message(Delistings)"  
   - Configure each with the respective credentials (Telegram bot token, Twitter OAuth2, Discord webhook URL).  
   - Connect the true output of "If" to all three.

9. **Add alert sending nodes for listings:**  
   - Telegram node: "Send Listing Alert To Telegram"  
   - Twitter node: "Create Tweet Listing"  
   - Discord node: "Send a message (new listings)"  
   - Configure credentials similarly.  
   - Connect the true output of "If1" to all three.

10. **Add two Supabase nodes for storing IDs:**  
    - Names: "Store ID for future Updates" (for delistings) and "Store ID for future updates1" (for listings)  
    - Configure to insert or update the event ID in Supabase.  
    - Connect "Send a message(Delistings)" to "Store ID for future Updates"  
    - Connect "Send a message (new listings)" to "Store ID for future updates1".

11. **Verify and test connections and data flow.**  
    - Check that each API call returns expected data.  
    - Ensure Supabase queries and inserts work correctly.  
    - Confirm alerts are posted to Telegram, Twitter, and Discord.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                           |
|------------------------------------------------------------------------------------------------|------------------------------------------|
| Workflow connects crypto exchange listing/delisting events to Telegram, Twitter, and Discord.  | Workflow purpose summary                  |
| Requires valid API credentials for Telegram bot, Twitter (OAuth2), Discord webhooks, and Supabase credentials. | Credential setup instructions             |
| Rate limits and API failures should be handled externally or through retry mechanisms if needed. | General robustness consideration          |
| For splitting array items, JavaScript code node is used to convert bulk API response to individual items. | Custom code block details                  |
| Supabase is used as persistent storage to avoid duplicate alerts.                              | Database integration details              |

---

This document provides a comprehensive reference for understanding, reproducing, and maintaining the "Crypto Exchange Listing & Delisting Alerts to Telegram, X, and Discord" workflow in n8n.