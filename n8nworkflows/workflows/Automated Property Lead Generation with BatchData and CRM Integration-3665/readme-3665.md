Automated Property Lead Generation with BatchData and CRM Integration

https://n8nworkflows.xyz/workflows/automated-property-lead-generation-with-batchdata-and-crm-integration-3665


# Automated Property Lead Generation with BatchData and CRM Integration

### 1. Workflow Overview

This n8n workflow, "Automated Property Lead Generation with BatchData and CRM Integration," automates the discovery and notification of high-potential real estate investment opportunities. It operates on a scheduled basis to scan real estate markets for properties that match specified criteria, identify new or changed listings, filter for prospects with strong investment potential (e.g., high equity, absentee owners), enrich those leads with detailed information, and notify sales or acquisition teams via email and Slack.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Market Scanning**: Periodic triggering and configuration of API parameters for property search.
- **1.2 BatchData API Query and Result Retrieval**: Calling BatchData's API with search criteria to obtain property listings.
- **1.3 Previous Data Retrieval and Comparison**: Managing state to detect new or changed properties against past scans.
- **1.4 Property Splitting and Filtering**: Splitting individual property records and filtering for high-potential leads.
- **1.5 Detailed Property Data Enrichment**: Fetching comprehensive property and owner details for filtered leads.
- **1.6 Notification Formatting and Delivery**: Creating formatted email and Slack messages and dispatching them to teams.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Market Scanning

**Overview:**  
This block triggers the workflow on a customizable schedule and sets up the search parameters and authentication data to query the BatchData API.

**Nodes Involved:**  
- Schedule Market Scan  
- BatchData API Configuration

**Node Details:**  

- **Schedule Market Scan**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow execution every hour (interval set to hours).  
  - *Configuration:* Simple hourly trigger; no additional parameters.  
  - *Input/Output:* No input; outputs trigger event to next node.  
  - *Potential Failures:* Timing misconfigurations; node downtime causes missed runs.

- **BatchData API Configuration**  
  - *Type:* Set  
  - *Role:* Stores API key and property search parameters (e.g., Austin TX, market values between $250k-$600k, minimum equity 30%, property type = SFR, limit 100).  
  - *Configuration:*  
    - `apiKey` stored as a string (placeholder "YOUR_BATCHDATA_API_KEY").  
    - `searchParameters` as an object with city, state, market value range, equity, property type, and limit.  
  - *Key Expressions:* JavaScript expression sets `searchParameters` object.  
  - *Input/Output:* Receives trigger event; outputs to HTTP Request node.  
  - *Edge Cases:* Missing or invalid API key leads to API authentication failure.

---

#### 1.2 BatchData API Query and Result Retrieval

**Overview:**  
Performs the actual property search by sending a POST request to BatchData’s API using the configured parameters and credentials.

**Nodes Involved:**  
- Query BatchData Properties

**Node Details:**  

- **Query BatchData Properties**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to `https://api.batchdata.com/api/v1/properties/search` with search parameters in JSON body.  
  - *Configuration:*  
    - URL and method fixed as above.  
    - Authorization via HTTP Header with API key provided by previous node.  
    - Body parameters dynamically set from `searchParameters` fields (city, state, market values, equity, property type, limit).  
  - *Expressions:* Parameters bound with expressions like `={{ $json.searchParameters.city }}`.  
  - *Input/Output:* Receives API key and parameters; outputs list of matching properties.  
  - *Potential Failures:* API rate limits, invalid parameters, network timeouts, authentication failures.  
  - *Version-Specific:* Requires HTTP Request node v4+ for generic header auth.

---

#### 1.3 Previous Data Retrieval and Comparison

**Overview:**  
Retrieves the results from previous workflow runs and compares them with the new API results, identifying new and changed property listings.

**Nodes Involved:**  
- Get Previous Results  
- Compare Results

**Node Details:**  

- **Get Previous Results**  
  - *Type:* Code  
  - *Role:* Loads stored previous property data from static/global workflow data; initializes if none found.  
  - *Function:* Uses `getWorkflowStaticData` for persistent data across workflow runs to maintain continuity.  
  - *Inputs:* Current batch of properties from API.  
  - *Outputs:* Combines current properties and previous properties in single JSON for comparison.  
  - *Edge Cases:* Empty or corrupted static data may cause comparison to fail.

- **Compare Results**  
  - *Type:* Code  
  - *Role:* Compares current and previous property arrays to find:  
    - New properties (not seen before)  
    - Changed properties (existing properties with changes in key fields like marketValue, status, ownerStatus, lastSaleDate).  
  - *Method:* Builds a map by property ID for rapid lookup.  
  - Stores the current properties back into static data for next run persistence.  
  - *Output:* JSON object with `newProperties`, `changedProperties`, and combined `allChanges`.  
  - *Potential Failures:* Unexpected input structure, key missing fields, comparing non-identical IDs.

---

#### 1.4 Property Splitting and Filtering

**Overview:**  
Separates each identified changed or new property into individual messages and applies filters to select only high-potential leads based on equity and owner status.

**Nodes Involved:**  
- Split Properties  
- Filter High Potential

**Node Details:**  

- **Split Properties**  
  - *Type:* Split Out  
  - *Role:* Splits the array `allChanges` into single property items for individual processing downstream.  
  - *Parameters:* Field to split out is `allChanges`.  
  - *Input/Output:* Input is array of changed/new properties; output is one property per item.  

- **Filter High Potential**  
  - *Type:* Filter  
  - *Role:* Applies conditions to identify properties with:  
    - Equity percentage > 40%  
    - Owner status containing "absentee" (case-sensitive and strict match).  
  - *Parameters:* Two conditions combined with AND operator.  
  - *Input/Output:* Filters each property item; passes only matching leads.  
  - *Possible Issue:* Case sensitivity might exclude variants like "Absentee" or "ABSENTEE" unless normalized.

---

#### 1.5 Detailed Property Data Enrichment

**Overview:**  
For each filtered lead, retrieves comprehensive property details and owner information from BatchData API to prepare content for notifications.

**Nodes Involved:**  
- Get Property Details

**Node Details:**  

- **Get Property Details**  
  - *Type:* HTTP Request  
  - *Role:* Sends GET request to `https://api.batchdata.com/api/v1/properties/{{ $json.id }}` to fetch full details on specific property by its ID.  
  - *Authentication:* Uses the same API key in authorization header.  
  - *Input/Output:* Input is filtered property with ID; output enriched property details including address, valuation, owner contacts, last sale info, and more.  
  - *Potential Failures:* Invalid or missing property ID, API errors, rate limiting.  
  - *Expressions:* URL dynamically constructed using property ID.

---

#### 1.6 Notification Formatting and Delivery

**Overview:**  
Formats detailed email and Slack messages for high-potential leads, and sends notifications to the defined sales team email and Slack channel.

**Nodes Involved:**  
- Format Email Content  
- Send Email Alert  
- Post to Slack

**Node Details:**  

- **Format Email Content**  
  - *Type:* Set  
  - *Role:* Creates structured content for email and Slack notifications using property and owner data.  
  - *Configuration:*  
    - Generates an `emailSubject` combining address components.  
    - Constructs rich HTML `emailContent` with property specs, owner info, equity percentage, and Google Maps link based on property address.  
    - Prepares plain text styled Slack message with similar data and map link.  
  - *Expressions:* Use mustache-style templating (e.g. `{{ $json.address.street }}`) with conditional fallbacks for absent data.  
  - *Input/Output:* Input enriched property details; outputs formatted message fields.

- **Send Email Alert**  
  - *Type:* Email Send  
  - *Role:* Sends an email to `salesteam@yourcompany.com` with alerts from formatted content.  
  - *Parameters:*  
    - From: `alerts@yourcompany.com`  
    - To: `salesteam@yourcompany.com`  
    - Subject from `emailSubject`  
    - Body from previous node’s content (implied).  
  - *Credentials:* Requires configured SMTP credentials.  
  - *Failure Modes:* SMTP misconfiguration, delivery failures, invalid recipient addresses.

- **Post to Slack**  
  - *Type:* Slack  
  - *Role:* Posts notification message to Slack channel via webhook.  
  - *Configuration:* Text pulled from formatted Slack message field.  
  - *Credentials:* Requires valid Slack webhook credentials.  
  - *Failure Modes:* Network issues, webhook invalidation, rate limits.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                        | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                             |
|-------------------------|--------------------|-------------------------------------|--------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Market Scan     | Schedule Trigger   | Workflow trigger every hour          | —                              | BatchData API Configuration      | ## Main Workflow Flow This part of the workflow handles the regular scanning ...                        |
| BatchData API Configuration | Set              | Store API key and search parameters  | Schedule Market Scan             | Query BatchData Properties        | ## Main Workflow Flow This part of the workflow handles the regular scanning ...                        |
| Query BatchData Properties | HTTP Request      | Search properties via BatchData API  | BatchData API Configuration     | Get Previous Results              | ## Main Workflow Flow This part of the workflow handles the regular scanning ...                        |
| Get Previous Results     | Code               | Retrieve stored previous properties  | Query BatchData Properties       | Compare Results                  | ## Main Workflow Flow This part of the workflow handles the regular scanning ...                        |
| Compare Results          | Code               | Compare current with previous properties | Get Previous Results          | Split Properties                 | ## Main Workflow Flow This part of the workflow handles the regular scanning ...                        |
| Split Properties         | Split Out           | Split combined properties into singles | Compare Results               | Filter High Potential            | ## Property Filtering & Analysis Here we filter the properties based on criteria ...                    |
| Filter High Potential    | Filter             | Select high equity & absentee owner properties | Split Properties           | Get Property Details             | ## Property Filtering & Analysis Here we filter the properties based on criteria ...                    |
| Get Property Details     | HTTP Request       | Fetch detailed property info         | Filter High Potential           | Format Email Content             | ## Property Filtering & Analysis Here we filter the properties based on criteria ...                    |
| Format Email Content     | Set                | Format email and Slack notification  | Get Property Details            | Send Email Alert, Post to Slack  | ## Notifications This section delivers the property leads to the sales team through multiple channels: |
| Send Email Alert         | Email Send         | Email alert with property lead       | Format Email Content            | —                               | ## Notifications This section delivers the property leads to the sales team through multiple channels: |
| Post to Slack            | Slack              | Slack notification for quick alerts  | Format Email Content            | —                               | ## Notifications This section delivers the property leads to the sales team through multiple channels: |
| Sticky Note Main Flow    | Sticky Note        | Explanation on main scanning flow    | —                              | —                               |                                                                                                       |
| Sticky Note Filter & Analyze | Sticky Note    | Explanation on filtering & analysis  | —                              | —                               |                                                                                                       |
| Sticky Note Notifications | Sticky Note       | Explanation on notification delivery | —                              | —                               |                                                                                                       |
| Sticky Note Instructions | Sticky Note        | Setup instructions for API, SMTP, Slack | —                          | —                               |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**:  
   - Add a **Schedule Trigger** node named "Schedule Market Scan".  
   - Set interval to every "1 hour".  
   
2. **Configure API Key & Search Parameters**:  
   - Add a **Set** node named "BatchData API Configuration".  
   - Define two fields:  
     - `apiKey`: string, set your BatchData API key here.  
     - `searchParameters`: object with:  
       `{ city: "Austin", state: "TX", minimumMarketValue: 250000, maximumMarketValue: 600000, minimumEquity: 30, propertyType: ["SFR"], limit: 100 }`  
   - Connect "Schedule Market Scan" → "BatchData API Configuration".

3. **Query BatchData API**:  
   - Add **HTTP Request** node named "Query BatchData Properties".  
   - Set method to POST.  
   - URL: `https://api.batchdata.com/api/v1/properties/search`.  
   - Authentication: HTTP Header Auth using `apiKey` from previous node.  
   - Body parameters: Map properties from `searchParameters` object in incoming JSON to POST parameters: city, state, minimumMarketValue, maximumMarketValue, minimumEquity, propertyType, limit.  
   - Connect "BatchData API Configuration" → "Query BatchData Properties".  
   - Configure credentials with your BatchData API key.

4. **Retrieve Previous Results and Initialize if Needed**:  
   - Add a **Code** node "Get Previous Results" with JavaScript:  
     ```js
     const workflowStaticData = getWorkflowStaticData('global');
     if (!workflowStaticData.hasOwnProperty('previousProperties')) {
       workflowStaticData.previousProperties = [];
     }
     return [{
       json: {
         ...items[0].json,
         previousProperties: workflowStaticData.previousProperties,
         currentProperties: items[0].json.data.properties || [],
       }
     }];
     ```  
   - Connect "Query BatchData Properties" → "Get Previous Results".

5. **Compare Current and Previous Property Data**:  
   - Add another **Code** node "Compare Results".  
   - Insert this JS to identify new and changed properties:  
     ```js
     const currentProperties = items[0].json.currentProperties;
     const previousProperties = items[0].json.previousProperties;
     const previousPropertiesMap = {};
     for (const property of previousProperties) {
       previousPropertiesMap[property.id] = property;
     }
     const newProperties = currentProperties.filter(p => !previousPropertiesMap[p.id]);
     const changedProperties = currentProperties.filter(p => {
       const prev = previousPropertiesMap[p.id];
       if (!prev) return false;
       return p.marketValue !== prev.marketValue ||
              p.status !== prev.status ||
              p.ownerStatus !== prev.ownerStatus ||
              p.lastSaleDate !== prev.lastSaleDate;
     });
     const workflowStaticData = getWorkflowStaticData('global');
     workflowStaticData.previousProperties = currentProperties;
     return [{
       json: {
         ...items[0].json,
         newProperties,
         changedProperties,
         allChanges: [...newProperties, ...changedProperties],
       }
     }];
     ```  
   - Connect "Get Previous Results" → "Compare Results".

6. **Split Properties for Individual Processing**:  
   - Add **Split Out** node "Split Properties".  
   - Set field to split: `allChanges`.  
   - Connect "Compare Results" → "Split Properties".

7. **Filter High Potential Properties**:  
   - Add **Filter** node "Filter High Potential".  
   - Add two filters combined with AND:  
     - Numeric filter: `equityPercentage` > 40  
     - String contains filter: `ownerStatus` contains `"absentee"` (case sensitive)  
   - Connect "Split Properties" → "Filter High Potential".

8. **Fetch Detailed Property Data**:  
   - Add **HTTP Request** node "Get Property Details".  
   - Method: GET.  
   - URL: `https://api.batchdata.com/api/v1/properties/{{ $json.id }}`.  
   - Auth: HTTP Header Auth with API key same as previous.  
   - Connect "Filter High Potential" → "Get Property Details".

9. **Format Notifications Content**:  
   - Add **Set** node "Format Email Content".  
   - Create fields using templated strings:  
     - `emailSubject`: `"New Property Opportunity: {{ $json.address.street }}, {{ $json.address.city }}, {{ $json.address.state }}"`  
     - `emailContent`: build HTML string with property and owner details and Google Maps link.  
     - `slackMessage`: concise text with property summary and maps link in Slack markdown.  
   - Connect "Get Property Details" → "Format Email Content".

10. **Send Email Alerts**:  
    - Add **Email Send** node "Send Email Alert".  
    - Configure:  
      - From: `alerts@yourcompany.com`  
      - To: `salesteam@yourcompany.com`  
      - Subject: from `emailSubject` field  
      - Content: from `emailContent` field (HTML).  
    - Connect "Format Email Content" → "Send Email Alert".  
    - Set up SMTP credentials in n8n.

11. **Post to Slack Notification**:  
    - Add **Slack** node "Post to Slack".  
    - Text input: use `slackMessage` field.  
    - Connect "Format Email Content" → "Post to Slack".  
    - Configure Slack webhook credentials.

**Important:**  
- Configure BatchData HTTP header authentication using API key in all HTTP Request nodes.  
- Ensure SMTP and Slack credentials are set in n8n credential manager.  
- Adjust search/filter parameters as per your target market and criteria.  
- Retain static workflow data for proper new/changed property detection.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                |
|--------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| BatchData API overview: provides extensive US property and owner info including valuation, sales, foreclosure data | https://www.batchdata.com/ (official platform site)           |
| Alert email contains detailed property and owner info plus direct Google Maps link                                 | Embedded in Format Email Content node                           |
| Slack messages are brief updates linking to property location                                                     | Embedded in Format Email Content node                           |
| Setup requires adding your BatchData API key and configuring SMTP and Slack credentials                            | See Sticky Note Instructions in workflow for step reminders    |
| Workflow aims to save hours of manual property research and identify early investment leads via automation        | Workflow description in overview                                |
| Pay attention to case sensitivity in owner status filter to avoid missing matches                                  | Filter High Potential node’s string condition                  |

---

This completes the structured reference document for the "Automated Property Lead Generation with BatchData and CRM Integration" n8n workflow. It should enable detailed understanding, reliable reproduction, and safe modification for future enhancements or customizations.