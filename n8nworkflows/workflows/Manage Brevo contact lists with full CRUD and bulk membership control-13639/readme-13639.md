Manage Brevo contact lists with full CRUD and bulk membership control

https://n8nworkflows.xyz/workflows/manage-brevo-contact-lists-with-full-crud-and-bulk-membership-control-13639


# Manage Brevo contact lists with full CRUD and bulk membership control

This document provides a comprehensive technical reference for the **Brevo List Management Master Template**. This workflow extends the native n8n Brevo node capabilities by providing direct API implementations for full CRUD (Create, Read, Update, Delete) operations on contact lists and bulk membership management.

---

### 1. Workflow Overview

The workflow serves as a modular toolkit for managing Brevo contact lists via the Brevo v3 API. It is designed to handle tasks that are often limited in standard integrations, such as advanced pagination for large list sets, specific list metadata updates, and bulk contact additions/removals.

**Logical Blocks:**
*   **1.1 List Management (CRUD):** Operations to fetch all lists, create new ones, retrieve specific details, update names/folders, and delete lists.
*   **1.2 Membership Control:** Operations to list all contacts within a specific list (with auto-pagination) and bulk add/remove contacts using email arrays.

---

### 2. Block-by-Block Analysis

#### 2.1 List Management (CRUD)
**Overview:** This block handles the lifecycle of contact lists within Brevo.
**Nodes Involved:** `Get All Lists`, `Aggregate Lists Pages`, `Create a List`, `Get a List’s Details`, `Update a List`, `Delete a List`.

*   **Node: Get All Lists**
    *   **Type:** HTTP Request
    *   **Configuration:** GET `https://api.brevo.com/v3/contacts/lists`. Uses pagination with an offset of `{{ $pageCount * 50 }}` and a limit of 50.
    *   **Edge Cases:** Fails if the API key is invalid or if the rate limit is exceeded during high-frequency polling.
*   **Node: Aggregate Lists Pages**
    *   **Type:** Aggregate
    *   **Configuration:** Merges the `lists` field from the paginated responses of "Get All Lists" into a single array.
*   **Node: Create a List**
    *   **Type:** HTTP Request (POST)
    *   **Expressions:** `folderId` (defaults to 1), `name` (e.g., "New Customers").
*   **Node: Update a List**
    *   **Type:** HTTP Request (PUT)
    *   **Expressions:** Path contains `{listId}`. Body contains the new `name`.

#### 2.2 Membership Control
**Overview:** Manages which contacts belong to which lists.
**Nodes Involved:** `Get Contacts in a List`, `Aggregate Contacts Pages`, `Add Existing Contacts to a List`, `Delete Contacts from a List`.

*   **Node: Get Contacts in a List**
    *   **Type:** HTTP Request (GET)
    *   **Configuration:** Targets `.../lists/{listId}/contacts`. Uses a higher pagination limit (500) and offset `{{ $pageCount * 500 }}`.
    *   **Edge Cases:** Large lists may require multiple cycles; ensure timeout settings in n8n are sufficient.
*   **Node: Add Existing Contacts to a List**
    *   **Type:** HTTP Request (POST)
    *   **Expressions:** Sends an array of emails: `{{ ["email1@example.com", "email2@example.com"] }}`.
    *   **Constraint:** Brevo limits bulk additions to 150 emails per request.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Get All Lists | HTTP Request | Fetch all lists (Paginated) | - | Aggregate Lists Pages | Retrieves all existing lists with limiting and sorting. |
| Aggregate Lists Pages | Aggregate | Merge paginated results | Get All Lists | - | - |
| Create a List | HTTP Request | Create new list | - | - | Create a new contact list within a specific folder. |
| Get a List’s Details | HTTP Request | Fetch list metadata | - | - | Fetch metadata for a specific list ID. |
| Update a List | HTTP Request | Rename/Move list | - | - | Rename or move an existing list. |
| Delete a List | HTTP Request | Remove list | - | - | Permanently remove a list (contacts remain). |
| Get Contacts in a List | HTTP Request | Fetch list members | - | Aggregate Contacts Pages | Returns a list of all contacts belonging to a specific list. |
| Aggregate Contacts Pages | Aggregate | Merge member results | Get Contacts in a List | - | - |
| Add Existing Contacts to a List | HTTP Request | Bulk add to list | - | - | Bulk add contacts using emails or IDs (up to 150). |
| Delete Contacts from a List | HTTP Request | Bulk remove from list | - | - | Remove specific contacts from a list. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Credentials:** Create a "Brevo API" (SendInBlue) credential in n8n using your API Key v3.
2.  **Get All Lists:**
    *   Add an **HTTP Request** node. Method: `GET`, URL: `https://api.brevo.com/v3/contacts/lists`.
    *   Enable **Pagination**. Set "Limit" to 50. Set "Offset" to `={{ $pageCount * 50 }}`.
    *   Set "Pagination Complete When" to "Expression" and use `={{ $pageCount * 50 >= $response.body.count }}`.
3.  **Aggregate Results:** Connect an **Aggregate** node to the "Get All Lists" node. Set it to "Merge Lists" and aggregate the `lists` field.
4.  **Create/Update/Delete:**
    *   Use **HTTP Request** nodes with the corresponding Methods (POST, PUT, DELETE).
    *   URLs follow the pattern `https://api.brevo.com/v3/contacts/lists/{listId}`.
5.  **Membership Management:**
    *   Create an **HTTP Request** for "Get Contacts in a List". URL: `https://api.brevo.com/v3/contacts/lists/{{your_id}}/contacts`.
    *   Configure pagination similar to Step 2, but use a 500 limit/offset.
6.  **Bulk Operations:**
    *   Use **HTTP Request** (POST). URL: `.../contacts/add` or `.../contacts/remove`.
    *   In the Body, use "Fixed" or "Expression" to pass an array of emails under the key `emails`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Detailed guide on Brevo List Management | [n8n Playbook Article](https://n8nplaybook.com/post/2026/02/brevo-list-management-n8n-api/) |
| Brevo v3 API Reference | [Brevo API Documentation](https://developers.brevo.com/reference) |
| Native Brevo integration help | [n8n Brevo Node](https://n8nplaybook.com/go/brevo/) |