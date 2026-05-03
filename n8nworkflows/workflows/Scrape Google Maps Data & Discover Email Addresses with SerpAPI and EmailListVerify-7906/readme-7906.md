Scrape Google Maps Data & Discover Email Addresses with SerpAPI and EmailListVerify

https://n8nworkflows.xyz/workflows/scrape-google-maps-data---discover-email-addresses-with-serpapi-and-emaillistverify-7906


# Scrape Google Maps Data & Discover Email Addresses with SerpAPI and EmailListVerify

### 1. Workflow Overview

This workflow automates the process of scraping detailed business data from Google Maps using SerpAPI and enriches it with generic email addresses found via the EmailListVerify API. It is designed for users needing rich local business data, including contact info, reviews, ratings, and email addresses, at a lower cost and complexity than directly using Google Maps API.

The workflow logically divides into the following blocks:

- **1.1 Input Reception and Scheduling:** Reads Google Maps search URLs from Google Sheets and triggers the workflow manually or on a set schedule.
- **1.2 Search Parameter Extraction:** Parses keywords and geographic coordinates from input URLs for query construction.
- **1.3 SerpAPI Google Maps Data Scraping Loop:** Iteratively requests paginated search results from SerpAPI, extracts and merges all results.
- **1.4 Data Cleaning and Formatting:** Filters, merges, and deduplicates the scraped data.
- **1.5 Data Persistence:** Saves the cleaned data to a Google Sheets spreadsheet.
- **1.6 Email Address Discovery:** For listings with websites, extracts domain names and queries EmailListVerify for generic email addresses.
- **1.7 Final Data Update:** Appends or updates the spreadsheet with email-enriched data.

The workflow includes error handling to continue despite API errors and uses Google Sheets both as input and output for ease of use and data management.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Scheduling

**Overview:**  
This block initiates the workflow either manually or on a weekly schedule, and loads the Google Maps search URLs from a Google Sheet.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)  
- Run workflow once a week (Schedule Trigger)  
- Google Sheets - Get searches to scrap

**Node Details:**

- **When clicking "Execute Workflow"**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual start of the workflow.  
  - *Input:* None  
  - *Output:* Starts flow to the Google Sheets node.  
  - *Edge Cases:* None.  
  - *Version:* v1

- **Run workflow once a week**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the workflow every week.  
  - *Parameters:* Interval set to 1 week.  
  - *Edge Cases:* Workflow won’t run if n8n instance is offline at scheduled time.  
  - *Version:* v1.1

- **Google Sheets - Get searches to scrap**  
  - *Type:* Google Sheets  
  - *Role:* Reads the list of Google Maps URLs to scrape from a Google Sheet (sheet "Add your search url here", first tab).  
  - *Parameters:* Document ID and Sheet ID point to a specific spreadsheet.  
  - *Credentials:* Uses Google Sheets OAuth2.  
  - *Input:* Trigger nodes.  
  - *Output:* List of URLs for subsequent processing.  
  - *Edge Cases:* Failures if credentials expire or the sheet is inaccessible.  

---

#### 2.2 Search Parameter Extraction

**Overview:**  
Extracts the search keyword and geographic location (latitude/longitude) from each Google Maps search URL.

**Nodes Involved:**  
- Extract keyword and location from URL

**Node Details:**

- **Extract keyword and location from URL**  
  - *Type:* Set  
  - *Role:* Uses regex expressions on the input URL to extract:  
     - `keyword`: part matching `/search/(.*?)/`  
     - `geo`: geographic coordinates from the `@` symbol segment.  
  - *Input:* URL string from Google Sheets node.  
  - *Output:* JSON with `keyword` and `geo` fields for API queries.  
  - *Edge Cases:* Regex fails if URL format changes or is malformed.  
  - *Version:* v3.2

---

#### 2.3 SerpAPI Google Maps Data Scraping Loop

**Overview:**  
Sends queries to SerpAPI to retrieve paginated Google Maps search results, extracts the "start" parameter for pagination, and repeats until no next page exists.

**Nodes Involved:**  
- SERPAPI - Scrape Google Maps URL  
- Extract next start value  
- Continue IF Loop is complete  
- Merge all values from SERPAPI  
- Split out items

**Node Details:**

- **SERPAPI - Scrape Google Maps URL**  
  - *Type:* HTTP Request  
  - *Role:* Calls SerpAPI with parameters: engine = google_maps, query = keyword, location = geo, type=search, start=page offset.  
  - *Credentials:* Uses SerpAPI API key (credential required).  
  - *Error Handling:* Configured to continue on error, preventing workflow halt if request fails.  
  - *Input:* Pagination start parameter and search parameters.  
  - *Output:* JSON response with search results and pagination info.  
  - *Version:* v4.1  
  - *Edge Cases:* API key invalid, quota exceeded, network errors, malformed responses.

- **Extract next start value**  
  - *Type:* Code  
  - *Role:* Parses the `next` URL from SerpAPI pagination to extract the `start` parameter for the next page.  
  - *Input:* SerpAPI response JSON.  
  - *Output:* Adds or updates `$json.start` for pagination.  
  - *Version:* v2  
  - *Edge Cases:* Missing pagination info, malformed URLs.

- **Continue IF Loop is complete**  
  - *Type:* IF  
  - *Role:* Checks if `start` and SerpAPI `next` link exist; if yes, loops to SERPAPI node for another page, else proceeds.  
  - *Input:* Pagination data.  
  - *Output:* Loop continuation or exit.  
  - *Version:* v1  
  - *Edge Cases:* Infinite loop risk if pagination parameters are malformed.

- **Merge all values from SERPAPI**  
  - *Type:* Code  
  - *Role:* Iterates over all SERPAPI node outputs, collects and aggregates all `local_results` arrays from each page into a single array.  
  - *Input:* Multiple paginated search results.  
  - *Output:* Aggregated array of all results.  
  - *Version:* v2  
  - *Edge Cases:* Empty or missing local_results; error causes early return.

- **Split out items**  
  - *Type:* Item Lists  
  - *Role:* Splits the aggregated array `allData` into individual items for further processing.  
  - *Input:* Aggregated data array.  
  - *Output:* Individual place entries.  
  - *Version:* v3.1

---

#### 2.4 Data Cleaning and Formatting

**Overview:**  
Filters out empty entries, merges nested data structures into a flat list, removes duplicates by `place_id`, and prepares data for output.

**Nodes Involved:**  
- Remove empty values  
- Transform data in the right format  
- Remove duplicate items

**Node Details:**

- **Remove empty values**  
  - *Type:* Filter  
  - *Role:* Filters out any records where the first element in the array is empty (likely malformed or incomplete).  
  - *Input:* Split items.  
  - *Output:* Filtered items.  
  - *Version:* v1

- **Transform data in the right format**  
  - *Type:* Code  
  - *Role:* Flattens nested JSON arrays from multiple inputs into a single array of place objects, removing nulls.  
  - *Logic:* Iterates over all input entries, merges their JSON values into a single list.  
  - *Input:* Filtered items.  
  - *Output:* Cleaned and flattened list of place objects.  
  - *Version:* v2  
  - *Edge Cases:* Null or unexpected data structures.

- **Remove duplicate items**  
  - *Type:* Item Lists  
  - *Role:* Removes duplicates by comparing `place_id` field, ensuring uniqueness of places.  
  - *Input:* Transformed data.  
  - *Output:* Deduplicated list.  
  - *Version:* v3.1

---

#### 2.5 Data Persistence

**Overview:**  
Saves the cleaned and deduplicated Google Maps data into a Google Sheet for storage and later use.

**Nodes Involved:**  
- Save google map data to googlesheet

**Node Details:**

- **Save google map data to googlesheet**  
  - *Type:* Google Sheets  
  - *Role:* Appends or updates rows in a Google Sheet ("Results" tab) with place details.  
  - *Fields:* Maps many place attributes such as title, phone, website, rating, reviews, place_id, amenities, coordinates, and more.  
  - *Matching:* Uses `place_id` to deduplicate/update rows.  
  - *Credentials:* Google Sheets OAuth2 required.  
  - *Input:* Deduplicated place data.  
  - *Output:* Data saved to sheet, triggers next block to check website presence.  
  - *Version:* v4.2  
  - *Edge Cases:* Google Sheets quota limits, API errors.

---

#### 2.6 Email Address Discovery

**Overview:**  
Checks if each listing has a website, extracts domain names from URLs, then queries EmailListVerify API to discover generic emails associated with those domains.

**Nodes Involved:**  
- Check if the listing has a website  
- Transform website into domain name  
- Use EmailListVerify API to find generic emails  
- Add rows in Google Sheets

**Node Details:**

- **Check if the listing has a website**  
  - *Type:* IF  
  - *Role:* Filters listings that have a valid website field that does not start with "[undefined]".  
  - *Input:* Place data with website field.  
  - *Output:* Routes listings with websites to domain extraction, others are ignored.  
  - *Version:* v2.2

- **Transform website into domain name**  
  - *Type:* Code  
  - *Role:* Extracts the domain name from the full website URL, removing "www." prefix if present.  
  - *Input:* Listings with website URLs.  
  - *Output:* Adds `domain` field for API queries.  
  - *Version:* v2  
  - *Edge Cases:* Malformed URLs, missing slashes, unexpected URL formats.

- **Use EmailListVerify API to find generic emails**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to EmailListVerify API with domain in JSON body to retrieve generic emails.  
  - *Credentials:* HTTP Header Auth with EmailListVerify API key required.  
  - *Input:* Domain names.  
  - *Output:* API response with email data.  
  - *Version:* v4.2  
  - *Edge Cases:* API rate limits, invalid API key, network issues.

- **Add rows in Google Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Updates the "Emails" tab in the spreadsheet with the enriched data including found emails.  
  - *Credentials:* Google Sheets OAuth2 required.  
  - *Input:* Email enriched listings.  
  - *Output:* Persisted records with emails.  
  - *Version:* v4.2

---

### 3. Summary Table

| Node Name                            | Node Type          | Functional Role                                    | Input Node(s)                          | Output Node(s)                           | Sticky Note                                                                                         |
|------------------------------------|--------------------|---------------------------------------------------|--------------------------------------|-----------------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow"   | Manual Trigger     | Manual start trigger                              | -                                    | Google Sheets - Get searches to scrap   | Click on Execute Workflow to run the workflow manually                                            |
| Run workflow once a week            | Schedule Trigger   | Weekly automatic trigger                          | -                                    | Google Sheets - Get searches to scrap   | Adjust frequency to your own needs                                                                |
| Google Sheets - Get searches to scrap | Google Sheets      | Reads input URLs from Google Sheet                | Manual Trigger, Schedule Trigger     | Extract keyword and location from URL   | Copy my template and connect it to n8n https://docs.google.com/spreadsheets/d/1d5m3tmmqM1TPL0fnrrbhaPdoXVs4EIrniDI-nv01UKg/edit?gid=0#gid=0 |
| Extract keyword and location from URL | Set               | Extracts keyword and geo params from URL          | Google Sheets - Get searches to scrap | SERPAPI - Scrape Google Maps URL        | Scrap Google Map                                                                                   |
| SERPAPI - Scrape Google Maps URL   | HTTP Request      | Calls SerpAPI to retrieve Google Maps search data | Extract keyword and location from URL | Extract next start value, error output  | Add your SERPAPI API Key Start a free trial at serpapi.com and get your API key in "Your account" section |
| Extract next start value            | Code               | Extracts next pagination start value from response| SERPAPI - Scrape Google Maps URL     | Continue IF Loop is complete             |                                                                                                   |
| Continue IF Loop is complete        | IF                 | Controls pagination loop                          | Extract next start value              | SERPAPI - Scrape Google Maps URL, Merge all values from SERPAPI |                                                                                                   |
| Merge all values from SERPAPI       | Code               | Aggregates all paginated results                  | Continue IF Loop is complete          | Split out items                         |                                                                                                   |
| Split out items                    | Item Lists          | Splits aggregated array into individual items    | Merge all values from SERPAPI         | Remove empty values                     |                                                                                                   |
| Remove empty values                | Filter              | Filters out empty entries                          | Split out items                      | Transform data in the right format       |                                                                                                   |
| Transform data in the right format | Code               | Flattens and merges nested data                   | Remove empty values                  | Remove duplicate items                   |                                                                                                   |
| Remove duplicate items             | Item Lists          | Removes duplicate places by place_id              | Transform data in the right format   | Save google map data to googlesheet     |                                                                                                   |
| Save google map data to googlesheet| Google Sheets      | Saves place data to Google Sheets                  | Remove duplicate items               | Check if the listing has a website       |                                                                                                   |
| Check if the listing has a website | IF                 | Filters listings with websites                     | Save google map data to googlesheet  | Transform website into domain name       |                                                                                                   |
| Transform website into domain name | Code               | Extracts domain from website URLs                  | Check if the listing has a website    | Use EmailListVerify API to find generic emails |                                                                                                   |
| Use EmailListVerify API to find generic emails | HTTP Request      | Queries EmailListVerify API for generic emails    | Transform website into domain name    | Add rows in Google Sheets                | Add your ELV API Key Start a free trial at EmailListVerify.com and get your API key. Find email address with EmailListVerify API if the website is known |
| Add rows in Google Sheets          | Google Sheets      | Updates sheet with email-enriched data             | Use EmailListVerify API to find generic emails | -                                       |                                                                                                   |
| Sticky Note1                      | Sticky Note         | Guidance on setting execution frequency            | -                                    | -                                       | Adjust frequency to your own needs                                                                |
| Sticky Note2                      | Sticky Note         | Template spreadsheet link                           | -                                    | -                                       | Copy my template and connect it to n8n Template link: https://docs.google.com/spreadsheets/d/1d5m3tmmqM1TPL0fnrrbhaPdoXVs4EIrniDI-nv01UKg/edit?gid=0#gid=0 |
| Sticky Note3                      | Sticky Note         | Reminder to add SerpAPI API key                     | -                                    | -                                       | Add your SERPAPI API Key Start a free trial at serpapi.com and get your API key in "Your account" section |
| Sticky Note4                      | Sticky Note         | Full workflow description and link                  | -                                    | -                                       | Read Me: Full guide at https://lempire.notion.site/Scrape-Google-Maps-places-with-n8n-b7f1785c3d474e858b7ee61ad4c21136?pvs=4 |
| Sticky Note5                      | Sticky Note         | Reminder to add EmailListVerify API key             | -                                    | -                                       | Add your ELV API Key Start a free trial at EmailListVerify.com and get your API key.               |
| Sticky Note6                      | Sticky Note         | Email discovery description                         | -                                    | -                                       | Find email address with EmailListVerify API if the website is known                               |
| Sticky Note7                      | Sticky Note         | Identification of Google Maps scraping block       | -                                    | -                                       | Scrap Google Map                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** named "When clicking \"Execute Workflow\"" for manual runs.  
   - Add a **Schedule Trigger** named "Run workflow once a week" with an interval of 1 week for automatic runs.

2. **Google Sheets Input Node:**  
   - Add a **Google Sheets** node named "Google Sheets - Get searches to scrap".  
   - Configure OAuth2 credentials.  
   - Set Document URL to your input spreadsheet containing Google Maps search URLs.  
   - Set Sheet Name to the tab where URLs are listed (default first tab).

3. **Extract Search Parameters:**  
   - Add a **Set** node named "Extract keyword and location from URL".  
   - Add two fields:  
     - `keyword` with expression: `{{$json.URL.match(/\/search\/(.*?)\//)[1]}}`  
     - `geo` with expression: `{{$json.URL.match(/(@[^\/?]+)/)[1]}}`

4. **SerpAPI Request Node:**  
   - Add an **HTTP Request** node named "SERPAPI - Scrape Google Maps URL".  
   - Set method to GET.  
   - URL: `https://serpapi.com/search.json`  
   - Query parameters:  
     - engine: `google_maps`  
     - q: `={{$json.keyword}}`  
     - ll: `={{$json.geo}}`  
     - type: `search`  
     - start: `={{$json.start || 0}}`  
   - Authenticate with SerpAPI credentials (set your API key).  
   - On error: continue with error output.

5. **Extract Pagination Start Parameter:**  
   - Add a **Code** node named "Extract next start value".  
   - JavaScript code to parse `serpapi_pagination.next` URL and extract `start` value, saving it to `$json.start`.

6. **Pagination Control IF Node:**  
   - Add an **IF** node "Continue IF Loop is complete".  
   - Condition: Check if both `$json.search_parameters.start` and `$json.serpapi_pagination.next` are not empty.  
   - Connect TRUE output back to the SerpAPI HTTP Request node for looping.

7. **Merge All Pages:**  
   - Add a **Code** node named "Merge all values from SERPAPI".  
   - JavaScript merges all `local_results` from all paginated responses into one array.

8. **Split Aggregated Items:**  
   - Add an **Item Lists** node "Split out items" and configure to split by `allData` field.

9. **Remove Empty Entries:**  
   - Add a **Filter** node "Remove empty values".  
   - Filter where first element is not empty.

10. **Transform Data Format:**  
    - Add a **Code** node "Transform data in the right format".  
    - Flatten nested JSON objects into a merged list, excluding nulls.

11. **Remove Duplicates:**  
    - Add an **Item Lists** node "Remove duplicate items".  
    - Configure to remove duplicates comparing `place_id`.

12. **Save to Google Sheets:**  
    - Add a **Google Sheets** node "Save google map data to googlesheet".  
    - Use the correct spreadsheet and sheet ("Results" tab).  
    - Map all relevant place fields (title, phone, website, rating, reviews, place_id, etc.).  
    - Set operation to append or update using `place_id` as the matching column.  
    - Use OAuth2 credentials.

13. **Check for Website Presence:**  
    - Add an **IF** node "Check if the listing has a website".  
    - Condition: Website field does not start with "[undefined]".

14. **Extract Domain from Website:**  
    - Add a **Code** node "Transform website into domain name".  
    - JavaScript extracts domain from website URL, removing "www." prefix.

15. **EmailListVerify API Call:**  
    - Add an **HTTP Request** node "Use EmailListVerify API to find generic emails".  
    - Set method POST, URL: `https://api.emaillistverify.com/api/findContact`.  
    - JSON body: `{"domain": "{{$json.domain}}"}`.  
    - Authenticate with HTTP Header Auth using your EmailListVerify API key.

16. **Save Emails to Google Sheets:**  
    - Add a **Google Sheets** node "Add rows in Google Sheets".  
    - Target "Emails" tab in the same spreadsheet.  
    - Map relevant fields including email and confidence score.  
    - Use OAuth2 credentials.

17. **Connect All Nodes:**  
    - Connect triggers to Google Sheets input.  
    - Chain nodes as per pagination loop and processing order described.  
    - Set proper error handling on HTTP requests to continue on failures.

18. **Add Sticky Notes:**  
    - Add sticky notes near relevant nodes to provide usage instructions and reminders such as adding API keys and URLs.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow allows scraping Google Maps data efficiently with SerpAPI, providing more data points at lower cost.     | https://lempire.notion.site/Scrape-Google-Maps-places-with-n8n-b7f1785c3d474e858b7ee61ad4c21136?pvs=4                   |
| Copy the Google Sheets template and connect it to n8n input for URLs.                                                  | https://docs.google.com/spreadsheets/d/1d5m3tmmqM1TPL0fnrrbhaPdoXVs4EIrniDI-nv01UKg/edit?gid=0#gid=0                     |
| Add your SerpAPI API key in the credentials panel to enable scraping.                                                  | serpapi.com - Your account section for API key                                                                            |
| Add your EmailListVerify API key to find generic emails for known websites.                                            | https://app.emaillistverify.com/api                                                                                        |
| Adjust workflow execution frequency to meet your requirements (manual or scheduled).                                  | Sticky note near triggers                                                                                                 |
| Click "Execute Workflow" manually to run the workflow on demand.                                                       | Sticky note near manual trigger                                                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal, offensive, or protected elements. All data processed is lawful and public.