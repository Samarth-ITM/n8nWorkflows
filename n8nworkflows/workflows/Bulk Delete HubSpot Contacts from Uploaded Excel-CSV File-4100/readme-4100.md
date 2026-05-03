Bulk Delete HubSpot Contacts from Uploaded Excel/CSV File

https://n8nworkflows.xyz/workflows/bulk-delete-hubspot-contacts-from-uploaded-excel-csv-file-4100


# Bulk Delete HubSpot Contacts from Uploaded Excel/CSV File

---

### 1. Workflow Overview

This workflow automates the bulk deletion of HubSpot contacts based on an uploaded Excel or CSV file containing contact emails. It is designed for users who want to efficiently remove multiple contacts in HubSpot by simply uploading a file with their email addresses.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the uploaded file via a webhook and extracts data from it.
- **1.2 Data Parsing:** Parses the extracted file data to isolate email addresses.
- **1.3 Batch Processing Loop:** Iterates over each parsed email in batches to manage load and API rate limits.
- **1.4 Contact Search and Deletion:** For each email, searches HubSpot for the contact and deletes it if found.
- **1.5 Error Handling and Fallbacks:** Gracefully continues processing on errors like missing contacts or API issues.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives the uploaded Excel/CSV file through a webhook endpoint and extracts its contents.

**Nodes Involved:**  
- Webhook  
- Extract File Data  
- Sticky Note (instructional)

**Node Details:**

- **Webhook**  
  - *Type:* Webhook node, HTTP POST method  
  - *Role:* Entry point to receive the uploaded file from an external app or API client.  
  - *Configuration:*  
    - Path set to a unique webhook ID.  
    - HTTP Method: POST.  
  - *Inputs:* External HTTP request with file upload.  
  - *Outputs:* File data forwarded to the next node.  
  - *Edge Cases:*  
    - Invalid or missing file upload.  
    - Unauthorized external calls (no auth configured).  
  - *Sticky Note:* Instructs user to copy the webhook URL for use in their app or Postman.

- **Extract File Data**  
  - *Type:* ExtractFromFile node  
  - *Role:* Extracts tabular data from the uploaded Excel (.xlsx) or CSV file.  
  - *Configuration:*  
    - Operation set to "xlsx" (can be changed for CSV or other formats).  
  - *Inputs:* Binary file data from Webhook.  
  - *Outputs:* JSON array of rows with file content.  
  - *Edge Cases:*  
    - Unsupported file format.  
    - Corrupted or malformed files.  
  - *Sticky Note:* Advises user how to change file format if needed.

---

#### 1.2 Data Parsing

**Overview:**  
Transforms the extracted raw file data into structured JSON focusing on the email addresses to be processed.

**Nodes Involved:**  
- Parse Data  
- Sticky Note (field name instruction)

**Node Details:**

- **Parse Data**  
  - *Type:* Set node  
  - *Role:* Maps the extracted file data field (e.g., 'emails') to a new JSON property named "email".  
  - *Configuration:*  
    - Assignment: Set `email` field with value from the original data header `emails`.  
  - *Inputs:* JSON rows from Extract File Data.  
  - *Outputs:* JSON with normalized email field for downstream processing.  
  - *Edge Cases:*  
    - Field name mismatch if uploaded file header differs.  
    - Empty or null email values.  
  - *Sticky Note:* Instructs user to change the data header name here if the file uses a different column name.

---

#### 1.3 Batch Processing Loop

**Overview:**  
Splits the list of emails into batches for sequential processing to handle large datasets and API rate limits.

**Nodes Involved:**  
- Loop Over Items

**Node Details:**

- **Loop Over Items**  
  - *Type:* SplitInBatches node  
  - *Role:* Processes contacts in manageable batches (default batch size not explicitly set, uses default).  
  - *Configuration:*  
    - Reset option disabled (`reset = false`), meaning it continues from previous batch if restarted.  
    - On error set to continue processing next batches.  
  - *Inputs:* JSON array of email objects from Parse Data.  
  - *Outputs:* One item or batch at a time forwarded for contact search.  
  - *Edge Cases:*  
    - Batch size too large causing timeout or API throttling.  
    - Errors within batch items handled gracefully.  

---

#### 1.4 Contact Search and Deletion

**Overview:**  
For each email, this block searches HubSpot for the contact and deletes it if found; otherwise, it skips to the next.

**Nodes Involved:**  
- Search Contact  
- If contact exists (conditional)  
- Delete Contact  
- Replace Me (NoOp fallback)

**Node Details:**

- **Search Contact**  
  - *Type:* HubSpot node (search operation)  
  - *Role:* Queries HubSpot contacts filtered by the email address to identify the contact ID.  
  - *Configuration:*  
    - Operation: Search.  
    - Limit: 1 result to optimize performance.  
    - Authentication: HubSpot App Token.  
    - Filter: Email equals current item's email field.  
  - *Inputs:* Single email JSON from Loop Over Items.  
  - *Outputs:* Contact data if found, empty if not.  
  - *Error Handling:* Continues on errors to avoid workflow stop.  
  - *Edge Cases:*  
    - Email not found returns empty result.  
    - API authentication failure or rate limiting.  

- **If contact exists**  
  - *Type:* If node (conditional)  
  - *Role:* Checks if the search returned a contact ID.  
  - *Configuration:*  
    - Condition: Checks if `$json.id` exists and is non-empty.  
  - *Inputs:* Output from Search Contact.  
  - *Outputs:*  
    - True branch: Contact ID exists.  
    - False branch: Contact not found.  
  - *Edge Cases:* False negatives if API response format changes.

- **Delete Contact**  
  - *Type:* HubSpot node (delete operation)  
  - *Role:* Deletes the contact in HubSpot by ID.  
  - *Configuration:*  
    - Operation: Delete.  
    - Contact ID: From `$json.id` of found contact.  
    - Authentication: HubSpot App Token.  
  - *Inputs:* True branch from If node.  
  - *Outputs:* Result of deletion operation.  
  - *Edge Cases:*  
    - Deletion fails due to permissions or API errors.  
    - Contact already deleted externally.  

- **Replace Me**  
  - *Type:* NoOp node  
  - *Role:* Acts as a passthrough for items where contact was not found or after deletion to continue the workflow without error.  
  - *Inputs:* False branch from If node and output from Delete Contact.  
  - *Outputs:* Feeds back into Loop Over Items to continue processing.  
  - *Edge Cases:* None, just a placeholder to maintain flow.

---

#### 1.5 Error Handling and Flow Control

**Overview:**  
Ensures that errors during batch processing or API calls do not halt the entire workflow, allowing it to continue processing remaining contacts.

**Nodes Involved:**  
- Loop Over Items (configured to continue on error)  
- Search Contact (configured to continue on error)  
- If contact exists (conditional check)  
- Replace Me (fallback node)

**Node Details:**

- *Error continuation* is configured in nodes like Loop Over Items and Search Contact to avoid failures stopping the workflow.  
- This setup allows partial successes and logs errors without full interruption.

---

### 3. Summary Table

| Node Name          | Node Type            | Functional Role                         | Input Node(s)       | Output Node(s)         | Sticky Note                                                                 |
|--------------------|----------------------|---------------------------------------|---------------------|------------------------|-----------------------------------------------------------------------------|
| Webhook            | Webhook              | Receive uploaded file via HTTP POST   | -                   | Extract File Data      | Copy the webhook URL and use it in your app or API client (like Postman).   |
| Extract File Data   | ExtractFromFile      | Extract data from uploaded Excel/CSV | Webhook             | Parse Data             | Change file format if needed in Extract File Data Node.                     |
| Parse Data         | Set                  | Map file data header to email field   | Extract File Data    | Loop Over Items        | Change Data Header name if needed in Parse Data Node. current -> emails     |
| Loop Over Items    | SplitInBatches        | Process emails in batches              | Parse Data           | Search Contact (on output), empty branch continues |                                                                             |
| Search Contact     | HubSpot (search)      | Search HubSpot contact by email       | Loop Over Items      | If contact exists      |                                                                             |
| If contact exists  | If                    | Check if contact found in HubSpot     | Search Contact       | Delete Contact (true), Replace Me (false) |                                                                             |
| Delete Contact     | HubSpot (delete)      | Delete HubSpot contact by ID           | If contact exists    | Replace Me             |                                                                             |
| Replace Me         | NoOp                  | Pass through non-found or deleted items | Delete Contact, If contact exists (false) | Loop Over Items         |                                                                             |
| Sticky Note        | Sticky Note           | Instructional note                    | -                   | -                      | Copy the webhook URL and use it in your app or API client (like Postman).   |
| Sticky Note1       | Sticky Note           | Instructional note                    | -                   | -                      | Change file format if needed in Extract File Data Node.                     |
| Sticky Note2       | Sticky Note           | Instructional note                    | -                   | -                      | Change Data Header name if needed in Parse Data Node. current -> emails     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use a unique string (e.g., "abf99fbe-fa03-4f55-b978-db7cba42a565")  
   - This node serves as the entry point for file upload.

2. **Create Extract File Data Node**  
   - Type: ExtractFromFile  
   - Operation: Set to "xlsx" for Excel files (change to "csv" if needed)  
   - Connect Webhook output to this node.

3. **Create Parse Data Node**  
   - Type: Set  
   - Add assignment: create a new field `email` with value from the extracted file's email column (default field name: "emails")  
   - Connect Extract File Data output to this node.

4. **Create Loop Over Items Node**  
   - Type: SplitInBatches  
   - Leave batch size default or set as needed for rate limits  
   - Set "Reset" option to false  
   - Connect Parse Data output to this node.

5. **Create Search Contact Node**  
   - Type: HubSpot node  
   - Operation: Search  
   - Limit: 1  
   - Authentication: HubSpot App Token (set up credentials in n8n before use)  
   - Filter: Filter by property `email` equals current item’s `email` field (`={{ $json.email }}`)  
   - Connect Loop Over Items output (second output) to this node.  
   - Set error handling to "Continue On Fail" to avoid workflow stop on errors.

6. **Create If contact exists Node**  
   - Type: If  
   - Condition: Check if `$json.id` exists (string operation "exists", true)  
   - Connect Search Contact output to this node.

7. **Create Delete Contact Node**  
   - Type: HubSpot node  
   - Operation: Delete  
   - Contact ID: Use the found contact ID (`={{ $json.id }}`)  
   - Authentication: HubSpot App Token  
   - Connect "true" output of If node to this node.

8. **Create Replace Me Node**  
   - Type: NoOp (No Operation)  
   - Connect "false" output of If node to this node (for contacts not found)  
   - Also connect Delete Contact output to this node (to continue after deletion)

9. **Connect Replace Me output back to Loop Over Items**  
   - This closes the processing loop and ensures the workflow continues with next batch/item.

10. **Add Sticky Notes for User Guidance**  
    - Add notes near Webhook with instructions to copy the webhook URL.  
    - Add notes near Extract File Data to remind changing file format if needed.  
    - Add notes near Parse Data to remind changing the field name if the uploaded file has different headers.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow requires a valid HubSpot App Token credential configured in n8n for authentication.         | HubSpot API documentation: https://developers.hubspot.com/docs/api/overview |
| Ensure the uploaded Excel/CSV file has a column named 'emails' (or adjust the Parse Data node accordingly). | Data preparation best practices.               |
| Webhook URL must be accessible externally from where the file upload originates (e.g., Postman, app).      | n8n Webhook documentation: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/ |
| The workflow handles errors gracefully, continuing processing when individual contact deletions fail.      | Workflow resilience best practices.            |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---