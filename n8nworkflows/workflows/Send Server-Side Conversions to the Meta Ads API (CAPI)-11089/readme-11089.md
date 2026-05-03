Send Server-Side Conversions to the Meta Ads API (CAPI)

https://n8nworkflows.xyz/workflows/send-server-side-conversions-to-the-meta-ads-api--capi--11089


# Send Server-Side Conversions to the Meta Ads API (CAPI)

---

### 1. Workflow Overview

This workflow implements a server-side conversion tracking solution for Meta Ads using the Meta Conversions API (CAPI). It enables sending user conversion events directly from your backend or integration endpoint to Meta’s servers, bypassing browser limitations like ad blockers and iOS privacy restrictions.  

The workflow is structured into four main logical blocks:

**1.1 Input Reception**  
- Receives incoming conversion data via a Webhook node exposed as a REST endpoint.

**1.2 Data Normalization and Hashing**  
- Cleans and standardizes Personally Identifiable Information (PII) such as email, phone number, first name, and last name.  
- Hashes these PII fields securely using SHA-256, as required by Meta’s CAPI specifications.

**1.3 Payload Assembly**  
- Computes event timestamps and consolidates all data including hashed PII, event metadata, and browser-related identifiers (`fbc`, `fbp`, IP address, user agent) into the exact JSON structure expected by Meta.

**1.4 Sending to Meta’s API and Responding**  
- Sends the prepared event payload to the Meta Ads API endpoint using an authenticated HTTP request.  
- Returns an appropriate HTTP response to the original sender confirming successful processing.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block captures the incoming POST requests with conversion data from external sources such as website forms or backend systems. It acts as the workflow’s entry point.

**Nodes Involved:**  
- Webhook  
- Sticky Note1 (documentation aid)

**Node Details:**

- **Webhook**  
  - Type: `n8n-nodes-base.webhook` (Trigger node)  
  - Configuration: Listens on path `meta-conversion-api` for HTTP POST requests. Response mode set to `responseNode` to allow response customization.  
  - Key Variables: Incoming JSON body with fields `email`, `phone`, `firstName`, `lastName`, `fbc`, `fbp`. Also reads HTTP headers for IP and user-agent.  
  - Input: External HTTP POST request  
  - Output: JSON payload forwarded to next node  
  - Edge Cases: Missing required fields, malformed JSON, network failures, or invalid HTTP methods. Should be handled upstream or by monitoring.  
  - Sticky Note1 attached: Explains how to use this node as the endpoint and the required JSON structure.

---

#### 2.2 Data Normalization and Hashing

**Overview:**  
This block cleans and normalizes PII by trimming whitespace, lowercasing strings, and removing non-numeric characters from phone numbers. Then, it securely hashes these fields with SHA-256 to comply with Meta’s privacy and security requirements.

**Nodes Involved:**  
- Edit Normalize PII (Set node)  
- Crypto - Hash Email (Crypto node)  
- Crypto - Hash Phone (Crypto node)  
- Crypto - First Name (Crypto node)  
- Crypto - Last Name (Crypto node)  
- Sticky Note2 (documentation aid)

**Node Details:**

- **Edit Normalize PII**  
  - Type: `n8n-nodes-base.set`  
  - Role: Applies text normalization on incoming fields:  
    - `normalized_email` = trimmed, lowercased email  
    - `normalized_phone` = phone number stripped of non-digits  
    - `firstName`, `lastName` = trimmed, lowercased first and last names  
  - Input: Raw JSON from Webhook  
  - Output: JSON with normalized fields added  
  - Edge Cases: Missing input fields, unexpected data types. May cause expression failures if fields are undefined.

- **Crypto - Hash Email**  
  - Type: `n8n-nodes-base.crypto`  
  - Role: Hashes normalized email using SHA-256, outputs `hashed_em`  
  - Input: Receives normalized_email from previous node  
  - Output: JSON with hashed email  

- **Crypto - Hash Phone**  
  - Type: `n8n-nodes-base.crypto`  
  - Role: Hashes normalized phone number using SHA-256, outputs `hashed_ph`  
  - Input: normalized_phone from previous node  

- **Crypto - First Name**  
  - Type: `n8n-nodes-base.crypto`  
  - Role: Hashes normalized firstName using SHA-256, outputs `hashed_firstName`  

- **Crypto - Last Name**  
  - Type: `n8n-nodes-base.crypto`  
  - Role: Hashes normalized lastName using SHA-256, outputs `hashed_lastName`  

- **Sticky Note2** attached to this block explains that this hashing step is automatic, secure, and requires no manual configuration.

- Edge Cases: Hashing failures (unlikely), missing or malformed inputs causing invalid hashes.

---

#### 2.3 Payload Assembly

**Overview:**  
This block generates the final structured payload reflecting Meta’s required format. It adds event metadata such as timestamp and event name, maps hashed PII and browser identifiers, and prepares the JSON for API submission.

**Nodes Involved:**  
- Set - Compute Timestamps & Map Fields (Set node)  
- Preparing for HTTP Request Payload (Code node)  
- Sticky Note (large documentation note)  

**Node Details:**

- **Set - Compute Timestamps & Map Fields**  
  - Type: `n8n-nodes-base.set`  
  - Role:  
    - Computes `conversion_timestamp` as current UNIX timestamp.  
    - Defines static `event_name` as `SubmitApplication`.  
    - Maps hashed PII fields (`hashed_em`, `hashed_ph`, `hashed_firstName`, `hashed_lastName`) from previous Crypto nodes.  
    - Retrieves Facebook browser cookie identifiers (`fbc`, `fbp`), user IP, and user agent from Webhook request headers.  
  - Input: Hashed PII nodes and Webhook data  
  - Output: JSON with all mapped fields for payload  
  - Edge Cases: Missing headers (IP/User-Agent), timestamp computation issues (unlikely).

- **Preparing for HTTP Request Payload**  
  - Type: `n8n-nodes-base.code` (JavaScript)  
  - Role: Constructs the exact payload object expected by Meta CAPI:  
    - `event_name`, `event_time`, `action_source` set to “website”  
    - `user_data` object includes all hashed PII and identifiers  
  - Inputs: JSON from previous Set node  
  - Outputs: JSON formatted as Meta CAPI expects under `data` array  
  - Edge Cases: Incorrect field references, code errors, missing data fields causing invalid request payload.

- **Sticky Note** (large, general note) explains the overall workflow and usage, including the YouTube setup video link.

---

#### 2.4 Sending to Meta’s API and Responding

**Overview:**  
Sends the assembled payload to the Meta Conversions API using an authenticated HTTP POST request. Finally, it responds back to the original webhook caller with a success status and any API response details.

**Nodes Involved:**  
- Sending Events To Facebook Pixel (HTTP Request node)  
- Respond to Webhook (Respond node)  
- Sticky Note3 (documentation aid)

**Node Details:**

- **Sending Events To Facebook Pixel**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: Issues POST request to Meta CAPI endpoint:  
    - URL: `https://graph.facebook.com/v24.0/PIXEL_ID_HERE/events` (replace `PIXEL_ID_HERE` with your actual Pixel ID)  
    - Authentication: HTTP Bearer Token (Meta Access Token) configured in n8n credentials  
    - Headers: `Content-Type` set to `application/json`  
    - Body: JSON payload from previous node  
  - Input: JSON payload from code node  
  - Output: Response JSON from Meta API  
  - Edge Cases: Authentication failure, network errors, invalid Pixel ID, API rate limits, malformed payloads.

- **Respond to Webhook**  
  - Type: `n8n-nodes-base.respondToWebhook`  
  - Role: Sends HTTP 200 response with the API response back to the original webhook caller  
  - Input: Output of HTTP Request node  
  - Output: HTTP response to client  
  - Edge Cases: Failures in sending response, client disconnects.

- **Sticky Note3** reminds user to replace Pixel ID and configure Bearer token credentials properly.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                         | Input Node(s)                | Output Node(s)                    | Sticky Note                                                                                          |
|-------------------------------|-------------------------|---------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook Trigger         | Entry point, receives conversion data | -                           | Edit Normalize PII               | **👇 STEP 1: START HERE** Explains usage as endpoint and required JSON structure.                   |
| Edit Normalize PII            | Set                     | Normalize and clean PII fields         | Webhook                     | Crypto - Hash Email              | **🔒 STEP 2: SECURE & HASH DATA** Auto normalize and hash user data, no config needed.             |
| Crypto - Hash Email           | Crypto                  | SHA-256 hash normalized email          | Edit Normalize PII           | Crypto - Hash Phone             | **🔒 STEP 2: SECURE & HASH DATA**                                                                       |
| Crypto - Hash Phone           | Crypto                  | SHA-256 hash normalized phone number   | Crypto - Hash Email          | Crypto - First Name             | **🔒 STEP 2: SECURE & HASH DATA**                                                                       |
| Crypto - First Name           | Crypto                  | SHA-256 hash normalized first name     | Crypto - Hash Phone          | Crypto - Last Name              | **🔒 STEP 2: SECURE & HASH DATA**                                                                       |
| Crypto - Last Name            | Crypto                  | SHA-256 hash normalized last name      | Crypto - First Name          | Set - Compute Timestamps & Map Fields | **🔒 STEP 2: SECURE & HASH DATA**                                                                       |
| Set - Compute Timestamps & Map Fields | Set               | Map fields, compute timestamp and event data | Crypto - Last Name           | Preparing for HTTP Request Payload |                                                                                                    |
| Preparing for HTTP Request Payload | Code                 | Assemble final JSON payload for Meta CAPI | Set - Compute Timestamps & Map Fields | Sending Events To Facebook Pixel  |                                                                                                    |
| Sending Events To Facebook Pixel | HTTP Request          | Send event payload to Meta API          | Preparing for HTTP Request Payload | Respond to Webhook             | **🚀 STEP 3: SEND TO META** Replace Pixel ID & configure Bearer token credential.                   |
| Respond to Webhook            | Respond to Webhook       | Send HTTP 200 response back to caller  | Sending Events To Facebook Pixel | -                               |                                                                                                    |
| Sticky Note                   | Sticky Note             | Documentation and overview              | -                           | -                               | Detailed workflow overview with YouTube setup link: https://youtu.be/_fdMPIYEvFM                    |
| Sticky Note1                  | Sticky Note             | Documentation for Webhook node          | -                           | -                               | Explains webhook usage and required JSON data structure                                            |
| Sticky Note2                  | Sticky Note             | Documentation for Hashing block         | -                           | -                               | Explains data normalization and hashing are automatic and secure                                   |
| Sticky Note3                  | Sticky Note             | Documentation for final sending step    | -                           | -                               | Instructions to replace Pixel ID and add Bearer Token credential                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Type: Webhook  
   - Path: `meta-conversion-api`  
   - HTTP Method: POST  
   - Response Mode: `responseNode`  
   - Note: This node acts as the public endpoint. Copy and use its URL in your external system to send JSON-formatted conversion data.  
   - Required incoming JSON fields: `email`, `phone`, `firstName`, `lastName`, `fbc`, `fbp`.

2. **Add a Set node named "Edit Normalize PII"**  
   - Assignments:  
     - `normalized_email` = `{{$json.body.email.trim().toLowerCase()}}`  
     - `normalized_phone` = `{{$json.body.phone.replace(/\D/g, '')}}`  
     - `firstName` = `{{$json.body.firstName.trim().toLowerCase()}}`  
     - `lastName` = `{{$json.body.lastName.trim().toLowerCase()}}`  
   - Connect Webhook node output to this node.

3. **Add a Crypto node "Crypto - Hash Email"**  
   - Type: SHA256  
   - Value: `{{$json.normalized_email}}`  
   - Data Property Name: `hashed_em`  
   - Connect "Edit Normalize PII" output to this node.

4. **Add a Crypto node "Crypto - Hash Phone"**  
   - Type: SHA256  
   - Value: `{{$json.normalized_phone}}`  
   - Data Property Name: `hashed_ph`  
   - Connect "Crypto - Hash Email" output to this node.

5. **Add a Crypto node "Crypto - First Name"**  
   - Type: SHA256  
   - Value: `{{$json.firstName}}`  
   - Data Property Name: `hashed_firstName`  
   - Connect "Crypto - Hash Phone" output to this node.

6. **Add a Crypto node "Crypto - Last Name"**  
   - Type: SHA256  
   - Value: `{{$json.lastName}}`  
   - Data Property Name: `hashed_lastName`  
   - Connect "Crypto - First Name" output to this node.

7. **Add a Set node "Set - Compute Timestamps & Map Fields"**  
   - Assignments:  
     - `conversion_timestamp` = `{{ Math.floor(Date.now() / 1000) }}` (UNIX epoch seconds)  
     - `event_name` = `SubmitApplication` (or change as needed)  
     - `hashed_em` = `{{ $('Crypto - Hash Email').item.json.hashed_em }}`  
     - `hashed_ph` = `{{ $json.hashed_ph }}`  
     - `hashed_firstName` = `{{ $json.hashed_firstName }}`  
     - `hashed_lastName` = `{{ $json.hashed_lastName }}`  
     - `fbc` = `{{ $('Webhook').item.json.body.fbc }}`  
     - `fbp` = `{{ $('Webhook').item.json.body.fbp }}`  
     - `ipAddress` = `{{ $('Webhook').item.json.headers['x-forwarded-for'] }}`  
     - `userAgent` = `{{ $('Webhook').item.json.headers['user-agent'] }}`  
   - Connect "Crypto - Last Name" output to this node.

8. **Add a Code node "Preparing for HTTP Request Payload"**  
   - JavaScript code:  
     ```javascript
     const incomingData = $json;

     const capiPayload = {
       data: [
         {
           event_name: incomingData.event_name,
           event_time: incomingData.conversion_timestamp,
           action_source: 'website',

           user_data: {
             fn: incomingData.hashed_firstName,
             ln: incomingData.hashed_lastName,
             em: incomingData.hashed_em,
             ph: incomingData.hashed_ph,
             fbc: incomingData.fbc,
             fbp: incomingData.fbp,
             client_ip_address: incomingData.ipAddress,
             client_user_agent: incomingData.userAgent,
           },
         },
       ],
     };

     return capiPayload;
     ```
   - Connect "Set - Compute Timestamps & Map Fields" output to this node.

9. **Add an HTTP Request node "Sending Events To Facebook Pixel"**  
   - Method: POST  
   - URL: `https://graph.facebook.com/v24.0/PIXEL_ID_HERE/events` (replace `PIXEL_ID_HERE` with your Meta Pixel ID)  
   - Authentication: HTTP Bearer Auth (create credential with your Meta CAPI Access Token)  
   - Headers: Content-Type = application/json  
   - Body Content: Use JSON input from previous node  
   - Connect "Preparing for HTTP Request Payload" output to this node.

10. **Add a Respond to Webhook node**  
    - Response Code: 200  
    - Respond With: All incoming items  
    - Connect "Sending Events To Facebook Pixel" output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                              | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow acts as a server-side endpoint to send conversion events directly to the Meta (Facebook) Conversions API, bypassing browser ad blockers and iOS privacy changes.             | General workflow purpose                             |
| Watch the full setup video on YouTube for detailed configuration guidance.                                                                                                                 | https://youtu.be/_fdMPIYEvFM                         |
| Required fields in webhook JSON: `email`, `phone`, `firstName`, `lastName`, `fbc`, and `fbp`. Ensure your external system sends these properly.                                          | Webhook input requirements                           |
| Replace `PIXEL_ID_HERE` in the HTTP Request node URL with your actual Meta Pixel ID.                                                                                                        | HTTP Request node configuration                       |
| Create an HTTP Bearer credential in n8n with your Meta CAPI Access Token and assign it to the HTTP Request node for authentication.                                                        | Credential setup                                     |
| Recommended to monitor for missing or malformed data to avoid errors during normalization and hashing steps.                                                                               | Error handling advice                                |
| The workflow uses SHA-256 hashing for PII fields to comply with Meta’s security requirements, ensuring privacy and data protection.                                                        | Data security compliance                             |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---