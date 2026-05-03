Connect Pipedrive Deal Outcomes to GA4 & Google Ads via Measurement Protocol

https://n8nworkflows.xyz/workflows/connect-pipedrive-deal-outcomes-to-ga4---google-ads-via-measurement-protocol-8215


# Connect Pipedrive Deal Outcomes to GA4 & Google Ads via Measurement Protocol

### 1. Workflow Overview

This workflow integrates Pipedrive deal outcomes with Google Analytics 4 (GA4) and Google Ads using the Measurement Protocol. Its main purpose is to track deal stage changes and related person information from Pipedrive, verify user consent and client identification, and send corresponding conversion or event data to GA4 for analytics and advertising attribution.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Listen for deal updates from Pipedrive via webhook trigger.
- **1.2 Deal and Pipeline Data Retrieval:** Fetch detailed deal info and pipeline stages.
- **1.3 Stage Filtering:** Identify if the deal stage corresponds to a target stage for tracking.
- **1.4 Person Data Retrieval:** Fetch the related person’s details and custom fields.
- **1.5 Consent and Client ID Verification:** Extract and verify client_id and consent status.
- **1.6 GA4 Event Construction and Submission:** Build the GA4 measurement protocol payload and send it via HTTP request.
- **1.7 Conditional Branching:** Proceed only if client_id exists and consent is granted.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block starts the workflow by triggering upon changes in Pipedrive deals.

- **Nodes Involved:**  
  - Pipedrive Trigger  
  - Assign Variables

- **Node Details:**

  - **Pipedrive Trigger**  
    - Type: Trigger node, specialized for Pipedrive webhook events.  
    - Configuration: Listens for deal update events (default event settings).  
    - Inputs: None (trigger node).  
    - Outputs: Emits deal update payload.  
    - Edge Cases: Possible webhook misfires or downtime; ensure Pipedrive webhook is correctly configured.  
    - Version: 1.1  

  - **Assign Variables**  
    - Type: Set node for initializing or restructuring data from the trigger.  
    - Configuration: Likely extracts deal ID or relevant identifiers from the trigger event for downstream use.  
    - Inputs: From Pipedrive Trigger.  
    - Outputs: Passes data to “Get a deal” node.  
    - Edge Cases: Missing or malformed data in webhook payload.  
    - Version: 3.4  

---

#### 1.2 Deal and Pipeline Data Retrieval

- **Overview:**  
  Retrieves detailed deal data by ID and fetches all stages of the relevant pipeline.

- **Nodes Involved:**  
  - Get a deal  
  - get stages for pipeline  
  - Split Out  
  - Filter  
  - If correct stage name

- **Node Details:**

  - **Get a deal**  
    - Type: Pipedrive node to retrieve deal details.  
    - Configuration: Uses deal ID from previous node to fetch complete deal info.  
    - Inputs: From Assign Variables.  
    - Outputs: Passes deal data to “get stages for pipeline”.  
    - Edge Cases: Deal ID may not exist or API failure.  
    - Version: 1  

  - **get stages for pipeline**  
    - Type: HTTP Request node.  
    - Configuration: Calls Pipedrive API endpoint to get all stages for the pipeline of the deal.  
    - Inputs: From Get a deal (likely using pipeline ID from deal data).  
    - Outputs: Passes stages array to Split Out.  
    - Edge Cases: API rate limits, missing pipeline ID.  
    - Version: 4.2  

  - **Split Out**  
    - Type: Split Out node.  
    - Configuration: Splits the array of pipeline stages into individual items for filtering.  
    - Inputs: From get stages for pipeline.  
    - Outputs: Passes individual stages to Filter.  
    - Edge Cases: Empty stages array.  
    - Version: 1  

  - **Filter**  
    - Type: Filter node.  
    - Configuration: Filters stages matching certain criteria (e.g., matching deal stage).  
    - Inputs: From Split Out.  
    - Outputs: Passes matching stage(s) to If correct stage name.  
    - Edge Cases: No matching stage found.  
    - Version: 2.2  

  - **If correct stage name**  
    - Type: If node.  
    - Configuration: Checks if the stage name corresponds to a target stage for triggering GA4 event (e.g., “Won”).  
    - Inputs: From Filter.  
    - Outputs: If true, proceeds to “Get a person”; if false, workflow stops or branches off.  
    - Edge Cases: Stage name mismatch or missing data.  
    - Version: 2.2  

---

#### 1.3 Person Data Retrieval

- **Overview:**  
  Fetches the person related to the deal and retrieves their custom fields.

- **Nodes Involved:**  
  - Get a person  
  - get personFields

- **Node Details:**

  - **Get a person**  
    - Type: Pipedrive node to retrieve person details by person ID linked to deal.  
    - Configuration: Uses person ID from deal data.  
    - Inputs: From If correct stage name (true branch).  
    - Outputs: Passes person data to get personFields.  
    - Edge Cases: Missing person ID, API errors.  
    - Version: 1  

  - **get personFields**  
    - Type: HTTP Request node.  
    - Configuration: Calls Pipedrive API to get detailed custom fields of the person (e.g., client_id, consent).  
    - Inputs: From Get a person.  
    - Outputs: Passes full person data including custom fields to “Get client_id and consent_granted”.  
    - Edge Cases: API errors, missing fields.  
    - Version: 4.2  

---

#### 1.4 Consent and Client ID Verification

- **Overview:**  
  Extracts client_id and consent status from person custom fields and conditionally proceeds if both exist.

- **Nodes Involved:**  
  - Get client_id and consent_granted  
  - If client_id exists and consent is given

- **Node Details:**

  - **Get client_id and consent_granted**  
    - Type: Code node (JavaScript).  
    - Configuration: Parses personFields to extract the client_id and consent_granted flags.  
    - Inputs: From get personFields.  
    - Outputs: Passes extracted values to decision node.  
    - Edge Cases: Missing or malformed fields, unexpected data types.  
    - Version: 2  

  - **If client_id exists and consent is given**  
    - Type: If node.  
    - Configuration: Checks if client_id is present and consent_granted is truthy.  
    - Inputs: From Get client_id and consent_granted.  
    - Outputs:  
      - True: Proceeds to “Construct GA4 object”.  
      - False: Ends or branches off (no GA4 event sent).  
    - Edge Cases: Incorrect boolean parsing, missing data.  
    - Version: 2.2  

---

#### 1.5 GA4 Event Construction and Submission

- **Overview:**  
  Constructs the GA4 Measurement Protocol payload and sends it via HTTP request to GA4 endpoint.

- **Nodes Involved:**  
  - Construct GA4 object  
  - HTTP Request — Send GA4

- **Node Details:**

  - **Construct GA4 object**  
    - Type: Code node.  
    - Configuration: Builds JSON payload including client_id, event name, parameters such as transaction value, currency, user properties, etc., based on deal and person data.  
    - Inputs: From If client_id exists and consent is given (true branch).  
    - Outputs: Passes constructed payload to HTTP Request node.  
    - Edge Cases: Missing required fields, invalid data formatting.  
    - Version: 2  

  - **HTTP Request — Send GA4**  
    - Type: HTTP Request node.  
    - Configuration: Sends POST request to GA4 Measurement Protocol endpoint with JSON payload constructed earlier.  
    - Inputs: From Construct GA4 object.  
    - Outputs: Receives response from GA4 API, which could be logged or used for error handling.  
    - Edge Cases: Network timeouts, HTTP errors, invalid API keys, quota limits.  
    - Version: 4.2  

---

### 3. Summary Table

| Node Name                          | Node Type              | Functional Role                                     | Input Node(s)                | Output Node(s)                     | Sticky Note                    |
|-----------------------------------|------------------------|----------------------------------------------------|-----------------------------|----------------------------------|-------------------------------|
| Pipedrive Trigger                 | Pipedrive Trigger      | Initiates workflow on Pipedrive deal updates       | None                        | Assign Variables                  |                               |
| Assign Variables                 | Set                    | Extracts key variables from trigger event          | Pipedrive Trigger           | Get a deal                       |                               |
| Get a deal                       | Pipedrive              | Retrieves detailed deal information                 | Assign Variables            | get stages for pipeline          |                               |
| get stages for pipeline          | HTTP Request           | Fetches stages for the deal’s pipeline              | Get a deal                  | Split Out                       |                               |
| Split Out                       | Split Out              | Splits stages array into single stage items         | get stages for pipeline     | Filter                         |                               |
| Filter                         | Filter                 | Filters stages for relevant stage matching          | Split Out                  | If correct stage name            |                               |
| If correct stage name           | If                     | Checks if stage corresponds to target stage         | Filter                      | Get a person                    |                               |
| Get a person                   | Pipedrive              | Retrieves person details linked to deal              | If correct stage name       | get personFields                |                               |
| get personFields              | HTTP Request           | Fetches person’s custom fields                        | Get a person                | Get client_id and consent_granted |                               |
| Get client_id and consent_granted | Code                   | Extracts client_id and consent status                | get personFields            | If client_id exists and consent is given |                               |
| If client_id exists and consent is given | If                     | Checks presence of client_id and consent              | Get client_id and consent_granted | Construct GA4 object (true), none (false) |                               |
| Construct GA4 object           | Code                   | Builds GA4 Measurement Protocol payload              | If client_id exists and consent is given | HTTP Request — Send GA4      |                               |
| HTTP Request — Send GA4       | HTTP Request           | Sends event data to GA4 Measurement Protocol         | Construct GA4 object        | None                           |                               |
| Sticky Note                   | Sticky Note            | (No content)                                          |                             |                                  |                               |
| Sticky Note1                  | Sticky Note            | (No content)                                          |                             |                                  |                               |
| Sticky Note2                  | Sticky Note            | (No content)                                          |                             |                                  |                               |
| Sticky Note3                  | Sticky Note            | (No content)                                          |                             |                                  |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Pipedrive Trigger” Node**  
   - Type: Pipedrive Trigger  
   - Configuration: Listen for “deal updated” events (default).  
   - No credentials required beyond Pipedrive OAuth2.  

2. **Create “Assign Variables” Node**  
   - Type: Set node  
   - Configuration: Extract deal ID and other useful fields from the trigger data to pass downstream.  
   - Connect output of “Pipedrive Trigger” to input of this node.  

3. **Create “Get a deal” Node**  
   - Type: Pipedrive node  
   - Configuration: Set operation to “Get Deal” using deal ID from “Assign Variables”.  
   - Connect “Assign Variables” output to this node input.  

4. **Create “get stages for pipeline” Node**  
   - Type: HTTP Request  
   - Configuration:  
     - Method: GET  
     - URL: Pipedrive API endpoint for pipeline stages, e.g., `/pipelines/{pipelineId}/stages`  
     - Use pipeline ID from “Get a deal” node.  
     - Authentication: Pipedrive OAuth2 credentials.  
   - Connect “Get a deal” output to this node input.  

5. **Create “Split Out” Node**  
   - Type: Split Out  
   - Configuration: Split the array of stages into individual items.  
   - Connect “get stages for pipeline” output to this node input.  

6. **Create “Filter” Node**  
   - Type: Filter  
   - Configuration: Filter stages where stage ID or name matches the deal’s current stage.  
   - Connect “Split Out” output to this node input.  

7. **Create “If correct stage name” Node**  
   - Type: If  
   - Configuration: Check if the stage name matches the target stage for tracking (e.g., “Won”).  
   - Connect “Filter” output to this node input.  

8. **Create “Get a person” Node**  
   - Type: Pipedrive node  
   - Configuration: Get person using person ID from the deal data.  
   - Connect “If correct stage name” (true output) to this node input.  

9. **Create “get personFields” Node**  
   - Type: HTTP Request  
   - Configuration:  
     - Method: GET  
     - URL: Pipedrive API endpoint to get person custom fields, e.g., `/persons/{personId}/fields`  
     - Authentication: Pipedrive OAuth2 credentials.  
   - Connect “Get a person” output to this node input.  

10. **Create “Get client_id and consent_granted” Node**  
    - Type: Code (JavaScript)  
    - Configuration: Write code to parse personFields and extract `client_id` and `consent_granted`.  
    - Connect “get personFields” output to this node input.  

11. **Create “If client_id exists and consent is given” Node**  
    - Type: If  
    - Configuration: Check if `client_id` is non-empty and `consent_granted` is true.  
    - Connect “Get client_id and consent_granted” output to this node input.  

12. **Create “Construct GA4 object” Node**  
    - Type: Code (JavaScript)  
    - Configuration: Build GA4 Measurement Protocol payload including:  
      - `client_id`  
      - Event name (e.g., “purchase” or “conversion”)  
      - Parameters such as transaction amount, currency, deal ID, user properties, etc.  
    - Connect true output of “If client_id exists and consent is given” to this node input.  

13. **Create “HTTP Request — Send GA4” Node**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: GA4 Measurement Protocol endpoint (https://www.google-analytics.com/mp/collect)  
      - Authentication: Use API secret and measurement ID in query parameters or headers.  
      - Body: JSON, from “Construct GA4 object”.  
    - Connect “Construct GA4 object” output to this node input.  

14. **Test the workflow** by pushing deal updates through Pipedrive and verifying GA4 events received.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow depends on correct configuration of Pipedrive API credentials and OAuth2 authentication.   | Pipedrive OAuth2 setup in n8n credentials.                                                     |
| GA4 Measurement Protocol requires API secret and Measurement ID from Google Analytics property.          | https://developers.google.com/analytics/devguides/collection/protocol/ga4                      |
| The workflow assumes custom person fields for client_id and consent_granted are properly set in Pipedrive. | Ensure these custom fields exist and are consistently populated in Pipedrive.                   |
| Possible improvements include error handling for API limits and retries on HTTP request failures.         | Consider adding error workflows or retries in n8n for robustness.                               |
| Example JavaScript code snippets for extracting client_id and consent_granted can be adapted as needed.   |                                                                                               |

---

This document enables a comprehensive understanding and reproduction of the workflow enabling seamless Pipedrive to GA4 conversion tracking integration.