Sync Adobe Commerce Customer Data to KlickTipp with Value-Based Tagging

https://n8nworkflows.xyz/workflows/sync-adobe-commerce-customer-data-to-klicktipp-with-value-based-tagging-9469


# Sync Adobe Commerce Customer Data to KlickTipp with Value-Based Tagging

### 1. Workflow Overview

This workflow automates the synchronization of customer and order data from Adobe Commerce (Magento 2) to the KlickTipp marketing platform, enriching subscriber profiles with detailed attributes and applying conditional tags based on purchase behavior. It targets e-commerce businesses seeking to streamline customer data management and enable dynamic segmentation for targeted messaging.

The workflow consists of four logical blocks:

- **1.1 Data Reception:** Periodic polling of recent orders and updated customers from Adobe Commerce.
- **1.2 Data Saving:** Transferring detailed customer and order data into KlickTipp as subscriber profiles with enriched custom fields.
- **1.3 Routing for Tagging:** Decision logic to evaluate order attributes (total amount and SKU presence) to determine applicable tags.
- **1.4 Contact Tagging:** Applying conditional tags to KlickTipp contacts for high-value customers and specific product purchasers.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Reception

**Overview:**  
This block periodically triggers the workflow and fetches recent orders and updated customer data from Adobe Commerce within the last 6 minutes to ensure near-real-time synchronization.

**Nodes Involved:**  
- Schedule Trigger  
- Get Adobe Commerce orders  
- Get Adobe Commerce customers  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow every minute (interval in minutes)  
  - *Configuration:* Interval-based trigger firing every 1 minute  
  - *Input/Output:* No input; outputs trigger event to two nodes in parallel  
  - *Edge Cases:* Trigger delays can cause overlapping data fetch; configured with a 6-minute lookback to avoid missing data  
  - *Version:* 1.2

- **Get Adobe Commerce orders**  
  - *Type:* Magento 2 API node  
  - *Role:* Retrieves all orders created or updated in the last 6 minutes  
  - *Configuration:*  
    - Resource: order  
    - Operation: getAll  
    - Return all: true  
    - Filter JSON: Filters orders by created_at field with condition greater or equal to current time minus 6 minutes, sorted ascending by creation date  
  - *Input/Output:* Receives trigger from Schedule Trigger; outputs order data as JSON array  
  - *Credentials:* Magento 2 API with access token and base URL configured  
  - *Edge Cases:* API rate limits, authentication failure, empty order sets, time zone consistency for date filtering  
  - *Version:* 1

- **Get Adobe Commerce customers**  
  - *Type:* Magento 2 API node  
  - *Role:* Retrieves all customers updated in the last 6 minutes  
  - *Configuration:*  
    - Operation: getAll  
    - Return all: true  
    - Filter JSON: Filters customers by updated_at field with condition greater or equal to current time minus 6 minutes, sorted ascending by updated date  
  - *Input/Output:* Receives trigger from Schedule Trigger; outputs customer data as JSON array  
  - *Credentials:* Magento 2 API same as above  
  - *Edge Cases:* Similar to orders node, plus potential data inconsistencies if customer updates happen rapidly  
  - *Version:* 1

---

#### 1.2 Data Saving

**Overview:**  
Transfers the fetched customer and order data into KlickTipp by subscribing or updating contacts, enriching profiles with detailed address and purchase information.

**Nodes Involved:**  
- Transfer order data to KlickTipp  
- Transfer customers to KlickTipp  

**Node Details:**

- **Transfer order data to KlickTipp**  
  - *Type:* KlickTipp node (community node)  
  - *Role:* Subscribes or updates KlickTipp subscriber with order-related fields  
  - *Configuration:*  
    - Resource: subscriber  
    - Operation: subscribe  
    - Email: mapped from order’s `customer_email` field  
    - Custom fields: includes first name, last name, country, city, street, postal code, phone, payment transaction ID, grand total, receipt URL, and concatenated product names from order items  
    - OnError: continue error without stopping workflow  
  - *Input/Output:* Inputs orders data from "Get Adobe Commerce orders" node; outputs to routing node  
  - *Credentials:* KlickTipp API with username/password credentials  
  - *Key Expressions:*  
    - Product names concatenated from items array filtering for non-empty names  
    - Receipt URL dynamically constructed with PayPal sandbox transaction link  
  - *Edge Cases:* Missing or malformed fields in order data, API authentication failure, partial data in nested JSON fields, network timeouts  
  - *Version:* 3

- **Transfer customers to KlickTipp**  
  - *Type:* KlickTipp node  
  - *Role:* Subscribes or updates KlickTipp subscriber with customer profile data  
  - *Configuration:*  
    - Resource: subscriber  
    - Operation: subscribe  
    - Email: customer email field  
    - Custom fields: first name, last name, country, state (region), city, street, postal code, phone from the first customer address  
  - *Input/Output:* Inputs customers data from "Get Adobe Commerce customers" node  
  - *Credentials:* KlickTipp API same as above  
  - *Edge Cases:* Customers without addresses, missing fields, multiple addresses (only first used), API errors  
  - *Version:* 3

---

#### 1.3 Routing for Tagging

**Overview:**  
Evaluates order data to determine which tags should be applied to contacts, based on order value and product SKU conditions.

**Nodes Involved:**  
- Route by SKU and total amount  

**Node Details:**

- **Route by SKU and total amount**  
  - *Type:* Switch node  
  - *Role:* Routes data to different outputs based on conditions  
  - *Configuration:*  
    - Two outputs with conditions:  
      - "Order Value ≥ 100": Checks if the order’s grand_total is greater or equal to 100  
      - "Order contains clothing": Checks if any item in order has SKU 'TEST-002' (example SKU for clothing)  
    - Multiple outputs can fire simultaneously (allMatchingOutputs: true)  
  - *Input/Output:* Inputs order JSON from "Transfer order data to KlickTipp"; outputs to tagging nodes  
  - *Key Expressions:*  
    - Numeric comparison on total amount  
    - Boolean check on presence of SKU in items array  
  - *Edge Cases:* Missing grand_total, items array empty or undefined, SKU casing mismatch, unexpected data types  
  - *Version:* 3.2

---

#### 1.4 Contact Tagging

**Overview:**  
Applies KlickTipp tags to contacts dynamically based on routing decisions, enhancing segmentation for marketing automation.

**Nodes Involved:**  
- Tag contact for high-value order  
- Tag contact for clothing purchase  

**Node Details:**

- **Tag contact for high-value order**  
  - *Type:* KlickTipp node  
  - *Role:* Adds "Premium Customer" tag to subscriber if order value ≥ 100  
  - *Configuration:*  
    - Resource: contact-tagging  
    - Email: mapped from order email  
    - TagId: ["13548739"] (corresponds to Premium Customer tag)  
  - *Input/Output:* Input from routing node output "Order Value ≥ 100"  
  - *Credentials:* KlickTipp API  
  - *Edge Cases:* Tag ID mismatch, API auth issues, email field missing  
  - *Version:* 3

- **Tag contact for clothing purchase**  
  - *Type:* KlickTipp node  
  - *Role:* Adds "Clothing Buyer" tag if order contains SKU 'TEST-002'  
  - *Configuration:*  
    - Resource: contact-tagging  
    - Email: order email  
    - TagId: ["13548800"] (corresponds to Clothing Buyer tag)  
  - *Input/Output:* Input from routing node output "Order contains clothing"  
  - *Credentials:* KlickTipp API  
  - *Edge Cases:* Same as above  
  - *Version:* 3

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                          | Input Node(s)                   | Output Node(s)                            | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
|-------------------------------|-------------------------|----------------------------------------|--------------------------------|------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger        | Initiates workflow periodically        |                                | Get Adobe Commerce orders, Get Adobe Commerce customers |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Get Adobe Commerce orders      | Magento 2 API           | Fetches recent orders                   | Schedule Trigger               | Transfer order data to KlickTipp          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Get Adobe Commerce customers   | Magento 2 API           | Fetches recently updated customers     | Schedule Trigger               | Transfer customers to KlickTipp            |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Transfer order data to KlickTipp| KlickTipp node          | Subscribes order data to KlickTipp     | Get Adobe Commerce orders      | Route by SKU and total amount              |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Transfer customers to KlickTipp| KlickTipp node          | Subscribes customer data to KlickTipp  | Get Adobe Commerce customers   |                                          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Route by SKU and total amount  | Switch                  | Routes data for conditional tagging    | Transfer order data to KlickTipp| Tag contact for high-value order, Tag contact for clothing purchase |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Tag contact for high-value order| KlickTipp node         | Tags high-value customers               | Route by SKU and total amount  |                                          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Tag contact for clothing purchase| KlickTipp node        | Tags customers buying clothing          | Route by SKU and total amount  |                                          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Sticky Note                   | Sticky Note             | Workflow introduction and setup notes  |                                |                                          | Community Node Disclaimer: This workflow uses KlickTipp community nodes.\n\n### Introduction\nThis workflow monitors orders and customers in Adobe Commerce, automatically creating or updating contacts in KlickTipp, enriching profiles for segmentation and automated messaging. Tags are applied dynamically: high-value orders (≥100) receive a "Premium Customer" tag, and purchases with certain SKUs (e.g., clothing) are assigned product-based tags. Perfect for e-commerce businesses, online retailers, and digital shops that want to eliminate manual data entry and ensure every buyer and customer receives the right messages.\n\n### Setup Instructions\n1. **KlickTipp Preparation**\n      - Prepare **custom fields**\n       - `Payment ID`\n       - `Total`\n       - `Receipt URL`\n       - `Products`\n      - Prepare **tags**:\n       - `Premium customer`\n       - `Clothing buyer`\n\n2. **Credential Configuration**\n     - Connect your Magento account using an **Access Token/Base URL** from the Magento Admin Dashboard (System → Extensions → Integrations).\n     - Authenticate your KlickTipp connection with **username/password** credentials (API access required).\n\n### Customization\n- **Trigger options:** If your Commerce edition supports **webhooks**, you can replace polling with a **Webhook** trigger.  \n- **Cadence & overlap:** 1–30 min are typical; a 1–2 min overlap in the filter to avoid gaps.  \n- **Routing variants:** Change the SKU list, switch to category checks, or add more value tiers. |
| Sticky Note4                  | Sticky Note             | Label for contact tagging block        |                                |                                          | ## 4. Contact tagging                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Sticky Note3                  | Sticky Note             | Label for routing block                 |                                |                                          | ## 3. Routing for tagging                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Sticky Note1                  | Sticky Note             | Label for data saving block             |                                |                                          | ## 2. Data saving                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Sticky Note2                  | Sticky Note             | Label for data reception block          |                                |                                          | ## 1. Data reception                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Parameters: Set interval to trigger every 1 minute (field: "minutes")  

2. **Create Get Adobe Commerce orders node**  
   - Type: Magento 2 API  
   - Credentials: Configure your Magento 2 API credentials (Access Token and Base URL)  
   - Parameters:  
     - Resource: order  
     - Operation: getAll  
     - Return All: true  
     - Filter Type: JSON  
     - Filter JSON:  
       ```json
       {
         "search_criteria": {
           "filter_groups": [
             {
               "filters": [
                 {
                   "field": "created_at",
                   "condition_type": "gteq",
                   "value": "{{ $now.toUTC().minus({ minutes: 6 }).toFormat('yyyy-LL-dd HH:mm:ss') }}"
                 }
               ]
             }
           ],
           "sortOrders": [
             {
               "field": "created_at",
               "direction": "ASC"
             }
           ]
         }
       }
       ```  
   - Connect Schedule Trigger output to this node  

3. **Create Get Adobe Commerce customers node**  
   - Type: Magento 2 API  
   - Credentials: Same Magento 2 API credentials as above  
   - Parameters:  
     - Operation: getAll  
     - Return All: true  
     - Filter Type: JSON  
     - Filter JSON:  
       ```json
       {
         "search_criteria": {
           "filter_groups": [
             {
               "filters": [
                 {
                   "field": "updated_at",
                   "condition_type": "gteq",
                   "value": "{{ $now.toUTC().minus({ minutes: 6 }).toFormat('yyyy-LL-dd HH:mm:ss') }}"
                 }
               ]
             }
           ],
           "sortOrders": [
             {
               "field": "updated_at",
               "direction": "ASC"
             }
           ]
         }
       }
       ```  
   - Connect Schedule Trigger output to this node  

4. **Create Transfer order data to KlickTipp node**  
   - Type: KlickTipp node (community node)  
   - Credentials: Configure KlickTipp API credentials (username/password with API access)  
   - Parameters:  
     - Resource: subscriber  
     - Operation: subscribe  
     - Email: expression `{{$json.customer_email}}`  
     - Custom Fields:  
       - fieldFirstName: `{{$json.customer_firstname}}`  
       - fieldLastName: `{{$json.customer_lastname}}`  
       - fieldCountry: `{{$json.billing_address.country_id}}`  
       - fieldCity: `{{$json.billing_address.city}}`  
       - fieldStreet1: `{{$json.billing_address?.street?.[0] || ''}}`  
       - fieldZip: `{{$json.billing_address.postcode}}`  
       - fieldPhone: `{{$json.billing_address.telephone}}`  
       - field223236 (Payment ID): `{{$json.payment.last_trans_id}}`  
       - field223239 (Total): `{{$json.grand_total}}`  
       - field223245 (Receipt URL): `=https://www.sandbox.paypal.com/cgi-bin/webscr?cmd=_view-a-trans&id={{ $json.payment.last_trans_id }}` (literal string with embedded expression)  
       - field223237 (Products):  
         ```
         ={{ 
           ($json.items ?? []) 
             .map(i => i.name) 
             .filter(Boolean) 
             .join(', ') 
         }}
         ```  
   - OnError: Continue on error  
   - Connect "Get Adobe Commerce orders" output to this node  

5. **Create Transfer customers to KlickTipp node**  
   - Type: KlickTipp node  
   - Credentials: KlickTipp API same as above  
   - Parameters:  
     - Resource: subscriber  
     - Operation: subscribe  
     - Email: `{{$json.email}}`  
     - Custom Fields:  
       - fieldFirstName: `{{$json.firstname}}`  
       - fieldLastName: `{{$json.lastname}}`  
       - fieldCountry: `{{$json.addresses[0].country_id}}`  
       - fieldState: `{{$json.addresses[0].region.region}}`  
       - fieldCity: `{{$json.addresses[0].city}}`  
       - fieldStreet1: `{{$json.addresses[0].street[0]}}`  
       - fieldZip: `{{$json.addresses[0].postcode}}`  
       - fieldPhone: `{{$json.addresses[0].telephone}}`  
   - Connect "Get Adobe Commerce customers" output to this node  

6. **Create Route by SKU and total amount node**  
   - Type: Switch  
   - Parameters:  
     - Rules:  
       - Output Key: "Order Value ≥ 100"  
         - Condition: Number comparison, greater than or equal, expression:  
           `={{ $('Get Adobe Commerce orders').item.json.grand_total }}`  
           Right value: 100  
       - Output Key: "Order contains clothing"  
         - Condition: Boolean true operator on expression:  
           ```
           ={{ 
             ($('Get Adobe Commerce orders').item.json.items ?? []).some(it => (it.sku ?? '') === 'TEST-002') 
           }}
           ```  
     - Options: Enable "allMatchingOutputs" to allow multiple outputs simultaneously  
   - Connect "Transfer order data to KlickTipp" output to this node  

7. **Create Tag contact for high-value order node**  
   - Type: KlickTipp node  
   - Credentials: KlickTipp API  
   - Parameters:  
     - Resource: contact-tagging  
     - Email: `{{$json.email}}`  
     - TagId: `["13548739"]` (Premium customer tag)  
   - Connect "Route by SKU and total amount" output "Order Value ≥ 100" to this node  

8. **Create Tag contact for clothing purchase node**  
   - Type: KlickTipp node  
   - Credentials: KlickTipp API  
   - Parameters:  
     - Resource: contact-tagging  
     - Email: `{{$json.email}}`  
     - TagId: `["13548800"]` (Clothing buyer tag)  
   - Connect "Route by SKU and total amount" output "Order contains clothing" to this node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Community Node Disclaimer: This workflow uses KlickTipp community nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow note                                        |
| Introduction: This workflow monitors orders and customers in Adobe Commerce, automatically creating or updating contacts in KlickTipp, enriching profiles for segmentation and automated messaging. Tags are applied dynamically: high-value orders (≥100) receive a "Premium Customer" tag, and purchases with certain SKUs (e.g., clothing) are assigned product-based tags. Perfect for e-commerce businesses, online retailers, and digital shops that want to eliminate manual data entry and ensure every buyer and customer receives the right messages.                                                                                                  | Workflow note                                        |
| Setup Instructions: 1. Prepare KlickTipp custom fields: Payment ID, Total, Receipt URL, Products. 2. Prepare KlickTipp tags: Premium customer, Clothing buyer. 3. Configure Magento 2 API credentials (Access Token/Base URL) via Magento Admin Dashboard → System → Extensions → Integrations. 4. Configure KlickTipp API credentials (username/password with API access).                                                                                                                                                                                                                                                                                                          | Workflow note                                        |
| Customization: If Adobe Commerce edition supports webhooks, replace Schedule Trigger with Webhook Trigger for real-time sync. Typical trigger cadence is 1-30 minutes with 1-2 minutes overlap to avoid data gaps. Routing conditions can be customized by SKU lists, category checks, or additional value tiers.                                                                                                                                                                                                                                                                                                                                                                                                            | Workflow note                                        |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow built with n8n, a workflow automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.