UTM Marketing Attribution Reports with Google Sheets, GPT-4o & Gmail

https://n8nworkflows.xyz/workflows/utm-marketing-attribution-reports-with-google-sheets--gpt-4o---gmail-8513


# UTM Marketing Attribution Reports with Google Sheets, GPT-4o & Gmail

---

### 1. Workflow Overview

This workflow automates the generation and distribution of **weekly lead attribution reports** based on UTM/source data collected in a Google Sheet. Its core purpose is to analyze incoming lead data, aggregate key marketing metrics such as lead counts and cost per lead (CPL) by source, leverage GPT-4o (Azure OpenAI) to create a professional HTML report, and email the report via Gmail.

The workflow is logically divided into these blocks:

- **1.1 Schedule Trigger:** Periodically starts the workflow to fetch new lead data.
- **1.2 Data Retrieval from Google Sheets:** Extracts raw lead data from a designated sheet.
- **1.3 Data Aggregation & Normalization:** Processes raw sheet data to count leads per source, aggregate marketing spend, and compute CPL.
- **1.4 AI Report Generation:** Uses an AI language model chain with GPT-4o-mini to transform aggregated data into a well-formatted, business-professional HTML report.
- **1.5 Email Dispatch:** Sends the generated report as an HTML email to a predefined recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow automatically every hour to ensure reports reflect recent lead data.

- **Nodes Involved:**  
  - `Schedule Trigger`

- **Node Details:**  
  - **Type & Role:** Schedule Trigger — starts the workflow on a timed interval.  
  - **Configuration:** Set to trigger every hour (interval in hours).  
  - **Inputs/Outputs:** No inputs; outputs proceed to data retrieval node.  
  - **Edge Cases:** If the workflow is paused or the n8n instance is down, scheduled runs will not occur. Time zone misconfiguration can cause unexpected trigger times.  
  - **Sticky Note:** Explains this node’s role as starting point and timing.

#### 1.2 Data Retrieval from Google Sheets

- **Overview:**  
  Pulls all lead records from a specific Google Sheet tab ("Form responses 1"), including columns like Name, Email, Phone, Source, and spend data.

- **Nodes Involved:**  
  - `Get row(s) in sheet`

- **Node Details:**  
  - **Type & Role:** Google Sheets node — reads rows from a designated sheet tab.  
  - **Configuration:**  
    - Document ID and Sheet Name explicitly selected from a Google Sheet named "Source/UTM Attribution and Reporting( testing)".  
    - No filters or limits, so it retrieves all rows.  
  - **Credentials:** Uses OAuth2 credentials for Google Sheets API access.  
  - **Input/Output:** Input from Schedule Trigger; outputs raw rows to the first Code node.  
  - **Edge Cases:**  
    - API rate limits or credential expiration may cause errors.  
    - Sheet structure changes (renamed columns) can break downstream logic.  
    - Large datasets may cause timeouts or performance degradation.  
  - **Sticky Note:** Describes the sheet and data pulled.

#### 1.3 Data Aggregation & Normalization

- **Overview:**  
  Applies two sequential JavaScript Code nodes to clean, normalize, and aggregate the raw lead data. It collects total leads, counts per source, marketing spend by platform, and computes CPL.

- **Nodes Involved:**  
  - `Code1`  
  - `Code`

- **Node Details:**  

  - **Code1:**  
    - **Type & Role:** Code node — normalizes keys (trims, lowercase), counts leads per source, sums spend per channel (Instagram, LinkedIn, Google Ads), and calculates CPL.  
    - **Key Logic:**  
      - Iterates through each row, normalizes properties for consistency.  
      - Accumulates counts and spend per source.  
      - Calculates cost per lead (spend divided by leads) per source; outputs "N/A" if spend or leads are zero.  
    - **Input/Output:** Input from Google Sheets node; output is an aggregated JSON object with total leads, sources, spend, and CPL.  
    - **Edge Cases:**  
      - Missing or malformed spend fields may cause NaN or incorrect CPL calculations.  
      - Sources not in the predefined list have no spend accumulation.  
      - Empty data leads to zero totals and "N/A" CPL.  
    - **Sticky Note:** Details normalization and aggregation steps.

  - **Code:**  
    - **Type & Role:** Code node — further condenses the aggregated data into a summary object focusing on total leads and count per source, preserving the full leads list for reference.  
    - **Key Logic:**  
      - Maps all input data into a single summary JSON with total leads, counts by source, and full leads.  
    - **Input/Output:** Input from Code1; output feeds into the AI report generation block.  
    - **Edge Cases:** Similar to above; ensures only one summarizing report is generated instead of multiple per lead.  
    - **Sticky Note:** Explains final aggregation and summary generation.

#### 1.4 AI Report Generation

- **Overview:**  
  Sends the aggregated JSON data to a LangChain LLM node that uses GPT-4o-mini to generate a professional weekly lead attribution report in responsive HTML format.

- **Nodes Involved:**  
  - `Basic LLM Chain`  
  - `Azure OpenAI Chat Model1`

- **Node Details:**  

  - **Basic LLM Chain:**  
    - **Type & Role:** LangChain LLM chain node — formats the prompt to instruct GPT-4o to create an HTML report from JSON data.  
    - **Configuration:**  
      - Prompt includes explicit instructions for layout (card style, sections, mobile-friendly, emojis) and content (Summary, Key Metrics, Top Sources, CPL, Insights, Priority Actions).  
      - Uses a message role to set the assistant’s persona as a professional business reporting assistant.  
    - **Input/Output:** Input from summary Code node; outputs generated HTML text.  
    - **Edge Cases:**  
      - If input JSON is malformed or empty, the output may be irrelevant or empty.  
      - LLM API failures or token limits can cause errors or truncated output.  
    - **Sticky Note:** Describes prompt and output expectations.

  - **Azure OpenAI Chat Model1:**  
    - **Type & Role:** Azure OpenAI GPT-4o-mini model node — the actual language model providing text generation.  
    - **Configuration:** Uses GPT-4o-mini model with default options.  
    - **Credentials:** Azure OpenAI API credentials required.  
    - **Input/Output:** Connected as the AI engine for the Basic LLM Chain.  
    - **Edge Cases:** API quota limits, authentication failures, or latency issues.  
    - **Sticky Note:** Specifies this is the LLM engine node.

#### 1.5 Email Dispatch

- **Overview:**  
  Sends the generated HTML report via Gmail to a fixed recipient with a weekly reports subject line.

- **Nodes Involved:**  
  - `Send Follow-up Email1`

- **Node Details:**  
  - **Type & Role:** Gmail node — sends an email with dynamic HTML content.  
  - **Configuration:**  
    - Recipient: newscctv22@gmail.com  
    - Subject: "weekely reports" (note: typo preserved)  
    - Message body: HTML output from the Basic LLM Chain node.  
  - **Credentials:** Uses OAuth2 credentials for Gmail.  
  - **Input/Output:** Input from Basic LLM Chain; no outputs (end node).  
  - **Edge Cases:**  
    - SMTP errors, authentication failures, or quota issues.  
    - If the HTML content is malformed, email clients may render poorly.  
  - **Sticky Note:** Describes email purpose and content.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                         | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                   |
|-------------------------|--------------------------------|---------------------------------------|--------------------------|--------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger               | Initiates workflow every hour         |                          | Get row(s) in sheet      | Runs the workflow at a set interval (currently every hour). Acts as the starting point.       |
| Get row(s) in sheet    | Google Sheets                  | Retrieves lead data from Google Sheet | Schedule Trigger          | Code1                    | Pulls lead data from your Google Sheet (Form responses 1). Includes Name, Email, Phone, Source, spend. |
| Code1                  | Code                          | Normalizes and aggregates raw data    | Get row(s) in sheet       | Code                     | Normalizes sheet data, aggregates lead counts per source, collects spend, computes CPL.       |
| Code                   | Code                          | Summarizes aggregated lead data       | Code1                     | Basic LLM Chain          | Further aggregates leads into one summary object to prepare for report generation.            |
| Basic LLM Chain        | LangChain LLM Chain           | Generates professional HTML report    | Code                      | Send Follow-up Email1     | Takes JSON data, prompts AI to create a weekly lead attribution report in responsive HTML.    |
| Azure OpenAI Chat Model1 | LangChain Azure OpenAI Model | GPT-4o-mini LLM engine                 |                          | Basic LLM Chain (AI model) | The actual GPT-4o-mini engine providing language understanding and text generation.           |
| Send Follow-up Email1  | Gmail                         | Sends generated report via email      | Basic LLM Chain           |                          | Sends the generated HTML report to designated recipient with subject "weekely reports".        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Parameters: Set interval to every 1 hour (field: hours)  
   - This will start the workflow automatically every hour.

2. **Add a Google Sheets node ("Get row(s) in sheet")**  
   - Type: Google Sheets  
   - Credentials: Connect with OAuth2 Google Sheets credentials.  
   - Parameters:  
     - Document ID: Use your Google Sheet ID (e.g., "1BqdjkHQrA9Pz_td4N9H7pL5T8Poh0cL8ERXN_N3ogeU")  
     - Sheet Name: Select the sheet tab containing lead data ("Form responses 1").  
     - Operation: "Get rows" (default).  
   - Connect Schedule Trigger output to this node.

3. **Add a Code node ("Code1") for normalization and aggregation**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the script that:  
     - Normalizes all keys to lowercase and trims whitespace.  
     - Counts leads per source.  
     - Sums spend amounts per channel (Instagram, LinkedIn, Google Ads).  
     - Calculates CPL per source (spend divided by leads), outputs "N/A" if data missing.  
   - Connect Google Sheets node output to this node.

4. **Add a second Code node ("Code") for summary aggregation**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the script that:  
     - Aggregates all leads into one summary JSON object: total leads, source counts, and full leads array.  
   - Connect Code1 output to this node.

5. **Add a LangChain LLM Chain node ("Basic LLM Chain")**  
   - Type: LangChain Chain LLM  
   - Parameters:  
     - Prompt type: Define prompt  
     - Text prompt: Instruct it to create a professional weekly lead attribution report in responsive HTML from JSON data, specifying layout, sections (Summary, Key Metrics, Top Sources, CPL, Insights, Priority Actions), mobile-friendly design, use of emojis, and output only HTML.  
     - Messages: Add system message defining the assistant as a professional business reporting assistant.  
   - Connect Code node output to this node.

6. **Add an Azure OpenAI Chat Model node ("Azure OpenAI Chat Model1")**  
   - Type: LangChain Azure OpenAI Model  
   - Parameters: Set model to "gpt-4o-mini" (or your preferred GPT-4o variant).  
   - Credentials: Link your Azure OpenAI API credentials.  
   - Connect this node as the AI model for the Basic LLM Chain node (set in LLM Chain’s AI model parameter).

7. **Add a Gmail node ("Send Follow-up Email1")**  
   - Type: Gmail  
   - Credentials: Set up OAuth2 Gmail credentials with sending permissions.  
   - Parameters:  
     - Send To: "newscctv22@gmail.com" (or your target recipient)  
     - Subject: "weekely reports" (note the exact spelling)  
     - Message: Expression referencing the output HTML from the Basic LLM Chain node, e.g., `={{ $json.text }}`  
   - Connect Basic LLM Chain output to this node.

8. **Connect all nodes in order:**  
   Schedule Trigger → Get row(s) in sheet → Code1 → Code → Basic LLM Chain → Send Follow-up Email1  
   Additionally, connect Azure OpenAI Chat Model node as the AI model for Basic LLM Chain.

9. **Test the workflow:**  
   - Run manually or wait for scheduled trigger.  
   - Check Google Sheets data correctness and API credentials.  
   - Verify email delivery and HTML report rendering.  

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow outputs a weekly report in responsive HTML with a professional, card-style design including emojis for clarity. | Enhances readability on mobile email clients and messaging apps like Telegram.                              |
| The Google Sheet document and sheet name must remain consistent or require updating in the Google Sheets node. | Source/UTM Attribution and Reporting( testing), sheet "Form responses 1".                                   |
| Azure OpenAI GPT-4o-mini is used to balance cost and output quality; model can be swapped if desired.       | Requires valid Azure OpenAI API credentials with GPT-4o access.                                              |
| Gmail OAuth2 credentials must be set up with appropriate sending permissions for the recipient email.       | Ensure Gmail account security and API quota compliance.                                                     |
| Code nodes use normalization to handle potential data inconsistencies in sheet columns (e.g., trailing spaces). | Important for robust aggregation and accurate CPL calculation.                                              |
| The email subject contains a spelling typo "weekely reports" which may be intentional or should be corrected. | Adjust as needed for professionalism.                                                                       |

---

Disclaimer: The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.

---