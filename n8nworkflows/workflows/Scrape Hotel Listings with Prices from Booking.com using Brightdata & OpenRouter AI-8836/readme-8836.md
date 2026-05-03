Scrape Hotel Listings with Prices from Booking.com using Brightdata & OpenRouter AI

https://n8nworkflows.xyz/workflows/scrape-hotel-listings-with-prices-from-booking-com-using-brightdata---openrouter-ai-8836


# Scrape Hotel Listings with Prices from Booking.com using Brightdata & OpenRouter AI

### 1. Workflow Overview

This workflow automates scraping and presenting hotel listings with prices from Booking.com using Bright Data’s scraping services and OpenRouter AI for data refinement. The target use case is to provide a user-friendly hotel list from a city input received via chat, leveraging real-time web scraping combined with AI-powered data formatting and price calculations.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives a chat message with the city to scrape hotels for.
- **1.2 Data Scraping Initiation:** Triggers Bright Data’s web scraper to start the extraction of hotel listings from Booking.com.
- **1.3 Scraping Monitoring & Data Download:** Monitors the scraping job’s status and downloads the snapshot content once ready, looping until completion.
- **1.4 Data Cleaning & Aggregation:** Cleans and structures the raw scraped data, then aggregates it into a single dataset.
- **1.5 AI-Powered Formatting & Calculation:** Uses OpenRouter AI to convert the aggregated data into a human-friendly hotel list, employing a calculator tool for price conversions.
- **1.6 Final Output:** Presents the formatted hotel list as the workflow’s response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for a chat message containing the city name to scrape and sets the input parameter for downstream nodes.

- **Nodes Involved:**  
  - When chat message received  
  - parameters

- **Node Details:**  

  **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Entry point that triggers the workflow upon receiving a chat message.  
  - Configuration: Uses webhook with response mode "lastNode" to return output from the last node in the workflow.  
  - Inputs: External chat message webhook  
  - Outputs: Passes chat input JSON to next node.  
  - Edge cases: Missing or malformed chat input could fail downstream parameter setting.

  **parameters**  
  - Type: Set node  
  - Role: Extracts and assigns the chat input string to a parameter named `city`.  
  - Configuration: Assigns `city` string using expression `={{ $json.chatInput }}` from incoming JSON.  
  - Inputs: From chat trigger node  
  - Outputs: JSON with `city` property forwarded.  
  - Edge cases: If chatInput is empty or invalid, scraping will target an empty city.

#### 2.2 Data Scraping Initiation

- **Overview:**  
  Starts the Bright Data scraping job using the city parameter and predefined search parameters targeting Booking.com hotel listings.

- **Nodes Involved:**  
  - Initiate batch extraction from URL

- **Node Details:**  

  **Initiate batch extraction from URL**  
  - Type: Bright Data node  
  - Role: Triggers a web scraping collection job on Bright Data with specific parameters for Booking.com.  
  - Configuration:  
    - URLs: JSON array with template URL for Booking.com, embedding the `city` parameter dynamically (`{{ $json.city }}`).  
    - Check-in/out dates fixed (Nov 1-8, 2025), 2 adults, 1 child, 1 room, property type "Hostels".  
    - Dataset ID specified for Booking listings search.  
  - Credentials: Bright Data API key required.  
  - Inputs: Receives city parameter JSON.  
  - Outputs: Contains `snapshot_id` for the scraping job.  
  - Edge cases: API key auth failure, invalid dataset ID, invalid city input causing no results.

#### 2.3 Scraping Monitoring & Data Download

- **Overview:**  
  Polls Bright Data to check if the scraping job is ready, loops with wait intervals, then downloads the snapshot data when ready.

- **Nodes Involved:**  
  - hotels (SplitInBatches)  
  - Check the status of a batch extraction  
  - check if result ready (IF)  
  - Download the snapshot content  
  - Wait  
  - check Snapshot Again (NoOp)

- **Node Details:**  

  **hotels**  
  - Type: SplitInBatches  
  - Role: Controls batch processing of items; used here to trigger status check iterations.  
  - Configuration: Reset enabled to start fresh.  
  - Inputs: From batch extraction initiation.  
  - Outputs: On completion triggers status check.  
  - Edge cases: Empty batches or stalled iteration may affect flow.

  **Check the status of a batch extraction**  
  - Type: Bright Data node  
  - Role: Monitors the scraping job’s progress using `snapshot_id` from the initiation node.  
  - Configuration: Uses `monitorProgressSnapshot` operation with dynamic `snapshot_id`.  
  - Credentials: Bright Data API key required.  
  - Inputs: Receives `snapshot_id` from initiation node via expression.  
  - Outputs: JSON containing current status (e.g., "ready" or other).  
  - Edge cases: API errors, invalid snapshot ID, delays.

  **check if result ready**  
  - Type: IF node  
  - Role: Checks if the status returned is "ready".  
  - Configuration: Condition checks if `$json.status` equals "ready".  
  - Inputs: From status check node.  
  - Outputs:  
    - True branch: proceeds to download snapshot content  
    - False branch: loops back to Wait node to delay next status check.  
  - Edge cases: If status never becomes "ready", workflow loops indefinitely.

  **Download the snapshot content**  
  - Type: Bright Data node  
  - Role: Downloads the scraped data once ready.  
  - Configuration: Uses `downloadSnapshot` operation with dynamic `snapshot_id`.  
  - Credentials: Bright Data API key required.  
  - Inputs: From IF node’s true branch.  
  - Outputs: Raw scraped data JSON array.  
  - OnError: Configured to continue on error to avoid complete failure.  
  - Edge cases: Download failures, partial data.

  **Wait**  
  - Type: Wait node  
  - Role: Introduces delay between status checks.  
  - Configuration: Default wait time (not explicitly set here).  
  - Inputs: From IF node’s false branch and error output of download node.  
  - Outputs: Loops back to check snapshot again node.  
  - Edge cases: Excessive delays may increase total workflow runtime.

  **check Snapshot Again**  
  - Type: NoOp node  
  - Role: Acts as a connector to restart batch item splitting and status checking.  
  - Inputs: From Wait node.  
  - Outputs: To hotels node for batch iteration.  
  - Edge cases: None functional; purely structural.

#### 2.4 Data Cleaning & Aggregation

- **Overview:**  
  Extracts relevant hotel data fields from raw scraped JSON and aggregates all items into a single structured dataset.

- **Nodes Involved:**  
  - clean data (Set)  
  - Aggregate

- **Node Details:**  

  **clean data**  
  - Type: Set node  
  - Role: Filters and reassigns key hotel properties for clarity and downstream use.  
  - Configuration: Assigns:  
    - `title` from `$json.title`  
    - `address` from `$json.address`  
    - `original_price` from `$json.original_price` (number)  
    - `final_price` from `$json.final_price` (number)  
  - Inputs: Raw data from snapshot download node.  
  - Outputs: Cleaned, simplified JSON objects.  
  - Edge cases: Missing fields in scraped data may cause null or undefined values.

  **Aggregate**  
  - Type: Aggregate node  
  - Role: Combines all cleaned data items into a single JSON array for AI processing.  
  - Configuration: Uses "aggregateAllItemData" to collect all batch items.  
  - Inputs: From clean data node.  
  - Outputs: One aggregated array object.  
  - Edge cases: Empty input results in empty aggregation.

#### 2.5 AI-Powered Formatting & Calculation

- **Overview:**  
  Uses OpenRouter AI to convert the aggregated hotel data into a human-readable list, including price currency conversion with a Calculator tool.

- **Nodes Involved:**  
  - Human Friendly Results  
  - OpenRouter Chat Model2  
  - Calculator

- **Node Details:**  

  **Human Friendly Results**  
  - Type: Langchain Agent node  
  - Role: Converts the hotel list JSON into a nicely formatted text output for display.  
  - Configuration:  
    - Input text: uses the aggregated hotel list JSON string (`={{ $json }}`)  
    - System message prompt instructs:  
      - Convert list into well-presented human display  
      - Use CALC tool for price conversion to euros  
      - Start response with "Here is the list of hotels available on Booking in the city of [city]"  
    - Output parser enabled to handle AI response.  
  - Inputs: Aggregated list from Aggregate node; AI model and calculator tool connected.  
  - Outputs: Final formatted text.  
  - Edge cases: AI model failure or unexpected output format; price calculation errors.

  **OpenRouter Chat Model2**  
  - Type: Langchain OpenRouter language model  
  - Role: Provides AI language model capabilities to the workflow.  
  - Configuration: Temperature set to 0 for deterministic output.  
  - Credentials: OpenRouter API key required.  
  - Inputs: Receives prompt from Human Friendly Results node.  
  - Outputs: AI-generated text output.  
  - Edge cases: API quota limits, network errors, invalid credentials.

  **Calculator**  
  - Type: Langchain Calculator tool node  
  - Role: Performs currency conversions or mathematical calculations as requested by AI prompt.  
  - Configuration: No user parameters; invoked by AI prompt.  
  - Inputs: AI prompt context from Human Friendly Results node.  
  - Outputs: Calculation results back to AI agent.  
  - Edge cases: Malformed expressions or unsupported calculations could cause errors.

#### 2.6 Final Output

- **Overview:**  
  The formatted hotel list is returned as the chat response, completing the workflow cycle.

- **Nodes Involved:**  
  - None additional; output is from the last AI agent node.

- **Node Details:**  
  - The workflow’s last node (Human Friendly Results) outputs the well-formatted hotel list text back through the chat webhook.

---

### 3. Summary Table

| Node Name                        | Node Type                                 | Functional Role                       | Input Node(s)                       | Output Node(s)                       | Sticky Note                                                                                                   |
|---------------------------------|-------------------------------------------|------------------------------------|-----------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received       | Langchain Chat Trigger                     | Entry point, receives city input   | -                                 | parameters                         |                                                                                                              |
| parameters                      | Set                                       | Assign city parameter from chat    | When chat message received         | Initiate batch extraction from URL |                                                                                                              |
| Initiate batch extraction from URL | Bright Data                               | Start scraping job on Bright Data  | parameters                        | hotels                            | 👉 Get your free Bright Data API key here: Bright Data https://get.brightdata.com/unstoppable                 |
| hotels                         | SplitInBatches                             | Batch control for status checking  | Initiate batch extraction from URL | Check the status of a batch extraction |                                                                                                              |
| Check the status of a batch extraction | Bright Data                              | Monitor scraping job status        | hotels                           | check if result ready              | 👉 Get your free Bright Data API key here: Bright Data https://get.brightdata.com/unstoppable                 |
| check if result ready           | IF                                        | Checks if scraping is "ready"      | Check the status of a batch extraction | Download the snapshot content / Wait |                                                                                                              |
| Download the snapshot content   | Bright Data                               | Download scraped data snapshot     | check if result ready             | clean data / Wait                 | 👉 Get your free Bright Data API key here: Bright Data https://get.brightdata.com/unstoppable                 |
| Wait                           | Wait                                      | Delay between status checks        | check if result ready / Download the snapshot content (error) | check Snapshot Again              |                                                                                                              |
| check Snapshot Again            | NoOp                                      | Loop control connector             | Wait                            | hotels                           |                                                                                                              |
| clean data                     | Set                                       | Extract and assign key hotel fields | Download the snapshot content    | Aggregate                        |                                                                                                              |
| Aggregate                     | Aggregate                                 | Aggregate all cleaned hotel data   | clean data                      | Human Friendly Results           |                                                                                                              |
| Human Friendly Results          | Langchain Agent                           | Format hotel list for display      | Aggregate / OpenRouter Chat Model2 / Calculator | -                              |                                                                                                              |
| OpenRouter Chat Model2          | Langchain OpenRouter LM                    | AI language model                  | Human Friendly Results           | Human Friendly Results           |                                                                                                              |
| Calculator                    | Langchain Calculator tool                   | Perform price conversions          | Human Friendly Results           | Human Friendly Results           |                                                                                                              |
| Sticky Note                   | Sticky Note                                | Documentation                      | -                               | -                                | # Booking Hotels Scraper ... [Phil | Inforeole](https://inforeole.fr)                                        |
| Sticky Note1                  | Sticky Note                                | Documentation                      | -                               | -                                | ## Start\n\nAsk for City to scrape\nSend Hotel listing Request                                               |
| Sticky Note2                  | Sticky Note                                | Documentation                      | -                               | -                                | ## Wait for results\n\nIt takes time to get result (scrap, bypass captchas...)                                |
| Sticky Note3                  | Sticky Note                                | Documentation                      | -                               | -                                | ## Process Hotel listing\nGet Hotel listing (if not ready, loop)\nProcess it in a Fiendly way                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create trigger node: "When chat message received"**  
   - Type: Langchain Chat Trigger  
   - Configure webhookId (auto-generated)  
   - Set response mode to `lastNode` to return final response.

2. **Add "parameters" node (Set)**  
   - Add field `city` as string  
   - Assign value from expression: `={{ $json.chatInput }}`  
   - Connect input from "When chat message received" node.

3. **Add "Initiate batch extraction from URL" (Bright Data node)**  
   - Credentials: Configure Bright Data API credentials.  
   - Resource: `webScrapper`  
   - Operation: `triggerCollectionByUrl`  
   - Dataset ID: Set to `"gd_m4bf7a917zfezv9d5"` (Booking Listings Search)  
   - URLs: Use JSON array template:  
     ```json
     [{
       "url": "https://www.booking.com",
       "location": "{{ $json.city }}",
       "check_in": "2025-11-01T00:00:00.000Z",
       "check_out": "2025-11-08T00:00:00.000Z",
       "adults": 2,
       "children": 1,
       "rooms": 1,
       "property_type": "Hostels",
       "currency": "",
       "country": ""
     }]
     ```  
   - Connect input from "parameters" node.

4. **Add "hotels" node (SplitInBatches)**  
   - Enable reset option.  
   - Connect input from "Initiate batch extraction from URL".

5. **Add "Check the status of a batch extraction" (Bright Data node)**  
   - Credentials: Bright Data API credentials.  
   - Resource: `webScrapper`  
   - Operation: `monitorProgressSnapshot`  
   - Snapshot ID: Set via expression: `={{ $('Initiate batch extraction from URL').item.json.snapshot_id }}`  
   - Connect input from "hotels" node.

6. **Add "check if result ready" (IF node)**  
   - Condition: Check if `$json.status === "ready"`  
   - Connect input from "Check the status of a batch extraction".  
   - True output connects to "Download the snapshot content".  
   - False output connects to "Wait".

7. **Add "Download the snapshot content" (Bright Data node)**  
   - Credentials: Bright Data API credentials.  
   - Resource: `webScrapper`  
   - Operation: `downloadSnapshot`  
   - Snapshot ID: `={{ $('Check the status of a batch extraction').item.json.snapshot_id }}`  
   - OnError: Set to continue on error (to avoid total failure).  
   - Connect input from IF node’s true branch.

8. **Add "Wait" node**  
   - Use default wait duration or configure as needed (e.g., 10-30 seconds).  
   - Connect input from IF node’s false branch and error output of "Download the snapshot content".

9. **Add "check Snapshot Again" (NoOp node)**  
   - Connect input from "Wait" node.  
   - Connect output to "hotels" node (to restart batch iteration).

10. **Add "clean data" node (Set)**  
    - Assign these fields from incoming JSON:  
      - `title`: `={{ $json.title }}`  
      - `address`: `={{ $json.address }}`  
      - `original_price`: `={{ $json.original_price }}` as number  
      - `final_price`: `={{ $json.final_price }}` as number  
    - Connect input from "Download the snapshot content".

11. **Add "Aggregate" node**  
    - Set operation to `aggregateAllItemData` to combine all batch items.  
    - Connect input from "clean data".

12. **Add "Calculator" node (Langchain Calculator tool)**  
    - No parameters to set.  
    - Connect input from "Human Friendly Results" node’s AI tool input.

13. **Add "OpenRouter Chat Model2" node (Langchain OpenRouter LM)**  
    - Credentials: Configure OpenRouter API key.  
    - Set temperature to 0 (deterministic).  
    - Connect input from "Human Friendly Results" node’s AI language model input.

14. **Add "Human Friendly Results" node (Langchain Agent)**  
    - Parameters:  
      - Text input: `={{ $json }}` (aggregated hotel list)  
      - System message:  
        ```
        Convert this list of hotels into a well-presented list for human display.

        Use the CALC tool to show prices in euros.

        Start the response with 'Here is the list of hotels available on Booking in the city of' followed by the name of the city
        ```  
    - Enable output parser.  
    - Connect input from "Aggregate" node.  
    - Connect AI tool input to "Calculator" node.  
    - Connect AI language model input to "OpenRouter Chat Model2" node.  
    - This node’s output is the final workflow response.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow automates extracting and presenting hotel listings from Booking.com, leveraging Bright Data for scraping and OpenRouter AI for data refinement.                                                                     | Overview and purpose                                                 |
| 👉 Get your free Bright Data API key here: [Bright Data](https://get.brightdata.com/unstoppable)                                                                                                                                 | Required for scraping nodes                                          |
| [Phil | Inforeole](https://inforeole.fr) - Professional automation services contact                                                                                                                                                | Project credit and contact                                           |
| The workflow waits and loops until scraping results are ready to handle delays in scraping and captcha bypassing.                                                                                                                | Workflow resilience                                                 |
| AI temperature set to 0 to ensure deterministic and consistent output formatting.                                                                                                                                                | AI model configuration                                              |
| Fixed check-in and check-out dates (Nov 1-8, 2025) and room parameters are hardcoded but can be parameterized for flexibility.                                                                                                   | Potential improvement                                               |
| The Calculator tool is invoked inside the AI prompt to convert prices into euros dynamically.                                                                                                                                   | AI & tool integration                                               |
| The workflow uses multiple sticky notes for documentation and user guidance inside the editor for clarity.                                                                                                                       | Documentation aids                                                  |

---

This detailed analysis and reconstruction guide enables understanding, modification, and reproduction of the scraping and AI formatting workflow for Booking.com hotel listings using n8n, Bright Data, and OpenRouter AI.