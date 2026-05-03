Monitor PKI Certificates & CRLs for Expiration with Telegram & SMS Alerts

https://n8nworkflows.xyz/workflows/monitor-pki-certificates---crls-for-expiration-with-telegram---sms-alerts-9865


# Monitor PKI Certificates & CRLs for Expiration with Telegram & SMS Alerts

### 1. Workflow Overview

This workflow, titled **"PKI Certificate & CRL Monitor - Auto Expiration Alert System"**, automates the monitoring of Public Key Infrastructure (PKI) elements, specifically Certificate Authorities (CA) certificates, Certificate Revocation Lists (CRLs), and associated web services. It is designed to regularly extract URLs related to PKI components from a Trusted Service List (TSL) XML source, download and parse these certificates and CRLs, check their validity periods and revocation update schedules, and send timely alerts via Telegram and SMS if any are approaching expiration or service downtime.

The workflow uses scheduled and manual triggers to initiate the process every 12 hours with a configurable warning threshold of 17 hours before expiration or next update. The main logical blocks are:

- **1.1 TSL URL Extraction and Preparation:** Downloads TSL XML, extracts certificate URLs, and formats them into a JSON list.
- **1.2 URL Processing Loop:** Iterates over each extracted URL to classify and handle them as CRLs, CA certificates, or other web services.
- **1.3 CRL Monitoring Pipeline:** Downloads CRL files, parses them with OpenSSL, extracts update times, and flags near-expiry status.
- **1.4 CA Certificate Monitoring Pipeline:** Downloads CA certificates, parses them with OpenSSL, extracts validity dates, and flags near-expiry or expired certificates.
- **1.5 Website Availability Monitoring:** Checks non-CRL/CA URLs for HTTP 200 responses and alerts if the service is down after retries.
- **1.6 Alert Dispatch:** Sends alerts via Telegram and SMS for CRLs, CAs, and website availability issues.

---

### 2. Block-by-Block Analysis

#### 1.1 TSL URL Extraction and Preparation
- **Overview:**  
  This block downloads the Hungarian TSL XML from NMHH, extracts all X509Certificate entries, converts them from base64 to PEM format, parses to extract CRL and CA URLs, and compiles all URLs into a JSON object for further processing.

- **Nodes Involved:**  
  - Execute Command: `Collect Checking URL list`  
  - Set: `Get Checking URL list to URL objects`  
  - Split Out: `Get Url From List`

- **Node Details:**

  - **Collect Checking URL list (Execute Command)**  
    - Runs shell commands to install dependencies (libxml2-utils, curl, jq), download TSL XML, parse certificates, extract URLs, and generate a JSON file of URLs.  
    - Key outputs: JSON object with all discovered URLs.  
    - Potential failures: network errors, missing dependencies, malformed XML, shell command errors.

  - **Get Checking URL list to URL objects (Set)**  
    - Converts the stdout JSON string from previous node into an object to be used in workflow.  
    - Inputs: stdout from `Collect Checking URL list`.  
    - Outputs: JSON with URLs field containing all extracted URLs.

  - **Get Url From List (Split Out)**  
    - Splits the URLs JSON object into individual URL items for batch processing.  
    - Inputs: URLs object from previous node.  
    - Outputs: Individual URL items for looping.

---

#### 1.2 URL Processing Loop
- **Overview:**  
  Splits the list of URLs and processes them sequentially in batches of one. Classifies each URL as CRL, CA certificate, or other, directing the flow accordingly.

- **Nodes Involved:**  
  - Split In Batches: `Loop Over Items -- Process Checking`  
  - If: `Is this CRL ?`  
  - If: `Is this CA?`

- **Node Details:**

  - **Loop Over Items -- Process Checking (Split In Batches)**  
    - Processes URLs one by one to maintain sequential logic.  
    - Inputs: individual URLs from `Get Url From List`.  
    - Outputs: one URL item per iteration.

  - **Is this CRL ? (If)**  
    - Checks if URL contains "crl=" or ".crl" to identify CRL URLs.  
    - If true, routes to CRL processing; else, to CA check.

  - **Is this CA? (If)**  
    - Checks if URL contains "ca=", ".crt", or ".cer" to identify CA certificate URLs.  
    - If true, routes to CA processing; else, to website availability check.

- **Potential issues:**  
  - URLs with ambiguous or missing indicators may be misclassified.  
  - Network or parsing failures in downstream nodes.

---

#### 1.3 CRL Monitoring Pipeline
- **Overview:**  
  Downloads CRL files, decodes, parses with OpenSSL to extract last update and next update times, and checks if the next update is within 17 hours to trigger alerts.

- **Nodes Involved:**  
  - HTTP Request: `Get CRL URL`  
  - If: `CRL Available?`  
  - Write Binary File: `Write Binary - TempFile : crlfile.der`  
  - Execute Command: `B64 Encode : crlfile.der`  
  - Execute Command: `OpenSSL Parse CRL : crlfile.der`  
  - Code: `Parse Data OpenSSL output`  
  - Code: `nextUpdate - TimeFilter`  
  - If: `Good Or Evil`  
  - Set: `Set-Info for CRL Alert`  
  - Execute Command: `CRL Alert --- Telegam & SMS`

- **Node Details:**

  - **Get CRL URL (HTTP Request)**  
    - Downloads CRL binary file from the URL with 10s timeout, retries on failure.  
    - Response type: file.  
    - Failures: network timeouts, HTTP errors.

  - **CRL Available? (If)**  
    - Checks if HTTP status code is 200 (OK).  
    - Only processes if CRL is available.

  - **Write Binary - TempFile : crlfile.der (Write Binary File)**  
    - Saves downloaded CRL file as "crlfile.der" for processing.  
    - Continues on error to avoid blocking workflow.

  - **B64 Encode : crlfile.der (Execute Command)**  
    - Base64 encodes the CRL file to prepare for OpenSSL parsing.  
    - Uses `base64 -w0 crlfile.der`.  
    - Continues on error.

  - **OpenSSL Parse CRL : crlfile.der (Execute Command)**  
    - Decodes base64 and pipes to `openssl crl` to extract text info without outputting the body.  
    - Continues on error.

  - **Parse Data OpenSSL output (Code)**  
    - Extracts `Last Update`, `Next Update`, and Issuer `CN` from OpenSSL output using regex.  
    - Outputs JSON with these fields.

  - **nextUpdate - TimeFilter (Code)**  
    - Converts `Next Update` to a Date object, calculates hours difference from now.  
    - Sets `crlStatus` to "ALERT!" if less than 17 hours remain; otherwise "READY!".  
    - Outputs status with timestamps.

  - **Good Or Evil (If)**  
    - Checks if `crlStatus` equals "ALERT!" to trigger alert branch.

  - **Set-Info for CRL Alert (Set)**  
    - Prepares alert information with last update, next update, issuer CN, and status.

  - **CRL Alert --- Telegam & SMS (Execute Command)**  
    - Sends Telegram and SMS alerts with CRL expiration warning.  
    - Requires user to configure Telegram Bot Token, Chat ID, phone numbers, and Textbelt API key.  
    - Retries Telegram send until successful.  
    - Possible failures: network issues, API auth errors.

---

#### 1.4 CA Certificate Monitoring Pipeline
- **Overview:**  
  Downloads CA certificates, decodes and parses them with OpenSSL, extracts certificate validity period, checks expiration status with a 17-hour warning threshold, and sends alerts if necessary.

- **Nodes Involved:**  
  - HTTP Request: `Get CA URL`  
  - If: `CA Available?`  
  - Write Binary File: `Write Binary - TempFile : cacertfile.crt`  
  - Execute Command: `B64 Encode : cacertfile.crt`  
  - Execute Command: `OpenSSL Parse CA Cert : cacertfile.crt`  
  - Code: `Parse Data OpenSSL CA output`  
  - Code: `nextUpdate - TimeFilter1`  
  - If: `Alive?`  
  - Set: `Set-Info for CA Alert`  
  - Execute Command: `CA Alert --- Telegam & SMS`

- **Node Details:**

  - **Get CA URL (HTTP Request)**  
    - Downloads CA certificate file with 10s timeout and retries.  
    - Response type: file.

  - **CA Available? (If)**  
    - Checks if HTTP status code is 200 (OK).

  - **Write Binary - TempFile : cacertfile.crt (Write Binary File)**  
    - Saves downloaded certificate as "cacertfile.crt".

  - **B64 Encode : cacertfile.crt (Execute Command)**  
    - Base64 encodes the certificate file.

  - **OpenSSL Parse CA Cert : cacertfile.crt (Execute Command)**  
    - Decodes base64 and runs `openssl x509` to parse certificate data.

  - **Parse Data OpenSSL CA output (Code)**  
    - Extracts `Not Before`, `Not After`, and Subject `CN` from OpenSSL output with regex.

  - **nextUpdate - TimeFilter1 (Code)**  
    - Converts `Not After` to Date, calculates if certificate expired or will expire within 17 hours.  
    - Sets `caStatus` to "EXPIRED!", "ALERT!", or "READY!".

  - **Alive? (If)**  
    - Triggers alert path if status is not "READY!".

  - **Set-Info for CA Alert (Set)**  
    - Prepares alert info with dates, subject CN, and status.

  - **CA Alert --- Telegam & SMS (Execute Command)**  
    - Sends Telegram and SMS alerts for CA certificate expiration issues.  
    - Requires user configuration similar to CRL alert node.

---

#### 1.5 Website Availability Monitoring
- **Overview:**  
  For URLs that are neither CRL nor CA certificate, the workflow checks HTTP status. If the first check fails, it waits and retries. If both fail, it sends alerts for potential service downtime.

- **Nodes Involved:**  
  - HTTP Request: `Get Non-CRL/CA URL`  
  - If: `Website Available?`  
  - Wait: `Wait B4 Retry`  
  - HTTP Request: `Retry to Non-CRL/CA URL`  
  - If: `Website Available Now?`  
  - Execute Command: `Send Website Down - Telegram & SMS`  
  - Set: `Respose Extend With Requested URL`

- **Node Details:**

  - **Get Non-CRL/CA URL (HTTP Request)**  
    - Downloads content from non-CRL/CA URLs with 10s timeout and retries.

  - **Website Available? (If)**  
    - Checks if HTTP status code is 200.

  - **Wait B4 Retry (Wait)**  
    - Pauses workflow before retrying.

  - **Retry to Non-CRL/CA URL (HTTP Request)**  
    - Retries the HTTP request.

  - **Website Available Now? (If)**  
    - Checks if retry succeeded.  
    - If not, triggers alert.

  - **Send Website Down - Telegram & SMS (Execute Command)**  
    - Sends Telegram and SMS alerts for website downtime.  
    - Requires user configuration similar to other alert nodes.

  - **Respose Extend With Requested URL (Set)**  
    - Adds URL context to the response for alert message clarity.

- **Potential issues:**  
  - False positives due to transient network issues.  
  - Rate limits on alert APIs.

---

#### 1.6 Triggers and General Configuration
- **Nodes Involved:**  
  - Manual Trigger: `Execute With Manual Start`  
  - Schedule Trigger: `Execute With Scheduled Start`

- **Node Details:**

  - **Execute With Manual Start (Manual Trigger)**  
    - Allows manual execution of the workflow.

  - **Execute With Scheduled Start (Schedule Trigger)**  
    - Automatically runs workflow every 12 hours.

- **Sticky Notes:**  
  Provide documentation for configuration, thresholds, TSL source, alert placeholders, and pipeline summaries.

---

### 3. Summary Table

| Node Name                         | Node Type              | Functional Role                          | Input Node(s)                        | Output Node(s)                          | Sticky Note                                                                                         |
|----------------------------------|------------------------|----------------------------------------|------------------------------------|----------------------------------------|--------------------------------------------------------------------------------------------------|
| Execute With Manual Start         | Manual Trigger         | Manual start trigger                    | -                                  | Collect Checking URL list               | ## PKI Certificate & CRL Monitor: Automated PKI monitoring with alerts every 12 hours (17h warn) |
| Execute With Scheduled Start      | Schedule Trigger       | Scheduled start every 12 hours          | -                                  | Collect Checking URL list               | ## PKI Certificate & CRL Monitor: Automated PKI monitoring with alerts every 12 hours (17h warn) |
| Collect Checking URL list         | Execute Command        | Download and extract certificate URLs from TSL | Execute With Manual/Scheduled Start | Get Checking URL list to URL objects    | ## TSL SOURCE CONFIGURATION: Default Hungarian TSL; modify URL in this node; installs dependencies |
| Get Checking URL list to URL objects | Set                  | Parse JSON string to URL object         | Collect Checking URL list           | Get Url From List                       | ## URL Extraction Process: Extract X509Certificate tags and produce URLs JSON                     |
| Get Url From List                 | Split Out              | Split URLs into individual items         | Get Checking URL list to URL objects | Loop Over Items -- Process Checking     | ## URL Processing Loop: Process URLs sequentially, classify as CRL, CA, or other                  |
| Loop Over Items -- Process Checking | Split In Batches     | Process URLs one by one, batching size 1 | Get Url From List                  | Is this CRL ?, (empty branch)           | ## URL Processing Loop: Process URLs sequentially, classify as CRL, CA, or other                  |
| Is this CRL ?                    | If                     | Check if URL is CRL                      | Loop Over Items -- Process Checking | Get CRL URL, Is this CA?                | ## CRL Monitoring Pipeline: Detect crl URLs, parse, alert if close to expiration                   |
| Get CRL URL                      | HTTP Request           | Download CRL file                       | Is this CRL ?                      | CRL Available?                         | ## CRL Monitoring Pipeline: Detect crl URLs, parse, alert if close to expiration                   |
| CRL Available?                  | If                     | Check if CRL download succeeded (HTTP 200) | Get CRL URL                      | Write Binary - TempFile : crlfile.der, Loop Over Items -- Process Checking | ## CRL Monitoring Pipeline: Detect crl URLs, parse, alert if close to expiration                   |
| Write Binary - TempFile : crlfile.der | Write Binary File  | Save CRL file to disk                    | CRL Available?                    | B64 Encode : crlfile.der, Loop Over Items -- Process Checking | ## CRL Monitoring Pipeline: Detect crl URLs, parse, alert if close to expiration                   |
| B64 Encode : crlfile.der         | Execute Command        | Base64 encode CRL file                   | Write Binary - TempFile : crlfile.der | OpenSSL Parse CRL : crlfile.der        | ## CRL Monitoring Pipeline: Detect crl URLs, parse, alert if close to expiration                   |
| OpenSSL Parse CRL : crlfile.der  | Execute Command        | Parse CRL with OpenSSL                   | B64 Encode : crlfile.der           | Parse Data OpenSSL output              | ## CRL Monitoring Pipeline: Detect crl URLs, parse, alert if close to expiration                   |
| Parse Data OpenSSL output        | Code                   | Extract Last/Next Update and Issuer CN  | OpenSSL Parse CRL : crlfile.der    | nextUpdate - TimeFilter                | ## CRL Monitoring Pipeline: Detect crl URLs, parse, alert if close to expiration                   |
| nextUpdate - TimeFilter          | Code                   | Check if CRL next update is near (17h)  | Parse Data OpenSSL output          | Good Or Evil                          | ## Threshold Configuration: Warning threshold 17 hours before expiration                           |
| Good Or Evil                    | If                     | Branch on CRL status "ALERT!"            | nextUpdate - TimeFilter            | Set-Info for CRL Alert, Loop Over Items -- Process Checking | ## ALERT CONFIGURATION: Configure Telegram/SMS credentials in alert nodes                         |
| Set-Info for CRL Alert           | Set                    | Prepare CRL alert message data           | Good Or Evil                      | CRL Alert --- Telegam & SMS            | ## ALERT CONFIGURATION: Configure Telegram/SMS credentials in alert nodes                         |
| CRL Alert --- Telegam & SMS      | Execute Command        | Send CRL expiration alerts               | Set-Info for CRL Alert             | Loop Over Items -- Process Checking     | ## ALERT CONFIGURATION: Replace placeholders with real tokens & numbers                           |
| Is this CA?                     | If                     | Check if URL is CA certificate            | Is this CRL ?                    | Get CA URL, Get Non-CRL/CA URL         | ## CA Certificate Monitoring Pipeline: Detect ca URLs, parse, alert if close to expiration        |
| Get CA URL                      | HTTP Request           | Download CA certificate                   | Is this CA?                      | CA Available?                        | ## CA Certificate Monitoring Pipeline: Detect ca URLs, parse, alert if close to expiration        |
| CA Available?                   | If                     | Check if CA download succeeded (HTTP 200) | Get CA URL                      | Write Binary - TempFile : cacertfile.crt, Loop Over Items -- Process Checking | ## CA Certificate Monitoring Pipeline: Detect ca URLs, parse, alert if close to expiration        |
| Write Binary - TempFile : cacertfile.crt | Write Binary File  | Save CA cert to disk                      | CA Available?                    | B64 Encode : cacertfile.crt, Loop Over Items -- Process Checking | ## CA Certificate Monitoring Pipeline: Detect ca URLs, parse, alert if close to expiration        |
| B64 Encode : cacertfile.crt      | Execute Command        | Base64 encode CA certificate              | Write Binary - TempFile : cacertfile.crt | OpenSSL Parse CA Cert : cacertfile.crt| ## CA Certificate Monitoring Pipeline: Detect ca URLs, parse, alert if close to expiration        |
| OpenSSL Parse CA Cert : cacertfile.crt | Execute Command  | Parse CA certificate with OpenSSL         | B64 Encode : cacertfile.crt       | Parse Data OpenSSL CA output           | ## CA Certificate Monitoring Pipeline: Detect ca URLs, parse, alert if close to expiration        |
| Parse Data OpenSSL CA output     | Code                   | Extract Not Before, Not After, Subject CN | OpenSSL Parse CA Cert : cacertfile.crt | nextUpdate - TimeFilter1              | ## CA Certificate Monitoring Pipeline: Detect ca URLs, parse, alert if close to expiration        |
| nextUpdate - TimeFilter1         | Code                   | Check CA certificate expiration status    | Parse Data OpenSSL CA output      | Alive?                               | ## Threshold Configuration: Warning threshold 17 hours before expiration                           |
| Alive?                         | If                     | Branch on CA status not READY              | nextUpdate - TimeFilter1          | Set-Info for CA Alert, Loop Over Items -- Process Checking | ## ALERT CONFIGURATION: Configure Telegram/SMS credentials in alert nodes                         |
| Set-Info for CA Alert            | Set                    | Prepare CA alert message data              | Alive?                          | CA Alert --- Telegam & SMS             | ## ALERT CONFIGURATION: Configure Telegram/SMS credentials in alert nodes                         |
| CA Alert --- Telegam & SMS       | Execute Command        | Send CA certificate expiration alerts     | Set-Info for CA Alert             | Loop Over Items -- Process Checking     | ## ALERT CONFIGURATION: Replace placeholders with real tokens & numbers                           |
| Get Non-CRL/CA URL              | HTTP Request           | Download non-CRL/CA URL content            | Is this CA?                      | Respose Extend With Requested URL      | ## Website Availability Monitoring: Check non-CRL/CA URLs for HTTP 200 status                      |
| Respose Extend With Requested URL | Set                    | Add URL info to response                    | Get Non-CRL/CA URL              | Website Available?                     | ## Website Availability Monitoring: Check non-CRL/CA URLs for HTTP 200 status                      |
| Website Available?              | If                     | Check if HTTP status 200                     | Respose Extend With Requested URL | Loop Over Items -- Process Checking, Wait B4 Retry | ## Website Availability Monitoring: Retry on failure with wait interval                         |
| Wait B4 Retry                  | Wait                   | Pause before retrying failed HTTP request   | Website Available? (fail branch)  | Retry to Non-CRL/CA URL                | ## Website Availability Monitoring: Retry on failure with wait interval                         |
| Retry to Non-CRL/CA URL        | HTTP Request           | Retry HTTP request after wait                | Wait B4 Retry                   | Website Available Now?                 | ## Website Availability Monitoring: Retry on failure with wait interval                         |
| Website Available Now?          | If                     | Check HTTP status after retry                 | Retry to Non-CRL/CA URL          | Loop Over Items -- Process Checking, Send Website Down - Telegram & SMS | ## ALERT CONFIGURATION: Configure Telegram/SMS credentials in alert nodes                         |
| Send Website Down - Telegram & SMS | Execute Command     | Send alerts for website downtime              | Website Available Now? (fail branch) | Loop Over Items -- Process Checking   | ## ALERT CONFIGURATION: Replace placeholders with real tokens & numbers                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**
   - Add a **Manual Trigger** node named `Execute With Manual Start`.
   - Add a **Schedule Trigger** node named `Execute With Scheduled Start` set to run every 12 hours.

2. **TSL URL Extraction Block:**
   - Add an **Execute Command** node `Collect Checking URL list`:
     - Command: installs dependencies (libxml2-utils, curl, jq), downloads Hungarian TSL XML, extracts X509Certificate tags, converts base64 to PEM, extracts URLs, and outputs JSON.
     - Must produce a JSON string with URLs.
   - Add a **Set** node `Get Checking URL list to URL objects`:
     - Assign value from the previous node's `stdout` as JSON object.
   - Add a **Split Out** node `Get Url From List`:
     - Split the `urls` field into individual items.

3. **Processing Loop:**
   - Add a **Split In Batches** node `Loop Over Items -- Process Checking`:
     - Batch size: 1.
     - Connect from `Get Url From List`.

4. **CRL Detection and Processing:**
   - Add an **If** node `Is this CRL ?`:
     - Condition: URL contains `crl=` or `.crl`.
     - True branch to CRL processing, false to CA check.
   - CRL Processing Nodes:
     - **HTTP Request** `Get CRL URL` to download the CRL file (response as file, timeout 10s, retry on fail).
     - **If** `CRL Available?` checking HTTP 200.
     - **Write Binary File** `Write Binary - TempFile : crlfile.der` to save CRL.
     - **Execute Command** `B64 Encode : crlfile.der` to base64 encode CRL file.
     - **Execute Command** `OpenSSL Parse CRL : crlfile.der` to parse CRL with OpenSSL.
     - **Code** `Parse Data OpenSSL output` with regex to extract `Last Update`, `Next Update`, and Issuer CN.
     - **Code** `nextUpdate - TimeFilter` to calculate hours until `Next Update`; set `crlStatus` as `ALERT!` if under 17 hours.
     - **If** `Good Or Evil` branching on `crlStatus == "ALERT!"`.
     - **Set** `Set-Info for CRL Alert` to prepare alert message fields.
     - **Execute Command** `CRL Alert --- Telegam & SMS` to send alerts via Telegram and SMS (requires configuration).

5. **CA Certificate Detection and Processing:**
   - Add an **If** node `Is this CA?`:
     - Condition: URL contains `ca=`, `.crt`, or `.cer`.
     - True branch to CA processing, false branch to website availability.
   - CA Processing Nodes:
     - **HTTP Request** `Get CA URL` to download CA certificate (response as file, timeout 10s, retry on fail).
     - **If** `CA Available?` checking HTTP 200.
     - **Write Binary File** `Write Binary - TempFile : cacertfile.crt` to save certificate.
     - **Execute Command** `B64 Encode : cacertfile.crt` to base64 encode.
     - **Execute Command** `OpenSSL Parse CA Cert : cacertfile.crt` to parse certificate.
     - **Code** `Parse Data OpenSSL CA output` extracting `Not Before`, `Not After`, `Subject CN`.
     - **Code** `nextUpdate - TimeFilter1` checks expiry; sets `caStatus` to `EXPIRED!`, `ALERT!`, or `READY!`.
     - **If** `Alive?` branches if status not `READY!`.
     - **Set** `Set-Info for CA Alert` to prepare alert message.
     - **Execute Command** `CA Alert --- Telegam & SMS` to send alerts (requires config).

6. **Website Availability Monitoring:**
   - **HTTP Request** `Get Non-CRL/CA URL` downloads other URLs (response as string, timeout 10s, retry on fail).
   - **Set** `Respose Extend With Requested URL` to include URL context for alerts.
   - **If** `Website Available?` checks HTTP 200.
     - If not available:
       - **Wait** `Wait B4 Retry` to pause before retry.
       - **HTTP Request** `Retry to Non-CRL/CA URL` retries download.
       - **If** `Website Available Now?` checks HTTP 200 after retry.
         - If still down, **Execute Command** `Send Website Down - Telegram & SMS` sends alerts.

7. **Connect all branches back to `Loop Over Items -- Process Checking`** to continue batch processing.

8. **Configure Alert Nodes:**
   - Replace placeholders in alert command nodes with:
     - Telegram Bot Token and Chat ID.
     - Phone numbers for SMS.
     - Textbelt API key.

9. **Set Thresholds:**
   - Adjust warning threshold hours (default 17) inside the code nodes `nextUpdate - TimeFilter` (CRL) and `nextUpdate - TimeFilter1` (CA).

10. **Add Sticky Notes:**
    - Add informative sticky notes for pipeline descriptions, threshold, TSL source, and alert configuration for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                | Context or Link                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| Automated monitoring of PKI components with alerts via Telegram and SMS every 12 hours. Warning threshold is 17 hours before expiration.                                                                    | Sticky Note: Workflow overview                                                             |
| Default TSL source is Hungarian NMHH TSL XML: http://www.nmhh.hu/tl/pub/HU_TL.xml. Modify in `Collect Checking URL list` node. Installs dependencies: libxml2-utils, curl, jq, OpenSSL on runtime environment. | Sticky Note: TSL SOURCE CONFIGURATION                                                     |
| URL Extraction downloads TSL, extracts X509Certificate tags, converts base64 to PEM, parses certificates to extract CRL and CA URLs, outputs JSON list.                                                    | Sticky Note: URL Extraction Process                                                       |
| Processing loop handles URLs one by one, classifying as CRL, CA cert, or other, ensuring sequential processing.                                                                                            | Sticky Note: URL Processing Loop                                                          |
| CRL monitoring detects URLs with 'crl=' or '.crl', downloads CRL, parses update timestamps, alerts if next update is within 17 hours.                                                                        | Sticky Note: CRL Monitoring Pipeline                                                      |
| CA cert monitoring detects 'ca=', '.crt', or '.cer' URLs, downloads certs, parses validity dates, alerts if expired or expiring within 17 hours.                                                            | Sticky Note: CA Certificate Monitoring Pipeline                                           |
| Website availability monitoring checks non-CRL/CA URLs for HTTP 200 status, retries once after wait, alerts if still unavailable.                                                                           | Sticky Note: Website Availability Monitoring                                              |
| Alert nodes require user configuration for Telegram Bot Token, Chat ID, phone numbers, and Textbelt API key. Replace placeholders accordingly.                                                              | Sticky Note: ALERT CONFIGURATION                                                          |
| Warning threshold default is 17 hours; change in code nodes `nextUpdate - TimeFilter` and `nextUpdate - TimeFilter1`.                                                                                        | Sticky Note: Threshold Configuration                                                      |

---

This structured documentation provides a clear, detailed understanding of the complete PKI monitoring workflow, enabling reproduction, modification, and troubleshooting.