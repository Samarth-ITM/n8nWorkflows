Smart IoT Device Health Monitor with ScrapeGraphAI and Telegram

https://n8nworkflows.xyz/workflows/smart-iot-device-health-monitor-with-scrapegraphai-and-telegram-6930


# Smart IoT Device Health Monitor with ScrapeGraphAI and Telegram

### 1. Workflow Overview

This workflow, titled **Smart IoT Device Health Monitor with ScrapeGraphAI and Telegram**, is designed to automate monitoring of IoT devices via a web dashboard, analyze device health with AI, alert users through Telegram when issues arise, and log all monitoring activity for trend tracking.

**Target Use Cases:**  
- Continuous monitoring of IoT devices without needing direct API access (scrapes dashboard via AI).  
- Real-time health checks including device status, battery, temperature, and connectivity.  
- Automated alerting to operators or administrators through Telegram messaging when device health degrades or problems are detected.  
- Historical logging for later analysis or troubleshooting.

**Logical Blocks:**  
- **1.1 Input Reception:** Timer trigger that initiates the workflow every 30 minutes.  
- **1.2 AI Data Extraction:** Uses ScrapeGraphAI to scrape and parse device data from the IoT dashboard URL.  
- **1.3 Device Health Analysis:** Custom code node that processes scraped data to compute health scores and identify issues.  
- **1.4 Alert Decision:** Conditional node to determine if an alert should be sent based on health thresholds and detected problems.  
- **1.5 Alert Notification:** Sends formatted alerts to a Telegram chat if necessary.  
- **1.6 Logging:** Logs the results of each monitoring cycle for record-keeping.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block defines the schedule for periodic monitoring. It triggers the workflow every 30 minutes to initiate data extraction.

**Nodes Involved:**  
- ⏰ Timer  
- ⏰ Timer Info (sticky note)

**Node Details:**  

- **⏰ Timer**  
  - *Type/Role:* Schedule Trigger  
  - *Config:* Cron expression set to trigger every 30 minutes (`*/30 * * * *`), timezone aware.  
  - *Input/Output:* No input; outputs trigger event to "🤖 Get Data" node.  
  - *Edge Cases:* Misconfiguration of cron could cause missed or too frequent triggers. Timezone mismatches may cause unexpected run times.

- **⏰ Timer Info**  
  - *Type:* Sticky Note (documentation)  
  - *Role:* Explains cron configuration options and scheduling details.  
  - *No inputs or outputs.*

---

#### 1.2 AI Data Extraction

**Overview:**  
This block scrapes IoT dashboard data using AI, extracting device information such as status, battery level, temperature, and connectivity into structured JSON without needing an API.

**Nodes Involved:**  
- 🤖 Get Data (ScrapeGraphAI node)  
- 🤖 Scraper Info (sticky note)

**Node Details:**  

- **🤖 Get Data**  
  - *Type/Role:* ScrapeGraphAI node — AI-powered web scraping and data extraction.  
  - *Config:*  
    - `userPrompt` instructs AI to extract JSON with device info (id, name, status, battery, temperature) and a summary count of devices.  
    - `websiteUrl` points to the IoT dashboard URL (placeholder `https://your-iot-dashboard.com/devices`).  
  - *Input:* Trigger from Timer node.  
  - *Output:* JSON data payload representing device states.  
  - *Edge Cases:*  
    - AI extraction can fail if webpage layout changes, content is dynamic, or site is protected.  
    - Network timeouts or connectivity issues.  
    - Invalid or unreachable dashboard URL.  
  - *Version:* Requires ScrapeGraphAI node to be installed and configured with appropriate credentials.

- **🤖 Scraper Info**  
  - *Type:* Sticky Note  
  - *Role:* Describes the AI scraping step and the types of data extracted.

---

#### 1.3 Device Health Analysis

**Overview:**  
Processes the scraped data, calculates overall health score, identifies problem devices (offline, low battery, overheating), and generates alerts as needed.

**Nodes Involved:**  
- 📊 Analyze (Code node)  
- 📊 Analyzer Info (sticky note)

**Node Details:**  

- **📊 Analyze**  
  - *Type/Role:* Code node (JavaScript) that analyzes device data.  
  - *Config:*  
    - Reads input JSON data array from previous node.  
    - Calculates health score as percentage of online devices.  
    - Detects offline devices, devices with battery < 20%, and temperature > 70°C.  
    - Generates problem messages and alert list based on thresholds.  
    - Logs health score and problem count to console.  
  - *Input:* Scraped JSON data from "🤖 Get Data".  
  - *Output:* JSON object with analysis results including health_score, problems array, alerts array, counts, and timestamp.  
  - *Edge Cases:*  
    - Missing or malformed input data can cause runtime errors.  
    - Division by zero if total_devices is zero handled gracefully.  
    - Unexpected device property types may cause analysis issues.  
  - *Version:* Requires n8n supporting JavaScript Code node version 2.

- **📊 Analyzer Info**  
  - *Type:* Sticky Note  
  - *Role:* Explains analysis goals and outputs.

---

#### 1.4 Alert Decision

**Overview:**  
Conditionally checks if an alert should be sent based on health score and detected problems to avoid unnecessary notifications.

**Nodes Involved:**  
- 🚨 Need Alert? (IF node)  
- 🚨 Alert Info (sticky note)

**Node Details:**  

- **🚨 Need Alert?**  
  - *Type/Role:* IF node that checks alert criteria.  
  - *Config:*  
    - Triggers TRUE branch if either:  
      - `health_score` < 80, OR  
      - Number of problems > 0.  
  - *Input:* Analysis output.  
  - *Output:* TRUE branch leads to sending alert, FALSE branch skips alert but continues logging.  
  - *Edge Cases:*  
    - Missing properties in input JSON could cause condition evaluation errors.  
    - Thresholds are hardcoded; might need tuning for different environments.

- **🚨 Alert Info**  
  - *Type:* Sticky Note  
  - *Role:* Explains alerting logic and thresholds.

---

#### 1.5 Alert Notification

**Overview:**  
Sends a formatted alert message to a specified Telegram chat if problems are detected.

**Nodes Involved:**  
- 📱 Send Alert (Telegram node)  
- 📱 Telegram Info (sticky note)

**Node Details:**  

- **📱 Send Alert**  
  - *Type/Role:* Telegram node for posting messages.  
  - *Config:*  
    - Operation: `post` (send message).  
    - Requires Telegram credentials configured with bot token.  
    - Chat ID must be set (replace placeholder `YOUR_CHAT_ID` with actual chat ID).  
    - Message formatting can include emojis and detailed problem descriptions (not explicitly shown in JSON but implied in sticky note).  
  - *Input:* TRUE branch output from "🚨 Need Alert?".  
  - *Output:* Confirmation or error of message send operation.  
  - *Edge Cases:*  
    - Invalid or missing Telegram credentials or chat ID.  
    - Telegram API rate limits or network errors.  
    - Message formatting errors if variables are malformed.

- **📱 Telegram Info**  
  - *Type:* Sticky Note  
  - *Role:* Instructions on chat ID replacement and message formatting details.

---

#### 1.6 Logging

**Overview:**  
Logs the results of each monitoring cycle for record-keeping and trend tracking, including health scores, device counts, problems, and alert status.

**Nodes Involved:**  
- 📝 Log Data (Code node)  
- 📝 Logger Info (sticky note)

**Node Details:**  

- **📝 Log Data**  
  - *Type/Role:* Code node that prepares log entry.  
  - *Config:*  
    - Extracts key metrics from analysis output: timestamp, health_score, device counts, problem count, and alert sent flag.  
    - Logs summary to console.  
    - Placeholder comment for extending to database or persistent storage insertion.  
  - *Input:* Output from "🚨 Need Alert?" node (both TRUE and FALSE branches).  
  - *Output:* JSON log data object for downstream use or storage.  
  - *Edge Cases:*  
    - Missing input properties cause errors.  
    - No persistence implemented by default; requires user extension.  
  - *Version:* Requires n8n supporting JavaScript Code node version 2.

- **📝 Logger Info**  
  - *Type:* Sticky Note  
  - *Role:* Describes logging purpose and data captured.

---

### 3. Summary Table

| Node Name       | Node Type                   | Functional Role                   | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                  |
|-----------------|-----------------------------|---------------------------------|-----------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| 📋 Overview     | Sticky Note                 | Workflow description             | -                     | -                       | # Simple IoT Monitor 🏭 ... Features: Auto monitoring, smart alerts, device health check, battery, temperature |
| ⏰ Timer Info    | Sticky Note                 | Explains timer scheduling        | -                     | -                       | # Step 1: Timer ⏰ ... Runs every 30 minutes; cron examples included                                        |
| ⏰ Timer         | Schedule Trigger            | Triggers workflow every 30min    | -                     | 🤖 Get Data              |                                                                                                              |
| 🤖 Scraper Info | Sticky Note                 | Describes AI scraping step       | -                     | -                       | # Step 2: Get Data 🤖 ... AI extracts device info without API                                                |
| 🤖 Get Data     | ScrapeGraphAI               | Extract IoT device data via AI   | ⏰ Timer               | 📊 Analyze               |                                                                                                              |
| 📊 Analyzer Info| Sticky Note                 | Describes health analysis        | -                     | -                       | # Step 3: Check Health 📊 ... Calculates health score, problems, alerts                                     |
| 📊 Analyze      | Code Node (JavaScript)      | Analyzes device data             | 🤖 Get Data            | 🚨 Need Alert?, 📝 Log Data|                                                                                                              |
| 🚨 Alert Info   | Sticky Note                 | Describes alert conditions       | -                     | -                       | # Step 4: Send Alert 🚨 ... Smart alerting with spam prevention                                            |
| 🚨 Need Alert?  | IF Node                    | Decides if alert is needed       | 📊 Analyze             | 📱 Send Alert, 📝 Log Data|                                                                                                              |
| 📱 Telegram Info| Sticky Note                 | Explains Telegram alerting       | -                     | -                       | # Step 5: Telegram 📱 ... Replace YOUR_CHAT_ID, get chat ID from @userinfobot                               |
| 📱 Send Alert   | Telegram Node               | Sends alert messages to Telegram | 🚨 Need Alert?         | -                       |                                                                                                              |
| 📝 Logger Info  | Sticky Note                 | Describes logging step           | -                     | -                       | # Step 6: Log Data 📝 ... Logs health score, device counts, problems, timestamps                            |
| 📝 Log Data     | Code Node (JavaScript)      | Logs monitoring results          | 🚨 Need Alert?         | -                       |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**:
   - Name: `⏰ Timer`
   - Set the Trigger to Cron, expression: `*/30 * * * *` (every 30 minutes)
   - Connect no input; this node starts the flow.

3. **Add a ScrapeGraphAI node**:
   - Name: `🤖 Get Data`
   - Configure credentials for ScrapeGraphAI (setup API key as per your environment).  
   - Set parameters:  
     - `websiteUrl`: Your IoT dashboard URL, e.g., `https://your-iot-dashboard.com/devices`  
     - `userPrompt`:  
       ```
       Extract IoT device data as JSON:

       {
         "devices": [
           {
             "id": "device_id",
             "name": "device_name",
             "status": "online/offline/error",
             "battery": 85,
             "temperature": 25
           }
         ],
         "summary": {
           "total": 10,
           "online": 8,
           "offline": 2
         }
       }

       Focus on device health and status.
       ```
   - Connect output of `⏰ Timer` to input of `🤖 Get Data`.

4. **Add a Code node** for analysis:
   - Name: `📊 Analyze`
   - Set language to JavaScript.
   - Paste the provided analysis code (calculates health, collects problems, generates alerts).
   - Connect output of `🤖 Get Data` to input of `📊 Analyze`.

5. **Add an IF node**:
   - Name: `🚨 Need Alert?`
   - Configure conditions:  
     - First condition: `health_score` < 80  
     - OR second condition: `problems.length` > 0  
     - Use expressions:  
       - `{{$json["health_score"]}} < 80`  
       - `{{$json["problems"].length}} > 0`  
   - Connect output of `📊 Analyze` to input of this IF node.

6. **Add a Telegram node** for alerts:
   - Name: `📱 Send Alert`
   - Set operation to `post` message.  
   - Configure Telegram Bot credentials (bot token).  
   - Set `Chat ID` to your Telegram chat ID (replace placeholder).  
   - Craft message text using expressions referencing problems and health score (optional, based on your formatting needs).  
   - Connect TRUE output of `🚨 Need Alert?` to input of `📱 Send Alert`.

7. **Add a Code node** for logging:
   - Name: `📝 Log Data`
   - Set language to JavaScript.  
   - Paste the given logging code that extracts key metrics and logs to console (can extend to save in DB).  
   - Connect both TRUE and FALSE outputs of `🚨 Need Alert?` to input of `📝 Log Data`.

8. **Add Sticky Notes** (optional but recommended for clarity):
   - Add notes for each step explaining purpose, configuration tips, and usage instructions as per the provided sticky notes content.

9. **Test the workflow** by executing manually or waiting for the scheduled trigger.

10. **Adjust parameters**:
    - Change dashboard URL in `🤖 Get Data`.  
    - Update Telegram chat ID in `📱 Send Alert`.  
    - Tune alert thresholds in `🚨 Need Alert?` or in the code node as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Replace `YOUR_CHAT_ID` in Telegram node with actual chat ID obtained via Telegram bot @userinfobot.     | Telegram Info sticky note                                                                            |
| Cron expressions can be customized for different monitoring frequencies (every 5, 15, or 60 minutes).   | Timer Info sticky note                                                                               |
| ScrapeGraphAI enables no-API scraping but depends on webpage structure stability and network availability. | Scraper Info sticky note                                                                             |
| Logs are currently output to console; extend the logging code node to save to a database or file as needed. | Logger Info sticky note                                                                              |
| The workflow prevents alert spam by only sending notifications when health score is below 80% or problems exist. | Alert Info sticky note                                                                               |
| Use n8n version supporting Code node v2 and ScrapeGraphAI node integration for compatibility.           | General technical requirement                                                                         |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.