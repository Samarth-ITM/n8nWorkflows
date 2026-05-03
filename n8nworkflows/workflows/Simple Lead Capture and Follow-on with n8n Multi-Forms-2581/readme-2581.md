Simple Lead Capture and Follow-on with n8n Multi-Forms

https://n8nworkflows.xyz/workflows/simple-lead-capture-and-follow-on-with-n8n-multi-forms-2581


# Simple Lead Capture and Follow-on with n8n Multi-Forms

### 1. Workflow Overview

This workflow implements a **Simple Lead Capture and Follow-on with n8n Multi-Forms** designed for startups, side projects, or small businesses aiming for an easy and engaging way to collect newsletter signups and additional user information. It replaces costly or off-putting third-party forms with a fully integrated multi-step form experience entirely within n8n, leveraging Google Sheets for data storage and Slack for notifications.

The workflow is logically divided into three main blocks:

- **1.1 Initial Lead Capture**  
  Captures the user’s email through a form trigger and appends it to a Google Sheet, while sending a Slack notification of the new signup.

- **1.2 Multi-Step Survey Follow-on**  
  Uses chained n8n Form nodes to present a multi-page survey collecting detailed user information (personal, professional, and interest-related). This information updates the same Google Sheet row created in the initial capture step.

- **1.3 Completion Screen Display**  
  Shows a customized completion screen thanking the user and confirming submission completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Initial Lead Capture

**Overview:**  
This block triggers when a user submits the initial signup form with their email. It records the email and submission date to Google Sheets and sends a notification to a Slack channel.

**Nodes Involved:**  
- Sign Up Form (formTrigger)  
- Capture Email (Google Sheets)  
- Notify New Signup! (Slack)  

**Node Details:**

- **Sign Up Form**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing user's email for newsletter signup.  
  - *Configuration:*  
    - Path: `newsletter-signup` (exposes webhook URL)  
    - Form title and description encouraging newsletter signup.  
    - Single required Email field.  
    - Response mode set to “lastNode” (responds after last node).  
  - *Connections:* Output to Capture Email node.  
  - *Failure Modes:* Webhook access issues, invalid email input, or network failures.  
  - *Version Specifics:* Uses v2.1 of the Form Trigger node.  

- **Capture Email**  
  - *Type:* Google Sheets (Append operation)  
  - *Role:* Appends new row with email, submission timestamp (`submittedAt`), and execution id to a Google Sheet.  
  - *Configuration:*  
    - Document and sheet IDs configured for a public example spreadsheet.  
    - Columns: `date`, `email`, `execution_id`.  
  - *Inputs:* From Sign Up Form.  
  - *Outputs:* To Notify New Signup! node.  
  - *Failure Modes:* Google Sheets API quota limits, auth failures, malformed data in inputs.  
  - *Credentials:* Google Sheets OAuth2 required.  
  - *Version Specifics:* v4.5.  

- **Notify New Signup!**  
  - *Type:* Slack node  
  - *Role:* Sends a notification message to a Slack channel alerting of a new newsletter signup.  
  - *Configuration:*  
    - Message posted in `#general` channel (by name).  
    - Uses Slack’s block kit to format message with the user’s email extracted dynamically.  
  - *Inputs:* From Capture Email node.  
  - *Outputs:* To About You form node to continue flow.  
  - *Failure Modes:* Slack API auth failure, rate limits, channel not found errors.  
  - *Credentials:* Slack OAuth required.  
  - *Version Specifics:* v2.2.  

---

#### 2.2 Multi-Step Survey Follow-on

**Overview:**  
Presents a series of three chained multi-page forms to collect additional user details (personal info, interests, and beta tester interest). Each form submission passes data to the next, culminating in an update to the original Google Sheets row.

**Nodes Involved:**  
- About You (Form)  
- Your Interests (Form)  
- Join Beta Testers (Form)  
- Capture More Info (Google Sheets Update)  

**Node Details:**

- **About You**  
  - *Type:* Form node (multi-page form)  
  - *Role:* Collects personal details: first name, last name, country/region, job level, and job function (multi-select).  
  - *Configuration:*  
    - Form title: “Thanks For Signing Up!”  
    - Button label: “Continue (1 of 3)”  
    - Required fields: first name, last name, job level, job function.  
    - Optional: country/region.  
  - *Inputs:* From Notify New Signup!.  
  - *Outputs:* To Your Interests node.  
  - *Failure Modes:* Validation errors, user abandoning form, webhook timeout.  
  - *Version Specifics:* v1.  

- **Your Interests**  
  - *Type:* Form node  
  - *Role:* Surveys user familiarity with no-code automation and their goals for the product.  
  - *Configuration:*  
    - Form title: “What Brings You Here?”  
    - Button label: “Continue (2 of 3)”  
    - Required dropdown for experience level and textarea for product goals.  
    - Personalizes description with `<name>` placeholder (though no expression present, could be enhanced).  
  - *Inputs:* From About You.  
  - *Outputs:* To Join Beta Testers node.  
  - *Failure Modes:* Same as About You.  
  - *Version Specifics:* v1.  

- **Join Beta Testers**  
  - *Type:* Form node  
  - *Role:* Asks if the user wants to join the beta testers list.  
  - *Configuration:*  
    - Form title: “Join Our Beta Testers List”  
    - Button label: “Finish (3 of 3)”  
    - Required dropdown: Yes, No, Maybe.  
  - *Inputs:* From Your Interests.  
  - *Outputs:* To Capture More Info node.  
  - *Failure Modes:* Same as above.  
  - *Version Specifics:* v1.  

- **Capture More Info**  
  - *Type:* Google Sheets (Update operation)  
  - *Role:* Updates the row in the Google Sheet corresponding to the current execution id with all collected form data from the three survey steps.  
  - *Configuration:*  
    - Uses `execution_id` as matching column to find the row to update.  
    - Updates columns: job level, last name, first name, job function (joined into string), product goals, country/region, beta tester enrollment choice, and product experience.  
    - Sheet and document linked identical to Capture Email node.  
  - *Inputs:* From Join Beta Testers node.  
  - *Outputs:* To Show Completion Screen node.  
  - *Failure Modes:* Row not found due to execution id mismatch, Google Sheets API errors, data formatting issues.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Version Specifics:* v4.5.  

---

#### 2.3 Completion Screen Display

**Overview:**  
Displays a final confirmation page to the user after successful survey submission, thanking them and providing contact information.

**Nodes Involved:**  
- Show Completion Screen (Form node, completion operation)  

**Node Details:**

- **Show Completion Screen**  
  - *Type:* Form node (completion page)  
  - *Role:* Presents a customized thank-you message to the user to confirm the end of the form process.  
  - *Configuration:*  
    - Operation: completion  
    - Title: “NewsLetter Signup Short Survey Complete”  
    - Completion Title: “Thank you!”  
    - Completion Message includes gratitude, contact email placeholder `<EMAIL>`, and homepage placeholder `<HOMEPAGE>`.  
  - *Inputs:* From Capture More Info node.  
  - *Outputs:* None (end of workflow).  
  - *Failure Modes:* Webhook timeout, user abandonment before completion.  
  - *Version Specifics:* v1.  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                                      | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                          |
|---------------------|---------------------|-----------------------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Sign Up Form        | Form Trigger        | Entry form capturing user email for newsletter signup | —                      | Capture Email           | ## 1. Easy Lead Capture with n8n Forms [Learn more about Form Triggers](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger) |
| Capture Email       | Google Sheets       | Append user email, date, and execution ID to sheet   | Sign Up Form            | Notify New Signup!       | ## 1. Easy Lead Capture with n8n Forms                                                               |
| Notify New Signup!  | Slack               | Sends Slack notification of new signup               | Capture Email           | About You               | ## 1. Easy Lead Capture with n8n Forms                                                               |
| About You           | Form                | Collects personal user details (name, job, location) | Notify New Signup!       | Your Interests          | ## 2. Follow-on Short Survey via Multi-Step Forms [Read more about n8n Form node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form/) |
| Your Interests      | Form                | Gathers user experience and product goals             | About You               | Join Beta Testers       | ## 2. Follow-on Short Survey via Multi-Step Forms                                                   |
| Join Beta Testers   | Form                | Asks about interest in joining beta testers list      | Your Interests          | Capture More Info       | ## 2. Follow-on Short Survey via Multi-Step Forms                                                   |
| Capture More Info   | Google Sheets       | Updates Google Sheets row with all user details       | Join Beta Testers       | Show Completion Screen  | ## 2. Follow-on Short Survey via Multi-Step Forms                                                   |
| Show Completion Screen | Form               | Displays final thank-you completion screen            | Capture More Info       | —                       | ## 3. Customise Your Completion Screen [Read more about n8n Form node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form/) |
| Sticky Note1        | Sticky Note         | Explains initial lead capture block                    | —                      | —                       | ## 1. Easy Lead Capture with n8n Forms [Learn more about Form Triggers](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger) |
| Sticky Note         | Sticky Note         | Explains multi-step survey block                       | —                      | —                       | ## 2. Follow-on Short Survey via Multi-Step Forms [Read more about n8n Form node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form/) |
| Sticky Note2        | Sticky Note         | Explains completion screen customization               | —                      | —                       | ## 3. Customise Your Completion Screen [Read more about n8n Form node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form/) |
| Sticky Note4        | Sticky Note         | General overview and usage tips                        | —                      | —                       | ## Try It Out! This template builds a simple newsletter signup form with a follow-on short survey entirely in n8n! Using multi-page forms to maximize user engagement. Check out example sheet: https://docs.google.com/spreadsheets/d/15W1PiFjCoiEBHHKKCRVMLmpKg4AWIy9w1dQ2Dq8qxPs/edit?usp=sharing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node - "Sign Up Form"**  
   - Type: Form Trigger (version 2.1)  
   - Path: `newsletter-signup`  
   - Form Title: “Sign Up for My Newsletter”  
   - Form Description: “Use this form to signup for my newsletter where members will receive the latest workflow templates from me before everyone else! You can unsubscribe at any time.”  
   - Button Label: “Sign Up to Newsletter”  
   - Fields: One field of type Email, labeled “Email”, required.  
   - Response Mode: `lastNode`  
   - No credentials needed here.  

2. **Create Google Sheets Node - "Capture Email"**  
   - Type: Google Sheets (Append Operation, version 4.5)  
   - Credentials: Connect your Google Sheets OAuth2 account.  
   - Document ID: Use your target Google Sheet or the example one (ID: `15W1PiFjCoiEBHHKKCRVMLmpKg4AWIy9w1dQ2Dq8qxPs`)  
   - Sheet Name: Use the sheet with `gid=0` (usually "Sheet1")  
   - Columns to Append:  
     - `date` = Expression: `{{$json.submittedAt}}`  
     - `email` = Expression: `{{$json.Email}}`  
     - `execution_id` = Expression: `{{$execution.id}}`  
   - Connect Sign Up Form output to this node.  

3. **Create Slack Node - "Notify New Signup!"**  
   - Type: Slack (version 2.2)  
   - Credentials: Connect your Slack OAuth token.  
   - Channel: `#general` (or your preferred channel by name)  
   - Message: Use block kit with a section block containing text:  
     `"{{ $('Sign Up Form').first().json.Email.extractEmail() }} *just signed up to the newsletter!*"`  
   - Connect Capture Email output to this node.  

4. **Create Form Node - "About You"**  
   - Type: Form (version 1)  
   - Form Title: “Thanks For Signing Up!”  
   - Button Label: “Continue (1 of 3)”  
   - Form Description: Optional survey introduction text.  
   - Fields:  
     - First Name (text, required)  
     - Last Name (text, required)  
     - Country/Region (text, optional)  
     - Job Level (dropdown, required, options: CEO, VP, Director, Manager, Non-manager, Student or Intern, Other)  
     - Job Function (dropdown, multi-select, required, options: Accounting/Finance, Admin/Office, Customer Service, Design, Engineering/Software, HR/Operations, Leadership/Management, Legal, Other)  
   - Connect Notify New Signup! output to this node.  

5. **Create Form Node - "Your Interests"**  
   - Type: Form (version 1)  
   - Form Title: “What Brings You Here?”  
   - Button Label: “Continue (2 of 3)”  
   - Form Description: “Thanks <name>! Please tell us why you are interested in our product? It'll help us tailor your onboarding and communication journeys to better suit your needs.”  
   - Fields:  
     - How familiar are you with no-code automation? (dropdown, required) options:  
       - I've Just started or exploring no-code automation tools  
       - I've tried tools like Zapier to automate small tasks  
       - I've built several no-code automations and workflows already  
     - Describe briefly what you'd like to get out of our product? (textarea, required) placeholder: “Eg. short term pain points and long term solutions...”  
   - Connect About You output to this node.  

6. **Create Form Node - "Join Beta Testers"**  
   - Type: Form (version 1)  
   - Form Title: “Join Our Beta Testers List”  
   - Button Label: “Finish (3 of 3)”  
   - Form Description: Explains beta tester benefits and voluntary nature.  
   - Field: Would you like to be considered for our beta testers list? (dropdown, required) options: Yes, No, Maybe  
   - Connect Your Interests output to this node.  

7. **Create Google Sheets Node - "Capture More Info"**  
   - Type: Google Sheets (Update Operation, version 4.5)  
   - Credentials: Use the same Google Sheets OAuth2 credentials as before.  
   - Document ID and Sheet Name: Same as "Capture Email" node.  
   - Matching Column: `execution_id` (to update correct row)  
   - Columns to update:  
     - `first_name` = From About You form (First Name)  
     - `last_name` = From About You form (Last Name)  
     - `country_region` = From About You form (Country/Region)  
     - `job_level` = From About You form (Job Level)  
     - `job_function` = From About You form (Job Function) joined as comma-separated string  
     - `product_experience` = From Your Interests form ("How familiar are you with no-code automation?")  
     - `product_goals` = From Your Interests form ("Describe briefly what you'd like to get out of our product?")  
     - `enrol_betatesters` = From Join Beta Testers form (Beta tester interest)  
     - `execution_id` = Use current execution id  
   - Connect Join Beta Testers output to this node.  

8. **Create Form Node - "Show Completion Screen"**  
   - Type: Form (version 1)  
   - Operation: Completion screen  
   - Form Title: “NewsLetter Signup Short Survey Complete”  
   - Completion Title: “Thank you!”  
   - Completion Message:  
     ```
     Many thanks for taking the time to complete this short survey. A community representative will contact you shortly!

     We hope you enjoy the newsletter and please feel free to contact us at <EMAIL> should you have any questions.

     Go back to <HOMEPAGE>.
     ```  
   - Connect Capture More Info output to this node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow leverages n8n’s new multi-page form capabilities, enabling richer user interactions beyond a single form page.                                                                                                  | [n8n multi-page forms documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form/)          |
| Example Google Sheet used for demonstration: a public sheet named "Newsletter Signup" with columns matching the form data.                                                                                                   | https://docs.google.com/spreadsheets/d/15W1PiFjCoiEBHHKKCRVMLmpKg4AWIy9w1dQ2Dq8qxPs/edit?usp=sharing                   |
| Slack notifications use block kit formatting to make messages clearer and more engaging.                                                                                                                                     | https://api.slack.com/block-kit                                                                                         |
| Keep forms simple to maximize completion rates; consider breaking longer forms into multiple steps to maintain user engagement.                                                                                              | Best practice suggestion from workflow description                                                                      |
| For further assistance or community support, join the n8n Discord or Forum.                                                                                                                                                   | Discord: https://discord.com/invite/XPKeKXeB7d , Forum: https://community.n8n.io/                                        |
| Completion screen can be customized to redirect users to your homepage or blog instead of showing a static message, enhancing user experience.                                                                              | Suggested customization idea                                                                                            |

---

This document fully captures the workflow’s logic, node configurations, and implementation details to enable advanced users or automation agents to understand, reproduce, or extend the workflow confidently.