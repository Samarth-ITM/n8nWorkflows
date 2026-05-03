Real-time lead routing in Webflow

https://n8nworkflows.xyz/workflows/real-time-lead-routing-in-webflow-2033


# Real-time lead routing in Webflow

### 1. Workflow Overview

This workflow implements real-time lead routing for form submissions received from a Webflow website. Its primary purpose is to enrich the submitted lead data using Datagma’s API and, based on the enrichment results, determine which Calendly link (one-on-one demo or group demo) should be presented to the user on the website.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives form submissions from Webflow via webhook.
- **1.2 Email Domain Extraction & Validation:** Extracts the domain from the submitted email and verifies if it belongs to a free email provider.
- **1.3 Data Enrichment:** Uses the Datagma API to enrich company data based on the extracted domain.
- **1.4 Data Simplification & Qualification:** Simplifies Datagma’s complex response and applies qualification logic to route leads.
- **1.5 Response Delivery:** Sends the qualification result back to Webflow for display on the website.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block receives incoming POST requests from Webflow form submissions.
- **Nodes Involved:** `Receive form submission from Webflow`

##### Node: Receive form submission from Webflow
- **Type:** Webhook node
- **Role:** Entry point to the workflow; listens for POST requests at the path `6545426b-ff78-47af-8e20-a6e9f5259c8e`.
- **Configuration:** Configured with HTTP POST method, response mode set to wait for a response node before replying.
- **Input:** External HTTP POST payload from Webflow form submission.
- **Output:** Emits the full JSON body, including the submitted email.
- **Version Requirements:** Compatible with n8n versions supporting webhook nodes.
- **Edge Cases:** Invalid or missing payload; webhook authentication is not configured, so public exposure should be considered.
- **Sub-workflows:** None.

---

#### 2.2 Email Domain Extraction & Validation

- **Overview:** Extracts the email domain from the form submission and checks if it is a free email provider.
- **Nodes Involved:** `Get domain from email`, `Verify professional email`

##### Node: Get domain from email
- **Type:** Set node
- **Role:** Extract the domain part from the submitted email address.
- **Configuration:** Uses expression `{{$json.body.email.split("@")[1]}}` to split the email string and assign the domain to a new field `domain`.
- **Input:** Output from the webhook node.
- **Output:** Adds a `domain` string field to the JSON.
- **Edge Cases:** Emails without '@' character might cause undefined or error; no explicit validation for email format.
- **Sub-workflows:** None.

##### Node: Verify professional email
- **Type:** Code node (JavaScript)
- **Role:** Check if the extracted domain is part of an extensive predefined list of free or disposable email providers.
- **Configuration:** Contains a hardcoded large array of free email domains; returns a boolean `free_email`.
- **Input:** Receives `domain` field from the previous node.
- **Output:** Adds `free_email` boolean field indicating whether the email is from a free provider.
- **Edge Cases:** Domain case sensitivity handled via `.toLowerCase()`. If domain is missing or malformed, may cause logic to fail or return false negatives.
- **Sub-workflows:** None.

---

#### 2.3 Data Enrichment

- **Overview:** Enriches the lead's company data by querying the Datagma API using the extracted domain.
- **Nodes Involved:** `Enrich with Datagma`

##### Node: Enrich with Datagma
- **Type:** HTTP Request node
- **Role:** Sends a GET request to Datagma’s API endpoint with query parameters including the email domain and API key.
- **Configuration:** 
  - URL: `https://gateway.datagma.net/api/ingress/v2/full`
  - Query parameters:
    - `data` set to the extracted domain
    - `companyPremium=true`, `companyFull=true`, `companyEmployees=false`
    - `employeeCountry=US`
    - `apiId` set to the user’s Datagma API key (must replace `"YOUR_API_KEY"` with a valid key)
  - Headers: Accepts JSON.
- **Input:** Uses the domain from the previous node.
- **Output:** Receives detailed company data JSON from Datagma.
- **Edge Cases:** 
  - API key missing or invalid leads to authentication errors.
  - Network timeouts or rate limits.
  - Unexpected API response structure may cause errors downstream.
- **Sub-workflows:** None.

---

#### 2.4 Data Simplification & Qualification

- **Overview:** Simplifies the complex Datagma response to extract key fields and applies business rules to qualify leads for routing.
- **Nodes Involved:** `Simplify Datagma Output`, `Qualify Account`

##### Node: Simplify Datagma Output
- **Type:** Set node
- **Role:** Extracts relevant company information and the free email flag into a simpler JSON structure.
- **Configuration:** 
  - Extracts and converts fields such as:
    - `company_size` by parsing `employeesAmountInLinkedin` string after removing spaces.
    - `industry`, `founded`, `linkedin Url`, `company_description`, `funding_amount`, `company_revenue`, `companyName`.
    - `free_mail_provider` from the output of `Verify professional email`.
- **Input:** Receives full Datagma response and `free_email` flag.
- **Output:** Simplified JSON with essential company details and free email status.
- **Edge Cases:** Missing or unexpected fields in Datagma response may cause parsing or undefined errors; `fundingRoundsList[2]` assumes at least 3 funding rounds exist.
- **Sub-workflows:** None.

##### Node: Qualify Account
- **Type:** Code node (JavaScript)
- **Role:** Implements lead qualification logic based on company size.
- **Configuration:** 
  - Default `result` is set to `2` (group demo).
  - If company size (from simplified data) is greater than 100, sets `result` to `1` (one-on-one demo).
  - Designed to be customized for other criteria.
- **Input:** Simplified company data including `company_size`.
- **Output:** Adds `result` field to JSON guiding lead routing.
- **Edge Cases:** Missing or invalid `company_size` defaults to group demo; no validation for non-numeric values.
- **Sub-workflows:** None.

---

#### 2.5 Response Delivery

- **Overview:** Sends the lead qualification result back to the Webflow site to dynamically update the user experience.
- **Nodes Involved:** `Send result to Webflow`

##### Node: Send result to Webflow
- **Type:** Respond to Webhook node
- **Role:** Returns a JSON response containing the lead routing result back to the original Webflow form submission request.
- **Configuration:** 
  - HTTP status code 200.
  - CORS headers allowing all origins and standard headers/methods.
  - Response body JSON: `{"result": <qualification result>}`.
- **Input:** Receives the output from `Qualify Account`.
- **Output:** HTTP response to Webflow.
- **Edge Cases:** If upstream nodes fail or do not return `result`, response may be empty or invalid. CORS policy is permissive.
- **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                         | Input Node(s)                      | Output Node(s)                | Sticky Note                                                                                             |
|-------------------------------|------------------------|---------------------------------------|----------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------|
| Receive form submission from Webflow | Webhook                | Entry point, receives form submissions | None                             | Get domain from email         |                                                                                                       |
| Get domain from email          | Set                    | Extract domain from email              | Receive form submission from Webflow | Verify professional email     |                                                                                                       |
| Verify professional email      | Code                   | Check if domain is free email provider | Get domain from email            | Enrich with Datagma           |                                                                                                       |
| Enrich with Datagma            | HTTP Request           | Enrich company data via Datagma API   | Verify professional email        | Simplify Datagma Output       | Add your own Datagma API key here. Get your key here: https://app.datagma.com/user-api                |
| Simplify Datagma Output        | Set                    | Simplify Datagma response              | Enrich with Datagma              | Qualify Account              |                                                                                                       |
| Qualify Account                | Code                   | Apply qualification logic              | Simplify Datagma Output          | Send result to Webflow        | Tweak the code to fit your own criteria. Qualified leads have more than 100 employees in this example. |
| Send result to Webflow         | Respond to Webhook     | Return routing decision to Webflow     | Qualify Account                 | None                         |                                                                                                       |
| Sticky Note                   | Sticky Note            | Workflow overview and guide            | None                             | None                         | This workflow will allow you to enrich in real-time a form submission from Webflow. See full guide: https://lempire.notion.site/Real-time-lead-routing-9fc55c9a5a17415ba736cbdbf5d43a30?pvs=4 |
| Sticky Note1                  | Sticky Note            | Reminder for Datagma API key           | None                             | None                         | Add your own Datagma API key here. Get your key here: https://app.datagma.com/user-api               |
| Sticky Note2                  | Sticky Note            | Qualification logic note                | None                             | None                         | Tweak the code to fit your own criteria. In this example, qualified leads have more than 100 employees. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node:**
   - Name: `Receive form submission from Webflow`.
   - Set HTTP Method to `POST`.
   - Set the webhook path (e.g., `6545426b-ff78-47af-8e20-a6e9f5259c8e`).
   - Response mode: Wait for response node.

2. **Add a Set node:**
   - Name: `Get domain from email`.
   - Add a string field named `domain`.
   - Use expression: `{{$json.body.email.split("@")[1]}}` to extract domain from the email submitted.
   - Connect output from webhook node.

3. **Add a Code node:**
   - Name: `Verify professional email`.
   - Paste JavaScript code that:
     - Maintains a large list of free email domains.
     - Checks if `domain` is in the list, returns `free_email` boolean.
   - Connect output from `Get domain from email`.

4. **Add an HTTP Request node:**
   - Name: `Enrich with Datagma`.
   - Method: GET.
   - URL: `https://gateway.datagma.net/api/ingress/v2/full`.
   - Query parameters:
     - `data`: Use expression: `{{$json.domain}}`
     - `companyPremium`: `true`
     - `companyFull`: `true`
     - `companyEmployees`: `false`
     - `employeeCountry`: `US`
     - `apiId`: Your Datagma API key (replace placeholder).
   - Headers: `Accept: application/json`.
   - Connect output from `Verify professional email`.

5. **Add a Set node:**
   - Name: `Simplify Datagma Output`.
   - Map fields from Datagma response to simpler fields:
     - `company_size`: Parse integer from `$json.company.premium.employeesAmountInLinkedin` (remove spaces).
     - `industry`: `$json.company.premium.industries`
     - `founded`: `$json.company.premium.founded`
     - `linkedin Url`: `$json.company.premium.url`
     - `company_description`: `$json.company.premium.about`
     - `funding_amount`: `$json.company.full.cards.fundingRoundsList[2].moneyRaised.value`
     - `company_revenue`: `$json.company.full.cards.overviewFields.revenueRange`
     - `companyName`: `$json.company.premium.name`
     - `free_mail_provider`: Pull from `free_email` field output from `Verify professional email`.
   - Connect output from `Enrich with Datagma`.

6. **Add a Code node:**
   - Name: `Qualify Account`.
   - JavaScript logic:
     - Default `result` = 2 (group demo).
     - If `company_size` > 100, set `result` = 1 (one-on-one demo).
   - Connect output from `Simplify Datagma Output`.

7. **Add a Respond to Webhook node:**
   - Name: `Send result to Webflow`.
   - Set response code: 200.
   - Set response headers:
     - `Access-Control-Allow-Origin`: `*`
     - `Access-Control-Allow-Headers`: `Content-Type`
     - `Access-Control-Allow-Methods`: `GET, POST`
   - Response body: JSON with field `result` from the incoming JSON.
     - Expression: `={"result":{{$json["result"]}}}`
   - Connect output from `Qualify Account`.

8. **Connect nodes as described:**
   - Webhook → Get domain from email → Verify professional email → Enrich with Datagma → Simplify Datagma Output → Qualify Account → Send result to Webflow.

9. **Credentials setup:**
   - No special credentials needed except for the Datagma API key which must be inserted directly into the HTTP Request node’s query parameters (replace `"YOUR_API_KEY"` with your actual key).

10. **Optional: Add Sticky Notes for documentation:**
    - Add notes for API key instructions and qualification logic explanations.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow will allow you to enrich in real-time a form submission from Webflow. Based on the result, a specific Calendly link will be shown on the website. | Full guide: https://lempire.notion.site/Real-time-lead-routing-9fc55c9a5a17415ba736cbdbf5d43a30?pvs=4                                          |
| Add your own Datagma API key here. Get your key here: https://app.datagma.com/user-api                                                                    | Datagma API key management                                                                                                                  |
| Tweak the qualification code to fit your own criteria. In this example, leads with more than 100 employees are routed to a one-on-one demo.                 | Qualification logic can be customized to business needs                                                                                      |

---

This reference fully documents the “Real-time lead routing in Webflow” workflow, providing detailed insights into its structure, logic, and implementation to enable reproduction, modification, and troubleshooting.