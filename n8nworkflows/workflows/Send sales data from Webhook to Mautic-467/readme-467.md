Send sales data from Webhook to Mautic

https://n8nworkflows.xyz/workflows/send-sales-data-from-webhook-to-mautic-467


# Send sales data from Webhook to Mautic

### 1. Workflow Overview

This workflow automates the synchronization and management of user data between a Teachable webhook (triggered by various student-related events) and a Mautic marketing automation system. Its main purpose is to:

- Receive real-time event data from Teachable about students (new users, updates, sales).
- Search for the corresponding user in Mautic by email.
- Create or update the user’s contact record in Mautic.
- Manage subscription status tags (add or remove the `#unsubscribe` tag based on marketing preferences).
- Tag users with the course name related to their sale.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives and preprocesses incoming webhook requests from Teachable.
- **1.2 User Identification:** Searches for the user in Mautic and prepares user data.
- **1.3 User Data Management:** Creates or updates user contact records in Mautic and handles subscription tags.
- **1.4 Sale Tagging:** Tags users with course-specific tags upon sale events.
- **1.5 Subscription Tag Handling:** Adds or removes the `#unsubscribe` tag depending on user's subscription preferences.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives the webhook request from Teachable, extracts the relevant student data and event type, and routes processing depending on the event category.

**Nodes Involved:**  
- Webhook  
- Set Webhook Request  
- Switch Webhook Types  
- Split Full Name

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Entry point receiving HTTP POST requests at path `/PuHq2RQsmc3HXB/hook`.  
  - Config: Accepts POST, rawBody disabled (automatic JSON parsing).  
  - Output: Raw webhook body forwarded downstream.  
  - Potential failures: Timeout, invalid request, missing body.

- **Set Webhook Request**  
  - Type: Set  
  - Role: Extracts and stores `student` object and `type` from incoming webhook body for simplified access.  
  - Key Expressions:  
    - `student = {{$node["Webhook"].json["body"]["object"]}}`  
    - `type = {{$node["Webhook"].json["body"]["type"]}}`  
  - Input: Webhook node output  
  - Output: Object with `student` and `type` fields.  
  - Corner cases: Missing fields in payload.

- **Switch Webhook Types**  
  - Type: Switch  
  - Role: Routes the workflow based on webhook event type prefix: events containing "User." or "Sale."  
  - Config: Checks if `type` string contains "User." or "Sale."  
  - Input: Set Webhook Request output  
  - Output: Two branches to handle user-related or sale-related logic.  
  - Edge cases: Unhandled event types are ignored.

- **Split Full Name**  
  - Type: Function  
  - Role: Splits full student name into `firstName` and `lastName` by dividing on spaces.  
  - Code: Uses string split and slice operations to separate the last word as lastName and the rest as firstName.  
  - Input: After "User." branch from Switch Webhook Types  
  - Output: Modified student object with first and last names separated.  
  - Edge cases: Empty or single-word names default to empty firstName or lastName.

---

#### 1.2 User Identification

**Overview:**  
Searches Mautic for the user by email extracted from the webhook data and prepares a user identifier for further operations.

**Nodes Involved:**  
- Find User  
- If not found return -1  
- Set userFound  
- @MAIN STUDENT DATA (Merge node)

**Node Details:**

- **Find User**  
  - Type: Mautic  
  - Role: Searches Mautic contacts for one matching the email from student data.  
  - Config: Search option by email, limit 1, OAuth2 authentication.  
  - Input: From Switch Webhook Types ("User." branch)  
  - Output: User data if found, else empty.  
  - Failures: API auth failure, network timeout, missing email.

- **If not found return -1**  
  - Type: Function  
  - Role: Normalizes missing user by setting `id` to `-1` if user not found.  
  - Input: Find User output  
  - Output: JSON with `id` either actual user ID or `-1`.  
  - Edge cases: Empty responses handled gracefully.

- **Set userFound**  
  - Type: Set  
  - Role: Stores the user ID (`userFound`) for downstream usage.  
  - Input: If not found return -1 output  
  - Output: JSON with single key `userFound`.  
  - Notes: Keeps only `userFound` field to avoid clutter.

- **@MAIN STUDENT DATA**  
  - Type: Merge  
  - Role: Combines the student data and userFound ID by inner join on index to unify data in one object.  
  - Input: Outputs from Set userFound and Split Full Name nodes  
  - Output: Unified JSON containing `student`, `type`, and `userFound` fields.  
  - Edge cases: Both inputs must have matching index; otherwise, merge fails.

---

#### 1.3 User Data Management

**Overview:**  
Handles creation of new Mautic contacts or updates existing ones, including email, first name, and last name. Also manages subscription tags based on user marketing preferences.

**Nodes Involved:**  
- IF NOT userFound  
- Mautic (Create User)  
- Switch User.type  
- Update User  
- IF unsubscribe_from_marketing_emails  
- Unsubscribe User  
- Remove unsubscribe

**Node Details:**

- **IF NOT userFound**  
  - Type: If  
  - Role: Checks if `userFound` equals `-1` (user not found) to decide whether to create a new contact or update.  
  - Input: @MAIN STUDENT DATA  
  - Output: Two branches:  
    - True: Create new user  
    - False: Update existing user  
  - Edge cases: String comparison with regex to detect `-1`.

- **Mautic (Create User)**  
  - Type: Mautic  
  - Role: Creates a new contact in Mautic with student email, first name, last name.  
  - Input: IF NOT userFound true branch  
  - Config: OAuth2 authentication, fields from `student` object.  
  - Output: Newly created user data.  
  - Failures: Duplicate email, API errors.

- **Switch User.type**  
  - Type: Switch  
  - Role: Routes update logic based on user event type (e.g., updated, unsubscribe, subscribe).  
  - Input: IF NOT userFound false branch (existing user)  
  - Config Rules:  
    - "User.updated"  
    - "User.unsubscribe_from_marketing_emails"  
    - "User.subscribe_to_marketing_emails"  
  - Outputs: Three branches for different update scenarios.

- **Update User**  
  - Type: Mautic  
  - Role: Updates existing Mautic contact fields with latest student data.  
  - Input: Switch User.type "User.updated" branch  
  - Config: Contact ID from `userFound`, updated email and names, OAuth2 authentication.  
  - Output: Update response.  
  - Failures: Invalid contact ID, auth errors.

- **IF unsubscribe_from_marketing_emails**  
  - Type: If  
  - Role: Checks the boolean field `unsubscribe_from_marketing_emails` in student data to decide whether to unsubscribe or resubscribe.  
  - Input: Update User output  
  - Output: Two branches, true (unsubscribe) or false (remove unsubscribe tag).

- **Unsubscribe User**  
  - Type: Mautic  
  - Role: Adds the `#unsubscribe` tag to a user to mark them as unsubscribed.  
  - Input: IF unsubscribe_from_marketing_emails true branch or Switch User.type "User.unsubscribe_from_marketing_emails" branch  
  - Config: Updates contact tags with `#unsubscribe`.  
  - Failures: Tag addition errors.

- **Remove unsubscribe**  
  - Type: Mautic  
  - Role: Removes the `#unsubscribe` tag from a user to mark them as subscribed again.  
  - Input: IF unsubscribe_from_marketing_emails false branch or Switch User.type "User.subscribe_to_marketing_emails" branch  
  - Config: Removes `#unsubscribe` tag.  
  - Failures: Tag removal errors.

---

#### 1.4 Sale Tagging

**Overview:**  
Handles tagging the user in Mautic with the name of the course after a sale event.

**Nodes Involved:**  
- Find User To Tag Sale  
- Tag User

**Node Details:**

- **Find User To Tag Sale**  
  - Type: Mautic  
  - Role: Searches Mautic for the user by email in sale events.  
  - Config: Search by email from `student.user.email` field, OAuth2 authentication, limit 1.  
  - Input: Switch Webhook Types "Sale." branch  
  - Output: User contact data.

- **Tag User**  
  - Type: Mautic  
  - Role: Updates user tags by adding the course name from the webhook student data.  
  - Input: Find User To Tag Sale output  
  - Config: Updates `tags` field to the course name (e.g., `student.course.name`).  
  - Edge cases: Missing course name or email.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                               | Input Node(s)                 | Output Node(s)                       | Sticky Note                                |
|---------------------------|--------------------|-----------------------------------------------|------------------------------|------------------------------------|--------------------------------------------|
| Webhook                   | Webhook            | Receives POST request from Teachable webhook | None                         | Set Webhook Request                 |                                            |
| Set Webhook Request        | Set                | Extracts student and type from webhook body  | Webhook                      | Switch Webhook Types                |                                            |
| Switch Webhook Types       | Switch             | Routes processing by event type prefix        | Set Webhook Request           | Find User, Split Full Name, Find User To Tag Sale |                                            |
| Find User                 | Mautic             | Searches user by email in Mautic               | Switch Webhook Types          | If not found return -1              |                                            |
| If not found return -1     | Function           | Sets id to -1 if user not found                 | Find User                    | Set userFound                      |                                            |
| Set userFound              | Set                | Stores userFound ID                             | If not found return -1        | @MAIN STUDENT DATA                 |                                            |
| Split Full Name            | Function           | Splits full student name into first and last  | Switch Webhook Types          | @MAIN STUDENT DATA                 |                                            |
| @MAIN STUDENT DATA         | Merge              | Merges student data and userFound ID           | Set userFound, Split Full Name| IF NOT userFound                  |                                            |
| IF NOT userFound           | If                 | Checks if userFound is -1                        | @MAIN STUDENT DATA           | Mautic (Create User), Switch User.type |                                            |
| Mautic                    | Mautic             | Creates new Mautic contact                      | IF NOT userFound (true)       | None                             |                                            |
| Switch User.type           | Switch             | Routes update logic based on user event type   | IF NOT userFound (false)      | Update User, Unsubscribe User, Remove unsubscribe |                                            |
| Update User               | Mautic             | Updates user contact info                        | Switch User.type ("User.updated") | IF unsubscribe_from_marketing_emails |                                            |
| IF unsubscribe_from_marketing_emails | If                 | Checks unsubscribe status                       | Update User                  | Unsubscribe User, Remove unsubscribe |                                            |
| Unsubscribe User          | Mautic             | Adds #unsubscribe tag to user                    | IF unsubscribe_from_marketing_emails (true), Switch User.type ("User.unsubscribe_from_marketing_emails") | None |                                            |
| Remove unsubscribe        | Mautic             | Removes #unsubscribe tag from user               | IF unsubscribe_from_marketing_emails (false), Switch User.type ("User.subscribe_to_marketing_emails") | None |                                            |
| Find User To Tag Sale      | Mautic             | Finds user for sale tagging                      | Switch Webhook Types ("Sale.")| Tag User                         |                                            |
| Tag User                  | Mautic             | Tags user with course name                        | Find User To Tag Sale         | None                             |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `PuHq2RQsmc3HXB/hook`  
   - Options: Raw Body disabled (parse JSON automatically)

2. **Create Set Node "Set Webhook Request"**  
   - Extract `student` from `$node["Webhook"].json["body"]["object"]`  
   - Extract `type` from `$node["Webhook"].json["body"]["type"]`  
   - Keep only these two fields.

3. **Create Switch Node "Switch Webhook Types"**  
   - Input: `{{$node["Set Webhook Request"].json["type"]}}`  
   - Rules:  
     - Output 0: If `type` contains `"User."`  
     - Output 1: If `type` contains `"Sale."`

4. **For "User." branch:**

   a. **Create Mautic Node "Find User"**  
      - Operation: Get All Contacts  
      - Limit: 1  
      - Search: Email = `{{$node["Set Webhook Request"].json["student"]["email"]}}`  
      - Authentication: OAuth2 with Mautic credentials

   b. **Create Function Node "If not found return -1"**  
      - Code:  
        ```
        items[0].json.id = items[0].json.id || -1;
        return items;
        ```
   
   c. **Create Set Node "Set userFound"**  
      - Set `userFound` to `{{$node["If not found return -1"].json["id"]}}`  
      - Keep only `userFound`.

   d. **Create Function Node "Split Full Name"**  
      - Code:  
        ```
        const student = items[0].json.student;
        student.firstName = student.name ? student.name.split(' ').slice(0, -1).join(' ') : '';
        student.lastName = student.name ? student.name.split(' ').slice(-1).join(' ') : '';
        items[0].json.student = student;
        return items;
        ```

   e. **Create Merge Node "@MAIN STUDENT DATA"**  
      - Mode: Inner Join by Index  
      - Input 1: Output of "Set userFound"  
      - Input 2: Output of "Split Full Name"

   f. **Create If Node "IF NOT userFound"**  
      - Condition: Check if `{{$node["@MAIN STUDENT DATA"].json["userFound"]}}` matches regex `-1` (string equals "-1")

   g. **Create Mautic Node "Mautic" (Create User)**  
      - Operation: Create Contact  
      - Email: `{{$node["@MAIN STUDENT DATA"].json["student"]["email"]}}`  
      - First Name: `{{$node["@MAIN STUDENT DATA"].json["student"]["firstName"]}}`  
      - Last Name: `{{$node["@MAIN STUDENT DATA"].json["student"]["lastName"]}}`  
      - Authentication: OAuth2

   h. **Create Switch Node "Switch User.type"**  
      - Input: `{{$node["@MAIN STUDENT DATA"].json["type"]}}`  
      - Rules:  
        - Output 0: Equals "User.updated"  
        - Output 1: Equals "User.unsubscribe_from_marketing_emails"  
        - Output 2: Equals "User.subscribe_to_marketing_emails"

   i. **Create Mautic Node "Update User"**  
      - Operation: Update Contact  
      - Contact ID: `{{$node["@MAIN STUDENT DATA"].json["userFound"]}}`  
      - Update email, first and last name with student data  
      - Authentication: OAuth2

   j. **Create If Node "IF unsubscribe_from_marketing_emails"**  
      - Condition: Boolean check if `{{$node["@MAIN STUDENT DATA"].json["student"]["unsubscribe_from_marketing_emails"]}}` is true

   k. **Create Mautic Node "Unsubscribe User"**  
      - Operation: Update Contact  
      - Contact ID: `{{$node["@MAIN STUDENT DATA"].json["userFound"]}}`  
      - Add Tag: `#unsubscribe`  
      - Authentication: OAuth2

   l. **Create Mautic Node "Remove unsubscribe"**  
      - Operation: Update Contact  
      - Contact ID: `{{$node["@MAIN STUDENT DATA"].json["userFound"]}}`  
      - Remove Tag: `#unsubscribe`  
      - Authentication: OAuth2

5. **For "Sale." branch:**

   a. **Create Mautic Node "Find User To Tag Sale"**  
      - Operation: Get All Contacts  
      - Limit: 1  
      - Search: Email = `{{$node["Set Webhook Request"].json["student"]["user"]["email"]}}`  
      - Authentication: OAuth2

   b. **Create Mautic Node "Tag User"**  
      - Operation: Update Contact  
      - Contact ID: `{{$node["Find User To Tag Sale"].json["id"]}}`  
      - Add Tag: `{{$node["Set Webhook Request"].json["student"]["course"]["name"]}}`  
      - Authentication: OAuth2

6. **Connect nodes following the logical flow as described in the Overview and Block-by-Block sections.**

7. **Set Credentials:**  
   - Configure the Mautic OAuth2 credentials once and assign them to all Mautic nodes.  
   - No additional credentials needed for the webhook node.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                       |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| This workflow tags users with the course name dynamically, enabling targeted marketing based on purchased courses. | Core feature to link sales events to marketing tags. |
| OAuth2 authentication for Mautic nodes requires prior setup of OAuth2 credentials in n8n credential manager.   | Mautic OAuth2 credential configuration.              |
| The `#unsubscribe` tag is used as a marker for marketing email subscription status in Mautic.                   | Subscription management via tags in Mautic.           |
| The workflow assumes the webhook payload follows Teachable's webhook schema, especially `student` and `type`.   | Ensure webhook source compatibility.                  |

---

This document provides a thorough understanding and stepwise guidance to reproduce or extend the described n8n workflow, ensuring robust integration between Teachable events and Mautic marketing automation.