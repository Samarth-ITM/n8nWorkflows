Plan AI-powered travel itineraries with Apify, OpenAI, and Google Docs

https://n8nworkflows.xyz/workflows/plan-ai-powered-travel-itineraries-with-apify--openai--and-google-docs-13921


# Plan AI-powered travel itineraries with Apify, OpenAI, and Google Docs

# 1. Workflow Overview

This workflow collects trip details from a web form, gathers hotel and flight options via Apify scrapers, generates destination recommendations with OpenAI, then compiles everything into a Google Doc and returns the document link to the user.

Typical use cases:
- Travel agencies producing quick client-ready trip briefs
- Personal travel planning assistants
- Concierge-style lead capture forms that deliver a polished itinerary document automatically

## 1.1 Input Reception and Normalization
The workflow begins with an n8n form where the user enters departure and destination airports, dates, destination city, and traveler count. A Set node then normalizes these values into reusable variables.

## 1.2 Parallel Travel Data Collection and AI Enrichment
After normalization, three branches run in parallel:
- Booking.com hotel scraping via Apify
- Google Flights scraping via Apify
- OpenAI-powered destination recommendations via a LangChain LLM chain

## 1.3 Data Consolidation and Structuring
The three branches are merged, then a Code node transforms the raw outputs into a single structured travel-plan object containing trip details, hotel options, flight options, and AI-generated recommendations.

## 1.4 Google Docs Generation and User Response
The workflow creates a new Google Doc, formats the full travel plan into a plain-text report, inserts the content into the document, and shows the final document link on the form completion page.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Normalization

### Overview
This block captures traveler input through an n8n-hosted form and standardizes the values for downstream nodes. It ensures airports are uppercased and all trip metadata is available with concise variable names.

### Nodes Involved
- Travel Form
- Set Variables

### Node Details

#### Travel Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`; workflow entry point and public form trigger.
- **Configuration choices:**
  - Form title: `✈️ Travel Planner`
  - Response mode: `lastNode`, so the form waits for the workflow result and displays the final completion response from the last form node.
  - Form description explains that flights, accommodations, restaurants, and attractions will be found automatically.
  - Fields collected:
    - Departure Airport (IATA Code) — required
    - Destination City — required
    - Destination Airport (IATA Code) — required
    - Check-in Date — required date
    - Check-out Date — required date
    - Number of Travelers — required number
- **Key expressions or variables used:** None in configuration; outputs user-submitted field labels as JSON keys.
- **Input and output connections:**
  - No input node; this is an entry point.
  - Outputs to `Set Variables`.
- **Version-specific requirements:** Type version `2.2`; requires a recent n8n version with Form Trigger support.
- **Edge cases or potential failure types:**
  - Invalid airport code format is not validated beyond required-field presence.
  - Date order is not validated; check-out may be before check-in unless additional validation is added.
  - Traveler count may be zero or negative if front-end constraints are not enforced properly.
  - Public form access depends on active workflow and reachable webhook URL.
- **Sub-workflow reference:** None.

#### Set Variables
- **Type and technical role:** `n8n-nodes-base.set`; maps raw form keys to normalized internal variable names.
- **Configuration choices:**
  - Creates:
    - `departureAirport`
    - `destination`
    - `destinationAirport`
    - `checkinDate`
    - `checkoutDate`
    - `travelers`
  - Uppercases airport codes with `.toUpperCase()`.
- **Key expressions or variables used:**
  - `{{ $json['Departure Airport (IATA Code)'].toUpperCase() }}`
  - `{{ $json['Destination City'] }}`
  - `{{ $json['Destination Airport (IATA Code)'].toUpperCase() }}`
  - `{{ $json['Check-in Date'] }}`
  - `{{ $json['Check-out Date'] }}`
  - `{{ $json['Number of Travelers'] }}`
- **Input and output connections:**
  - Input: `Travel Form`
  - Outputs in parallel to:
    - `Booking.com Scraper`
    - `Google Flights Scraper`
    - `AI Recommendations`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - `.toUpperCase()` fails if the airport fields are undefined or null.
  - Destination city is not trimmed or normalized.
  - No date formatting conversion is performed; downstream services must accept the raw form date format.
- **Sub-workflow reference:** None.

---

## 2.2 Parallel Hotel Data Fetching

### Overview
This branch uses an Apify actor to scrape Booking.com hotel results for the requested destination and dates. A following HTTP Request fetches the actor’s dataset items.

### Nodes Involved
- Booking.com Scraper
- HTTP Request Hotels

### Node Details

#### Booking.com Scraper
- **Type and technical role:** `@apify/n8n-nodes-apify.apify`; launches an Apify actor run against Booking.com search data.
- **Configuration choices:**
  - Actor source: Apify Store
  - Actor: `voyager/booking-scraper`
  - Search target uses destination city, date range, and traveler count
  - Hardcoded search settings:
    - `rooms: 1`
    - `children: 0`
    - `currency: USD`
    - `language: en-us`
    - `maxItems: 5`
    - `sortBy: bayesian_review_score`
- **Key expressions or variables used:**
  - `{{ $json.destination }}`
  - `{{ $json.checkinDate }}`
  - `{{ $json.checkoutDate }}`
  - `{{ $json.travelers }}`
- **Input and output connections:**
  - Input: `Set Variables`
  - Output: `HTTP Request Hotels`
- **Version-specific requirements:** Type version `1`; requires Apify community/integration node installed and configured.
- **Edge cases or potential failure types:**
  - Invalid or ambiguous destination string may return poor results or none.
  - Apify actor auth failure if Apify credential is invalid.
  - Actor run timeouts or quota exhaustion.
  - Dataset may not contain expected fields if actor output schema changes.
- **Sub-workflow reference:** None.

#### HTTP Request Hotels
- **Type and technical role:** `n8n-nodes-base.httpRequest`; retrieves scraped hotel records from the Apify dataset API.
- **Configuration choices:**
  - URL: `https://api.apify.com/v2/datasets/{{ $json.defaultDatasetId }}/items`
  - Authentication: Generic Credential Type using `httpQueryAuth`
  - Expects Apify token as query parameter auth
- **Key expressions or variables used:**
  - `{{ $json.defaultDatasetId }}`
- **Input and output connections:**
  - Input: `Booking.com Scraper`
  - Output: `Merge All Data` on input 0
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - `defaultDatasetId` missing if actor run failed or output format changed.
  - HTTP 401/403 if token query auth is misconfigured.
  - Empty dataset if scraping produced no results.
  - Large datasets could impact execution time, though this workflow requests only 5 items.
- **Sub-workflow reference:** None.

---

## 2.3 Parallel Flight Data Fetching

### Overview
This branch uses an Apify Google Flights scraper actor to search round-trip flights using the submitted airport codes and dates. A follow-up HTTP Request fetches the resulting JSON from a URL returned by the actor.

### Nodes Involved
- Google Flights Scraper
- HTTP Request Flights

### Node Details

#### Google Flights Scraper
- **Type and technical role:** `@apify/n8n-nodes-apify.apify`; launches an Apify actor to scrape Google Flights results.
- **Configuration choices:**
  - Actor source: Apify Store
  - Actor: `johnvc/Google-Flights-Data-Scraper-Flight-and-Price-Search`
  - Inputs:
    - departure airport IATA
    - arrival airport IATA
    - outbound date
    - return date
    - adult traveler count
    - `travel_class: 1`
    - `currency: USD`
- **Key expressions or variables used:**
  - `{{ $json.departureAirport }}`
  - `{{ $json.destinationAirport }}`
  - `{{ $json.checkinDate }}`
  - `{{ $json.checkoutDate }}`
  - `{{ $json.travelers }}`
- **Input and output connections:**
  - Input: `Set Variables`
  - Output: `HTTP Request Flights`
- **Version-specific requirements:** Type version `1`; requires Apify integration and valid Apify credentials.
- **Edge cases or potential failure types:**
  - Invalid IATA codes can produce no results or actor errors.
  - Flight data availability depends on Google Flights scraping stability.
  - Actor may return schema variations over time.
  - Auth, usage limits, and timeout issues on Apify.
- **Sub-workflow reference:** None.

#### HTTP Request Flights
- **Type and technical role:** `n8n-nodes-base.httpRequest`; fetches the flight results from a URL returned by the Apify actor.
- **Configuration choices:**
  - URL: `{{ $json.output.flightResults }}`
  - Authentication: Generic Credential Type with `httpQueryAuth`
- **Key expressions or variables used:**
  - `{{ $json.output.flightResults }}`
- **Input and output connections:**
  - Input: `Google Flights Scraper`
  - Output: `Merge All Data` on input 1
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - `output.flightResults` missing if actor output changes or run fails.
  - Returned URL may expire or be inaccessible.
  - Auth credential may be unnecessary or may conflict depending on actual endpoint behavior, but here it is configured the same as hotel retrieval.
  - Empty or malformed JSON can break downstream parsing assumptions.
- **Sub-workflow reference:** None.

---

## 2.4 Parallel AI Recommendation Generation

### Overview
This branch prompts an OpenAI chat model to generate destination-specific restaurants, attractions, local advice, and a suggested itinerary. It enriches the factual scraped data with narrative guidance.

### Nodes Involved
- AI Recommendations
- OpenAI Recommendations Model

### Node Details

#### AI Recommendations
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; LangChain LLM chain node that sends a structured prompt to a connected chat model.
- **Configuration choices:**
  - Prompt type: `define`
  - Long-form custom prompt instructs the model to act as a travel expert.
  - Requests:
    - Top 10 restaurants
    - Top 10 monuments and historical sites
    - Top 10 places to visit
    - Local tips
    - Suggested itinerary based on dates
  - Response is requested in organized sections with bullet points.
- **Key expressions or variables used:**
  - `{{ $('Set Variables').item.json.destination }}`
  - `{{ $('Set Variables').item.json.checkinDate }}`
  - `{{ $('Set Variables').item.json.checkoutDate }}`
  - `{{ $('Set Variables').item.json.travelers }}`
- **Input and output connections:**
  - Main input: `Set Variables`
  - AI language model input: `OpenAI Recommendations Model`
  - Main output: `Merge All Data` on input 2
- **Version-specific requirements:** Type version `1.4`; requires LangChain-capable n8n version and compatible model node connection.
- **Edge cases or potential failure types:**
  - Model may hallucinate outdated or inaccurate venue information.
  - Token usage may be high because the prompt requests extensive content.
  - Rate limits or quota exhaustion on OpenAI.
  - Output structure may vary, affecting downstream formatting quality though not hard parsing.
- **Sub-workflow reference:** None.

#### OpenAI Recommendations Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; provides the actual OpenAI chat model for the chain.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - No additional tools enabled
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Output via AI language model connection to `AI Recommendations`
- **Version-specific requirements:** Type version `1.3`; requires valid OpenAI credentials.
- **Edge cases or potential failure types:**
  - Invalid API key or revoked credential
  - Model availability differences by account
  - Rate limits and token quota failures
- **Sub-workflow reference:** None.

---

## 2.5 Data Consolidation and Structuring

### Overview
This block merges the three parallel outputs and converts heterogeneous raw data into one normalized JSON object. It extracts top hotel and flight options and preserves the AI text as a single recommendation body.

### Nodes Involved
- Merge All Data
- Process All Data

### Node Details

#### Merge All Data
- **Type and technical role:** `n8n-nodes-base.merge`; gathers the outputs from three branches before processing.
- **Configuration choices:**
  - `numberInputs: 3`
  - Receives hotels, flights, and AI recommendations
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input 0: `HTTP Request Hotels`
  - Input 1: `HTTP Request Flights`
  - Input 2: `AI Recommendations`
  - Output: `Process All Data`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If one branch errors and no error handling exists, merge may never complete.
  - Different item counts across branches can produce merge behavior that may surprise users depending on execution semantics.
- **Sub-workflow reference:** None.

#### Process All Data
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript normalization and aggregation layer.
- **Configuration choices:**
  - Pulls source data directly from named nodes using expressions like `$('HTTP Request Hotels').all()`.
  - Builds:
    - `tripDetails`
    - `hotels`
    - `flights`
    - `recommendations`
    - `generatedAt`
  - Hotel handling:
    - Supports either an array response or individual objects
    - Extracts up to 5 hotels
    - Sorts by rating if more than 5 are present
  - Flight handling:
    - Supports array or single-object responses
    - Reads `best_flights` and up to 2 from `other_flights`
    - Extracts airline, number, price, duration, stops, airport/times, aircraft
    - Limits final list to 5
  - AI handling:
    - Reads `text` or `output`
- **Key expressions or variables used:**
  - `$('HTTP Request Hotels').all()`
  - `$('HTTP Request Flights').all()`
  - `$('AI Recommendations').all()`
  - `$('Set Variables').first().json`
- **Input and output connections:**
  - Input: `Merge All Data`
  - Output: `Create Document`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - The initial filtered variables `hotelsInput`, `flightsInput`, `aiInput` are computed but not used; harmless but unnecessary.
  - Direct object equality comparisons in those filters are fragile if reused later.
  - Schema drift in Apify outputs can leave hotels or flights empty.
  - `price` formatting may create double dollar signs later because some values are already prefixed.
  - Duration is labeled in minutes in the document even if source duration may already be a formatted string.
  - No deduplication of hotel or flight results.
- **Sub-workflow reference:** None.

---

## 2.6 Google Doc Creation and Final User Response

### Overview
This block creates a document, assembles the final plain-text travel report, inserts it into Google Docs, then returns the document URL to the user through the form completion page.

### Nodes Involved
- Create Document
- Prepare Document Content
- Update Document
- Prepare Form Ending
- Form Ending

### Node Details

#### Create Document
- **Type and technical role:** `n8n-nodes-base.googleDocs`; creates a new Google Doc file.
- **Configuration choices:**
  - Title pattern:
    - `Travel Plan - {departureAirport} to {destinationAirport} for {travelers} people`
  - Folder ID is currently placeholder text: `YOUR_FOLDER_ID`
- **Key expressions or variables used:**
  - `{{ $json.tripDetails.departureAirport }}`
  - `{{ $json.tripDetails.destinationAirport }}`
  - `{{ $json.tripDetails.travelers }}`
- **Input and output connections:**
  - Input: `Process All Data`
  - Output: `Prepare Document Content`
- **Version-specific requirements:** Type version `2`; requires Google Docs OAuth2 credential with Docs/Drive access.
- **Edge cases or potential failure types:**
  - This node will fail until `folderId` is replaced with a real Google Drive folder ID.
  - OAuth scopes may be insufficient for file creation.
  - Destination folder permissions may block document creation.
- **Sub-workflow reference:** None.

#### Prepare Document Content
- **Type and technical role:** `n8n-nodes-base.code`; composes the final travel plan text and derives the document URL.
- **Configuration choices:**
  - Retrieves document metadata from `Create Document`
  - Retrieves normalized travel plan object from `Process All Data`
  - Determines document ID using fallback keys:
    - `id`
    - `documentId`
    - `docId`
  - Builds a report with sections:
    - Header and generation time
    - Trip details
    - Flights
    - Hotels
    - AI recommendations
- **Key expressions or variables used:**
  - `$('Create Document').first().json`
  - `$('Process All Data').first().json`
- **Input and output connections:**
  - Input: `Create Document`
  - Output: `Update Document`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If Google Docs output field names differ, document ID extraction may fail.
  - Flight price line uses `Price: $${f.price}` while upstream may already store values like `$123`; this can render as `$$123`.
  - The section says “top recommended hotels” but the completion page later says “Top 3 recommended hotels,” whereas processing actually aims for top 5.
  - Large AI content may make the inserted document very long.
- **Sub-workflow reference:** None.

#### Update Document
- **Type and technical role:** `n8n-nodes-base.googleDocs`; inserts the generated content into the created Google Doc.
- **Configuration choices:**
  - Operation: `update`
  - Action: `insert`
  - Insert text: `{{ $json.content }}`
  - Document identifier field is configured in `documentURL`, but it is passed the document ID rather than a full URL.
- **Key expressions or variables used:**
  - `{{ $json.content }}`
  - `{{ $json.documentId }}`
- **Input and output connections:**
  - Input: `Prepare Document Content`
  - Output: `Prepare Form Ending`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Parameter name `documentURL` receiving an ID may still work only if the node accepts IDs in that field; otherwise it may fail.
  - Insert action may append at default position rather than replacing any existing placeholder content.
  - OAuth permission issues can also affect update.
- **Sub-workflow reference:** None.

#### Prepare Form Ending
- **Type and technical role:** `n8n-nodes-base.code`; extracts the document URL for the final form response.
- **Configuration choices:**
  - Reads `documentUrl` from `Prepare Document Content`
  - Returns only that field
- **Key expressions or variables used:**
  - `$('Prepare Document Content').first().json`
- **Input and output connections:**
  - Input: `Update Document`
  - Output: `Form Ending`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If document URL construction failed earlier, the completion page will display an invalid or blank link.
- **Sub-workflow reference:** None.

#### Form Ending
- **Type and technical role:** `n8n-nodes-base.form`; renders the completion page for the original form submission.
- **Configuration choices:**
  - Operation: `completion`
  - Completion title: `✅ Your travel plan is ready!`
  - Completion message includes:
    - document URL
    - summary of included content
- **Key expressions or variables used:**
  - `{{ $json.documentUrl }}`
- **Input and output connections:**
  - Input: `Prepare Form Ending`
  - No downstream output
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases or potential failure types:**
  - If the workflow errors before this point, users will not see the friendly completion page.
  - The bullet list claims “Top 3 recommended hotels,” but the workflow processes up to 5 hotels.
- **Sub-workflow reference:** None.

---

## 2.7 Documentation Sticky Notes

### Overview
These nodes are purely visual annotations inside the canvas. They do not participate in execution but contain setup guidance, links, and block descriptions that are important for human operators.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation.
- **Configuration choices:** Contains a full workflow overview, setup instructions, requirements, and help links.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation.
- **Configuration choices:** Documents the trigger and input normalization block.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation.
- **Configuration choices:** Documents the parallel data fetching block and includes Apify integration docs link.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation.
- **Configuration choices:** Explains merge and processing behavior.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; block annotation.
- **Configuration choices:** Describes Google Docs generation and includes Google Docs node documentation link.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Travel Form | n8n-nodes-base.formTrigger | Public form entry point collecting trip details |  | Set Variables | ## 1. Trigger and input<br>The form collects departure/destination airports, dates, and traveler count. The Set node normalizes these into variables used by all downstream nodes. |
| Set Variables | n8n-nodes-base.set | Normalizes form data into internal variables | Travel Form | Booking.com Scraper, Google Flights Scraper, AI Recommendations | ## 1. Trigger and input<br>The form collects departure/destination airports, dates, and traveler count. The Set node normalizes these into variables used by all downstream nodes. |
| Booking.com Scraper | @apify/n8n-nodes-apify.apify | Launches Apify Booking.com scraping actor | Set Variables | HTTP Request Hotels | ## 2. Parallel data fetching<br>[Apify integration docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-apify/)<br>Three tasks run in parallel: Booking.com scrapes top 5 hotels by review score, Google Flights scrapes the best flights, and OpenAI generates restaurant, attraction, and itinerary recommendations. |
| HTTP Request Hotels | n8n-nodes-base.httpRequest | Fetches hotel dataset items from Apify | Booking.com Scraper | Merge All Data | ## 2. Parallel data fetching<br>[Apify integration docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-apify/)<br>Three tasks run in parallel: Booking.com scrapes top 5 hotels by review score, Google Flights scrapes the best flights, and OpenAI generates restaurant, attraction, and itinerary recommendations. |
| Google Flights Scraper | @apify/n8n-nodes-apify.apify | Launches Apify Google Flights scraping actor | Set Variables | HTTP Request Flights | ## 2. Parallel data fetching<br>[Apify integration docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-apify/)<br>Three tasks run in parallel: Booking.com scrapes top 5 hotels by review score, Google Flights scrapes the best flights, and OpenAI generates restaurant, attraction, and itinerary recommendations. |
| HTTP Request Flights | n8n-nodes-base.httpRequest | Fetches flight result payload from returned URL | Google Flights Scraper | Merge All Data | ## 2. Parallel data fetching<br>[Apify integration docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-apify/)<br>Three tasks run in parallel: Booking.com scrapes top 5 hotels by review score, Google Flights scrapes the best flights, and OpenAI generates restaurant, attraction, and itinerary recommendations. |
| AI Recommendations | @n8n/n8n-nodes-langchain.chainLlm | Generates destination recommendations and itinerary text | Set Variables; OpenAI Recommendations Model (AI model input) | Merge All Data | ## 2. Parallel data fetching<br>[Apify integration docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-apify/)<br>Three tasks run in parallel: Booking.com scrapes top 5 hotels by review score, Google Flights scrapes the best flights, and OpenAI generates restaurant, attraction, and itinerary recommendations. |
| OpenAI Recommendations Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies OpenAI chat model to the LLM chain |  | AI Recommendations | ## 2. Parallel data fetching<br>[Apify integration docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-apify/)<br>Three tasks run in parallel: Booking.com scrapes top 5 hotels by review score, Google Flights scrapes the best flights, and OpenAI generates restaurant, attraction, and itinerary recommendations. |
| Merge All Data | n8n-nodes-base.merge | Combines hotels, flights, and AI outputs | HTTP Request Hotels, HTTP Request Flights, AI Recommendations | Process All Data | ## 3. Merge and process<br>All three data streams are merged into a single item. The Code node structures hotels, flights, and AI recommendations into a clean object for the Google Doc. |
| Process All Data | n8n-nodes-base.code | Normalizes and aggregates all travel data | Merge All Data | Create Document | ## 3. Merge and process<br>All three data streams are merged into a single item. The Code node structures hotels, flights, and AI recommendations into a clean object for the Google Doc. |
| Create Document | n8n-nodes-base.googleDocs | Creates a new Google Doc for the travel plan | Process All Data | Prepare Document Content | ## 4. Generate Google Doc and respond<br>[Google Docs node docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledocs/)<br>A new Google Doc is created, populated with the formatted travel plan, and the form completion page returns the document link to the user. |
| Prepare Document Content | n8n-nodes-base.code | Builds formatted report text and document URL | Create Document | Update Document | ## 4. Generate Google Doc and respond<br>[Google Docs node docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledocs/)<br>A new Google Doc is created, populated with the formatted travel plan, and the form completion page returns the document link to the user. |
| Update Document | n8n-nodes-base.googleDocs | Inserts the generated content into the document | Prepare Document Content | Prepare Form Ending | ## 4. Generate Google Doc and respond<br>[Google Docs node docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledocs/)<br>A new Google Doc is created, populated with the formatted travel plan, and the form completion page returns the document link to the user. |
| Prepare Form Ending | n8n-nodes-base.code | Passes document URL to final completion screen | Update Document | Form Ending | ## 4. Generate Google Doc and respond<br>[Google Docs node docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledocs/)<br>A new Google Doc is created, populated with the formatted travel plan, and the form completion page returns the document link to the user. |
| Form Ending | n8n-nodes-base.form | Displays form completion page with document link | Prepare Form Ending |  | ## 4. Generate Google Doc and respond<br>[Google Docs node docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledocs/)<br>A new Google Doc is created, populated with the formatted travel plan, and the form completion page returns the document link to the user. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation and setup guidance |  |  | ## Try It Out!<br>### Collect travel details via a form, then scrape flights and hotels while AI generates recommendations — all compiled into a Google Doc.<br>Great for travel agencies, personal trip planning, or any service that needs to deliver a complete travel brief automatically.<br>### How it works<br>* User submits a travel form with airports, dates, and number of travelers.<br>* Three parallel tasks run: Booking.com hotel scraping, Google Flights scraping, and OpenAI recommendation generation.<br>* Results are merged, formatted, and written to a new Google Doc.<br>* The user receives a link to their personalized travel plan.<br>### Setup steps<br>* Create an [Apify](https://apify.com) account and add your API token as both an "Apify API" credential and an "HTTP Query Auth" credential (parameter name: `token`).<br>* Add your OpenAI API key as an "OpenAI" credential.<br>* Connect Google Docs via OAuth2 and update the `folderId` in the "Create Document" node.<br>* Activate the workflow and share the form URL.<br>### Requirements<br>* Apify account (for Booking.com and Google Flights scrapers)<br>* OpenAI API key<br>* Google account with Docs and Drive access<br>### Need Help?<br>Join the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/)!<br>Happy Hacking! |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for input block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for parallel fetching block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for merge/process block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note for Google Docs/output block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it something like: `Plan travel itineraries with AI using Apify, OpenAI, and Google Docs`.

2. **Add a Form Trigger node** named `Travel Form`.
   - Type: `Form Trigger`
   - Set:
     - **Form Title:** `✈️ Travel Planner`
     - **Form Description:** `Plan your perfect trip! Enter your travel details below and we'll find the best flights, accommodations, restaurants, and attractions for you.`
     - **Response Mode:** `Last Node`
   - Add these required fields:
     1. `Departure Airport (IATA Code)` — text, placeholder `e.g., JFK, LAX, LHR`
     2. `Destination City` — text, placeholder `e.g., Paris, Tokyo, Barcelona`
     3. `Destination Airport (IATA Code)` — text, placeholder `e.g., CDG, NRT, BCN`
     4. `Check-in Date` — date
     5. `Check-out Date` — date
     6. `Number of Travelers` — number, placeholder `1`

3. **Add a Set node** named `Set Variables`.
   - Connect `Travel Form -> Set Variables`.
   - Create these assignments:
     - `departureAirport` as string: `{{ $json['Departure Airport (IATA Code)'].toUpperCase() }}`
     - `destination` as string: `{{ $json['Destination City'] }}`
     - `destinationAirport` as string: `{{ $json['Destination Airport (IATA Code)'].toUpperCase() }}`
     - `checkinDate` as string: `{{ $json['Check-in Date'] }}`
     - `checkoutDate` as string: `{{ $json['Check-out Date'] }}`
     - `travelers` as number: `{{ $json['Number of Travelers'] }}`

4. **Create Apify credentials** before building the scraper branches.
   - Add an `Apify API` credential with your Apify API token.
   - Add an `HTTP Query Auth` credential with:
     - Query parameter name: `token`
     - Value: your Apify API token
   - The workflow uses both credential types.

5. **Add an Apify node** named `Booking.com Scraper`.
   - Connect `Set Variables -> Booking.com Scraper`.
   - Select actor from Apify Store:
     - `voyager/booking-scraper`
   - Use a custom input body similar to:
     - search = destination city
     - checkIn = check-in date
     - checkOut = check-out date
     - rooms = 1
     - adults = traveler count
     - children = 0
     - currency = USD
     - language = en-us
     - maxItems = 5
     - sortBy = bayesian_review_score
   - Use expressions:
     - `{{ $json.destination }}`
     - `{{ $json.checkinDate }}`
     - `{{ $json.checkoutDate }}`
     - `{{ $json.travelers }}`
   - Attach your `Apify API` credential.

6. **Add an HTTP Request node** named `HTTP Request Hotels`.
   - Connect `Booking.com Scraper -> HTTP Request Hotels`.
   - Configure:
     - Method: `GET`
     - URL: `https://api.apify.com/v2/datasets/{{ $json.defaultDatasetId }}/items`
     - Authentication: `Generic Credential Type`
     - Generic Auth Type: `HTTP Query Auth`
   - Attach the Apify query-auth credential.

7. **Add another Apify node** named `Google Flights Scraper`.
   - Connect `Set Variables -> Google Flights Scraper`.
   - Select actor from Apify Store:
     - `johnvc/Google-Flights-Data-Scraper-Flight-and-Price-Search`
   - Set custom input body:
     - `departure_id` = departure airport
     - `arrival_id` = destination airport
     - `outbound_date` = check-in date
     - `return_date` = check-out date
     - `adults` = traveler count
     - `travel_class` = 1
     - `currency` = USD
   - Use expressions:
     - `{{ $json.departureAirport }}`
     - `{{ $json.destinationAirport }}`
     - `{{ $json.checkinDate }}`
     - `{{ $json.checkoutDate }}`
     - `{{ $json.travelers }}`
   - Attach the same `Apify API` credential.

8. **Add an HTTP Request node** named `HTTP Request Flights`.
   - Connect `Google Flights Scraper -> HTTP Request Flights`.
   - Configure:
     - Method: `GET`
     - URL: `{{ $json.output.flightResults }}`
     - Authentication: `Generic Credential Type`
     - Generic Auth Type: `HTTP Query Auth`
   - Attach the Apify query-auth credential.

9. **Create OpenAI credentials**.
   - Add an `OpenAI` credential with a valid API key.

10. **Add an OpenAI Chat Model node** named `OpenAI Recommendations Model`.
    - Type: `OpenAI Chat Model`
    - Set model to `gpt-4o-mini`.
    - Attach your OpenAI credential.

11. **Add an AI Chain node** named `AI Recommendations`.
    - Connect `Set Variables -> AI Recommendations` as a normal main connection.
    - Connect `OpenAI Recommendations Model -> AI Recommendations` as the AI language model connection.
    - Set prompt type to `Define`.
    - Use a structured travel-expert prompt requesting:
      - Top 10 restaurants
      - Top 10 monuments & historical sites
      - Top 10 places to visit
      - Local tips
      - Suggested itinerary
    - Reference values from `Set Variables`:
      - destination
      - travel dates
      - traveler count

12. **Add a Merge node** named `Merge All Data`.
    - Set **Number of Inputs** to `3`.
    - Connect:
      - `HTTP Request Hotels -> Merge All Data` to input 0
      - `HTTP Request Flights -> Merge All Data` to input 1
      - `AI Recommendations -> Merge All Data` to input 2

13. **Add a Code node** named `Process All Data`.
    - Connect `Merge All Data -> Process All Data`.
    - Paste logic that:
      - Reads hotel items from `HTTP Request Hotels`
      - Reads flight items from `HTTP Request Flights`
      - Reads AI text from `AI Recommendations`
      - Reads trip details from `Set Variables`
      - Builds one output object with:
        - `tripDetails`
        - `hotels`
        - `flights`
        - `recommendations`
        - `generatedAt`
    - Required output shape:
      - `tripDetails.destination`
      - `tripDetails.departureAirport`
      - `tripDetails.destinationAirport`
      - `tripDetails.checkIn`
      - `tripDetails.checkOut`
      - `tripDetails.travelers`
      - `hotels` array
      - `flights` array
      - `recommendations` string
      - `generatedAt` ISO timestamp

14. **Create Google Docs credentials**.
    - Add a `Google Docs OAuth2` credential.
    - Ensure the Google account has access to both Google Docs and the target Drive folder.

15. **Add a Google Docs node** named `Create Document`.
    - Connect `Process All Data -> Create Document`.
    - Operation: create document
    - Title expression:
      - `Travel Plan - {{ $json.tripDetails.departureAirport }} to {{ $json.tripDetails.destinationAirport }} for {{ $json.tripDetails.travelers }} people`
    - Set the destination `folderId` to a real Google Drive folder ID.
   - Attach the Google Docs OAuth2 credential.

16. **Add a Code node** named `Prepare Document Content`.
    - Connect `Create Document -> Prepare Document Content`.
    - Build a string report containing:
      - Header with destination and generation timestamp
      - Trip details
      - Flights section
      - Hotels section
      - AI recommendations section
    - Also derive:
      - `documentId`
      - `documentUrl` as `https://docs.google.com/document/d/{documentId}/edit`
      - `content`
    - Make sure the node outputs all three fields.

17. **Add a Google Docs node** named `Update Document`.
    - Connect `Prepare Document Content -> Update Document`.
    - Operation: `Update`
    - Add an action:
      - `Insert`
      - Text: `{{ $json.content }}`
    - For the document reference, provide the created document ID or URL depending on your n8n version’s accepted field behavior.
    - Attach the same Google Docs credential.

18. **Add a Code node** named `Prepare Form Ending`.
    - Connect `Update Document -> Prepare Form Ending`.
    - Return:
      - `documentUrl` from `Prepare Document Content`

19. **Add a Form node** named `Form Ending`.
    - Connect `Prepare Form Ending -> Form Ending`.
    - Set operation to `Completion`.
    - Completion title:
      - `✅ Your travel plan is ready!`
    - Completion message should include:
      - the document URL
      - a summary of included content

20. **Check all connection paths carefully**.
    - Required graph:
      - `Travel Form -> Set Variables`
      - `Set Variables -> Booking.com Scraper -> HTTP Request Hotels -> Merge All Data`
      - `Set Variables -> Google Flights Scraper -> HTTP Request Flights -> Merge All Data`
      - `Set Variables -> AI Recommendations -> Merge All Data`
      - `OpenAI Recommendations Model -> AI Recommendations` via AI model connection
      - `Merge All Data -> Process All Data -> Create Document -> Prepare Document Content -> Update Document -> Prepare Form Ending -> Form Ending`

21. **Replace placeholders before testing**.
    - Most important: replace `YOUR_FOLDER_ID` in `Create Document`.
    - Confirm Apify credentials use the same valid token.
    - Confirm the OpenAI model is available to your account.

22. **Test with realistic travel data**.
    - Example:
      - Departure airport: `JFK`
      - Destination city: `Paris`
      - Destination airport: `CDG`
      - Valid future dates
      - Travelers: `2`

23. **Verify output quality**.
    - Check whether:
      - Hotels are populated
      - Flights are populated
      - AI recommendations appear in full
      - Google Doc is created in the intended folder
      - Form completion page displays a working document link

24. **Optionally harden the workflow**.
    - Add validation for:
      - Check-out date after check-in date
      - Positive traveler count
      - IATA code length
    - Add error handling branches for Apify or OpenAI failures.
    - Normalize currency/price formatting to avoid duplicated `$` characters.
    - Update the completion message if you want it to say top 5 hotels instead of top 3.

### Sub-workflow setup
This workflow does **not** use any Execute Workflow or other sub-workflow nodes. No separate sub-workflow needs to be created.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create an Apify account and add your API token as both an `Apify API` credential and an `HTTP Query Auth` credential with parameter name `token`. | https://apify.com |
| Add your OpenAI API key as an `OpenAI` credential. | OpenAI account setup |
| Connect Google Docs via OAuth2 and update the `folderId` in the `Create Document` node. | Google Docs / Google Drive setup |
| Activate the workflow and share the form URL. | Deployment note |
| Apify integration documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-apify/ |
| Google Docs node documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledocs/ |
| Community support via Discord | https://discord.com/invite/XPKeKXeB7d |
| Community support via Forum | https://community.n8n.io/ |
| Requirements: Apify account, OpenAI API key, Google account with Docs and Drive access | Environment prerequisites |
| Workflow intent: collect travel details, scrape flights and hotels, generate AI recommendations, and compile everything into a Google Doc | Functional summary |

## Additional implementation observations
- The workflow is currently **inactive**.
- `Create Document` contains a placeholder folder ID and will fail until replaced.
- The completion message mentions **Top 3 hotels**, but processing logic attempts to keep **up to 5** hotels.
- The document formatting logic may show `$$` in some price lines because some values are already prefixed with `$`.
- No explicit retry, fallback, or branch-level error handling is configured.