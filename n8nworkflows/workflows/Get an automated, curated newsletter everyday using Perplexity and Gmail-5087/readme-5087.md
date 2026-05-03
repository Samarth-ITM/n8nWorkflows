Get an automated, curated newsletter everyday using Perplexity and Gmail

https://n8nworkflows.xyz/workflows/get-an-automated--curated-newsletter-everyday-using-perplexity-and-gmail-5087


# Get an automated, curated newsletter everyday using Perplexity and Gmail

### 1. Workflow Overview

This workflow, titled **Perplexity Newsletter**, automates the daily creation and drafting of a curated newsletter email. It targets users who want to receive a concise, insightful digest of domain-specific news in three areas: MBA admissions, Product Management, and AI Automation. The workflow leverages Perplexity’s AI API to fetch recent, relevant news with expert commentary and then compiles these insights into a formatted HTML email draft saved to Gmail.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Date Setup**: Runs daily at a specified hour and calculates the current and past dates for filtering recent news.
- **1.2 Domain-Specific News Fetching (Perplexity Nodes)**: Uses Perplexity nodes to retrieve curated news and commentary for MBA, Product Management, and AI Automation.
- **1.3 Data Aggregation and Formatting**: Merges outputs from the three domains and formats them into a styled HTML newsletter.
- **1.4 Email Draft Creation**: Sends the compiled newsletter as a draft email in Gmail for review or sending.

Supporting this are documentation and setup instruction sticky notes to guide configuration and customization.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Date Setup

- **Overview:**  
  This block initiates the workflow daily at 8 AM and sets date variables used to filter news links published within the last week.

- **Nodes Involved:**  
  - Daily Trigger  
  - Set Dates

- **Node Details:**

  - **Daily Trigger**  
    - *Type & Role:* Schedule trigger node that starts the workflow automatically based on time.  
    - *Configuration:* Triggers every day at 8:00 AM.  
    - *Inputs:* None (start node).  
    - *Outputs:* Triggers "Set Dates" node.  
    - *Edge Cases:* Misconfigured timezone or cron expression might cause unexpected trigger times.

  - **Set Dates**  
    - *Type & Role:* Sets workflow variables for date filtering.  
    - *Configuration:*  
      - `currentDate`: Current date in 'yyyy-MM-dd' format.  
      - `lastweekDate`: Date 7 days prior to current date, same format.  
    - *Expressions:* Uses JavaScript date functions with `$now`.  
    - *Inputs:* From "Daily Trigger".  
    - *Outputs:* Feeds into all three Perplexity nodes.  
    - *Edge Cases:* Timezone mismatches or date formatting errors could affect news relevance.

---

#### 2.2 Domain-Specific News Fetching (Perplexity Nodes)

- **Overview:**  
  This block uses three Perplexity nodes to gather curated news and expert commentary from each domain, filtered by the dates set previously. Each node runs independently but simultaneously.

- **Nodes Involved:**  
  - MBA News  
  - Product Management  
  - AI Automation News

- **Node Details:**

  - **MBA News**  
    - *Type & Role:* Perplexity API node querying recent MBA-related news for international students.  
    - *Configuration:*  
      - Model: "sonar"  
      - System prompt defines expert content curation role with tone and output format instructions.  
      - User prompt dynamically filters news links published between `currentDate` and `lastweekDate`.  
    - *Expressions:* Uses date variables from "Set Dates" node to filter news recency.  
    - *Inputs:* From "Set Dates".  
    - *Outputs:* To "Merge" node (input 0).  
    - *Edge Cases:*  
      - API key or quota issues.  
      - No news found for date range.  
      - Response formatting errors.

  - **Product Management**  
    - *Type & Role:* Perplexity node focused on product management news and trends.  
    - *Configuration:* Similar to MBA News but with prompts tailored to product launches, jobs, and strategy insights.  
    - *Inputs:* From "Set Dates".  
    - *Outputs:* To "Merge" node (input 1).  
    - *Edge Cases:* Same as MBA News.

  - **AI Automation News**  
    - *Type & Role:* Perplexity node gathering AI automation workflows, tools, and productivity hacks.  
    - *Configuration:* Tailored prompt for practical AI use cases and commentary.  
    - *Inputs:* From "Set Dates".  
    - *Outputs:* To "Merge" node (input 2).  
    - *Edge Cases:* Same as above.

---

#### 2.3 Data Aggregation and Formatting

- **Overview:**  
  This block merges the three news streams and formats them into a single HTML newsletter with clear sections and clickable links.

- **Nodes Involved:**  
  - Merge  
  - Combine messages (Code node)

- **Node Details:**

  - **Merge**  
    - *Type & Role:* Merges inputs from the three Perplexity nodes into a single stream.  
    - *Configuration:* Number of inputs set to 3, each corresponds to one domain’s output.  
    - *Inputs:* MBA News (input 0), Product Management (input 1), AI Automation News (input 2).  
    - *Outputs:* To "Combine messages".  
    - *Edge Cases:* Missing input from any Perplexity node could cause incomplete data or errors.

  - **Combine messages**  
    - *Type & Role:* JavaScript Code node that formats merged messages into styled HTML sections.  
    - *Configuration:*  
      - Defines section titles matching the three domains.  
      - Replaces newlines with `<br>`, formats markdown-like syntax to HTML tags, and converts URLs to clickable links with custom styles.  
      - Joins all sections into one HTML string assigned to `message`.  
    - *Inputs:* From "Merge".  
    - *Outputs:* To "Gmail".  
    - *Edge Cases:* Unexpected message formats or missing fields could break HTML formatting.

---

#### 2.4 Email Draft Creation

- **Overview:**  
  Sends the formatted newsletter content as a draft email in Gmail for manual review and sending.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**

  - **Gmail**  
    - *Type & Role:* Gmail node to create/send email drafts.  
    - *Configuration:*  
      - Recipient email hardcoded as `v.abhijay@gmail.com` (update as needed).  
      - Subject set to "Aspyre Community Content".  
      - Email body is the HTML message from "Combine messages".  
      - Uses OAuth2 credentials for Gmail account access.  
    - *Inputs:* From "Combine messages".  
    - *Outputs:* None (end node).  
    - *Edge Cases:*  
      - OAuth token expiry or invalid credentials.  
      - Gmail API rate limits or errors.  
      - If the recipient email is incorrect, email will not be delivered.

---

#### 2.5 Documentation & Setup Instructions (Sticky Notes)

- **Workflow Documentation Sticky Note**  
  Provides a summary of workflow purpose, logic, required credentials, and customization tips.

- **Setup Instructions Sticky Note**  
  Details configuration steps for API keys, Gmail credentials, recipient email, schedule timing, and content customization.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                        | Input Node(s)                 | Output Node(s)         | Sticky Note                                                                                                               |
|---------------------|-------------------------|-------------------------------------|------------------------------|------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger       | Schedule Trigger         | Initiates workflow daily at 8 AM    | None                         | Set Dates              |                                                                                                                           |
| Set Dates           | Set                     | Sets current and last week dates    | Daily Trigger                | MBA News, Product Management, AI Automation News |                                                                                                                           |
| MBA News            | Perplexity              | Fetches curated MBA news links      | Set Dates                   | Merge (input 0)         |                                                                                                                           |
| Product Management  | Perplexity              | Fetches product management news     | Set Dates                   | Merge (input 1)         |                                                                                                                           |
| AI Automation News  | Perplexity              | Fetches AI automation news           | Set Dates                   | Merge (input 2)         |                                                                                                                           |
| Merge               | Merge                   | Combines news from 3 Perplexity nodes | MBA News, Product Management, AI Automation News | Combine messages       |                                                                                                                           |
| Combine messages    | Code                    | Formats merged news into HTML       | Merge                       | Gmail                  |                                                                                                                           |
| Gmail               | Gmail                   | Sends formatted newsletter as draft | Combine messages            | None                   | Update recipient email in this node; requires Gmail OAuth2 credentials setup                                              |
| Workflow Documentation | Sticky Note            | Documents workflow purpose and usage | None                       | None                   | "This workflow runs every day at 9 AM to fetch curated news in three domains and send formatted drafts via Gmail."         |
| Setup Instructions  | Sticky Note              | Lists required configurations       | None                       | None                   | "⚙️ Configuration Needed: Perplexity API key, SMTP credentials, recipient email, schedule timing, and content customization." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Trigger" Node**  
   - Type: Schedule Trigger  
   - Set to trigger every day at 8:00 AM (hour 8, minute 0).

2. **Add "Set Dates" Node**  
   - Type: Set  
   - Add two string fields:  
     - `currentDate` with value: `={{ $now.format('yyyy-MM-dd') }}`  
     - `lastweekDate` with value: `={{ $now.minus({days: 7}).format('yyyy-MM-dd') }}`  
   - Connect "Daily Trigger" output to "Set Dates" input.

3. **Create "MBA News" Perplexity Node**  
   - Type: Perplexity (or OpenAI with similar prompt if Perplexity not available)  
   - Use model "sonar".  
   - System prompt: Expert curator role for MBA news with instructions on tone and output format (copy content from workflow).  
   - User prompt: Request 1 relevant link published between `{{ $json.currentDate }}` and `{{ $json.lastweekDate }}` for Indian MBA applicants with a concise commentary.  
   - Credentials: Set Perplexity API credentials.  
   - Connect "Set Dates" output to "MBA News" input.

4. **Create "Product Management" Perplexity Node**  
   - Similar to "MBA News" but with prompts tailored for product management news, launches, jobs, and strategy.  
   - Connect "Set Dates" output to "Product Management" input.

5. **Create "AI Automation News" Perplexity Node**  
   - Similar setup, with prompt focused on AI workflow automation and productivity hacks.  
   - Connect "Set Dates" output to "AI Automation News" input.

6. **Create "Merge" Node**  
   - Type: Merge  
   - Set number of inputs to 3.  
   - Connect outputs of "MBA News" (input 0), "Product Management" (input 1), and "AI Automation News" (input 2) to respective inputs of "Merge".

7. **Create "Combine messages" Code Node**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the formatting code that:  
     - Defines section titles: "MBA News", "Product Management", "AI Automation News".  
     - Converts markdown-style text to HTML with headings, bold, italics, and clickable URLs.  
   - Connect "Merge" output to "Combine messages" input.

8. **Create "Gmail" Node**  
   - Type: Gmail  
   - Authentication: OAuth2 credentials (create and configure credentials in n8n for Gmail).  
   - Set "Send To" email to desired recipient (update from `v.abhijay@gmail.com`).  
   - Set Subject: "Aspyre Community Content".  
   - For message body, set to `={{ $json.message }}` (HTML output from previous node).  
   - Connect "Combine messages" output to "Gmail" input.

9. **Add Sticky Notes for Documentation and Setup** (Optional)  
   - Create two sticky notes containing:  
     - Workflow overview and usage instructions.  
     - Configuration steps: API keys, Gmail credentials, recipient email, scheduling, and prompt customization.

10. **Test Workflow**  
    - Run manually or wait for scheduled trigger.  
    - Verify the Gmail draft is created with correct content and formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow runs every day at 8 AM by default; modify the "Daily Trigger" node to change schedule time.                                        | Scheduling configuration                                                                         |
| Requires valid Perplexity API credentials; alternatively, OpenAI or similar AI nodes can be adapted with equivalent prompts.              | API credential setup                                                                             |
| Gmail OAuth2 credentials must be configured in n8n; ensure permissions allow draft creation.                                                | Gmail API integration                                                                            |
| Customize news categories by editing system and user prompts in the Perplexity nodes.                                                      | Content customization                                                                           |
| Email subject and recipient must be updated in the Gmail node to suit user preferences.                                                    | Email delivery configuration                                                                    |
| HTML formatting in "Combine messages" node uses custom styles for headings and links; can be adapted for branding or styling needs.       | Email format customization                                                                       |
| Sticky notes provide inline documentation and setup instructions for easy maintenance.                                                     | Workflow documentation                                                                           |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow and complies with all applicable content policies. No illegal, offensive, or protected content is included. All data handled is legal and public.