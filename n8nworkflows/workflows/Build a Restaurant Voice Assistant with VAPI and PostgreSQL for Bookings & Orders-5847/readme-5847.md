Build a Restaurant Voice Assistant with VAPI and PostgreSQL for Bookings & Orders

https://n8nworkflows.xyz/workflows/build-a-restaurant-voice-assistant-with-vapi-and-postgresql-for-bookings---orders-5847


# Build a Restaurant Voice Assistant with VAPI and PostgreSQL for Bookings & Orders

---

### 1. Workflow Overview

This n8n workflow titled **"Voice Assistant – Restaurant Booking, Orders & Info System"** is designed to implement a voice-driven assistant for a restaurant. It supports two primary use cases:

- **1.1 Table Booking & Order Handling:**  
  The assistant receives voice requests for table reservations or food orders, saves these details into a PostgreSQL database, and immediately confirms the booking or order back to the user.

- **1.2 Restaurant Information Provider:**  
  The assistant handles voice requests seeking information about the restaurant (e.g., opening hours, menu), retrieves relevant data from the database, and responds with the requested details.

The workflow is logically divided into two main blocks based on these functionalities, each triggered by separate webhook nodes that interface with a voice assistant platform (VAPI). Each block then interacts with a PostgreSQL database for persistence or data retrieval and finally sends a structured JSON response back to the voice assistant for user feedback.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Table Booking & Order Handling

- **Overview:**  
  This block handles incoming voice requests related to table bookings and food orders. It waits for any necessary processing, updates or inserts booking/order data into the PostgreSQL database, and sends confirmation back to the voice assistant.

- **Nodes Involved:**  
  - Trigger: Voice Request (VAPI)  
  - Wait For Response  
  - Update Data (Table Booking / Orders)  
  - Respond: Booking/Order Confirmation (VAPI)  
  - Sticky Note (contextual documentation)

- **Node Details:**

  1. **Trigger: Voice Request (VAPI)**  
     - *Type & Role:* Webhook node, acts as the entry point for voice booking/order requests via HTTP POST.  
     - *Configuration:* Listens on a unique webhook path; expects POST requests with voice assistant payload.  
     - *Key Expressions:* Payload accessed via `body.message.toolCalls[0].id` to track conversation context.  
     - *Input/Output:* No input nodes; outputs to "Wait For Response".  
     - *Failure Modes:* HTTP request invalid, missing payload, or webhook misconfiguration can cause failures.  
     - *Version:* Uses webhook typeVersion 2.

  2. **Wait For Response**  
     - *Type & Role:* Wait node; used to pause the workflow, likely to await external processing or user interaction.  
     - *Configuration:* No timeout or custom parameters set, defaults apply.  
     - *Input/Output:* Receives from "Trigger: Voice Request (VAPI)"; outputs to "Update Data (Table Booking / Orders)".  
     - *Failure Modes:* Potential indefinite wait if external event is not triggered; requires external webhook or manual resume.

  3. **Update Data (Table Booking / Orders)**  
     - *Type & Role:* PostgreSQL node to upsert booking/order data into the database.  
     - *Configuration:*  
       - Operation: Upsert (update existing or insert new record)  
       - Schema: "public"  
       - Table: "id" (likely a table name or possibly misconfigured; usually should be a table name like "bookings")  
       - Columns auto-mapped from input data  
     - *Credentials:* Uses an existing PostgreSQL credential named "Postgres-test".  
     - *Input/Output:* Input from "Wait For Response"; outputs to "Respond: Booking/Order Confirmation (VAPI)".  
     - *Failure Modes:* Database connection errors, constraint violations, or mapping errors.  
     - *Version:* PostgreSQL node version 2.6.

  4. **Respond: Booking/Order Confirmation (VAPI)**  
     - *Type & Role:* Respond to Webhook node; sends JSON confirmation response back to the voice assistant.  
     - *Configuration:*  
       - Response body dynamically constructed using an expression:  
         ```json
         {
           "results": [
             {
               "toolCallId": "{{ $('Trigger: Voice Request (VAPI)').item.json.body.message.toolCalls[0].id }}",
               "result": "{{ $json.status }}"
             }
           ]
         }
         ```  
       - Responds with JSON containing the original toolCallId and status from the previous node's JSON.  
     - *Input/Output:* Input from "Update Data (Table Booking / Orders)"; terminates the webhook response cycle.  
     - *Failure Modes:* Expression failures if expected fields missing; improper JSON format causing response errors.  
     - *Version:* Respond to Webhook node version 1.2.

  5. **Sticky Note**  
     - *Content Summary:* Provides high-level context for Table Booking & Order Handling block:  
       > - Workflow lets the voice assistant take table reservations and food orders.  
       > - Saves booking/order details into the database and confirms instantly.

---

#### 2.2 Restaurant Information Provider

- **Overview:**  
  This block handles voice requests for restaurant information, such as availability or details. It queries the PostgreSQL database for the requested information and responds with the data in JSON format to the voice assistant.

- **Nodes Involved:**  
  - Trigger: Info Request (VAPI)  
  - Get Restaurant Info (Postgres)  
  - Wait For Response1  
  - Respond: Restaurant Details (VAPI)  
  - Sticky Note1 (contextual documentation)

- **Node Details:**

  1. **Trigger: Info Request (VAPI)**  
     - *Type & Role:* Webhook node, entry point for info-related voice requests via HTTP POST.  
     - *Configuration:* Unique webhook path specified; listens for POST requests.  
     - *Key Expressions:* Extracts `body.message.toolCalls[0].id` for response correlation.  
     - *Input/Output:* No input nodes; outputs to "Get Restaurant Info (Postgres)".  
     - *Failure Modes:* Missing or malformed payloads; webhook path conflicts.  
     - *Version:* Webhook node version 2.

  2. **Get Restaurant Info (Postgres)**  
     - *Type & Role:* PostgreSQL node to select restaurant information from a database table.  
     - *Configuration:*  
       - Operation: Select  
       - Schema: "public"  
       - Table: "id" (likely a placeholder or misconfiguration; expected to be the actual table name storing restaurant info)  
       - No specific columns or filters set (full table select implied).  
     - *Credentials:* Uses "Postgres-test" PostgreSQL credential.  
     - *Input/Output:* Input from "Trigger: Info Request (VAPI)"; outputs to "Wait For Response1".  
     - *Failure Modes:* DB connectivity issues, table name errors, large data volumes causing timeouts.  
     - *Version:* PostgreSQL node version 2.6.

  3. **Wait For Response1**  
     - *Type & Role:* Wait node; pauses workflow execution, possibly to synchronize response timing.  
     - *Configuration:* Default wait parameters.  
     - *Input/Output:* Input from "Get Restaurant Info (Postgres)"; outputs to "Respond: Restaurant Details (VAPI)".  
     - *Failure Modes:* Indefinite waiting if external trigger missing.

  4. **Respond: Restaurant Details (VAPI)**  
     - *Type & Role:* Respond to Webhook node; returns restaurant information JSON to the voice assistant.  
     - *Configuration:*  
       - Response body built dynamically with expression:  
         ```json
         {
           "results": [
             {
               "toolCallId": "{{ $('Trigger: Info Request (VAPI)').item.json.body.message.toolCalls[0].id }}",
               "result": "{{ $json.available }}"
             }
           ]
         }
         ```  
       - Sends back the toolCallId and an "available" field from the query results.  
     - *Input/Output:* Input from "Wait For Response1"; ends webhook response.  
     - *Failure Modes:* Expression errors if "available" field missing; malformed JSON output.  
     - *Version:* Respond to Webhook node version 1.2.

  5. **Sticky Note1**  
     - *Content Summary:* Describes the Restaurant Info Provider block:  
       > - Voice assistant answers queries about restaurant info.  
       > - Saves and confirms details instantly (note: text identical to first sticky note, likely copy-paste).

---

### 3. Summary Table

| Node Name                         | Node Type              | Functional Role                         | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                   |
|----------------------------------|------------------------|---------------------------------------|-------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| Trigger: Voice Request (VAPI)     | Webhook                | Entry for booking/order voice requests| None                          | Wait For Response                     | See Table Booking & Order Handling description in Sticky Note                                |
| Wait For Response                | Wait                   | Pause before processing booking/order | Trigger: Voice Request (VAPI)  | Update Data (Table Booking / Orders) | See Table Booking & Order Handling description in Sticky Note                                |
| Update Data (Table Booking / Orders)| PostgreSQL            | Upsert booking/order data in DB       | Wait For Response             | Respond: Booking/Order Confirmation   | See Table Booking & Order Handling description in Sticky Note                                |
| Respond: Booking/Order Confirmation (VAPI) | Respond to Webhook | Confirm booking/order to voice assistant | Update Data (Table Booking / Orders)| None                                 | See Table Booking & Order Handling description in Sticky Note                                |
| Sticky Note                     | Sticky Note            | Documentation note                    | None                          | None                                  | ## Table Booking & Order Handling - explains workflow purpose and flow                      |
| Trigger: Info Request (VAPI)      | Webhook                | Entry for restaurant info voice requests | None                          | Get Restaurant Info (Postgres)        | See Restaurant Info Provider description in Sticky Note                                     |
| Get Restaurant Info (Postgres)    | PostgreSQL             | Select restaurant info from DB        | Trigger: Info Request (VAPI)  | Wait For Response1                    | See Restaurant Info Provider description in Sticky Note                                     |
| Wait For Response1               | Wait                   | Pause before responding with info     | Get Restaurant Info (Postgres) | Respond: Restaurant Details (VAPI)   | See Restaurant Info Provider description in Sticky Note                                     |
| Respond: Restaurant Details (VAPI)| Respond to Webhook    | Return restaurant info to voice assistant | Wait For Response1            | None                                  | See Restaurant Info Provider description in Sticky Note                                     |
| Sticky Note1                    | Sticky Note            | Documentation note                    | None                          | None                                  | ## Restaurant Info Provider - repeats workflow purpose and flow                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n titled "Voice Assistant – Restaurant Booking, Orders & Info System".**

2. **Create the "Table Booking & Order Handling" block:**

   a. Add a **Webhook node** named **"Trigger: Voice Request (VAPI)"**  
      - Set HTTP Method to POST  
      - Path: a unique string (e.g., "9f6b9125-0b75-41c2-87a9-1b6746c89e2e")  
      - Response Mode: "Response Node" (delays response until respond node)  

   b. Add a **Wait node** named **"Wait For Response"**  
      - Default settings (no timeout)  

   c. Connect "Trigger: Voice Request (VAPI)" output to "Wait For Response" input.

   d. Add a **PostgreSQL node** named **"Update Data (Table Booking / Orders)"**  
      - Operation: Upsert  
      - Schema: public  
      - Table: (set actual bookings/orders table name, e.g., "bookings")  
      - Columns: Auto map input data  
      - Credentials: Configure PostgreSQL credentials (name it "Postgres-test" or as preferred)  

   e. Connect "Wait For Response" output to "Update Data (Table Booking / Orders)".

   f. Add a **Respond to Webhook node** named **"Respond: Booking/Order Confirmation (VAPI)"**  
      - Respond with: JSON  
      - Response Body:  
        ```json
        {
          "results": [
            {
              "toolCallId": "{{ $('Trigger: Voice Request (VAPI)').item.json.body.message.toolCalls[0].id }}",
              "result": "{{ $json.status }}"
            }
          ]
        }
        ```  

   g. Connect "Update Data (Table Booking / Orders)" output to "Respond: Booking/Order Confirmation (VAPI)".

3. **Create the "Restaurant Information Provider" block:**

   a. Add a **Webhook node** named **"Trigger: Info Request (VAPI)"**  
      - HTTP Method: POST  
      - Path: unique string (e.g., "efe2c13f-1ba5-46e1-9996-57bdc6041973")  
      - Response Mode: "Response Node"  

   b. Add a **PostgreSQL node** named **"Get Restaurant Info (Postgres)"**  
      - Operation: Select  
      - Schema: public  
      - Table: (set actual restaurant info table name, e.g., "restaurant_info")  
      - Credentials: Use same PostgreSQL credentials as above  

   c. Connect "Trigger: Info Request (VAPI)" output to "Get Restaurant Info (Postgres)".

   d. Add a **Wait node** named **"Wait For Response1"**  
      - Default settings  

   e. Connect "Get Restaurant Info (Postgres)" output to "Wait For Response1".

   f. Add a **Respond to Webhook node** named **"Respond: Restaurant Details (VAPI)"**  
      - Respond with: JSON  
      - Response Body:  
        ```json
        {
          "results": [
            {
              "toolCallId": "{{ $('Trigger: Info Request (VAPI)').item.json.body.message.toolCalls[0].id }}",
              "result": "{{ $json.available }}"
            }
          ]
        }
        ```  

   g. Connect "Wait For Response1" output to "Respond: Restaurant Details (VAPI)".

4. **Add Sticky Notes for documentation:**

   a. For the booking/order block, add a sticky note describing the purpose and flow.

   b. For the info provider block, add a similar sticky note.

5. **Set workflow settings:**

   - Timezone: Asia/Kolkata  
   - Execution order: sequential (v1)  
   - Caller policy: workflowsFromSameOwner  

6. **Configure credentials:**

   - PostgreSQL: Provide connection details to your PostgreSQL instance with access to booking/order and restaurant info tables.  
   - No explicit external credentials for VAPI; interaction handled via webhook calls.

7. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| The workflow relies on the VAPI voice assistant platform to POST JSON payloads containing `toolCalls` with unique IDs for conversational context.                 | VAPI integration context                                |
| The use of Wait nodes without timeout implies external processes or manual triggers must resume execution; otherwise, workflow may stall indefinitely.             | Workflow design caution                                 |
| The PostgreSQL table names are shown as `"id"` in the nodes but should be replaced with actual table names like `"bookings"` or `"restaurant_info"` before use.   | Database schema setup                                   |
| Sticky notes provide essential contextual documentation embedded directly inside the workflow UI, improving maintainability and comprehension.                    | Workflow documentation best practice                    |
| Ensure PostgreSQL credentials have appropriate permissions for SELECT and UPSERT operations on relevant tables.                                                   | Database security and permissions                        |

---

**Disclaimer:** The text provided is exclusively generated based on an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected content. All data manipulated are legal and public.