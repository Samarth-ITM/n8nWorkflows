Automated Weekly Product Promotion Emails for E-commerce with Algolia and Gmail

https://n8nworkflows.xyz/workflows/automated-weekly-product-promotion-emails-for-e-commerce-with-algolia-and-gmail-10730


# Automated Weekly Product Promotion Emails for E-commerce with Algolia and Gmail

### 1. Workflow Overview

This workflow automates the weekly sending of a promotional email newsletter for an e-commerce site powered by Algolia. It targets marketing teams or e-commerce owners who want to regularly promote discounted products to their subscribers without manual intervention or expensive email marketing platforms.

The workflow logically breaks down into these main blocks:

- **1.1 Scheduled Trigger and Date Calculation**  
  Triggered every Sunday at 8:00 AM (with manual run support), it calculates the current promotion week’s date range (Sunday to Saturday) dynamically.

- **1.2 Product Data Retrieval and Processing**  
  Queries Algolia for products currently on sale, extracts and cleans relevant product information to prepare for newsletter inclusion.

- **1.3 Newsletter Composition**  
  Builds an HTML email template embedding the promotion date range and product details.

- **1.4 Subscriber List Retrieval and Email Dispatch**  
  Fetches the subscriber list from Google Sheets and sends the personalized newsletter email to each subscriber via Gmail.

Each block consists of a series of nodes that process data sequentially or in parallel, with intermediate data merges and transformations to prepare the final email content.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Date Calculation

- **Overview:**  
  This block triggers the workflow weekly on Sundays at 8 AM, determines the current day and date, and calculates the valid promotion date range (Sunday to Saturday) for the newsletter. It supports manual runs by dynamically adjusting the date range based on the current or given date.

- **Nodes Involved:**  
  - On sundays 8.00 am (Schedule Trigger)  
  - What day are we ? (Set)  
  - Sunday ? (If)  
  - Output date of today (Set)  
  - Find last Monday's date (DateTime)  
  - Find last Sunday's date (DateTime)  
  - Clean Promo start date (DateTime)  
  - Calculate end of range (DateTime)  
  - Clean Promo end date (DateTime)  
  - Output Promo Validity Date Range (Set)  
  - Merge data for newsletter (Merge)

- **Node Details:**

  - **On sundays 8.00 am**  
    - Type: Schedule Trigger  
    - Configuration: Triggers weekly on Sundays at 8:00 AM.  
    - Inputs: None (start node)  
    - Outputs: Triggers downstream nodes.

  - **What day are we ?**  
    - Type: Set  
    - Role: Extracts and sets date components (day of week, day of month, month, year, timestamp) from the trigger or previous node.  
    - Key expressions: Extracts values like `Day of week`, `Day of month`, `Month`, `Year`, and `timestamp` from incoming JSON.  
    - Input: From schedule trigger  
    - Output: JSON with date info.

  - **Sunday ?**  
    - Type: If  
    - Role: Checks if current day is Sunday to branch the logic accordingly.  
    - Condition: `$json["Day of week"] === "Sunday"`  
    - Input: From "What day are we ?"  
    - Outputs: Two branches (true/false).

  - **Output date of today**  
    - Type: Set  
    - Role: Passes current timestamp forward for date calculations.  
    - Input: "Sunday ?" node (true branch)  
    - Output: JSON with timestamp.

  - **Find last Monday's date**  
    - Type: DateTime  
    - Role: Rounds the current date to the nearest Monday (start of week).  
    - Input: From "Output date of today"  
    - Output: Adds field `lastMondayDate`.

  - **Find last Sunday's date**  
    - Type: DateTime  
    - Role: Subtracts 1 day from `lastMondayDate` to get last Sunday's date.  
    - Input: From "Find last Monday's date"  
    - Output: Adds field `Previous sunday`.

  - **Clean Promo start date**  
    - Type: DateTime  
    - Role: Formats `Previous sunday` date into a readable format (e.g., "Nov-9").  
    - Input: From "Find last Sunday's date"  
    - Output: Formatted start date.

  - **Calculate end of range**  
    - Type: DateTime  
    - Role: Adds 6 days to the promo start date to get the promo end date (Saturday).  
    - Input: From "Clean Promo start date"  
    - Output: New date field `newDate`.

  - **Clean Promo end date**  
    - Type: DateTime  
    - Role: Formats the promo end date to a readable string (e.g., "Nov-15-2025").  
    - Input: From "Calculate end of range"  
    - Output: Formatted end date.

  - **Output Promo Validity Date Range**  
    - Type: Set  
    - Role: Creates a string combining the start and end date to form the validity range for the newsletter (e.g., "Nov-9 to Nov-15-2025").  
    - Input: From "Clean Promo end date"  
    - Output: Field `validityRange`.

  - **Merge data for newsletter**  
    - Type: Merge  
    - Role: This node merges date range data with product data later on for newsletter composition.  
    - Input: One input from date calculation branch, second from product aggregation later.

- **Edge Cases / Potential Failures:**  
  - Date calculation errors due to timezone differences (UTC used to mitigate).  
  - Manual runs might have inconsistent date inputs if not properly handled.  
  - If the schedule trigger fails, workflow won't start automatically.

---

#### 2.2 Product Data Retrieval and Processing

- **Overview:**  
  This block queries Algolia for products tagged as currently on sale, extracts only the necessary fields, and prepares the clean product list for the newsletter.

- **Nodes Involved:**  
  - Request products from Algolia (HTTP Request)  
  - Extract Discounted Products (SplitOut)  
  - Keep only useful product data (Set)  
  - Put back all hits in one clean array (Aggregate)

- **Node Details:**

  - **Request products from Algolia**  
    - Type: HTTP Request  
    - Configuration: POST request to Algolia API endpoint querying products with `on_sale:true` filter. Requests 6 hits per query, retrieving fields: `name`, `original_price_eur`, `price_eur`, `description`, `image`.  
    - Authentication: Custom HTTP header with Algolia API credentials.  
    - Input: Triggered by schedule trigger node.  
    - Output: Full Algolia JSON response.

  - **Extract Discounted Products**  
    - Type: SplitOut  
    - Role: Splits the JSON array under the `hits` key into individual product items for processing.  
    - Input: From Algolia HTTP response.  
    - Output: Items each representing one product.

  - **Keep only useful product data**  
    - Type: Set  
    - Role: Filters and reformats product data to only include necessary fields for the newsletter: `name`, `price_eur`, `description`, `objectID`, `image`, `original_price_eur`.  
    - Input: From split products.  
    - Output: Clean product JSON objects.

  - **Put back all hits in one clean array**  
    - Type: Aggregate  
    - Role: Aggregates cleaned product JSONs back into a single array under the key `hits`.  
    - Input: From cleaned product data.  
    - Output: Aggregated array of products.

- **Edge Cases / Potential Failures:**  
  - API request failures (network issues, auth errors).  
  - Algolia index misconfiguration or missing fields.  
  - Empty product list if no products are on sale.  
  - Data inconsistency if fields are missing or malformed.

---

#### 2.3 Newsletter Composition

- **Overview:**  
  This block composes the HTML newsletter using a custom template, embedding the promotion date range and the clean product data array to generate a complete, styled promotional email.

- **Nodes Involved:**  
  - Merge data for newsletter (Merge)  
  - Aggregate data for Newsletter (Aggregate)  
  - Newsletter Generation (HTML Format) (HTML)

- **Node Details:**

  - **Merge data for newsletter**  
    - Type: Merge  
    - Role: Combines the promo date range data and the aggregated product data into one JSON object for use in the HTML template.  
    - Input: Two inputs — one from date range output, one from product data aggregation.  
    - Output: Merged JSON with `validityRange` and `hits` array.

  - **Aggregate data for Newsletter**  
    - Type: Aggregate  
    - Role: Ensures all merged data is collected into a single item for HTML processing (wraps in `dataForNewsletter`).  
    - Input: From merge node.  
    - Output: Single JSON item for the newsletter node.

  - **Newsletter Generation (HTML Format)**  
    - Type: HTML  
    - Role: Contains a comprehensive, responsive HTML email template for the newsletter. It uses dynamic placeholders to insert the promotion date range and product details (name, description, price, image).  
    - The HTML template is designed for compatibility with major email clients, includes inline styles, and renders a grid of products with images and prices.  
    - Input: Aggregated newsletter data.  
    - Output: JSON containing an `html` field with the fully rendered newsletter content.

- **Edge Cases / Potential Failures:**  
  - Template rendering errors if expected JSON structure is missing or malformed.  
  - HTML email compatibility issues with some email clients (template is carefully designed to minimize this).  
  - Large emails may be flagged by spam filters.

---

#### 2.4 Subscriber List Retrieval and Email Dispatch

- **Overview:**  
  This block retrieves the list of newsletter subscribers from a Google Sheet and sends the generated newsletter HTML email to each subscriber's email address via Gmail.

- **Nodes Involved:**  
  - Get row(s) in sheet (Google Sheets)  
  - Send newsletter (Gmail)

- **Node Details:**

  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Retrieves all rows from the configured sheet containing subscriber emails.  
    - Sheet: First sheet (`gid=0`) in a specific Google Sheets document.  
    - Credentials: OAuth2 credentials for Google Sheets access.  
    - Input: From the newsletter HTML node.  
    - Output: Items each containing one subscriber email.

  - **Send newsletter**  
    - Type: Gmail  
    - Role: Sends the newsletter email to each subscriber email address retrieved.  
    - Email `To`: Extracted dynamically from the subscriber row (`Email` column).  
    - Email `Subject`: Includes the promotion validity date range dynamically.  
    - Email `Body`: Uses the rendered HTML from the newsletter generation node.  
    - Credentials: OAuth2 Gmail account credentials.  
    - Input: From Google Sheets node.  
    - Output: Email sending status.

- **Edge Cases / Potential Failures:**  
  - Google Sheets API rate limits or auth errors.  
  - Empty or malformed subscriber email addresses.  
  - Gmail sending limits or auth errors.  
  - Emails marked as spam or undelivered.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                               | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                                                         |
|--------------------------------|---------------------|----------------------------------------------|----------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| On sundays 8.00 am             | Schedule Trigger    | Weekly trigger every Sunday 8 AM              | -                                | Request products from Algolia, What day are we ? | Every Week Sends a weekly newsletter on Sundays at 8.00 am containing discounted products for the coming week                       |
| Request products from Algolia  | HTTP Request       | Queries Algolia for discounted products       | On sundays 8.00 am               | Extract Discounted Products          | Gather products currently in promotion from Algolia - Get from Algolia all the products currently in promotion...                  |
| Extract Discounted Products    | SplitOut           | Splits Algolia hits array into individual products | Request products from Algolia    | Keep only useful product data        | Gather products currently in promotion from Algolia                                                                                 |
| Keep only useful product data  | Set                | Filters product data to essential fields      | Extract Discounted Products      | Put back all hits in one clean array | Gather products currently in promotion from Algolia                                                                                 |
| Put back all hits in one clean array | Aggregate          | Aggregates all product data back into array   | Keep only useful product data    | Merge data for newsletter (input 2) | Gather products currently in promotion from Algolia                                                                                 |
| What day are we ?              | Set                | Extracts day, month, year, timestamp info     | On sundays 8.00 am               | Sunday ?                            | Determine the valid date range for the promotion (strictly no-code) - This branch determines the current day of the week...          |
| Sunday ?                      | If                 | Checks if today is Sunday                      | What day are we ?                | Output date of today, Find last Monday's date | Determine the valid date range for the promotion                                                                                     |
| Output date of today           | Set                | Passes current timestamp forward               | Sunday ?                        | Find last Monday's date             | Determine the valid date range for the promotion                                                                                     |
| Find last Monday's date        | DateTime           | Rounds date to last Monday                      | Output date of today             | Find last Sunday's date             | Determine the valid date range for the promotion                                                                                     |
| Find last Sunday's date        | DateTime           | Calculates last Sunday from Monday             | Find last Monday's date          | Clean Promo start date              | Determine the valid date range for the promotion                                                                                     |
| Clean Promo start date         | DateTime           | Formats promo start date                        | Find last Sunday's date          | Calculate end of range              | Determine the valid date range for the promotion                                                                                     |
| Calculate end of range         | DateTime           | Adds 6 days to start date to get promo end     | Clean Promo start date           | Clean Promo end date                | Determine the valid date range for the promotion                                                                                     |
| Clean Promo end date           | DateTime           | Formats promo end date                          | Calculate end of range           | Output Promo Validity Date Range    | Determine the valid date range for the promotion                                                                                     |
| Output Promo Validity Date Range | Set                | Creates combined date range string              | Clean Promo end date             | Merge data for newsletter (input 1) | Promo Validity Date (date range)                                                                                                    |
| Merge data for newsletter      | Merge              | Combines promo date range and product data     | Output Promo Validity Date Range, Put back all hits in one clean array | Aggregate data for Newsletter        | -                                                                                                                                   |
| Aggregate data for Newsletter  | Aggregate          | Wraps merged data for newsletter rendering     | Merge data for newsletter        | Newsletter Generation (HTML Format) | Newsletter prepared - ![Newsletter structure](https://gkfvbshnthawbcaugpwp.supabase.co/storage/v1/object/public/n8n-diagrams/2025%20Nov%2011%20-%20Newsletter%20full.png) |
| Newsletter Generation (HTML Format) | HTML               | Builds final HTML newsletter with dynamic data | Aggregate data for Newsletter    | Get row(s) in sheet                | Load HTML Template - This node contains the HTML newsletter template for Dog Treats...                                               |
| Get row(s) in sheet            | Google Sheets      | Retrieves subscriber emails from Google Sheet  | Newsletter Generation (HTML Format) | Send newsletter                   | Get Newsletter subscribers - Sheet containing the updated list of all subscribers to this newsletter.                                |
| Send newsletter               | Gmail              | Sends newsletter email to each subscriber       | Get row(s) in sheet             | -                                   | Send Newsletter to subscribers - Send the answer to the customer                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `On sundays 8.00 am`  
   - Type: Schedule Trigger  
   - Configure: Set to trigger every week on Sunday at 8:00 AM.

2. **Create HTTP Request Node to Query Algolia**  
   - Name: `Request products from Algolia`  
   - Type: HTTP Request  
   - Configure:  
     - Method: POST  
     - URL: `https://<ALGOLIA_APP_ID>-dsn.algolia.net/1/indexes/<INDEX_NAME>/query` (replace placeholders)  
     - Body Type: JSON  
     - Body:  
       ```json
       {
         "filters": "on_sale:true",
         "hitsPerPage": 6,
         "attributesToRetrieve": ["name","original_price_eur", "price_eur", "description", "image"]
       }
       ```  
     - Authentication: Custom HTTP header with Algolia API credentials (Application ID and API key).

3. **Add SplitOut Node**  
   - Name: `Extract Discounted Products`  
   - Type: SplitOut  
   - Configure: Split on field `hits`.

4. **Add Set Node to Keep Useful Product Data**  
   - Name: `Keep only useful product data`  
   - Type: Set  
   - Assign fields: `name`, `price_eur`, `description`, `objectID`, `image`, `original_price_eur` mapped from incoming JSON.

5. **Add Aggregate Node**  
   - Name: `Put back all hits in one clean array`  
   - Type: Aggregate  
   - Configure: Aggregate all items into one array under field `hits`.

6. **Create Set Node to Extract Date Info**  
   - Name: `What day are we ?`  
   - Type: Set  
   - Assign variables: `Day of week`, `Day of month`, `Month`, `Year`, `timestamp` from the incoming trigger data.

7. **Add If Node to Check if Sunday**  
   - Name: `Sunday ?`  
   - Type: If  
   - Condition: Check if `Day of week` equals `"Sunday"`.

8. **Add Set Node to Output Current Timestamp**  
   - Name: `Output date of today`  
   - Type: Set  
   - Assign `timestamp` field for date downstream calculations.

9. **Add DateTime Nodes for Date Calculations:**  
   - `Find last Monday's date`: Round current date to nearest Monday.  
   - `Find last Sunday's date`: Subtract 1 day from last Monday.  
   - `Clean Promo start date`: Format last Sunday as "LLL-d".  
   - `Calculate end of range`: Add 6 days to promo start date.  
   - `Clean Promo end date`: Format promo end date as "LLL-d-y".

10. **Add Set Node to Create Validity Range**  
    - Name: `Output Promo Validity Date Range`  
    - Assign string combining start and end dates: `{{ Clean Promo start date.formattedDate }} to {{ Clean Promo end date.formattedDate }}`.

11. **Add Merge Node**  
    - Name: `Merge data for newsletter`  
    - Merge the promo date range data and aggregated product data (`Put back all hits in one clean array`) using a merge node in 'Wait for Both Inputs' mode.

12. **Add Aggregate Node for Final Newsletter Data**  
    - Name: `Aggregate data for Newsletter`  
    - Aggregate all merged data into a single JSON item under `dataForNewsletter`.

13. **Add HTML Node to Generate Newsletter**  
    - Name: `Newsletter Generation (HTML Format)`  
    - Paste the full HTML newsletter template.  
    - Use expressions to inject date range and product fields dynamically from `dataForNewsletter`.

14. **Add Google Sheets Node to Get Subscribers**  
    - Name: `Get row(s) in sheet`  
    - Configure: Connect to Google Sheets document and sheet containing subscriber emails.  
    - Credentials: Google OAuth2 credential.

15. **Add Gmail Node to Send Emails**  
    - Name: `Send newsletter`  
    - Configure:  
      - To: Expression to get subscriber email from Google Sheets row.  
      - Subject: Include dynamic promo validity range.  
      - Message: Use HTML output from newsletter generation node.  
    - Credentials: Gmail OAuth2 credential.

16. **Connect Nodes in Order:**  
    - `On sundays 8.00 am` → `Request products from Algolia` → `Extract Discounted Products` → `Keep only useful product data` → `Put back all hits in one clean array` → `Merge data for newsletter` (input 2)  
    - `On sundays 8.00 am` → `What day are we ?` → `Sunday ?` → true branch → `Output date of today` → `Find last Monday's date` → `Find last Sunday's date` → `Clean Promo start date` → `Calculate end of range` → `Clean Promo end date` → `Output Promo Validity Date Range` → `Merge data for newsletter` (input 1)  
    - `Merge data for newsletter` → `Aggregate data for Newsletter` → `Newsletter Generation (HTML Format)` → `Get row(s) in sheet` → `Send newsletter`

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow automatically sends a beautifully designed HTML newsletter every Sunday at 8 AM, featuring products currently on sale from your Algolia-powered e-commerce store. Setup time is about 15 minutes with basic credentials and sheet preparation.                                                                                                    | See Sticky Note "Automated Weekly Newsletter for E-commerce Promotions (based on Algolia)"                   |
| The HTML template includes responsive design and inlined styles optimized for most email clients, including Outlook, Gmail, and mobile devices. It uses placeholders for dynamic product data and the promo date range.                                                                                                                                        | See node: Newsletter Generation (HTML Format)                                                                |
| The workflow supports manual runs during the week, recalculating the promo week range accordingly, making testing and staging easier.                                                                                                                                                                                                                           | See Sticky Note about smart date range calculation                                                           |
| To customize: change the Algolia filter (`on_sale:true`), adjust product count (`hitsPerPage`), modify schedule trigger timing, or personalize the HTML template with branding colors.                                                                                                                                                                           | Refer to node "Request products from Algolia" and HTML template node                                          |
| For questions or feedback, contact: emir.belkahia@gmail.com or LinkedIn at [linkedin.com/in/emirbelkahia](https://www.linkedin.com/in/emirbelkahia/)                                                                                                                                                                                                           | See Sticky Note12                                                                                             |
| Newsletter structure diagrams are available for visual reference: [Newsletter structure image](https://gkfvbshnthawbcaugpwp.supabase.co/storage/v1/object/public/n8n-diagrams/2025%20Nov%2011%20-%20Newsletter%20structure%202.png) and [Newsletter full structure](https://gkfvbshnthawbcaugpwp.supabase.co/storage/v1/object/public/n8n-diagrams/2025%20Nov%2011%20-%20Newsletter%20full.png) | Provided in Sticky Notes 5 and 9                                                                              |

---

This document fully describes and analyzes the automated weekly promotional email workflow using Algolia and Gmail in n8n, enabling users to understand, reproduce, and customize the workflow confidently.