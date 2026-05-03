Compare Hotel Prices Across Booking Platforms with Scrape.do and Google Sheets

https://n8nworkflows.xyz/workflows/compare-hotel-prices-across-booking-platforms-with-scrape-do-and-google-sheets-10949


# Compare Hotel Prices Across Booking Platforms with Scrape.do and Google Sheets

### 1. Workflow Overview

This workflow automates the comparison of hotel prices across multiple booking platforms (Booking.com, Agoda, Expedia) by scraping real-time data using Scrape.do and then sending a summarized email report to the user. It is designed to receive user input via a form, validate and normalize that input, execute parallel scraping operations, consolidate the results, and notify the user with the best price options.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Validation**  
  Captures user input from a web form, validates essential parameters (hotel name, city, check-in/out dates, email), and prepares normalized data for subsequent processing.

- **1.2 Data Scraping and Normalization**  
  In parallel, sends HTTP requests to Scrape.do API to scrape Booking.com, Agoda, and Expedia with the user parameters. Parses and normalizes the extracted data for uniformity.

- **1.3 Analysis and User Notification**  
  Merges all scraped data, analyzes prices to find the best deals, constructs an email summary, and sends it to the user’s email address.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block triggers upon form submission, captures the user’s input, and normalizes the request data to ensure consistent date formats and default values when necessary.

**Nodes Involved:**  
- On form submission  
- Parse & Validate Request

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing user input from a web form titled "Best Hotel Prices" with fields: Hotel Name (required), Check-in Date (required), Check-out Date (required), City (required), Email (required).  
  - *Configuration:* Webhook ID set for external form submissions.  
  - *Input/Output:* No input; outputs form data JSON.  
  - *Edge cases:* Missing required fields will prevent submission; empty email or date fields fallback to defaults in downstream node.  

- **Parse & Validate Request**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Normalize and validate form data, ensuring all necessary fields exist, applying fallback values if missing.  
  - *Configuration:* Extracts fields from input JSON, assigns defaults: hotel name "Hotel", city "Istanbul", check-in "2024-12-25", check-out "2024-12-28". Also duplicates check-in/out dates into ISO format fields.  
  - *Key expressions:* Uses `$input.first().json` to access form data; fallback logic with `||`.  
  - *Input:* Form submission node output.  
  - *Output:* JSON with normalized hotelName, city, checkIn, checkOut, Email, and ISO-format dates.  
  - *Edge cases:* If user omits any field, defaults prevent failures downstream; potential silent use of defaults if user input is invalid (no explicit validation beyond presence).  

---

#### 2.2 Data Scraping and Normalization

**Overview:**  
This block performs parallel scraping of hotel prices from Booking.com, Agoda, and Expedia using the Scrape.do API. Each platform’s raw data is then parsed and normalized to a standard format including platform name, price, currency, room type, availability, and URLs.

**Nodes Involved:**  
- Scrape Booking.com  
- Parse Booking.com Data  
- Scrape Agoda  
- Parse Agoda Data  
- Scrape Expedia  
- Parse Expedia Data  

**Node Details:**

- **Scrape Booking.com**  
  - *Type:* HTTP Request  
  - *Role:* Calls Scrape.do API with a dynamically constructed Booking.com search URL, including hotel name, city, and check-in/out dates.  
  - *Configuration:*  
    - URL: `https://api.scrape.do`  
    - Query parameters: token from `$vars.SCRAPEDO_TOKEN`, URL with encoded hotel+city, check-in/out dates, render true, wait until DOM content loaded, no resource blocking, returnJSON true, geoCode US.  
    - Timeout: 60 seconds.  
  - *Input:* Normalized user input.  
  - *Output:* JSON response from Scrape.do.  
  - *Edge cases:* Scrape.do API failures, timeouts, invalid token (requires valid SCRAPEDO_TOKEN global variable). `continueOnFail` enabled to prevent workflow halt on failures.

- **Parse Booking.com Data**  
  - *Type:* Code Node  
  - *Role:* Simulates parsing of Booking.com scraped data (currently generates random price between 180 and 280 USD). Extracts and preserves original user data fields.  
  - *Key expressions:* Uses `$input.first().json` for previous data.  
  - *Output:* Normalized JSON with platform="Booking.com", random price, currency USD, roomType "Standard Room", static URL, availability true, plus hotel and user details.  
  - *Edge cases:* Placeholder parsing logic; real scraping data format may differ, requiring adjustment.

- **Scrape Agoda**  
  - *Type:* HTTP Request  
  - *Role:* Similar to Booking.com, sends request to Scrape.do with Agoda-specific URL parameters.  
  - *Configuration:* URL encodes city and hotel name, check-in/out dates, same Scrape.do parameters as above.  
  - *Input:* Normalized user input.  
  - *Edge cases:* Same as Booking.com scraping node.

- **Parse Agoda Data**  
  - *Type:* Code Node  
  - *Role:* Simulates parsing Agoda data with random pricing, preserves user data.  
  - *Output:* JSON with platform="Agoda.com", random price, currency USD, standard room, availability true.  
  - *Edge cases:* Same as Booking.com parsing node.

- **Scrape Expedia**  
  - *Type:* HTTP Request  
  - *Role:* Sends scraping request targeting Expedia search results via Scrape.do with parameters city, dates, hotel.  
  - *Configuration:* Similar to other scraping nodes with Scrape.do API.  
  - *Edge cases:* Same as other scraping nodes.

- **Parse Expedia Data**  
  - *Type:* Code Node  
  - *Role:* Simulates parsing Expedia scraped data, random price generation, preserves user data fields.  
  - *Output:* JSON normalized with platform="Expedia.com", price, currency, room type, availability.  
  - *Edge cases:* Same placeholder parsing limitations.

---

#### 2.3 Analysis and User Notification

**Overview:**  
This block merges the data from all scraping nodes, aggregates and sorts prices, composes a detailed email body summarizing hotel price comparisons, and sends the email to the user.

**Nodes Involved:**  
- Merge  
- Code in JavaScript (price analysis & email composition)  
- Send a message (Gmail)

**Node Details:**

- **Merge**  
  - *Type:* Merge Node  
  - *Role:* Combines outputs from the four parallel inputs: parsed Booking.com, Agoda, Expedia data, and the original normalized input.  
  - *Configuration:* Number of inputs set to 4, merging method default (likely "Wait for all inputs").  
  - *Input:* Parsed data nodes and original data node.  
  - *Output:* Aggregated array of all platform price data plus user info.  
  - *Edge cases:* If any scraping node fails and returns no data, merge handles partial data due to `continueOnFail`.

- **Code in JavaScript**  
  - *Type:* Code Node  
  - *Role:* Processes merged items to:  
    - Extract user email and hotel info  
    - Collect all platform prices in array  
    - Sort prices ascending  
    - Compose a multiline email body with medals (🥇🥈🥉) for top prices, total platforms counted, cheapest price and savings if applicable  
    - Defaults to a test email if none provided  
  - *Key expressions:* Iterates over all merged items; constructs string with Unicode characters for clarity; careful preservation of email and hotel data.  
  - *Output:* JSON containing emailBody, hotelName, city, userEmail, price count, and success flag.  
  - *Edge cases:* Handles empty price array by reporting no prices found; default test email ensures no email sending failure.

- **Send a message**  
  - *Type:* Gmail Node  
  - *Role:* Sends the composed email to the user using Gmail integration.  
  - *Configuration:*  
    - Recipient: `{{ $json.userEmail }}`  
    - Subject: "Otel Fiyatları - {{ $json.hotelName }} ({{ $json.city }})"  
    - Message body: `{{ $json.emailBody }}`  
    - Uses Gmail OAuth2 credentials (configured separately).  
  - *Input:* Output from code node.  
  - *Edge cases:* Requires valid Gmail credentials; email sending failure possible if credentials expired or quota exceeded.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                       | Input Node(s)                  | Output Node(s)             | Sticky Note                                                  |
|------------------------|---------------------|------------------------------------|-------------------------------|----------------------------|--------------------------------------------------------------|
| On form submission     | Form Trigger        | Capture user input from web form    | -                             | Parse & Validate Request    | Section 1 Input & Validation                                 |
| Parse & Validate Request| Code                | Normalize and validate form data    | On form submission            | Scrape Booking.com, Scrape Agoda, Scrape Expedia, Merge | Section 1 Input & Validation                                 |
| Scrape Booking.com     | HTTP Request        | Scrape Booking.com data via Scrape.do | Parse & Validate Request       | Parse Booking.com Data      | Section 2 Data Scraping & Normalization; Warning SCRAPEDO_TOKEN required |
| Parse Booking.com Data | Code                | Parse Booking.com scraped data      | Scrape Booking.com            | Merge                      | Section 2 Data Scraping & Normalization                      |
| Scrape Agoda           | HTTP Request        | Scrape Agoda data via Scrape.do     | Parse & Validate Request       | Parse Agoda Data            | Section 2 Data Scraping & Normalization; Warning SCRAPEDO_TOKEN required |
| Parse Agoda Data       | Code                | Parse Agoda scraped data            | Scrape Agoda                  | Merge                      | Section 2 Data Scraping & Normalization                      |
| Scrape Expedia         | HTTP Request        | Scrape Expedia data via Scrape.do   | Parse & Validate Request       | Parse Expedia Data          | Section 2 Data Scraping & Normalization; Warning SCRAPEDO_TOKEN required |
| Parse Expedia Data     | Code                | Parse Expedia scraped data          | Scrape Expedia                | Merge                      | Section 2 Data Scraping & Normalization                      |
| Merge                  | Merge               | Combine all parsed data and input   | Parse Booking.com Data, Parse Agoda Data, Parse Expedia Data, Parse & Validate Request | Code in JavaScript         | Section 3 Analysis & Notification                            |
| Code in JavaScript     | Code                | Analyze prices and compose email    | Merge                        | Send a message             | Section 3 Analysis & Notification                            |
| Send a message         | Gmail               | Send email notification to user     | Code in JavaScript            | -                          | Section 3 Analysis & Notification                            |
| Sticky Note Main       | Sticky Note         | Overview and setup instructions     | -                             | -                          | Main workflow explanation and setup instructions            |
| Sticky Note Section 1  | Sticky Note         | Input & Validation description      | -                             | -                          | Section 1 Input & Validation                                 |
| Sticky Note Section 2  | Sticky Note         | Data Scraping & Normalization description | -                        | -                          | Section 2 Data Scraping & Normalization                      |
| Sticky Note Section 3  | Sticky Note         | Analysis & Notification description | -                             | -                          | Section 3 Analysis & Notification                            |
| Sticky Note Warning    | Sticky Note         | SCRAPEDO_TOKEN variable requirement | -                             | -                          | Warning for SCRAPEDO_TOKEN variable requirement             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the trigger node: "On form submission"**  
   - Node type: Form Trigger  
   - Configure webhook with a unique ID (auto-generated or user-defined).  
   - Form title: "Best Hotel Prices"  
   - Form fields (all required):  
     - Hotel Name (text)  
     - Check-in Date (date)  
     - Check-out Date (date)  
     - City (text)  
     - Email (text)  

2. **Add "Parse & Validate Request" node**  
   - Node type: Code (JavaScript)  
   - Connect input from "On form submission" node.  
   - Code logic: Extract form fields from input JSON, assign default values if fields are missing (hotelName: "Hotel", city: "Istanbul", checkIn/Out: "2024-12-25"/"2024-12-28"), duplicate check-in/out into ISO strings, preserve Email.  
   - Output normalized JSON object.

3. **Create three parallel HTTP Request nodes for scraping:**  
   - Credentials: None required for HTTP Request nodes, but ensure a global variable or credential named `SCRAPEDO_TOKEN` is created with your Scrape.do API token.  
   
   a. **Scrape Booking.com**  
      - URL: `https://api.scrape.do`  
      - Query parameters:  
        - token: `{{$vars.SCRAPEDO_TOKEN}}`  
        - url: `https://www.booking.com/searchresults.html?ss={{encodeURIComponent(hotelName + ' ' + city)}}&checkin={{checkInISO}}&checkout={{checkOutISO}}`  
        - render: `true`  
        - waitUntil: `domcontentloaded`  
        - blockResources: `false`  
        - returnJSON: `true`  
        - geoCode: `us`  
      - Timeout: 60 seconds  
      - Connect input from "Parse & Validate Request".  
      - Enable `continueOnFail` to true.

   b. **Scrape Agoda**  
      - URL: `https://api.scrape.do`  
      - Query parameters:  
        - token: `{{$vars.SCRAPEDO_TOKEN}}`  
        - url: `https://www.agoda.com/search?city={{encodeURIComponent(city)}}&checkIn={{checkInISO}}&checkOut={{checkOutISO}}&searchTerm={{encodeURIComponent(hotelName)}}`  
        - Other parameters as above.  
      - Timeout 60 seconds  
      - Connect input from "Parse & Validate Request".  
      - Enable `continueOnFail`.

   c. **Scrape Expedia**  
      - URL: `https://api.scrape.do`  
      - Query parameters:  
        - token: `{{$vars.SCRAPEDO_TOKEN}}`  
        - url: `https://www.expedia.com/Hotel-Search?destination={{encodeURIComponent(city)}}&startDate={{checkInISO}}&endDate={{checkOutISO}}&searchTerms={{encodeURIComponent(hotelName)}}`  
        - Other parameters as above.  
      - Timeout 60 seconds  
      - Connect input from "Parse & Validate Request".  
      - Enable `continueOnFail`.

4. **Add three Code nodes to parse scraped data:**  
   - For each platform (Booking.com, Agoda, Expedia), create a Code node connected respectively to the HTTP Request node outputs.  
   - Code logic:  
     - Receive scraping response.  
     - Generate a random price between 180 and 280 USD (placeholder for real parsing).  
     - Set platform name according to source.  
     - Set currency to "USD".  
     - Set roomType to "Standard Room".  
     - Set availability to true.  
     - Set URL to main platform homepage.  
     - Preserve original hotelName, city, checkIn, checkOut, and Email fields from input.  

5. **Add a Merge node:**  
   - Set number of inputs to 4.  
   - Connect the outputs of three Parse nodes and the original "Parse & Validate Request" node to the Merge node inputs.  
   - Use default merge mode to wait for all inputs.

6. **Add a Code node "Code in JavaScript" to analyze and compose email:**  
   - Connect input from Merge node.  
   - Code logic:  
     - Iterate all merged items to extract user email, hotel info, and each platform’s price data.  
     - Sort prices ascending.  
     - Compose a multi-line string email body including hotel details, platform prices with medal emojis (🥇🥈🥉), cheapest price highlight, and savings calculation.  
     - Use 'bugrahan@example.com' as fallback email if none provided.  
     - Return JSON with emailBody, hotelName, city, userEmail, priceCount, success flag.

7. **Add "Send a message" node (Gmail):**  
   - Credentials: Configure Gmail OAuth2 credentials in n8n.  
   - Parameters:  
     - Send To: `={{ $json.userEmail }}`  
     - Subject: `"Otel Fiyatları - {{ $json.hotelName }} ({{ $json.city }})"`  
     - Message: `={{ $json.emailBody }}`  
   - Connect input from the "Code in JavaScript" node.

8. **Create global variable or credential for `SCRAPEDO_TOKEN`:**  
   - Obtain API token from https://scrape.do after signup.  
   - Add token as a global variable or environment credential accessible in workflow.  
   - This token is used in all HTTP Request nodes.

9. **Testing and Deployment:**  
   - Use the webhook URL from "On form submission" node to submit test data via the form interface.  
   - Verify emails are sent with correct prices and formatting.  
   - Monitor logs for scraping errors or timeouts.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                  |
|----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow requires a valid Scrape.do API token stored in `SCRAPEDO_TOKEN` variable for scraping functionality.           | Warning Sticky Note                                              |
| Gmail node requires OAuth2 credential setup to send emails on behalf of user.                                               | Gmail Node Setup                                                |
| The scraping parsing nodes currently simulate prices with random values; real HTML parsing logic should be implemented for production use. | Placeholder parsing logic in Parse Booking.com, Agoda, Expedia nodes |
| Workflow overview and setup instructions are documented in the main sticky note node at the top of the workflow.           | Sticky Note Main                                                 |
| To extend sources, add additional HTTP Request nodes targeting other booking platforms and parse their responses similarly. | Customization note in Main Sticky Note                           |
| Scrape.do official website for API token: https://scrape.do                                                                | External resource                                                |