Automated Gold Price Threshold Alerts via LINE with Web Scraping

https://n8nworkflows.xyz/workflows/automated-gold-price-threshold-alerts-via-line-with-web-scraping-6265


# Automated Gold Price Threshold Alerts via LINE with Web Scraping

### 1. Workflow Overview

This workflow automates the monitoring of gold prices from a specific webpage and sends an alert via the LINE messaging platform if the gold price exceeds a predefined threshold. It is designed for users who want to receive timely notifications about gold price fluctuations without manual checking. The workflow includes the following logical blocks:

- **1.1 Scheduled Webpage Fetching:** Periodically triggers the workflow to fetch the gold price webpage every 6 hours.
- **1.2 Data Extraction:** Parses the fetched HTML content to extract the current gold price.
- **1.3 Price Condition Evaluation:** Checks if the extracted gold price exceeds a specific threshold.
- **1.4 Alert Notification:** Sends an alert message via LINE if the price condition is met.

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Webpage Fetching

- **Overview:**  
  This block triggers the workflow every 6 hours and performs an HTTP GET request to retrieve the HTML content of the gold price website.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Webpage  
  - Sticky Note (Schedule explanation)  
  - Sticky Note1 (HTTP request context)  
  - Sticky Note2 (HTML content context)

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow execution every 6 hours.  
    - Configuration: Set to trigger every 6 hours using interval scheduling.  
    - Input: None (start node)  
    - Output: Sends trigger signal to "Get Webpage" node.  
    - Edge Cases: Workflow will not run if n8n instance is down or paused; consider time zone impact if relevant.  

  - **Get Webpage**  
    - Type: HTTP Request  
    - Role: Fetches the target webpage HTML content.  
    - Configuration:  
      - Method: GET (default)  
      - URL: https://www.goldtraders.or.th/  
      - No authentication or special headers used.  
    - Input: Trigger from Schedule Trigger  
    - Output: Raw HTML content forwarded to "Extract Price" node.  
    - Edge Cases: Network errors, site downtime, or changes to webpage URL may cause failure. No retry logic specified.  

  - **Sticky Note** (Schedule explanation)  
    - Content: "Schedule to check every 6 hours to see the current gold price"  
    - Role: Documentation for users about scheduling frequency.

  - **Sticky Note1** (HTTP request context)  
    - Content: "To get normal webpage, we can use HTTP request without any authoriation and the output will be HTML code"  
    - Role: Explains no authorization is needed for the HTTP request and output type.

  - **Sticky Note2** (HTML content context)  
    - Content: "We would specify what we want from the HTML code earlier eg. Price -- You can find the element by right click > inspect"  
    - Role: Provides guidance on how to identify the HTML element to extract.

---

#### 2.2 Data Extraction

- **Overview:**  
  This block parses the HTML content received from the webpage request and extracts the current gold price using a CSS selector.

- **Nodes Involved:**  
  - Extract Price  
  - Sticky Note2 (continued contextual note)

- **Node Details:**  

  - **Extract Price**  
    - Type: HTML  
    - Role: Extracts specific content from the HTML string.  
    - Configuration:  
      - Operation: Extract HTML content  
      - Extraction values: Uses CSS selector `#DetailPlace_uc_goldprices1_lblBLBuy` to extract the gold buying price element.  
    - Input: Raw HTML from "Get Webpage" node  
    - Output: JSON object with key `#DetailPlace_uc_goldprices1_lblBLBuy` containing the extracted price text.  
    - Edge Cases:  
      - If the selector changes due to webpage redesign, extraction will fail.  
      - If the element is missing, output may be empty or null, causing downstream errors.

---

#### 2.3 Price Condition Evaluation

- **Overview:**  
  Evaluates whether the extracted gold price exceeds a preset threshold (52300). Only if this condition is true will the alert proceed.

- **Nodes Involved:**  
  - If (conditional node)  
  - Sticky Note3 (price filtering explanation)

- **Node Details:**  

  - **If**  
    - Type: If (conditional)  
    - Role: Filters workflow execution based on gold price comparison.  
    - Configuration:  
      - Condition 1: Checks if the extracted gold price (converted from string to float by removing non-numeric characters) is greater than 52300.  
      - Condition 2: An empty string equality check with no effect (likely placeholder or redundant).  
      - Logical operator: AND (both conditions must be true).  
    - Input: JSON with extracted price from "Extract Price" node.  
    - Output:  
      - True branch: Continues to send alert if price > 52300.  
      - False branch: Ends workflow silently.  
    - Key Expression: `={{ parseFloat($json['#DetailPlace_uc_goldprices1_lblBLBuy'].replace(/[^\d.]/g, '')) }}`  
    - Edge Cases:  
      - If price extraction fails or returns non-numeric value, parseFloat may return NaN causing condition to fail or error.  
      - The second empty condition is redundant and does not affect logic.  

  - **Sticky Note3**  
    - Content: "We do not want all the alerts. Thus, we only filter to alert once the price is more than the given number. In order to compare, we also need to convert the text to number as well."  
    - Role: Explains the rationale for filtering alerts based on numeric comparison.

---

#### 2.4 Alert Notification

- **Overview:**  
  Sends a push message via the LINE messaging API when the gold price exceeds the threshold.

- **Nodes Involved:**  
  - Send Line Message  
  - Sticky Note4 (alert explanation)

- **Node Details:**  

  - **Send Line Message**  
    - Type: HTTP Request  
    - Role: Pushes a text message to a LINE user or group using LINE Bot API.  
    - Configuration:  
      - URL: `https://api.line.me/v2/bot/message/push`  
      - Method: POST  
      - Body: JSON specifying recipient ID and message text including the extracted gold price.  
      - Authentication: HTTP Header Auth with stored LINE channel access token credentials.  
    - Input: Triggered from the true branch of the If node.  
    - Output: Response from LINE API (not used further).  
    - Key Expression in Body: `"ราคาทองวันนี้  {{ $json['#DetailPlace_uc_goldprices1_lblBLBuy'] }}"` (Thai for "Today's gold price")  
    - Edge Cases:  
      - Authentication token expiry or invalid credentials cause 401 errors.  
      - Network issues or LINE API downtime cause failures.  
      - Rate limiting by LINE API if many messages sent.  

  - **Sticky Note4**  
    - Content: "When the condition is met, it'll send the message via line. This can be other platform such as telegram or email"  
    - Role: Notes extendability to other notification platforms.

---

### 3. Summary Table

| Node Name         | Node Type            | Functional Role                   | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                     |
|-------------------|----------------------|---------------------------------|------------------------|-----------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger  | Schedule Trigger     | Periodically triggers workflow   | None                   | Get Webpage           | Schedule to check every 6 hours to see the current gold price                                   |
| Get Webpage       | HTTP Request         | Fetch HTML content of gold price | Schedule Trigger       | Extract Price         | To get normal webpage, we can use HTTP request without any authoriation and the output will be HTML code |
| Extract Price     | HTML Extract         | Extract gold price from HTML     | Get Webpage            | If                    | We would specify what we want from the HTML code earlier eg. Price -- You can find the element by right click > inspect |
| If                | If Condition         | Check if price > 52300           | Extract Price          | Send Line Message     | We do not want all the alerts. Thus, we only filter to alert once the price is more than the given number. In order to compare, we also need to convert the text to number as well. |
| Send Line Message | HTTP Request         | Send LINE alert message          | If (true branch)       | None                  | When the condition is met, it'll send the message via line. This can be other platform such as telegram or email |

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Set interval to every 6 hours.

2. **Create "Get Webpage" node**  
   - Type: HTTP Request  
   - Set HTTP Method: GET  
   - URL: `https://www.goldtraders.or.th/`  
   - No authentication required.  
   - Connect "Schedule Trigger" output to this node's input.

3. **Create "Extract Price" node**  
   - Type: HTML  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `#DetailPlace_uc_goldprices1_lblBLBuy`  
     - CSS Selector: `#DetailPlace_uc_goldprices1_lblBLBuy`  
   - Connect "Get Webpage" output to this node's input.

4. **Create "If" node**  
   - Type: If (Condition)  
   - Add condition:  
     - Left Value: `={{ parseFloat($json['#DetailPlace_uc_goldprices1_lblBLBuy'].replace(/[^\d.]/g, '')) }}`  
     - Operator: Greater than (>)  
     - Right Value: `52300`  
   - Connect "Extract Price" output to this node's input.

5. **Create "Send Line Message" node**  
   - Type: HTTP Request  
   - HTTP Method: POST  
   - URL: `https://api.line.me/v2/bot/message/push`  
   - Set Body Format: JSON  
   - Body:  
     ```json
     {
       "to": "Ue9cc622e33e5333e3784298412ec9aed",
       "messages": [
         {
           "type": "text",
           "text": "ราคาทองวันนี้  {{ $json['#DetailPlace_uc_goldprices1_lblBLBuy'] }}"
         }
       ]
     }
     ```  
   - Authentication: HTTP Header Auth  
     - Create and use credentials with LINE channel access token (OAuth2 token or channel access token).  
   - Connect the "If" node's **true** output to this node's input.

6. **Leave the "If" node's false output unconnected** to stop the workflow if the condition is not met.

7. **Optional:** Add sticky notes with the following contents for documentation:  
   - Schedule explanation (every 6 hours)  
   - HTTP request and HTML extraction notes  
   - Price filtering explanation  
   - Alert notification explanation

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                                         |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Use browser developer tools ("Inspect") to find the CSS selector of the gold price element on the webpage. | This helps define the extraction value in the HTML node.                                                                |
| The LINE Messaging API requires an access token to authenticate requests.                                  | LINE API documentation: https://developers.line.biz/en/docs/messaging-api/overview/                                     |
| This workflow can be extended to other platforms like Telegram, email, or SMS by replacing the notification node. | Provides flexibility for different user preferences.                                                                    |

---

**Disclaimer:** The text provided originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.