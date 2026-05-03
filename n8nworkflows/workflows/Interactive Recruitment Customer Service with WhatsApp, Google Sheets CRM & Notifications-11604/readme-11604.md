Interactive Recruitment Customer Service with WhatsApp, Google Sheets CRM & Notifications

https://n8nworkflows.xyz/workflows/interactive-recruitment-customer-service-with-whatsapp--google-sheets-crm---notifications-11604


# Interactive Recruitment Customer Service with WhatsApp, Google Sheets CRM & Notifications

### 1. Workflow Overview

This workflow automates interactive customer service recruitment communications via WhatsApp integrated with Google Sheets CRM and email notifications. It targets recruitment service providers who want to handle inquiries, new requests, tracking, complaints, and human agent escalation seamlessly on WhatsApp.

Logical blocks:

- **1.1 WhatsApp Incoming Message Reception**  
  Captures incoming WhatsApp messages via webhooks and extracts detailed message data.

- **1.2 Media Handling**  
  Detects media messages, downloads media content from WhatsApp API, and stores them in a Data Table.

- **1.3 Session Management & Bot Enablement**  
  Tracks user sessions, labels menu navigation state, and controls bot activation per user.

- **1.4 Conversation Logging**  
  Logs all incoming and outgoing messages with metadata for auditing and analytics.

- **1.5 Menu Cooldown & Main Menu Presentation**  
  Implements a cooldown to prevent menu spamming and sends an interactive main menu to users.

- **1.6 Menu Routing**  
  Routes user selections from the main menu to specific service flows.

- **1.7 Sub-Menu Routing**  
  Handles nested menu actions triggered by user selections and prepares responses.

- **1.8 Service-Specific Processing**  
  Executes distinct logic for services: Price Inquiry, New Request, Request Tracking, Worker Transfer, Translation, Complaints, About Office, and Human Agent contact.

- **1.9 Google Sheets CRM Integration and Email Notifications**  
  Saves service requests and complaints into Google Sheets tabs and sends notification emails to staff.

- **1.10 Outgoing Message Preparation and Sending**  
  Prepares WhatsApp API payloads, sends messages, and logs outgoing communication.

---

### 2. Block-by-Block Analysis

#### 1.1 WhatsApp Incoming Message Reception

- **Overview:**  
  Receives WhatsApp messages via multiple webhook endpoints, verifies webhook subscriptions, and extracts structured data from all message types.

- **Nodes Involved:**  
  - `Webhook`  
  - `Verify Webhook2`  
  - `استخراج البيانات` (Data Extraction Code)  

- **Node Details:**

  - **Webhook**  
    - Type: Webhook node  
    - Configured to receive messages at path `/whatsapp`, supports multiple HTTP methods including GET for verification.  
    - Outputs raw WhatsApp webhook JSON.  
    - Handles verification challenges (`hub.challenge`) and forwards to Verify Webhook2 or data extraction.

  - **Verify Webhook2**  
    - Type: Respond to Webhook  
    - Responds with challenge token or “OK” for webhook verification requests.  
    - Input: Webhook node output for verification.  
    - Output: HTTP 200 response.

  - **استخراج البيانات** (Data Extraction)  
    - Type: Code node (JavaScript)  
    - Parses incoming WhatsApp webhook JSON to extract:  
      - Sender and business phone numbers  
      - Contact name  
      - Message type and content (text, interactive replies, media metadata)  
      - Handles all WhatsApp message types uniformly (text, image, audio, video, document, interactive buttons/lists)  
      - Returns array of normalized message objects for downstream processing.  
    - Edge Cases: Missing or malformed webhook data, unsupported message types.  
    - Input: Webhook output JSON.  
    - Output: Normalized message JSON objects.

#### 1.2 Media Handling

- **Overview:**  
  Detects media messages, fetches media URLs from WhatsApp API, downloads media files as base64, and stores metadata and content in a Data Table.

- **Nodes Involved:**  
  - `في وسائط؟` (Media Present? IF)  
  - `Action in an app` (Fetch media URL)  
  - `HTTP Request` (Download media file)  
  - `حفظ الميديا` (Save media to Data Table)  
  - `Upsert row(s)` (Upsert media record in Data Table)  
  - `Code in JavaScript4` (Prepare media metadata output)  

- **Node Details:**

  - **في وسائط؟**  
    - Type: IF node  
    - Checks if the message contains a media_id (media present).  
    - Routes: Yes → fetch media; No → skip.

  - **Action in an app**  
    - Type: HTTP Request  
    - Calls WhatsApp Graph API to get media URL and mime type using media_id.  
    - Auth: Bearer token credential.  
    - Potential failures: Auth errors, invalid media_id, API downtime.

  - **HTTP Request**  
    - Type: HTTP Request  
    - Downloads media file as binary stream from obtained URL with authorization.  
    - Configured for file response.  
    - Failures: Timeout, network errors.

  - **حفظ الميديا**  
    - Type: Code node  
    - Converts downloaded binary to base64, extracts mime, filename, caption, and prepares record for Data Table.  
    - Cleans filename and preserves metadata.  
    - Logs size and type info.  
    - Edge cases: Missing binary data.

  - **Upsert row(s)**  
    - Type: Data Table node  
    - Upserts media metadata and base64 content into “media” Data Table keyed by media_id.  
    - Ensures no duplicate entries.  
    - Input: Code node output.  
    - Output: Stored media record.

  - **Code in JavaScript4**  
    - Type: Code node  
    - Returns cleaned media metadata for further processing downstream.

#### 1.3 Session Management & Bot Enablement

- **Overview:**  
  Tracks user session states and bot enablement status globally and per user, managing conversation context and bot responsiveness.

- **Nodes Involved:**  
  - `Code in JavaScript1` (Label Tracker & Bot Enable Check)  
  - `bot on?` (IF node to route based on bot activation)  
  - `اقرأ الحالة` (Read session state)  
  - `Code in JavaScript2` (Update session state)  

- **Node Details:**

  - **Code in JavaScript1**  
    - Maintains global labels per user to track current menu or submenu.  
    - Manages bot master switch (on/off) and per-user overrides.  
    - Inputs interactive_reply to assign labels like “الأسعار”, “طلب جديد”, “الشكاوى”, etc.  
    - Outputs bot_enabled flag.

  - **bot on?**  
    - IF node filters messages to proceed only if bot is enabled for the user.  
    - If disabled, workflow ends with a webhook response.

  - **اقرأ الحالة**  
    - Reads session state from global storage keyed by phone_number_id and phone_number.  
    - Outputs session state and timestamps for routing.

  - **Code in JavaScript2**  
    - Updates session state in global storage after each message or button reply.  
    - Stores last interaction type (button id or typed text).  
    - Ensures session name and phone stored.

#### 1.4 Conversation Logging

- **Overview:**  
  Logs every incoming and outgoing message into global memory for audit and analytics, preserving key metadata.

- **Nodes Involved:**  
  - `سجل الرسائل` (Log incoming messages)  
  - Multiple `سجل رسالة خارجة*` nodes (Log outgoing messages for various services)

- **Node Details:**

  - **سجل الرسائل**  
    - Code node logs incoming message metadata and media info (type, id, caption).  
    - Keeps last 5000 messages to limit memory.  
    - Updates contact names if changed.

  - **سجل رسالة خارجة* (0 through 14)**  
    - Each logs outgoing message sent to WhatsApp API.  
    - Extracts text from payloads (text or interactive).  
    - Maintains global message log with direction “out”.  
    - Handles fallback previews if text extraction fails.

#### 1.5 Menu Cooldown & Main Menu Presentation

- **Overview:**  
  Implements a 5-minute cooldown per user to prevent menu spam and sends an interactive main menu with 8 service options.

- **Nodes Involved:**  
  - `رسالة جديدة؟` (IF node to check new message)  
  - `Menu Cooldown Check` (Cooldown logic in code)  
  - `If3` (IF node based on cooldown pass)  
  - `حضّر الإرسال – القائمة الرئيسية` (Prepare main menu payload)  
  - `سجل رسالة خارجة (عام)` (Log outgoing main menu message)  
  - `إرسال القائمة الرئيسية` (Send menu via WhatsApp API)  
  - `Respond to Webhook` (Webhook response)

- **Node Details:**

  - **رسالة جديدة؟**  
    - Checks if the interactive_reply is empty → new message triggers menu.

  - **Menu Cooldown Check**  
    - Uses global storage to track last menu send timestamp per user key.  
    - If more than 5 minutes passed, sets flag to send menu.

  - **If3**  
    - Routes to sending menu only if cooldown passed.

  - **حضّر الإرسال – القائمة الرئيسية**  
    - Constructs WhatsApp interactive list message with 8 menu options (Prices, New Request, Track, Transfer, Translation, Complaints, About Office, Talk to Agent).  
    - Includes footer and call-to-action button.

  - **سجل رسالة خارجة (عام)**  
    - Logs the outgoing menu message.

  - **إرسال القائمة الرئيسية**  
    - Sends prepared JSON payload to WhatsApp Graph API using HTTP Bearer token authentication.

  - **Respond to Webhook**  
    - Returns HTTP 200 with JSON status success to WhatsApp.

#### 1.6 Menu Routing

- **Overview:**  
  Routes user interactive replies from the main menu to corresponding service flows.

- **Nodes Involved:**  
  - `sub route` (Switch node for nested replies)  
  - `menu route` (Switch node for main menu options)  

- **Node Details:**

  - **sub route**  
    - Switches based on reply prefix:  
      - `price_*` → Price inquiry  
      - `req_*` → New request nationality selection  
      - `complaint_*` → Complaint subtype selection  
      - `m_*` → Main menu options  
    - Routes to appropriate nodes for data retrieval, message preparation, and logging.

  - **menu route**  
    - Switches on exact interactive_reply matching main menu options:  
      - Prices, New Request, Track Request, Worker Transfer, Translation, Complaints, About Office, Talk to Agent  
    - Routes to message preparation nodes for each service.

#### 1.7 Sub-Menu Routing & Data Fetching

- **Overview:**  
  Fetches static data or Google Sheets data for nationality-related info and triggers secondary flows.

- **Nodes Involved:**  
  - `Get row(s)` (Data Table lookup for prices)  
  - `السير` (Static list of CV links per nationality)  
  - Google Sheets nodes for requests, transfers, complaints  
  - Various IF nodes to match nationality or complaint types  

- **Node Details:**

  - **Get row(s)**  
    - Reads price data from a Data Table keyed by nationality code.

  - **السير**  
    - Outputs static JSON array mapping nationality codes to CV links.

  - **Google Sheets nodes**  
    - Append or update rows for new requests, transfers, and complaints keyed by phone number.

  - **IF nodes**  
    - Match nationality or complaint types to prepare accurate responses.

#### 1.8 Service-Specific Processing

- **Overview:**  
  Each service flow prepares a WhatsApp message, logs outgoing message, stores data in Google Sheets, and optionally sends email notifications.

- **Services and Key Nodes:**

  - **Price Inquiry**  
    - Prepare message with price per nationality (`حضّر الإرسال – الأسعار`)  
    - Send response and log message.

  - **New Request**  
    - Prepare nationality selection list (`حضّر الإرسال – طلب جديد`)  
    - On nationality selection: send CV link (`تحضير إرسال - رسالة السير الذاتية`)  
    - Save request to “طلبات جديدة” Google Sheet (`إدخال طلب جديد`)  
    - Send confirmation message (`تحضير إرسال - رسالة السير الذاتية1`)  
    - Email notification to staff (`Send a message` node).

  - **Request Tracking**  
    - Ask user to wait (`حضّر الإرسال – طلب متابعة`)  
    - Lookup request in Google Sheets (`متابعة`)  
    - Conditional response: send status (`تحضير إرسال - رسالة المتابعة`) or not found message (`تحضير إرسال - رسالة المتابعة لا يوجد`)  
    - Log outgoing message.

  - **Worker Transfer**  
    - Send transfer service link (`حضّر الإرسال – نقل الخادمات`)  
    - Save transfer request to “نقل الخادمات” Google Sheet (`إدخال نقل خادمات`)  
    - Confirmation message (`حضر الارسال - نقل الخادمات`)  
    - Email notification (`Send a message1`).

  - **Translation Service**  
    - Send static info text with WhatsApp contact link (`حضّر الإرسال – الترجمة`)  
    - Log outgoing message.

  - **Complaints**  
    - Present complaint type list (`حضّر الإرسال – شكوى`)  
    - Save complaint to “شكاوي” Google Sheet (`إدخال نقل خادمات1`)  
    - Confirmation message (`حضّر الإرسال – شكوى تم التسجيل`)  
    - Email notification (`Send a message2`).

  - **About Office**  
    - Send static text about office licenses, nationalities, prices, insurance (`حضّر الإرسال – لالمكتب`)  
    - Log outgoing message.

  - **Talk to Agent**  
    - Send “please wait” message (`حضّر الإرسال – الموظف`)  
    - Email notification to staff (`Send a message3`).

#### 1.9 Google Sheets CRM Integration and Email Notifications

- **Overview:**  
  Stores all customer requests, transfers, complaints in dedicated Google Sheets tabs and sends email notifications for staff awareness.

- **Nodes Involved:**  
  - `إدخال طلب جديد` (New requests sheet)  
  - `إدخال نقل خادمات` and `إدخال نقل خادمات1` (Transfers sheet)  
  - `إدخال نقل خادمات1` (Complaints sheet)  
  - `Send a message`, `Send a message1`, `Send a message2`, `Send a message3` (Gmail nodes for email notifications)  

- **Node Details:**  
  - Sheets are identified by document ID and sheet GID, with columns mapped for phone number, client name, nationality, status, or complaint type.  
  - Email nodes send formatted messages to `fahadrecr@gmail.com` including client details and request type.  
  - Credentials: Google Sheets OAuth2 & Gmail OAuth2.

#### 1.10 Outgoing Message Preparation and Sending

- **Overview:**  
  Prepares WhatsApp message payloads in JSON, logs message metadata, and sends via WhatsApp Business API.

- **Nodes Involved:**  
  - Multiple `حضّر الإرسال – *` code nodes for message payloads  
  - Multiple `سجل رسالة خارجة*` code nodes to log outgoing messages  
  - Multiple HTTP Request nodes to send messages  
  - `Respond to Webhook` nodes to acknowledge WhatsApp API  

- **Node Details:**  
  - Message payloads support text and interactive list messages.  
  - Outgoing messages are logged with text extraction and metadata.  
  - HTTP Requests use Bearer token for authorization.  
  - Responses to WhatsApp webhook confirm successful processing.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                                             | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                                       |
|-------------------------------|---------------------|-------------------------------------------------------------|-------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook             | Receives WhatsApp messages                                  | -                             | Verify Webhook2, If5                  | WhatsApp Webhook Entry Points: handles all message types, verification, and routes to extraction                   |
| Verify Webhook2               | Respond to Webhook  | Responds to webhook verification requests                   | Webhook                       | -                                    |                                                                                                                  |
| استخراج البيانات               | Code                | Parses incoming WhatsApp messages into structured data      | If5                           | في وسائط؟                            | Message Data Extraction: extracts contact, message, media info uniformly                                         |
| في وسائط؟                     | IF                  | Checks if media content is present                           | استخراج البيانات               | Action in an app, تنزيل الوسائط        | Media File Management: manages detection of media presence                                                      |
| Action in an app              | HTTP Request        | Fetches media URL and metadata from WhatsApp API            | في وسائط؟                     | HTTP Request                         |                                                                                                                  |
| HTTP Request                 | HTTP Request        | Downloads media file as binary                               | Action in an app              | حفظ الميديا                          |                                                                                                                  |
| حفظ الميديا                   | Code                | Converts media to base64 and prepares for storage           | HTTP Request                  | Upsert row(s)                       |                                                                                                                  |
| Upsert row(s)                | Data Table          | Stores media metadata and content in Data Table             | حفظ الميديا                   | Code in JavaScript4                  |                                                                                                                  |
| Code in JavaScript4          | Code                | Prepares cleaned media metadata for further use             | Upsert row(s)                 | Code in JavaScript1                  |                                                                                                                  |
| Code in JavaScript1          | Code                | Tracks menu labels and bot enablement per user              | استخراج البيانات               | bot on?                            | Bot Enable Check & Session Tracking: tracks menu state and bot activation                                        |
| bot on?                     | IF                  | Routes processing based on bot enabled status               | Code in JavaScript1           | اقرأ الحالة, Respond to Webhook      |                                                                                                                  |
| اقرأ الحالة                   | Code                | Reads current user session state                             | bot on?                      | رسالة جديدة؟                        |                                                                                                                  |
| رسالة جديدة؟                 | IF                  | Checks if message is new (no interactive reply)             | اقرأ الحالة                   | Menu Cooldown Check, sub route       | Main Menu Display: controls menu sending based on message type                                                  |
| Menu Cooldown Check          | Code                | Enforces 5-minute cooldown for main menu display            | رسالة جديدة؟                 | If3                               |                                                                                                                  |
| If3                         | IF                  | Checks if menu sending is allowed (cooldown passed)         | Menu Cooldown Check           | حضّر الإرسال – القائمة الرئيسية        |                                                                                                                  |
| حضّر الإرسال – القائمة الرئيسية | Code                | Prepares interactive main menu message payload              | If3                          | سجل رسالة خارجة (عام)                |                                                                                                                  |
| سجل رسالة خارجة (عام)         | Code                | Logs outgoing main menu message                              | حضّر الإرسال – القائمة الرئيسية | إرسال القائمة الرئيسية                |                                                                                                                  |
| إرسال القائمة الرئيسية       | HTTP Request        | Sends main menu message via WhatsApp API                    | سجل رسالة خارجة (عام)          | Respond to Webhook                  |                                                                                                                  |
| Respond to Webhook           | Respond to Webhook  | Sends HTTP 200 response to WhatsApp                          | إرسال القائمة الرئيسية         | -                                    |                                                                                                                  |
| sub route                   | Switch              | Routes nested interactive replies (price_*, req_*, etc.)    | رسالة جديدة؟                 | Get row(s), السير, حضّر الإرسال – شكوى تم التسجيل, menu route | Smart Menu Navigation: routes sub-menu actions                                                                  |
| menu route                  | Switch              | Routes main menu selections to service flows                 | sub route                    | حضّر الإرسال – الأسعار, حضّر الإرسال – طلب جديد, ... |                                                                                                                  |
| Get row(s)                  | Data Table          | Retrieves price data for selected nationality                | sub route                    | If                               |                                                                                                                  |
| السير                       | Code                | Provides static CV link data per nationality                 | sub route                    | If1                              |                                                                                                                  |
| If                         | IF                  | Checks nationality match for price or CV link               | Get row(s), السير            | حضّر الإرسال – الأسعار, حضّر إرسال - رسالة السير الذاتية |                                                                                                                  |
| If1                        | IF                  | Checks nationality match for new request CV link            | السير                        | رسالة السير                       |                                                                                                                  |
| حضّر الإرسال – الأسعار       | Code                | Prepares price inquiry WhatsApp message                      | menu route, If               | سجل رسالة خارجة4                   |                                                                                                                  |
| سجل رسالة خارجة4             | Code                | Logs outgoing price inquiry message                          | حضّر الإرسال – الأسعار         | الأسعار - اختر جنسية               |                                                                                                                  |
| الأسعار - اختر جنسية         | HTTP Request        | Sends price inquiry message to WhatsApp                      | سجل رسالة خارجة4             | Respond to Webhook                 |                                                                                                                  |
| حضّر الإرسال – طلب جديد       | Code                | Prepares nationality selection list for new request         | menu route                   | سجل رسالة خارجة1                   |                                                                                                                  |
| سجل رسالة خارجة1             | Code                | Logs outgoing new request nationality selection              | حضّر الإرسال – طلب جديد         | طلب جديد - اختر جنسية              |                                                                                                                  |
| طلب جديد - اختر جنسية        | HTTP Request        | Sends new request nationality selection message             | سجل رسالة خارجة1             | Respond to Webhook                 |                                                                                                                  |
| تحضير إرسال - رسالة السير الذاتية | Code                | Prepares CV link message for selected nationality            | If1                         | سجل رسالة خارجة14                 |                                                                                                                  |
| سجل رسالة خارجة14            | Code                | Logs outgoing CV link message                                | تحضير إرسال - رسالة السير الذاتية | رسالة السير                       |                                                                                                                  |
| رسالة السير                 | HTTP Request        | Sends CV link message to WhatsApp                            | سجل رسالة خارجة14            | إدخال طلب جديد, Respond to Webhook  |                                                                                                                  |
| إدخال طلب جديد             | Google Sheets       | Saves new request data to “طلبات جديدة” sheet               | رسالة السير                 | Wait                             |                                                                                                                  |
| Wait                       | Wait                | Delays to allow backend processing                           | إدخال طلب جديد              | تحضير إرسال - رسالة السير الذاتية1    |                                                                                                                  |
| تحضير إرسال - رسالة السير الذاتية1 | Code                | Prepares confirmation message after new request save        | Wait                        | سجل رسالة خارجة10                 |                                                                                                                  |
| سجل رسالة خارجة10            | Code                | Logs confirmation message                                    | تحضير إرسال - رسالة السير الذاتية1 | رسالة تأكيد السيرة               |                                                                                                                  |
| رسالة تأكيد السيرة           | HTTP Request        | Sends confirmation message to WhatsApp                       | سجل رسالة خارجة10            | Respond to Webhook                 |                                                                                                                  |
| Send a message              | Gmail               | Sends email notification of new request to staff            | Wait                        | -                                |                                                                                                                  |
| حضّر الإرسال – طلب متابعة    | Code                | Prepares “please wait” text message for request tracking    | menu route                   | سجل رسالة خارجة5                   |                                                                                                                  |
| سجل رسالة خارجة5             | Code                | Logs outgoing tracking wait message                          | حضّر الإرسال – طلب متابعة      | متابعة                          |                                                                                                                  |
| متابعة                     | Google Sheets       | Looks up request status by phone number                      | متابعة - اطلب رقم            | Code in JavaScript                 |                                                                                                                  |
| Code in JavaScript          | Code                | Checks if request found, sets found flag                     | متابعة                      | If2                              |                                                                                                                  |
| If2                        | IF                  | Routes based on found flag to send status or error message  | Code in JavaScript           | تحضير إرسال - رسالة المتابعة, تحضير إرسال - رسالة المتابعة لا يوجد |                                                                                                                  |
| تحضير إرسال - رسالة المتابعة  | Code                | Prepares status message for found request                    | If2 (true)                   | سجل رسالة خارجة11                 |                                                                                                                  |
| سجل رسالة خارجة11            | Code                | Logs status message                                         | تحضير إرسال - رسالة المتابعة  | رسالة المتابعة                   |                                                                                                                  |
| رسالة المتابعة               | HTTP Request        | Sends status message                                        | سجل رسالة خارجة11            | Respond to Webhook                 |                                                                                                                  |
| تحضير إرسال - رسالة المتابعة لا يوجد | Code                | Prepares error message for not found request                | If2 (false)                  | سجل رسالة خارجة12                 |                                                                                                                  |
| سجل رسالة خارجة12            | Code                | Logs error message                                         | تحضير إرسال - رسالة المتابعة لا يوجد | رسالة المتابعة1                  |                                                                                                                  |
| رسالة المتابعة1              | HTTP Request        | Sends error message                                        | سجل رسالة خارجة12            | Respond to Webhook                 |                                                                                                                  |
| حضّر الإرسال – نقل الخادمات   | Code                | Prepares transfer service link message                      | menu route                   | سجل رسالة خارجة6                   |                                                                                                                  |
| سجل رسالة خارجة6             | Code                | Logs transfer link message                                  | حضّر الإرسال – نقل الخادمات    | نقل خادمات - اختر جنسية           |                                                                                                                  |
| نقل خادمات - اختر جنسية      | HTTP Request        | Sends transfer nationality selection message               | سجل رسالة خارجة6             | Respond to Webhook, نقل خادمات من الشيت |                                                                                                                  |
| نقل خادمات من الشيت           | Google Sheets       | Reads transfer nationality data                             | نقل خادمات - اختر جنسية       | إدخال نقل خادمات                 |                                                                                                                  |
| إدخال نقل خادمات           | Google Sheets       | Saves transfer request data to “نقل الخادمات” sheet        | نقل خادمات من الشيت          | Wait1                            |                                                                                                                  |
| Wait1                      | Wait                | Delays for backend processing                               | إدخال نقل خادمات            | حضر الارسال - نقل الخادمات          |                                                                                                                  |
| حضر الارسال - نقل الخادمات    | Code                | Prepares transfer confirmation message                      | Wait1                       | سجل رسالة خارجة13                 |                                                                                                                  |
| سجل رسالة خارجة13            | Code                | Logs transfer confirmation message                          | حضر الارسال - نقل الخادمات    | رسالة تأكيد طلب النقل الخادمات      |                                                                                                                  |
| رسالة تأكيد طلب النقل الخادمات | HTTP Request        | Sends transfer confirmation message                         | سجل رسالة خارجة13            | Respond to Webhook                 |                                                                                                                  |
| Send a message1             | Gmail               | Sends transfer request email notification                   | Wait1                       | -                                |                                                                                                                  |
| حضّر الإرسال – الترجمة        | Code                | Prepares static translation service info message           | menu route                   | سجل رسالة خارجة7                   |                                                                                                                  |
| سجل رسالة خارجة7             | Code                | Logs translation message                                   | حضّر الإرسال – الترجمة         | الترجمة - رابط                   |                                                                                                                  |
| الترجمة - رابط               | HTTP Request        | Sends translation service message                          | سجل رسالة خارجة7             | Respond to Webhook                 |                                                                                                                  |
| حضّر الإرسال – شكوى           | Code                | Prepares complaint type selection list                     | menu route                   | سجل رسالة خارجة2                   |                                                                                                                  |
| سجل رسالة خارجة2             | Code                | Logs complaint type list message                            | حضّر الإرسال – شكوى           | الشكاوى - اطلب تفاصيل             |                                                                                                                  |
| الشكاوى - اطلب تفاصيل         | HTTP Request        | Sends complaint details request message                    | سجل رسالة خارجة2             | Respond to Webhook                 |                                                                                                                  |
| إدخال نقل خادمات1          | Google Sheets       | Inserts or updates complaint records in “شكاوي” sheet      | Send a message2              | سجل رسالة خارجة3                   |                                                                                                                  |
| سجل رسالة خارجة3             | Code                | Logs complaint confirmation                                | إدخال نقل خادمات1            | الشكاوى - اطلب تفاصيل1            |                                                                                                                  |
| الشكاوى - اطلب تفاصيل1        | HTTP Request        | Sends complaint registration confirmation                  | سجل رسالة خارجة3             | Respond to Webhook, Send a message2 |                                                                                                                  |
| Send a message2             | Gmail               | Sends complaint notification email to staff                | الشكاوى - اطلب تفاصيل1        | -                                |                                                                                                                  |
| حضّر الإرسال – لالمكتب        | Code                | Prepares static info about office message                   | menu route                   | سجل رسالة خارجة8                   |                                                                                                                  |
| سجل رسالة خارجة8             | Code                | Logs office info message                                   | حضّر الإرسال – لالمكتب         | عن المكتب                      |                                                                                                                  |
| عن المكتب                   | HTTP Request        | Sends office info message to WhatsApp                      | سجل رسالة خارجة8             | Respond to Webhook                 |                                                                                                                  |
| حضّر الإرسال – الموظف         | Code                | Prepares human agent wait message                           | menu route                   | سجل رسالة خارجة9                   |                                                                                                                  |
| سجل رسالة خارجة9             | Code                | Logs human agent message                                   | حضّر الإرسال – الموظف          | التحدث مع موظف                 |                                                                                                                  |
| التحدث مع موظف              | HTTP Request        | Sends human agent wait message to WhatsApp                 | سجل رسالة خارجة9             | Respond to Webhook, Send a message3 |                                                                                                                  |
| Send a message3             | Gmail               | Sends human agent notification email to staff              | التحدث مع موظف              | -                                |                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Webhook`  
   - Path: `whatsapp` (supports GET and POST)  
   - Response Mode: `responseNode`  
   - Purpose: Receive incoming WhatsApp messages and verification requests.

2. **Create Respond to Webhook Node for Verification:**  
   - Name: `Verify Webhook2`  
   - Respond with text: `{{$json.query['hub.challenge'] || 'OK'}}`  
   - Connect `Webhook` GET flow to this node.

3. **Data Extraction Code Node:**  
   - Name: `استخراج البيانات`  
   - JavaScript to parse WhatsApp webhook JSON for messages, contacts, media, and interactive replies.  
   - Outputs array of normalized messages for each incoming message.  
   - Connect `Webhook` POST flow here.

4. **Media Presence IF Node:**  
   - Name: `في وسائط؟`  
   - Condition: Check if `media_id` is non-empty in message JSON.  
   - True → Fetch media URL and download media; False → skip media steps.

5. **Fetch Media URL HTTP Request:**  
   - Name: `Action in an app`  
   - Method: GET  
   - URL: `https://graph.facebook.com/v17.0/{{ $json.media_id }}?fields=url,mime_type`  
   - Auth: HTTP Bearer Token with WhatsApp API token.

6. **Download Media HTTP Request:**  
   - Name: `HTTP Request`  
   - URL: `={{ $json.url }}`  
   - Response format: file/binary  
   - Auth: Same Bearer token.

7. **Save Media Code Node:**  
   - Name: `حفظ الميديا`  
   - Converts binary to base64, prepares metadata like filename, mime, caption, timestamp.  
   - Outputs JSON for Data Table upsert.

8. **Upsert Media Data Table Node:**  
   - Name: `Upsert row(s)`  
   - Data Table: `media`  
   - Operation: Upsert by `media_id`.  
   - Map all media metadata columns.

9. **Prepare Media Metadata Code Node:**  
   - Name: `Code in JavaScript4`  
   - Reads media record, cleans and ensures consistent output.

10. **Label Tracker & Bot Enablement Code Node:**  
    - Name: `Code in JavaScript1`  
    - Tracks current label (menu section) per user and checks if bot is enabled globally or per user.

11. **Bot Enabled IF Node:**  
    - Name: `bot on?`  
    - Condition: `bot_enabled` equals true  
    - True → Continue; False → Respond to webhook and end.

12. **Session Read Code Node:**  
    - Name: `اقرأ الحالة`  
    - Reads current session state and timestamp from global storage.

13. **New Message IF Node:**  
    - Name: `رسالة جديدة؟`  
    - Condition: `interactive_reply` is empty → implies new message.

14. **Menu Cooldown Check Code Node:**  
    - Name: `Menu Cooldown Check`  
    - Checks 5-minute cooldown per user for menu sending.

15. **Cooldown IF Node:**  
    - Name: `If3`  
    - If cooldown passed → prepare main menu.

16. **Prepare Main Menu Code Node:**  
    - Name: `حضّر الإرسال – القائمة الرئيسية`  
    - Constructs WhatsApp interactive list message with 8 main options.

17. **Log Outgoing Message Code Node:**  
    - Name: `سجل رسالة خارجة (عام)`  
    - Logs outgoing main menu message.

18. **Send Main Menu HTTP Request:**  
    - Name: `إرسال القائمة الرئيسية`  
    - POST to WhatsApp API with Bearer token.

19. **Respond to Webhook Node:**  
    - Name: `Respond to Webhook`  
    - Responds with status success JSON.

20. **Menu Routing Switch Node:**  
    - Name: `menu route`  
    - Routes based on exact `interactive_reply` value.

21. **Sub Route Switch Node:**  
    - Name: `sub route`  
    - Routes based on prefix of `interactive_reply` (price_, req_, complaint_, m_).

22. **Static Data Code Node for CV Links:**  
    - Name: `السير`  
    - Returns static nationality-to-CV link mappings.

23. **Google Sheets Nodes for CRM:**  
    - Create Google Sheets nodes for “طلبات جديدة”, “نقل الخادمات”, and “شكاوي”.  
    - Configure columns and mapping for phone number, client name, status, complaint, etc.  
    - Operation: Append or Update keyed by phone number.

24. **Prepare Outgoing Messages for Each Service:**  
    - Create multiple code nodes to prepare WhatsApp message payloads for prices, new requests, tracking, transfers, translation, complaints, about office, human agent.

25. **Send Outgoing Messages HTTP Request Nodes:**  
    - Multiple HTTP Request nodes configured with WhatsApp API endpoint and Bearer token.

26. **Log Outgoing Messages for Each Service:**  
    - Multiple code nodes to log outgoing messages with text extraction and metadata.

27. **Email Notification Nodes:**  
    - Create Gmail nodes to send emails to staff with request/complaint details.

28. **Wait Nodes:**  
    - Insert Wait nodes to delay for backend processing before sending confirmations.

29. **Webhook Response Nodes:**  
    - Place Respond to Webhook nodes after each outgoing message to acknowledge WhatsApp.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| Workflow is part of a larger system with a dashboard for message display, data saving, and human escalation.                                                    | Sticky Note4                                                           |
| WhatsApp webhook entry points support text, interactive replies, media, and webhook verification.                                                               | Sticky Note3                                                           |
| Media handling supports images, video, audio, voice notes, documents; preserves original filenames and mime types; stores media in Data Table.                  | Sticky Note1                                                           |
| Bot enable check includes global on/off switch and per-user overrides; maintains session states and labels for menu navigation.                                  | Sticky Note2                                                           |
| Conversation logging keeps last 5000 messages with metadata for audit, analytics, debugging, and customer history.                                              | Sticky Note5                                                           |
| Main menu includes 8 interactive options with cooldown to prevent spam; all outgoing messages are logged and sent via WhatsApp API with Bearer token.           | Sticky Note6                                                           |
| Menu routing uses main and sub-routing switches to handle nested menu options and prepare/send appropriate responses.                                           | Sticky Note7                                                           |
| Service flows integrate Google Sheets as CRM for requests, transfers, complaints; send email notifications to staff; maintain state and provide confirmations.  | Sticky Note8                                                           |

---

This completes a full analysis and reproduction guide for the "Interactive Recruitment Customer Service with WhatsApp, Google Sheets CRM & Notifications" workflow. It enables advanced users and AI agents to understand, maintain, and extend the workflow confidently.