Sync Shopify orders with your Zendesk contacts

https://n8nworkflows.xyz/workflows/sync-shopify-orders-with-your-zendesk-contacts-1808


# Sync Shopify orders with your Zendesk contacts

### 1. Workflow Overview

This workflow automates the synchronization of Shopify customer order data with Zendesk contacts. It ensures that whenever a Shopify customer's data is updated, the corresponding Zendesk contact record is created or updated with the latest email, phone number, and order information.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Detect Shopify customer updates via a webhook trigger.
- **1.2 Zendesk Contact Retrieval:** Search for existing Zendesk contacts by email.
- **1.3 Data Extraction and Merging:** Extract essential Zendesk contact identifiers and merge Shopify and Zendesk data.
- **1.4 Conditional Logic:** Determine if the contact exists in Zendesk and if contact data needs updating.
- **1.5 Contact Creation or Update:** Create a new Zendesk contact if none exists or update the existing contact if data has changed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by listening to Shopify customer update events. It triggers whenever a customer's data is modified in Shopify.

- **Nodes Involved:**  
  - On customer updated (Shopify Trigger)

- **Node Details:**  

  - **On customer updated**  
    - Type: Shopify Trigger  
    - Role: Starts workflow on Shopify customer update event (`customers/update` topic).  
    - Configuration: Uses Shopify API credentials linked to the account.  
    - Input: Webhook event from Shopify when a customer is updated.  
    - Output: Customer data JSON containing email, phone, first name, last name, etc.  
    - Edge Cases:  
      - Webhook failure or misconfiguration leading to missed events.  
      - Shopify API credential expiration or invalid.  

---

#### 2.2 Zendesk Contact Retrieval

- **Overview:**  
  Searches Zendesk for a contact matching the Shopify customer's email to determine if the contact already exists.

- **Nodes Involved:**  
  - Search contact by email address (Zendesk)

- **Node Details:**  

  - **Search contact by email address**  
    - Type: Zendesk node (Search user)  
    - Role: Queries Zendesk users filtered by the email address received from Shopify.  
    - Configuration: Limits results to 1 user; uses Zendesk API credentials.  
    - Input: Email from Shopify customer data.  
    - Output: Zendesk user data if found (including id, email, phone).  
    - Edge Cases:  
      - API rate limits or authentication errors with Zendesk.  
      - No matching Zendesk user found returns empty results.  
      - Expression failure if email is missing or malformed.  

---

#### 2.3 Data Extraction and Merging

- **Overview:**  
  This block extracts minimal necessary fields (UserId, email, phone) from the Zendesk search result and merges it with the Shopify customer data based on email.

- **Nodes Involved:**  
  - Keep only UserId and email (Set)  
  - Add Zendesk contact Id to Shopify data (Merge)

- **Node Details:**  

  - **Keep only UserId and email**  
    - Type: Set node  
    - Role: Filters Zendesk user JSON to keep only `id` (as `ZendeskUserId`), `email` (as `ZendeskEmail`), and `phone` (as `ZendeskPhone`).  
    - Configuration: Keeps only these fields, discarding others.  
    - Input: Zendesk user search result.  
    - Output: Clean JSON with only the above fields.  
    - Edge Cases:  
      - If Zendesk search returned no results, fields would be empty, affecting downstream conditions.  

  - **Add Zendesk contact Id to Shopify data**  
    - Type: Merge node (mergeByKey)  
    - Role: Joins Shopify customer data and Zendesk contact data by matching `email` (Shopify) and `ZendeskEmail` (Zendesk).  
    - Configuration: Merge mode is "mergeByKey" on email fields.  
    - Input: Shopify data (from trigger) and Zendesk extracted data (from Set node).  
    - Output: Combined data containing Shopify and Zendesk fields.  
    - Edge Cases:  
      - Emails not matching exactly due to case or formatting differences could cause merge failures.  

---

#### 2.4 Conditional Logic

- **Overview:**  
  This block evaluates if the Zendesk contact exists and whether the contact's phone data has changed, guiding the workflow to update, create, or skip.

- **Nodes Involved:**  
  - User exists in Zendesk (If)  
  - Contact data is modified (If)  
  - NoOp (No operation placeholder)

- **Node Details:**  

  - **User exists in Zendesk**  
    - Type: If node  
    - Role: Checks if `ZendeskUserId` is not empty, i.e., user exists in Zendesk.  
    - Input: Merged data with `ZendeskUserId`.  
    - Output:  
      - True branch: User exists.  
      - False branch: User does not exist.  
    - Edge Cases:  
      - False negatives if `ZendeskUserId` field missing or empty due to upstream errors.  

  - **Contact data is modified**  
    - Type: If node  
    - Role: Compares Shopify phone number with Zendesk phone number to detect changes.  
    - Input: Merged data containing `phone` (Shopify) and `ZendeskPhone` (Zendesk).  
    - Output:  
      - True branch: Phone numbers differ.  
      - False branch: Phone numbers are equal; no update needed.  
    - Edge Cases:  
      - Missing phone fields could cause false positives or negatives.  
      - Different formatting may cause inequality even if numbers are the same.  

  - **NoOp**  
    - Type: No Operation node  
    - Role: Terminates path when no update is necessary.  
    - Input: False branch of "Contact data is modified".  
    - Output: None.  
    - Edge Cases: None.  

---

#### 2.5 Contact Creation or Update

- **Overview:**  
  Depending on the condition results, this block either updates the existing Zendesk contact with the new phone data or creates a new Zendesk contact with the Shopify customer data.

- **Nodes Involved:**  
  - Update contact in Zendesk (Zendesk)  
  - Create contact in Zendesk (Zendesk)

- **Node Details:**  

  - **Update contact in Zendesk**  
    - Type: Zendesk node (Update user)  
    - Role: Updates the existing Zendesk user's phone number using their `ZendeskUserId`.  
    - Configuration:  
      - `id` set to `ZendeskUserId` from merged data.  
      - Updates the `phone` field with Shopify `phone` value or 0 if missing.  
    - Input: True branch of "Contact data is modified".  
    - Output: Updated Zendesk user data.  
    - Edge Cases:  
      - API failures or invalid user ID.  
      - Phone field missing or invalid format might cause update errors.  

  - **Create contact in Zendesk**  
    - Type: Zendesk node (Create user)  
    - Role: Creates a new Zendesk contact with Shopify customer details (name, email, phone).  
    - Configuration:  
      - Name constructed from Shopify `first_name` and `last_name`.  
      - Email and phone fields from Shopify data; phone defaults to a blank space if missing.  
    - Input: False branch of "User exists in Zendesk".  
    - Output: Newly created Zendesk user data.  
    - Edge Cases:  
      - API errors or duplicate records if contact exists but was not found.  
      - Missing required fields (email) causing creation failure.  

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                             | Input Node(s)               | Output Node(s)              | Sticky Note                    |
|-----------------------------|------------------------|---------------------------------------------|-----------------------------|-----------------------------|-------------------------------|
| On customer updated          | Shopify Trigger        | Starts workflow on Shopify customer update | -                           | Add Zendesk contact Id to Shopify data |                               |
| Search contact by email adress | Zendesk                | Searches Zendesk user by Shopify email      | Add Zendesk contact Id to Shopify data | Keep only UserId and email    |                               |
| Keep only UserId and email  | Set                    | Extracts Zendesk UserId, email, phone       | Search contact by email adress | Add Zendesk contact Id to Shopify data |                               |
| Add Zendesk contact Id to Shopify data | Merge (mergeByKey)    | Merges Shopify and Zendesk data by email    | On customer updated, Keep only UserId and email | User exists in Zendesk         |                               |
| User exists in Zendesk      | If                     | Checks if Zendesk user exists                | Add Zendesk contact Id to Shopify data | Contact data is modified, Create contact in Zendesk |                               |
| Contact data is modified    | If                     | Checks if phone data differs between Shopify and Zendesk | User exists in Zendesk       | Update contact in Zendesk, NoOp |                               |
| Update contact in Zendesk   | Zendesk                | Updates Zendesk contact phone number         | Contact data is modified     | -                           |                               |
| Create contact in Zendesk   | Zendesk                | Creates new Zendesk contact                   | User exists in Zendesk       | -                           |                               |
| NoOp                       | No Operation           | Ends path if no update needed                 | Contact data is modified     | -                           |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger Node**  
   - Type: Shopify Trigger  
   - Parameters: Topic = `customers/update`  
   - Credentials: Link Shopify API credentials.  
   - Position: Start of workflow.

2. **Create Zendesk Node to Search Contact by Email**  
   - Type: Zendesk  
   - Operation: Search user  
   - Parameters:  
     - Limit = 1  
     - Filters: Query = `{{$json["email"]}}`  
   - Credentials: Link Zendesk API credentials.  
   - Connect input from Shopify Trigger.

3. **Create Set Node to Keep Only Zendesk UserId, Email, Phone**  
   - Type: Set  
   - Parameters:  
     - Keep only set = true  
     - Values to set:  
       - Number field `ZendeskUserId` = `{{$json["id"]}}`  
       - String field `ZendeskEmail` = `{{$json["email"]}}`  
       - String field `ZendeskPhone` = `{{$json["phone"]}}`  
   - Connect input from Zendesk Search node.

4. **Create Merge Node to Combine Shopify and Zendesk Data**  
   - Type: Merge  
   - Mode: mergeByKey  
   - Property Name 1 = `email` (from Shopify)  
   - Property Name 2 = `ZendeskEmail` (from Set node)  
   - Connect two inputs:  
     - Input 1 from Shopify Trigger  
     - Input 2 from Set node

5. **Create If Node to Check if User Exists in Zendesk**  
   - Type: If  
   - Condition: Number field `ZendeskUserId` is not empty.  
   - Connect input from Merge node.

6. **Create If Node to Check if Contact Data is Modified**  
   - Type: If  
   - Condition: String field `phone` (Shopify) is not equal to `ZendeskPhone` (Zendesk).  
   - Connect input from True branch of User Exists node.

7. **Create Zendesk Node to Update Contact**  
   - Type: Zendesk  
   - Operation: Update user  
   - Parameters:  
     - ID = `{{$json["ZendeskUserId"]}}`  
     - Update fields: Phone = `{{$json["phone"] ?? 0}}`  
   - Credentials: Zendesk API credentials.  
   - Connect input from True branch of Contact Data is Modified node.

8. **Create No Operation Node (NoOp)**  
   - Type: NoOp  
   - Connect input from False branch of Contact Data is Modified node.

9. **Create Zendesk Node to Create Contact**  
   - Type: Zendesk  
   - Operation: Create user  
   - Parameters:  
     - Name = `{{$json["first_name"]}} {{$json["last_name"]}}`  
     - Additional fields:  
       - Email = `{{$json["email"]}}`  
       - Phone = `{{$json["phone"] ?? ' '}}`  
   - Credentials: Zendesk API credentials.  
   - Connect input from False branch of User Exists node.

10. **Verify all connections follow the described flow:**  
    - Shopify Trigger → Merge & Zendesk Search → Set → Merge → If(User exists) → If (Contact modified) → Update / NoOp  
    - User does not exist → Create contact  

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Prerequisites: Shopify and Zendesk accounts with valid credentials configured in n8n.                | Shopify credentials: https://docs.n8n.io/integrations/builtin/credentials/shopify/              |
|                                                                                                      | Zendesk credentials: https://docs.n8n.io/integrations/builtin/credentials/zendesk/              |
| This workflow uses the Shopify webhook for customer updates to maintain real-time sync with Zendesk. |                                                                                               |

---

This documentation fully covers the workflow structure, node-by-node functionality, and provides explicit instructions for rebuilding or modifying the workflow in n8n. It also anticipates potential failure points such as authentication issues, missing data, and API limitations.