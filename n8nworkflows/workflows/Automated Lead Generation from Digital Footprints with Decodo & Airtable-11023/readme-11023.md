Automated Lead Generation from Digital Footprints with Decodo & Airtable

https://n8nworkflows.xyz/workflows/automated-lead-generation-from-digital-footprints-with-decodo---airtable-11023


# Automated Lead Generation from Digital Footprints with Decodo & Airtable

### 1. Workflow Overview

This workflow automates lead generation by leveraging digital footprint data through Decodo’s search and scraping APIs combined with Airtable as a lead database. It targets marketing or sales teams aiming to discover and enrich leads based on specific technology usage and industry filters.

The workflow is logically divided into the following blocks:

- **1.1 Configuration and Initiation:** Manual trigger and parameter setup to define search scope.
- **1.2 Initial Lead Discovery:** Conduct Google search via Decodo, filter and extract unique domain leads.
- **1.3 Lead Database Integrity Check:** Loop over leads, check Airtable for existing records, create or update accordingly.
- **1.4 Conditional Enrichment:** Identify leads missing primary email information and perform deeper data extraction.
- **1.5 Deep Enrichment Pipeline:** Search for contact pages and scrape them to extract high-quality email contacts.
- **1.6 Final Data Consolidation:** Update Airtable records with enriched contact details and finalize lead status.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration and Initiation

**Overview:**  
Sets the initial parameters for searching leads and starts the workflow manually.

**Nodes Involved:**  
- Manual Trigger  
- Config: Set Search Params

**Node Details:**

- **Manual Trigger**  
  - *Type:* Manual trigger node for user-initiated workflow start.  
  - *Config:* No parameters, simply triggers the workflow.  
  - *Input:* None (entry point)  
  - *Output:* Triggers next node.

- **Config: Set Search Params**  
  - *Type:* Set node to define parameters for the Decodo search.  
  - *Config:* Sets custom string values for `tech_footprint` ("We use Klaviyo"), `target_industry` (empty, to be customized), `target_geo` ("United States"), and `target_locale` ("en-US").  
  - *Input:* Triggered by Manual Trigger.  
  - *Output:* Provides parameters for the Google Search node.  
  - *Edge Cases:* If `target_industry` remains empty, search results may be too broad or irrelevant.

---

#### 2.2 Initial Lead Discovery

**Overview:**  
Executes a Google search with Decodo, processes results to extract unique domains, and prepares an array of leads with domain, source URL, title, and lead type.

**Nodes Involved:**  
- Decodo: Google Search  
- Code: Initial Domain Filter

**Node Details:**

- **Decodo: Google Search**  
  - *Type:* Decodo search node performing Google search queries.  
  - *Config:* Uses dynamic query combining `tech_footprint` and `target_industry`, limited to 50 results, geo and locale set from config node, JS rendering disabled (headless=false) for cost efficiency.  
  - *Credentials:* Decodo API key needed.  
  - *Input:* Parameters from Config node.  
  - *Output:* Raw search results JSON.  
  - *Edge Cases:* API quota limits, network errors, incomplete results.

- **Code: Initial Domain Filter**  
  - *Type:* JavaScript code node for parsing Decodo search results.  
  - *Config:* Extracts domain names from search URLs using regex, removes duplicates, assigns lead type as "Paid Ad Lead" or "Organic Lead" based on results origin. Returns an array of objects with `source_url`, `source_title`, `domain`, and `lead_type`.  
  - *Input:* Raw Decodo search results.  
  - *Output:* Filtered list of unique leads.  
  - *Edge Cases:* Malformed URLs, missing fields, unexpected data structure in Decodo results.  
  - *Notes:* Gracefully handles missing data container by returning an error item.

---

#### 2.3 Lead Database Integrity Check

**Overview:**  
Iterates over each lead from the filtered list, checks Airtable if the lead's domain exists, then either creates a new record or updates the existing one.

**Nodes Involved:**  
- Loop: Process Leads  
- Airtable: Check Existence  
- If: Lead Exists?  
- Airtable: Create Lead  
- Airtable: Update Lead  
- Data Merger: ID Finalizer

**Node Details:**

- **Loop: Process Leads**  
  - *Type:* SplitInBatches node to process leads one by one.  
  - *Config:* Defaults, processes each lead individually.  
  - *Input:* Filtered leads array from previous code node.  
  - *Output:* Single lead item for next nodes.  
  - *Edge Cases:* Large number of leads may impact performance or API rate limits.

- **Airtable: Check Existence**  
  - *Type:* Airtable search node.  
  - *Config:* Searches "Leads" table for records matching the current lead's `domain` field using formula filter.  
  - *Credentials:* Airtable Personal Access Token required.  
  - *Input:* Single lead item.  
  - *Output:* Search results indicating lead existence.  
  - *Edge Cases:* Airtable API limits, connection failures.

- **If: Lead Exists?**  
  - *Type:* Conditional node.  
  - *Config:* Checks if the Airtable search result is empty (lead does not exist) or not.  
  - *Input:* Result from Airtable Check.  
  - *Output:*  
    - True branch (lead exists) → Airtable: Update Lead  
    - False branch (new lead) → Airtable: Create Lead

- **Airtable: Create Lead**  
  - *Type:* Airtable create record node.  
  - *Config:* Creates a new record with domain, status ("New Lead"), lead type, dates, and source URL.  
  - *Input:* New lead data.  
  - *Output:* Newly created record data including Airtable ID.

- **Airtable: Update Lead**  
  - *Type:* Airtable update record node.  
  - *Config:* Updates existing record by ID with updated status ("Updated"), lead type, source URL, and updated date.  
  - *Input:* Existing record ID and lead data.  
  - *Output:* Updated record data.

- **Data Merger: ID Finalizer**  
  - *Type:* Set node.  
  - *Config:* Extracts Airtable record ID and domain from the previous node’s output and merges into a single object.  
  - *Input:* Output from Create or Update Airtable nodes.  
  - *Output:* Enriched lead item with Airtable ID for further processing.

---

#### 2.4 Conditional Enrichment

**Overview:**  
Determines if a lead requires further enrichment by checking if the primary email field is empty, and routes leads accordingly.

**Nodes Involved:**  
- If: Enrichment Needed?

**Node Details:**

- **If: Enrichment Needed?**  
  - *Type:* Conditional node.  
  - *Config:* Checks if `Primary Email` field from Airtable is empty.  
  - *Input:* Lead data with Airtable fields merged.  
  - *Output:*  
    - True branch (email missing) → Data Merger: ID Finalizer (to continue enrichment)  
    - False branch (email present) → Loop: Process Leads (skip enrichment, continue loop)  
  - *Edge Cases:* Missing or malformed email fields in Airtable.

---

#### 2.5 Deep Enrichment Pipeline

**Overview:**  
Executes a targeted Decodo Google search for contact/about pages of the lead domain, scrapes the contact page if found, extracts emails using regex, and prioritizes the best contact email.

**Nodes Involved:**  
- Decodo: Email Search  
- Code: Initial Enrichment  
- If: Contact Page Exists?  
- Decodo: Scrape Contact Page  
- Code: Finalize Contact Data  
- If: Contact Email Exists?  
- Airtable: Update Lead Contact Email

**Node Details:**

- **Decodo: Email Search**  
  - *Type:* Decodo Google Search node.  
  - *Config:* Queries site-restricted Google search for URLs containing contact-related keywords on the lead domain, limited to 20 results, configured with geo and locale from config.  
  - *Input:* Lead domain from previous steps.  
  - *Output:* Search results with potential contact page URLs and snippets.

- **Code: Initial Enrichment**  
  - *Type:* JavaScript code node.  
  - *Config:* Parses Decodo search results for emails and contact page URLs using regex patterns. Extracts first found email and first relevant contact page URL.  
  - *Input:* Output from Decodo Email Search node.  
  - *Output:* Enriched lead data with `contact_email` and `contact_page_url` fields.  
  - *Edge Cases:* No emails or contact URLs found, malformed search results.

- **If: Contact Page Exists?**  
  - *Type:* Conditional node.  
  - *Config:* Checks if `contact_page_url` is present and non-empty.  
  - *Input:* Output from Initial Enrichment code node.  
  - *Output:*  
    - True branch → Decodo: Scrape Contact Page  
    - False branch → Loop: Process Leads (skip scraping, continue loop)

- **Decodo: Scrape Contact Page**  
  - *Type:* Decodo scraping node.  
  - *Config:* Scrapes the provided `contact_page_url` to extract page content.  
  - *Credentials:* Decodo API key.  
  - *Input:* Contact page URL from previous node.  
  - *Output:* Raw content of contact page for email extraction.  
  - *Edge Cases:* Page not reachable, scraping errors or delays.

- **Code: Finalize Contact Data**  
  - *Type:* JavaScript code node.  
  - *Config:* Extracts all emails from scraped page content using regex, prioritizes emails by keywords (e.g., sales, contact, founder), and selects the best email.  
  - *Input:* Raw content from scraper node.  
  - *Output:* Lead data enriched with `final_contact_email` and a count of emails found.  
  - *Edge Cases:* No emails found, low-quality data.

- **If: Contact Email Exists?**  
  - *Type:* Conditional node.  
  - *Config:* Checks if `final_contact_email` is non-empty.  
  - *Input:* Output from Finalize Contact Data node.  
  - *Output:*  
    - True branch → Airtable: Update Lead Contact Email  
    - False branch → Loop: Process Leads (skip updating, continue loop)

- **Airtable: Update Lead Contact Email**  
  - *Type:* Airtable update node.  
  - *Config:* Updates Airtable lead record with `Primary Email`, updated status ("Enrichment Complete"), updated date, and `Contact Page URL`.  
  - *Input:* Lead data with Airtable ID and enriched email.  
  - *Output:* Updated Airtable record.  
  - *Edge Cases:* Airtable API failures, data mismatch.

---

#### 2.6 Final Data Consolidation and Loop Continuation

**Overview:**  
Ensures each lead is properly processed and loop continues until all leads are handled.

**Nodes Involved:**  
- Loop: Process Leads (re-entry connections)

**Node Details:**  
- The loop node receives output from various branches (e.g., skipping enrichment, after updating contact email) and continues processing remaining leads.  
- Proper loop closure and error handling ensure workflow stability.

---

### 3. Summary Table

| Node Name                     | Node Type                  | Functional Role                             | Input Node(s)                      | Output Node(s)                               | Sticky Note                                                                                                         |
|-------------------------------|----------------------------|--------------------------------------------|----------------------------------|----------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Manual Trigger                | Manual Trigger             | Initiates the workflow                      | —                                | Config: Set Search Params                     |                                                                                                                     |
| Config: Set Search Params     | Set                       | Defines search parameters                   | Manual Trigger                   | Decodo: Google Search                         | ## Configuration: Set Search Parameters This node is the central control panel for lead generation...               |
| Decodo: Google Search         | Decodo Search             | Performs initial Google search              | Config: Set Search Params        | Code: Initial Domain Filter                    | ## Data Sourcing & Lead List Generation Decodo executes customized query...                                          |
| Code: Initial Domain Filter   | Code                      | Extracts unique domains & lead types        | Decodo: Google Search            | Loop: Process Leads                           | ## Data Sourcing & Lead List Generation Decodo executes customized query...                                          |
| Loop: Process Leads           | SplitInBatches            | Iterates over leads                         | Code: Initial Domain Filter      | Airtable: Check Existence                      | ## Database Integrity Loop The loop processes each lead and runs Find/Create/Update logic...                          |
| Airtable: Check Existence     | Airtable Search           | Checks if lead already exists in DB        | Loop: Process Leads              | If: Lead Exists?                              | ## Database Integrity Loop The loop processes each lead and runs Find/Create/Update logic...                          |
| If: Lead Exists?              | If                        | Conditional branch: lead exists?            | Airtable: Check Existence        | Airtable: Update Lead (true), Airtable: Create Lead (false) | ## Database Integrity Loop The loop processes each lead and runs Find/Create/Update logic...                          |
| Airtable: Create Lead         | Airtable Create           | Creates new lead record                      | If: Lead Exists? (false branch) | Data Merger: ID Finalizer                     | ## Database Integrity Loop The loop processes each lead and runs Find/Create/Update logic...                          |
| Airtable: Update Lead         | Airtable Update           | Updates existing lead record                 | If: Lead Exists? (true branch)  | If: Enrichment Needed?                        | ## Database Integrity Loop The loop processes each lead and runs Find/Create/Update logic...                          |
| Data Merger: ID Finalizer     | Set                       | Extracts Airtable ID & merges lead data     | Airtable: Create Lead, Airtable: Update Lead | Decodo: Email Search, If: Enrichment Needed? | ## Data Consolidation & Continuation ID Finalizer extracts the Airtable Record ID...                                  |
| If: Enrichment Needed?        | If                        | Checks if lead needs enrichment (email empty) | Airtable: Update Lead           | Data Merger: ID Finalizer (true), Loop: Process Leads (false) |                                                                                                                     |
| Decodo: Email Search          | Decodo Search             | Searches for contact/about pages on domain | Data Merger: ID Finalizer        | Code: Initial Enrichment                      | ## Deep Enrichment Pipeline Decodo conditionally scrapes target contact page...                                       |
| Code: Initial Enrichment      | Code                      | Extracts email and contact page URL from snippets | Decodo: Email Search           | If: Contact Page Exists?                      | ## Deep Enrichment Pipeline Decodo conditionally scrapes target contact page...                                       |
| If: Contact Page Exists?      | If                        | Checks if contact page URL exists           | Code: Initial Enrichment         | Decodo: Scrape Contact Page (true), Loop: Process Leads (false) | ## Deep Enrichment Pipeline Decodo conditionally scrapes target contact page...                                       |
| Decodo: Scrape Contact Page   | Decodo Scraper            | Scrapes contact page content                 | If: Contact Page Exists? (true) | Code: Finalize Contact Data                   | ## Deep Enrichment Pipeline Decodo conditionally scrapes target contact page...                                       |
| Code: Finalize Contact Data   | Code                      | Extracts best contact email from scraped content | Decodo: Scrape Contact Page     | If: Contact Email Exists?                      | ## Deep Enrichment Pipeline Decodo conditionally scrapes target contact page...                                       |
| If: Contact Email Exists?     | If                        | Checks if valid contact email found          | Code: Finalize Contact Data      | Airtable: Update Lead Contact Email (true), Loop: Process Leads (false) | ## Deep Enrichment Pipeline Decodo conditionally scrapes target contact page...                                       |
| Airtable: Update Lead Contact Email | Airtable Update       | Updates Airtable record with enriched email | If: Contact Email Exists? (true) | Loop: Process Leads                           | ## Deep Enrichment Pipeline Decodo conditionally scrapes target contact page...                                       |
| Sticky Note                   | Sticky Note               | Documentation and overview                   | —                               | —                                            | # Digital Footprint Lead Generation: Decodo & Airtable ...                                                           |
| Sticky Note1                  | Sticky Note               | Configuration instructions                   | —                               | —                                            | ## Configuration: Set Search Parameters ...                                                                           |
| Sticky Note2                  | Sticky Note               | Deep enrichment explanation                   | —                               | —                                            | ## Deep Enrichment Pipeline ...                                                                                        |
| Sticky Note3                  | Sticky Note               | Data consolidation explanation                | —                               | —                                            | ## Data Consolidation & Continuation ...                                                                               |
| Sticky Note4                  | Sticky Note               | Database integrity loop explanation           | —                               | —                                            | ## Database Integrity Loop ...                                                                                         |
| Sticky Note5                  | Sticky Note               | Data sourcing and lead list generation        | —                               | —                                            | ## Data Sourcing & Lead List Generation ...                                                                            |
| Sticky Note8                  | Sticky Note               | Promotional coupon and signup link            | —                               | —                                            | ## 🎁 Exclusive 80% Discount! Get 80% OFF... [Click here to Sign Up & Claim](https://visit.decodo.com/c/6679292/3071239/17480) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create Set Node "Config: Set Search Params":**  
   - Define variables:  
     - `tech_footprint` = "We use Klaviyo" (string)  
     - `target_industry` = "" (string, customizable)  
     - `target_geo` = "United States" (string)  
     - `target_locale` = "en-US" (string)  
   - Connect output of Manual Trigger to this node.

3. **Create Decodo Google Search Node:**  
   - Type: Decodo search node  
   - Operation: `google_search`  
   - Query: Combine `tech_footprint` and `target_industry` from Config node. Use expression: `={{ $('Config: Set Search Params').item.json.tech_footprint + " " + $('Config: Set Search Params').item.json.target_industry }}`  
   - Geo and Locale: Use expressions from Config node (`target_geo`, `target_locale`).  
   - Results limit: 50  
   - Headless: false (JS rendering disabled)  
   - Credentials: Set Decodo API credentials.  
   - Connect Config output to this node.

4. **Create Code Node "Code: Initial Domain Filter":**  
   - JavaScript code to parse Decodo results:  
     - Extract domains from URLs using regex.  
     - Remove duplicates.  
     - Tag leads as "Paid Ad Lead" or "Organic Lead" based on result type.  
   - Input: Decodo Google Search output.  
   - Output: Array of unique leads with domain, source_url, source_title, lead_type.  
   - Connect Decodo Google Search output to this node.

5. **Create SplitInBatches Node "Loop: Process Leads":**  
   - Default config to process each lead individually.  
   - Connect Code node output to this node.

6. **Create Airtable Search Node "Airtable: Check Existence":**  
   - Base: Select your Airtable base containing Leads table.  
   - Table: Leads  
   - Operation: Search  
   - Filter formula: `={{ "({Domain} = '" + $json.domain + "')" }}` (checks if domain exists)  
   - Credentials: Airtable Personal Access Token.  
   - Connect Loop output to this node.

7. **Create If Node "If: Lead Exists?":**  
   - Condition: Check if Airtable search returned empty results (lead does not exist).  
   - True branch: lead exists  
   - False branch: new lead  
   - Connect Airtable Check output to this node.

8. **Create Airtable Create Node "Airtable: Create Lead":**  
   - Base/Table same as above.  
   - Map fields:  
     - Domain: from lead domain  
     - Status: "New Lead"  
     - Lead Type: from lead data  
     - Date Added/Updated: current time (`$now`)  
     - Source URL: from lead data  
   - Credentials: Airtable Token.  
   - Connect False branch (lead does not exist) of If node here.

9. **Create Airtable Update Node "Airtable: Update Lead":**  
   - Update record by ID (from Airtable search result).  
   - Update fields:  
     - Status: "Updated"  
     - Lead Type, Source URL, Date Updated  
   - Connect True branch (lead exists) of If node here.

10. **Create Set Node "Data Merger: ID Finalizer":**  
    - Extract Airtable record ID and domain from previous node output.  
    - Connect outputs of both Airtable Create and Update nodes to this node.

11. **Create If Node "If: Enrichment Needed?":**  
    - Condition: Check if `Primary Email` field is empty or missing in Airtable record.  
    - True branch: email missing → continue enrichment  
    - False branch: email exists → loop back to Loop node for next lead  
    - Connect Airtable Update Lead output to this node.

12. **Connect True branch of "If: Enrichment Needed?" back to "Data Merger: ID Finalizer"** (to continue enrichment pipeline).

13. **Create Decodo Google Search Node "Decodo: Email Search":**  
    - Operation: google_search  
    - Query: Use expression to search within domain for contact pages:  
      `={{ 'site:' + $json.domain + ' (inurl:contact OR inurl:about OR intitle:"Contact Us" OR intitle:"About Us")' }}`  
    - Results limit: 20  
    - Geo and Locale from Config node.  
    - Credentials: Decodo API.  
    - Connect "Data Merger: ID Finalizer" output here.

14. **Create Code Node "Code: Initial Enrichment":**  
    - Extract emails and contact page URLs from Decodo search snippets using regex.  
    - Store first found email and first matching contact page URL.  
    - Connect Decodo Email Search output to this node.

15. **Create If Node "If: Contact Page Exists?":**  
    - Condition: Check if `contact_page_url` is non-empty.  
    - True branch: proceed to scrape contact page  
    - False branch: loop back to "Loop: Process Leads" (skip enrichment)  
    - Connect Code Initial Enrichment output here.

16. **Create Decodo Scrape Node "Decodo: Scrape Contact Page":**  
    - Scrapes content from `contact_page_url`.  
    - Credentials: Decodo API.  
    - Connect True branch of contact page check here.

17. **Create Code Node "Code: Finalize Contact Data":**  
    - Extracts all emails from scraped content.  
    - Prioritizes emails based on keywords (sales, founder, contact).  
    - Returns `final_contact_email`.  
    - Connect Decodo Scrape Contact Page output here.

18. **Create If Node "If: Contact Email Exists?":**  
    - Checks if `final_contact_email` is non-empty.  
    - True branch: update Airtable with new email  
    - False branch: loop back to "Loop: Process Leads"  
    - Connect Code Finalize Contact Data output here.

19. **Create Airtable Update Node "Airtable: Update Lead Contact Email":**  
    - Update record by Airtable ID.  
    - Fields updated:  
      - `Primary Email`: `final_contact_email`  
      - `Contact Page URL`  
      - Status: "Enrichment Complete"  
      - Date Updated  
    - Credentials: Airtable Token.  
    - Connect True branch of Contact Email Exists? node here.

20. **Connect Airtable Update Lead Contact Email output back to "Loop: Process Leads"** to continue processing remaining leads.

21. **Configure Airtable Base Schema:**  
    - Table named "Leads" with fields:  
      - Domain (Single Line Text, primary field)  
      - Primary Email (Email)  
      - Contact Page URL (URL)  
      - Source URL (URL)  
      - Lead Type (Single Select: "Paid Ad Lead", "Organic Lead")  
      - Status (Single Select: "New Lead", "Updated", "Enrichment Complete")  
      - Date Added, Date Updated (DateTime fields)

22. **Set up credentials:**  
    - Decodo API key credential with API access.  
    - Airtable Personal Access Token with read/write access to the Leads base.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| The workflow performs a **Find/Create/Update** sequence for every lead, ensuring database integrity and efficient use of API credits.           | Sticky Note on main workflow overview.                                                                           |
| Airtable base must have specific fields as outlined, especially the `Lead Type` and `Status` single select fields matching workflow logic.      | Sticky Note on Airtable setup instructions.                                                                      |
| Decodo API usage is optimized by disabling JS rendering (headless=false) for initial searches to reduce costs.                                  | Code comments and Decodo node configurations.                                                                     |
| Email extraction prioritizes high-value contacts (sales, founder, CEO) before falling back to generic addresses (info, contact, support).       | Code: Finalize Contact Data node explanation.                                                                     |
| Use the coupon code `ATTAN8N` for 80% discount on Decodo’s Advanced Scraping API plan.                                                           | Promotional Sticky Note with signup link: https://visit.decodo.com/c/6679292/3071239/17480                         |
| The workflow handles error cases gracefully by checking for missing data containers and empty search results to prevent crashes.                 | Code nodes include robust checks.                                                                                  |
| Customize `target_industry` in the Config node to focus lead search on specific verticals or company types.                                      | Config: Set Search Params node instructions.                                                                       |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, respecting all relevant content policies and legality. All data processed is public and lawful.