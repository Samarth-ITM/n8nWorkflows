Compare Flight Fares from Skyscanner, Air India & More with Email Alerts

https://n8nworkflows.xyz/workflows/compare-flight-fares-from-skyscanner--air-india---more-with-email-alerts-6054


# Compare Flight Fares from Skyscanner, Air India & More with Email Alerts

### 1. Workflow Overview

This workflow automates the process of checking and comparing real-time flight fares from multiple airline and travel aggregator APIs (Skyscanner, Air India, IndiGo, Akasa Air). It is designed to run on a scheduled basis, aggregating flight data for specified routes and dates, consolidating all results, sorting them by price, and finally sending a summarized email alert with the best flight deals. The workflow is ideal for users who want to monitor flight prices dynamically across several providers and receive timely notifications to save on travel costs.

The workflow logic is divided into the following functional blocks:

- **1.1 Scheduled Trigger and Input Setup:** Automatically initiates the workflow at defined intervals and sets flight search parameters.
- **1.2 API Data Fetching:** Calls the flight data APIs of Skyscanner, Akasa Air, Air India, and IndiGo in parallel.
- **1.3 Data Merging:** Combines the data returned by individual APIs into a unified dataset.
- **1.4 Data Comparison and Sorting:** Processes the unified dataset to extract relevant flight details and sorts the options by price.
- **1.5 Notification:** Sends an email with the sorted flight fare information to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Input Setup

**Overview:**  
This block sets up a scheduled trigger to run the workflow automatically at defined intervals. It then prepares the input parameters (origin, destination, departure and return dates) for the flight search APIs.

**Nodes Involved:**  
- Set Schedule  
- Set Input Data  
- Sticky Note (Trigger scheduling explanation)  
- Sticky Note (Input parameters explanation)  

**Node Details:**  

- **Set Schedule**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow automatically based on a time interval schedule.  
  - Configuration: Uses a generic interval trigger (e.g., every hour/day; exact interval unspecified).  
  - Inputs: None (trigger node)  
  - Outputs: Triggers next node with JSON containing flight search parameters (expected in `body` property).  
  - Failures: Misconfiguration of schedule or lack of parameters in trigger payload may cause failures.

- **Set Input Data**  
  - Type: Set  
  - Role: Extracts and sets flight search parameters (origin, destination, departureDate, returnDate) from the schedule trigger output.  
  - Configuration: Sets four string fields using expressions referencing `Set Schedule` node outputs (e.g., `={{$node['Set Schedule'].json.body.origin}}`).  
  - Inputs: From Set Schedule  
  - Outputs: Passes structured parameters to downstream API request nodes.  
  - Failures: Expression failures if schedule trigger payload is missing expected fields.

- **Sticky Note (Trigger Explanation)**  
  - Content: "Triggers the workflow at a scheduled time to check flight fares automatically."  
  - Coverage: Explains the role of schedule triggering.

- **Sticky Note (Input Parameters Explanation)**  
  - Content: "Sets the input parameters like origin, destination, and dates for flight searches"  
  - Coverage: Clarifies the purpose of the Set Input Data node.

---

#### 1.2 API Data Fetching

**Overview:**  
This block makes HTTP requests to multiple flight fare APIs (Skyscanner, Akasa Air, Air India, IndiGo) concurrently using the parameters set earlier. It retrieves live flight fare data from each provider.

**Nodes Involved:**  
- Skyscanner API  
- Akasa Air API  
- Air India API  
- IndiGo API  
- Sticky Note (API data fetching explanation)

**Node Details:**

- **Skyscanner API**  
  - Type: HTTP Request  
  - Role: Fetches flight data from Skyscanner API endpoint (`https://api.skyscanner.net/flights`).  
  - Configuration: Simple GET request with no additional options configured (assumed parameters are passed via URL or headers, not explicitly shown).  
  - Inputs: From Set Input Data  
  - Outputs: JSON response with flight data.  
  - Failures: Could fail due to authentication errors, rate limits, or network issues.

- **Akasa Air API**  
  - Type: HTTP Request  
  - Role: Fetches flight data from Akasa Air API endpoint (`https://api.akasa.com/v1/flights`).  
  - Configuration: Basic GET request, no explicit auth or parameters shown.  
  - Inputs: From Set Input Data  
  - Outputs: JSON flight data.  
  - Failures: Similar to Skyscanner API.

- **Air India API**  
  - Type: HTTP Request  
  - Role: Fetches flight data from Air India API endpoint (`https://api.airindia.net/flights`).  
  - Configuration: Basic GET request.  
  - Inputs: From Set Input Data  
  - Outputs: JSON flight data.  
  - Failures: Network or API errors possible.

- **IndiGo API**  
  - Type: HTTP Request  
  - Role: Fetches flight data from IndiGo API endpoint (`https://api.idigo.com/v1/flights`).  
  - Configuration: Basic GET request.  
  - Inputs: From Set Input Data  
  - Outputs: JSON flight data.  
  - Failures: Similar risks as above.

- **Sticky Note (API Explanation)**  
  - Content: "Fetches live flight fare data from different airlines using the provided API endpoints."  
  - Coverage: Describes the purpose of the four HTTP Request nodes.

---

#### 1.3 Data Merging

**Overview:**  
Multiple merge nodes consolidate the results obtained from the various APIs into a single combined dataset, facilitating unified processing in subsequent steps.

**Nodes Involved:**  
- Merge API Data  
- Merge Both API Data  
- Merge All API Results  
- Sticky Note1  
- Sticky Note7  
- Sticky Note2  

**Node Details:**

- **Merge API Data**  
  - Type: Merge  
  - Role: Combines results from Skyscanner API and Akasa Air API.  
  - Configuration: Mode set to "mergeByIndex" to merge items at corresponding indexes.  
  - Inputs: Skyscanner API (main input), Akasa Air API (secondary input)  
  - Outputs: Merged dataset combining flight data from these two sources.  
  - Failures: If inputs have mismatched data length or missing data, merge may produce incomplete results.

- **Merge Both API Data**  
  - Type: Merge  
  - Role: Merges Air India API and IndiGo API results.  
  - Configuration: mergeByIndex.  
  - Inputs: Air India API (main), IndiGo API (secondary)  
  - Outputs: Combined Air India + IndiGo data.  
  - Failures: Similar to above.

- **Merge All API Results**  
  - Type: Merge  
  - Role: Final consolidation of the two previously merged datasets: (Skyscanner + Akasa Air) and (Air India + IndiGo).  
  - Configuration: mergeByIndex.  
  - Inputs: Merge API Data (main), Merge Both API Data (secondary)  
  - Outputs: Unified flight data from all four providers.  
  - Failures: Data structure inconsistencies can cause issues.

- **Sticky Notes:**  
  - Sticky Note1: "Combines the flight data from Skyscanner and Akasa Air into a single dataset."  
  - Sticky Note7: "Merges the flight data from Air India and IndiGo with the previous dataset."  
  - Sticky Note2: "Consolidates all API data (Skyscanner, Akasa Air, Air India, IndiGo) into one unified result."  

---

#### 1.4 Data Comparison and Sorting

**Overview:**  
This block processes the unified dataset to extract relevant flight information such as provider, price, currency, and booking URL. It then sorts all available flight options by price ascending to identify the best deals.

**Nodes Involved:**  
- Compare Data and Sorting Price  
- Sticky Note3  

**Node Details:**

- **Compare Data and Sorting Price**  
  - Type: Function  
  - Role: Custom JavaScript code iterates through the merged results, extracts flight fare details, and sorts them by price.  
  - Configuration:  
    - Checks existence of data arrays inside merged results at indexes 0 and 1.  
    - For each flight entry, extracts: provider name, price, currency, booking URL.  
    - Combines all flights into a single array.  
    - Sorts array by price ascending.  
    - Returns the sorted array as JSON.  
  - Inputs: From Merge All API Results  
  - Outputs: Sorted flight fare list.  
  - Failures:  
    - If input data is malformed or missing expected properties, function may throw errors.  
    - Price sorting may fail if price fields are non-numeric or absent.  

- **Sticky Note3:**  
  - Content: "Compares all flight fares and sorts them by price to find the best deals."  

---

#### 1.5 Notification

**Overview:**  
Sends an email containing the sorted flight fare comparison results to a predefined user email address.

**Nodes Involved:**  
- Send Response via Email  
- Sticky Note4  

**Node Details:**

- **Send Response via Email**  
  - Type: Email Send  
  - Role: Sends flight fare update email with sorted results to the user.  
  - Configuration:  
    - Email subject: "Real-Time Flight Fare Update"  
    - Recipient: abc@gmail.com (hardcoded)  
    - Sender: xyz@gmail.com (hardcoded)  
    - Email format: Plain text  
    - Email body: Includes `{json.results}`, which should be the output of the previous node (Compare Data and Sorting Price).  
    - Credentials: Uses SMTP credentials named "SMTP -test" configured in n8n.  
  - Inputs: From Compare Data and Sorting Price  
  - Outputs: None  
  - Failures:  
    - SMTP authentication errors or connectivity issues.  
    - Empty or malformed email body if previous node outputs nothing.

- **Sticky Note4:**  
  - Content: "Sends the sorted flight fare comparison results to the user via email."  

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                                | Input Node(s)           | Output Node(s)             | Sticky Note                                                       |
|-------------------------|--------------------|-----------------------------------------------|------------------------|----------------------------|------------------------------------------------------------------|
| Set Schedule            | Schedule Trigger   | Triggers workflow on schedule                   | None                   | Set Input Data             | Triggers the workflow at a scheduled time to check flight fares automatically. |
| Set Input Data          | Set                | Sets flight search parameters                   | Set Schedule            | Skyscanner API, Akasa Air API, Air India API, IndiGo API | Sets the input parameters like origin, destination, and dates for flight searches |
| Skyscanner API          | HTTP Request       | Fetches flight data from Skyscanner             | Set Input Data          | Merge API Data             | Fetches live flight fare data from different airlines using the provided API endpoints. |
| Akasa Air API           | HTTP Request       | Fetches flight data from Akasa Air               | Set Input Data          | Merge API Data             | Fetches live flight fare data from different airlines using the provided API endpoints. |
| Merge API Data          | Merge              | Combines Skyscanner and Akasa Air data          | Skyscanner API, Akasa Air API | Merge All API Results     | Combines the flight data from Skyscanner and Akasa Air into a single dataset. |
| Air India API           | HTTP Request       | Fetches flight data from Air India                | Set Input Data          | Merge Both API Data        | Fetches live flight fare data from different airlines using the provided API endpoints. |
| IndiGo API              | HTTP Request       | Fetches flight data from IndiGo                   | Set Input Data          | Merge Both API Data        | Fetches live flight fare data from different airlines using the provided API endpoints. |
| Merge Both API Data     | Merge              | Combines Air India and IndiGo data                | Air India API, IndiGo API | Merge All API Results     | Merges the flight data from Air India and IndiGo with the previous dataset. |
| Merge All API Results   | Merge              | Consolidates all merged API data                  | Merge API Data, Merge Both API Data | Compare Data and Sorting Price | Consolidates all API data (Skyscanner, Akasa Air, Air India, IndiGo) into one unified result. |
| Compare Data and Sorting Price | Function       | Extracts flight info and sorts by price          | Merge All API Results   | Send Response via Email    | Compares all flight fares and sorts them by price to find the best deals. |
| Send Response via Email | Email Send         | Sends flight fare comparison results via email  | Compare Data and Sorting Price | None                    | Sends the sorted flight fare comparison results to the user via email. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Set Schedule`  
   - Set the interval to your preferred schedule (e.g., every hour or daily).  
   - This node will trigger the workflow automatically.

3. **Add a Set node:**  
   - Name: `Set Input Data`  
   - Connect from `Set Schedule`.  
   - Add four string fields: `origin`, `destination`, `departureDate`, `returnDate`.  
   - Set their values using expressions to pull from the `Set Schedule` node’s JSON body, e.g.:  
     - `origin` = `{{$node["Set Schedule"].json["body"]["origin"]}}`  
     - Similar for the other three fields.  
   - This node prepares the flight search parameters.

4. **Add four HTTP Request nodes for APIs:**  
   - Names and URLs:  
     - `Skyscanner API` → `https://api.skyscanner.net/flights`  
     - `Akasa Air API` → `https://api.akasa.com/v1/flights`  
     - `Air India API` → `https://api.airindia.net/flights`  
     - `IndiGo API` → `https://api.idigo.com/v1/flights`  
   - Each node’s method: GET  
   - Connect all four nodes from `Set Input Data` node.  
   - Configure query parameters or headers as required by each API (note: actual parameters not detailed in source, so use appropriate API documentation).  
   - Ensure authentication is configured if needed.

5. **Add a Merge node:**  
   - Name: `Merge API Data`  
   - Mode: `mergeByIndex`  
   - Connect inputs from `Skyscanner API` (main input) and `Akasa Air API` (secondary input).

6. **Add another Merge node:**  
   - Name: `Merge Both API Data`  
   - Mode: `mergeByIndex`  
   - Connect inputs from `Air India API` (main) and `IndiGo API` (secondary).

7. **Add a third Merge node:**  
   - Name: `Merge All API Results`  
   - Mode: `mergeByIndex`  
   - Connect inputs from `Merge API Data` (main) and `Merge Both API Data` (secondary).

8. **Add a Function node:**  
   - Name: `Compare Data and Sorting Price`  
   - Connect from `Merge All API Results`.  
   - Paste the following JavaScript code:  
     ```javascript
     const results = [];

     // Process Skyscanner results
     if ($node['Merge All API Results'].json[0].data) {
       $node['Merge All API Results'].json[0].data.forEach(flight => {
         results.push({
           provider: 'Skyscanner',
           price: flight.price,
           currency: flight.currency,
           booking_url: flight.booking_url
         });
       });
     }

     // Process Akasa Air results
     if ($node['Merge All API Results'].json[1].data) {
       $node['Merge All API Results'].json[1].data.forEach(flight => {
         results.push({
           provider: 'Akasa Air',
           price: flight.price,
           currency: flight.currency,
           booking_url: flight.booking_url
         });
       });
     }

     // Process Air India and IndiGo data may be included here or separately depending on data structure.
     // Adjust as needed for those providers.

     // Sort by price ascending
     results.sort((a, b) => a.price - b.price);

     return results;
     ```
   - Modify the code to process all four providers’ data if necessary.

9. **Add an Email Send node:**  
   - Name: `Send Response via Email`  
   - Connect from `Compare Data and Sorting Price`.  
   - Set parameters:  
     - Subject: "Real-Time Flight Fare Update"  
     - To Email: `abc@gmail.com` (replace with actual recipient)  
     - From Email: `xyz@gmail.com` (replace with authorized sender email)  
     - Email Format: Plain text  
     - Text: Use expression to include results, e.g., `{{$json}}` or formatted output.  
   - Assign SMTP credentials configured in n8n for sending emails.

10. **Test the workflow:**  
    - Trigger manually or wait for scheduled execution.  
    - Verify API requests succeed, data merges correctly, sorting works, and email is sent with the expected content.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The workflow requires valid API endpoints and credentials for each airline/travel provider.   | API documentation for Skyscanner, Akasa Air, Air India, IndiGo.    |
| SMTP credentials must be configured in n8n to enable email sending.                           | n8n SMTP Credentials setup guide.                                  |
| The schedule trigger can be customized to any interval depending on user preference.          | n8n Schedule Trigger node documentation.                           |
| Sticky notes in the workflow provide contextual explanations for easier understanding.       | Visible within the n8n workflow editor.                            |
| Adjust JavaScript code in Function node to match actual API response formats if different.    | n8n Function node scripting guide.                                 |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automation workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.