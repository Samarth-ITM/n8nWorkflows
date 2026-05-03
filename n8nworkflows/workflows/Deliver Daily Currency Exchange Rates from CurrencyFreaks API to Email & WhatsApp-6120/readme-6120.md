Deliver Daily Currency Exchange Rates from CurrencyFreaks API to Email & WhatsApp

https://n8nworkflows.xyz/workflows/deliver-daily-currency-exchange-rates-from-currencyfreaks-api-to-email---whatsapp-6120


# Deliver Daily Currency Exchange Rates from CurrencyFreaks API to Email & WhatsApp

### 1. Workflow Overview

This workflow automates the daily retrieval and distribution of multi-currency exchange rates using the CurrencyFreaks API. It is designed to run every day at a fixed time (7:30 AM IST), fetch the latest currency exchange rates relative to INR, and send these updates via both Email and WhatsApp to specified recipients.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow daily at the configured time.
- **1.2 Configuration Setup**: Defines API credentials and target currencies.
- **1.3 API Request and Response Handling**: Calls the CurrencyFreaks API to fetch latest exchange rates and waits briefly to respect API limits.
- **1.4 Recipient Setup and Message Preparation**: Sets the email and WhatsApp recipients and dynamically creates the message subject and body.
- **1.5 Notification Delivery**: Sends the prepared exchange rate updates via Email and WhatsApp.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** This block triggers the entire workflow daily at 7:30 AM IST to ensure fresh currency data is fetched and disseminated.
- **Nodes Involved:** 
  - Daily Trigger (7:30 AM IST)
  - Sticky Note (related comment)

- **Node Details:**

  - **Daily Trigger (7:30 AM IST)**
    - Type: Schedule Trigger
    - Role: Initiates workflow execution at a precise daily time.
    - Configuration: Set to trigger every day at 7:30 AM IST.
    - Input: None
    - Output: Triggers "Set Config: API Key & Currencies" node.
    - Edge Cases: If the n8n instance is down or paused at trigger time, workflow will not run.
    - Sticky Note: Explains the trigger purpose and timing.

#### 1.2 Configuration Setup

- **Overview:** This block defines the API key and the list of target currencies to be requested from the API.
- **Nodes Involved:** 
  - Set Config: API Key & Currencies
  - Sticky Note1

- **Node Details:**

  - **Set Config: API Key & Currencies**
    - Type: Set
    - Role: Stores API key and currency symbols as variables for reuse.
    - Configuration: 
      - API key placeholder string `Enter_your_api_key` (to be replaced by user).
      - Currencies list: INR, CAD, AUD, CNY, EUR, USD.
    - Input: Trigger from Schedule Trigger.
    - Output: Passes configuration to API Request node.
    - Edge Cases: Missing or invalid API key will cause API request failure.
    - Sticky Note: Explains purpose of this node to define API key and currencies.

#### 1.3 API Request and Response Handling

- **Overview:** This block makes a call to the CurrencyFreaks API to retrieve the latest exchange rates based on the configured currencies, then waits briefly to ensure stability and compliance with API rate limits.
- **Nodes Involved:** 
  - Fetch Exchange Rates (CurrencyFreaks)
  - Wait for API Response (5s)
  - Sticky Note2
  - Sticky Note3

- **Node Details:**

  - **Fetch Exchange Rates (CurrencyFreaks)**
    - Type: HTTP Request
    - Role: Sends GET request to CurrencyFreaks API endpoint to fetch latest exchange rates.
    - Configuration:
      - URL constructed using expressions to inject API key and currency symbols dynamically.
      - Example URL pattern: `https://api.currencyfreaks.com/v2.0/rates/latest?apikey={{ $json['API Key'] }}&symbols={{ $json['Currencies'] }}`
      - No additional HTTP options specified.
    - Input: Receives API key and currencies from previous node.
    - Output: JSON response with exchange rates.
    - Edge Cases: 
      - HTTP errors (e.g., 401 Unauthorized if API key invalid).
      - Network timeout or API unavailability.
      - Rate limit exceeded errors.
    - Sticky Note: Describes the API call purpose.

  - **Wait for API Response (5s)**
    - Type: Wait
    - Role: Delays workflow for 5 seconds after API call to avoid rapid successive requests.
    - Configuration: Fixed 5-second wait.
    - Input: Receives API response from HTTP Request node.
    - Output: Proceeds to recipient setup node.
    - Edge Cases: Delay may be insufficient if API rate limits are very strict.
    - Sticky Note: Notes purpose of wait to respect API usage and maintain stability.

#### 1.4 Recipient Setup and Message Preparation

- **Overview:** This block sets the email and WhatsApp recipients and constructs the message subject and body dynamically for the notifications.
- **Nodes Involved:** 
  - Set Email & WhatsApp Recipients
  - Create Message Subject & Body
  - Sticky Note4
  - Sticky Note5

- **Node Details:**

  - **Set Email & WhatsApp Recipients**
    - Type: Set
    - Role: Defines recipients’ contact details.
    - Configuration:
      - Email_id set to a fixed email address `abc@gmail.com`.
      - WhatsApp recipient phone number is configured later in WhatsApp node.
    - Input: From Wait node.
    - Output: Passes recipient info to message creation node.
    - Edge Cases: Invalid or missing recipient addresses will cause delivery failures.
    - Sticky Note: Explains this node sets recipients.

  - **Create Message Subject & Body**
    - Type: Set
    - Role: Creates the email subject line and prepares message content.
    - Configuration:
      - Subject: "Today's Currency Exchange Rates"
      - The body text is prepared in sending nodes by referencing API data dynamically.
    - Input: Receives recipient details.
    - Output: Passes subject and body to sending nodes.
    - Edge Cases: Expression failures if previous nodes produce unexpected data.
    - Sticky Note: Notes dynamic message creation for subject and body.

#### 1.5 Notification Delivery

- **Overview:** This block sends the formatted currency exchange rate updates via Email and WhatsApp using configured credentials.
- **Nodes Involved:** 
  - Send WhatsApp Alert
  - Send Email Alert
  - Sticky Note6

- **Node Details:**

  - **Send WhatsApp Alert**
    - Type: WhatsApp
    - Role: Sends WhatsApp message to configured recipient.
    - Configuration:
      - Text body includes emoji and exchange rate data dynamically injected from API response.
      - PhoneNumberId and recipient phone number set with expressions and static values.
      - Uses WhatsApp API credentials.
    - Input: Receives message subject/body and recipient info.
    - Output: None (endpoint node).
    - Edge Cases: 
      - API authentication failure.
      - Invalid recipient number.
      - Message formatting or expression errors.
    - Credentials: WhatsApp API credential named "WhatsApp-test".
    - Sticky Note: Describes sending update over WhatsApp.

  - **Send Email Alert**
    - Type: Email Send
    - Role: Sends email with the exchange rate update.
    - Configuration:
      - Email text body with embedded rates from API response.
      - Subject dynamically taken from "Create Message Subject & Body".
      - Recipient email from "Set Email & WhatsApp Recipients".
      - From email fixed as `xyz@gmail.com`.
      - Uses SMTP credentials.
    - Input: Receives message subject/body and recipient info.
    - Output: None (endpoint node).
    - Edge Cases:
      - SMTP authentication failure.
      - Invalid recipient email.
      - Expression errors in message.
    - Credentials: SMTP credential named "SMTP -test".
    - Sticky Note: Describes sending update over Email.

---

### 3. Summary Table

| Node Name                       | Node Type               | Functional Role                              | Input Node(s)                    | Output Node(s)                         | Sticky Note                                                                                 |
|--------------------------------|-------------------------|----------------------------------------------|---------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------|
| Daily Trigger (7:30 AM IST)     | Schedule Trigger        | Trigger workflow daily at fixed time         | None                            | Set Config: API Key & Currencies     | Triggers the workflow every day at a fixed time (e.g., 7 AM IST) to fetch and send updates. |
| Set Config: API Key & Currencies| Set                    | Define API key and target currencies         | Daily Trigger                   | Fetch Exchange Rates (CurrencyFreaks)| Define your API key and target currencies (INR, CAD, AUD, CNY, EUR, USD) in a manual node. |
| Fetch Exchange Rates (CurrencyFreaks) | HTTP Request      | Fetch latest exchange rates from CurrencyFreaks API | Set Config: API Key & Currencies | Wait for API Response (5s)           | Calls the exchange rate API using your API key and fetches latest rates with INR as base.  |
| Wait for API Response (5s)      | Wait                   | Delay to respect API rate limits and stability| Fetch Exchange Rates            | Set Email & WhatsApp Recipients       | Adds a short delay to ensure API rate limits are respected and system stability is maintained.|
| Set Email & WhatsApp Recipients | Set                    | Set recipient email and WhatsApp contact     | Wait for API Response           | Create Message Subject & Body          | Set the list of email addresses and WhatsApp numbers who should receive the currency update.|
| Create Message Subject & Body   | Set                    | Prepare email subject and message body       | Set Email & WhatsApp Recipients | Send WhatsApp Alert, Send Email Alert | Dynamically generate subject line and message body with exchange rates.                    |
| Send WhatsApp Alert             | WhatsApp                | Send currency update via WhatsApp             | Create Message Subject & Body   | None                                 | Sends the formatted currency rate update via WhatsApp.                                    |
| Send Email Alert               | Email Send              | Send currency update via Email                 | Create Message Subject & Body   | None                                 | Sends the formatted currency rate update via Email.                                       |
| Sticky Note                    | Sticky Note             | Informational comment                         | None                            | None                                 | Triggers the workflow every day at a fixed time (e.g., 7 AM IST) to fetch and send updates. |
| Sticky Note1                   | Sticky Note             | Informational comment                         | None                            | None                                 | Define your API key and target currencies (INR, CAD, AUD, CNY, EUR, USD) in a manual node.  |
| Sticky Note2                   | Sticky Note             | Informational comment                         | None                            | None                                 | Calls the exchange rate API using your API key and fetches latest rates with INR as base.  |
| Sticky Note3                   | Sticky Note             | Informational comment                         | None                            | None                                 | Adds a short delay to ensure API rate limits are respected and system stability is maintained.|
| Sticky Note4                   | Sticky Note             | Informational comment                         | None                            | None                                 | Set the list of email addresses and WhatsApp numbers who should receive the currency update.|
| Sticky Note5                   | Sticky Note             | Informational comment                         | None                            | None                                 | Dynamically generate subject line and message body with exchange rates.                    |
| Sticky Note6                   | Sticky Note             | Informational comment                         | None                            | None                                 | Sends the formatted currency rate update via Email and WhatsApp.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**
   - Name: `Daily Trigger (7:30 AM IST)`
   - Type: Schedule Trigger
   - Configure: Set to trigger every day at 7:30 AM IST.
   - Connect output to next node.

2. **Create a Set Node for API Key and Currencies**
   - Name: `Set Config: API Key & Currencies`
   - Type: Set
   - Parameters:
     - Add two fields:
       - `API` (string): Set to your CurrencyFreaks API key.
       - `Currencies` (string): Set to `INR,CAD,AUD,CNY,EUR,USD`.
   - Connect input from `Daily Trigger (7:30 AM IST)`.
   - Output to next node.

3. **Create HTTP Request Node to Fetch Exchange Rates**
   - Name: `Fetch Exchange Rates (CurrencyFreaks)`
   - Type: HTTP Request
   - Parameters:
     - Method: GET
     - URL: Use expression to construct URL as:
       ```
       https://api.currencyfreaks.com/v2.0/rates/latest?apikey={{ $json["API"] }}&symbols={{ $json["Currencies"] }}
       ```
     - No authentication or additional headers required.
   - Connect input from `Set Config: API Key & Currencies`.
   - Output to next node.

4. **Create Wait Node**
   - Name: `Wait for API Response (5s)`
   - Type: Wait
   - Parameters:
     - Wait Time: 5 seconds
   - Connect input from `Fetch Exchange Rates (CurrencyFreaks)`.
   - Output to next node.

5. **Create Set Node for Email and WhatsApp Recipients**
   - Name: `Set Email & WhatsApp Recipients`
   - Type: Set
   - Parameters:
     - Add field:
       - `Email_id` (string): Set to recipient email, e.g., `abc@gmail.com`.
     - WhatsApp recipient phone number will be configured in WhatsApp node.
   - Connect input from `Wait for API Response (5s)`.
   - Output to next node.

6. **Create Set Node to Generate Message Subject and Body**
   - Name: `Create Message Subject & Body`
   - Type: Set
   - Parameters:
     - Add field:
       - `Subject` (string): Set to `Today's Currency Exchange Rates`.
     - The message body will be constructed within the sending nodes using expressions referencing API data.
   - Connect input from `Set Email & WhatsApp Recipients`.
   - Output to next nodes.

7. **Create WhatsApp Node to Send Alert**
   - Name: `Send WhatsApp Alert`
   - Type: WhatsApp
   - Parameters:
     - Operation: Send
     - Recipient Phone Number: Set to intended recipient's WhatsApp number, e.g., `+919876567854`.
     - Phone Number ID: Set as appropriate for your WhatsApp Business API.
     - Text Body: Use expression to format message, e.g.:
       ```
       📈 Today's Currency Exchange Rates – {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.data }}
       Base: INR
       USD:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.USD }}
       EUR:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.EUR }}
       CAD:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.CAD }}
       AUD:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.AUD }}
       CNY:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.CNY }}
       — Auto-sent by FX Update Bot
       ```
   - Credentials: Configure your WhatsApp API credentials.
   - Connect input from `Create Message Subject & Body`.

8. **Create Email Send Node**
   - Name: `Send Email Alert`
   - Type: Email Send
   - Parameters:
     - To Email: Use expression `={{ $('Set Email & WhatsApp Recipients').item.json.Email_id }}`
     - From Email: Your verified sending email, e.g., `xyz@gmail.com`.
     - Subject: Use expression `={{ $('Create Message Subject & Body').item.json.Subject }}`
     - Text Body:
       ```
       📈 Today's Currency Exchange Rates – {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.data }}

       Base: INR

       USD:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.USD }}
       EUR:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.EUR }}
       CAD:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.CAD }}
       AUD:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.AUD }}
       CNY:  {{ $('Fetch Exchange Rates (CurrencyFreaks)').item.json.rates.CNY }}

       — Auto-sent by FX Update Bot
       ```
     - Email Format: Text
   - Credentials: Configure SMTP credentials.
   - Connect input from `Create Message Subject & Body`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow triggers every day at a fixed time (7:30 AM IST) to ensure timely delivery of updated currency rates.                                      | Scheduling information                               |
| API key must be obtained from [CurrencyFreaks](https://currencyfreaks.com/) and inserted in the configuration node to enable API access.               | CurrencyFreaks API                                   |
| WhatsApp API requires proper setup of WhatsApp Business API credentials and phone number IDs for sending messages.                                       | WhatsApp Business API documentation                  |
| SMTP credentials must be valid and configured for sending emails through the chosen email provider.                                                      | SMTP email provider documentation                     |
| Expressions dynamically refer to JSON paths in the API response; confirm API response format matches expected keys to avoid runtime errors.              | n8n Expression guide                                 |
| Consider error handling for API rate limits, invalid credentials, and network failures by adding additional nodes or workflows in production use cases.  | n8n error handling best practices                      |
| This workflow can be adapted or extended to include additional currencies or messaging platforms as needed.                                              | Customization and extensibility                        |

---

**Disclaimer:** The content provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.