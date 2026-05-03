Revive Expired Real Estate Listings with OpenAI and Gmail Follow-ups

https://n8nworkflows.xyz/workflows/revive-expired-real-estate-listings-with-openai-and-gmail-follow-ups-6280


# Revive Expired Real Estate Listings with OpenAI and Gmail Follow-ups

### 1. Workflow Overview

This workflow automates the reactivation of expired real estate listings by identifying listings inactive for over 30 days, generating personalized follow-up emails using OpenAI, and sending these emails via Gmail. It is designed for real estate professionals to recover potentially lost sales opportunities by engaging property owners proactively.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Data Retrieval:** Periodic trigger to start the workflow and retrieval of current property listings from a Google Sheet.
- **1.2 Expiration Filtering:** Filtering listings that have been inactive for more than 30 days.
- **1.3 AI-Powered Email Generation:** Using OpenAI to create personalized follow-up emails to owners of expired listings.
- **1.4 Email Dispatch:** Sending the AI-generated emails to property owners via Gmail.

Additional context and instructions are provided in sticky notes for setup and usage guidelines.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Data Retrieval

- **Overview:**  
This block initiates the workflow on a monthly basis and loads all property listings from a Google Sheet to process.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Listings

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow every 30 days automatically.  
    - *Configuration:* Interval set to trigger once every 30 days (`daysInterval: 30`).  
    - *Input:* None (trigger node).  
    - *Output:* Triggers "Get Listings" node.  
    - *Edge Cases:* Workflow will not run if n8n instance is down at scheduled time; no retries configured.  

  - **Get Listings**  
    - *Type:* Google Sheets  
    - *Role:* Reads all property listings from a specified Google Sheet document and sheet.  
    - *Configuration:* Requires Google Sheets OAuth2 credentials; document ID and sheet name must be set.  
    - *Expressions:* None.  
    - *Input:* Trigger from Schedule Trigger.  
    - *Output:* Passes all rows (listings) to "Check >30 Days" node.  
    - *Edge Cases:* Authentication failure, missing or renamed sheet/document, empty sheet, rate limiting on Google Sheets API.  
    - *Version:* n8n Google Sheets node version 4.  

#### 2.2 Expiration Filtering

- **Overview:**  
Filters listings to include only those with a `last_activity` date older than 30 days (i.e., expired listings).

- **Nodes Involved:**  
  - Check >30 Days

- **Node Details:**

  - **Check >30 Days**  
    - *Type:* If (conditional filter)  
    - *Role:* Evaluates each listing’s `last_activity` date against the current timestamp minus 30 days (2592000000 milliseconds).  
    - *Configuration:* Condition uses expression:  
      `Date.now() - new Date($json["last_activity"]).getTime() > 2592000000`  
      which checks if the listing was last active more than 30 days ago.  
    - *Input:* Listings from "Get Listings".  
    - *Output:* Passes listings that meet the condition to "OpenAI" node; discards others.  
    - *Edge Cases:* Invalid or missing `last_activity` values may cause evaluation errors or false negatives; date parsing errors; timezone considerations.  
    - *Version:* n8n If node version 1.  

#### 2.3 AI-Powered Email Generation

- **Overview:**  
Generates a friendly yet professional personalized follow-up email for each expired listing using OpenAI language model.

- **Nodes Involved:**  
  - OpenAI

- **Node Details:**

  - **OpenAI**  
    - *Type:* OpenAI (Langchain)  
    - *Role:* Creates a customized reactivation email based on property details and owner information.  
    - *Configuration:*  
      - Model ID linked via credential (e.g., GPT-4).  
      - Message prompt includes detailed instructions and listing data interpolated via expressions, such as:  
        ```
        You are an assistant helping a real estate agent re-engage expired property listings.

        Generate a personalized follow-up email to a property owner, based on the following data:

        Property Title: {{$json["title"]}}
        Owner Name: {{$json["owner_name"]}}
        Property Type: {{$json["property_type"]}}
        Location: {{$json["location"]}}
        Last Activity Date: {{$json["last_activity"]}}

        Tone: Friendly but professional.
        Goal: Convince the owner to relist their property, highlight market opportunities or changes since last contact.

        Do not use placeholders or template-style responses. Write it like a
        ```
      - No placeholders or templates allowed, the AI must generate final text.  
    - *Input:* Filtered expired listings from "Check >30 Days".  
    - *Output:* Generated email body passed to "Send Email".  
    - *Edge Cases:* API request failures, rate limiting, malformed inputs, credential expiry.  
    - *Version:* Node version 1.8.  

#### 2.4 Email Dispatch

- **Overview:**  
Sends the AI-generated personalized email to each property owner using Gmail.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**

  - **Send Email**  
    - *Type:* Gmail  
    - *Role:* Dispatches the follow-up email to the property owner’s email address.  
    - *Configuration:*  
      - OAuth2 credentials for Gmail required.  
      - Subject fixed as: "Your Property Listing Is About to Expire".  
      - Message body interpolated directly from incoming data (the AI-generated email text).  
      - Uses expressions to include property owner name, property title, location, last activity date, and property type in the message.  
    - *Input:* Email content from "OpenAI" node.  
    - *Output:* None (end of flow).  
    - *Edge Cases:* Authentication errors, email sending limits, invalid recipient emails, connectivity issues.  
    - *Version:* Gmail node version 1.  

---

### 3. Summary Table

| Node Name       | Node Type                         | Functional Role                     | Input Node(s)      | Output Node(s)    | Sticky Note                                                                                      |
|-----------------|----------------------------------|-----------------------------------|--------------------|-------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger| Schedule Trigger                 | Initiates workflow every 30 days  | None               | Get Listings       |                                                                                                |
| Get Listings    | Google Sheets                   | Reads listings from Google Sheet  | Schedule Trigger   | Check >30 Days     |                                                                                                |
| Check >30 Days  | If                             | Filters listings inactive >30 days| Get Listings       | OpenAI             |                                                                                                |
| OpenAI          | OpenAI (Langchain)              | Generates personalized emails     | Check >30 Days     | Send Email         |                                                                                                |
| Send Email      | Gmail                          | Sends email to property owner     | OpenAI             | None               |                                                                                                |
| Sticky Note     | Sticky Note                    | Documentation / explanation       | None               | None               | # Expired Listing Revival Agent\n\n## Problem\n... (full content in section 5)                  |
| Sticky Note1    | Sticky Note                    | Label for block flow overview     | None               | None               | ## Flow                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to run every 30 days (set daysInterval to 30).  
   - Position it as the first node.

2. **Add the Google Sheets node ("Get Listings"):**  
   - Type: Google Sheets  
   - Connect input from Schedule Trigger.  
   - Configure credentials using Google Sheets OAuth2 (set up or select existing).  
   - Set the Document ID to the target Google Sheet containing listings.  
   - Set the Sheet Name to the sheet with listing data (e.g., "Listings").  
   - This node reads all rows in the sheet.

3. **Add the If node ("Check >30 Days"):**  
   - Type: If  
   - Connect input from Google Sheets node.  
   - Set condition:  
     - Mode: Number  
     - Value1: Use expression `={{ Date.now() - new Date($json["last_activity"]).getTime() }}`  
     - Operation: Greater than (`>`)  
     - Value2: `2592000000` (milliseconds in 30 days)  
   - This filters listings inactive for more than 30 days.

4. **Add the OpenAI node:**  
   - Type: OpenAI (Langchain)  
   - Connect input from the "true" output of the If node.  
   - Configure OpenAI API credentials with your API key.  
   - Select model appropriate for text generation (e.g., GPT-4).  
   - In the messages parameter, add a system or user message including the following prompt, using expressions to inject listing data:  
     ```
     You are an assistant helping a real estate agent re-engage expired property listings.

     Generate a personalized follow-up email to a property owner, based on the following data:

     Property Title: {{$json["title"]}}
     Owner Name: {{$json["owner_name"]}}
     Property Type: {{$json["property_type"]}}
     Location: {{$json["location"]}}
     Last Activity Date: {{$json["last_activity"]}}

     Tone: Friendly but professional.
     Goal: Convince the owner to relist their property, highlight market opportunities or changes since last contact.

     Do not use placeholders or template-style responses. Write it like a
     ```

5. **Add the Gmail node ("Send Email"):**  
   - Type: Gmail  
   - Connect input from OpenAI node.  
   - Configure Gmail OAuth2 credentials.  
   - Set the email subject to: "Your Property Listing Is About to Expire".  
   - For the message body, use the text generated by OpenAI (from `$json.text` or the output field containing the generated email).  
   - Set recipient email to the listing owner’s email (make sure this field is mapped correctly in the data).  
   - Use expressions to personalize the message if needed (e.g., owner name, property title).

6. **(Optional) Add a node to update the listing status in Google Sheets or CRM after email sent to track follow-ups.**  
   - This workflow does not include this step but can be extended accordingly.

7. **Add Sticky Notes for documentation:**  
   - Create sticky notes with the overview, problem statement, and setup instructions for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                       | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| **Expired Listing Revival Agent**: Automates detection and re-engagement of inactive real estate listings older than 30 days, generating personalized AI emails to owners and sending follow-ups via Gmail. Designed to help real estate agents recover lost sales opportunities.                    | Full sticky note content within the workflow.                                                  |
| Setup instructions include creating a Google Sheet with columns: `title`, `owner_name`, `email`, `property_type`, `location`, `last_activity`.                                                                                                                                                   | Section 2.1 and sticky note details.                                                           |
| The OpenAI prompt carefully instructs to avoid placeholders and produce final, user-ready emails in a friendly but professional tone.                                                                                                                                                            | Section 2.3 prompt details.                                                                     |
| Gmail OAuth2 credentials must be configured properly to allow sending emails via the Gmail node.                                                                                                                                                                                                  | Section 2.4 and node configuration notes.                                                     |
| The workflow assumes date strings in `last_activity` are valid and parseable ISO date strings; invalid formats may cause filtering errors.                                                                                                                                                       | Edge case notes in 2.2.                                                                         |
| The workflow currently does not handle replies or update listing statuses post-email, but this feature can be added for improved automation and tracking.                                                                                                                                         | Mentioned in sticky note and optional step in section 4.                                       |
| This workflow is suitable for real estate agents, brokers, CRM developers, marketing teams, and listing management agencies who want to automate renewal outreach efficiently.                                                                                                                     | Sticky note "For Who" section.                                                                 |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.