Enrich new Discourse members with Clearbit then notify in Slack

https://n8nworkflows.xyz/workflows/enrich-new-discourse-members-with-clearbit-then-notify-in-slack-2109


# Enrich new Discourse members with Clearbit then notify in Slack

### 1. Workflow Overview

This n8n workflow automates the enrichment and notification process for new members joining a Discourse community. It is designed primarily for Sales and Customer Success teams who want to identify and be alerted about potentially valuable new users, prospects, or existing customers registering on Discourse.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures new user registration events from Discourse via a webhook trigger.
- **1.2 Email Filtering:** Filters out common personal email domains to save enrichment credits.
- **1.3 User Enrichment with Clearbit:** Uses Clearbit API to enrich user data (person and company).
- **1.4 High-Value Lead Filtering:** Applies criteria to identify high-value leads based on company metrics.
- **1.5 Notification Delivery:** Sends formatted Slack messages with quick action buttons for high-value leads.
- **1.6 No Enrichment Handling:** Handles cases where Clearbit does not return enrichment data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new Discourse user events using a webhook. It acts as the entry point for the workflow triggered by Discourse's native webhook when a new member registers.

- **Nodes Involved:**  
  - On new Discourse user

- **Node Details:**

  - **On new Discourse user**  
    - Type: Webhook Trigger  
    - Role: Receives HTTP POST requests from Discourse when a new user is created.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook path (e.g., `abde7a49-208b-4bce-bcb9-910c4e529b06`)  
      - Webhook ID set for Discourse integration.  
    - Inputs: External HTTP POST from Discourse webhook  
    - Outputs: User JSON payload containing user email and other details.  
    - Edge cases: Webhook misconfiguration, network failures, Discourse API changes.  
    - Notes: Requires creating and configuring a webhook in Discourse admin panel at `/admin/api/web_hooks/new/edit`. See [Sticky Note] for setup instructions.

#### 2.2 Email Filtering

- **Overview:**  
  Filters out new users with common personal email domains (gmail, yahoo, hotmail, proton) to save Clearbit enrichment credits, focusing only on potentially valuable business-related emails.

- **Nodes Involved:**  
  - Filter out common personal emails

- **Node Details:**

  - **Filter out common personal emails**  
    - Type: Filter  
    - Role: Conditional filtering based on email domain substring absence.  
    - Configuration:  
      - Conditions combined with OR operator: The user email must **not** contain `@gmail.`, `@yahoo.`, `@hotmail.`, or `@proton.`  
      - Case-sensitive, strict string validation.  
    - Input: JSON user object from webhook node.  
    - Output: Passes only emails outside these common domains.  
    - Edge cases: New personal email domains not covered; false positives/negatives if domain format varies.  
    - Notes: Saves enrichment credits by avoiding unnecessary API calls.

#### 2.3 User Enrichment with Clearbit

- **Overview:**  
  Enriches the filtered user’s email via Clearbit’s "person" API. If found, fetches additional company data based on the person's employment domain. If not found, passes to a no-operation node for alternative handling.

- **Nodes Involved:**  
  - Enrich user with Clearbit  
  - Get company info  
  - No clearbit enrichment available (NoOp)

- **Node Details:**

  - **Enrich user with Clearbit**  
    - Type: Clearbit Node  
    - Role: Enriches user personal data by email.  
    - Configuration:  
      - Email expression: `={{ $json.body.user.email }}`  
      - Resource: Person  
      - On error: Continue workflow even on 404 (email not found).  
    - Input: User email from filter node.  
    - Outputs:  
      - Main output: Enriched person data object if found.  
      - Error output: Also continued, but empty if no data.  
    - Edge cases: API rate limits, 404 if email not found, network errors.  
    - Credential: Clearbit API key configured.  
    - Notes: Returns empty output on 404, handled downstream.

  - **Get company info**  
    - Type: Clearbit Node  
    - Role: Fetches company details based on employment domain from enriched user data.  
    - Configuration:  
      - Domain expression: `={{ $json.employment.domain }}` (from enriched person data)  
      - Resource: Company (default)  
    - Input: Output from "Enrich user with Clearbit".  
    - Output: Company enrichment data attached to the JSON object.  
    - Edge cases: Missing employment domain, company not found, API errors.  
    - Credential: Same Clearbit API key.

  - **No clearbit enrichment available**  
    - Type: No Operation (NoOp)  
    - Role: Placeholder node for users without Clearbit enrichment.  
    - Configuration: None.  
    - Input: Error or empty output path from "Enrich user with Clearbit".  
    - Output: None (ends workflow here).  
    - Notes: Workflow optionally ends here if no data found. See related sticky note suggesting alternatives.

#### 2.4 High-Value Lead Filtering

- **Overview:**  
  Applies business logic to identify "high-value" leads based on company size and global ranking metrics, ensuring only qualified leads trigger Slack notifications.

- **Nodes Involved:**  
  - Filter for high value leads

- **Node Details:**

  - **Filter for high value leads**  
    - Type: Filter  
    - Role: Filters companies with at least 30 employees and Alexa global rank under 100,000.  
    - Configuration:  
      - Conditions combined with AND operator:  
        - `$json.metrics.employees >= 30`  
        - `$json.metrics.alexaGlobalRank <= 100000`  
    - Input: Company enrichment data from "Get company info" node.  
    - Output: Only passes leads meeting criteria.  
    - Edge cases: Missing metrics, inaccurate or outdated data, zero or null values causing filter failure.  
    - Notes: Filter criteria are customizable; see sticky note for adjustment suggestions.

#### 2.5 Notification Delivery

- **Overview:**  
  Sends a formatted Slack message to a designated channel to notify the team about the new high-value lead, including user profile info and quick action buttons.

- **Nodes Involved:**  
  - Post message in Channel

- **Node Details:**

  - **Post message in Channel**  
    - Type: Slack Node  
    - Role: Posts rich message blocks to a Slack channel.  
    - Configuration:  
      - Channel: Configured to `#team-design` by default (changeable).  
      - Message type: Block Kit with sections, context, divider, and action buttons.  
      - Text and blocks use expressions to pull data from "Enrich user with Clearbit" and "On new Discourse user" nodes, including:  
        - User avatar image URL  
        - Full name, job title, company name, industry  
        - Buttons: "Open LinkedIn Profile" (link to LinkedIn handle), "Email member" (mailto link)  
    - Input: Filtered high-value lead data.  
    - Output: Slack API response.  
    - Credentials: Slack OAuth2 configured.  
    - Edge cases: Slack API rate limits, invalid channel, missing user data causing broken links or UI.

#### 2.6 No Enrichment Handling

- **Overview:**  
  Provides a logical endpoint for users whose emails are not found in Clearbit, preventing workflow failure and suggesting potential manual follow-up or other enrichment services.

- **Nodes Involved:**  
  - No clearbit enrichment available  
  - Sticky Note1

- **Node Details:**

  - **No clearbit enrichment available**  
    - See section 2.3 User Enrichment.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content:  
      "**👈 Optional**\nIf the workflow ends here, the email wasn't found in Clearbit. Consider checking with another enrichment service or sending a Slack message for manual verification."  
    - Position: Near "No clearbit enrichment available" node.  
    - Notes: Provides user guidance on handling no data cases.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                           | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                                    |
|-------------------------------|---------------------|-----------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| On new Discourse user          | Webhook             | Trigger on new Discourse user creation  | —                           | Filter out common personal emails | **1. ☝️ Set up `On new Discourse user` Trigger and Webhook in Discourse** See video: https://www.loom.com/share/d379895004374ddc85dc9171ca37c139?t=32&sid=da64c668-f7f5-4d49-982e-d1e72fb77fcc |
| Filter out common personal emails | Filter              | Filter out common personal email domains | On new Discourse user        | Enrich user with Clearbit    | Saves on Enrichment credits                                                                                                   |
| Enrich user with Clearbit      | Clearbit            | Enrich user personal data via email     | Filter out common personal emails | Get company info / No clearbit enrichment available | Clearbit returns 404 (empty output) if email not found                                                                        |
| Get company info               | Clearbit            | Fetch company data based on domain       | Enrich user with Clearbit    | Filter for high value leads  |                                                                                                                               |
| Filter for high value leads    | Filter              | Identify high-value leads based on metrics | Get company info             | Post message in Channel      | **Optional 👇** Change filter criteria to define "high value"                                                                  |
| Post message in Channel        | Slack               | Send Slack notification for high-value lead | Filter for high value leads  | —                           | **2. 👇 Set up `Post message in Channel` node** Change channel to desired Slack channel                                        |
| No clearbit enrichment available | NoOp                | End workflow for users without Clearbit data | Enrich user with Clearbit (error output) | —                           | **👈 Optional** Consider other enrichment or manual verify (Sticky Note1)                                                      |
| Sticky Note                   | Sticky Note         | Workflow description and video link     | —                           | —                           | ### Enrich new Discourse members then notify in Slack for high value leads [🎥 Watch set up video](https://www.loom.com/share/d379895004374ddc85dc9171ca37c139?sid=0996f0d2-aff2-45a7-aae9-c62df4fb0799) ![Slack example](https://i.ibb.co/s9MfsjV/Screenshot-2024-02-21-at-13-51-29.png#full-width) |
| Sticky Note1                  | Sticky Note         | Guidance on webhook setup in Discourse  | —                           | —                           | See detailed webhook creation instructions in Discourse admin panel                                                           |
| Sticky Note2                  | Sticky Note         | Guidance on high-value leads filter criteria | —                           | —                           | Optional: Change filter criteria to define "high value"                                                                       |
| Sticky Note3                  | Sticky Note         | Workflow title, description, and video  | —                           | —                           | Introductory note with branding and video link                                                                                |
| Sticky Note4                  | Sticky Note         | Guidance on Slack message node setup     | —                           | —                           | Change Slack channel parameter to desired channel                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node:**  
   - Type: Webhook  
   - Name: `On new Discourse user`  
   - HTTP Method: POST  
   - Path: Generate unique path (e.g., `abde7a49-208b-4bce-bcb9-910c4e529b06`)  
   - Save credentials if required (none needed here).  
   - Configure Discourse webhook at `https://{your-discourse-domain}/admin/api/web_hooks/new/edit` to send new user events to this webhook URL.

2. **Add Filter Node:**  
   - Type: Filter  
   - Name: `Filter out common personal emails`  
   - Conditions (OR combinator):  
     - Email (from webhook payload path `body.user.email`) **does not contain** `@gmail.`  
     - Email **does not contain** `@yahoo.`  
     - Email **does not contain** `@hotmail.`  
     - Email **does not contain** `@proton.`  
   - Connect webhook node output to this filter’s input.

3. **Add Clearbit Node for Person Enrichment:**  
   - Type: Clearbit  
   - Name: `Enrich user with Clearbit`  
   - Resource: Person  
   - Email: Expression `={{ $json.body.user.email }}`  
   - Credentials: Configure Clearbit API key  
   - On Error: Continue workflow  
   - Connect filter’s "true" output to this node.

4. **Add Clearbit Node for Company Enrichment:**  
   - Type: Clearbit  
   - Name: `Get company info`  
   - Domain: Expression `={{ $json.employment.domain }}` (from previous Clearbit person data)  
   - Credentials: Same Clearbit API key  
   - Connect main output of “Enrich user with Clearbit” to this node.

5. **Add Filter Node for High-Value Leads:**  
   - Type: Filter  
   - Name: `Filter for high value leads`  
   - Conditions (AND combinator):  
     - Number of employees (`$json.metrics.employees`) >= 30  
     - Alexa Global Rank (`$json.metrics.alexaGlobalRank`) <= 100000  
   - Connect output of “Get company info” node to this filter.

6. **Add Slack Node to Send Notification:**  
   - Type: Slack  
   - Name: `Post message in Channel`  
   - Channel: Set to your team Slack channel (e.g., `#sales` or `#team-design`)  
   - Message Type: Block Kit (use Slack blocks UI)  
   - Text: Provide summary text (e.g., “A high value lead just signed up on our Discourse community 👇”)  
   - Blocks:  
     - Section with user information (avatar, full name, title, company, industry) using expressions referencing “Enrich user with Clearbit” node JSON fields  
     - Action buttons for LinkedIn profile and email with URLs constructed from Clearbit and webhook data respectively  
   - Credentials: Configure Slack OAuth2 credentials  
   - Connect true output of “Filter for high value leads” node to this Slack node.

7. **Add No Operation Node for Missing Enrichment:**  
   - Type: NoOp  
   - Name: `No clearbit enrichment available`  
   - Connect error output or empty output of “Enrich user with Clearbit” node to this node to handle users not found in Clearbit.

8. **Add Sticky Notes (Optional):**  
   - Add sticky notes near key nodes to document setup instructions, optional configurations, and user guidance. Examples: webhook setup, filter criteria customization, Slack channel configuration.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                        | Context or Link                                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Workflow template is designed for Sales and Customer Success teams to detect valuable new Discourse members via Clearbit enrichment and Slack notifications.       | Workflow purpose                                                                                               |
| Watch a quick set up video (~2 min) showing detailed configuration steps and customization:                                                                        | https://www.loom.com/share/d379895004374ddc85dc9171ca37c139?sid=bb28df29-bc91-4d32-a657-0bfbaaf50cc7            |
| Discourse webhook creation instructions can be found in the video starting at timestamp 32s:                                                                      | https://www.loom.com/share/d379895004374ddc85dc9171ca37c139?t=32&sid=da64c668-f7f5-4d49-982e-d1e72fb77fcc      |
| Slack message format uses Slack Block Kit with dynamic user and company data, including avatar, job title, company name, industry, and actionable buttons.          | Slack Block Kit documentation: https://api.slack.com/block-kit                                                    |
| Clearbit API returns 404 and empty output if email not found; workflow gracefully continues (no failure). Alternate enrichment or manual verification is suggested. | Clearbit API behavior                                                                                           |

---

This document fully describes the workflow structure, stepwise logic, configuration, and integration points to enable reproduction, maintenance, and extension by advanced users and AI agents alike.