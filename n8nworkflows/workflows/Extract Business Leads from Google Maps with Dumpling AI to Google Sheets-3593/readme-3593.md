Extract Business Leads from Google Maps with Dumpling AI to Google Sheets

https://n8nworkflows.xyz/workflows/extract-business-leads-from-google-maps-with-dumpling-ai-to-google-sheets-3593


# Extract Business Leads from Google Maps with Dumpling AI to Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of local business leads from Google Maps using Dumpling AI and logs the structured data into a Google Sheet. It is designed primarily for marketers, sales teams, agencies, virtual assistants, or anyone needing to build lead lists or conduct location-specific outreach efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Initiates the workflow manually or via other triggers.
- **1.2 AI-Powered Google Maps Search**: Sends a search query to Dumpling AI to retrieve structured business data from Google Maps.
- **1.3 Data Splitting**: Splits the array of business results into individual items for processing.
- **1.4 Data Logging**: Appends each business record as a row into a specified Google Sheet tab.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow execution manually, allowing users to test or trigger the process on demand.

- **Nodes Involved:**  
  - Trigger: Manual Test Run

- **Node Details:**  
  - **Trigger: Manual Test Run**  
    - Type: Manual Trigger  
    - Role: Entry point for manual workflow execution.  
    - Configuration: No parameters; triggers workflow when user manually initiates it.  
    - Inputs: None  
    - Outputs: Triggers the next node "Search Google Maps via Dumpling AI".  
    - Edge Cases: None typical; user must manually trigger.  
    - Version: Compatible with n8n v1.x and above.

#### 2.2 AI-Powered Google Maps Search

- **Overview:**  
  Sends a POST request to Dumpling AI’s API to perform a Google Maps search based on a predefined query string. It retrieves a JSON response containing an array of places matching the search.

- **Nodes Involved:**  
  - Search Google Maps via Dumpling AI

- **Node Details:**  
  - **Search Google Maps via Dumpling AI**  
    - Type: HTTP Request  
    - Role: Connects to Dumpling AI API to fetch business listings.  
    - Configuration:  
      - Method: POST  
      - URL: `https://app.dumplingai.com/api/v1/search-maps`  
      - Body: JSON with fields:  
        - `query`: `"best+restaurants+in+New+York"` (default, customizable)  
        - `language`: `"en"`  
      - Authentication: HTTP Header Auth with Dumpling AI API key in Authorization header.  
    - Key Expressions: The JSON body is statically set but can be parameterized for dynamic queries.  
    - Inputs: Trigger node output  
    - Outputs: JSON response containing `places[]` array with detailed business info.  
    - Edge Cases:  
      - API key invalid or expired → authentication failure.  
      - Network timeout or API downtime.  
      - Query returns empty or malformed data.  
      - Rate limiting or credit exhaustion on Dumpling AI account.  
    - Version: HTTP Request node version 4.2 used.

#### 2.3 Data Splitting

- **Overview:**  
  Takes the array of places from Dumpling AI’s response and splits it into individual items so each business can be processed separately.

- **Nodes Involved:**  
  - Split Places List for Processing

- **Node Details:**  
  - **Split Places List for Processing**  
    - Type: Split Out  
    - Role: Splits the `places` array into separate workflow items.  
    - Configuration:  
      - Field to split out: `places`  
    - Inputs: Output from HTTP Request node containing `places[]` array.  
    - Outputs: Individual JSON objects representing each place.  
    - Edge Cases:  
      - Empty or missing `places` array → no output items.  
      - Malformed JSON → expression failure.  
    - Version: Split Out node version 1.

#### 2.4 Data Logging

- **Overview:**  
  Appends each individual business record as a new row into a Google Sheet tab named `Leads`, mapping relevant fields such as name, address, phone number, website, rating, price level, type, booking link, and position.

- **Nodes Involved:**  
  - Save Results to Google Sheet (Place Info)

- **Node Details:**  
  - **Save Results to Google Sheet (Place Info)**  
    - Type: Google Sheets  
    - Role: Appends rows to a Google Sheet with structured business data.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Linked to a specific Google Sheet (URL cached in node)  
      - Sheet Name: `Leads` tab  
      - Columns mapped explicitly with expressions:  
        - `Name`: `{{$json.title}}`  
        - `Address`: `{{$json.address}}`  
        - `Phone number`: `{{$json.phoneNumber}}`  
        - `Website`: `{{$json.website}}`  
        - `Rating`: `{{$json.rating}}`  
        - `Price Level`: `{{$json.priceLevel}}`  
        - `Type`: `{{$json.type}}`  
        - `Booking Link`: `{{$json.bookingLinks[0]}}` (first booking link if available)  
        - `Position`: `{{$json.position}}`  
      - Mapping Mode: Define below (explicit column mapping)  
      - Credential: Google Sheets OAuth2 account connected.  
    - Inputs: Individual place JSON from Split Out node.  
    - Outputs: None (append operation)  
    - Edge Cases:  
      - Google Sheets API quota exceeded or permission denied.  
      - Missing or malformed fields in input JSON → empty cells or errors.  
      - Network issues during append operation.  
    - Version: Google Sheets node version 4.5.

---

### 3. Summary Table

| Node Name                         | Node Type          | Functional Role                      | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                                         |
|----------------------------------|--------------------|------------------------------------|-----------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                      | Sticky Note        | Documentation and workflow summary | None                        | None                           | 🔍 Workflow Goal: Automatically search Google Maps using Dumpling AI and log results to Google Sheets. See detailed steps and tips. |
| Trigger: Manual Test Run          | Manual Trigger     | Starts workflow manually            | None                        | Search Google Maps via Dumpling AI |                                                                                                                                    |
| Search Google Maps via Dumpling AI | HTTP Request       | Sends search query to Dumpling AI  | Trigger: Manual Test Run     | Split Places List for Processing |                                                                                                                                    |
| Split Places List for Processing  | Split Out          | Splits places array into items      | Search Google Maps via Dumpling AI | Save Results to Google Sheet (Place Info) |                                                                                                                                    |
| Save Results to Google Sheet (Place Info) | Google Sheets      | Appends each place to Google Sheet  | Split Places List for Processing | None                           |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Trigger: Manual Test Run`  
   - Purpose: To manually start the workflow for testing.

3. **Add an HTTP Request node:**  
   - Name: `Search Google Maps via Dumpling AI`  
   - Set Method to `POST`.  
   - URL: `https://app.dumplingai.com/api/v1/search-maps`  
   - Authentication: Select `Header Auth` and configure with your Dumpling AI API key in the Authorization header.  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "query": "best+restaurants+in+New+York",
       "language": "en"
     }
     ```  
   - Connect the Manual Trigger node output to this node’s input.

4. **Add a Split Out node:**  
   - Name: `Split Places List for Processing`  
   - Field to split out: `places` (this corresponds to the array returned by Dumpling AI)  
   - Connect the HTTP Request node output to this node’s input.

5. **Add a Google Sheets node:**  
   - Name: `Save Results to Google Sheet (Place Info)`  
   - Operation: Append  
   - Connect your Google Sheets OAuth2 credentials.  
   - Document ID: Link to your Google Sheet document (the one with the `Leads` tab).  
   - Sheet Name: `Leads`  
   - Define columns explicitly with the following mappings:  
     - `Name`: `={{ $json.title }}`  
     - `Address`: `={{ $json.address }}`  
     - `Phone number`: `={{ $json.phoneNumber }}`  
     - `Website`: `={{ $json.website }}`  
     - `Rating`: `={{ $json.rating }}`  
     - `Price Level`: `={{ $json.priceLevel }}`  
     - `Type`: `={{ $json.type }}`  
     - `Booking Link`: `={{ $json.bookingLinks[0] }}` (first booking link)  
     - `Position`: `={{ $json.position }}`  
   - Connect the Split Out node output to this node’s input.

6. **Connect all nodes in sequence:**  
   Manual Trigger → HTTP Request → Split Out → Google Sheets Append.

7. **Test the workflow:**  
   - Run the Manual Trigger node.  
   - Verify that the Google Sheet is populated with business leads matching the query.

8. **Optional customization:**  
   - Modify the `query` field in the HTTP Request node to target different locations or business types.  
   - Replace the Manual Trigger with a Schedule Trigger or Webhook node for automated or event-driven runs.  
   - Add filtering or enrichment nodes as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Dumpling AI signup and API key generation required for the HTTP Request node authentication.    | https://www.dumplingai.com/                      |
| Google Sheet must have a tab named `Leads` with columns: Name, Address, Phone number, Website, Rating, Price Level, Type, Booking Link, Position | Google Sheets setup instructions in workflow description |
| To automate runs, replace Manual Trigger with Schedule or Webhook nodes.                         | Workflow customization tips                      |
| Workflow saves Dumpling AI credits per query; monitor usage to avoid unexpected costs.           | Dumpling AI API usage considerations             |
| Possible extensions: add GPT nodes for lead qualification or summaries, or integrate with CRM.  | Suggested workflow enhancements                   |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and customizing the "Extract Business Leads from Google Maps with Dumpling AI to Google Sheets" workflow in n8n.