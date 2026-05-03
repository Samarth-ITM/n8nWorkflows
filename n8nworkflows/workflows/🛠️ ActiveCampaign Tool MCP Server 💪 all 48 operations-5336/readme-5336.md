🛠️ ActiveCampaign Tool MCP Server 💪 all 48 operations

https://n8nworkflows.xyz/workflows/----activecampaign-tool-mcp-server----all-48-operations-5336


# 🛠️ ActiveCampaign Tool MCP Server 💪 all 48 operations

### 1. Workflow Overview

This workflow, titled **ActiveCampaign Tool MCP Server**, serves as a comprehensive server-side implementation enabling 48 distinct ActiveCampaign operations via n8n. Its purpose is to provide a unified webhook-driven endpoint that listens for AI-powered requests and executes respective ActiveCampaign API calls. The workflow is designed to cover full CRUD (Create, Read, Update, Delete) capabilities and related actions for multiple ActiveCampaign resources, targeting advanced users who want to programmatically manage their ActiveCampaign data through n8n automation.

The logical blocks group nodes by ActiveCampaign resource type, each performing the typical operations on that resource:

- **1.1 Input Reception**: Webhook trigger node that receives incoming requests.
- **1.2 Account Management**: Nodes related to ActiveCampaign accounts.
- **1.3 Account Contact Management**: Operations on account-contact relations.
- **1.4 Connection Management**: Handling ActiveCampaign connections.
- **1.5 Contact Management**: Contact CRUD and related operations.
- **1.6 Contact List Management**: Adding/removing contacts to/from lists.
- **1.7 Contact Tag Management**: Adding/removing tags from contacts.
- **1.8 Deal Management**: CRUD and note operations on deals.
- **1.9 E-commerce Customer Management**: Managing e-commerce customers.
- **1.10 E-commerce Order Management**: Managing orders and products.
- **1.11 List Management**: Retrieving lists.
- **1.12 Tag Management**: CRUD actions on tags.

Each block is visually separated by sticky notes and clusters nodes performing similar resource operations. All nodes depend on the initial webhook node and use data extracted via AI expressions to dynamically define parameters.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures incoming requests via a webhook endpoint that triggers the entire workflow, enabling AI-driven routing to specific ActiveCampaign operations.
- **Nodes Involved:**
  - ActiveCampaign Tool MCP Server (mcpTrigger)
- **Node Details:**
  - **Type:** MCP Trigger (Langchain integration node)
  - **Config:** Listens on webhook path `/activecampaign-tool-mcp`
  - **Expressions:** None directly; triggers workflow with AI payload
  - **Connections:** Outputs to all operation nodes via "ai_tool" connection
  - **Edge Cases:** Webhook authorization issues, malformed or incomplete input, timeout if no request
  - **Version:** 1

#### 1.2 Account Management

- **Overview:** Manage ActiveCampaign accounts including creating, retrieving, updating, deleting, and listing accounts.
- **Nodes Involved:**
  - Create an account
  - Delete an account
  - Get an account
  - Get many accounts
  - Update an account
  - Sticky Note labeled "## Account"
- **Node Details:**

  | Node Name         | Details                                                                                                    |
  |-------------------|------------------------------------------------------------------------------------------------------------|
  | Create an account  | ActiveCampaign Tool node; creates account with dynamic name from AI input `Name`                           |
  | Delete an account  | Deletes account by `Account_Id` from AI input                                                             |
  | Get an account     | Retrieves account details by `Account_Id`                                                                 |
  | Get many accounts  | Retrieves multiple accounts with `Limit`, `Simple`, and `Return_All` flags from AI input                   |
  | Update an account  | Updates account by `Account_Id` with fields from AI input                                                  |
  | Sticky Note       | Visual label for the Account block                                                                         |
  - **Authentication:** Uses ActiveCampaign API credentials (must be set)
  - **Edge Cases:** Authentication failure, invalid IDs, missing parameters, API rate limits

#### 1.3 Account Contact Management

- **Overview:** CRUD operations for linking contacts to accounts.
- **Nodes Involved:**
  - Create an account contact
  - Delete an account contact
  - Update an account contact
  - Sticky Note "## Accountcontact"
- **Node Details:**
  - Operations take `Account`, `Contact`, and `Account_Contact_Id` from AI input
  - Edge cases include invalid contact/account IDs, missing parameters, credentials errors

#### 1.4 Connection Management

- **Overview:** Manage ActiveCampaign connections including create, read, update, delete, and list.
- **Nodes Involved:**
  - Create a connection
  - Delete a connection
  - Get a connection
  - Get many connections
  - Update a connection
  - Sticky Note "## Connection"
- **Node Details:**
  - Inputs include `Name`, `Link_Url`, `Logo_Url`, `Service`, `Externalid`, `Connection_Id`
  - Edge cases: malformed URLs, invalid connection IDs, API failures

#### 1.5 Contact Management

- **Overview:** CRUD contacts with create, retrieve, update, delete, and list.
- **Nodes Involved:**
  - Create a contact
  - Delete a contact
  - Get a contact
  - Get many contacts
  - Update a contact
  - Sticky Note "## Contact"
- **Node Details:**
  - Key inputs: `Email`, `Contact_Id`, `Update_If_Exists`, pagination parameters
  - Edge cases: duplicate emails, invalid IDs, missing required email on create

#### 1.6 Contact List Management

- **Overview:** Add or remove contacts from lists.
- **Nodes Involved:**
  - Add a contact to a list
  - Remove a contact from a list
  - Sticky Note "## Contactlist"
- **Node Details:**
  - Inputs: `Contact_Id`, `List_Id`
  - Edge cases: invalid list or contact IDs, removing non-existent association

#### 1.7 Contact Tag Management

- **Overview:** Add or remove tags to contacts.
- **Nodes Involved:**
  - Add a contact tag
  - Remove a contact tag
  - Sticky Note "## Contacttag"
- **Node Details:**
  - Inputs: `Contact_Id`, `Tag_Id`, `Contact_Tag_Id`
  - Edge cases: duplicate tags, invalid IDs, API rate limits

#### 1.8 Deal Management

- **Overview:** Create, read, update, delete deals and manage deal notes.
- **Nodes Involved:**
  - Create a deal
  - Create a deal note
  - Delete a deal
  - Get a deal
  - Get many deals
  - Update a deal
  - Update a deal note
  - Sticky Note "## Deal"
- **Node Details:**
  - Inputs include `Group`, `Owner`, `Stage`, `Title`, `Value`, `Contact`, `Currency`, `Deal_Id`, `Deal_Note`, `Deal_Note_Id`
  - Edge cases: invalid deal IDs, missing required fields, note update/delete failures

#### 1.9 E-commerce Customer Management

- **Overview:** Manage e-commerce customers including create, read, update, delete, and list.
- **Nodes Involved:**
  - Create an e-commerce customer
  - Delete an e-commerce customer
  - Get an e-commerce customer
  - Get many e-commerce customers
  - Update an e-commerce customer
  - Sticky Note "## Ecommercecustomer"
- **Node Details:**
  - Inputs: `Email`, `Externalid`, `Connectionid`, `Ecommerce_Customer_Id`
  - Edge cases: invalid customer IDs, missing email, credential errors

#### 1.10 E-commerce Order Management

- **Overview:** Create, retrieve, update, delete e-commerce orders and retrieve order products.
- **Nodes Involved:**
  - Create an e-commerce order
  - Delete an e-commerce order
  - Get an e-commerce order
  - Get many e-commerce orders
  - Update an e-commerce order
  - Get many ecommerce orders (products)
  - Get an e-commerce order product by product ID
  - Get an e-commerce order product by order ID
  - Sticky Note "## Ecommerceorder" and "## Ecommerceorderproducts"
- **Node Details:**
  - Inputs: multiple including `Email`, `Source`, `Currency`, `Customerid`, `Externalid`, `Total_Price`, `Connectionid`, `Abandoned_Date`, `Order_Products`, `Order_Id`, `Procuct_Id`
  - Edge cases: complex JSON input for products, ID mismatches, date format errors

#### 1.11 List Management

- **Overview:** Retrieve lists.
- **Nodes Involved:**
  - Get many lists
  - Sticky Note "## List"
- **Node Details:**
  - Inputs: `Limit`, `Simple`, `Return_All`
  - Edge cases: large data volume, API throttling

#### 1.12 Tag Management

- **Overview:** Create, read, update, delete tags.
- **Nodes Involved:**
  - Create a tag
  - Delete a tag
  - Get a tag
  - Get many tags
  - Update a tag
  - Sticky Note "## Tag"
- **Node Details:**
  - Inputs: `Name`, `Tag_Type`, `Tag_Id`
  - Edge cases: duplicate tag names, invalid IDs

---

### 3. Summary Table

| Node Name                      | Node Type                   | Functional Role               | Input Node(s)                | Output Node(s)               | Sticky Note                              |
|--------------------------------|-----------------------------|------------------------------|------------------------------|------------------------------|-----------------------------------------|
| ActiveCampaign Tool MCP Server  | MCP Trigger                 | Entry webhook trigger        | -                            | All ActiveCampaign nodes      |                                         |
| Create an account              | ActiveCampaign Tool         | Create account               | ActiveCampaign Tool MCP Server | -                            | ## Account                              |
| Delete an account              | ActiveCampaign Tool         | Delete account               | ActiveCampaign Tool MCP Server | -                            | ## Account                              |
| Get an account                 | ActiveCampaign Tool         | Retrieve account             | ActiveCampaign Tool MCP Server | -                            | ## Account                              |
| Get many accounts             | ActiveCampaign Tool         | List accounts                | ActiveCampaign Tool MCP Server | -                            | ## Account                              |
| Update an account             | ActiveCampaign Tool         | Update account               | ActiveCampaign Tool MCP Server | -                            | ## Account                              |
| Sticky Note                   | Sticky Note                 | Visual label for accounts    | -                            | -                            | ## Account                              |
| Create an account contact     | ActiveCampaign Tool         | Link contact to account      | ActiveCampaign Tool MCP Server | -                            | ## Accountcontact                       |
| Delete an account contact     | ActiveCampaign Tool         | Remove contact-account link  | ActiveCampaign Tool MCP Server | -                            | ## Accountcontact                       |
| Update an account contact     | ActiveCampaign Tool         | Update contact-account link  | ActiveCampaign Tool MCP Server | -                            | ## Accountcontact                       |
| Sticky Note                   | Sticky Note                 | Visual label for acc. contact| -                            | -                            | ## Accountcontact                       |
| Create a connection           | ActiveCampaign Tool         | Create connection            | ActiveCampaign Tool MCP Server | -                            | ## Connection                          |
| Delete a connection           | ActiveCampaign Tool         | Delete connection            | ActiveCampaign Tool MCP Server | -                            | ## Connection                          |
| Get a connection              | ActiveCampaign Tool         | Retrieve connection          | ActiveCampaign Tool MCP Server | -                            | ## Connection                          |
| Get many connections          | ActiveCampaign Tool         | List connections             | ActiveCampaign Tool MCP Server | -                            | ## Connection                          |
| Update a connection           | ActiveCampaign Tool         | Update connection            | ActiveCampaign Tool MCP Server | -                            | ## Connection                          |
| Sticky Note                   | Sticky Note                 | Visual label for connection  | -                            | -                            | ## Connection                          |
| Create a contact              | ActiveCampaign Tool         | Create contact               | ActiveCampaign Tool MCP Server | -                            | ## Contact                             |
| Delete a contact              | ActiveCampaign Tool         | Delete contact               | ActiveCampaign Tool MCP Server | -                            | ## Contact                             |
| Get a contact                 | ActiveCampaign Tool         | Retrieve contact             | ActiveCampaign Tool MCP Server | -                            | ## Contact                             |
| Get many contacts             | ActiveCampaign Tool         | List contacts                | ActiveCampaign Tool MCP Server | -                            | ## Contact                             |
| Update a contact             | ActiveCampaign Tool         | Update contact               | ActiveCampaign Tool MCP Server | -                            | ## Contact                             |
| Sticky Note                   | Sticky Note                 | Visual label for contacts    | -                            | -                            | ## Contact                             |
| Add a contact to a list       | ActiveCampaign Tool         | Add contact to list          | ActiveCampaign Tool MCP Server | -                            | ## Contactlist                         |
| Remove a contact from a list  | ActiveCampaign Tool         | Remove contact from list     | ActiveCampaign Tool MCP Server | -                            | ## Contactlist                         |
| Sticky Note                   | Sticky Note                 | Visual label for contactlist | -                            | -                            | ## Contactlist                         |
| Add a contact tag             | ActiveCampaign Tool         | Add tag to contact           | ActiveCampaign Tool MCP Server | -                            | ## Contacttag                          |
| Remove a contact tag          | ActiveCampaign Tool         | Remove tag from contact      | ActiveCampaign Tool MCP Server | -                            | ## Contacttag                          |
| Sticky Note                   | Sticky Note                 | Visual label for contacttag  | -                            | -                            | ## Contacttag                          |
| Create a deal                | ActiveCampaign Tool         | Create deal                  | ActiveCampaign Tool MCP Server | -                            | ## Deal                               |
| Create a deal note           | ActiveCampaign Tool         | Add note to deal             | ActiveCampaign Tool MCP Server | -                            | ## Deal                               |
| Delete a deal                | ActiveCampaign Tool         | Delete deal                  | ActiveCampaign Tool MCP Server | -                            | ## Deal                               |
| Get a deal                   | ActiveCampaign Tool         | Retrieve deal                | ActiveCampaign Tool MCP Server | -                            | ## Deal                               |
| Get many deals               | ActiveCampaign Tool         | List deals                   | ActiveCampaign Tool MCP Server | -                            | ## Deal                               |
| Update a deal               | ActiveCampaign Tool         | Update deal                  | ActiveCampaign Tool MCP Server | -                            | ## Deal                               |
| Update a deal note           | ActiveCampaign Tool         | Update deal note             | ActiveCampaign Tool MCP Server | -                            | ## Deal                               |
| Sticky Note                   | Sticky Note                 | Visual label for deals       | -                            | -                            | ## Deal                               |
| Create an e-commerce customer | ActiveCampaign Tool         | Create ecommerce customer    | ActiveCampaign Tool MCP Server | -                            | ## Ecommercecustomer                  |
| Delete an e-commerce customer | ActiveCampaign Tool         | Delete ecommerce customer    | ActiveCampaign Tool MCP Server | -                            | ## Ecommercecustomer                  |
| Get an e-commerce customer    | ActiveCampaign Tool         | Retrieve ecommerce customer  | ActiveCampaign Tool MCP Server | -                            | ## Ecommercecustomer                  |
| Get many e-commerce customers | ActiveCampaign Tool         | List ecommerce customers     | ActiveCampaign Tool MCP Server | -                            | ## Ecommercecustomer                  |
| Update an e-commerce customer | ActiveCampaign Tool         | Update ecommerce customer    | ActiveCampaign Tool MCP Server | -                            | ## Ecommercecustomer                  |
| Sticky Note                   | Sticky Note                 | Visual label for ecommerce cust| -                          | -                            | ## Ecommercecustomer                  |
| Create an e-commerce order    | ActiveCampaign Tool         | Create ecommerce order       | ActiveCampaign Tool MCP Server | -                            | ## Ecommerceorder                    |
| Delete an e-commerce order    | ActiveCampaign Tool         | Delete ecommerce order       | ActiveCampaign Tool MCP Server | -                            | ## Ecommerceorder                    |
| Get an e-commerce order       | ActiveCampaign Tool         | Retrieve ecommerce order     | ActiveCampaign Tool MCP Server | -                            | ## Ecommerceorder                    |
| Get many e-commerce orders    | ActiveCampaign Tool         | List ecommerce orders        | ActiveCampaign Tool MCP Server | -                            | ## Ecommerceorder                    |
| Update an e-commerce order    | ActiveCampaign Tool         | Update ecommerce order       | ActiveCampaign Tool MCP Server | -                            | ## Ecommerceorder                    |
| Get many ecommerce orders     | ActiveCampaign Tool         | Get ecommerce order products | ActiveCampaign Tool MCP Server | -                            | ## Ecommerceorderproducts            |
| Get an e-commerce order product by product ID | ActiveCampaign Tool | Retrieve ecommerce order product by product ID | ActiveCampaign Tool MCP Server | - | ## Ecommerceorderproducts            |
| Get an e-commerce order product by order ID | ActiveCampaign Tool | Retrieve ecommerce order product by order ID | ActiveCampaign Tool MCP Server | - | ## Ecommerceorderproducts            |
| Sticky Note                   | Sticky Note                 | Visual label for ecommerce orders/products | -                   | -                            | ## Ecommerceorderproducts            |
| Get many lists               | ActiveCampaign Tool         | Retrieve lists               | ActiveCampaign Tool MCP Server | -                            | ## List                              |
| Sticky Note                   | Sticky Note                 | Visual label for lists       | -                            | -                            | ## List                              |
| Create a tag                 | ActiveCampaign Tool         | Create tag                  | ActiveCampaign Tool MCP Server | -                            | ## Tag                               |
| Delete a tag                 | ActiveCampaign Tool         | Delete tag                  | ActiveCampaign Tool MCP Server | -                            | ## Tag                               |
| Get a tag                    | ActiveCampaign Tool         | Retrieve tag                | ActiveCampaign Tool MCP Server | -                            | ## Tag                               |
| Get many tags                | ActiveCampaign Tool         | List tags                   | ActiveCampaign Tool MCP Server | -                            | ## Tag                               |
| Update a tag                | ActiveCampaign Tool         | Update tag                  | ActiveCampaign Tool MCP Server | -                            | ## Tag                               |
| Sticky Note                  | Sticky Note                 | Visual label for tags       | -                            | -                            | ## Tag                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node:**
   - Add an **MCP Trigger** node (type `@n8n/n8n-nodes-langchain.mcpTrigger`).
   - Configure the webhook path as `/activecampaign-tool-mcp`.
   - This node will be the single entry point for all AI-driven requests.

2. **Set Up ActiveCampaign API Credentials:**
   - Create or use existing ActiveCampaign API credentials in n8n.
   - Assign these credentials to all ActiveCampaign Tool nodes created below.

3. **Create nodes for Account Management:**
   - Add **ActiveCampaign Tool** nodes for:
     - Create an account: use resource `account`, operation `create`, name from AI input `Name`.
     - Delete an account: resource `account`, operation `delete`, accountId from AI input `Account_Id`.
     - Get an account: resource `account`, operation `get`, accountId from AI input `Account_Id`.
     - Get many accounts: resource `account`, operation `getAll`, parameters `Limit`, `Simple`, `Return_All` from AI input.
     - Update an account: resource `account`, operation `update`, accountId and updateFields from AI input.
   - Add a sticky note titled `## Account` above these nodes.

4. **Create nodes for Account Contact Management:**
   - Add nodes for create, delete, update operations on `accountContact` resource.
   - Use AI inputs `Account`, `Contact`, `Account_Contact_Id`.
   - Add sticky note `## Accountcontact`.

5. **Create nodes for Connection Management:**
   - Add nodes for create, delete, get, getAll, update operations on `connection` resource.
   - Pass AI inputs `Name`, `Link_Url`, `Logo_Url`, `Service`, `Externalid`, `Connection_Id`.
   - Add sticky note `## Connection`.

6. **Create nodes for Contact Management:**
   - Add nodes for create, delete, get, getAll, update operations on `contact` resource.
   - Use AI inputs `Email`, `Contact_Id`, `Update_If_Exists`.
   - Add sticky note `## Contact`.

7. **Create nodes for Contact List Management:**
   - Add nodes for adding/removing contacts to/from lists (`contactList` resource).
   - Use AI inputs `Contact_Id`, `List_Id`.
   - Add sticky note `## Contactlist`.

8. **Create nodes for Contact Tag Management:**
   - Add nodes for adding/removing tags (`contactTag` resource).
   - Use AI inputs `Contact_Id`, `Tag_Id`, `Contact_Tag_Id`.
   - Add sticky note `## Contacttag`.

9. **Create nodes for Deal Management:**
   - Add nodes for create, delete, get, getAll, update deals and notes (`deal` resource).
   - Inputs include `Group`, `Owner`, `Stage`, `Title`, `Value`, `Contact`, `Currency`, `Deal_Id`, `Deal_Note`, `Deal_Note_Id`.
   - Add sticky note `## Deal`.

10. **Create nodes for E-commerce Customer Management:**
    - Nodes for create, delete, get, getAll, update ecommerce customers (`ecommerceCustomer` resource).
    - Use inputs like `Email`, `Externalid`, `Connectionid`, `Ecommerce_Customer_Id`.
    - Add sticky note `## Ecommercecustomer`.

11. **Create nodes for E-commerce Order Management:**
    - Nodes for create, delete, get, getAll, update ecommerce orders (`ecommerceOrder` resource).
    - Also add nodes for retrieving ecommerce order products (`ecommerceOrderProducts` resource) by product ID or order ID.
    - Inputs include complex JSON for products and various IDs.
    - Add sticky notes `## Ecommerceorder` and `## Ecommerceorderproducts`.

12. **Create nodes for List Management:**
    - Node for getting many lists (`list` resource).
    - Inputs include `Limit`, `Simple`, `Return_All`.
    - Add sticky note `## List`.

13. **Create nodes for Tag Management:**
    - Nodes for create, delete, get, getAll, update tags (`tag` resource).
    - Inputs include `Name`, `Tag_Type`, `Tag_Id`.
    - Add sticky note `## Tag`.

14. **Connect all ActiveCampaign Tool nodes to the MCP Trigger node via an "ai_tool" output connection:**
    - This connection is required to route AI requests to the proper operation node.

15. **Configure each ActiveCampaign Tool node to use the ActiveCampaign API credentials.**

16. **Test the workflow by sending requests with corresponding AI input keys for each operation.**

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is a centralized MCP server implementation enabling AI-driven multi-operation ActiveCampaign access. | Workflow designed by David Ashby (email: david.ashby.lds@gmail.com).                               |
| Use proper ActiveCampaign API credentials with sufficient permissions for all resource operations managed here. | ActiveCampaign API documentation: https://developers.activecampaign.com/reference                 |
| Ensure AI input keys are correctly formatted and provided to avoid expression errors in node parameters.       | n8n expression docs: https://docs.n8n.io/nodes/expressions/                                        |
| Sticky notes provide useful visual grouping to understand resource domains inside the workflow editor.         | Best practice to keep workflow organized in large multi-operation designs.                         |

---

**Disclaimer:**  
The provided content originates exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected data. All manipulated data are legal and public.