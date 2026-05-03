Enrich Seller Data with Email & Domain Lookup using Bright Data & Google Search

https://n8nworkflows.xyz/workflows/enrich-seller-data-with-email---domain-lookup-using-bright-data---google-search-4072


# Enrich Seller Data with Email & Domain Lookup using Bright Data & Google Search

---

### 1. Workflow Overview

This workflow automates the enrichment of seller data stored in a Postgres database by performing web lookups to find missing contact details such as secondary emails and company domains. It leverages Bright Data’s Web Unlocker to perform Google searches that bypass restrictions and then uses HTML parsing to extract relevant email addresses and domain information.

The workflow is logically grouped into the following blocks:

- **1.1 Trigger & Data Retrieval**: Initiates the process manually or on schedule and fetches batches of seller records from the Postgres database that require enrichment.
- **1.2 Domain & Email Existence Check**: Examines existing data fields (`domain`, `email`) to decide the appropriate search strategy.
- **1.3 Google Search via Bright Data**: Performs Google searches using Bright Data’s Web Unlocker with queries based on available seller information.
- **1.4 HTML Parsing & Email Extraction**: Extracts URLs and email addresses from retrieved search results using HTML extraction and custom code.
- **1.5 Data Validation & Normalization**: Applies logic to verify that extracted domains and emails are relevant to the seller.
- **1.6 Update Postgres Database**: Updates the seller records with enriched data such as new emails and domain names.
- **1.7 Control Flow & Rate Limiting**: Manages batching, waiting between requests, and conditional paths to handle data absence or existing values.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Retrieval

**Overview:**  
Starts the workflow either manually or on a schedule and reads sellers from the Postgres database with missing primary emails to process them in batches.

**Nodes Involved:**  
- `When clicking ‘Test workflow’` (Manual Trigger)  
- `Schedule Trigger` (Scheduled Trigger)  
- `Read the Database` (Postgres Select)  
- `Process by Batch` (SplitInBatches)

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Manual workflow start  
  - Configuration: Default, no parameters  
  - Input: None  
  - Output: Triggers the workflow  
  - Edge Cases: None  
  - Notes: Allows manual testing

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Periodic execution (e.g., every minute)  
  - Configuration: Interval by minutes  
  - Input: None  
  - Output: Triggers workflow periodically  
  - Edge Cases: Timing conflicts if long runs occur

- **Read the Database**  
  - Type: Postgres  
  - Role: Select seller records missing `primary_email`  
  - Configuration: Select from `seller_data` table in `public` schema; limit 60; where `primary_email IS NULL`  
  - Input: Trigger nodes  
  - Output: Items with seller data JSON  
  - Credentials: Postgres connection  
  - Edge Cases: Database connection errors, empty results

- **Process by Batch**  
  - Type: SplitInBatches  
  - Role: Processes sellers 5 at a time to manage load and API limits  
  - Configuration: Batch size = 5  
  - Input: Seller records from DB  
  - Output: Batches of items for downstream processing  
  - Edge Cases: Partial batches, empty batches

---

#### 2.2 Domain & Email Existence Check

**Overview:**  
Checks if the seller record already contains a domain or other key info to decide search strategy and avoid redundant lookups.

**Nodes Involved:**  
- `Switch`  
- `Switch1`

**Node Details:**  

- **Switch**  
  - Type: Switch  
  - Role: Routes based on presence of `domain` and other fields  
  - Configuration:  
    - Output "Domain exists" if `domain` field exists  
    - Output "No domain but with business address and trade name" if no domain but has `business_address` and `trade_name`  
    - Fallback output: "extra"  
  - Input: Items from batch processing  
  - Output: Routes to appropriate Bright Data nodes or DB update nodes  
  - Edge Cases: Missing or malformed fields causing wrong routing

- **Switch1**  
  - Type: Switch  
  - Role: Compares extracted email and root domain to verify consistency  
  - Configuration:  
    - Output "extracted email matches the first two domains" if email includes root domain  
    - Output "domain does not match with extracted email" if email and domain mismatch  
    - Output "no extracted email but with root domain" if email missing but domain present  
    - Fallback output: "extra"  
  - Input: Output from code node that extracts domain/email  
  - Output: Routes to different Postgres update nodes accordingly  
  - Edge Cases: Emails with subdomains, partial matches, or missing data

---

#### 2.3 Google Search via Bright Data

**Overview:**  
Performs Google search queries using Bright Data’s Web Unlocker API to retrieve potential sources of missing emails or domains.

**Nodes Involved:**  
- `BrightData`  
- `BrightData1`

**Node Details:**  

- **BrightData**  
  - Type: Bright Data (community node)  
  - Role: Search Google for `domain + email` when domain exists  
  - Configuration:  
    - URL: `https://www.google.com/search?q={{ $json.domain }}+email`  
    - Zone: `web_unlocker1` (configured Bright Data API zone)  
    - Country: US  
    - Format: JSON  
  - Input: Items routed by `Switch` when domain exists  
  - Output: Raw HTML JSON from Google search  
  - Credentials: Bright Data API key  
  - Edge Cases: API limits, network errors, Captcha bypass failures

- **BrightData1**  
  - Type: Bright Data  
  - Role: Search Google combining seller name, trade name, and business address when domain missing  
  - Configuration:  
    - URL: `https://www.google.com/search?q=email+{{ encodeURIComponent($('Switch').item.json.trade_name || $('Switch').item.json.seller_name) }}+{{ encodeURIComponent($('Switch').item.json.business_address) }}`  
    - Zone, Country, Format same as above  
  - Input: Items routed by `Switch` when domain missing but address/trade name present  
  - Output: Raw HTML JSON  
  - Edge Cases: Same as BrightData node

---

#### 2.4 HTML Parsing & Email Extraction

**Overview:**  
Extracts relevant links and emails from the HTML content of Google search results using CSS selectors and custom JavaScript code.

**Nodes Involved:**  
- `HTML`  
- `HTML1`  
- `Extract Emails`  
- `Code2`  
- `Split Out`  
- `Split Out1`  
- `Filter`  
- `Filter1`  
- `Aggregate`  
- `Aggregate1`

**Node Details:**  

- **HTML**  
  - Type: HTML Extract  
  - Role: Extract Google search result snippets for domain search results  
  - Configuration:  
    - Operation: Extract HTML content from `body`  
    - CSS Selector: `div[jscontroller] .N54PNb` (Google search snippet container)  
    - Return array, skip images  
  - Input: `BrightData` output  
  - Output: JSON array of extracted elements

- **HTML1**  
  - Same as `HTML`, but processes output from `BrightData1`

- **Extract Emails**  
  - Type: Code  
  - Role: Extract emails from concatenated HTML snippet text  
  - Configuration: Uses regex to find email addresses in joined text  
  - Input: Output from `HTML`  
  - Output: Array of emails

- **Code2**  
  - Type: Code  
  - Role: Same function as `Extract Emails` but for `HTML1` output  
  - Input: `HTML1` output JSON  
  - Output: Emails array

- **Split Out & Split Out1**  
  - Type: SplitOut  
  - Role: Iterates over email arrays to process emails individually  
  - Input: Emails array from `Extract Emails`/`Code2`  
  - Output: Individual email items

- **Filter & Filter1**  
  - Type: Filter  
  - Role: Filters emails to keep only those matching the seller's root domain  
  - Input: Individual emails  
  - Output: Filtered emails only

- **Aggregate & Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates filtered email items back into a single array  
  - Input: Filtered emails  
  - Output: Aggregated email arrays

**Edge Cases:**  
- Emails may be malformed or false positives  
- CSS selectors may break if Google changes layout  
- No emails found, empty arrays  
- Multiple emails to choose from

---

#### 2.5 Data Validation & Normalization

**Overview:**  
Processes candidate URLs and emails to determine if they belong to the seller, normalizes domain names, and decides which domain/email to keep.

**Nodes Involved:**  
- `Code`  
- `If2`

**Node Details:**  

- **Code**  
  - Type: Code  
  - Role:  
    - Extracts main domain from URLs  
    - Normalizes seller and trade names (lowercase, remove spaces/special chars)  
    - Checks if website domain matches seller or trade name  
    - Returns root domain if matched, else null  
  - Input: URLs and emails extracted from HTML extraction nodes  
  - Output: Object with `rootDomain`, URLs, extracted emails, and seller details  
  - Edge Cases: Null or undefined URLs, mismatches, error handling returns null domain

- **If2**  
  - Type: If  
  - Role: Checks if database query returned any email data  
  - Configuration: Condition checks if `$json.data[0]?.email` exists  
  - Routes to update or further processing accordingly

---

#### 2.6 Update Postgres Database

**Overview:**  
Writes enriched domain and secondary email information back to the Postgres database.

**Nodes Involved:**  
- `Postgres1`  
- `Postgres2`  
- `Postgres3`  
- `Postgres4`

**Node Details:**  

- **Postgres1**  
  - Type: Postgres  
  - Role: Updates record with new email and domain data after email existence confirmed  
  - Operation: Update on `seller_data` table  
  - Matching by `seller_id`  
  - Input: Data from `Check if email exists` node  
  - Credentials: Postgres account  
  - Edge Cases: DB connection, update conflicts

- **Postgres2**  
  - Type: Postgres  
  - Role: Updates seller record when domain exists but no new email found  
  - Input: From `Switch` fallback route and `Switch1` fallback route  
  - Configuration: Update `seller_data` with `seller_id` only (may be for logging or fallback)

- **Postgres3**  
  - Type: Postgres  
  - Role: Updates record with domain and extracted email when email matches domain  
  - Input: From `Switch1` output "extracted email matches the first two domains"

- **Postgres4**  
  - Type: Postgres  
  - Role: Updates record with domain only in cases of domain/email mismatch or missing email but domain present  
  - Input: From `Switch1` outputs "domain does not match with extracted email" and "no extracted email but with root domain"

**Edge Cases:**  
- Conflicting updates if concurrent runs occur  
- Partial or missing data in update fields

---

#### 2.7 Control Flow & Rate Limiting

**Overview:**  
Manages timing between requests and controls iteration flow to avoid hitting API limits and ensures orderly processing.

**Nodes Involved:**  
- `Wait`  
- `Wait1`

**Node Details:**  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow (no duration set explicitly in JSON, likely default) after DB update before next batch  
  - Ensures pacing between batches

- **Wait1**  
  - Type: Wait  
  - Role: Explicit 2-second pause before running `BrightData1` node  
  - Helps avoid API rate limiting or blocking

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                                    | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                      |
|---------------------------|----------------------------|---------------------------------------------------|-----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger             | Manual start trigger                              | None                              | Read the Database                |                                                                                                |
| Schedule Trigger           | Schedule Trigger           | Scheduled start trigger                           | None                              | Read the Database                |                                                                                                |
| Read the Database          | Postgres                   | Fetch seller records missing primary email       | When clicking ‘Test workflow’, Schedule Trigger | Process by Batch                 | ## Read and iterate through the database                                                      |
| Process by Batch           | SplitInBatches             | Split seller records into batches of 5            | Read the Database                 | Switch                          |                                                                                                |
| Switch                    | Switch                    | Route based on presence of domain and business data | Process by Batch                 | BrightData, BrightData1, Postgres2 | ## Search Domain+Email in Google using Bright Data<br>If the domain exist, use the search query "{{$json.domain}}+email". |
| BrightData                 | Bright Data                | Google search for domain+email queries            | Switch (Domain exists)            | HTML                           |                                                                                                |
| HTML                      | HTML Extract               | Extract HTML snippet elements from Google search | BrightData                      | Extract Emails                  |                                                                                                |
| Extract Emails            | Code                      | Extract emails from HTML snippets                  | HTML                             | Split Out                      |                                                                                                |
| Split Out                 | SplitOut                  | Iterate over extracted emails                      | Extract Emails                   | Filter                        |                                                                                                |
| Filter                    | Filter                    | Filter emails matching the seller domain          | Split Out                       | Aggregate                     |                                                                                                |
| Aggregate                 | Aggregate                 | Aggregate filtered emails into array               | Filter                         | Check if email exists          |                                                                                                |
| Check if email exists     | If                        | Check if email exists in DB                         | Aggregate                     | Postgres1, Wait1               |                                                                                                |
| Postgres1                 | Postgres                  | Update DB with new email and domain                | Check if email exists           | Wait                         |                                                                                                |
| Wait                      | Wait                      | Wait before next batch                              | Postgres1, Postgres2, Postgres3, Postgres4 | Process by Batch              |                                                                                                |
| BrightData1               | Bright Data               | Google search for seller name+address+email       | Wait1                          | HTML1                        |                                                                                                |
| HTML1                     | HTML Extract              | Extract HTML snippet elements from Google search | BrightData1                   | Code2                        |                                                                                                |
| Code2                     | Code                      | Extract emails from HTML snippets                   | HTML1                         | Split Out1                   |                                                                                                |
| Split Out1                | SplitOut                  | Iterate over extracted emails                       | Code2                         | Filter1                      |                                                                                                |
| Filter1                   | Filter                    | Filter emails matching the seller domain           | Split Out1                    | Aggregate1                   |                                                                                                |
| Aggregate1                | Aggregate                 | Aggregate filtered emails into array                | Filter1                      | Edit Fields3                 |                                                                                                |
| Edit Fields3              | Set                       | Assign extracted URLs, emails, and normalize domain | Aggregate1                   | If2                         |                                                                                                |
| If2                       | If                        | Check if DB returned email data                     | Edit Fields3                 | Postgres1, Code              |                                                                                                |
| Code                      | Code                      | Validate domain ownership and normalize names      | If2                          | Switch1                     | ## Clean up the data and save it to Postgres database                                         |
| Switch1                   | Switch                    | Route based on email/domain matching                | Code                        | Postgres3, Postgres4, Postgres2 |                                                                                                |
| Postgres3                 | Postgres                  | Update DB with matched domain and extracted email  | Switch1                     | Wait                       |                                                                                                |
| Postgres4                 | Postgres                  | Update DB with domain only or mismatched email     | Switch1                     | Wait                       |                                                                                                |
| Postgres2                 | Postgres                  | Update DB fallback or no domain cases               | Switch, Switch1              | Wait                       |                                                                                                |
| Wait1                     | Wait                      | Wait 2 seconds before next search                    | Check if email exists         | BrightData1                 |                                                                                                |
| Sticky Note               | Sticky Note               | Note on domain+email search strategy                | None                        | None                       | ## Search Domain+Email in Google using Bright Data<br>If the domain exist, use the search query "{{$json.domain}}+email". |
| Sticky Note1              | Sticky Note               | Note on seller name+address+email search strategy  | None                        | None                       | ## Search Seller Name+Address+Email in Google using Bright Data<br>If the domain exist, use the search query "{{$json.seller_name}}+{{ $json.seller_address }}+email". |
| Sticky Note2              | Sticky Note               | Note on data cleanup and DB save                     | None                        | None                       | ## Clean up the data and save it to Postgres database                                         |
| Sticky Note3              | Sticky Note               | Note on reading and iterating through the database  | None                        | None                       | ## Read and iterate through the database                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`. Leave default config.  
   - Add a **Schedule Trigger** node named `Schedule Trigger`. Configure to run every minute (or desired interval).

2. **Add Postgres Select Node**  
   - Add a **Postgres** node named `Read the Database`.  
   - Configure credentials for your Postgres database.  
   - Operation: `select`.  
   - Table: `seller_data` (schema `public`).  
   - Condition: `primary_email IS NULL`.  
   - Limit: 60 records.  
   - Sort by `seller_id` ascending.

3. **Add SplitInBatches Node**  
   - Add a **SplitInBatches** node named `Process by Batch`.  
   - Connect it to the `Read the Database` node.  
   - Set batch size to 5.

4. **Add Switch Node for Domain Check**  
   - Add a **Switch** node named `Switch`.  
   - Configure 3 outputs:  
     - "Domain exists": Condition — `$json.domain` exists.  
     - "No domain but with business address and trade name": Condition — No domain but both `business_address` and `trade_name` exist.  
     - Fallback: "extra".

5. **Add Bright Data Nodes**  
   - Add **BrightData** node named `BrightData` for "Domain exists" path.  
     - URL: `https://www.google.com/search?q={{ $json.domain }}+email`  
     - Zone: your configured Bright Data Web Unlocker zone  
     - Country: US  
     - Credentials: Bright Data API key  
   - Add **BrightData** node named `BrightData1` for "No domain but with business address and trade name" path.  
     - URL: `https://www.google.com/search?q=email+{{ encodeURIComponent($('Switch').item.json.trade_name || $('Switch').item.json.seller_name) }}+{{ encodeURIComponent($('Switch').item.json.business_address) }}`  
     - Same zone, country, credentials as above.

6. **Add HTML Extract Nodes**  
   - Add **HTML Extract** node named `HTML` connected to `BrightData`.  
     - Operation: `extractHtmlContent`  
     - Data property: `body`  
     - Extraction: CSS selector `div[jscontroller] .N54PNb` (return array, skip images)  
   - Add **HTML Extract** node named `HTML1` connected to `BrightData1` with same settings.

7. **Add Email Extraction Code Nodes**  
   - Add **Code** node `Extract Emails` connected to `HTML`.  
     - JS: Extract emails with regex from joined text content.  
   - Add **Code** node `Code2` connected to `HTML1`.  
     - Same email extraction code.

8. **Add SplitOut Nodes**  
   - Add **SplitOut** nodes `Split Out` (from `Extract Emails`) and `Split Out1` (from `Code2`) to iterate over emails.

9. **Add Filter Nodes**  
   - Add **Filter** nodes `Filter` and `Filter1` after respective SplitOut nodes.  
   - Condition: Email must contain the seller’s root domain (extracted by regex).

10. **Add Aggregate Nodes**  
    - Add **Aggregate** nodes `Aggregate` and `Aggregate1` after respective Filter nodes to combine emails back.

11. **Add If Node to Check Email Existence**  
    - Add **If** node `Check if email exists` connected to `Aggregate`.  
    - Condition: Check if DB query email data exists (`data[0]?.email`).

12. **Add Postgres Update Nodes**  
    - Add **Postgres** nodes `Postgres1`, `Postgres2`, `Postgres3`, and `Postgres4` for various updating conditions.  
    - Configure all to update `seller_data` table matching on `seller_id`.  
    - Map fields accordingly (email, domain, etc.).

13. **Add Wait Nodes**  
    - Add **Wait** node `Wait` connected after all Postgres update nodes to pace processing.  
    - Add **Wait** node `Wait1` (2 seconds) before `BrightData1` to reduce API rate issues.

14. **Add Code Node to Normalize and Validate Domains**  
    - Add **Code** node named `Code` after `If2` (which checks if email data found).  
    - Implement domain extraction and normalization logic, verify domain belongs to seller or trade name.

15. **Add Switch Node `Switch1`**  
    - Use to route based on email/domain matching results.  
    - Configure outputs for matched, mismatched, no email but domain, and fallback.

16. **Add Set Node `Edit Fields3`**  
    - Assign extracted URLs, emails, and normalized domains from previous steps for DB update.

17. **Connect all nodes respecting the logical flow**  
    - Triggers → Read DB → Process batches → Switch → Bright Data search → HTML extract → Email extract → Filter → Aggregate → If check → Postgres update → Wait → Next batch.

18. **Configure Credentials**  
    - Add Postgres credentials.  
    - Add Bright Data API credentials with Bearer token authentication.

19. **Test Workflow**  
    - Manually trigger or schedule and monitor logs for errors or missing data.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Bright Data Web Unlocker setup instructions and zone creation are required to use the Bright Data nodes effectively.     | See setup instructions and Bright Data dashboard documentation                                  |
| Postgres table schema must match the expected fields (see SQL definition in setup instructions).                          | Use provided SQL schema for `seller_data` table                                                 |
| Node `HTML` and `HTML1` rely on Google’s current HTML structure; changes to Google search page may require selector updates. | CSS selector used: `div[jscontroller] .N54PNb`.                                                  |
| Email extraction regex captures most common email formats but may need adjustment for edge cases.                        | Regex: `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`                                         |
| Rate limiting is managed by `Wait` nodes; adjust timing as needed based on API quotas and response times.                |                                                                                                |
| Workflow designed for n8n self-hosted instances with community nodes installed.                                           | Bright Data node requires `n8n-nodes-brightdata` community package                               |
| For enhanced enrichment, consider integrating additional APIs such as Hunter.io or Clearbit for email validation.        | Suggested in customization tips                                                                 |

---

**Disclaimer:** The text provided stems exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---