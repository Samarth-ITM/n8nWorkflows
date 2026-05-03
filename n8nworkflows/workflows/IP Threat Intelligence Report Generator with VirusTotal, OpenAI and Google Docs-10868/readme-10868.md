IP Threat Intelligence Report Generator with VirusTotal, OpenAI and Google Docs

https://n8nworkflows.xyz/workflows/ip-threat-intelligence-report-generator-with-virustotal--openai-and-google-docs-10868


# IP Threat Intelligence Report Generator with VirusTotal, OpenAI and Google Docs

### 1. Workflow Overview

This workflow automates the generation of an IP Threat Intelligence report by integrating data from VirusTotal, geolocation services, and OpenAI’s language models, culminating in a formatted report updated in Google Docs. It is designed for cybersecurity analysts or threat intelligence teams who want to quickly gather, analyze, and document threat information related to an IP address.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the IP address input via a web form.
- **1.2 Data Enrichment:** Queries VirusTotal and geolocation services in parallel to gather intelligence related to the IP address.
- **1.3 Data Aggregation:** Merges the responses from VirusTotal and geolocation queries.
- **1.4 AI Analysis:** Uses an AI agent with OpenAI’s language model to analyze and summarize the merged intelligence.
- **1.5 Report Update:** Updates a Google Docs document with the AI-generated threat intelligence report.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Receives the IP address input from the user through a web form trigger, initiating the workflow.

- **Nodes Involved:**  
  - Landing Page Url  
  - Set IP Address1

- **Node Details:**  

  - **Landing Page Url**  
    - *Type & Role:* Form Trigger node; serves as the entry point for user input.  
    - *Configuration:* Configured with a webhook to receive HTTP requests containing the IP address. The webhook ID is empty, indicating a default or dynamically assigned webhook.  
    - *Expressions/Variables:* Captures the IP address submitted by the user (expected to be passed in form data).  
    - *Connections:* Output connected to "Set IP Address1".  
    - *Failure Cases:* Missing or malformed IP input; webhook not reachable.  
    - *Version:* 2.2  

  - **Set IP Address1**  
    - *Type & Role:* Set node; prepares and sets the IP address variable for downstream nodes.  
    - *Configuration:* Likely extracts and sets the IP address from the incoming webhook data into a workflow variable or JSON property for further processing.  
    - *Expressions/Variables:* Uses expressions to map form input to a named field (e.g., `ipAddress`).  
    - *Connections:* Outputs to "Query Geolocation1" and "VirusTotal HTTP Request1" in parallel.  
    - *Failure Cases:* Empty or invalid IP address set here will propagate errors downstream.  
    - *Version:* 1

#### 1.2 Data Enrichment

- **Overview:**  
  Queries external services to gather intelligence about the IP address: VirusTotal for threat data and a geolocation API for location data.

- **Nodes Involved:**  
  - Query Geolocation1  
  - VirusTotal HTTP Request1

- **Node Details:**  

  - **Query Geolocation1**  
    - *Type & Role:* HTTP Request node; calls a geolocation API to fetch location information for the IP address.  
    - *Configuration:* Configured with a GET request to a geolocation service endpoint, passing the IP address as a parameter. Likely includes authentication or API key in headers or query parameters.  
    - *Expressions/Variables:* Uses the IP address variable from "Set IP Address1".  
    - *Connections:* Output connected to "Merge Results1".  
    - *Failure Cases:* API rate limiting, invalid API key, network timeouts, invalid IP format causing API errors.  
    - *Version:* 3  

  - **VirusTotal HTTP Request1**  
    - *Type & Role:* HTTP Request node; queries VirusTotal's IP report API to obtain threat intelligence related to the IP.  
    - *Configuration:* Uses the VirusTotal API credentials (authenticated via the "virusTotalApi" credential). Sends a GET request to VirusTotal’s IP endpoint with the IP address.  
    - *Expressions/Variables:* Uses IP address from "Set IP Address1".  
    - *Connections:* Output connected to "Merge Results1".  
    - *Failure Cases:* API quota exceeded, invalid API key, network errors, IP not found in VirusTotal database.  
    - *Version:* 4.3  
    - *Credentials:* Requires VirusTotal API credential configured in n8n.

#### 1.3 Data Aggregation

- **Overview:**  
  Combines results from the geolocation and VirusTotal queries into a single data set for AI analysis.

- **Nodes Involved:**  
  - Merge Results1

- **Node Details:**  

  - **Merge Results1**  
    - *Type & Role:* Merge node; combines multiple incoming data streams.  
    - *Configuration:* Set to merge data from both "Query Geolocation1" and "VirusTotal HTTP Request1" nodes. The parameter `alwaysOutputData` is true, ensuring output even if one input is empty.  
    - *Expressions/Variables:* Merges IP-related data objects into a unified JSON for AI processing.  
    - *Connections:* Output connected to "AI Agent1".  
    - *Failure Cases:* If one API fails or returns empty, the merge still outputs data but with incomplete info.  
    - *Version:* 1  

#### 1.4 AI Analysis

- **Overview:**  
  Invokes an AI agent powered by OpenAI’s chat model to analyze the merged data and generate a human-readable threat intelligence report.

- **Nodes Involved:**  
  - AI Agent1  
  - OpenAI Chat Model1

- **Node Details:**  

  - **OpenAI Chat Model1**  
    - *Type & Role:* Language model node; interfaces with OpenAI’s chat completion API.  
    - *Configuration:* Configured with OpenAI credentials and model parameters (e.g., model name, temperature). Used as the language model backend for the AI Agent.  
    - *Expressions/Variables:* Receives prompt data constructed from merged intelligence.  
    - *Connections:* Output connected to "AI Agent1" through the `ai_languageModel` port.  
    - *Failure Cases:* API rate limits, authentication errors, malformed prompt causing API errors, network issues.  
    - *Version:* 1.2  
    - *Credentials:* Requires OpenAI API credentials.  

  - **AI Agent1**  
    - *Type & Role:* Langchain agent node; orchestrates the interaction with the language model to generate the threat report.  
    - *Configuration:* Uses OpenAI Chat Model1 as the AI backend. Processes merged input data to produce a summarized report.  
    - *Expressions/Variables:* Takes merged IP data as input, outputs formatted AI-generated text.  
    - *Connections:* Output connected to "Update a document1".  
    - *Failure Cases:* Language model API errors, invalid input format, timeout.  
    - *Version:* 3  

#### 1.5 Report Update

- **Overview:**  
  Updates a Google Docs document with the AI-generated threat intelligence report content.

- **Nodes Involved:**  
  - Update a document1

- **Node Details:**  

  - **Update a document1**  
    - *Type & Role:* Google Docs node; modifies an existing document by inserting or updating content.  
    - *Configuration:* Configured with Google OAuth2 credentials; target document ID specified. Inserts AI agent output into the document, possibly replacing or appending to a placeholder or template section.  
    - *Expressions/Variables:* Uses AI-generated report text from "AI Agent1".  
    - *Connections:* Final node in the workflow.  
    - *Failure Cases:* OAuth token expiration, permission denied, document not found, API quota exceeded.  
    - *Version:* 2  
    - *Credentials:* Requires Google Docs OAuth2 credentials.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role               | Input Node(s)          | Output Node(s)        | Sticky Note                         |
|-------------------------|----------------------------------|------------------------------|-----------------------|-----------------------|-----------------------------------|
| Landing Page Url         | Form Trigger                     | Receive IP input             | —                     | Set IP Address1       |                                   |
| Set IP Address1          | Set                             | Prepare IP variable           | Landing Page Url       | Query Geolocation1, VirusTotal HTTP Request1 |                                   |
| Query Geolocation1       | HTTP Request                    | Fetch IP geolocation data     | Set IP Address1        | Merge Results1        |                                   |
| VirusTotal HTTP Request1 | HTTP Request                    | Fetch VirusTotal threat data  | Set IP Address1        | Merge Results1        |                                   |
| Merge Results1           | Merge                           | Combine API responses         | Query Geolocation1, VirusTotal HTTP Request1 | AI Agent1             |                                   |
| OpenAI Chat Model1       | Langchain LM Chat OpenAI        | Language model backend        | —                     | AI Agent1 (ai_languageModel) |                                   |
| AI Agent1               | Langchain Agent                  | Generate threat report        | Merge Results1, OpenAI Chat Model1 (ai_languageModel) | Update a document1     |                                   |
| Update a document1       | Google Docs                     | Update report document        | AI Agent1              | —                     |                                   |
| Sticky Note              | Sticky Note                     | Comment/Annotation            | —                     | —                     |                                   |
| Sticky Note6             | Sticky Note                     | Comment/Annotation            | —                     | —                     |                                   |
| Sticky Note7             | Sticky Note                     | Comment/Annotation            | —                     | —                     |                                   |
| Sticky Note15            | Sticky Note                     | Comment/Annotation            | —                     | —                     |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node (Landing Page Url):**  
   - Type: Form Trigger  
   - Configure webhook to accept IP address input (e.g., field named `ipAddress`).  
   - Save and activate webhook.

2. **Create a Set node (Set IP Address1):**  
   - Type: Set  
   - Map the incoming form field `ipAddress` to a new workflow variable or JSON field (e.g., `ipAddress`).  
   - Connect output from "Landing Page Url" to this node.

3. **Create an HTTP Request node (Query Geolocation1):**  
   - Type: HTTP Request  
   - Configure GET request to a geolocation API endpoint (e.g., ipinfo.io, ip-api.com).  
   - Parameterize the IP address using expression referencing `ipAddress` from previous node.  
   - Include necessary headers or API keys.  
   - Connect output from "Set IP Address1" to this node.

4. **Create an HTTP Request node (VirusTotal HTTP Request1):**  
   - Type: HTTP Request  
   - Configure GET request to VirusTotal’s IP report endpoint (`https://www.virustotal.com/api/v3/ip_addresses/{{ipAddress}}`).  
   - Use VirusTotal API credentials configured in n8n.  
   - Parameterize IP address similarly.  
   - Connect output from "Set IP Address1" to this node.

5. **Create a Merge node (Merge Results1):**  
   - Type: Merge  
   - Set mode to combine inputs from both "Query Geolocation1" and "VirusTotal HTTP Request1".  
   - Enable `alwaysOutputData` to ensure output even if one input is missing.  
   - Connect outputs from both HTTP Request nodes to this node.

6. **Create an OpenAI Chat Model node (OpenAI Chat Model1):**  
   - Type: Langchain LM Chat OpenAI  
   - Set OpenAI API credentials.  
   - Configure model parameters (model name, temperature, max tokens).  
   - No direct input connection; this node serves as an AI backend.

7. **Create an AI Agent node (AI Agent1):**  
   - Type: Langchain Agent  
   - Configure to use "OpenAI Chat Model1" as the language model backend (connect via `ai_languageModel` port).  
   - Input data comes from "Merge Results1".  
   - Configure prompt template or logic for summarizing IP threat intelligence.  
   - Connect output to next node.

8. **Create a Google Docs node (Update a document1):**  
   - Type: Google Docs  
   - Set Google OAuth2 credentials.  
   - Specify the document ID to update.  
   - Configure the node to insert or update document content with AI Agent output.  
   - Connect input from "AI Agent1".

9. **(Optional) Create sticky notes for documentation:**  
   - Add Sticky Note nodes near logical groups for annotations or reminders.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                     |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------|
| VirusTotal API requires registration and API key; quota limits may apply                      | https://www.virustotal.com/gui/join-us             |
| OpenAI API usage requires careful cost management and API key setup                           | https://platform.openai.com/account/api-keys        |
| Google Docs API requires OAuth2 credentials with appropriate scopes (drive.file, docs)        | https://developers.google.com/docs/api/quickstart   |
| Consider IP input validation at the form level or in "Set IP Address1" node for robustness    | IP format regex validation or pre-check             |
| Handle possible API failures gracefully to avoid workflow halts                              | Implement error workflows or fallback nodes          |

---

**Disclaimer:** The provided text is generated from an automated n8n workflow. It complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.