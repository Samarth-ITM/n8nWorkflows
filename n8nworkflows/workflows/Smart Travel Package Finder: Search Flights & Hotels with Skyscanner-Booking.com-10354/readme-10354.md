Smart Travel Package Finder: Search Flights & Hotels with Skyscanner-Booking.com

https://n8nworkflows.xyz/workflows/smart-travel-package-finder--search-flights---hotels-with-skyscanner-booking-com-10354


# Smart Travel Package Finder: Search Flights & Hotels with Skyscanner-Booking.com

### 1. Workflow Overview

This workflow, **Smart Travel Package Finder - Search Flights & Hotels with Skyscanner-Booking.com**, automates the process of creating personalized travel itineraries by integrating flight and hotel search APIs, combining results, generating optimized travel packages, and delivering them via formatted HTML email. It targets users who want a convenient way to quickly obtain travel package options with price comparisons, rankings, and booking links.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Receives incoming travel requests via webhook and extracts/validates input parameters.
- **1.2 Parallel Flight and Hotel Search:** Queries Skyscanner (flights) and Booking.com (hotels) APIs in parallel using the validated inputs.
- **1.3 Data Merging and Processing:** Merges flight and hotel data streams and runs a code node that combines, normalizes, and ranks travel packages, including fallback mock data.
- **1.4 Email Content Generation:** Formats the ranked itineraries into a rich HTML email with detailed cards and a comparison table.
- **1.5 Email Delivery and Response:** Sends the generated email via Gmail and responds to the webhook caller with success confirmation and summary data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
  This block receives HTTP POST travel requests, parses the JSON body, validates essential fields (destination, departure, dates, email, adults), and assigns default values if missing.

- **Nodes Involved:**  
  - 📥 Travel Request Webhook  
  - 📝 Parse & Validate Inputs

- **Node Details:**

  - **📥 Travel Request Webhook**  
    - Type: Webhook (HTTP POST trigger)  
    - Configuration: Listens on path `/travel-search`, accepts raw JSON body, responds via a response node later.  
    - Key Expressions: None (raw input captured in `$json.body`)  
    - Inputs: External HTTP POST request  
    - Outputs: Passes raw request JSON downstream  
    - Edge Cases: Invalid JSON body, missing required parameters, HTTP timeout or malformed requests.

  - **📝 Parse & Validate Inputs**  
    - Type: Set node (used for extracting and defaulting values)  
    - Configuration: Extracts destination, departure, checkInDate, checkOutDate, notificationEmail (or email fallback), adults from the request body; applies default values if these are missing (e.g., Shanghai for destination, New York for departure, 1 adult).  
    - Key Expressions: Uses expressions like `={{ $json.body.destination || 'Shanghai' }}`  
    - Inputs: From webhook node  
    - Outputs: Normalized parameters for API queries  
    - Edge Cases: Missing or malformed parameters, fallback to defaults may yield unexpected search results.

---

#### 2.2 Parallel Flight and Hotel Search

- **Overview:**  
  Performs concurrent API requests to Skyscanner for flight options and Booking.com for hotel options based on validated inputs.

- **Nodes Involved:**  
  - ✈️ Search Flights (Skyscanner)  
  - 🏨 Search Hotels (Booking.com)  

- **Node Details:**

  - **✈️ Search Flights (Skyscanner)**  
    - Type: HTTP Request  
    - Configuration: Sends GET requests to Skyscanner flight search API endpoint on RapidAPI.  
    - Query Parameters: originSkyId and destinationSkyId from input, fixed entity IDs for origin and destination cities, dates, number of adults, currency USD, market locale `en-US`, countryCode `US`.  
    - Authentication: Generic HTTP Header Auth via RapidAPI key credential.  
    - Timeout: 30 seconds  
    - Continue On Fail: Enabled (to allow workflow to proceed even if flight API fails)  
    - Inputs: Validated parameters  
    - Outputs: JSON flight search results including itineraries  
    - Edge Cases: API rate limits, authentication errors, timeout, incomplete data, empty results.

  - **🏨 Search Hotels (Booking.com)**  
    - Type: HTTP Request  
    - Configuration: Sends GET requests to Booking.com hotel search API endpoint on RapidAPI.  
    - Query Parameters: destination type "city", fixed destination ID (-1746443), check-in/out dates, adult count, ordering by price, currency USD, metric units, 1 room, page 0.  
    - Authentication: Generic HTTP Header Auth via RapidAPI key credential.  
    - Timeout: 30 seconds  
    - Continue On Fail: Enabled  
    - Inputs: Validated parameters  
    - Outputs: JSON hotel search results  
    - Edge Cases: API failures, auth errors, no results returned.

---

#### 2.3 Data Merging and Processing

- **Overview:**  
  This block merges flight and hotel data streams, then runs a JavaScript code node that normalizes, combines, and ranks travel packages with fallback mock data if APIs fail.

- **Nodes Involved:**  
  - 🔀 Merge Flight & Hotel Data  
  - 🧮 Generate Itinerary Combinations  

- **Node Details:**

  - **🔀 Merge Flight & Hotel Data**  
    - Type: Merge node  
    - Configuration: Combines the two input streams (flights and hotels) using "combine" mode to maintain both datasets for processing.  
    - Inputs: Outputs from flight and hotel HTTP requests  
    - Outputs: Combined data for code processing

  - **🧮 Generate Itinerary Combinations**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Parses merged data to separate flights and hotels.  
      - Limits flights and hotels to top 10 results each.  
      - Extracts search parameters for use in fallback data and processing.  
      - If no flight or hotel data available (API failure), inserts mock data with reasonable defaults.  
      - Calculates number of nights between check-in and check-out.  
      - Processes flights and hotels to normalized structures with fields like airline, price, rating, booking links.  
      - Generates all possible flight-hotel combinations.  
      - Sorts combinations by total package price ascending.  
      - Calculates savings compared to most expensive package.  
      - Returns top 5 travel package itineraries with metadata including counts and timestamp.  
    - Inputs: Merged flight and hotel data  
    - Outputs: JSON with ranked itinerary packages and search metadata  
    - Edge Cases: Incomplete or malformed API data, timezone/date parsing issues, empty results, fallback data correctness.

---

#### 2.4 Email Content Generation

- **Overview:**  
  Converts the ranked travel itineraries JSON into a rich, styled HTML email including detailed itinerary cards and a comparison table.

- **Nodes Involved:**  
  - 🎨 Format HTML Email  

- **Node Details:**

  - **🎨 Format HTML Email**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Processes itinerary data passed from previous code node.  
      - Defines helper functions for date, time, and currency formatting.  
      - Generates detailed itinerary cards including flight and hotel details, prices, booking links, ratings, and highlights best value option.  
      - Builds a comparison table summarizing flight, hotel, individual prices, and total package prices.  
      - Assembles complete responsive HTML email with header, summary stats, itinerary cards, comparison table, and footer disclaimers.  
      - Returns JSON with email subject, HTML body, recipient email, itineraries count, best price, and search parameters for downstream nodes.  
    - Inputs: Itineraries JSON from combinations node  
    - Outputs: Email content JSON for sending  
    - Edge Cases: Special characters in data, malformed URLs, empty itinerary list, formatting errors.

---

#### 2.5 Email Delivery and Response

- **Overview:**  
  Sends the generated HTML email through Gmail and returns an HTTP response to the webhook caller confirming delivery status and summary info.

- **Nodes Involved:**  
  - ✉️ Send via Gmail  
  - 📤 Webhook Response

- **Node Details:**

  - **✉️ Send via Gmail**  
    - Type: Gmail node  
    - Configuration:  
      - Sends email to recipient from previous node's JSON.  
      - Uses subject and HTML message body.  
      - OAuth2 authentication configured for Gmail account.  
    - Inputs: Email content node output  
    - Outputs: Confirmation of email sent  
    - Edge Cases: Gmail OAuth token expiry, sending limits, email delivery failures.

  - **📤 Webhook Response**  
    - Type: Respond to Webhook node  
    - Configuration:  
      - Responds with JSON success message including: confirmation, number of itineraries generated, best price, recipient email, search details, and timestamp.  
    - Inputs: Gmail send confirmation  
    - Outputs: HTTP response to original POST caller  
    - Edge Cases: Response failure if prior node fails, webhook timeout.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                                  | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                                               |
|--------------------------------|-------------------------|-------------------------------------------------|----------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| 📥 Travel Request Webhook       | Webhook                 | Receives travel request via HTTP POST           | External HTTP Request            | 📝 Parse & Validate Inputs     | Introduction: Automates travel itinerary creation by searching flights and hotels via APIs... See detailed intro note.     |
| 📝 Parse & Validate Inputs      | Set                     | Extracts and defaults input parameters          | 📥 Travel Request Webhook         | ✈️ Search Flights (Skyscanner), 🏨 Search Hotels (Booking.com) | Setup Instructions: Add API credentials, configure Gmail OAuth2, customize webhook URL and templates. See note.           |
| ✈️ Search Flights (Skyscanner)  | HTTP Request            | Queries Skyscanner API for flight options       | 📝 Parse & Validate Inputs        | 🔀 Merge Flight & Hotel Data   | Workflow Steps: Trigger, validate, parallel search flights and hotels, merge data, AI generate, format, send, respond.     |
| 🏨 Search Hotels (Booking.com)  | HTTP Request            | Queries Booking.com API for hotel availability  | 📝 Parse & Validate Inputs        | 🔀 Merge Flight & Hotel Data   | Workflow Steps (same as above)                                                                                            |
| 🔀 Merge Flight & Hotel Data    | Merge                   | Combines flight and hotel search results        | ✈️ Search Flights, 🏨 Search Hotels | 🧮 Generate Itinerary Combinations | Workflow Steps (same as above)                                                                                            |
| 🧮 Generate Itinerary Combinations | Code (JavaScript)       | Normalizes, combines, ranks travel packages      | 🔀 Merge Flight & Hotel Data      | 🎨 Format HTML Email           | Workflow Steps (same as above)                                                                                            |
| 🎨 Format HTML Email            | Code (JavaScript)       | Creates rich HTML email with itinerary details  | 🧮 Generate Itinerary Combinations | ✉️ Send via Gmail             | Workflow Steps (same as above)                                                                                            |
| ✉️ Send via Gmail               | Gmail                   | Sends formatted itinerary email                  | 🎨 Format HTML Email              | 📤 Webhook Response            | Workflow Steps (same as above)                                                                                            |
| 📤 Webhook Response             | Respond to Webhook      | Sends HTTP response confirming process success  | ✉️ Send via Gmail                 | None                          | Workflow Steps (same as above)                                                                                            |
| Sticky Note                    | Sticky Note             | Introduction and workflow overview                | None                            | None                          | ## Introduction Automates travel itinerary creation by searching flights and hotels via APIs, then uses AI to generate personalized recommendations delivered as HTML emails through Gmail. ## How It Works Webhook receives travel requests, searches Skyscanner and Booking.com APIs in parallel, merges results, uses AI to create optimized itineraries, formats as HTML email, sends via Gmail. ## Workflow Template Webhook → Parse & Validate → Parallel Searches (Flights: Skyscanner | Hotels: Booking.com) → Merge Data → AI Generate Itinerary → Format HTML Email → Send Gmail → Webhook Response |
| Sticky Note1                   | Sticky Note             | Setup instructions and use cases                  | None                            | None                          | ## Setup Instructions 1. API Configuration: Add Skyscanner and Booking.com API credentials in n8n. 2. Gmail Setup: Configure OAuth2 authentication. 3. Customization: Copy webhook URL, adjust validation rules, modify AI prompts and HTML template. ## Prerequisites - Skyscanner API key - Booking.com API credentials - Gmail with OAuth2 - n8n instance ## Use Cases - Personal vacation planning - Business travel arrangements ## Customization - Add APIs (Kiwi, Expedia) - Filter by budget, Modify email design ## Benefits - Saves 2-3 hours per trip - Real-time pricing comparison |
| Sticky Note2                   | Sticky Note             | Detailed workflow step description                 | None                            | None                          | ## Workflow Steps 1. Trigger & Validate: Webhook receives request, extracts destination/dates/budget/preferences, validates data, converts to API parameters. 2. Parallel Search: Skyscanner fetches flights with price/duration/airline. Booking.com retrieves hotels with ratings/pricing. Merge combines both into single JSON object. 3. AI Generation: AI analyzes merged data, evaluates by price/duration/rating, creates itinerary with daily schedule, pairings, costs, and rankings. 4. Delivery: Converts JSON to HTML email with tables and booking links. Gmail sends email. Webhook confirms success. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Name: `📥 Travel Request Webhook`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `travel-search`  
   - Response Mode: Use response node (respondToWebhook node later)  
   - Accept raw JSON body  
   - Save and note the webhook URL for external requests.

2. **Create a Set Node for Input Parsing**  
   - Name: `📝 Parse & Validate Inputs`  
   - Type: Set  
   - Configure fields:  
     - `destination` = expression `{{$json.body.destination || 'Shanghai'}}`  
     - `departure` = expression `{{$json.body.departure || 'New York'}}`  
     - `checkInDate` = expression `{{$json.body.checkInDate || '2025-12-01'}}`  
     - `checkOutDate` = expression `{{$json.body.checkOutDate || '2025-12-08'}}`  
     - `notificationEmail` = expression `{{$json.body.notificationEmail || $json.body.email}}`  
     - `adults` = expression `{{$json.body.adults || 1}}`  
   - Connect output of webhook node to this node.

3. **Create HTTP Request Node for Flights**  
   - Name: `✈️ Search Flights (Skyscanner)`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://sky-scrapper.p.rapidapi.com/api/v1/flights/searchFlights`  
   - Query Parameters:  
     - `originSkyId` = `{{$json.departure}}`  
     - `destinationSkyId` = `{{$json.destination}}`  
     - `originEntityId` = `27537542` (fixed ID)  
     - `destinationEntityId` = `27537579` (fixed ID)  
     - `date` = `{{$json.checkInDate}}`  
     - `adults` = `{{$json.adults}}`  
     - `currency` = `USD`  
     - `market` = `en-US`  
     - `countryCode` = `US`  
   - Set Authentication: Generic HTTP Header with RapidAPI key for Skyscanner  
   - Timeout: 30000 ms  
   - Enable "Continue On Fail" to true  
   - Connect `📝 Parse & Validate Inputs` to this node.

4. **Create HTTP Request Node for Hotels**  
   - Name: `🏨 Search Hotels (Booking.com)`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://booking-com.p.rapidapi.com/v1/hotels/search`  
   - Query Parameters:  
     - `dest_type` = `city`  
     - `dest_id` = `-1746443` (fixed ID for city)  
     - `checkin_date` = `{{$json.checkInDate}}`  
     - `checkout_date` = `{{$json.checkOutDate}}`  
     - `adults_number` = `{{$json.adults}}`  
     - `order_by` = `price`  
     - `filter_by_currency` = `USD`  
     - `units` = `metric`  
     - `room_number` = `1`  
     - `page_number` = `0`  
   - Set Authentication: Generic HTTP Header with RapidAPI key for Booking.com  
   - Timeout: 30000 ms  
   - Enable "Continue On Fail" to true  
   - Connect `📝 Parse & Validate Inputs` to this node in parallel with flight request node.

5. **Create Merge Node to Combine Flight and Hotel Data**  
   - Name: `🔀 Merge Flight & Hotel Data`  
   - Type: Merge  
   - Mode: Combine  
   - Connect outputs of both flight and hotel HTTP request nodes to inputs of this node.

6. **Create Code Node to Generate Itinerary Combinations**  
   - Name: `🧮 Generate Itinerary Combinations`  
   - Type: Code (JavaScript)  
   - Paste provided JavaScript code that:  
     - Extracts flights and hotels from merged data  
     - Applies fallback mock data if API fails  
     - Normalizes and combines data into flight-hotel packages  
     - Sorts and ranks packages by price  
     - Returns top 5 packages with metadata  
   - Connect `🔀 Merge Flight & Hotel Data` output to this node.

7. **Create Code Node to Format HTML Email**  
   - Name: `🎨 Format HTML Email`  
   - Type: Code (JavaScript)  
   - Paste provided JavaScript code that:  
     - Receives itinerary JSON  
     - Generates a styled HTML email with itinerary cards and comparison table  
     - Includes subject and recipient email in output JSON  
   - Connect `🧮 Generate Itinerary Combinations` output to this node.

8. **Create Gmail Node to Send Email**  
   - Name: `✉️ Send via Gmail`  
   - Type: Gmail  
   - Configure OAuth2 credentials for Gmail account  
   - Set "To" field to: `={{ $json.recipient }}`  
   - Set "Subject" field to: `={{ $json.subject }}`  
   - Set "Message" field to: `={{ $json.htmlBody }}`  
   - Connect `🎨 Format HTML Email` output to this node.

9. **Create Respond to Webhook Node**  
   - Name: `📤 Webhook Response`  
   - Type: Respond to Webhook  
   - Set response mode: JSON  
   - Response body: Use expression to build JSON response with success message, itineraries generated, best price, recipient, search details, and timestamp per provided code.  
   - Connect `✉️ Send via Gmail` output to this node.

10. **Activate Workflow and Test**  
    - Test by POSTing JSON travel search requests with parameters like destination, departure, checkInDate, checkOutDate, notificationEmail, and adults to the webhook URL.  
    - Verify email receipt and webhook JSON response.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow automates travel itinerary creation by integrating Skyscanner and Booking.com APIs, generating personalized recommendations emailed via Gmail. | Sticky Note near the webhook node (Introduction)                                                      |
| Setup requires API keys for Skyscanner and Booking.com via RapidAPI, plus Gmail OAuth2 credentials for email delivery.                      | Sticky Note near input parsing node (Setup Instructions)                                              |
| The workflow can be customized to add more APIs (e.g., Kiwi, Expedia), filter packages by budget, or modify email design and content.       | Sticky Note near input parsing node (Customization tip)                                               |
| Benefits include saving 2-3 hours per trip planning and real-time price comparisons for flights and hotels.                                 | Sticky Note near input parsing node (Benefits)                                                        |
| Detailed workflow steps: input reception, parallel API searches, data merging, AI combination/ranking, email formatting, sending, and webhook response. | Sticky Note near merge node (Workflow Steps)                                                          |
| Ensure API credentials and OAuth tokens are kept up to date to avoid authentication errors and ensure uninterrupted service.               | General best practice — no explicit note but important for production deployment                       |
| HTML email design includes responsive styling and highlights best value packages with badges and color coding for clarity.                  | Code node generating HTML email (🎨 Format HTML Email)                                                 |

---

This completes the comprehensive structured documentation of the **Smart Travel Package Finder** workflow, enabling understanding, reproduction, and modification by advanced users or automation agents.