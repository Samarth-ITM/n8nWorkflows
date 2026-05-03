Food Delivery Notifications and Easy Expense Tracking

https://n8nworkflows.xyz/workflows/food-delivery-notifications-and-easy-expense-tracking-2707


# Food Delivery Notifications and Easy Expense Tracking

### 1. Workflow Overview

This workflow automates the retrieval of food delivery order emails from Gmail, extracts key order details, and sends structured notifications to a Slack channel. It also embeds a clickable link in the Slack message that opens the Moze accounting app with pre-filled expense data, facilitating quick expense tracking.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Manual and automatic triggers to start the workflow.
- **1.2 Email Retrieval**: Fetching emails from Gmail that match specific subject keywords.
- **1.3 Data Extraction**: Parsing email content to extract order price, shop name, date, and time.
- **1.4 Slack Notification**: Sending a formatted Slack message with extracted details and a Moze app URL scheme link.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow either manually for testing or automatically via Gmail trigger when new matching emails arrive.

**Nodes Involved:**  
- Click to Test Flow (Manual Trigger)  
- Receive certain keyword Gmail Trigger (Gmail Trigger)

**Node Details:**

- **Click to Test Flow**  
  - Type: Manual Trigger  
  - Role: Allows manual execution of the workflow for immediate testing or on-demand runs.  
  - Configuration: No parameters needed.  
  - Inputs: None  
  - Outputs: Connects to "Get emails from Gmail with certain subject" node.  
  - Edge Cases: None significant; manual trigger depends on user action.

- **Receive certain keyword Gmail Trigger**  
  - Type: Gmail Trigger  
  - Role: Automatically polls Gmail for new emails with subject containing "透過 Uber Eats 系統送出的訂單".  
  - Configuration:  
    - Filter query: `subject:透過 Uber Eats 系統送出的訂單`  
    - Polling frequency: Every hour at 30 minutes past the hour  
    - Credentials: Linked Gmail OAuth2 account  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "Loop Over Items" node  
  - Edge Cases:  
    - Gmail API rate limits or authentication failures  
    - No matching emails found (workflow idle)  
    - Changes in email subject format may cause missed triggers

---

#### 1.2 Email Retrieval

**Overview:**  
Fetches all emails from Gmail matching the specified subject filter, either triggered manually or automatically.

**Nodes Involved:**  
- Get emails from Gmail with certain subject

**Node Details:**

- **Get emails from Gmail with certain subject**  
  - Type: Gmail node (operation: getAll)  
  - Role: Retrieves all emails matching the subject filter `subject:透過 Uber Eats 系統送出的訂單`.  
  - Configuration:  
    - Filter query: `subject:透過 Uber Eats 系統送出的訂單`  
    - Return all matching emails (no limit)  
    - Credentials: Linked Gmail OAuth2 account  
  - Inputs: From "Click to Test Flow" manual trigger  
  - Outputs: Connects to "Loop Over Items" node  
  - Edge Cases:  
    - Large number of emails may cause performance issues  
    - Gmail API errors or credential expiration  
    - Emails with unexpected content format may affect downstream parsing

---

#### 1.3 Data Extraction

**Overview:**  
Processes each email individually to extract order price, shop name, order date, and time using regular expressions on the email text content.

**Nodes Involved:**  
- Loop Over Items  
- Extract Price, Shop, Date, TIme

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each email item to process them one by one downstream.  
  - Configuration: Default batch options (no specific batch size set)  
  - Inputs: From "Get emails from Gmail with certain subject" or "Receive certain keyword Gmail Trigger"  
  - Outputs: Two outputs: main (processed items) and secondary (unused)  
  - Edge Cases:  
    - Large batch sizes may cause memory or timeout issues  
    - Empty input array leads to no processing

- **Extract Price, Shop, Date, TIme**  
  - Type: Set node  
  - Role: Extracts key order details from the email text using JavaScript expressions with regex matching.  
  - Configuration: Assigns four string fields:  
    - `price`: Extracted by regex matching a dollar amount pattern `\$(\d+(\.\d{2})?)` from the email text.  
    - `shop`: Extracted by matching the phrase "以下是您在[店名]訂購" capturing the shop name (supports Chinese characters, letters, numbers, spaces).  
    - `date`: Extracted from a date string like `Date: 2024年1月1日`, then reformatted to `2024.1.1`.  
    - `time`: Extracted from time strings with "上午" or "下午" and converted to 24-hour format (e.g., "下午 2:30" → "14:30").  
  - Key Expressions:  
    - Uses multiple regex matches on `$json["text"]` field.  
    - Conditional logic to convert AM/PM to 24-hour time.  
  - Inputs: From "Loop Over Items" node  
  - Outputs: Connects to "Send to Slack with Block" node  
  - Edge Cases:  
    - Regex may fail if email format changes or text field is missing  
    - Null or undefined matches cause expression errors  
    - Time conversion assumes specific Chinese AM/PM wording  
    - Emails without expected patterns will produce null or incorrect values

---

#### 1.4 Slack Notification

**Overview:**  
Sends a structured Slack message to a designated channel with the extracted order details and includes a clickable button linking to the Moze accounting app URL scheme for quick expense entry.

**Nodes Involved:**  
- Send to Slack with Block

**Node Details:**

- **Send to Slack with Block**  
  - Type: Slack node (messageType: block)  
  - Role: Sends a formatted Slack message containing order info and a Moze app link button.  
  - Configuration:  
    - Text fallback message includes shop, price, date, and Moze URL scheme link.  
    - Blocks UI JSON defines:  
      - Section block with markdown text showing order details (shop, price, date)  
      - Divider block  
      - Section block with a button accessory labeled "記帳" (Record Expense)  
      - Button URL: `moze3://expense?amount={{ price }}&account=信用卡&subcategory=外送&store={{ shop }}&date={{ date }}&time={{ time }}&project=生活開銷`  
    - Channel: Slack channel ID `C0883CJM1UH` (associated with food delivery notifications)  
    - Authentication: OAuth2 Slack credentials linked  
  - Inputs: From "Extract Price, Shop, Date, TIme" node  
  - Outputs: None (terminal node)  
  - Edge Cases:  
    - Slack API rate limits or authentication failures  
    - Invalid or missing extracted data may cause malformed messages or broken URLs  
    - Moze app URL scheme depends on correct formatting and user device support  
    - Channel ID must be valid and accessible by the bot user

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                          | Input Node(s)                         | Output Node(s)                    | Sticky Note                                                                                              |
|-----------------------------------|-----------------------|----------------------------------------|-------------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------|
| Click to Test Flow                 | Manual Trigger        | Manual start for testing or immediate run | None                                | Get emails from Gmail with certain subject |                                                                                                        |
| Receive certain keyword Gmail Trigger | Gmail Trigger         | Automatic trigger on new matching emails | None                                | Loop Over Items                  |                                                                                                        |
| Get emails from Gmail with certain subject | Gmail                 | Retrieve all emails with specific subject | Click to Test Flow                  | Loop Over Items                  |                                                                                                        |
| Loop Over Items                   | SplitInBatches        | Iterate over each email for processing  | Get emails from Gmail with certain subject, Receive certain keyword Gmail Trigger | Extract Price, Shop, Date, TIme |                                                                                                        |
| Extract Price, Shop, Date, TIme   | Set                   | Extract order details via regex          | Loop Over Items                     | Send to Slack with Block         |                                                                                                        |
| Send to Slack with Block          | Slack                 | Send formatted Slack notification with Moze app link | Extract Price, Shop, Date, TIme     | None                            | Contains a clickable button with Moze app URL scheme for quick expense tracking. See [Moze Documentation](https://moze.app/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To allow manual execution for testing.  
   - No parameters needed.

2. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Credentials: Connect your Gmail OAuth2 credentials.  
   - Parameters:  
     - Filters → Query: `subject:透過 Uber Eats 系統送出的訂單`  
     - Polling: Every hour at 30 minutes past the hour.

3. **Create Gmail Node to Retrieve Emails**  
   - Type: Gmail  
   - Credentials: Use the same Gmail OAuth2 credentials.  
   - Parameters:  
     - Operation: Get All  
     - Filters → Query: `subject:透過 Uber Eats 系統送出的訂單`  
     - Return All: True

4. **Create SplitInBatches Node ("Loop Over Items")**  
   - Type: SplitInBatches  
   - Purpose: To process each email individually.  
   - Default batch size is acceptable.

5. **Create Set Node ("Extract Price, Shop, Date, TIme")**  
   - Type: Set  
   - Purpose: Extract order details from email text using expressions.  
   - Parameters: Add the following fields with expressions:  
     - `price` (string): `={{ $json["text"].match(/\$(\d+(\.\d{2})?)/)[1] }}`  
     - `shop` (string): `={{ $json["text"].match(/以下是您在([\u4e00-\u9fa5a-zA-Z0-9\s]+)訂購/)[1] }}`  
     - `date` (string): `={{ $json["text"].match(/Date: (\d{4}年\d{1,2}月\d{1,2}日)/)[1].replace("年", ".").replace("月", ".").replace("日", "") }}`  
     - `time` (string):  
       ```
       ={{ 
         $json["text"].match(/(上午|下午) (\d{1,2}):(\d{2})/) ? 
         ($json["text"].match(/(上午|下午) (\d{1,2}):(\d{2})/)[1] === '下午' && $json["text"].match(/(上午|下午) (\d{1,2}):(\d{2})/)[2] !== '12' 
           ? (parseInt($json["text"].match(/(上午|下午) (\d{1,2}):(\d{2})/)[2]) + 12) + ':' + $json["text"].match(/(上午|下午) (\d{1,2}):(\d{2})/)[3] 
           : $json["text"].match(/(上午|下午) (\d{1,2}):(\d{2})/)[2] + ':' + $json["text"].match(/(上午|下午) (\d{1,2}):(\d{2})/)[3]
         )
         : null 
       }}
       ```

6. **Create Slack Node ("Send to Slack with Block")**  
   - Type: Slack  
   - Credentials: Connect Slack OAuth2 credentials.  
   - Parameters:  
     - Message Type: Block  
     - Channel: Select or enter channel ID `C0883CJM1UH` (or your own channel)  
     - Text (fallback):  
       ```
       Ubereat 訂餐資訊: 
       商家:  {{ $json.shop }}
       金額: {{ $json.price }}
       日期: {{ $json.date }}

       記帳網址:
       moze3://expense?amount={{ $json.price }}&account=信用卡&subcategory=外送&store={{ $json.shop }}&date={{ $json.date }}
       ```  
     - Blocks UI (JSON):  
       ```json
       {
         "blocks": [
           {
             "type": "section",
             "text": {
               "type": "mrkdwn",
               "text": "Ubereat 訂餐資訊:\n\n*商家:* {{ $json.shop }}\n*金額:* {{ $json.price }}\n*日期:* {{ $json.date }}"
             }
           },
           {
             "type": "divider"
           },
           {
             "type": "section",
             "text": {
               "type": "mrkdwn",
               "text": "Moze 記帳請點我"
             },
             "accessory": {
               "type": "button",
               "text": {
                 "type": "plain_text",
                 "text": "記帳",
                 "emoji": true
               },
               "value": "click",
               "url": "moze3://expense?amount={{ $json.price }}&account=信用卡&subcategory=外送&store={{ $json.shop }}&date={{ $json.date }}&project=生活開銷&time={{ $json.time }}",
               "action_id": "button-action"
             }
           }
         ]
       }
       ```

7. **Connect Nodes**  
   - Connect "Click to Test Flow" → "Get emails from Gmail with certain subject"  
   - Connect "Receive certain keyword Gmail Trigger" → "Loop Over Items"  
   - Connect "Get emails from Gmail with certain subject" → "Loop Over Items"  
   - Connect "Loop Over Items" → "Extract Price, Shop, Date, TIme"  
   - Connect "Extract Price, Shop, Date, TIme" → "Send to Slack with Block"

8. **Credentials Setup**  
   - Gmail OAuth2 credentials must be configured and authorized for both Gmail nodes.  
   - Slack OAuth2 credentials must be configured and authorized for Slack node.

9. **Testing and Validation**  
   - Test manual trigger to verify email retrieval and Slack notification.  
   - Validate regex extraction with sample emails.  
   - Confirm Slack message formatting and Moze app link functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                          |
|----------------------------------------------------------------------------------------------|----------------------------------------|
| Adjust the Gmail subject filter query to match other food delivery platforms or languages.  | Gmail node filter configuration        |
| Ensure regular expressions match the actual email content format to avoid extraction errors.| Data Extraction block                   |
| Moze app URL scheme allows pre-filling expense data for quick entry in the Moze accounting app. | [Moze Documentation](https://moze.app/) |
| Slack message includes a button with a URL scheme that requires the user’s device to support the Moze app URL scheme. | Slack Notification block                |
| Polling frequency for Gmail trigger is set to every hour at 30 minutes; adjust as needed.   | Gmail Trigger node                      |
| Use sample emails to test regex extraction before deploying workflow in production.          | Workflow testing recommendation        |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and maintaining the "Food Delivery Notifications and Easy Expense Tracking" workflow in n8n.