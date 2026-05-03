Generate Post-Event Reports with GPT-4, Email Delivery & Database Storage

https://n8nworkflows.xyz/workflows/generate-post-event-reports-with-gpt-4--email-delivery---database-storage-10281


# Generate Post-Event Reports with GPT-4, Email Delivery & Database Storage

### 1. Workflow Overview

This workflow automates the generation of post-event reports using AI (GPT-4), followed by emailing the report to stakeholders and saving the report data into a PostgreSQL database. It is designed for event organizers or marketing teams who want to streamline their reporting process by automatically collecting event data, summarizing it with AI, and distributing the results efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receive event-related data via HTTP POST webhook.
- **1.2 Data Collection:** Fetch attendees and engagement metrics from external APIs.
- **1.3 Data Processing:** Process and aggregate collected metrics for AI consumption.
- **1.4 AI Report Generation:** Use GPT-4-powered AI to generate a professional event report.
- **1.5 Report Distribution and Storage:** Email the generated report and save it to a PostgreSQL database.
- **1.6 Response to Trigger:** Respond back to the original webhook confirming success.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by listening for incoming HTTP POST requests containing event data. It acts as the entry point, triggering the entire process upon receiving the request.

- **Nodes Involved:**  
  - Webhook Trigger  
  - Sticky Note ("Start via HTTP POST request")

- **Node Details:**  

  - **Webhook Trigger**  
    - *Type & Role:* HTTP Webhook node, listens for incoming POST requests at `/event-report` path.  
    - *Configuration:*  
      - HTTP Method: POST  
      - Response Mode: Respond via a dedicated response node later in the workflow.  
    - *Inputs/Outputs:* No input connections; outputs to "Get Attendees" and "Get Engagement Metrics".  
    - *Edge Cases:* Potential issues include unauthorized or malformed requests; no authentication is configured here.  
    - *Version:* 1  

  - **Sticky Note**  
    - *Purpose:* Visual annotation indicating the start of the workflow via HTTP POST.

#### 2.2 Data Collection

- **Overview:**  
  This block gathers raw data necessary for the report by calling two external APIs to retrieve event attendees and engagement metrics.

- **Nodes Involved:**  
  - Get Attendees  
  - Get Engagement Metrics  
  - Sticky Note ("Fetches attendees & engagement metrics")

- **Node Details:**  

  - **Get Attendees**  
    - *Type & Role:* HTTP Request node to fetch attendee data.  
    - *Configuration:*  
      - URL: `https://oneclick.com/GetAttendees`  
      - Defaults: No authentication or additional headers configured.  
    - *Inputs:* Triggered by "Webhook Trigger".  
    - *Outputs:* Sends data to "Process Metrics".  
    - *Edge Cases:* API downtime, slow response, or unexpected response format may cause failures.  
    - *Version:* 4.1  

  - **Get Engagement Metrics**  
    - *Type & Role:* HTTP Request node to fetch engagement metrics.  
    - *Configuration:*  
      - URL: `https://oneclick.com/GetEngagement`  
      - No special authentication configured.  
    - *Inputs:* Triggered parallelly by "Webhook Trigger".  
    - *Outputs:* Sends data to "Process Metrics".  
    - *Edge Cases:* Same as above (API errors, timeouts, invalid data).  
    - *Version:* 4.1  

  - **Sticky Note**  
    - *Purpose:* Explains this block fetches attendees and engagement data.

#### 2.3 Data Processing

- **Overview:**  
  Combines and processes the raw data from both APIs to prepare a structured dataset for AI report generation.

- **Nodes Involved:**  
  - Process Metrics  
  - AI Agent  
  - Sticky Note ("Uses GPT-4 to generate professional reports")

- **Node Details:**  

  - **Process Metrics**  
    - *Type & Role:* Code node (JavaScript) to merge and transform metrics data.  
    - *Configuration:* No explicit code shown in JSON, but typically includes data normalization, aggregation, or enrichment.  
    - *Inputs:* Receives data from both "Get Attendees" and "Get Engagement Metrics".  
    - *Outputs:* Sends processed data to "AI Agent".  
    - *Edge Cases:* Expression errors if expected fields are missing, or data format mismatches.  
    - *Version:* 2  

  - **AI Agent**  
    - *Type & Role:* Langchain AI Agent node orchestrating AI-driven operations.  
    - *Configuration:* Default options, acts as a dispatcher to the AI language model node.  
    - *Inputs:* Receives processed metrics from "Process Metrics".  
    - *Outputs:* Sends outputs to "Send Report Email" and "Save to Database".  
    - *Edge Cases:* AI service availability, rate limits, malformed prompts.  
    - *Version:* 2.2  

  - **Sticky Note**  
    - *Purpose:* Highlights AI usage (GPT-4) for professional report generation.

#### 2.4 AI Report Generation

- **Overview:**  
  This block leverages GPT-4 to generate a textual post-event report based on structured input data.

- **Nodes Involved:**  
  - AI Generate Report  

- **Node Details:**  

  - **AI Generate Report**  
    - *Type & Role:* Langchain node using OpenAI GPT-4 for text generation.  
    - *Configuration:*  
      - Model: `gpt-4o-mini` (a GPT-4 variant)  
      - Max Tokens: 2000  
      - Temperature: 0.7 (balances creativity and determinism)  
    - *Credentials:* Uses stored OpenAI API credentials.  
    - *Inputs:* Connected to "AI Agent" node's language model input.  
    - *Outputs:* Sends generated report back to "AI Agent".  
    - *Edge Cases:* API quota exceeded, timeout, or generation errors.  
    - *Version:* 1  

#### 2.5 Report Distribution and Storage

- **Overview:**  
  This block sends the generated report via email to stakeholders and saves the report data to a PostgreSQL database for archival.

- **Nodes Involved:**  
  - Send Report Email  
  - Save to Database  
  - Sticky Note ("Sends report to stakeholders\n\nSaves reports to PostgreSQL")

- **Node Details:**  

  - **Send Report Email**  
    - *Type & Role:* Email Send node to distribute the report.  
    - *Configuration:*  
      - Subject: Dynamic, e.g., `Event Report: {{ $('Process Metrics').first().json.eventName }}`  
      - To: `xyz@gmail.com` (placeholder)  
      - From: `abc@gmail.com` (placeholder)  
      - SMTP Credentials: Configured with SMTP test credentials.  
    - *Inputs:* Receives from "AI Agent".  
    - *Outputs:* Connects to "Send Response".  
    - *Edge Cases:* SMTP authentication failure, email delivery issues, invalid email addresses.  
    - *Version:* 2  

  - **Save to Database**  
    - *Type & Role:* PostgreSQL node to save report data.  
    - *Configuration:*  
      - Schema: `public`  
      - Table: dynamically set (exact table name unspecified)  
      - Column mapping: automatic from input data  
      - Credentials: PostgreSQL test credentials configured  
    - *Inputs:* Receives from "AI Agent".  
    - *Outputs:* Connects to "Send Response".  
    - *Edge Cases:* DB connection failure, schema/table mismatch, data type conflicts.  
    - *Version:* 2.4  

  - **Sticky Note**  
    - *Purpose:* Explains the email and DB storage roles.

#### 2.6 Response to Trigger

- **Overview:**  
  Finalizes the workflow by responding to the initial webhook request confirming the report generation status.

- **Nodes Involved:**  
  - Send Response

- **Node Details:**  

  - **Send Response**  
    - *Type & Role:* Respond to Webhook node, sends JSON response back to the HTTP client.  
    - *Configuration:*  
      - Respond With: JSON  
      - Response Body:  
        ```json
        {
          "success": true,
          "message": "Report generated successfully",
          "eventId": "{{ $('Process Metrics').first().json.eventId }}",
          "reportSent": true
        }
        ```  
    - *Inputs:* Receives from both "Send Report Email" and "Save to Database".  
    - *Outputs:* None (terminal node).  
    - *Edge Cases:* None critical, but if upstream nodes fail, response may not be sent or may be inaccurate.  
    - *Version:* 1  

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role                          | Input Node(s)               | Output Node(s)                  | Sticky Note                                                        |
|--------------------|----------------------------------|----------------------------------------|-----------------------------|-------------------------------|-------------------------------------------------------------------|
| Webhook Trigger    | HTTP Webhook                     | Entry point, receives POST requests    |                             | Get Attendees, Get Engagement Metrics | Start via HTTP POST request                                       |
| Get Attendees      | HTTP Request                    | Fetch event attendees                   | Webhook Trigger             | Process Metrics                | Fetches attendees & engagement metrics                            |
| Get Engagement Metrics | HTTP Request                    | Fetch engagement data                   | Webhook Trigger             | Process Metrics                | Fetches attendees & engagement metrics                            |
| Process Metrics    | Code                            | Process and merge fetched data          | Get Attendees, Get Engagement Metrics | AI Agent                      | Uses GPT-4 to generate professional reports                      |
| AI Agent           | Langchain Agent                 | Orchestrates AI interaction             | Process Metrics             | Send Report Email, Save to Database | Uses GPT-4 to generate professional reports                      |
| AI Generate Report | Langchain LM Chat OpenAI        | Generates AI report text                 | AI Agent (ai_languageModel) | AI Agent                      |                                                                   |
| Send Report Email  | Email Send                      | Email report to stakeholders             | AI Agent                   | Send Response                 | Sends report to stakeholders; Saves reports to PostgreSQL        |
| Save to Database   | PostgreSQL                      | Save report data for archival            | AI Agent                   | Send Response                 | Sends report to stakeholders; Saves reports to PostgreSQL        |
| Send Response      | Respond to Webhook              | Responds to initial HTTP request        | Send Report Email, Save to Database |                               |                                                                   |
| Sticky Note        | Sticky Note                    | Annotation                              |                             |                               | Start via HTTP POST request                                       |
| Sticky Note1       | Sticky Note                    | Annotation                              |                             |                               | Fetches attendees & engagement metrics                            |
| Sticky Note5       | Sticky Note                    | Annotation                              |                             |                               | Sends report to stakeholders; Saves reports to PostgreSQL        |
| Sticky Note7       | Sticky Note                    | Annotation                              |                             |                               | Uses GPT-4 to generate professional reports                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: HTTP Webhook  
   - Path: `event-report`  
   - HTTP Method: POST  
   - Response Mode: Respond via a dedicated Respond to Webhook node later  
   - No authentication configured

2. **Create Get Attendees Node**  
   - Type: HTTP Request  
   - URL: `https://oneclick.com/GetAttendees`  
   - Method: GET (default)  
   - No authentication or headers  
   - Connect Webhook Trigger main output to this node's main input

3. **Create Get Engagement Metrics Node**  
   - Type: HTTP Request  
   - URL: `https://oneclick.com/GetEngagement`  
   - Method: GET (default)  
   - Connect Webhook Trigger main output to this node's main input, running in parallel with "Get Attendees"

4. **Create Process Metrics Node**  
   - Type: Code (JavaScript)  
   - Connect both "Get Attendees" and "Get Engagement Metrics" main outputs to this node's single input.  
   - Code should merge and process the JSON data from both requests into a unified structured object for reporting. (Custom logic needed depending on data structure.)

5. **Create AI Agent Node**  
   - Type: Langchain Agent  
   - Default options  
   - Connect "Process Metrics" output to "AI Agent" input

6. **Create AI Generate Report Node**  
   - Type: Langchain LM Chat OpenAI  
   - Model: `gpt-4o-mini`  
   - Max Tokens: 2000  
   - Temperature: 0.7  
   - Credentials: Configure with valid OpenAI API credentials  
   - Connect "AI Agent" node’s `ai_languageModel` output to this node's input  
   - Connect this node output back to "AI Agent" node's language model input

7. **Create Send Report Email Node**  
   - Type: Email Send  
   - Subject: `Event Report: {{ $('Process Metrics').first().json.eventName }}` (dynamic)  
   - To Email: `xyz@gmail.com` (replace with real recipient)  
   - From Email: `abc@gmail.com` (replace with authorized sender)  
   - Credentials: Configure SMTP credentials for sending email  
   - Connect "AI Agent" output to this node’s input

8. **Create Save to Database Node**  
   - Type: PostgreSQL  
   - Schema: `public`  
   - Table: Set to target table for reports (e.g., `event_reports`)  
   - Columns: Use auto mapping from input data (ensure fields match table schema)  
   - Credentials: Configure PostgreSQL connection credentials  
   - Connect "AI Agent" output to this node’s input

9. **Create Send Response Node**  
   - Type: Respond to Webhook  
   - Respond With: JSON  
   - Response Body:
     ```json
     {
       "success": true,
       "message": "Report generated successfully",
       "eventId": "{{ $('Process Metrics').first().json.eventId }}",
       "reportSent": true
     }
     ```  
   - Connect outputs from both "Send Report Email" and "Save to Database" nodes to this node’s input

10. **Add Sticky Notes for Clarity** (Optional but recommended)  
    - Near Webhook Trigger: "Start via HTTP POST request"  
    - Near Get Attendees & Engagement nodes: "Fetches attendees & engagement metrics"  
    - Near Process Metrics & AI nodes: "Uses GPT-4 to generate professional reports"  
    - Near Send Report Email & Save to Database nodes: "Sends report to stakeholders\n\nSaves reports to PostgreSQL"

---

### 5. General Notes & Resources

| Note Content                                                              | Context or Link                                |
|---------------------------------------------------------------------------|------------------------------------------------|
| The workflow requires valid API endpoints at `https://oneclick.com` for attendees and engagement metrics. Replace with real endpoints as needed. | External API integration                        |
| OpenAI GPT-4 credentials must be properly set up in n8n for AI nodes to function. | OpenAI API credential setup                     |
| SMTP credentials should be configured with a valid email provider for sending emails. | Email delivery setup                            |
| PostgreSQL database schema and table must exist and match the expected data fields to avoid insertion errors. | Database setup                                  |
| Sticky notes serve as in-editor documentation to improve maintainability and team understanding. | Workflow documentation best practice            |

---

**Disclaimer:**  
The text provided here is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.