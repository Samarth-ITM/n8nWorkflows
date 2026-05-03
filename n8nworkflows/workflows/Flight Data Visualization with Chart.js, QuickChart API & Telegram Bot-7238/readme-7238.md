Flight Data Visualization with Chart.js, QuickChart API & Telegram Bot

https://n8nworkflows.xyz/workflows/flight-data-visualization-with-chart-js--quickchart-api---telegram-bot-7238


# Flight Data Visualization with Chart.js, QuickChart API & Telegram Bot

### 1. Workflow Overview

This workflow implements a Telegram bot that provides interactive flight data visualizations based on a static CSV dataset containing flight records. Users interact via Telegram commands and buttons to select among four different chart types that analyze flight data:

- Top 10 Airlines (Bar Chart)
- Flight Duration Categories (Pie Chart)
- Price Distribution (Doughnut Chart)
- Price Trends by Flight Duration (Line Chart)

The workflow is logically divided into the following blocks:

**1.1 Entry Point & Command Detection**  
Handles incoming Telegram updates/messages, detects the `/start` command, and routes the user either to a welcome menu or to data processing.

**1.2 User Interface Presentation**  
Sends a visually friendly Telegram reply keyboard with labeled buttons for chart selection.

**1.3 Data Loading & Parsing**  
Reads the flight data CSV file from local storage and parses it into JSON records for analysis.

**1.4 Chart Type Routing**  
Based on user selection, routes the flow to one of four chart generation branches.

**1.5 Chart Data Processing & Chart.js Configuration**  
For each chart type, runs JavaScript code to aggregate and transform data, builds Chart.js configuration objects, and generates QuickChart API URLs.

**1.6 Chart Image Retrieval & Delivery**  
Fetches the chart images from QuickChart API using the generated URLs and sends the images back to the user on Telegram, with customized captions including insights.

---

### 2. Block-by-Block Analysis

#### 2.1 Entry Point & Command Detection

- **Overview:** Listens for all Telegram messages and callback queries. Detects if the message text is `/start` to trigger welcome menu or continue to chart generation.
- **Nodes Involved:** `Telegram Trigger`, `Check Start`
- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Entry point for Telegram updates (messages and callback queries)  
    - Configuration: Listens to `message` and `callback_query` update types  
    - Inputs: External Telegram client messages  
    - Outputs: JSON with message content and metadata  
    - Edge Cases: Telegram API downtime, malformed messages, webhook errors  

  - **Check Start**  
    - Type: If node  
    - Role: Checks if incoming message text equals `/start`  
    - Configuration: Case-sensitive strict string equality on `{{$json.message.text}}` to `/start`  
    - Inputs: From `Telegram Trigger`  
    - Outputs: Two branches: True (message is `/start`), False (other text)  
    - Edge Cases: Missing or unexpected message text field, case mismatches

#### 2.2 User Interface Presentation

- **Overview:** Sends a welcome message with a reply keyboard presenting four chart options with emojis and labels.
- **Nodes Involved:** `Send Welcome Message`
- **Node Details:**

  - **Send Welcome Message**  
    - Type: Telegram node (send message)  
    - Role: Sends welcome text and an interactive reply keyboard to the user  
    - Configuration:  
      - Text includes greeting and numbered chart options with emojis  
      - `chatId` dynamically set from incoming message chat id  
      - Reply keyboard with two rows and two buttons each, labeled with chart options  
      - Keyboard resize enabled for better mobile UX  
    - Inputs: Triggered from `Check Start` True branch  
    - Outputs: None (terminal message send)  
    - Edge Cases: Telegram API limits, chatId missing, user blocking bot  

#### 2.3 Data Loading & Parsing

- **Overview:** Reads the flight data CSV file from local filesystem and extracts structured JSON records for processing.
- **Nodes Involved:** `Read CSV File`, `Extract from File`
- **Node Details:**

  - **Read CSV File**  
    - Type: Read/Write File node  
    - Role: Reads file `/data/flights.csv` from local storage  
    - Configuration: Default read mode, file selector set to the CSV path  
    - Inputs: From `Check Start` False branch (message not `/start`)  
    - Outputs: Raw CSV file content  
    - Edge Cases: File missing or inaccessible, encoding issues  

  - **Extract from File**  
    - Type: Extract From File node  
    - Role: Parses CSV raw content into JSON object array, each representing a flight record  
    - Configuration: Default CSV parsing options  
    - Inputs: From `Read CSV File`  
    - Outputs: Multiple items, each a flight record JSON  
    - Edge Cases: Malformed CSV, inconsistent columns, empty file

#### 2.4 Chart Type Routing

- **Overview:** Routes the flow to one of four chart generation branches based on the exact text of user’s Telegram button selection.
- **Nodes Involved:** `Switch`
- **Node Details:**

  - **Switch**  
    - Type: Switch node  
    - Role: Routes based on the exact message text from Telegram Trigger  
    - Configuration: Four rules matching exact strings for the four chart types:
      - "1️⃣ Top Airlines (Bar Chart)" → output "bar"  
      - "2️⃣ Flight Duration Categories (Pie Chart)" → output "pie"  
      - "3️⃣ Price Distribution (Doughnut Chart)" → output "doughnut"  
      - "4️⃣ Price Trends (Line Plot)" → output "line"  
    - Inputs: From `Extract from File`  
    - Outputs: Four separate branches for each chart  
    - Edge Cases: User input not matching any button text (no route), case sensitivity issues

#### 2.5 Chart Data Processing & Chart.js Configuration

- **Overview:** Each chart branch has a Code node that processes the flight data JSON, computes statistics, generates a Chart.js configuration object, and creates a QuickChart API URL.
- **Nodes Involved:**  
  - `Process Data & Create Bar Chart`  
  - `Process Data & Create Pie Chart`  
  - `Process Data & Create Doughnut Chart`  
  - `Process Data & Create Line Chart`
  
- **Node Details:**

  - **Process Data & Create Bar Chart**  
    - Type: Code node (JavaScript)  
    - Role: Counts flights by airline, extracts top 10, generates bar chart config and QuickChart URL  
    - Key Logic: Aggregates airline flight counts, sorts descending, builds bar chart with blue/orange color palette  
    - Outputs: chatId, quickChartUrl, insights array with top airline stats  
    - Edge Cases: Missing airline field, empty data, too few airlines  

  - **Process Data & Create Pie Chart**  
    - Type: Code node  
    - Role: Categorizes flights by duration (short/medium/long haul), builds pie chart config and URL  
    - Key Logic: Parses duration from multiple possible fields, handles missing data with random distribution for demo  
    - Outputs: chatId, quickChartUrl, insights, durationBreakdown object  
    - Edge Cases: Missing or invalid duration, random category assignment may confuse real users  

  - **Process Data & Create Doughnut Chart**  
    - Type: Code node  
    - Role: Counts flights by price ranges (budget to luxury), creates doughnut chart config and URL  
    - Key Logic: Parses price, categorizes into five ranges, calculates average price for center label  
    - Outputs: chatId, quickChartUrl, insights, priceBreakdown  
    - Edge Cases: Missing or zero price, range overlaps handled by if-else order  

  - **Process Data & Create Line Chart**  
    - Type: Code node  
    - Role: Groups flights by duration ranges, calculates average price per range, creates line chart config and URL  
    - Key Logic: Bins durations, averages prices, filters out empty bins, identifies price trend direction  
    - Outputs: chatId, quickChartUrl, insights, trendData array  
    - Edge Cases: Missing duration or price, 0 values filtered, trend calculation sensitive to data noise  

#### 2.6 Chart Image Retrieval & Delivery

- **Overview:** For each chart type, fetches the chart image PNG from the QuickChart API and sends it as a photo with caption to the user’s Telegram chat.
- **Nodes Involved:**  
  - `Fetch Bar Chart Image`, `Send Bar Chart to Telegram`  
  - `Fetch Pie Chart Image`, `Send Pie Chart to Telegram`  
  - `Fetch Doughnut Chart Image`, `Send Doughnut Chart to Telegram`  
  - `Fetch Line Chart Image`, `Send Line Chart to Telegram`
  
- **Node Details:**

  - **Fetch * Chart Image nodes**  
    - Type: HTTP Request node  
    - Role: Downloads PNG image from QuickChart API using generated URL  
    - Configuration: URL parameterized from prior node’s quickChartUrl field, never error on response  
    - Outputs: Binary image data  
    - Edge Cases: QuickChart API failures, network timeout, very long URLs (some chart configs are large)  

  - **Send * Chart to Telegram nodes**  
    - Type: Telegram node (send photo)  
    - Role: Sends the fetched chart PNG to the Telegram chat  
    - Configuration:  
      - chatId from corresponding process node output  
      - Operation: sendPhoto with binary data enabled  
      - Caption contains custom insights summary with emojis and user prompts  
    - Edge Cases: Telegram media size limits, chatId issues, user blocking bot  

---

### 3. Summary Table

| Node Name                     | Node Type                   | Functional Role                       | Input Node(s)             | Output Node(s)                    | Sticky Note                                                                                                  |
|-------------------------------|-----------------------------|-------------------------------------|---------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger              | telegramTrigger             | Entry point for Telegram updates    |                           | Check Start                     | 📱 ENTRY POINT - Listens for Telegram messages & button clicks; triggers on /start and menu selections       |
| Check Start                  | if                          | Detects `/start` command             | Telegram Trigger          | Send Welcome Message, Read CSV File | 🎯 COMMAND DETECTOR - Smart filter to detect /start, routes to menu or data processing                        |
| Send Welcome Message         | telegram                    | Sends chart selection menu           | Check Start (true branch) |                                 | 🎨 USER INTERFACE - Reply keyboard with emoji buttons, mobile friendly                                     |
| Read CSV File                | readWriteFile               | Reads flight CSV data from disk      | Check Start (false branch)| Extract from File               | 📊 DATA SOURCE - Reads /data/flights.csv, ~1000 records, UTF-8 encoding                                     |
| Extract from File            | extractFromFile             | Parses CSV raw data to JSON          | Read CSV File             | Switch                         | 📊 DATA PARSER - Converts CSV rows to individual JSON flight records                                        |
| Switch                      | switch                      | Routes according to user’s chart choice | Extract from File         | 4 Chart Process nodes          | 🎛️ TRAFFIC CONTROLLER - Routes to correct chart type by exact string match                                  |
| Process Data & Create Bar Chart | code                      | Aggregates airline counts, builds bar chart config | Switch (bar output)       | Fetch Bar Chart Image           | 🎨 CHART GENERATOR - Bar Chart: Top 10 Airlines by flight count                                            |
| Fetch Bar Chart Image        | httpRequest                 | Downloads bar chart image from QuickChart | Process Data & Create Bar Chart | Send Bar Chart to Telegram     |                                                                                                              |
| Send Bar Chart to Telegram   | telegram                    | Sends bar chart image to user        | Fetch Bar Chart Image      |                                 |                                                                                                              |
| Process Data & Create Pie Chart | code                      | Categorizes by flight duration, builds pie chart config | Switch (pie output)       | Fetch Pie Chart Image           | 🎨 CHART GENERATOR - Pie Chart: Flight Duration Categories                                                |
| Fetch Pie Chart Image        | httpRequest                 | Downloads pie chart image from QuickChart | Process Data & Create Pie Chart | Send Pie Chart to Telegram      |                                                                                                              |
| Send Pie Chart to Telegram   | telegram                    | Sends pie chart image to user        | Fetch Pie Chart Image      |                                 |                                                                                                              |
| Process Data & Create Doughnut Chart | code                 | Categorizes by price ranges, builds doughnut chart config | Switch (doughnut output)  | Fetch Doughnut Chart Image      | 🎨 CHART GENERATOR - Doughnut Chart: Price Distribution                                                   |
| Fetch Doughnut Chart Image   | httpRequest                 | Downloads doughnut chart image from QuickChart | Process Data & Create Doughnut Chart | Send Doughnut Chart to Telegram |                                                                                                              |
| Send Doughnut Chart to Telegram | telegram                  | Sends doughnut chart image to user   | Fetch Doughnut Chart Image |                                 |                                                                                                              |
| Process Data & Create Line Chart | code                     | Analyzes price trends by duration, builds line chart config | Switch (line output)      | Fetch Line Chart Image          | 🎨 CHART GENERATOR - Line Chart: Price Trends by Flight Duration                                         |
| Fetch Line Chart Image       | httpRequest                 | Downloads line chart image from QuickChart | Process Data & Create Line Chart | Send Line Chart to Telegram     |                                                                                                              |
| Send Line Chart to Telegram  | telegram                    | Sends line chart image to user       | Fetch Line Chart Image     |                                 |                                                                                                              |
| Sticky Note                  | stickyNote                  | Documentation note                   |                           |                                 | 📱 ENTRY POINT - First user interaction node                                                                |
| Sticky Note1                 | stickyNote                  | Documentation note                   |                           |                                 | 🎯 COMMAND DETECTOR - /start command detection and routing logic                                            |
| Sticky Note2                 | stickyNote                  | Documentation note                   |                           |                                 | 🎨 USER INTERFACE - Welcome menu design                                                                     |
| Sticky Note3                 | stickyNote                  | Documentation note                   |                           |                                 | 📊 DATA SOURCE & PARSER - CSV file reading and parsing explanation                                         |
| Sticky Note4                 | stickyNote                  | Documentation note                   |                           |                                 | 🎛️ TRAFFIC CONTROLLER - Switch routing explanation                                                         |
| Sticky Note5                 | stickyNote                  | Documentation note                   |                           |                                 | 🎨 CHART GENERATOR - Overview of four chart types and processing flow                                      |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Telegram Trigger Node**  
- Node Type: Telegram Trigger  
- Configure to listen for updates type: `message` and `callback_query`  
- No additional fields needed  
- This node is the workflow entry point.

**Step 2: Add If Node "Check Start"**  
- Node Type: If  
- Condition: Check if expression `{{$json.message.text}}` equals `/start` (case-sensitive)  
- Connect output of Telegram Trigger to this node.

**Step 3: Configure "Send Welcome Message" Telegram Node**  
- Node Type: Telegram (send message)  
- Text:  
  ```
  ✈️ Welcome to Flight Data Analytics Bot!

  Choose your visualization:
  1️⃣ Top Airlines (Bar Chart)
  2️⃣ Flight Duration Categories (Pie Chart)
  3️⃣ Price Distribution (Doughnut Chart)
  4️⃣ Price Trends (Line Plot)
  ```  
- chatId: `{{$json.message.chat.id}}`  
- Reply Markup: replyKeyboard with two rows, two buttons each:
  - Row 1: "1️⃣ Top Airlines (Bar Chart)", "2️⃣ Flight Duration Categories (Pie Chart)"  
  - Row 2: "3️⃣ Price Distribution (Doughnut Chart)", "4️⃣ Price Trends (Line Plot)"  
- Enable keyboard resize option  
- Connect output True of Check Start to this node.

**Step 4: Add Read File Node "Read CSV File"**  
- Node Type: Read/Write File  
- File path: `/data/flights.csv` (ensure this file exists locally with UTF-8 encoding)  
- Connect output False of Check Start to this node.

**Step 5: Add Extract From File Node "Extract from File"**  
- Node Type: Extract From File  
- Default CSV extraction options  
- Connect output of Read CSV File to this node.

**Step 6: Add Switch Node "Switch"**  
- Node Type: Switch  
- Add four rules to match exact string in `{{$('Telegram Trigger').item.json.message.text}}`:  
  - "1️⃣ Top Airlines (Bar Chart)" → output "bar"  
  - "2️⃣ Flight Duration Categories (Pie Chart)" → output "pie"  
  - "3️⃣ Price Distribution (Doughnut Chart)" → output "doughnut"  
  - "4️⃣ Price Trends (Line Plot)" → output "line"  
- Connect output of Extract from File to this node.

**Step 7: For Each Chart Type, Add a Code Node to Process Data**  

- Create four separate code nodes:  
  - "Process Data & Create Bar Chart"  
  - "Process Data & Create Pie Chart"  
  - "Process Data & Create Doughnut Chart"  
  - "Process Data & Create Line Chart"  

- Use provided JavaScript code logic in each node to:  
  - Aggregate data appropriately (airline counts, duration categories, price ranges, price trends)  
  - Build Chart.js configuration objects for each chart type  
  - Create QuickChart API URLs embedding the Chart.js config  
  - Produce outputs: `chatId`, `quickChartUrl`, `insights`, and breakdown data as applicable  

- Connect corresponding outputs of Switch node to each respective code node.

**Step 8: For Each Chart Process Node, Add HTTP Request Node to Fetch Chart Image**  

- Node Type: HTTP Request  
- URL: `={{ $json.quickChartUrl }}`  
- Response: Set to return binary data (image)  
- Enable "Never Error" to handle fetch failures gracefully  
- Connect each code node output to its corresponding HTTP Request node.

**Step 9: For Each HTTP Request Node, Add Telegram Node to Send Photo**  

- Node Type: Telegram  
- Operation: `sendPhoto`  
- chatId: `={{ $('Process Data & Create ... Chart').first().json.chatId }}` (match correct process node)  
- Send binary data enabled  
- Caption: Custom text including key insights, e.g.,  
  - Bar Chart: "Vistara soars high with 350+ flights, leading the pack! ✈️\n\n/start again?"  
  - Pie Chart: "Long-haul dominates with 611 flights, while short-haul adds 279! 🌍\n\nDo you need /start?"  
  - Doughnut Chart: "Budget rules with 475 bookings under ₹10K! 💸\n\n/start again?"  
  - Line Chart: "Average prices peak at 14K for 6-8 hour flights! 📈\n\n/start ?"  
- Connect each HTTP node output to its respective Telegram node.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                  |
|------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Ensure the `/data/flights.csv` file exists and is accessible by n8n with UTF-8 encoding.                               | Data source for all analytics                                   |
| The workflow uses QuickChart API for serverless chart rendering: https://quickchart.io/                                | QuickChart API for dynamic chart image generation               |
| Telegram bot must be configured with proper OAuth2 credentials and webhook setup in n8n for Telegram Trigger and Telegram nodes | Telegram API integration requirements                           |
| Usage of reply keyboards improves mobile user experience and interaction simplicity.                                  | UX design in Telegram bots                                      |
| Chart.js configuration objects are dynamically built in JavaScript code nodes, allowing rich customization.           | Chart.js docs: https://www.chartjs.org/docs/latest/             |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, node configurations, and stepwise reproduction instructions for both advanced users and AI automation agents.