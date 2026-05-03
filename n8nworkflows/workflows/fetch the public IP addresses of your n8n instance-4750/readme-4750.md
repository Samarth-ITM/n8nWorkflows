fetch the public IP addresses of your n8n instance

https://n8nworkflows.xyz/workflows/fetch-the-public-ip-addresses-of-your-n8n-instance-4750


# fetch the public IP addresses of your n8n instance

### 1. Workflow Overview

This workflow is designed to fetch the public IP address(es) of the n8n instance hosting it. It is intended for scenarios where users or systems need to programmatically determine the external IP addresses used by the n8n server, such as for network diagnostics, firewall configuration, or audit logging.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** A secured webhook receives an external request, requiring header authentication.
- **1.2 Request Repetition:** A node duplicates the process to perform multiple HTTP requests for redundancy or multiple IP retrieval attempts.
- **1.3 IP Retrieval:** HTTP requests are sent to an external API (`https://api.ipify.org`) that returns the public IP address in JSON format.
- **1.4 Aggregation:** The results from multiple HTTP requests are aggregated into a single collection.
- **1.5 Response:** The aggregated unique IP addresses are returned as a JSON array in the webhook response, completing the request.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block exposes a webhook endpoint that listens for incoming requests and authenticates them via a header-based API key.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - *Type:* Trigger node for HTTP webhook requests.  
    - *Configuration:*  
      - Path set to a specific UUID (`4879bc79-d6f8-48df-bfe4-613366c7f399`).  
      - Authentication enabled using header authentication with a pre-configured API key credential.  
      - Response mode set to wait for a response node to send output (deferred response).  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Connected to the Repeat node.  
    - *Version:* 2.  
    - *Potential Failures:*  
      - Unauthorized requests due to missing or invalid API key header.  
      - Incorrect webhook path causing no trigger.  
      - Network issues preventing webhook accessibility.  
    - *Notes:* Requires a header-auth credential with a UUID or arbitrary random string as API key.  

#### 2.2 Request Repetition

- **Overview:**  
  To increase reliability or gather multiple samples, the workflow repeats the HTTP request 10 times.

- **Nodes Involved:**  
  - Repeat

- **Node Details:**  
  - **Repeat**  
    - *Type:* Set node configured to duplicate the incoming item multiple times.  
    - *Configuration:*  
      - Duplicate the incoming webhook item 10 times, effectively creating 10 identical items to trigger multiple HTTP requests.  
    - *Inputs:* From Webhook node.  
    - *Outputs:* Connected to HTTP Request node.  
    - *Version:* 3.4.  
    - *Potential Failures:*  
      - Excessive duplication causing performance degradation.  
      - Unexpected input format causing duplication failure.  

#### 2.3 IP Retrieval

- **Overview:**  
  This block makes HTTP requests to an external service that returns the public IP address in JSON format.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **HTTP Request**  
    - *Type:* HTTP Request node performing GET requests.  
    - *Configuration:*  
      - URL: `https://api.ipify.org`  
      - Query parameter: `format=json` to request JSON output.  
      - No additional headers or authentication.  
      - The node is set to always output data, ensuring consistent downstream processing.  
    - *Inputs:* Receives 10 duplicated items from Repeat node.  
    - *Outputs:* Connected to Aggregate node.  
    - *Version:* 4.2.  
    - *Potential Failures:*  
      - Network connectivity issues to `api.ipify.org`.  
      - API rate limiting or downtime.  
      - Unexpected response format if API changes.  

#### 2.4 Aggregation

- **Overview:**  
  Aggregates all HTTP request results into a single item to simplify response handling.

- **Nodes Involved:**  
  - Aggregate

- **Node Details:**  
  - **Aggregate**  
    - *Type:* Aggregate node used to combine multiple items into one.  
    - *Configuration:*  
      - Aggregates all item data into one collection, retaining all IP responses.  
    - *Inputs:* Multiple HTTP Request outputs (10 items).  
    - *Outputs:* Connected to Respond to Webhook node.  
    - *Version:* 1.  
    - *Potential Failures:*  
      - Large aggregation causing performance issues (unlikely here due to small data size).  

#### 2.5 Response

- **Overview:**  
  This node returns the unique set of public IP addresses as a JSON array, completing the webhook request.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  
  - **Respond to Webhook**  
    - *Type:* Response node to send data back to the webhook caller and end the workflow execution.  
    - *Configuration:*  
      - Responds with plain text content type.  
      - Response body is an expression that extracts the `ip` field from all aggregated items, removes duplicates, and converts the result to a JSON string.  
      - Expression used: `={{ $json.data.pluck('ip').unique().toJsonString() }}`  
    - *Inputs:* Aggregated data from Aggregate node.  
    - *Outputs:* None (terminates workflow).  
    - *Version:* 1.2.  
    - *Potential Failures:*  
      - Expression evaluation errors if data structure changes.  
      - Failure to respond in time if workflow latency grows.  

#### 2.6 Documentation Block

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - *Type:* Documentation node.  
    - *Content Summary:*  
      - Explains the workflow’s purpose: fetching the public IP addresses of the hosting n8n instance.  
      - Notes the prerequisite: a header-auth credential with a UUID or arbitrary random string.  
      - Provides an example curl command to invoke the webhook with API key header and shows example output.  
    - *Position:* Positioned visually separate to explain the workflow.  
    - *No inputs or outputs.*  

---

### 3. Summary Table

| Node Name          | Node Type               | Functional Role                      | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                   |
|--------------------|-------------------------|------------------------------------|------------------------|------------------------|-----------------------------------------------------------------------------------------------|
| Webhook            | Webhook                 | Entry point with header authentication | None                   | Repeat                 | Simple webhook with header-auth                                                              |
| Repeat             | Set                     | Duplicate incoming webhook item 10 times | Webhook                | HTTP Request           | Repeat 10 times                                                                              |
| HTTP Request       | HTTP Request            | Fetch public IP from external API  | Repeat                 | Aggregate              | Request public IP address information as json                                                |
| Aggregate          | Aggregate               | Combine all HTTP request results into one | HTTP Request           | Respond to Webhook     |                                                                                               |
| Respond to Webhook | Respond to Webhook      | Return unique IP addresses as JSON | Aggregate              | None                   | Return an array from the workflow and end the webhook invocation                             |
| Sticky Note        | Sticky Note             | Documentation                      | None                   | None                   | ## to fetch the public IP address(es) for the hosting n8n instance... example invocation included |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Set node type to **Webhook**.  
   - Configure the webhook path to a unique UUID string (e.g., `4879bc79-d6f8-48df-bfe4-613366c7f399`).  
   - Enable **Header Authentication** and link it to a pre-created header-auth credential containing a UUID or random string API key.  
   - Set **Response Mode** to `Response Node` (await response from downstream).  
   - Save.

2. **Create a Set Node for Repetition**  
   - Add a **Set** node named `Repeat`.  
   - Turn on **Duplicate Item** option.  
   - Set **Duplicate Count** to 10.  
   - Connect the output of the Webhook node to this Repeat node.  
   - Save.

3. **Create an HTTP Request Node**  
   - Add an **HTTP Request** node named `HTTP Request`.  
   - Set method to `GET`.  
   - Enter URL: `https://api.ipify.org`.  
   - Add query parameter: name=`format`, value=`json`.  
   - Ensure no authentication is set.  
   - Enable **Always Output Data** to true.  
   - Connect the output of the Repeat node to this HTTP Request node.  
   - Save.

4. **Add an Aggregate Node**  
   - Add an **Aggregate** node named `Aggregate`.  
   - Configure aggregation mode to aggregate all item data (default).  
   - Connect the output of the HTTP Request node to Aggregate node.  
   - Save.

5. **Add a Respond to Webhook Node**  
   - Add a **Respond to Webhook** node named `Respond to Webhook`.  
   - Set **Respond With** to `Text`.  
   - In the **Response Body** parameter, enter the expression:  
     `={{ $json.data.pluck('ip').unique().toJsonString() }}`  
   - Connect the output of the Aggregate node to this node.  
   - Save.

6. **(Optional) Add a Sticky Note**  
   - Add a **Sticky Note** node to document the workflow.  
   - Include notes about prerequisites (header-auth credential), purpose, and example curl usage:  
     ```
     ## to fetch the public IP address(es) for the hosting n8n instance

     * prerequisite: a header-auth credential with a uuid or an arbitrary random string.

     * example invocation
     $ curl -H "api-key: super-long-api-token" http://localhost:5678/webhook-test/4879bc79-d6f8-48df-bfe4-613366c7f399
     ["88.88.88.66", "88.88.88.88"]
     ```  
   - Position it suitably in the canvas for clarity.

7. **Credentials Setup**  
   - Create a **Header Auth** credential in n8n with a secure API key string.  
   - Attach this credential to the Webhook node’s authentication settings.

8. **Activate and Test**  
   - Activate the workflow.  
   - Test by sending a GET request to the webhook URL with the header `api-key` set to the configured API key.  
   - Confirm the response returns a JSON array of one or more IP addresses.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                       |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------|
| The workflow requires a header-auth credential with a UUID or arbitrary random string as API key.   | Credential setup needed for Webhook node authentication. |
| Example curl command is provided in the sticky note for quick testing.                               | `$ curl -H "api-key: super-long-api-token" http://localhost:5678/webhook-test/4879bc79-d6f8-48df-bfe4-613366c7f399` |
| The external API used is `https://api.ipify.org` which is a free, simple service to get public IP. | https://www.ipify.org/                                |

---

_Disclaimer: The text provided is generated exclusively from an automated n8n workflow. It respects all current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public._