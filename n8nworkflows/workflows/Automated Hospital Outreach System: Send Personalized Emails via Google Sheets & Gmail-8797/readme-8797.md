Automated Hospital Outreach System: Send Personalized Emails via Google Sheets & Gmail

https://n8nworkflows.xyz/workflows/automated-hospital-outreach-system--send-personalized-emails-via-google-sheets---gmail-8797


# Automated Hospital Outreach System: Send Personalized Emails via Google Sheets & Gmail

### 1. Workflow Overview

This workflow, titled **Automated Hospital Outreach System: Send Personalized Emails via Google Sheets & Gmail**, automates personalized outreach emails to hospitals within the Philippines. It is designed for users who want to send targeted email messages to multiple hospitals based on their geographic region (Luzon, Visayas, Mindanao). The workflow accepts a chat message where the first line specifies the region and subsequent lines list hospital names. It then retrieves hospital contact information from a Google Sheet segmented by region, composes a personalized email, and sends it via Gmail.

The workflow’s logic is divided into these main functional blocks:

- **1.1 Input Reception & Parsing:** Receives chat input listing region and hospital names; parses this input into structured hospital entries.
- **1.2 Batching:** Processes hospitals in batches to manage load and sequencing.
- **1.3 Region-Based Data Lookup:** Routes each hospital entry to the appropriate Google Sheets node depending on the specified region for contact data retrieval.
- **1.4 Email Composition & Delivery:** Sends personalized emails using data retrieved from Google Sheets.
- **1.5 Setup & User Guidance:** Sticky notes provide setup instructions, usage guidelines, and tutorial resources.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parsing

**Overview:**  
This block captures the user’s chat message input, which contains the target region and a list of hospital names. It parses this input text into structured JSON objects, each representing a hospital with its associated region.

**Nodes Involved:**  
- When chat message received  
- Hospital Parser

**Node Details:**

- **When chat message received**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Entry point webhook that triggers the workflow upon receiving a chat message.  
  - *Configuration:* Listens for incoming chat messages; no special filters.  
  - *Expressions:* None.  
  - *Connections:* Outputs to Hospital Parser.  
  - *Potential Failures:* Network issues, webhook misconfiguration, rate limits.

- **Hospital Parser**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses raw chat input text into structured data items, extracting region and hospital names line-by-line.  
  - *Configuration:* Splits input by newline, trims empty lines, assigns first line as region, remaining lines as hospital names.  
  - *Expressions:* Uses `$input.first().json.chatInput` to access incoming chat message.  
  - *Connections:* Outputs an array of JSON objects, each with `region` and `hospital` fields to Batch Sender.  
  - *Potential Failures:* Unexpected input format causing empty or invalid data; missing region line.

#### 2.2 Batching

**Overview:**  
Manages the processing of hospital entries in batches, enabling controlled and sequential handling of messages.

**Nodes Involved:**  
- Batch Sender

**Node Details:**

- **Batch Sender**  
  - *Type:* SplitInBatches  
  - *Role:* Splits incoming hospital entries into batches for sequential processing.  
  - *Configuration:* Default batch size (not explicitly set, so defaults apply), no reset on completion.  
  - *Expressions:* None.  
  - *Connections:* First output (empty) unused, second output feeds Region Switcher.  
  - *Potential Failures:* Large batch sizes could cause timeouts; batch size misconfiguration.

#### 2.3 Region-Based Data Lookup

**Overview:**  
Routes each hospital entry to the correct Google Sheets node based on the region to fetch hospital-specific contact information such as email.

**Nodes Involved:**  
- Region Switcher  
- LUZON FILES  
- VISAYAS FILES  
- MINDANAO

**Node Details:**

- **Region Switcher**  
  - *Type:* Switch  
  - *Role:* Directs each hospital JSON to the corresponding regional Google Sheet node based on the `region` value.  
  - *Configuration:* Case-insensitive string containment check for `"LUZON"`, `"VISAYAS"`, or `"MINDANAO"` in the `region` field.  
  - *Expressions:* Uses `{{$json.region}}` in conditions.  
  - *Connections:* Routes to one of LUZON FILES, VISAYAS FILES, or MINDANAO.  
  - *Potential Failures:* Region string mismatches or typos leading to no output; case sensitivity handled but partial matches could cause errors.

- **LUZON FILES**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves hospital data from the Luzon sheet tab filtered by hospital name.  
  - *Configuration:*  
    - Google Sheet ID: `1dNEKLvwSBgUvMJd1ekxz-R5_RN3ewlJTCOIHW4-bhIQ`  
    - Sheet name: Luzon  
    - Filter: Hospital Name equals `{{$json.hospital}}`  
  - *Expressions:* Uses dynamic filter on `Hospital Name` column.  
  - *Connections:* Output to Send Gmail Message.  
  - *Potential Failures:* Google Sheets API errors, incorrect sheet ID or name, no matching hospital found.

- **VISAYAS FILES**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves hospital data from the Visayas sheet tab filtered by hospital name.  
  - *Configuration:* Same as LUZON FILES but for Visayas tab.  
  - *Connections:* Output to Send Gmail Message.  
  - *Potential Failures:* Same as Luzon Files.

- **MINDANAO**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves hospital data from the Mindanao sheet tab. Unlike Luzon and Visayas, no filter is applied, so it retrieves the entire sheet (likely for one hospital per batch).  
  - *Configuration:* Sheet name Mindanao, no filter applied.  
  - *Connections:* Output to Send Gmail Message.  
  - *Potential Failures:* If multiple rows exist, may cause redundant emails; no filtering may lead to wrong data selection.

#### 2.4 Email Composition & Delivery

**Overview:**  
Constructs and sends a personalized Gmail message to each hospital using data fetched from the Google Sheet.

**Nodes Involved:**  
- Send Gmail Message

**Node Details:**

- **Send Gmail Message**  
  - *Type:* Gmail  
  - *Role:* Sends personalized emails to the hospital’s main email address.  
  - *Configuration:*  
    - Recipient: `{{$json['Main Email']}}` retrieved from sheet row.  
    - Subject: `"Letter of Intent: {{$json['Hospital Name']}}"`  
    - Message body: HTML formatted message with placeholders filled from Google Sheet data fields such as Hospital Name, Your Name, Credentials, Sample Hospitals, Video Links, and Contact Info.  
  - *Expressions:* Multiple `{{ $json[...] }}` placeholders for dynamic content.  
  - *Connections:* Outputs back to Batch Sender for processing next batch.  
  - *Potential Failures:* Gmail API quota exceeded, invalid email addresses, missing required fields, network errors.

#### 2.5 Setup & User Guidance

**Overview:**  
Provides user instructions, setup notes, and tutorial resources via sticky notes in the workflow.

**Nodes Involved:**  
- Sticky Note (Main)  
- Sticky Note1  
- Sticky Note2

**Node Details:**

- **Sticky Note (Main)**  
  - *Content:* Overview describing how to use the workflow: chat message format, regions, and hospital names.  
- **Sticky Note1**  
  - *Content:* Setup instructions including credentials setup for Google Sheets and Gmail, replacing Google Sheet ID, customizing email template, and usage steps.  
- **Sticky Note2**  
  - *Content:* Link to a YouTube tutorial video: https://www.youtube.com/embed/5u9W-Iegq6k

---

### 3. Summary Table

| Node Name               | Node Type                    | Functional Role                         | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                 |
|-------------------------|------------------------------|---------------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger      | Entry point receiving chat input      | -                        | Hospital Parser          | See main sticky note for chat input format and workflow overview.                                           |
| Hospital Parser          | Code (JavaScript)            | Parses chat message into hospital entries | When chat message received | Batch Sender             |                                                                                                             |
| Batch Sender             | SplitInBatches               | Processes hospital entries in batches | Hospital Parser           | Region Switcher          |                                                                                                             |
| Region Switcher          | Switch                      | Routes by region to correct sheet     | Batch Sender              | LUZON FILES, VISAYAS FILES, MINDANAO |                                                                                                             |
| LUZON FILES              | Google Sheets                | Retrieves hospital data for Luzon     | Region Switcher           | Send Gmail Message       | Requires Google Sheets credentials and correct Sheet ID/name.                                               |
| VISAYAS FILES            | Google Sheets                | Retrieves hospital data for Visayas   | Region Switcher           | Send Gmail Message       | Same as above.                                                                                              |
| MINDANAO                 | Google Sheets                | Retrieves hospital data for Mindanao  | Region Switcher           | Send Gmail Message       | No filter applied; ensure data correctness.                                                                 |
| Send Gmail Message       | Gmail                       | Sends personalized email              | LUZON FILES, VISAYAS FILES, MINDANAO | Batch Sender             | Requires Gmail credentials; customize email template as needed.                                             |
| Sticky Note              | Sticky Note                  | Workflow overview and chat format     | -                        | -                       | "Hospital Outreach Automation" overview.                                                                    |
| Sticky Note1             | Sticky Note                  | Setup guide and usage instructions    | -                        | -                       | Setup required steps for credentials and sheet ID replacement.                                              |
| Sticky Note2             | Sticky Note                  | Tutorial video link                   | -                        | -                       | YouTube tutorial: https://www.youtube.com/embed/5u9W-Iegq6k                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node:**  
   - Type: LangChain Chat Trigger  
   - No special parameters needed; this node will listen for incoming chat messages containing region and hospital names.

2. **Create "Hospital Parser" node:**  
   - Type: Code (JavaScript)  
   - Paste the following code to parse chat input:  
     ```javascript
     const rawText = $input.first().json.chatInput;
     const lines = rawText.split('\n').filter(line => line.trim() !== '');
     const region = lines[0] || 'Unknown';
     const hospitals = lines.slice(1);

     return hospitals.map(h => ({
       json: {
         region: region,
         hospital: h
       }
     }));
     ```  
   - Connect output of "When chat message received" to this node.

3. **Create "Batch Sender" node:**  
   - Type: SplitInBatches  
   - Use default batch size (customize if needed).  
   - Connect output of "Hospital Parser" to this node.

4. **Create "Region Switcher" node:**  
   - Type: Switch  
   - Configure three rules (case-insensitive contains):  
     - If `{{$json.region}}` contains "LUZON" → Output 1  
     - If `{{$json.region}}` contains "VISAYAS" → Output 2  
     - If `{{$json.region}}` contains "MINDANAO" → Output 3  
   - Connect second output of "Batch Sender" to "Region Switcher".

5. **Create three Google Sheets nodes for each region:**

   - **LUZON FILES:**  
     - Type: Google Sheets  
     - Document ID: Use your Google Sheet ID (default in example: `1dNEKLvwSBgUvMJd1ekxz-R5_RN3ewlJTCOIHW4-bhIQ`)  
     - Sheet Name: Luzon  
     - Filters: Filter rows where "Hospital Name" equals `{{$json.hospital}}`  
     - Connect output 1 of "Region Switcher" to this node.

   - **VISAYAS FILES:**  
     - Same as Luzon FILES but Sheet Name: Visayas  
     - Filter by "Hospital Name" equals `{{$json.hospital}}`  
     - Connect output 2 of "Region Switcher" to this node.

   - **MINDANAO:**  
     - Google Sheets node  
     - Same Document ID  
     - Sheet Name: Mindanao  
     - No filter applied (fetch whole sheet or customize as needed)  
     - Connect output 3 of "Region Switcher" to this node.

6. **Create "Send Gmail Message" node:**  
   - Type: Gmail  
   - Configure credentials for Gmail OAuth2.  
   - Set “To” field to `{{$json['Main Email']}}` from Google Sheet data.  
   - Set Subject: `"Letter of Intent: {{$json['Hospital Name']}}"`  
   - Set Message Body as HTML with placeholders, e.g.:  
     ```
     <p>Dear {{ $json["Hospital Name"] }},</p>
     <p>My name is {{ $json["Your Name"] }}, a {{ $json["Your Credentials"] }}.</p>
     <p>During a recent hospital admission, I personally experienced the challenges of paper-based real-time billing...</p>
     <p>Video links and other details...</p>
     <p>--<br>Best regards,<br>{{ $json["Your Name"] }}<br>{{ $json["Your Contact"] }}</p>
     ```
   - Connect outputs of all three Google Sheets nodes (LUZON FILES, VISAYAS FILES, MINDANAO) to this node.

7. **Connect output of "Send Gmail Message" back to the first output of "Batch Sender"** to continue batch processing.

8. **Set up credentials:**  
   - Google Sheets (API credentials with access to the specified spreadsheet).  
   - Gmail OAuth2 credentials with permission to send emails.

9. **Optional: Add Sticky Notes:**  
   - Add sticky notes for instructions, setup, and tutorial links as per content described in the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                           |
|-------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Hospital Outreach Automation: Send chat message with first line as region (LUZON, VISAYAS, MINDANAO) followed by hospital names. | Workflow usage overview (Sticky Note).                     |
| Setup required: Configure Google Sheets & Gmail credentials; replace Google Sheet ID with your own; customize email template. | Setup instructions (Sticky Note1).                         |
| YouTube tutorial video available at: https://www.youtube.com/embed/5u9W-Iegq6k                                           | Workflow tutorial (Sticky Note2).                           |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow built with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected material. All data processed is legal and public.