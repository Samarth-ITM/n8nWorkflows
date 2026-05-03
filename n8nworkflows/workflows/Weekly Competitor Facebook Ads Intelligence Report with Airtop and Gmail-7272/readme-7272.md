Weekly Competitor Facebook Ads Intelligence Report with Airtop and Gmail

https://n8nworkflows.xyz/workflows/weekly-competitor-facebook-ads-intelligence-report-with-airtop-and-gmail-7272


# Weekly Competitor Facebook Ads Intelligence Report with Airtop and Gmail

### 1. Workflow Overview

This workflow automates the weekly monitoring of a competitor’s active Facebook ads by extracting key advertising data and sending a consolidated intelligence report via email. It targets marketing analysts or competitive intelligence teams who want to save time on manual ad research and gain consistent insights into competitors’ messaging, offers, and creative trends.

The workflow is logically divided into three primary functional blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow once per week at a specific time.
- **1.2 Facebook Ad Library Extraction:** Uses the Airtop node to scrape and analyze up to 30 active ads from a competitor’s Facebook Ad Library page with a detailed prompt to extract relevant ad intelligence.
- **1.3 Email Report Delivery:** Composes and sends the extracted ad intelligence report as an HTML email via Gmail.

Additionally, a comprehensive sticky note provides documentation and setup instructions embedded directly within the workflow for user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block initiates the workflow execution on a weekly schedule, ensuring the intelligence report is generated and sent regularly without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `n8n-nodes-base.scheduleTrigger`  
    - Role: Triggers the workflow on a defined recurring schedule.  
    - Configuration:  
      - Interval set to trigger every week at 19:00 hours (7 PM).  
      - Version 1.2 node type supports interval scheduling with time-of-day precision.  
    - Input: No inputs (start node).  
    - Output: Passes trigger event to the next node ("Query Facebook Ad Library").  
    - Potential Failures: The only failure scenario could be if the n8n instance is offline or scheduler is disabled. No authentication required.  
    - No sub-workflow references.

#### 2.2 Facebook Ad Library Extraction

- **Overview:**  
  This block uses the Airtop node to query Facebook’s Ad Library for active ads from a specified competitor page. It runs a natural language extraction prompt to summarize each ad’s key details and formats the results as an HTML report.

- **Nodes Involved:**  
  - Query Facebook Ad Library

- **Node Details:**  
  - **Query Facebook Ad Library**  
    - Type: `n8n-nodes-base.airtop`  
    - Role: Executes a web automation and AI extraction task using Airtop service.  
    - Configuration:  
      - URL targeted: Facebook Ad Library page filtered for active ads in the US from a specific page ID (`15087023444`).  
      - Prompt instructs to extract up to 30 ads with details including message summary, main topic, CTA, ad active duration, language, and assumed target audience.  
      - Result format: HTML formatted report for readability.  
      - Session Mode: New session to ensure fresh scraping.  
      - Credentials: Uses Airtop API Key credential configured in n8n.  
    - Inputs: Receives trigger signal from Schedule Trigger node.  
    - Outputs: Passes extraction results (including HTML report in `data.modelResponse`) to the next node (“Send Report”).  
    - Potential Failures / Edge Cases:  
      - Airtop API quota exceeded or invalid API key leading to auth errors.  
      - Facebook Ad Library page structure changes causing extraction issues.  
      - Network timeouts.  
      - Prompt understanding failures or incomplete data extraction.  
    - Version: 1  
    - No sub-workflow references.

#### 2.3 Email Report Delivery

- **Overview:**  
  This block sends the compiled HTML intelligence report to a specified email recipient using Gmail with OAuth2 authentication.

- **Nodes Involved:**  
  - Send Report

- **Node Details:**  
  - **Send Report**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Sends email via Gmail service.  
    - Configuration:  
      - Recipient: `amir@airtop.ai` (can be customized).  
      - Subject: “Facebook Ad Library Competitor Report: ” (static prefix, no dynamic suffix).  
      - Message Body: Uses an expression to insert the HTML report from the previous node (`={{ $json.data.modelResponse }}`).  
      - Options: Default, no attachments or CC/BCC.  
      - Credentials: Uses Gmail OAuth2 credential configured in n8n for secure authentication.  
    - Inputs: Receives extracted report data from “Query Facebook Ad Library”.  
    - Outputs: None (end node).  
    - Potential Failures:  
      - Gmail authentication errors (expired or revoked OAuth tokens).  
      - Email sending limits or quota exceeded.  
      - Invalid recipient address causing delivery failure.  
    - Version: 2.1  
    - No sub-workflow references.

#### 2.4 Documentation & Setup Guidance

- **Overview:**  
  Provides detailed instructions, use case explanation, setup requirements, and suggestions for extending the workflow embedded as a sticky note.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Role: Non-executable node for user information and documentation.  
    - Configuration:  
      - Large note containing README-style content explaining use case, what the workflow does, how it works, setup instructions, and suggestions for future improvements.  
      - Includes important links such as [Airtop API key generation](https://portal.airtop.ai/api-keys).  
    - Position: Placed off to the side for reference.  
    - No inputs or outputs.  
    - No failure modes.  

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                     | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                                            |
|---------------------------|----------------------------|-----------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | n8n-nodes-base.scheduleTrigger | Initiates workflow weekly          | -                      | Query Facebook Ad Library | README content covers all nodes: Monitor competitor ads weekly, Airtop extraction, Gmail delivery. Setup instructions included.                       |
| Query Facebook Ad Library | n8n-nodes-base.airtop       | Extracts competitor’s Facebook ads | Schedule Trigger       | Send Report              | README content as above.                                                                                                                                 |
| Send Report               | n8n-nodes-base.gmail        | Sends HTML report via Gmail        | Query Facebook Ad Library | -                        | README content as above.                                                                                                                                 |
| Sticky Note               | n8n-nodes-base.stickyNote   | Documentation and setup instructions | -                      | -                        | Contains full README with use case, instructions, and next steps. Includes link: https://portal.airtop.ai/api-keys                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it**:  
   `Competitor Facebook Ads Intelligence Agent`.

2. **Add a Schedule Trigger node**:  
   - Node Type: `Schedule Trigger`  
   - Set parameters:  
     - Trigger interval: Every 1 week  
     - Trigger time: 19:00 (7 PM)  
   - No credentials required.

3. **Add an Airtop node** to query Facebook Ad Library:  
   - Node Type: `Airtop`  
   - Set parameters:  
     - URL: `https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&is_targeted_country=false&media_type=all&search_type=page&view_all_page_id=15087023444`  
       *(Replace `view_all_page_id` with your competitor’s Facebook page ID as needed.)*  
     - Resource: `extraction`  
     - Operation: `query`  
     - Session Mode: `new`  
     - Prompt:  
       ```
       This is a Facebook ad library. Extract up to 30 ads for each ad, extract:
       Message summary (1–2 concise sentences)

       Main topic (e.g., Discount, Awareness, Product Launch, Testimonial)

       Call-to-Action (CTA)** (e.g., Learn More, Book Now, Shop Now)

       How long the ad has been active (days active or start date, if displayed)

       Ad Language (Eng, Esp, et..)

       Reverse engineered target audience (hypothesis)

       Return the results in as a nice, HTML formatted report
       ```  
   - Credentials: Select or create Airtop API credentials with a valid API key from https://portal.airtop.ai/api-keys.

4. **Connect the Schedule Trigger node’s output to the Airtop node input.**

5. **Add a Gmail node** to send the report:  
   - Node Type: `Gmail`  
   - Parameters:  
     - Send To: `amir@airtop.ai` (customize recipient email as needed)  
     - Subject: `Facebook Ad Library Competitor Report: `  
     - Message: Use expression editor and set to `{{$json["data"]["modelResponse"]}}` to insert the extracted HTML report.  
     - Options: Leave defaults unless needed otherwise.  
   - Credentials: Configure Gmail OAuth2 credential with a Google account authorized to send emails.

6. **Connect the Airtop node output to the Gmail node input.**

7. **(Optional) Add a Sticky Note node** for documentation:  
   - Content: Paste the README text from the original workflow describing the use case, setup, and next steps.  
   - Position the note clearly for user reference.

8. **Activate the workflow** and confirm the nodes are connected as:  
   Schedule Trigger → Query Facebook Ad Library → Send Report.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                            | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow automates competitor Facebook ad monitoring and delivers weekly intelligence via email.                                                                                                                                                                        | General workflow purpose                                                                            |
| Airtop API key is required; generate at https://portal.airtop.ai/api-keys and configure in n8n credentials under “Airtop”.                                                                                                                                              | Airtop API key setup                                                                                |
| Gmail OAuth2 credential must be set up for sending emails. Use Google Cloud Console to create OAuth2 client credentials and authorize n8n.                                                                                                                              | Gmail OAuth2 credential setup                                                                       |
| Replace the Facebook Ad Library URL’s `view_all_page_id` parameter with your competitor’s Facebook page ID to customize the workflow.                                                                                                                                   | Customizing the target competitor                                                                 |
| Suggested enhancements: duplicate Airtop extraction for multiple competitors, add landing page analysis, send reports to Slack or archive in shared workspaces.                                                                                                       | Workflow extension ideas                                                                            |
| Sticky note embedded in the workflow contains detailed README including instructions, use case, and setup notes.                                                                                                                                                         | Embedded documentation                                                                             |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.