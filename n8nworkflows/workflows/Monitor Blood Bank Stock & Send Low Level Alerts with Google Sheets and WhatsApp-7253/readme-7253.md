Monitor Blood Bank Stock & Send Low Level Alerts with Google Sheets and WhatsApp

https://n8nworkflows.xyz/workflows/monitor-blood-bank-stock---send-low-level-alerts-with-google-sheets-and-whatsapp-7253


# Monitor Blood Bank Stock & Send Low Level Alerts with Google Sheets and WhatsApp

### 1. Workflow Overview

This workflow is designed to monitor blood stock levels in a blood bank using data maintained in a Google Sheet and send alert messages via WhatsApp when specific blood groups fall below predefined thresholds. It targets healthcare or blood bank administrators who need automated daily notifications to maintain adequate blood supplies.

The workflow is organized into the following logical blocks:

- **1.1 Scheduled Trigger**: Runs the workflow daily at a specified time.
- **1.2 Data Retrieval**: Fetches current blood stock data from a Google Sheet.
- **1.3 Data Processing**: Splits the data into manageable batches and evaluates stock levels against thresholds.
- **1.4 Alert Notification**: Sends WhatsApp alerts if any blood group’s stock is low.
- **1.5 Documentation**: Includes a sticky note summarizing the workflow process and Google Sheet columns.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow execution daily at 8:00 AM to ensure regular monitoring.

- **Nodes Involved:**  
  - Daily Check Blood Stock

- **Node Details:**  

  - **Daily Check Blood Stock**  
    - **Type:** Cron Trigger  
    - **Role:** Scheduled trigger to start the workflow daily.  
    - **Configuration:** Triggers once daily at 08:00 (hour=8). No minutes specified, so defaults to 0.  
    - **Input/Output:** No input; output triggers next node.  
    - **Edge Cases:** Cron job misconfiguration could cause missed runs; system downtime at trigger time may delay execution.

#### 1.2 Data Retrieval

- **Overview:**  
  Fetches the current blood stock data from a designated Google Sheet using service account authentication.

- **Nodes Involved:**  
  - Fetch Blood Stock

- **Node Details:**  

  - **Fetch Blood Stock**  
    - **Type:** Google Sheets  
    - **Role:** Reads blood stock data from Google Sheet range "Stock!A:E".  
    - **Configuration:**  
      - Sheet ID placeholder (“{{your_google_sheet_id}}”) to be replaced with actual sheet ID.  
      - Range: Stock!A:E (columns expected: Blood Type, Quantity, Threshold, Last Updated, Status).  
      - Authentication via service account credentials.  
    - **Input/Output:** Input from trigger; outputs rows read from sheet.  
    - **Edge Cases:**  
      - Incorrect sheet ID or permissions cause failure.  
      - Empty or malformed data may cause downstream errors.

#### 1.3 Data Processing

- **Overview:**  
  Processes the fetched stock data by splitting it into batches and checking each blood group's stock against its threshold to identify low stock levels.

- **Nodes Involved:**  
  - Get All Stock  
  - Check Stock Availability

- **Node Details:**  

  - **Get All Stock**  
    - **Type:** SplitInBatches  
    - **Role:** Splits large dataset into manageable parts (batch size not explicitly set, so defaults may apply).  
    - **Configuration:** No additional options set; default batch size applies.  
    - **Input/Output:** Receives data from Google Sheets node; outputs batches to next node.  
    - **Edge Cases:** Large datasets may cause performance issues if batch size is too large; batching logic errors could cause data loss or duplication.

  - **Check Stock Availability**  
    - **Type:** Code (JavaScript)  
    - **Role:** Checks each blood stock item against its threshold and collects those that are low.  
    - **Configuration:**  
      - Uses JavaScript to iterate over input items.  
      - Threshold is taken from each item’s `threshold` field or defaults to 10 units if missing.  
      - Constructs alert messages for blood groups below or equal to their threshold.  
      - Returns an array of JSON objects representing low stock alerts.  
    - **Key Expressions:**  
      - `const threshold = bloodStock.threshold || 10;`  
      - Conditional check: `if (bloodStock.quantity <= threshold)`  
    - **Input/Output:** Receives batches from SplitInBatches; outputs array of low stock alerts.  
    - **Edge Cases:**  
      - Missing or non-numeric quantity/threshold fields could cause logic errors.  
      - If no low stock items found, returns empty array, which downstream nodes must handle gracefully.

#### 1.4 Alert Notification

- **Overview:**  
  Sends WhatsApp alert messages to a predefined phone number if any blood group stock is low, including details and timestamp.

- **Nodes Involved:**  
  - Send Alert Message

- **Node Details:**  

  - **Send Alert Message**  
    - **Type:** WhatsApp  
    - **Role:** Sends alert message to notify about low blood stock.  
    - **Configuration:**  
      - Message body template includes current timestamp and list of low blood groups.  
      - Phone number ID and recipient phone number are hardcoded (placeholders should be replaced).  
      - Uses WhatsApp API credentials named "WhatsApp-test".  
    - **Expressions:**  
      - Message body uses expressions like:  
        ```
        =⚠️ Blood Stock Alert – {{ $now }}
        
        Blood group(s) low or unavailable:
        {{ $json.lowBloodList }}
        ```
      - Phone numbers set as: `=+919998887765` (sender) and `=+919988223377` (recipient).  
    - **Input/Output:**  
      - Input: List of low stock blood groups from the previous node.  
      - Output: Status of message sending (usually success/failure).  
    - **Edge Cases:**  
      - WhatsApp API authentication failures.  
      - Rate limits or message quota exceeded.  
      - Empty low stock list should ideally skip sending message but implementation depends on upstream filtering.  
      - Invalid phone numbers or formatting could cause message failures.

#### 1.5 Documentation

- **Overview:**  
  Provides a sticky note summarizing the workflow process steps and Google Sheet data columns for clarity.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  

  - **Sticky Note**  
    - **Type:** Sticky Note  
    - **Role:** Documentation within the workflow interface.  
    - **Content:**  
      - Lists workflow steps: Daily trigger, data fetch, data batching, stock check, alert sending.  
      - Describes Google Sheet columns: Blood Type, Quantity, Threshold, Last Updated, Status.  
    - **Position:** Top-left of workflow canvas.  
    - **Edge Cases:** None (purely informational).

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                      | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                              |
|------------------------|--------------------|------------------------------------|-----------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Daily Check Blood Stock | Cron Trigger       | Scheduled daily workflow start     | -                     | Fetch Blood Stock            | Workflow Process: 1. Daily Check Blood Stock → 2. Fetch Blood Stock → 3. Get All Stock → 4. Check Stock Availability → 5. Send Alert Message |
| Fetch Blood Stock       | Google Sheets      | Reads blood stock data from sheet  | Daily Check Blood Stock| Get All Stock                | Workflow Process: 1. Daily Check Blood Stock → 2. Fetch Blood Stock → 3. Get All Stock → 4. Check Stock Availability → 5. Send Alert Message |
| Get All Stock           | SplitInBatches     | Splits data into batches            | Fetch Blood Stock      | Check Stock Availability     | Workflow Process: 1. Daily Check Blood Stock → 2. Fetch Blood Stock → 3. Get All Stock → 4. Check Stock Availability → 5. Send Alert Message |
| Check Stock Availability| Code (JavaScript)  | Checks for low stock levels         | Get All Stock          | Send Alert Message           | Workflow Process: 1. Daily Check Blood Stock → 2. Fetch Blood Stock → 3. Get All Stock → 4. Check Stock Availability → 5. Send Alert Message |
| Send Alert Message      | WhatsApp           | Sends WhatsApp alert messages       | Check Stock Availability| -                          | Workflow Process: 1. Daily Check Blood Stock → 2. Fetch Blood Stock → 3. Get All Stock → 4. Check Stock Availability → 5. Send Alert Message |
| Sticky Note            | Sticky Note        | Workflow documentation              | -                     | -                           | ## Workflow Process 1. Daily Check Blood Stock: Triggers the workflow daily. 2. Fetch Blood Stock: Reads data from a Google Sheet. 3. Get All Stock: Collects all available blood stock details. 4. Check Stock Availability: Analyzes stock levels for low thresholds. 5. Send Alert Message: Sends WhatsApp alerts if stock is low. Sheet Columns: Blood Type, Quantity, Threshold, Last Updated, Status |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Type: Cron  
   - Name: "Daily Check Blood Stock"  
   - Configure to trigger daily at 08:00 (hour=8). Leave minutes empty or set to 0.

2. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Name: "Fetch Blood Stock"  
   - Configure:  
     - Authentication: Service Account (set up Google API service account credentials beforehand).  
     - Sheet ID: Insert your Google Sheet ID containing blood stock data.  
     - Range: `Stock!A:E` (columns for Blood Type, Quantity, Threshold, Last Updated, Status).  
   - Connect output of "Daily Check Blood Stock" to this node.

3. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Name: "Get All Stock"  
   - No special options needed (default batch size is acceptable).  
   - Connect output of "Fetch Blood Stock" to this node.

4. **Create Code Node for Stock Checking**  
   - Type: Code  
   - Name: "Check Stock Availability"  
   - Paste the following JavaScript code:  
     ```javascript
     const items = $input.all();
     const lowBloodStock = [];

     for (const item of items) {
       const bloodStock = item.json;
       const threshold = bloodStock.threshold || 10;
       if (bloodStock.quantity <= threshold) {
         lowBloodStock.push({
           blood_group: bloodStock.blood_group,
           quantity: bloodStock.quantity,
           threshold: threshold,
           alert_message: `Low blood stock alert: ${bloodStock.blood_group} has only ${bloodStock.quantity} units left (threshold: ${threshold})`
         });
       }
     }

     return lowBloodStock.map(blood => ({ json: blood }));
     ```  
   - Connect output of "Get All Stock" to this node.

5. **Create WhatsApp Node**  
   - Type: WhatsApp  
   - Name: "Send Alert Message"  
   - Configure:  
     - Operation: Send  
     - Text Body:  
       ```
       =⚠️ Blood Stock Alert – {{ $now }}

       Blood group(s) low or unavailable:
       {{ $json.lowBloodList }}

       Please arrange urgent replenishment to ensure availability for patients in need.

       For help, contact the blood bank team.

       Thank you!
       ```  
     - Phone Number ID: Replace with your WhatsApp API sender phone number ID (example: `+919998887765`).  
     - Recipient Phone Number: Replace with recipient phone number (example: `+919988223377`).  
     - Credentials: Configure WhatsApp API credentials with proper authentication.  
   - Connect output of "Check Stock Availability" to this node.

6. **Add Sticky Note for Documentation**  
   - Place a sticky note on the canvas.  
   - Content:  
     ```
     ## Workflow Process
     1. Daily Check Blood Stock: Triggers the workflow daily.
     2. Fetch Blood Stock: Reads data from a Google Sheet.
     3. Get All Stock: Collects all available blood stock details.
     4. Check Stock Availability: Analyzes stock levels for low thresholds.
     5. Send Alert Message: Sends WhatsApp alerts if stock is low.

     ## Sheet Columns
     - Blood Type: Type of blood (e.g., A+, O-).
     - Quantity: Current stock amount.
     - Threshold: Minimum acceptable stock level.
     - Last Updated: Date and time of last update.
     - Status: Current status (e.g., Low, Sufficient).
     ```

7. **Final Connections**  
   - Connect nodes in this order:  
     Daily Check Blood Stock → Fetch Blood Stock → Get All Stock → Check Stock Availability → Send Alert Message.

8. **Test and Validate**  
   - Verify Google Sheets API access and data correctness.  
   - Ensure WhatsApp API credentials are valid and message sending is successful.  
   - Test with sample data including blood groups below threshold to confirm alert triggering.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                            |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Replace all placeholder values such as Google Sheet ID and WhatsApp phone numbers with actual values before use.     | Credentials and configuration setup                         |
| Workflow can be adjusted to monitor other inventory types by modifying Google Sheet columns and code logic accordingly.| Adaptation potential                                        |
| WhatsApp API usage requires appropriate API setup and may have message rate limits or cost implications.             | WhatsApp Business API documentation                         |
| For detailed Google Sheets API setup and service account creation, refer to Google Cloud Console documentation.      | https://developers.google.com/sheets/api/guides/authorizing |
| The workflow assumes data in the Google Sheet is structured with columns: Blood Type, Quantity, Threshold, Last Updated, Status.| Data schema requirements                                   |

---

This completes the detailed analysis, documentation, and reproduction instructions for the "Monitor Blood Bank Stock & Send Low Level Alerts with Google Sheets and WhatsApp" workflow.