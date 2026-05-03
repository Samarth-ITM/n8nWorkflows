Compare Flight Prices Across Multiple Booking Platforms with Email Reports

https://n8nworkflows.xyz/workflows/compare-flight-prices-across-multiple-booking-platforms-with-email-reports-9788


# Compare Flight Prices Across Multiple Booking Platforms with Email Reports

### 1. Workflow Overview

This workflow automates the process of comparing flight prices across multiple popular booking platforms and delivering detailed flight price reports via email. It is designed to accept natural language flight search queries, parse and validate the input, scrape flight data in parallel from different vendors, analyze and aggregate the results, then format and send an email report back to the user.

**Target Use Cases:**  
- Travelers seeking best flight deals without manually visiting multiple platforms.  
- Automated flight price monitoring with email notifications.  
- Integrating flight price comparison into chatbots or other frontends via webhook input.

**Logical Blocks:**  

- **1.1 Input Reception:** Webhook receives flight search requests with user details and preferences.  
- **1.2 Input Parsing & Validation:** Code node extracts important flight parameters from natural language, validates completeness.  
- **1.3 Request Validation Branch:** Conditional node directing whether to proceed or return error.  
- **1.4 Parallel Flight Data Scraping:** Four SSH nodes run Python scrapers concurrently for Kayak, Skyscanner, Expedia, Google Flights.  
- **1.5 Aggregation & Analysis:** Code node consolidates scraped data, computes best deals and statistics.  
- **1.6 Email Report Formatting:** Code node prepares a plain text email summarizing flight options and best deals.  
- **1.7 Email Sending:** Email node sends the formatted report to the user’s email.  
- **1.8 Webhook Response:** Two response nodes send success or error JSON responses back to the original requester.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures incoming HTTP POST requests containing flight search queries, user email, and optional alert preferences.

- **Nodes Involved:**  
  - Webhook - Receive Flight Request

- **Node Details:**  
  - **Name:** Webhook - Receive Flight Request  
  - **Type:** Webhook  
  - **Technical Role:** Entry point for receiving flight search requests.  
  - **Configuration:**  
    - HTTP Method: POST  
    - Path: `/flight-price-compare`  
    - Allowed Origins: All (`*`)  
    - Responds using response node downstream.  
  - **Inputs:** None (external HTTP trigger)  
  - **Outputs:** JSON payload forwarded to next node  
  - **Edge Cases:** Missing or malformed requests, CORS issues if origins restricted  
  - **Sticky Note:** Explains input stage with example JSON (message, email)  

#### 1.2 Input Parsing & Validation

- **Overview:**  
  Parses natural language query and request body to extract flight parameters like origin, destination, dates, trip type, passengers, cabin class. Validates presence and format of required data.

- **Nodes Involved:**  
  - Parse & Validate Flight Request

- **Node Details:**  
  - **Name:** Parse & Validate Flight Request  
  - **Type:** Code (JavaScript)  
  - **Technical Role:** NLP enhanced parser and validator for flight search parameters.  
  - **Configuration:**  
    - Extracts email, user name, notification preference.  
    - Uses regex and mapping for airport codes and date parsing.  
    - Identifies trip type (one-way or round-trip).  
    - Checks for missing required fields and returns helpful messages if incomplete.  
  - **Key Expressions:** Multiple regex patterns for route and dates, airport code mappings, date parsing functions.  
  - **Inputs:** JSON from webhook  
  - **Outputs:** JSON with parsed parameters and status (`ready` or `missing_info` or `greeting`)  
  - **Edge Cases:** Unrecognized city names, invalid dates, incomplete input, greeting detection returns help message.  
  - **Sticky Note:** Details parsing and validation logic  

#### 1.3 Request Validation Branch

- **Overview:**  
  Checks if parsing output status is `ready` to proceed with scraping or else triggers error response.

- **Nodes Involved:**  
  - Check If Request Valid

- **Node Details:**  
  - **Name:** Check If Request Valid  
  - **Type:** If  
  - **Technical Role:** Flow control node validating readiness.  
  - **Configuration:** Condition checks if `$json.status == 'ready'`  
  - **Inputs:** Output of Parse & Validate Flight Request  
  - **Outputs:**  
    - If true: Connects to scrapers  
    - If false: Connects to webhook error response  
  - **Edge Cases:** False negatives if status field missing or altered  

#### 1.4 Parallel Flight Data Scraping

- **Overview:**  
  Executes four SSH nodes in parallel to run Python scraping scripts for Kayak, Skyscanner, Expedia, and Google Flights. Continues workflow even if some scrapers fail.

- **Nodes Involved:**  
  - Scrape Kayak  
  - Scrape Skyscanner  
  - Scrape Expedia  
  - Scrape Google Flights

- **Node Details (common):**  
  - **Type:** SSH  
  - **Technical Role:** Executes remote Python scraper passing flight parameters via CLI args.  
  - **Configuration:**  
    - Working directory: `/home/oneclick-server2/`  
    - Command template: `python3 flight_scraper.py [origin] [destination] [departureDateISO] [returnDateISO] [tripType] [passengers] [cabinClass] [platform]`  
    - Authentication: Private key SSH with configured credentials  
    - On error: Continue with error output (does not halt workflow)  
  - **Inputs:** Parsed flight parameters from validation node  
  - **Outputs:** STDOUT with scraped flight data lines or STDERR on failure  
  - **Edge Cases:** SSH connectivity issues, scraper script errors, timeouts, incomplete data  
  - **Sticky Note:** Explains parallel scraping with output format:  
    `AIRLINE|PRICE|DURATION|STOPS|TIME|TIME|URL`  

#### 1.5 Aggregation & Analysis

- **Overview:**  
  Aggregates output from all scrapers, parses flight details, identifies best deal and direct flights, computes average and max prices, and summarizes errors.

- **Nodes Involved:**  
  - Aggregate & Analyze Prices

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Technical Role:** Data consolidation, error handling, sorting, and statistics computation.  
  - **Configuration:**  
    - Iterates over all scraper outputs.  
    - Parses each line, extracts price (numeric), airline, stops, duration, times, booking URL.  
    - Filters out invalid or empty results.  
    - Sorts flights by price ascending.  
    - Computes best deal, average price, savings versus max price.  
    - Identifies best direct (non-stop) flight if available.  
  - **Inputs:** Combined outputs from all scraper SSH nodes  
  - **Outputs:** JSON with status, flight results array, best deals, stats, errors, timestamps  
  - **Edge Cases:** No flights found, all scrapers fail, malformed output lines  
  - **Sticky Note:** Describes analyze stage with outputs and processing steps  

#### 1.6 Email Report Formatting

- **Overview:**  
  Builds a plain text email report summarizing flight search, best deal highlight, top 10 flight options, and statistics suitable for sending.

- **Nodes Involved:**  
  - Format Email Report

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Technical Role:** Constructs user-friendly flight price comparison email content.  
  - **Configuration:**  
    - Checks if no results, sends failure subject and message.  
    - Otherwise formats route, dates, top 10 results with airline, price, stops, platform.  
    - Highlights best deal with savings.  
    - Includes average price, total results, disclaimer.  
  - **Inputs:** Aggregated flight analysis JSON  
  - **Outputs:** JSON with subject and text fields for email  
  - **Edge Cases:** No results scenario formatting  
  - **Sticky Note:** Details email report content and structure  

#### 1.7 Email Sending

- **Overview:**  
  Sends the formatted flight price comparison report to the user's email address using SMTP.

- **Nodes Involved:**  
  - Send Email Report

- **Node Details:**  
  - **Type:** Email Send  
  - **Technical Role:** Delivers email report to user inbox.  
  - **Configuration:**  
    - To: user's email parsed from input  
    - From: `flights@pricecomparison.com`  
    - Subject and Text: from previous node outputs  
    - Email format: plain text  
    - Credentials: configured SMTP server with test credentials  
  - **Inputs:** Email subject and text JSON from formatter node  
  - **Outputs:** Email sending status forwarded  
  - **Edge Cases:** SMTP authentication failure, invalid recipient address, network issues  

#### 1.8 Webhook Response

- **Overview:**  
  Provides JSON response to original webhook caller indicating success or failure of the request and processing.

- **Nodes Involved:**  
  - Webhook Response (Success)  
  - Webhook Response (Error)

- **Node Details:**  
  - **Webhook Response (Success):**  
    - Responds with success message including best price, airline, total results, and confirmation of email sent.  
  - **Webhook Response (Error):**  
    - Responds with failure message detailing missing info or error status.  
  - Both nodes use JSON response mode to reply to HTTP request.  
  - Inputs come from either success path (after email sent) or error path (invalid request).  
  - Edge cases include unexpected workflow failures, malformed outputs.

- **Sticky Note:** Explains response stage with success and error message contents.

---

### 3. Summary Table

| Node Name                     | Node Type        | Functional Role                     | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                   |
|-------------------------------|------------------|-----------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| Webhook - Receive Flight Request | Webhook          | Input reception                   | None                             | Parse & Validate Flight Request  | Explains input stage with example JSON input                                                 |
| Parse & Validate Flight Request | Code             | Parse & validate input            | Webhook - Receive Flight Request | Check If Request Valid           | Details parsing and validation logic                                                         |
| Check If Request Valid          | If               | Flow control on input validity    | Parse & Validate Flight Request  | Scrape Kayak, Skyscanner, Expedia, Google Flights / Webhook Response (Error) |                                                                                              |
| Scrape Kayak                   | SSH              | Flight data scraping (Kayak)      | Check If Request Valid            | Aggregate & Analyze Prices       | Parallel scraping, output format details                                                     |
| Scrape Skyscanner              | SSH              | Flight data scraping (Skyscanner) | Check If Request Valid            | Aggregate & Analyze Prices       | Parallel scraping, output format details                                                     |
| Scrape Expedia                 | SSH              | Flight data scraping (Expedia)    | Check If Request Valid            | Aggregate & Analyze Prices       | Parallel scraping, output format details                                                     |
| Scrape Google Flights          | SSH              | Flight data scraping (Google)     | Check If Request Valid            | Aggregate & Analyze Prices       | Parallel scraping, output format details                                                     |
| Aggregate & Analyze Prices     | Code             | Aggregate, analyze, and sort data | Scrape Kayak, Skyscanner, Expedia, Google Flights | Format Email Report              | Describes analyze stage with outputs and processing                                         |
| Format Email Report            | Code             | Format flight price email report  | Aggregate & Analyze Prices        | Send Email Report                | Details email report content and structure                                                  |
| Send Email Report              | Email Send       | Send email report to user         | Format Email Report               | Webhook Response (Success)       |                                                                                              |
| Webhook Response (Success)    | Respond to Webhook | Respond success JSON to caller    | Send Email Report                | None                            | Explains response stage success message                                                    |
| Webhook Response (Error)      | Respond to Webhook | Respond error JSON to caller      | Check If Request Valid (false branch) | None                       | Explains response stage error message                                                      |
| Sticky Note                   | Sticky Note      | Documentation notes               | None                             | None                            | 🎯 Workflow Purpose and key features                                                       |
| Sticky Note1                  | Sticky Note      | Documentation notes               | None                             | None                            | 📥 Input stage with example input                                                          |
| Sticky Note2                  | Sticky Note      | Documentation notes               | None                             | None                            | 🧠 Parse stage details                                                                     |
| Sticky Note3                  | Sticky Note      | Documentation notes               | None                             | None                            | 🔍 Scrape stage details                                                                    |
| Sticky Note4                  | Sticky Note      | Documentation notes               | None                             | None                            | 📊 Analyze stage details                                                                   |
| Sticky Note5                  | Sticky Note      | Documentation notes               | None                             | None                            | 📧 Report stage details                                                                    |
| Sticky Note6                  | Sticky Note      | Documentation notes               | None                             | None                            | ✅ Response stage details                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook - Receive Flight Request**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `flight-price-compare`  
   - Response Mode: Response Node  
   - Allowed Origins: `*` (all)  
   - No inputs, output forwards request JSON to next node.

2. **Add Parse & Validate Flight Request (Code node)**  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript code that:  
     - Extracts `message` or `query` from request body  
     - Extracts user email, name, notify preferences  
     - Maps city names to airport codes  
     - Parses departure and return dates with regex  
     - Determines trip type (one-way or round-trip)  
     - Validates required fields (origin, destination, dates)  
     - Returns JSON with status (`ready`, `missing_info` or `greeting`) and parsed parameters.  
   - Connect Webhook output to this node input.

3. **Add Check If Request Valid (If node)**  
   - Condition: Check if `$json.status` equals `ready`  
   - Connect Parse & Validate Flight Request output to this node input.

4. **Add SSH nodes for scraping** (Four nodes: Kayak, Skyscanner, Expedia, Google Flights)  
   - For each:  
     - Type: SSH  
     - Working Directory: `/home/oneclick-server2/`  
     - Command Template:  
       ```bash
       python3 /home/oneclick-server2/flight_scraper.py {{$json.origin}} {{$json.destination}} {{$json.departureDateISO}} {{$json.returnDateISO || ''}} {{$json.tripType}} {{$json.passengers}} {{$json.cabinClass}} [platform]
       ```  
       Replace `[platform]` with `kayak`, `skyscanner`, `expedia`, `googleflights` respectively.  
     - Authentication: SSH private key credentials (create or import valid private key)  
     - On Error: Continue with error output (do not fail workflow)  
   - Connect "true" output of Check If Request Valid to all four SSH nodes in parallel.

5. **Add Aggregate & Analyze Prices (Code node)**  
   - Type: Code (JavaScript)  
   - Paste JavaScript that:  
     - Collects all scraper outputs  
     - Parses each line for flight details (airline, price, stops, times, URL)  
     - Collects errors from scraper failures  
     - Sorts flights by price ascending  
     - Computes best deal, best direct flight, average price, savings  
     - Returns aggregated JSON results.  
   - Connect outputs from all four scraper SSH nodes to this node input (multi-input).

6. **Add Format Email Report (Code node)**  
   - Type: Code (JavaScript)  
   - Paste JavaScript that:  
     - Receives aggregated flight data  
     - Checks for no results case  
     - Formats plain text email report with flight route, dates, best deal, top 10 results, price stats  
     - Returns JSON with `subject` and `text` for email.  
   - Connect Aggregate & Analyze Prices output to this node input.

7. **Add Send Email Report (Email Send node)**  
   - Type: Email Send  
   - To Email: `={{$json.userEmail}}`  
   - From Email: `flights@pricecomparison.com`  
   - Subject: `={{$json.subject}}`  
   - Text: `={{$json.text}}`  
   - Email Format: Plain Text  
   - Credentials: Configure SMTP credentials for sending email (e.g., SMTP server, username, password)  
   - Connect Format Email Report output to this node input.

8. **Add Webhook Response (Success)**  
   - Type: Respond to Webhook  
   - Response Body (JSON):  
     ```json
     {
       "success": true,
       "message": "Flight comparison sent to {{$json.userEmail}}",
       "route": "{{$json.origin}} → {{$json.destination}}",
       "bestPrice": {{$json.bestDeal.price}},
       "airline": "{{$json.bestDeal.airline}}",
       "totalResults": {{$json.totalResults}}
     }
     ```  
   - Connect Send Email Report output to this node input.

9. **Add Webhook Response (Error)**  
   - Type: Respond to Webhook  
   - Response Body (JSON):  
     ```json
     {
       "success": false,
       "message": "={{$json.response || 'Request failed'}}",
       "status": "={{$json.status}}"
     }
     ```  
   - Connect "false" output of Check If Request Valid node to this node input.

10. **Finalize connections:**  
    - Webhook → Parse & Validate Flight Request → Check If Request Valid →  
      - True branch → Scrape Kayak, Scrape Skyscanner, Scrape Expedia, Scrape Google Flights (parallel) → Aggregate & Analyze Prices → Format Email Report → Send Email Report → Webhook Response (Success)  
      - False branch → Webhook Response (Error)

11. **Test trigger:**  
    - Send POST request with JSON body example:  
      ```json
      {
        "message": "Flight from New York to London on 25th March, one-way",
        "email": "user@example.com"
      }
      ```  
    - Verify email receipt and webhook JSON response.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow purpose: Smart Flight Price Comparison across multiple platforms with email reports.    | Sticky Note: 🎯 Workflow Purpose                |
| Input example JSON: message with query and user email for webhook.                               | Sticky Note1: 📥 Input Stage                     |
| Parsing includes airport code mapping and date extraction with regex and NLP-enhanced logic.    | Sticky Note2: 🧠 Parse Stage                      |
| Parallel scraping runs Python scripts remotely via SSH with continued execution on errors.      | Sticky Note3: 🔍 Scrape Stage                     |
| Aggregation node parses scraper outputs, sorts flights, finds best deals and computes stats.    | Sticky Note4: 📊 Analyze Stage                    |
| Email report includes best deal highlight, top 10 flights, stats, and booking links in plain text.| Sticky Note5: 📧 Report Stage                     |
| Responses to webhook caller include success or error JSON with helpful messages and confirmations.| Sticky Note6: ✅ Response Stage                   |
| Scraper script location referenced as `/home/oneclick-server2/flight_scraper.py` on remote SSH. | Requires deployment of scraper scripts and SSH setup. |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.