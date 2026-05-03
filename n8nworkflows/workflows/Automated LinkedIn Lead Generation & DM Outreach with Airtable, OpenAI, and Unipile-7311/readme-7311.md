Automated LinkedIn Lead Generation & DM Outreach with Airtable, OpenAI, and Unipile

https://n8nworkflows.xyz/workflows/automated-linkedin-lead-generation---dm-outreach-with-airtable--openai--and-unipile-7311


# Automated LinkedIn Lead Generation & DM Outreach with Airtable, OpenAI, and Unipile

---

## 1. Workflow Overview

This workflow automates LinkedIn lead generation and direct message (DM) outreach by integrating Airtable for data management, OpenAI for AI-powered content generation, and Unipile for messaging and connection management. Its primary use case is to identify relevant leads interacting with LinkedIn posts, qualify them based on custom criteria, enrich their data, and conduct personalized DM outreach campaigns.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Routing**: Receives webhook-triggered input and routes the flow based on requested actions.
- **1.2 Agency Data Scraping and Business Summary Extraction**: Scrapes agency websites for business data and generates structured summaries using OpenAI.
- **1.3 LinkedIn Post Scraping and Lead Extraction**: Scrapes LinkedIn posts and comments to identify potential leads.
- **1.4 Lead Data Enrichment**: Scrapes LinkedIn profiles of leads for detailed information enrichment.
- **1.5 Lead Qualification**: Uses OpenAI to qualify leads against custom criteria extracted from Airtable.
- **1.6 Connection Status Checking and Management**: Checks connection status of leads via Unipile API and updates Airtable.
- **1.7 Connection Request Sending**: Sends LinkedIn connection requests to appropriate leads and updates status.
- **1.8 Personalized DM Message Generation and Sending**: Generates personalized LinkedIn DM messages using OpenAI and sends them via Unipile.
- **1.9 Campaign and Timer Management**: Manages multiple concurrent campaigns, time slot validation, and execution limits to throttle workflow actions.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception and Routing
**Overview:**  
This block handles incoming webhook requests and routes workflow execution according to action specified in the query. It ensures workflow branches are executed only when relevant.

**Nodes Involved:**  
- Webhook  
- Switch

**Node Details:**

- **Webhook**  
  - *Type:* Webhook trigger  
  - *Configuration:* Listens on path `ai-powered-linkedin-dm-engine-v-0-1` (disabled by default)  
  - *Input/Output:* Receives HTTP requests with query parameters, outputs JSON data  
  - *Edge Cases:* Disabled by default; must be enabled and webhook URL shared properly  
  - *Notes:* Entry point for external triggers

- **Switch**  
  - *Type:* Routing switch node  
  - *Configuration:* Routes based on `$json.query.action` equality to `searchAgency`, `launchApifyScraper`, or `runQualification`  
  - *Input:* From Webhook  
  - *Output:* To respective branches for agency scraping, LinkedIn scraping, or qualification  
  - *Edge Cases:* Missing or unexpected action value leads to no output

---

### 1.2 Agency Data Scraping and Business Summary Extraction  
**Overview:**  
Scrapes the agency website using Jina AI API and generates a structured business summary, ideal client profile, and business offer through OpenAI.

**Nodes Involved:**  
- Get Agency Record Data (Airtable)  
- Scrape my Website (HTTP Request)  
- Business Summary Writer (OpenAI)  
- Agency Business Data (Set)  
- Update Agency Business Record (Airtable)  

**Node Details:**

- **Get Agency Record Data**  
  - Retrieves agency setup record from Airtable based on incoming record ID  
  - Requires Airtable API credentials  
  - Edge cases: Invalid record ID or API errors

- **Scrape my Website**  
  - Calls Jina AI scraping API with agency website URL and API key  
  - Sends authorization headers and expects text response  
  - Retries on failure enabled  
  - Edge cases: API key invalid, website unavailable, network errors

- **Business Summary Writer**  
  - Uses OpenAI `o4-mini` model to generate JSON with agency business summary, ideal client profile, and offer based on scraped text  
  - System prompt instructs AI to output strict JSON format without extra text  
  - Edge cases: API rate limits, invalid response format

- **Agency Business Data**  
  - Extracts JSON fields from AI response and sets them as separate workflow variables

- **Update Agency Business Record**  
  - Updates the agency record in Airtable with new business summary data and marks status to "Search Complete"  
  - Edge cases: Airtable API errors, record concurrency conflicts

---

### 1.3 LinkedIn Post Scraping and Lead Extraction  
**Overview:**  
Fetches LinkedIn post details and scrapes comments to identify leads who engaged with the post.

**Nodes Involved:**  
- Get Lead List Record - Apify Scraper (Airtable)  
- Get Post URL Data (Set)  
- Scrape LinkedIn Post (HTTP Request)  
- Get Post Information (Set)  
- Loop for More Comments (Code)  
- Page Limit for Demo (Limit)  
- Loop Over Comments Page (SplitInBatches)  
- Scrape LinkedIn Comments (HTTP Request)  
- Filter Author Comments Out (Filter)  
- Get Lead LinkedIn Profile URL (Set)  

**Node Details:**

- **Get Lead List Record - Apify Scraper**  
  - Retrieves lead list record from Airtable containing LinkedIn post URL and API keys  
  - Edge cases: Invalid record ID, missing API keys

- **Get Post URL Data**  
  - Extracts post URL, lead limit, lead list record ID, and LinkedIn post setup into workflow variables for downstream use

- **Scrape LinkedIn Post**  
  - Calls Apify LinkedIn post detail scraper API with post URL  
  - Requires Apify API key from record  
  - Retries enabled  
  - Edge cases: API errors, rate limits

- **Get Post Information**  
  - Sets key post data (ID, URL, text, author name/headline) and paging parameters extracted from lead list record

- **Loop for More Comments**  
  - Generates an array of page numbers from 1 to number of pages for iterative scraping

- **Page Limit for Demo**  
  - Limits number of pages to scrape based on lead list setup (demo constraint)

- **Loop Over Comments Page**  
  - Splits comment pages into batches for API scraping

- **Scrape LinkedIn Comments**  
  - Calls Apify post comments scraper API per page with limits and post ID  
  - Retries enabled

- **Filter Author Comments Out**  
  - Filters out comments made by the original post author to focus on leads

- **Get Lead LinkedIn Profile URL**  
  - Extracts lead LinkedIn profile URL, comment text, and comment ID for each lead comment

---

### 1.4 Lead Data Enrichment  
**Overview:**  
Scrapes detailed LinkedIn profile data for each lead and creates records in Airtable.

**Nodes Involved:**  
- Limit Profile Scraping (Limit)  
- Loop Over LinkedIn Profile (SplitInBatches)  
- Scraping LinkedIn Profile (HTTP Request)  
- Wait 1s (Wait)  
- Get Lead Scraped Data (Set)  
- Create Lead Record - Scraper (Airtable)  

**Node Details:**

- **Limit Profile Scraping**  
  - Limits the number of profiles scraped to lead limit from post setup

- **Loop Over LinkedIn Profile**  
  - Batches LinkedIn profile scraping calls

- **Scraping LinkedIn Profile**  
  - Calls Apify LinkedIn profile batch scraper API with profile URLs  
  - Retries enabled, max 2 tries

- **Wait 1s**  
  - Waits 1 second between profile scrapes to avoid rate limiting

- **Get Lead Scraped Data**  
  - Maps LinkedIn profile JSON fields into structured lead data fields (name, headline, company, location, public identifier, etc.)

- **Create Lead Record - Scraper**  
  - Creates new lead records in Airtable with enriched data, linked to lead list and post setup  
  - Retries enabled  
  - Edge cases: Airtable API errors, data mapping issues

---

### 1.5 Lead Qualification  
**Overview:**  
Qualifies leads by comparing their data against criteria using OpenAI and updates Airtable with qualification results.

**Nodes Involved:**  
- Get Lead List Record Data - Qualification (Airtable)  
- Get Leads Data - Qualification (Set)  
- Split Out Leads for Qualification Looping (SplitOut)  
- Get Lead Record Data - Qualification (Airtable)  
- Lead Data - Qualification (Set)  
- Lead Qualifier (OpenAI)  
- Qualification Result (Set)  
- If Qualified? (If)  
- Update Lead Record - Qualification Result - True (Airtable)  
- Update Lead Record - Qualification Result - False (Airtable)  
- Merge  

**Node Details:**

- **Get Lead List Record Data - Qualification**  
  - Retrieves lead list record containing qualification criteria

- **Get Leads Data - Qualification**  
  - Extracts leads array from record data

- **Split Out Leads for Qualification Looping**  
  - Splits leads into individual items for processing

- **Get Lead Record Data - Qualification**  
  - Retrieves detailed lead data from Airtable for each lead

- **Lead Data - Qualification**  
  - Prepares lead info and qualification criteria for AI input

- **Lead Qualifier**  
  - Uses OpenAI `o4-mini` model to determine if lead meets criteria, returning JSON with `qualified` boolean and reason  
  - Retries enabled

- **Qualification Result**  
  - Sets lead ID, qualification status, and reason from AI output

- **If Qualified?**  
  - Branches workflow based on qualification result

- **Update Lead Record - Qualification Result - True/False**  
  - Updates Airtable lead record with qualification status and reason accordingly

- **Merge**  
  - Merges both update outputs back to main flow

---

### 1.6 Connection Status Checking and Management  
**Overview:**  
Checks the LinkedIn connection status of leads using Unipile API and updates Airtable accordingly.

**Nodes Involved:**  
- Get Lead Data - Active Campaign (Set)  
- Split Out Lead - Active Campaign (SplitOut)  
- Get Lead Record - Campaign Operation (Airtable)  
- Allow Only Not Connected Status (Filter)  
- Limit Lead Flow - Profile Retrieve (Limit)  
- Loop Over Not Checked Lead (SplitInBatches)  
- Check Connection Status (HTTP Request)  
- Edit Fields (Set)  
- If Not Connected (If)  
- Update Lead Record - Connected (Airtable)  
- Update Lead Record - Not Connected (Airtable)  
- Update Status About Connection Status Check (NoOp)  
- Merge - But Only One can Be true (Merge)  

**Node Details:**

- **Get Lead Data - Active Campaign**  
  - Extracts leads, Unipile API keys, account IDs, limits, and DSN data for active campaign

- **Split Out Lead - Active Campaign**  
  - Splits leads array for processing

- **Get Lead Record - Campaign Operation**  
  - Retrieves lead record data from Airtable for each lead

- **Allow Only Not Connected Status**  
  - Filters leads whose connection status is "Not Connected"

- **Limit Lead Flow - Profile Retrieve**  
  - Limits number of processed leads based on daily retrieve limit and active campaigns count

- **Loop Over Not Checked Lead**  
  - Processes leads in batches

- **Check Connection Status**  
  - Calls Unipile API to check if lead is connected or not  
  - Retries enabled

- **Edit Fields**  
  - Extracts `provider_id` and connection boolean `is_relationship` from API response

- **If Not Connected**  
  - Branches by connection boolean: true (connected), false (not connected)

- **Update Lead Record - Connected**  
  - Updates Airtable with connection status "Connected" and DM status "Awaiting"

- **Update Lead Record - Not Connected**  
  - Updates Airtable with connection status "Not Connected"

- **Update Status About Connection Status Check**  
  - No operation node used for flow control

- **Merge - But Only One can Be true**  
  - Merges outputs for next processing steps

---

### 1.7 Connection Request Sending  
**Overview:**  
Sends LinkedIn connection requests via Unipile API to leads with "Not Connected" status and updates Airtable accordingly.

**Nodes Involved:**  
- Allow Only Not Connected Status (Filter)  
- Limit Lead Flow - Connection Request Sent (Limit)  
- Loop Over Lead - Send Connection Request (SplitInBatches)  
- Send Connection Request (HTTP Request)  
- If Invitation Sent (If)  
- Update Lead Record - Connection Pending (Airtable)  
- Connection Request Failed (NoOp)  
- Update Status Connection Batch Sent (NoOp)  

**Node Details:**

- **Allow Only Not Connected Status**  
  - Filters leads for connection requests

- **Limit Lead Flow - Connection Request Sent**  
  - Limits number of connection requests sent per day based on configured limits and active campaigns

- **Loop Over Lead - Send Connection Request**  
  - Processes leads in batches for sending requests

- **Send Connection Request**  
  - Calls Unipile API to send connection invitation using provider ID and account ID  
  - Continues on error to avoid workflow break

- **If Invitation Sent**  
  - Checks API response for success indicator "UserInvitationSent"

- **Update Lead Record - Connection Pending**  
  - Updates Airtable with connection status "Pending"

- **Connection Request Failed**  
  - No operation node for handling failed sends

- **Update Status Connection Batch Sent**  
  - No operation node for flow control

---

### 1.8 Personalized DM Message Generation and Sending  
**Overview:**  
Crafts personalized LinkedIn DM messages for connected leads using OpenAI and sends the messages via Unipile API, updating Airtable status accordingly.

**Nodes Involved:**  
- Allow Only Connected Lead (Filter)  
- Limit DM To Send (Limit)  
- Loop Over Lead To DM (SplitInBatches)  
- Lead Data - DM Personalization (Set)  
- Personalized DM Message Writer (OpenAI)  
- Send DM Message (HTTP Request)  
- If ChatStarted (If)  
- Update Lead Record - DM Sent (Airtable)  
- Send Message Failed (NoOp)  
- Update Status of DM Message Sent (NoOp)  

**Node Details:**

- **Allow Only Connected Lead**  
  - Filters leads with connection status "Connected"

- **Limit DM To Send**  
  - Limits number of DMs to send per run (default 3)

- **Loop Over Lead To DM**  
  - Processes leads in batches for DM sending

- **Lead Data - DM Personalization**  
  - Collects lead and campaign data for input to DM message generation

- **Personalized DM Message Writer**  
  - Uses OpenAI `o4-mini` to create personalized DM messages based on lead info, post context, campaign offer, CTA, and DM template  
  - Retries enabled

- **Send DM Message**  
  - Sends DM via Unipile API chat creation endpoint using provider ID and account ID  
  - Uses multipart form data  
  - Retries enabled

- **If ChatStarted**  
  - Checks if chat was successfully started

- **Update Lead Record - DM Sent**  
  - Updates Airtable lead record with chat ID, message ID, DM status "DM Sent", and the personalized DM message

- **Send Message Failed**  
  - No operation node for failure handling

- **Update Status of DM Message Sent**  
  - No operation node for flow control

---

### 1.9 Campaign and Timer Management  
**Overview:**  
Manages multiple active LinkedIn DM campaigns, checks current time against configured timer slots, throttles operations, and updates campaign execution status.

**Nodes Involved:**  
- Schedule Trigger  
- Search Active Campaign Status (Airtable)  
- Reset Number Active Campaign (Airtable)  
- Get Active Campaign Record ID (Set)  
- Loop Over Active Campaigns - Counting (SplitInBatches)  
- Search LinkedIn Campaign Count Record (Airtable)  
- Increment Count Active Campaign (Set)  
- Create or update Active Campaign Count record (Airtable)  
- Aggregate All Active Record Into One (Aggregate)  
- Create or update LinkedIn Campaign Record ID (Airtable)  
- Split Out Active Campaign (SplitOut)  
- Campaign Operation Data (Set)  
- Time Slot Matching Code (Code)  
- If Matching is True (If)  
- If Current Timer Slot Match Corresponding Timer Slot? (If)  
- Loop Over Active Campaigns1 (SplitInBatches)  
- Get Active Campaign Data (Set)  
- Limit one Item to Update Current Timer Slot (Limit)  
- If Corresponding Timer Slot = Timer 1/2/3 Slot? (If)  
- Change Next Current Timer to Timer 1/2/3 Slot (Airtable)  
- Merge - But Only One can Be true (Merge)  

**Node Details:**

- **Schedule Trigger**  
  - Runs workflow periodically every 25 to 34 minutes randomized interval

- **Search Active Campaign Status**  
  - Queries Airtable for campaigns with status "Active"

- **Reset Number Active Campaign**  
  - Resets daily active campaign counter in Airtable at beginning of run

- **Get Active Campaign Record ID**  
  - Extracts active campaign ID from search result

- **Loop Over Active Campaigns - Counting**  
  - Loops over active campaigns to count them

- **Search LinkedIn Campaign Count Record**  
  - Retrieves current campaign count record from Airtable

- **Increment Count Active Campaign**  
  - Increments active campaign count by 1

- **Create or update Active Campaign Count record**  
  - Updates campaign count record in Airtable

- **Aggregate All Active Record Into One**  
  - Aggregates active campaign IDs into single array

- **Create or update LinkedIn Campaign Record ID**  
  - Updates Airtable with aggregated active campaigns

- **Split Out Active Campaign**  
  - Splits campaigns into individual items for processing

- **Campaign Operation Data**  
  - Extracts timer slots, timezone, current timer, campaign ID, and execution counts for current campaign

- **Time Slot Matching Code**  
  - Custom JavaScript node that parses timer slot strings and compares current time to these slots  
  - Determines if current time matches any configured timer slot and tags accordingly

- **If Matching is True**  
  - Branches only if current time matches a timer slot

- **If Current Timer Slot Match Corresponding Timer Slot?**  
  - Checks if campaign current timer matches corresponding timer for execution

- **Loop Over Active Campaigns1**  
  - Loops over active campaigns for further processing

- **Get Active Campaign Data**  
  - Sets more detailed active campaign data for use downstream

- **Limit one Item to Update Current Timer Slot**  
  - Limits to one active campaign for timer update

- **If Corresponding Timer Slot = Timer 1/2/3 Slot?**  
  - Checks which timer slot matched and triggers update accordingly

- **Change Next Current Timer to Timer 1/2/3 Slot**  
  - Updates Airtable campaign record to set next timer slot to process

- **Merge - But Only One can Be true**  
  - Merges multiple timer slot update branches

- **Edge Cases:**  
  - Incorrect time slot format could cause mismatches  
  - Campaign status changes mid-execution may disrupt counting  
  - Airtable API rate limits or update conflicts

---

## 3. Summary Table

| Node Name                             | Node Type                     | Functional Role                                   | Input Node(s)                                  | Output Node(s)                                          | Sticky Note                                                                                              |
|-------------------------------------|-------------------------------|-------------------------------------------------|------------------------------------------------|--------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook                             | Webhook                      | Entry point, receives external trigger          |                                                | Switch                                                 |                                                                                                        |
| Switch                             | Switch                       | Routes workflow based on action query           | Webhook                                         | Get Agency Record Data, Get Lead List Record - Apify Scraper, Get Lead List Record Data - Qualification |                                                                                                        |
| Get Agency Record Data              | Airtable                     | Retrieves agency setup data                       | Switch (searchAgency)                           | Scrape my Website                                      | Searching About Agency                                                                                   |
| Scrape my Website                  | HTTP Request                 | Scrapes agency website via Jina AI API           | Get Agency Record Data                          | Business Summary Writer                                | Searching About Agency                                                                                   |
| Business Summary Writer            | OpenAI                      | Generates agency business summary JSON           | Scrape my Website                              | Agency Business Data                                   | Searching About Agency                                                                                   |
| Agency Business Data               | Set                         | Extracts AI output fields into variables         | Business Summary Writer                        | Update Agency Business Record                          | Searching About Agency                                                                                   |
| Update Agency Business Record      | Airtable                    | Updates agency record with summary data          | Agency Business Data                           |                                                        | Searching About Agency                                                                                   |
| Get Lead List Record - Apify Scraper | Airtable                    | Retrieves lead list record for LinkedIn post     | Switch (launchApifyScraper)                     | Get Post URL Data                                     | Scrape LinkedIn URL Post Branch                                                                         |
| Get Post URL Data                  | Set                         | Extracts post URL and lead limits                 | Get Lead List Record - Apify Scraper           | Scrape LinkedIn Post                                  | Scrape LinkedIn URL Post Branch                                                                         |
| Scrape LinkedIn Post              | HTTP Request                | Scrapes LinkedIn post details via Apify           | Get Post URL Data                              | Get Post Information                                  | Scrape LinkedIn URL Post Branch                                                                         |
| Get Post Information              | Set                         | Extracts post metadata and paging parameters      | Scrape LinkedIn Post                           | Loop for More Comments                                | Scrape LinkedIn URL Post Branch                                                                         |
| Loop for More Comments            | Code                        | Generates page numbers for comment scraping       | Get Post Information                           | Page Limit for Demo                                   | Scrape LinkedIn URL Post Branch                                                                         |
| Page Limit for Demo               | Limit                       | Limits comment pages scraped                       | Loop for More Comments                         | Loop Over Comments Page                               | Scrape LinkedIn URL Post Branch                                                                         |
| Loop Over Comments Page           | SplitInBatches              | Batches comment pages for scraping                 | Page Limit for Demo                            | Scrape LinkedIn Comments                              | Scrape LinkedIn URL Post Branch                                                                         |
| Scrape LinkedIn Comments          | HTTP Request                | Scrapes LinkedIn post comments via Apify          | Loop Over Comments Page                        | Filter Author Comments Out                            | Scrape LinkedIn URL Post Branch                                                                         |
| Filter Author Comments Out        | Filter                      | Filters out post author's comments                 | Scrape LinkedIn Comments                       | Get Lead LinkedIn Profile URL                         | Scrape LinkedIn URL Post Branch                                                                         |
| Get Lead LinkedIn Profile URL     | Set                         | Extracts lead profile URL and comment details      | Filter Author Comments Out                     | Limit Profile Scraping                                | Scrape LinkedIn URL Post Branch                                                                         |
| Limit Profile Scraping            | Limit                       | Limits number of LinkedIn profile scrapes          | Get Lead LinkedIn Profile URL                  | Loop Over LinkedIn Profile                            | Scrape LinkedIn URL Post Branch                                                                         |
| Loop Over LinkedIn Profile        | SplitInBatches              | Batches profile scraping calls                      | Limit Profile Scraping                         | Scraping LinkedIn Profile                             | Scrape LinkedIn URL Post Branch                                                                         |
| Scraping LinkedIn Profile         | HTTP Request                | Scrapes detailed LinkedIn profile data             | Loop Over LinkedIn Profile                     | Wait 1s                                              | Scrape LinkedIn URL Post Branch                                                                         |
| Wait 1s                          | Wait                        | Pauses 1 second between profile scrapes            | Scraping LinkedIn Profile                      | Get Lead Scraped Data                                | Scrape LinkedIn URL Post Branch                                                                         |
| Get Lead Scraped Data             | Set                         | Maps LinkedIn profile data to lead fields          | Wait 1s                                        | Create Lead Record - Scraper                         | Scrape LinkedIn URL Post Branch                                                                         |
| Create Lead Record - Scraper      | Airtable                    | Creates lead record with enriched data             | Get Lead Scraped Data                          | Loop Over LinkedIn Profile                           | Scrape LinkedIn URL Post Branch                                                                         |
| Get Lead List Record Data - Qualification | Airtable                    | Retrieves lead list record for qualification       | Switch (runQualification)                      | Get Leads Data - Qualification                        | Lead Qualification Branch                                                                               |
| Get Leads Data - Qualification   | Set                         | Extracts leads array                                | Get Lead List Record Data - Qualification      | Split Out Leads for Qualification Looping             | Lead Qualification Branch                                                                               |
| Split Out Leads for Qualification Looping | SplitOut                    | Splits leads for individual qualification          | Get Leads Data - Qualification                  | Get Lead Record Data - Qualification                   | Lead Qualification Branch                                                                               |
| Get Lead Record Data - Qualification | Airtable                    | Retrieves individual lead record for qualification | Split Out Leads for Qualification Looping      | Lead Data - Qualification                             | Lead Qualification Branch                                                                               |
| Lead Data - Qualification        | Set                         | Prepares lead data and qualification criteria      | Get Lead Record Data - Qualification            | Lead Qualifier                                       | Lead Qualification Branch                                                                               |
| Lead Qualifier                   | OpenAI                      | AI lead qualification decision                      | Lead Data - Qualification                      | Qualification Result                                 | Lead Qualification Branch                                                                               |
| Qualification Result             | Set                         | Extracts qualification status and reason           | Lead Qualifier                                 | If Qualified?                                        | Lead Qualification Branch                                                                               |
| If Qualified?                   | If                          | Branches on qualification result                    | Qualification Result                           | Update Lead Record - Qualification Result - True, Update Lead Record - Qualification Result - False | Lead Qualification Branch                                                                               |
| Update Lead Record - Qualification Result - True | Airtable                    | Updates lead record as qualified                     | If Qualified? (true)                           | Merge                                                | Lead Qualification Branch                                                                               |
| Update Lead Record - Qualification Result - False | Airtable                    | Updates lead record as not qualified                 | If Qualified? (false)                          | Merge                                                | Lead Qualification Branch                                                                               |
| Merge                           | Merge                       | Merges qualification update branches                | Update Lead Record - Qualification Result - True, Update Lead Record - Qualification Result - False |                                                        | Lead Qualification Branch                                                                               |
| Get Lead Data - Active Campaign  | Set                         | Extracts lead and API credentials for active campaign | Get Lead Record - LinkedIn DM Campaign         | Split Out Lead - Active Campaign                      | LinkedIn DM Campaign Workflow                                                                           |
| Split Out Lead - Active Campaign | SplitOut                    | Splits leads for connection status checking         | Get Lead Data - Active Campaign                 | Get Lead Record - Campaign Operation                  | LinkedIn DM Campaign Workflow                                                                           |
| Get Lead Record - Campaign Operation | Airtable                    | Retrieves lead record for campaign operation          | Split Out Lead - Active Campaign                | Allow Only Not Connected Status                       | LinkedIn DM Campaign Workflow                                                                           |
| Allow Only Not Connected Status | Filter                      | Filters leads with connection status "Not Connected" | Get Lead Record - Campaign Operation            | Limit Lead Flow - Profile Retrieve                    | LinkedIn DM Campaign Workflow                                                                           |
| Limit Lead Flow - Profile Retrieve | Limit                       | Limits leads processed based on daily limits         | Allow Only Not Connected Status                 | Loop Over Not Checked Lead                            | LinkedIn DM Campaign Workflow                                                                           |
| Loop Over Not Checked Lead       | SplitInBatches              | Batches leads for connection status checking         | Limit Lead Flow - Profile Retrieve              | Check Connection Status                               | LinkedIn DM Campaign Workflow                                                                           |
| Check Connection Status          | HTTP Request                | Calls Unipile API to check connection status          | Loop Over Not Checked Lead                      | Edit Fields                                          | LinkedIn DM Campaign Workflow                                                                           |
| Edit Fields                     | Set                         | Extracts provider ID and relationship boolean         | Check Connection Status                         | If Not Connected                                     | LinkedIn DM Campaign Workflow                                                                           |
| If Not Connected                | If                          | Branches on connection status                          | Edit Fields                                     | Update Lead Record - Connected, Update Lead Record - Not Connected | LinkedIn DM Campaign Workflow                                                                           |
| Update Lead Record - Connected   | Airtable                    | Updates lead as connected and DM awaiting             | If Not Connected (true)                         | Loop Over Not Checked Lead                            | LinkedIn DM Campaign Workflow                                                                           |
| Update Lead Record - Not Connected | Airtable                    | Updates lead as not connected                           | If Not Connected (false)                        | Loop Over Not Checked Lead                            | LinkedIn DM Campaign Workflow                                                                           |
| Update Status About Connection Status Check | NoOp                        | Flow control node after connection check              | Loop Over Not Checked Lead                      | Merge                                                | LinkedIn DM Campaign Workflow                                                                           |
| Merge - But Only One can Be true | Merge                       | Merges connection status update branches               | Update Lead Record - Connected, Update Lead Record - Not Connected | Loop Over One Active Campaign                        | LinkedIn DM Campaign Workflow                                                                           |
| Allow Only Not Connected Status | Filter                      | Filters leads for connection request sending           | Get Lead Record - Campaign Operation            | Limit Lead Flow - Connection Request Sent             | LinkedIn DM Campaign Workflow                                                                           |
| Limit Lead Flow - Connection Request Sent | Limit                       | Limits connection requests sent per day                 | Allow Only Not Connected Status                 | Loop Over Lead - Send Connection Request              | LinkedIn DM Campaign Workflow                                                                           |
| Loop Over Lead - Send Connection Request | SplitInBatches              | Batches leads for sending connection requests          | Limit Lead Flow - Connection Request Sent       | Send Connection Request                              | LinkedIn DM Campaign Workflow                                                                           |
| Send Connection Request          | HTTP Request                | Sends connection request via Unipile API               | Loop Over Lead - Send Connection Request        | If Invitation Sent                                   | LinkedIn DM Campaign Workflow                                                                           |
| If Invitation Sent              | If                          | Checks if invitation was sent successfully              | Send Connection Request                         | Update Lead Record - Connection Pending, Connection Request Failed | LinkedIn DM Campaign Workflow                                                                           |
| Update Lead Record - Connection Pending | Airtable                    | Updates lead status to pending connection               | If Invitation Sent (true)                       | Loop Over Lead - Send Connection Request              | LinkedIn DM Campaign Workflow                                                                           |
| Connection Request Failed        | NoOp                        | Flow control for failed connection requests             | If Invitation Sent (false)                      | Loop Over Lead - Send Connection Request              | LinkedIn DM Campaign Workflow                                                                           |
| Update Status Connection Batch Sent | NoOp                        | Flow control after sending connection requests          | Loop Over Lead - Send Connection Request        | Merge                                                | LinkedIn DM Campaign Workflow                                                                           |
| Allow Only Connected Lead       | Filter                      | Filters leads with connection status "Connected"        | Get Lead Record - Campaign Operation1           | Limit DM To Send                                     | LinkedIn DM Campaign Workflow                                                                           |
| Limit DM To Send                | Limit                       | Limits number of DMs sent per run                         | Allow Only Connected Lead                        | Loop Over Lead To DM                                 | LinkedIn DM Campaign Workflow                                                                           |
| Loop Over Lead To DM            | SplitInBatches              | Batches leads for DM sending                              | Limit DM To Send                                | Update Status of DM Message Sent, Lead Data - DM Personalization | LinkedIn DM Campaign Workflow                                                                           |
| Lead Data - DM Personalization | Set                         | Prepares personalization data for DM message generation | Loop Over Lead To DM                            | Personalized DM Message Writer                       | LinkedIn DM Campaign Workflow                                                                           |
| Personalized DM Message Writer  | OpenAI                      | Generates personalized LinkedIn DM messages              | Lead Data - DM Personalization                   | Send DM Message                                     | LinkedIn DM Campaign Workflow                                                                           |
| Send DM Message                | HTTP Request                | Sends DM via Unipile API                                  | Personalized DM Message Writer                   | If ChatStarted                                      | LinkedIn DM Campaign Workflow                                                                           |
| If ChatStarted                | If                          | Checks if chat was started successfully                   | Send DM Message                                  | Update Lead Record - DM Sent, Send Message Failed    | LinkedIn DM Campaign Workflow                                                                           |
| Update Lead Record - DM Sent    | Airtable                    | Updates lead record with DM sent status and message       | If ChatStarted (true)                            | Loop Over Lead To DM                                | LinkedIn DM Campaign Workflow                                                                           |
| Send Message Failed            | NoOp                        | Flow control for failed DM sending                        | If ChatStarted (false)                           | Loop Over Lead To DM                                | LinkedIn DM Campaign Workflow                                                                           |
| Update Status of DM Message Sent | NoOp                        | Flow control after DM message sent                         | Loop Over Lead To DM                             | Merge                                                | LinkedIn DM Campaign Workflow                                                                           |
| Schedule Trigger               | Schedule Trigger            | Triggers workflow periodically                            |                                                | Search Active Campaign Status                        | LinkedIn DM Campaign Workflow                                                                           |
| Search Active Campaign Status  | Airtable                    | Retrieves active LinkedIn DM campaigns                    | Schedule Trigger                                | Reset Number Active Campaign                         | LinkedIn DM Campaign Workflow                                                                           |
| Reset Number Active Campaign   | Airtable                    | Resets daily active campaign count                        | Search Active Campaign Status                    | Get Active Campaign Record ID                        | LinkedIn DM Campaign Workflow                                                                           |
| Get Active Campaign Record ID  | Set                         | Extracts active campaign ID                               | Reset Number Active Campaign                     | Loop Over Active Campaigns - Counting                | LinkedIn DM Campaign Workflow                                                                           |
| Loop Over Active Campaigns - Counting | SplitInBatches              | Loops over active campaigns                                | Get Active Campaign Record ID                     | Search LinkedIn Campaign Count Record                | LinkedIn DM Campaign Workflow                                                                           |
| Search LinkedIn Campaign Count Record | Airtable                    | Retrieves campaign count record                            | Loop Over Active Campaigns - Counting             | Increment Count Active Campaign                      | LinkedIn DM Campaign Workflow                                                                           |
| Increment Count Active Campaign | Set                         | Increments active campaign count                          | Search LinkedIn Campaign Count Record             | Create or update Active Campaign Count record        | LinkedIn DM Campaign Workflow                                                                           |
| Create or update Active Campaign Count record | Airtable                    | Updates campaign count record                              | Increment Count Active Campaign                   |                                                        | LinkedIn DM Campaign Workflow                                                                           |
| Aggregate All Active Record Into One | Aggregate                   | Aggregates active campaign IDs                             | Get Active LinkedIn Campaign Record ID            | Create or update LinkedIn Campaign Record ID          | LinkedIn DM Campaign Workflow                                                                           |
| Create or update LinkedIn Campaign Record ID | Airtable                    | Updates LinkedIn campaign record with aggregated IDs      | Aggregate All Active Record Into One              | Split Out Active Campaign                            | LinkedIn DM Campaign Workflow                                                                           |
| Split Out Active Campaign      | SplitOut                    | Splits active campaigns                                    | Create or update LinkedIn Campaign Record ID      | Campaign Operation Data                             | LinkedIn DM Campaign Workflow                                                                           |
| Campaign Operation Data        | Set                         | Extracts timer slots, timezone, current timer info        | Split Out Active Campaign                         | Time Slot Matching Code                             | LinkedIn DM Campaign Workflow                                                                           |
| Time Slot Matching Code        | Code                        | Checks if current time matches any configured timer slot  | Campaign Operation Data                           | If Matching is True                                | LinkedIn DM Campaign Workflow                                                                           |
| If Matching is True            | If                          | Proceeds if current time matches a timer slot             | Time Slot Matching Code                           | If Current Timer Slot Match Corresponding Timer Slot?, Loop Over Active Campaigns1 | LinkedIn DM Campaign Workflow                                                                           |
| If Current Timer Slot Match Corresponding Timer Slot? | If                          | Checks if current timer matches corresponding timer slot   | If Matching is True                             | Get Active Campaign Data, Loop Over Active Campaigns1 | LinkedIn DM Campaign Workflow                                                                           |
| Loop Over Active Campaigns1    | SplitInBatches              | Loops over active campaigns                                | If Current Timer Slot Match Corresponding Timer Slot? | Limit one Item to Update Current Timer Slot, Get Active Campaign Record ID - Timer Slot Verification | LinkedIn DM Campaign Workflow                                                                           |
| Get Active Campaign Data       | Set                         | Sets detailed active campaign data                         | If Current Timer Slot Match Corresponding Timer Slot? | Loop Over Active Campaigns1                        | LinkedIn DM Campaign Workflow                                                                           |
| Limit one Item to Update Current Timer Slot | Limit                       | Limits to one campaign for timer slot update               | Loop Over Active Campaigns1                        | If Corresponding Timer Slot = Timer 1 Slot?          | LinkedIn DM Campaign Workflow                                                                           |
| If Corresponding Timer Slot = Timer 1 Slot? | If                          | Checks if timer 1 slot matched                              | Limit one Item to Update Current Timer Slot       | Change Next Current Timer to Timer 2 Slot, If Corresponding Timer Slot = Timer 2 Slot? | LinkedIn DM Campaign Workflow                                                                           |
| If Corresponding Timer Slot = Timer 2 Slot? | If                          | Checks if timer 2 slot matched                              | If Corresponding Timer Slot = Timer 1 Slot? (no) | Change Next Current Timer to Timer 3 Slot, If Corresponding Timer Slot = Timer 3 Slot? | LinkedIn DM Campaign Workflow                                                                           |
| If Corresponding Timer Slot = Timer 3 Slot? | If                          | Checks if timer 3 slot matched                              | If Corresponding Timer Slot = Timer 2 Slot? (no) | Change Next Current Timer to Timer 1 Slot, Loop Over Active Campaigns1 | LinkedIn DM Campaign Workflow                                                                           |
| Change Next Current Timer to Timer 1 Slot | Airtable                    | Updates campaign to set current timer to timer 1 slot      | If Corresponding Timer Slot = Timer 3 Slot?       | Loop Over Active Campaigns1                        | LinkedIn DM Campaign Workflow                                                                           |
| Change Next Current Timer to Timer 2 Slot | Airtable                    | Updates campaign to set current timer to timer 2 slot      | If Corresponding Timer Slot = Timer 1 Slot?       | Loop Over Active Campaigns1                        | LinkedIn DM Campaign Workflow                                                                           |
| Change Next Current Timer to Timer 3 Slot | Airtable                    | Updates campaign to set current timer to timer 3 slot      | If Corresponding Timer Slot = Timer 2 Slot?       | Loop Over Active Campaigns1                        | LinkedIn DM Campaign Workflow                                                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook
   - Path: `ai-powered-linkedin-dm-engine-v-0-1`
   - Enable and note the URL for external triggers

2. **Add Switch Node**
   - Connect Webhook output to Switch input
   - Add three rules based on `$json.query.action`:
     - Equals `searchAgency` → output: searchAgency
     - Equals `launchApifyScraper` → output: launchApifyScraper
     - Equals `runQualification` → output: runQualification

---

### Agency Data Scraping Branch (searchAgency)

3. **Airtable Node: Get Agency Record Data**
   - Operation: Get record by ID `$json.query.recordId`
   - Base: Your Airtable base with agency setup
   - Table: Agency Setup
   - Credentials: Airtable API token

4. **HTTP Request Node: Scrape my Website**
   - Method: GET
   - URL: `https://r.jina.ai/{{ $json['Agency Website'] }}`
   - Headers:
     - Authorization: `Bearer {{ $json['API Key (from Jina AI)'][0] }}`
     - X-Retain-Images: none
     - X-Return-Format: text
   - Enable retry on failure

5. **OpenAI Node: Business Summary Writer**
   - Model: `o4-mini`
   - System prompt: Specialized prompt for agency analysis (as in workflow)
   - Input: scraped text from previous node
   - Output: JSON with keys: Agency Business Summary, Agency Ideal Client Profile, Agency Business Offer
   - Credential: OpenAI API key

6. **Set Node: Agency Business Data**
   - Map AI JSON output fields to separate variables for update

7. **Airtable Node: Update Agency Business Record**
   - Operation: Update record by ID `$json.query.recordId`
   - Update fields: Agency Business Summary, Agency Ideal Client Profile, Agency Business Offer, Agency Status = "Search Complete"
   - Use correct base and table
   - Credentials: Airtable API token

---

### LinkedIn Post Scraping & Lead Extraction Branch (launchApifyScraper)

8. **Airtable Node: Get Lead List Record - Apify Scraper**
   - Operation: Get record by ID `$json.query.recordId`
   - Base and table configured for Lead Lists
   - Credentials: Airtable API token

9. **Set Node: Get Post URL Data**
   - Extract fields: LinkedIn Post URL, Lead Limit, Lead List record ID, LinkedIn Post Setup from previous node

10. **HTTP Request Node: Scrape LinkedIn Post**
    - POST JSON body with `post_url` from previous node
    - Headers: Content-Type: application/json; Authorization: Bearer Apify API Key
    - URL: Apify LinkedIn post detail scraper endpoint
    - Enable retry on failure

11. **Set Node: Get Post Information**
    - Extract Post ID, URL, text, author name, author headline, lead limit, number of pages, number of comments from previous node and Airtable record

12. **Code Node: Loop for More Comments**
    - Generate array from 1 to Number of Pages to loop scraping comments

13. **Limit Node: Page Limit for Demo**
    - Limit number of pages to scrape according to config

14. **SplitInBatches Node: Loop Over Comments Page**

15. **HTTP Request Node: Scrape LinkedIn Comments**
    - POST JSON body with limit, page number, post IDs
    - Headers with Apify API Key
    - Enable retry on failure

16. **Filter Node: Filter Author Comments Out**
    - Filter out comments where author name equals post author name or empty

17. **Set Node: Get Lead LinkedIn Profile URL**
    - Extract lead profile URL, comment text, comment ID

---

### Lead Data Enrichment

18. **Limit Node: Limit Profile Scraping**
    - Limit number of profiles to scrape by lead limit

19. **SplitInBatches Node: Loop Over LinkedIn Profile**

20. **HTTP Request Node: Scraping LinkedIn Profile**
    - POST JSON body with usernames array
    - Headers with Apify API Key
    - Max retries: 2

21. **Wait Node: Wait 1s**
    - Pause 1 second between requests

22. **Set Node: Get Lead Scraped Data**
    - Map profile JSON data to structured lead fields

23. **Airtable Node: Create Lead Record - Scraper**
    - Create new lead records with enriched data linked to lead list and post setup
    - Enable retry on failure

---

### Lead Qualification Branch (runQualification)

24. **Airtable Node: Get Lead List Record Data - Qualification**
    - Get lead list record by ID

25. **Set Node: Get Leads Data - Qualification**
    - Extract leads array

26. **SplitOut Node: Split Out Leads for Qualification Looping**

27. **Airtable Node: Get Lead Record Data - Qualification**
    - Get individual lead record by ID

28. **Set Node: Lead Data - Qualification**
    - Prepare lead data and qualification criteria for AI

29. **OpenAI Node: Lead Qualifier**
    - Model: `o4-mini`
    - Input: Lead info and qualification criteria
    - Output: JSON with `qualified` boolean and reason
    - Enable retry on failure

30. **Set Node: Qualification Result**
    - Extract qualification result

31. **If Node: If Qualified?**
    - Branch true/false

32. **Airtable Node: Update Lead Record - Qualification Result - True**
    - Update lead record as qualified

33. **Airtable Node: Update Lead Record - Qualification Result - False**
    - Update lead record as not qualified

34. **Merge Node: Merge qualification update branches**

---

### Connection Status Checking and Management

35. **Set Node: Get Lead Data - Active Campaign**
    - Extract leads and Unipile credentials from campaign record

36. **SplitOut Node: Split Out Lead - Active Campaign**

37. **Airtable Node: Get Lead Record - Campaign Operation**

38. **Filter Node: Allow Only Not Connected Status**

39. **Limit Node: Limit Lead Flow - Profile Retrieve**

40. **SplitInBatches Node: Loop Over Not Checked Lead**

41. **HTTP Request Node: Check Connection Status**
    - Call Unipile API for user connection status
    - Retry enabled

42. **Set Node: Edit Fields**
    - Extract provider id and relationship boolean

43. **If Node: If Not Connected**

44. **Airtable Node: Update Lead Record - Connected**

45. **Airtable Node: Update Lead Record - Not Connected**

46. **NoOp Node: Update Status About Connection Status Check**

47. **Merge Node: Merge connection status update branches**

---

### Connection Request Sending

48. **Filter Node: Allow Only Not Connected Status**

49. **Limit Node: Limit Lead Flow - Connection Request Sent**

50. **SplitInBatches Node: Loop Over Lead - Send Connection Request**

51. **HTTP Request Node: Send Connection Request**
    - POST to Unipile invite API with provider ID and account ID
    - Continue on error enabled

52. **If Node: If Invitation Sent**

53. **Airtable Node: Update Lead Record - Connection Pending**

54. **NoOp Node: Connection Request Failed**

55. **NoOp Node: Update Status Connection Batch Sent**

---

### Personalized DM Message Generation and Sending

56. **Filter Node: Allow Only Connected Lead**

57. **Limit Node: Limit DM To Send** (default 3)

58. **SplitInBatches Node: Loop Over Lead To DM**

59. **Set Node: Lead Data - DM Personalization**

60. **OpenAI Node: Personalized DM Message Writer**
    - Model: `o4-mini`
    - Input: Lead info, post info, offer, CTA, DM template, agent first name
    - Output: JSON with `DM Message`
    - Retry enabled

61. **HTTP Request Node: Send DM Message**
    - POST to Unipile chats API with attendee IDs, account ID, and text
    - Multipart form data
    - Retry enabled

62. **If Node: If ChatStarted**

63. **Airtable Node: Update Lead Record - DM Sent**

64. **NoOp Node: Send Message Failed**

65. **NoOp Node: Update Status of DM Message Sent**

---

### Campaign and Timer Management

66. **Schedule Trigger Node**
    - Interval: every 25-34 minutes randomized

67. **Airtable Node: Search Active Campaign Status**
    - Search for campaigns with status "Active"

68. **Airtable Node: Reset Number Active Campaign**

69. **Set Node: Get Active Campaign Record ID**

70. **SplitInBatches Node: Loop Over Active Campaigns - Counting**

71. **Airtable Node: Search LinkedIn Campaign Count Record**

72. **Set Node: Increment Count Active Campaign**

73. **Airtable Node: Create or update Active Campaign Count record**

74. **Aggregate Node: Aggregate All Active Record Into One**

75. **Airtable Node: Create or update LinkedIn Campaign Record ID**

76. **SplitOut Node: Split Out Active Campaign**

77. **Set Node: Campaign Operation Data**

78. **Code Node: Time Slot Matching Code**
    - Parses and compares current time to timer slots

79. **If Node: If Matching is True**

80. **If Node: If Current Timer Slot Match Corresponding Timer Slot?**

81. **SplitInBatches Node: Loop Over Active Campaigns1**

82. **Set Node: Get Active Campaign Data**

83. **Limit Node: Limit one Item to Update Current Timer Slot**

84. **If Nodes: If Corresponding Timer Slot = Timer 1/2/3 Slot?**

85. **Airtable Nodes: Change Next Current Timer to Timer 1/2/3 Slot**

86. **Merge Node: Merge - But Only One can Be true**

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| [AI-Powered LinkedIn DM Engine Banner](https://postimg.cc/hhS7JzvH)                                                                                                       | Branding image for the workflow                                                                              |
| Airtable Base Link: https://airtable.com/app8PsaTvvqCjxt6W/shrzHRzHQgH3G5p15                                                                                            | Base used for managing agencies, leads, campaigns, and configurations                                        |
| Video and community setup instructions available on Skool: https://www.skool.com/ruben-ai                                                                               | Workflow setup and usage guidance                                                                             |
| Disclaimer image about LinkedIn DM campaign use: https://postimg.cc/zHknQnQZ                                                                                             | Important usage disclaimer                                                                                    |
| Sticky notes throughout the workflow provide contextual explanations for each major block and step                                                                     | Visual aid for users                                                                                           |

---

# DISCLAIMER

The text provided is exclusively sourced from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.

---

This reference document provides a detailed, structured overview to understand, reproduce, and modify the "Automated LinkedIn Lead Generation & DM Outreach with Airtable, OpenAI, and Unipile" workflow in n8n. It includes careful node-by-node analysis, enabling anticipation of errors, edge cases, and integration points.