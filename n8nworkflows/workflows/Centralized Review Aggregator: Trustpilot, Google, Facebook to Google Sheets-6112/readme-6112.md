Centralized Review Aggregator: Trustpilot, Google, Facebook to Google Sheets

https://n8nworkflows.xyz/workflows/centralized-review-aggregator--trustpilot--google--facebook-to-google-sheets-6112


# Centralized Review Aggregator: Trustpilot, Google, Facebook to Google Sheets

### 1. Workflow Overview

This workflow, titled **"MULTI-PLATFORM REVIEW AGGREGATOR"**, centralizes customer reviews from multiple online platforms—Trustpilot, Google, and Facebook—into a unified Google Sheets database. It automates daily synchronization of product reviews to facilitate streamlined monitoring and analysis.

The logical flow is organized into the following blocks:

- **1.1 Scheduled Trigger & Product Retrieval:** Initiates the workflow daily and fetches product data from Shopify to identify products for review aggregation.
- **1.2 Multi-Platform Review Fetching:** Concurrently retrieves reviews from Trustpilot, Google, and Facebook APIs for the identified products.
- **1.3 Data Normalization & Filtering:** Processes and normalizes the heterogeneous review data into a consistent format and filters out empty or missing review sets.
- **1.4 Data Storage:** Stores the consolidated, normalized reviews into a Google Sheets spreadsheet serving as a centralized review database.

Supporting this core logic are several sticky notes providing contextual information about workflow features, API requirements, platform flow, and business value.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Product Retrieval

**Overview:**  
This block triggers the workflow on a daily schedule and retrieves product information from Shopify to know which products’ reviews to fetch.

**Nodes Involved:**  
- Daily Review Sync  
- Get Products

**Node Details:**

- **Daily Review Sync**  
  - *Type & Role:* Schedule Trigger node; initiates workflow execution automatically on a defined schedule.  
  - *Configuration:* Default scheduling parameters (likely daily, exact time not specified).  
  - *Inputs/Outputs:* No input; outputs to "Get Products".  
  - *Potential Failures:* Scheduling misconfigurations; system downtime.  
  - *Version:* 1.2

- **Get Products**  
  - *Type & Role:* Shopify node; fetches product data from Shopify store.  
  - *Configuration:* Uses Shopify credentials; default query to fetch all products.  
  - *Inputs/Outputs:* Input from "Daily Review Sync"; outputs to three parallel HTTP Request nodes for fetching reviews.  
  - *Potential Failures:* API authentication errors; rate limits; empty product list.  
  - *Version:* 1

---

#### 2.2 Multi-Platform Review Fetching

**Overview:**  
This block concurrently fetches reviews from Trustpilot, Google, and Facebook APIs using HTTP Request nodes, based on products fetched previously.

**Nodes Involved:**  
- Fetch Trustpilot Reviews  
- Fetch Google Reviews  
- Fetch Facebook Reviews

**Node Details:**

- **Fetch Trustpilot Reviews**  
  - *Type & Role:* HTTP Request node; calls Trustpilot API to retrieve reviews.  
  - *Configuration:* Configured with Trustpilot API endpoint and authorization (likely API key or OAuth).  
  - *Inputs/Outputs:* Input from "Get Products"; output to "Normalize Review Data".  
  - *Variables:* Likely uses expressions to insert product IDs or identifiers into API request URLs.  
  - *Potential Failures:* API authentication failure; rate limiting; invalid product data; network issues.  
  - *Version:* 4.2

- **Fetch Google Reviews**  
  - *Type & Role:* HTTP Request node; calls Google Places or My Business API to retrieve reviews.  
  - *Configuration:* Configured with Google API endpoint and OAuth2 or API key credentials.  
  - *Inputs/Outputs:* Input from "Get Products"; output to "Normalize Review Data".  
  - *Variables:* Uses product/location identifiers for API calls.  
  - *Potential Failures:* OAuth token expiry; API quota exceeded; malformed requests.  
  - *Version:* 4.2

- **Fetch Facebook Reviews**  
  - *Type & Role:* HTTP Request node; calls Facebook Graph API to retrieve page reviews.  
  - *Configuration:* Uses Facebook access tokens and relevant page IDs.  
  - *Inputs/Outputs:* Input from "Get Products"; output to "Normalize Review Data".  
  - *Variables:* Dynamic insertion of page or product IDs.  
  - *Potential Failures:* Token expiration; permission errors; API limits.  
  - *Version:* 4.2

---

#### 2.3 Data Normalization & Filtering

**Overview:**  
This block standardizes the diverse review formats from each platform into a unified schema and filters out cases where no reviews are found.

**Nodes Involved:**  
- Normalize Review Data  
- Check if Reviews Found

**Node Details:**

- **Normalize Review Data**  
  - *Type & Role:* Code node; executes custom JavaScript to transform disparate review data into a consistent structure.  
  - *Configuration:* Contains code that maps fields like reviewer name, rating, review text, date, and source platform into a normalized format.  
  - *Inputs/Outputs:* Receives review data from all three HTTP Request nodes; outputs to "Check if Reviews Found".  
  - *Expressions:* Likely references input JSON data arrays and uses array/map functions for transformation.  
  - *Potential Failures:* Syntax or runtime errors in code; unexpected API response formats; empty data handling.  
  - *Version:* 2

- **Check if Reviews Found**  
  - *Type & Role:* If node; conditional branching based on whether normalized reviews exist.  
  - *Configuration:* Checks existence and non-empty status of review data array.  
  - *Inputs/Outputs:* Input from "Normalize Review Data"; outputs to "Store Reviews Database" if reviews exist; otherwise ends flow.  
  - *Potential Failures:* Expression evaluation errors; false negatives if data format changes.  
  - *Version:* 2

---

#### 2.4 Data Storage

**Overview:**  
This block writes the consolidated and filtered review data into a Google Sheets spreadsheet to maintain an up-to-date review database.

**Nodes Involved:**  
- Store Reviews Database

**Node Details:**

- **Store Reviews Database**  
  - *Type & Role:* Google Sheets node; appends or updates rows in a specified Google Sheets document.  
  - *Configuration:* Uses Google OAuth2 credentials; targets a specific spreadsheet and worksheet; maps normalized review fields to sheet columns.  
  - *Inputs/Outputs:* Input from "Check if Reviews Found"; no outputs (workflow ends here).  
  - *Potential Failures:* Authentication errors; quota exceeded; invalid spreadsheet ID or permissions; data mapping errors.  
  - *Version:* 4.5

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                      | Input Node(s)                  | Output Node(s)                            | Sticky Note                                |
|-------------------------|---------------------|------------------------------------|-------------------------------|------------------------------------------|--------------------------------------------|
| Daily Review Sync       | Schedule Trigger     | Triggers workflow daily             | —                             | Get Products                             |                                            |
| Get Products            | Shopify             | Fetches product list from Shopify  | Daily Review Sync              | Fetch Trustpilot Reviews, Fetch Google Reviews, Fetch Facebook Reviews |                                            |
| Fetch Trustpilot Reviews| HTTP Request        | Retrieves Trustpilot reviews        | Get Products                  | Normalize Review Data                    |                                            |
| Fetch Google Reviews    | HTTP Request        | Retrieves Google reviews            | Get Products                  | Normalize Review Data                    |                                            |
| Fetch Facebook Reviews  | HTTP Request        | Retrieves Facebook reviews          | Get Products                  | Normalize Review Data                    |                                            |
| Normalize Review Data   | Code                | Normalizes review data structure    | Fetch Trustpilot Reviews, Fetch Google Reviews, Fetch Facebook Reviews | Check if Reviews Found                   |                                            |
| Check if Reviews Found  | If                  | Checks if reviews exist             | Normalize Review Data          | Store Reviews Database                   |                                            |
| Store Reviews Database  | Google Sheets       | Stores reviews into Google Sheets   | Check if Reviews Found         | —                                        |                                            |
| Workflow Info           | Sticky Note         | Provides workflow context           | —                             | —                                        |                                            |
| Features & Config       | Sticky Note         | Describes features and configuration| —                             | —                                        |                                            |
| Business Value          | Sticky Note         | Explains business value             | —                             | —                                        |                                            |
| Platforms & Flow        | Sticky Note         | Details platforms and flow          | —                             | —                                        |                                            |
| API Requirements        | Sticky Note         | Lists API requirements              | —                             | —                                        |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Review Sync"**  
   - Node Type: Schedule Trigger  
   - Configure to trigger once daily at desired time (e.g., 00:00).  
   - No special credentials needed.

2. **Create "Get Products"**  
   - Node Type: Shopify  
   - Connect input from "Daily Review Sync" output.  
   - Configure with Shopify credentials (API key, password, store URL).  
   - Set operation to fetch all products or targeted products as needed.

3. **Create "Fetch Trustpilot Reviews"**  
   - Node Type: HTTP Request  
   - Connect input from "Get Products" output.  
   - Configure:  
     - HTTP Method: GET  
     - URL: Trustpilot API endpoint for reviews (use expressions to insert product IDs).  
     - Authentication: API key or OAuth as required by Trustpilot.  
     - Headers: Include authorization headers.  
   - Ensure response format is JSON.

4. **Create "Fetch Google Reviews"**  
   - Node Type: HTTP Request  
   - Connect input from "Get Products" output (parallel to Trustpilot node).  
   - Configure:  
     - HTTP Method: GET  
     - URL: Google Places or Google My Business API endpoint (dynamic insertion of place or product IDs).  
     - Authentication: Google OAuth2 credentials configured in n8n.  
     - Headers and parameters per Google API docs.

5. **Create "Fetch Facebook Reviews"**  
   - Node Type: HTTP Request  
   - Connect input from "Get Products" output (parallel to above).  
   - Configure:  
     - HTTP Method: GET  
     - URL: Facebook Graph API endpoint for page reviews with dynamic page/product IDs.  
     - Authentication: Facebook OAuth2 or access token credentials.  
     - Ensure permissions/scopes allow review access.

6. **Create "Normalize Review Data"**  
   - Node Type: Code  
   - Connect inputs from all three HTTP Request nodes.  
   - Write JavaScript to:  
     - Extract relevant fields (reviewer name, rating, comment, date, platform source) from each platform’s API response.  
     - Map into a unified JSON schema.  
     - Handle empty or missing data gracefully.

7. **Create "Check if Reviews Found"**  
   - Node Type: If  
   - Connect input from "Normalize Review Data".  
   - Condition: Check if the normalized reviews array is non-empty (e.g., `{{$json["reviews"] && $json["reviews"].length > 0}}`).  
   - True branch connects to "Store Reviews Database".  
   - False branch ends workflow or branches as desired.

8. **Create "Store Reviews Database"**  
   - Node Type: Google Sheets  
   - Connect input from "Check if Reviews Found" true output.  
   - Configure Google Sheets credentials (OAuth2).  
   - Set operation to append rows.  
   - Specify Spreadsheet ID and Worksheet name.  
   - Map normalized review fields to corresponding sheet columns.

9. **Add Sticky Notes**  
   - Add notes for workflow info, features & config, business value, platforms & flow, and API requirements as documentation inside n8n editor.

10. **Save and Activate Workflow**  
    - Test each node individually for correct API responses and data handling.  
    - Activate workflow for daily operation.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                      |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow aggregates reviews from Trustpilot, Google, and Facebook into Google Sheets daily. | Project description and business context.           |
| Requires valid Shopify credentials for product retrieval.                                   | Shopify API documentation (https://shopify.dev)    |
| Requires API credentials and permissions for Trustpilot, Google, and Facebook APIs.          | Respect API rate limits and OAuth token lifecycles. |
| Google Sheets node requires OAuth2 credentials with write permissions to target spreadsheet. | Google Sheets API docs (https://developers.google.com/sheets/api) |
| Normalization logic is critical to handle platform-specific data schema differences.          | Custom JavaScript code node used for data transformation. |
| Consider adding error handling nodes (e.g., Error Trigger or Set node) for production use.   | Not present in current workflow but recommended.    |

---

**Disclaimer:** The provided content is extracted exclusively from an automated workflow built with n8n, adhering strictly to current content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.