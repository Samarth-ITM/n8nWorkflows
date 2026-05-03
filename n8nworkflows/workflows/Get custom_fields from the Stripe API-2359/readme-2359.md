Get custom_fields from the Stripe API

https://n8nworkflows.xyz/workflows/get-custom_fields-from-the-stripe-api-2359


# Get custom_fields from the Stripe API

### 1. Workflow Overview

This workflow is designed to retrieve **custom fields** from Stripe checkout sessions, which are not directly accessible through the standard invoice or charge API endpoints. It targets users who want to extract detailed custom field information from their Stripe checkout data for analysis or integration purposes.

The workflow logic is divided into three main blocks:

- **1.1 Retrieve Checkout Sessions:** Queries the Stripe API to fetch all checkout sessions created within a configurable recent period (default last 7 days, actually 20 days in the current config). It handles pagination automatically to collect all relevant records.

- **1.2 Data Splitting and Transformation:** Processes the retrieved JSON array data by splitting it into individual sessions and further extracting each session’s `custom_fields` array into separate items for easier handling.

- **1.3 Filtering Custom Fields:** Applies a filter to keep only those custom fields matching specified keys (e.g., "nickname" and "job_title"), enabling users to focus on particular custom data of interest.

---

### 2. Block-by-Block Analysis

#### 2.1 Retrieve Checkout Sessions

- **Overview:**  
Fetches all Stripe checkout sessions created within a defined timeframe (last 20 days by default), using the Stripe API’s checkout sessions endpoint with JSON query parameters and pagination support to retrieve all pages of results.

- **Nodes Involved:**  
  - Stripe | Get latest checkout sessions1  
  - Sticky Note4 (documentation)

- **Node Details:**  

  - **Stripe | Get latest checkout sessions1**  
    - Type: HTTP Request  
    - Role: Retrieves checkout sessions from Stripe API with pagination  
    - Configuration:  
      - URL: `https://api.stripe.com/v1/checkout/sessions`  
      - Authentication: Stripe API credentials (predefined OAuth or API key)  
      - Query Parameters: JSON format filtering by `created` timestamp between 20 days ago and now  
      - Pagination: Enabled with `starting_after` parameter, completes when `has_more` flag is false  
      - Version: 4.2  
    - Key Expressions:  
      - `created.gte` and `created.lte` use dynamic date expressions based on `$today` minus 20 days and current time  
      - Pagination uses last ID of previous page to fetch next page  
    - Inputs: None (start node)  
    - Outputs: JSON array of checkout sessions  
    - Edge Cases:  
      - API rate limits or auth errors may cause failures  
      - Pagination misconfiguration could cause incomplete data  
      - Date expression errors if system date not available  
    - Sub-workflow: None  

  - **Sticky Note4**  
    - Type: Sticky Note (documentation)  
    - Content: Explains purpose of this node, time period customization, and pagination importance  
    - Covers: Stripe | Get latest checkout sessions1  

#### 2.2 Data Splitting and Transformation

- **Overview:**  
Splits the bulk JSON response into individual checkout sessions, and then extracts each session’s `custom_fields` array into separate items for granular processing and filtering.

- **Nodes Involved:**  
  - split all data  
  - split custom_fields  
  - Sticky Note5  

- **Node Details:**  

  - **split all data**  
    - Type: Split Out  
    - Role: Splits the main array of sessions (`data` field) into individual items  
    - Configuration:  
      - Field to split out: `data` (the array of checkout sessions)  
      - Options: Default (no additional options)  
    - Inputs: Output of Stripe API request node  
    - Outputs: Each checkout session as a separate item  
    - Edge Cases: Empty `data` array results in no output items  

  - **split custom_fields**  
    - Type: Split Out  
    - Role: Splits the `custom_fields` array inside each checkout session into individual items  
    - Configuration:  
      - Field to split out: `custom_fields`  
      - Include all other fields alongside each split item for context  
    - Inputs: Output of split all data node  
    - Outputs: Each custom field per session as an individual item  
    - Edge Cases: Sessions with empty or missing `custom_fields` produce no split items  

  - **Sticky Note5**  
    - Type: Sticky Note  
    - Content: Indicates that splitting is done to facilitate easier visualization and filtering of data  

#### 2.3 Filtering Custom Fields

- **Overview:**  
Filters the extracted custom fields to keep only those matching user-defined keys, such as "nickname" and "job_title," enabling targeted retrieval of meaningful custom data.

- **Nodes Involved:**  
  - Filter by custom_field  
  - Sticky Note6  

- **Node Details:**  

  - **Filter by custom_field**  
    - Type: Filter  
    - Role: Filters custom field items based on key value equality  
    - Configuration:  
      - Conditions:  
        - Custom field key equals "nickname" **AND**  
        - Custom field key equals "job_title"  
      - Case-sensitive, strict type validation  
    - Inputs: Output of split custom_fields node  
    - Outputs: Only items matching both keys pass through (note: given the AND combinator, no item can have both keys simultaneously, so this might be a logic oversight; possibly intended OR)  
    - Edge Cases: No matching custom fields result in empty output  
    - Potential logical issue: AND condition on two different key values can never be true for one item. Should likely be OR for meaningful filtering.  

  - **Sticky Note6**  
    - Type: Sticky Note  
    - Content: Explains that this node allows filtering contacts by custom fields, e.g., only those that filled "nickname" and "job_title"  

---

### 3. Summary Table

| Node Name                          | Node Type         | Functional Role                               | Input Node(s)                  | Output Node(s)           | Sticky Note                                                                                                                        |
|-----------------------------------|-------------------|-----------------------------------------------|-------------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Stripe | Get latest checkout sessions1 | HTTP Request     | Start node (no input)                         | split all data               | Sticky Note4 covers this node: explains retrieval and pagination setup                                                           |
| split all data                    | Split Out         | Splits the sessions array into individual items | Stripe | Get latest checkout sessions1 | split custom_fields         | Sticky Note5 covers this and next node: indicates splitting for easier visualization                                            |
| split custom_fields               | Split Out         | Splits custom_fields array into individual items | split all data                | Filter by custom_field     | Sticky Note5 covers this and previous node                                                                                        |
| Filter by custom_field            | Filter            | Filters custom fields by key ("nickname", "job_title") | split custom_fields           | (No output connected in JSON) | Sticky Note6 covers this node: explains filtering by custom fields                                                               |
| Sticky Note4                     | Sticky Note       | Documentation for Stripe sessions retrieval    |                               |                          | See above                                                                                                                        |
| Sticky Note5                     | Sticky Note       | Documentation for data splitting for visualization |                               |                          | See above                                                                                                                        |
| Sticky Note6                     | Sticky Note       | Documentation for filtering by custom fields   |                               |                          | See above                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the HTTP Request Node: "Stripe | Get latest checkout sessions1"**  
   - Type: HTTP Request  
   - URL: `https://api.stripe.com/v1/checkout/sessions`  
   - Authentication: Use Stripe API credentials (API key or OAuth2) configured in n8n  
   - HTTP Method: GET  
   - Query Parameters:  
     - Select "Specify Query" as JSON  
     - Enter JSON query:  
       ```json
       {
         "created": {
           "gte": {{ $today.minus(20, 'days').toSeconds() }},
           "lte": {{ $today.toSeconds() }}
         }
       }
       ```  
   - Enable Pagination:  
     - Pagination type: "Other"  
     - Pagination parameters:  
       - Parameter name: `starting_after`  
       - Parameter value: `={{ $response.body.data.last().id }}`  
     - Pagination complete expression: `={{ $response.body.has_more == false }}`  
   - Version: 4.2 or later (to support advanced pagination and JSON query)  

2. **Create a Split Out Node: "split all data"**  
   - Type: Split Out  
   - Field to split out: `data`  
   - Connect input from "Stripe | Get latest checkout sessions1" output  

3. **Create another Split Out Node: "split custom_fields"**  
   - Type: Split Out  
   - Field to split out: `custom_fields`  
   - Enable "Include all other fields" (to keep context)  
   - Connect input from "split all data" output  

4. **Create a Filter Node: "Filter by custom_field"**  
   - Type: Filter  
   - Conditions:  
     - Set combinator to OR (recommended correction):  
       - Custom field key equals "nickname"  
       - Custom field key equals "job_title"  
   - Set case sensitivity and strict type validation as needed  
   - Connect input from "split custom_fields" output  

5. **(Optional) Add Sticky Note Nodes for Documentation**  
   - Add notes explaining each major step, including:  
     - Retrieval and pagination details  
     - Reason for splitting data  
     - Filtering logic and usage examples  

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The Stripe API does not provide custom fields directly in invoice or charge data; this workflow extracts them from checkout sessions. | Stripe API Docs: https://docs.stripe.com/api/checkout/sessions                                           |
| Adjust the "created" filter to modify the date range for sessions retrieved.                                                           | Stripe API "created" parameter docs: https://docs.stripe.com/api/checkout/sessions/list#list_checkout_sessions-created |
| Pagination is crucial to retrieve all data beyond the first page; misconfiguration leads to partial results.                           | n8n Pagination docs: https://docs.n8n.io/nodes/n8n-nodes-base.httpRequest/#pagination                      |
| Filtering uses an AND combinator on keys "nickname" and "job_title" which will never both be true on the same custom field item.       | Logical correction recommended: use OR to filter items that match either key.                            |
| More workflow templates by the author available at: https://n8n.io/creators/solomon/                                                    | Author's n8n templates page                                                                              |

---

This documentation provides a complete and precise reference for understanding, reproducing, and modifying the "Get custom_fields from the Stripe API" workflow, ensuring robustness in data retrieval and processing.