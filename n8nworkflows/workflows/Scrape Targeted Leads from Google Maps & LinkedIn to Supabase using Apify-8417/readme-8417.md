Scrape Targeted Leads from Google Maps & LinkedIn to Supabase using Apify

https://n8nworkflows.xyz/workflows/scrape-targeted-leads-from-google-maps---linkedin-to-supabase-using-apify-8417


# Scrape Targeted Leads from Google Maps & LinkedIn to Supabase using Apify

---

### 1. Workflow Overview

This workflow automates the process of scraping targeted leads from Google Maps and LinkedIn using Apify actors, then cleans, formats, and stores the lead data into a Supabase database. It is designed primarily for sales teams, recruiters, and marketers looking to build prospect lists based on specific criteria such as job titles, industries, and locations.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives user input via a web form specifying search criteria (title/industry, location, source, number of results).
- **1.2 Source Routing:** Routes the workflow based on the selected data source (Google Maps, LinkedIn, or both).
- **1.3 Data Extraction:** Executes Apify actors to scrape leads from LinkedIn and/or Google Maps.
- **1.4 Data Formatting:** Cleans and structures the scraped data for each source separately.
- **1.5 Data Storage:** Saves the formatted data into corresponding Supabase tables.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Captures the user’s search criteria through a form and triggers the workflow start.

**Nodes Involved:**  
- Input desired lead

**Node Details:**

- **Input desired lead**  
  - Type: Form Trigger  
  - Role: Entry point node that collects user input via a web form.  
  - Configuration:  
    - Form fields:  
      - Title/Industry (required, text)  
      - Location (required, text)  
      - Source (required, dropdown with options: Google Maps, LinkedIn, Both)  
      - Number of results (optional, number)  
    - Webhook ID configured for external form submissions.  
  - Input/Output: Trigger node, outputs JSON with user inputs.  
  - Edge cases: Missing required fields; invalid number values; webhook connectivity issues.

---

#### 2.2 Source Routing

**Overview:**  
Determines which data scraping path(s) to execute based on the “Source” field selected in the form.

**Nodes Involved:**  
- Route source

**Node Details:**

- **Route source**  
  - Type: Switch  
  - Role: Routes the workflow into one or both scraping paths depending on source choice.  
  - Configuration:  
    - Switch on JSON field `$json.Source`  
    - Conditions:  
      - “Google Maps” → output “GoogleMaps”  
      - “LinkedIn” → output “LinkedIn”  
      - “Both” → output “Both” (runs LinkedIn then Google Maps)  
  - Input: Receives form data JSON.  
  - Output: One or two parallel branches to respective scrapers.  
  - Edge cases: Unrecognized source values; case sensitivity issues.

---

#### 2.3 Data Extraction

**Overview:**  
Runs Apify actors to scrape lead data from the selected platforms using the parameters from input.

**Nodes Involved:**  
- linkedin_dataset  
- googlemaps_dataset

**Node Details:**

- **linkedin_dataset**  
  - Type: Apify (Run actor and get dataset)  
  - Role: Runs the LinkedIn Profile Search Scraper actor on Apify to gather LinkedIn profiles matching criteria.  
  - Configuration:  
    - Actor ID for “LinkedIn Profile Search Scraper No Cookies”  
    - Input JSON:  
      - `locations`: array with user location  
      - `maxItems`: number of results requested  
      - `profileScraperMode`: “Full” (complete profile data)  
      - `searchQuery`: user input title/industry  
    - Authentication: Apify OAuth2 credentials  
    - Memory: 2048 MB  
  - Input: JSON from the routing node  
  - Output: Dataset JSON array of LinkedIn profiles  
  - Edge cases: API throttling, actor runtime errors, missing data fields.

- **googlemaps_dataset**  
  - Type: Apify (Run actor and get dataset)  
  - Role: Runs Google Maps Scraper actor on Apify to collect business leads based on location and industry.  
  - Configuration:  
    - Actor ID for “Google Maps Scraper (compass/crawler-google-places)”  
    - Input JSON:  
      - `locationQuery`: user location  
      - `maxCrawledPlacesPerSearch`: number of results requested  
      - `searchStringsArray`: array with title/industry  
      - Various scraping flags disabled (no contacts, no images, etc.)  
      - Language set to English  
    - Authentication: Apify OAuth2 credentials  
  - Input: JSON from the routing node  
  - Output: Dataset JSON array of Google Maps places  
  - Edge cases: API rate limits, incomplete place data, location not found.

---

#### 2.4 Data Formatting

**Overview:**  
Processes and structures the raw data from Apify datasets into a consistent format ready for database storage.

**Nodes Involved:**  
- get_linkedin  
- set_google_maps_column

**Node Details:**

- **get_linkedin**  
  - Type: Set  
  - Role: Extracts and formats key LinkedIn profile fields from the scraped JSON.  
  - Configuration:  
    - Combines first and last name into a single “name” field  
    - Maps fields such as publicIdentifier, linkedinUrl, headline, about, premium, verified, openProfile, topSkills, connectionsCount, followerCount  
    - Extracts last experience’s company LinkedIn URL, name, and duration as a combined string  
    - Extracts latest education school name  
  - Input: Raw LinkedIn dataset JSON  
  - Output: Cleaned JSON with mapped fields for storage  
  - Edge cases: Missing experience or education arrays; null or undefined fields.

- **set_google_maps_column**  
  - Type: Set  
  - Role: Formats Google Maps place data fields for database insertion.  
  - Configuration:  
    - Maps title, categoryName, address, neighborhood, street, city, postalCode, state, countryCode, website, phone, phoneUnformatted  
    - Converts location object to string with lat/lng fields  
    - Maps totalScore numeric field  
  - Input: Raw Google Maps dataset JSON  
  - Output: Formatted JSON for Supabase  
  - Edge cases: Missing or malformed location data; absent phone or website fields.

---

#### 2.5 Data Storage

**Overview:**  
Stores the formatted lead data into Supabase tables dedicated to each source.

**Nodes Involved:**  
- save_linkedin  
- save_googlemaps

**Node Details:**

- **save_linkedin**  
  - Type: Supabase node  
  - Role: Inserts or updates records in the Supabase “linkedin” table.  
  - Configuration:  
    - Maps fields from processed JSON to corresponding database columns (publicidentifier, linkedinurl, name, headline, about, premium, verified, openprofile, topskills, connectionscount, followercount, latest_experience, education)  
    - Uses Supabase API credentials  
  - Input: Cleaned LinkedIn JSON from get_linkedin node  
  - Output: Confirmation of database insert/update  
  - Edge cases: API authentication failure, data validation errors, database connectivity issues.

- **save_googlemaps**  
  - Type: Supabase node  
  - Role: Inserts or updates records in the Supabase “googlemaps” table.  
  - Configuration:  
    - Maps fields from processed JSON to corresponding database columns (title, category_name, address, neighborhood, street, city, postal_code, state, country_code, website, phone, phone_unformatted, location, total_score)  
    - Uses Supabase API credentials  
  - Input: Cleaned Google Maps JSON from set_google_maps_column node  
  - Output: Confirmation of database insert/update  
  - Edge cases: Same as save_linkedin node.

---

### 3. Summary Table

| Node Name            | Node Type                   | Functional Role                     | Input Node(s)          | Output Node(s)        | Sticky Note                                                   |
|----------------------|-----------------------------|-----------------------------------|-----------------------|-----------------------|---------------------------------------------------------------|
| Input desired lead    | Form Trigger                | Capture user search criteria      | —                     | Route source          | Explains form fields and purpose                              |
| Route source          | Switch                     | Routes based on selected source   | Input desired lead     | linkedin_dataset, googlemaps_dataset | Describes routing logic for sources                            |
| linkedin_dataset      | Apify (Run actor & dataset) | Scrape LinkedIn profiles          | Route source           | get_linkedin          |                                                               |
| get_linkedin         | Set                         | Format LinkedIn data               | linkedin_dataset       | save_linkedin         | Describes LinkedIn data processing                             |
| save_linkedin         | Supabase                    | Store LinkedIn leads              | get_linkedin           | —                     |                                                               |
| googlemaps_dataset    | Apify (Run actor & dataset) | Scrape Google Maps places         | Route source           | set_google_maps_column |                                                               |
| set_google_maps_column| Set                         | Format Google Maps data            | googlemaps_dataset     | save_googlemaps       | Describes Google Maps data processing                          |
| save_googlemaps       | Supabase                    | Store Google Maps leads           | set_google_maps_column | —                     |                                                               |
| Sticky Note           | Sticky Note                 | Main workflow overview            | —                     | —                     | Provides high-level workflow explanation                      |
| Sticky Note1          | Sticky Note                 | Data processing & storage overview| —                     | —                     | Details data cleaning for both data sources                   |
| Sticky Note2          | Sticky Note                 | Form input explanation            | —                     | —                     | Describes form fields with examples                           |
| Sticky Note3          | Sticky Note                 | LinkedIn profile reference        | —                     | —                     | Link to LinkedIn profile of workflow author                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node:**  
   - Name: `Input desired lead`  
   - Type: Form Trigger  
   - Configure form fields:  
     - Title/Industry (text, required)  
     - Location (text, required)  
     - Source (dropdown: Google Maps, LinkedIn, Both; required)  
     - Number of results (number, optional)  
   - Set a webhook ID for receiving external form submissions.

2. **Add a Switch node:**  
   - Name: `Route source`  
   - Type: Switch  
   - Condition: Evaluate `$json.Source` with three cases:  
     - Equals “Google Maps” → output “GoogleMaps”  
     - Equals “LinkedIn” → output “LinkedIn”  
     - Equals “Both” → output “Both”  
   - Connect `Input desired lead` output to this node input.

3. **Add Apify nodes for scraping:**  
   - **LinkedIn Scraper:**  
     - Name: `linkedin_dataset`  
     - Type: Apify (Run actor and get dataset)  
     - Actor ID: `M2FMdjRVeF1HPGFcc` (LinkedIn Profile Search Scraper)  
     - Set memory to 2048 MB  
     - Input JSON:  
       ```json
       {
         "locations": ["{{$json.Location}}"],
         "maxItems": {{$json["Number of results"]}},
         "profileScraperMode": "Full",
         "searchQuery": "{{$json["Title/Industry"]}}"
       }
       ```  
     - Authentication: Apify OAuth2 credentials (create or reuse, e.g., “Apify account 2”)  
     - Connect from `Route source` output “LinkedIn” and “Both” (first branch).

   - **Google Maps Scraper:**  
     - Name: `googlemaps_dataset`  
     - Type: Apify (Run actor and get dataset)  
     - Actor ID: `nwua9Gu5YrADL7ZDj` (Google Maps Scraper)  
     - Input JSON:  
       ```json
       {
         "includeWebResults": false,
         "language": "en",
         "locationQuery": "{{$json.Location}}",
         "maxCrawledPlacesPerSearch": {{$json["Number of results"]}},
         "maxImages": 0,
         "maximumLeadsEnrichmentRecords": 0,
         "scrapeContacts": false,
         "scrapeDirectories": false,
         "scrapeImageAuthors": false,
         "scrapePlaceDetailPage": false,
         "scrapeReviewsPersonalData": true,
         "scrapeTableReservationProvider": false,
         "searchStringsArray": ["{{$json["Title/Industry"]}}"],
         "skipClosedPlaces": false
       }
       ```  
     - Authentication: Apify OAuth2 credentials (same as above)  
     - Connect from `Route source` output “GoogleMaps” and “Both” (second branch).

4. **Add data formatting Set nodes:**  
   - **For LinkedIn:**  
     - Name: `get_linkedin`  
     - Type: Set  
     - Configure fields:  
       - Combine firstName and lastName to `name`  
       - Map other fields: publicIdentifier, linkedinUrl, headline, about, premium (boolean), verified (boolean), openProfile (boolean), topSkills, connectionsCount (number), followerCount (number), latest_experience (last experience company details concatenated), education (last school name)  
     - Connect output of `linkedin_dataset` to this node.

   - **For Google Maps:**  
     - Name: `set_google_maps_column`  
     - Type: Set  
     - Configure fields mapping title, categoryName, address, neighborhood, street, city, postalCode, state, countryCode, website, phone, phoneUnformatted, location as string with lat/lng, totalScore (number)  
     - Connect output of `googlemaps_dataset` to this node.

5. **Add Supabase nodes for data storage:**  
   - Create Supabase credentials with API key and URL (e.g., "Supabase account 2").

   - **LinkedIn storage:**  
     - Name: `save_linkedin`  
     - Type: Supabase  
     - Target table: “linkedin”  
     - Map fields from `get_linkedin` output JSON to table columns accordingly.  
     - Connect output of `get_linkedin` to this node.

   - **Google Maps storage:**  
     - Name: `save_googlemaps`  
     - Type: Supabase  
     - Target table: “googlemaps”  
     - Map fields from `set_google_maps_column` output JSON to table columns accordingly.  
     - Connect output of `set_google_maps_column` to this node.

6. **Connect nodes accordingly:**  
   - `Input desired lead` → `Route source`  
   - `Route source` → `linkedin_dataset` (for LinkedIn and Both)  
   - `Route source` → `googlemaps_dataset` (for Google Maps and Both)  
   - `linkedin_dataset` → `get_linkedin` → `save_linkedin`  
   - `googlemaps_dataset` → `set_google_maps_column` → `save_googlemaps`

7. **Optional:** Add sticky notes for documentation inside n8n canvas, explaining workflow overview, form input, and data processing steps.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Main workflow automatically scrapes targeted leads from LinkedIn and Google Maps and stores data in Supabase for prospecting purposes.                         | Workflow Overview Sticky Note                                    |
| Input form collects Title/Industry, Location, Source (Google Maps, LinkedIn, Both), and Number of results.                                                      | Form Input Sticky Note                                           |
| Data processing includes cleaning and formatting for each data source before storage.                                                                           | Data Processing & Storage Sticky Note                            |
| LinkedIn profile author: [David Tumusime LinkedIn](https://www.linkedin.com/in/tumusime-david/)                                                                | Sticky Note with LinkedIn Profile Link                           |
| Uses Apify Actors: “LinkedIn Profile Search Scraper No Cookies” and “Google Maps Scraper (compass/crawler-google-places)” with OAuth2 authentication            | Apify actors documentation: https://apify.com/actors            |
| Supabase used as backend database for storing leads with dedicated tables `linkedin` and `googlemaps`                                                           | Supabase docs: https://supabase.com/docs                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.