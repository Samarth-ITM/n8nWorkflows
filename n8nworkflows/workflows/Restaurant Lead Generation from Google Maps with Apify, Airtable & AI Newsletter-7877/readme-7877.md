Restaurant Lead Generation from Google Maps with Apify, Airtable & AI Newsletter

https://n8nworkflows.xyz/workflows/restaurant-lead-generation-from-google-maps-with-apify--airtable---ai-newsletter-7877


# Restaurant Lead Generation from Google Maps with Apify, Airtable & AI Newsletter

### 1. Workflow Overview

This workflow automates lead generation for restaurants by scraping Google Maps data, filtering and enriching it, and then creating an AI-generated newsletter distributed via email. The primary use case targets marketing or sales teams seeking curated restaurant leads from specific locations, leveraging data enrichment and natural language generation to transform raw data into actionable insights.

Logical blocks:

- **1.1 Input Reception and Google Maps Scraping:** User inputs location and scraping parameters via a form; an Apify Google Maps scraper actor is triggered to collect restaurant data.

- **1.2 Dataset Retrieval and Data Extraction:** Once scraping succeeds, the dataset of places is fetched; relevant fields are extracted and normalized.

- **1.3 Filtering, Sorting and Lead Creation:** Extracted data is sorted by reviews and ratings; leads with sufficient reviews are filtered and inserted into Airtable.

- **1.4 Data Aggregation for Newsletter:** Leads with reviews over 500 are grouped and aggregated into a summary string.

- **1.5 AI Newsletter Generation and Emailing:** The aggregated data is fed to an OpenAI-powered AI agent to generate a newsletter, which is then sent via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Google Maps Scraping

- **Overview:**  
  Collects user input from a web form and runs a Google Maps scraping actor on Apify using the input data.

- **Nodes Involved:**  
  - On form submission  
  - Run an Actor  
  - If  
  - No Operation, do nothing  
  - Sticky Note (Run an APIfy Google Map Actor)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point collecting user input (Location, what to scrape, number of items)  
    - Config: Dropdown with predefined location (e.g., Bangkok, Thailand), text input for scrape target, numeric input for max items  
    - Output: Passes JSON with form data fields  
    - Edge Cases: Invalid or missing form fields; user cancels submission  

  - **Run an Actor**  
    - Type: Apify Actor Execution  
    - Role: Executes the "Google Maps Scraper" actor from Apify store with dynamic inputs  
    - Config: Actor ID "nwua9Gu5YrADL7ZDj" (Google Maps Scraper), custom JSON body including location, search terms, max items, and scraping options (contacts, directories, reviews, etc.)  
    - Credentials: Apify API key required  
    - Input: Takes form submission JSON for parameters  
    - Output: Job status and execution metadata  
    - Edge Cases: API auth failures, actor runtime errors, rate limits, malformed inputs  

  - **If**  
    - Type: Conditional  
    - Role: Checks if the actor run status equals "SUCCEEDED"  
    - Input: Output from "Run an Actor"  
    - Output: Routes to dataset retrieval if succeeded, else to no-op  
    - Edge Cases: Unexpected status values, null or missing fields  

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Terminates the flow on failure or non-success status  

  - **Sticky Note**  
    - Content: Explains the actor execution flow and conditional branching  

#### 2.2 Dataset Retrieval and Data Extraction

- **Overview:**  
  Retrieves the dataset produced by the Apify actor and extracts essential restaurant data fields for further processing.

- **Nodes Involved:**  
  - Get dataset items  
  - Extract Essentials  
  - Sort by Review Count and Rating  
  - Sticky Note1 (Extract data from Dataset)  
  - Sticky Note2 (Extract only Essentials)

- **Node Details:**

  - **Get dataset items**  
    - Type: Apify Dataset Retrieval  
    - Role: Fetches scraped place data from Apify dataset using dataset ID from actor run output  
    - Credentials: Apify API key  
    - Input: Dataset ID dynamically retrieved  
    - Output: Raw JSON array of places  

  - **Extract Essentials**  
    - Type: Set  
    - Role: Normalizes and extracts key fields such as title, category, price, rating, review counts, address, phone, and URL  
    - Config: Assigns new fields by accessing and concatenating JSON fields, with fallback for phone  
    - Input: Raw place data from dataset  
    - Output: Simplified JSON objects with essentials only  
    - Edge Cases: Missing fields, null values, inconsistent address formats  

  - **Sort by Review Count and Rating**  
    - Type: Sort  
    - Role: Orders leads descending by review count, then rating  
    - Input: Extracted essentials  
    - Output: Sorted list for filtering and lead creation  

  - **Sticky Notes:**  
    - Highlight data extraction and sorting logic  

#### 2.3 Filtering, Sorting and Lead Creation

- **Overview:**  
  Filters the data for restaurants with more than 500 reviews, creates grouped textual data for newsletter, and inserts leads into Airtable.

- **Nodes Involved:**  
  - Reviews > 500 (If)  
  - Group Essentials  
  - Aggregate  
  - Lead Creator (Airtable)  
  - Sticky Note3 (Make an entry to Airtable)  
  - Sticky Note4 (Aggregate for the Newsletter)

- **Node Details:**

  - **Reviews > 500**  
    - Type: If  
    - Role: Filters restaurants with review count greater than 500  
    - Input: Sorted essentials  
    - Output: Passes qualifying leads to grouping, else no-op  

  - **Group Essentials**  
    - Type: Set  
    - Role: Formats each lead into a structured string containing name, category, price, rating, reviews count, address, phone, and Google Maps URL  
    - Input: Filtered leads  
    - Output: Textual representation for aggregation  

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all grouped lead strings into a single aggregated field for newsletter input  
    - Input: Grouped strings  
    - Output: Single aggregated object  

  - **Lead Creator**  
    - Type: Airtable  
    - Role: Creates records in Airtable base/table to store leads with detailed fields  
    - Config: Airtable base ID and table ID, mapped fields such as Phone, Title, Rating, Address, Category, Location (URL), Review Count, Average Price Range  
    - Credentials: Airtable Personal Access Token  
    - Input: Sorted essentials (not filtered by review count) to create leads  
    - Edge Cases: Airtable API rate limits, invalid data types, connectivity issues  

  - **Sticky Notes:**  
    - Describe Airtable entry process and aggregation intent for newsletter  

#### 2.4 AI Newsletter Generation and Emailing

- **Overview:**  
  Feeds aggregated restaurant lead data into an OpenAI based AI agent to generate a newsletter, then sends the newsletter via Gmail.

- **Nodes Involved:**  
  - AI Agent1  
  - OpenAI Chat Model  
  - Structured Output Parser1  
  - Send a message (Gmail)  
  - Sticky Note5 (Prepare and Send Email)

- **Node Details:**

  - **AI Agent1**  
    - Type: Langchain AI Agent  
    - Role: Orchestrates AI prompt and response parsing using OpenAI GPT-4.1-mini model  
    - Config: Uses structured output parser for response interpretation  
    - Input: Aggregated restaurant data summary  
    - Output: AI-generated newsletter with subject and body fields  

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4.1-mini model for AI Agent  
    - Credentials: OpenAI API key  
    - Input: Prompt from AI Agent  
    - Output: Chat completions  

  - **Structured Output Parser1**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI text output into structured JSON for email content  

  - **Send a message (Gmail)**  
    - Type: Gmail node  
    - Role: Sends email with the newsletter content  
    - Config: Uses Gmail OAuth2 credentials; dynamically sets recipient, subject, and message body from AI output  
    - Edge Cases: Authentication failures, email sending limits, invalid recipient address  

  - **Sticky Note:**  
    - Emphasizes newsletter creation and email sending workflow  

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                               | Input Node(s)           | Output Node(s)         | Sticky Note                                            |
|-------------------------|-----------------------------------|-----------------------------------------------|------------------------|------------------------|--------------------------------------------------------|
| On form submission      | Form Trigger                      | Collects user input to start scraping          | —                      | Run an Actor           | ## Run an APIfy Google Map Actor                        |
| Run an Actor            | Apify Actor Execution             | Runs Google Maps scraper actor                  | On form submission     | If                     | ## Run an APIfy Google Map Actor                        |
| If                      | Conditional                      | Checks actor run status                         | Run an Actor           | Get dataset items, No Operation, do nothing | ## Run an APIfy Google Map Actor                        |
| No Operation, do nothing | NoOp                            | Stops flow if scraping failed                   | If                     | —                      | ## Run an APIfy Google Map Actor                        |
| Get dataset items       | Apify Dataset Retrieval          | Fetches scraped data from Apify                 | If                     | Extract Essentials      | ## Extract data from Dataset                            |
| Extract Essentials      | Set                             | Extracts and normalizes key restaurant fields  | Get dataset items       | Sort by Review Count and Rating | ## Extract only Essentials                              |
| Sort by Review Count and Rating | Sort                       | Sorts restaurants by reviews and rating        | Extract Essentials      | Reviews > 500, Lead Creator | ## Extract only Essentials                              |
| Reviews > 500           | Conditional                     | Filters restaurants with more than 500 reviews | Sort by Review Count and Rating | Group Essentials, No Operation, do nothing | ## Aggregate for the Newletter                          |
| Group Essentials        | Set                             | Formats lead data into structured strings      | Reviews > 500           | Aggregate               | ## Aggregate for the Newletter                          |
| Aggregate               | Aggregate                       | Aggregates grouped strings for AI input        | Group Essentials        | AI Agent1              | ## Aggregate for the Newletter                          |
| Lead Creator            | Airtable                        | Creates lead records in Airtable                | Sort by Review Count and Rating | —                      | ## Make an entry to Airtable                            |
| AI Agent1               | Langchain AI Agent              | Generates newsletter from aggregated data      | Aggregate               | Send a message          | ## Prepare and Send Email                              |
| OpenAI Chat Model       | Langchain OpenAI Chat Model    | Provides GPT-4.1-mini model for AI Agent       | AI Agent1               | AI Agent1 (ai_languageModel) |                                                        |
| Structured Output Parser1 | Langchain Output Parser Structured | Parses AI output to structured JSON           | AI Agent1               | AI Agent1 (ai_outputParser) |                                                        |
| Send a message          | Gmail                          | Sends newsletter email                          | AI Agent1               | —                      | ## Prepare and Send Email                              |
| Sticky Note             | Sticky Note                    | Notes on APIfy actor run                        | —                      | —                      | ## Run an APIfy Google Map Actor                        |
| Sticky Note1            | Sticky Note                    | Notes on dataset extraction                      | —                      | —                      | ## Extract data from Dataset                            |
| Sticky Note2            | Sticky Note                    | Notes on essentials extraction and sorting      | —                      | —                      | ## Extract only Essentials                              |
| Sticky Note3            | Sticky Note                    | Notes on Airtable lead creation                  | —                      | —                      | ## Make an entry to Airtable                            |
| Sticky Note4            | Sticky Note                    | Notes on aggregation for newsletter              | —                      | —                      | ## Aggregate for the Newletter                          |
| Sticky Note5            | Sticky Note                    | Notes on AI newsletter preparation and sending  | —                      | —                      | ## Prepare and Send Email                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “On form submission” node**  
   - Type: Form Trigger  
   - Configure form with:  
     - Dropdown field "Location" with options (e.g., "Bangkok, Thailand")  
     - Text field "What to Scrape" (required)  
     - Number field "Number of Items to be Scraped" (required)  
   - Set button label: "Submit"  

2. **Add “Run an Actor” node (Apify)**  
   - Type: Apify Actor Execution  
   - Credential: Apify API key  
   - Actor ID: `nwua9Gu5YrADL7ZDj` (Google Maps Scraper)  
   - Custom JSON body:  
     - `"locationQuery"`: `{{$json.Location}}`  
     - `"maxCrawledPlacesPerSearch"`: `{{$json['Number of Items to be Scraped']}}`  
     - `"searchStringsArray"`: `["{{$json['What to Scrape']}}"]`  
     - Other scraping options as per default (scrapeContacts, scrapeDirectories, etc.)  
   - Connect output of form submission to this node  

3. **Add “If” node**  
   - Condition: Check if `{{$json.status}} == "SUCCEEDED"`  
   - Connect output of Run an Actor to If node  

4. **Add “No Operation, do nothing” node**  
   - Connect If node’s false output to this node  

5. **Add “Get dataset items” node (Apify)**  
   - Credential: Apify API key  
   - Dataset ID: `{{$json.defaultDatasetId}}` (from Run an Actor output)  
   - Connect If node’s true output to this node  

6. **Add “Extract Essentials” node (Set)**  
   - Assign fields:  
     - title = `{{$json.title}}`  
     - categoryName = `{{$json.categoryName}}`  
     - Average Price = `{{$json.price}}`  
     - rating = `{{$json.totalScore}}` (number)  
     - reviewsCount = `{{$json.reviewsCount}}` (number)  
     - address = concatenation of city, address, street, postalCode  
     - phone = `{{$json.phone || $json.phones[0]}}`  
     - url = `{{$json.url}}`  
   - Connect Get dataset items to this node  

7. **Add “Sort by Review Count and Rating” node (Sort)**  
   - Sort fields:  
     - reviewsCount, descending  
     - rating, descending  
   - Connect Extract Essentials to this node  

8. **Add “Reviews > 500” node (If)**  
   - Condition: `reviewsCount > 500`  
   - Connect Sort node to this node  

9. **Add “Group Essentials” node (Set)**  
   - Single assignment field “Restaurant_Data” with multiline string template:  
     ```
     Name: {{ $json.title }}
     Category: {{ $json.categoryName || '' }}
     Price Range: {{ $json['Average Price'] || '' }}
     Rating: {{ $json.rating }}
     Total Reviews: {{ $json.reviewsCount }}
     Address: {{ $json.address.toString() }}
     Phone: {{ $json.phone.toString() }}
     Google Map Location: {{ $json.url.toString() }}
     ```  
   - Connect true output of Reviews > 500 to this node  

10. **Add “Aggregate” node (Aggregate)**  
    - Aggregate field: Restaurant_Data  
    - Connect Group Essentials to Aggregate node  

11. **Add “Lead Creator” node (Airtable)**  
    - Credential: Airtable Personal Access Token  
    - Base ID: `appgqYxjdpsWWuk3p`  
    - Table ID: `tblyPsVMcVtWtBXNp`  
    - Map columns: Phone, Title, Rating, Address, Category, Location (URL), Review Count, Average Price Range  
    - Connect false output of Reviews > 500 to Lead Creator node (so all leads inserted, regardless of review filter)  
    - Connect Sort node output also to Lead Creator (to create leads)  

12. **Add “AI Agent1” node (Langchain Agent)**  
    - Configure with OpenAI Chat Model (next step)  
    - Enable output parser (Structured Output Parser1)  
    - Connect Aggregate node to AI Agent1  

13. **Add “OpenAI Chat Model” node**  
    - Model: GPT-4.1-mini  
    - Credentials: OpenAI API key  
    - Connect AI Agent1 node as language model to this node  

14. **Add “Structured Output Parser1” node**  
    - Connect AI Agent1 output parser to this node  

15. **Add “Send a message” node (Gmail)**  
    - Credential: Gmail OAuth2  
    - To: fixed or dynamic email (e.g., "prath002@gmail.com")  
    - Subject: `{{$json.output.Subject}}` (from AI output)  
    - Message: `{{$json.output.Body}}` (from AI output)  
    - Connect AI Agent1 main output to this node  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                              |
|--------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| Use the Apify Google Maps Scraper actor (ID: nwua9Gu5YrADL7ZDj) from the Apify store for scraping | https://console.apify.com/actors/nwua9Gu5YrADL7ZDj                         |
| Airtable base and table URLs for reference to setup columns and mappings                         | https://airtable.com/appgqYxjdpsWWuk3p                                      |
| Gmail OAuth2 credentials are required for sending emails                                         | Google Developer Console setup                                              |
| OpenAI GPT-4.1-mini model used for AI newsletter generation                                     | https://platform.openai.com/docs/models/gpt-4                                |
| The workflow assumes English language for scraping and AI processing                             | Configured in actor and AI nodes                                            |
| Potential API rate limits on Apify, Airtable, OpenAI, and Gmail should be accounted for          | Design retries and error handling as needed                                 |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.