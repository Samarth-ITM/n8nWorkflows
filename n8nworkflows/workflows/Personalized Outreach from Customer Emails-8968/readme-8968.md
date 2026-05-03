Personalized Outreach from Customer Emails

https://n8nworkflows.xyz/workflows/personalized-outreach-from-customer-emails-8968


# Personalized Outreach from Customer Emails

### 1. Workflow Overview

This workflow automates the creation of personalized sales outreach emails based on previous email correspondence with customers or prospects. It is designed for sales development representatives (SDRs) or marketing teams aiming to efficiently generate tailored email drafts that reflect the recipient's communication style, goals, and challenges.

The workflow is logically divided into the following blocks:

- **1.1 Target Customer Retrieval:** Connects to HubSpot to fetch a filtered list of decision-makers for outreach.
- **1.2 Email Correspondence Collection:** For each contact, gathers recent email threads from Gmail.
- **1.3 Persona Construction:** Uses AI to analyze collected emails and build a detailed customer persona.
- **1.4 Sales Email Generation:** Employs AI to draft a personalized sales email tailored to the persona and product offering.
- **1.5 Draft Email Creation:** Saves the AI-generated email as a Gmail draft for review and sending.

---

### 2. Block-by-Block Analysis

#### 1.1 Target Customer Retrieval

**Overview:**  
Fetches a focused list of decision-makers from HubSpot based on a predefined filter to target the most relevant prospects for outreach.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Get Contacts (HubSpot)  
- For Each Contact (SplitInBatches)  
- Contact Ref (NoOp)  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual trigger node to start the workflow manually.  
  - Configuration: No parameters; triggers the workflow on user action.  
  - Input/Output: No input; outputs trigger signal to "Get Contacts".  
  - Edge Cases: None significant; manual start only.

- **Get Contacts**  
  - Type: HubSpot node to search contacts.  
  - Configuration: Searches contacts with a filter on `hs_buying_role` set to "DECISION_MAKER". Uses OAuth2 authentication for HubSpot.  
  - Input: Trigger from manual trigger.  
  - Output: List of contacts matching the filter.  
  - Edge Cases: HubSpot API rate limits, OAuth token expiration, no contacts found.

- **For Each Contact**  
  - Type: SplitInBatches node to process each contact individually.  
  - Configuration: Default batch size; processes contacts one by one to avoid API overload.  
  - Input: List of contacts from HubSpot node.  
  - Output: Single contact passed to next node in each batch.  
  - Edge Cases: Empty contact list leads to no iterations.

- **Contact Ref**  
  - Type: NoOp (No operation) node.  
  - Configuration: Used as a reference point for workflow branching.  
  - Input: Single contact from batch.  
  - Output: Passes contact data onward.  
  - Edge Cases: None.

---

#### 1.2 Email Correspondence Collection

**Overview:**  
Retrieves recent email conversations from Gmail for each contact to gather context about their communication style and interests.

**Nodes Involved:**  
- Variables (Set)  
- Get All Customer's Correspondence (Gmail)  

**Node Details:**

- **Variables**  
  - Type: Set node to define variables needed downstream.  
  - Configuration: Extracts and sets `firstname`, `lastname`, `email`, and a fixed `product_to_sell` string describing the consulting package. It pulls these from the contact’s JSON properties.  
  - Input: Contact data from "Contact Ref".  
  - Output: Variables prepared for persona building and email generation.  
  - Edge Cases: Missing or malformed contact properties could cause empty variables.

- **Get All Customer's Correspondence**  
  - Type: Gmail node to fetch emails.  
  - Configuration: Retrieves up to 20 emails from Gmail filtered by `from:` set to the contact's email address. Operation set to "getAll" with full email data (not simple).  
  - Input: Variables with contact email.  
  - Output: List of recent email threads from the contact.  
  - Edge Cases: Gmail API rate limits, authentication issues, no emails found.

---

#### 1.3 Persona Construction

**Overview:**  
Analyzes the collected emails using an AI language model to extract a detailed profile of the customer’s decision-making style, communication preferences, pain points, and other behavioral attributes.

**Nodes Involved:**  
- Analyse and Build Persona (Information Extractor)  
- Google Gemini Chat Model (Language Model)  

**Node Details:**

- **Analyse and Build Persona**  
  - Type: Information Extractor (AI node) using Google Gemini.  
  - Configuration: Processes concatenated email subjects, dates, and messages to extract multiple persona attributes such as decision-making style, communication preferences, pain points, goals, work style, personality traits, buying behavior, business culture, and industry awareness. A detailed system prompt guides the AI on extraction criteria.  
  - Input: Emails from Gmail node, variables for email address.  
  - Output: JSON object with persona attributes.  
  - Edge Cases: AI misinterpretation, incomplete data leading to partial persona, API errors.

- **Google Gemini Chat Model**  
  - Type: Language model node configured with "models/gemini-2.0-flash".  
  - Configuration: Supports the information extractor node by handling AI calls.  
  - Input: Connected as AI language model for the information extractor.  
  - Output: Persona extraction results.  
  - Edge Cases: API key issues, timeouts, model unavailability.

---

#### 1.4 Sales Email Generation

**Overview:**  
Uses AI to draft a personalized sales email based on the constructed persona and the product details, tailoring tone, style, and content to the customer's profile.

**Nodes Involved:**  
- Generate Sales Email (Information Extractor)  
- Google Gemini Chat Model1 (Language Model)  

**Node Details:**

- **Generate Sales Email**  
  - Type: Information Extractor node.  
  - Configuration: Receives the persona profile and variables including the product description. The system prompt instructs the AI to write a sales email subject and body that matches the customer's communication style and addresses their values, goals, and pain points. The output includes only subject and HTML-styled body, omitting signature.  
  - Input: Persona JSON and variables.  
  - Output: Email subject and body.  
  - Edge Cases: AI output may lack coherence or relevance; HTML formatting errors; API failures.

- **Google Gemini Chat Model1**  
  - Type: Language model node same as above.  
  - Configuration: Supports the email generation AI node.  
  - Edge Cases: Same as previous Gemini model node.

---

#### 1.5 Draft Email Creation

**Overview:**  
Saves the AI-drafted sales email as a Gmail draft addressed to the contact for review and manual sending by sales reps.

**Nodes Involved:**  
- Create Draft Email For Review (Gmail)  

**Node Details:**

- **Create Draft Email For Review**  
  - Type: Gmail node.  
  - Configuration: Creates an email draft with subject and HTML body as generated by AI, addressed to the contact's email. Uses Gmail API with OAuth2 credentials.  
  - Input: AI-generated email content and contact email from variables.  
  - Output: Draft created in Gmail.  
  - Edge Cases: Gmail API rate limits, authentication errors, email formatting issues.

- Connection: Upon draft creation, the workflow loops back to process the next contact in the batch.

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                     | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                         |
|-------------------------------|-----------------------------------------|-----------------------------------|------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                          | Workflow start trigger             | None                         | Get Contacts                   |                                                                                                   |
| Get Contacts                  | HubSpot                                | Retrieve list of decision-makers   | When clicking ‘Test workflow’ | For Each Contact               | ## 1. Pick your target customers in Hubspot Start with a small, focused list so you can measure results. Pull a filtered list from HubSpot for precise targeting. |
| For Each Contact             | SplitInBatches                         | Process each contact individually  | Get Contacts                 | Contact Ref                   |                                                                                                   |
| Contact Ref                  | NoOp                                   | Reference node for contact data    | For Each Contact             | Variables                     |                                                                                                   |
| Variables                   | Set                                    | Define key variables for processing | Contact Ref                 | Get All Customer's Correspondence | ## 2. Get previous email threads                                                                 |
| Get All Customer's Correspondence | Gmail                              | Fetch recent emails from contact   | Variables                   | Analyse and Build Persona      |                                                                                                   |
| Analyse and Build Persona    | Information Extractor (AI)               | Build customer persona from emails | Get All Customer's Correspondence | Generate Sales Email          | ## 3. Build profile from the emails For each prospect, use AI to create a simple persona based on emails Use the **Information Extractor** node to tell the LLM which traits to pull (eg: role, goals, pain points, decision style, tone). |
| Google Gemini Chat Model     | Language Model (Google Gemini)           | AI model backend for persona build | Analyse and Build Persona    | Analyse and Build Persona AI input |                                                                                                   |
| Generate Sales Email         | Information Extractor (AI)               | Draft personalized sales email     | Analyse and Build Persona    | Create Draft Email For Review  | ## 4. Write the sales email using the persona Feed the persona to AI and have it draft a tailored pitch that matches the customer’s beliefs, priorities, and communication style: * Keep it short: clear problem, value, next step. * Mirror their tone (formal/casual). * Reference their goals or pain points from the emails. * Add a strong, specific call to action. |
| Google Gemini Chat Model1    | Language Model (Google Gemini)           | AI model backend for email generation | Generate Sales Email         | Generate Sales Email AI input  |                                                                                                   |
| Create Draft Email For Review | Gmail                                 | Save AI-generated email as draft   | Generate Sales Email         | For Each Contact              | ## 5. Create a draft in gmail Save the AI-written pitch as a draft. An SDR can skim, tweak, and send—turning a list of contacts into personalized outreach in minutes, not hours. |
| Sticky Note5                | Sticky Note                            | Instructional comment              | None                         | None                          | ## 1. Pick your target customers in Hubspot Start with a small, focused list so you can measure results. Pull a filtered list from HubSpot for precise targeting. |
| Sticky Note6                | Sticky Note                            | Instructional comment              | None                         | None                          | ## 2. Get previous email threads                                                                 |
| Sticky Note7                | Sticky Note                            | Instructional comment              | None                         | None                          | ## 3. Build profile from the emails For each prospect, use AI to create a simple persona based on emails Use the **Information Extractor** node to tell the LLM which traits to pull (eg: role, goals, pain points, decision style, tone). |
| Sticky Note8                | Sticky Note                            | Instructional comment              | None                         | None                          | ## 4. Write the sales email using the persona Feed the persona to AI and have it draft a tailored pitch that matches the customer’s beliefs, priorities, and communication style: * Keep it short: clear problem, value, next step. * Mirror their tone (formal/casual). * Reference their goals or pain points from the emails. * Add a strong, specific call to action. |
| Sticky Note9                | Sticky Note                            | Instructional comment              | None                         | None                          | ## 5. Create a draft in gmail Save the AI-written pitch as a draft. An SDR can skim, tweak, and send—turning a list of contacts into personalized outreach in minutes, not hours. |
| Sticky Note10               | Sticky Note                            | Workflow description and usage     | None                         | None                          | ## What this workflows does Create email draft for a list of prospect or customers, based on previous email conversations, for the topic you want. How it works 1. Pull customers from HubSpot 2. For each customer, pull all emails from Gmail. 3. AI builds profile 4. AI creates a custom email using the profile 5. Save a draft in Gmail for you to review How to use * Set the outreach topic in the **Variables** node * Connect to the Gmail account * Add a Gemini API key |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Add HubSpot Node to Get Contacts:**  
   - Type: HubSpot  
   - Operation: Search contacts  
   - Filter: `hs_buying_role` equals `DECISION_MAKER`  
   - Authentication: OAuth2 with HubSpot credentials  
   - Connect output of Manual Trigger to this node.

3. **Add SplitInBatches Node (For Each Contact):**  
   - Type: SplitInBatches  
   - Default batch size (1 usually)  
   - Connect output of HubSpot node to this node.

4. **Add NoOp Node (Contact Ref):**  
   - Type: No Operation (NoOp)  
   - Connect output of SplitInBatches node (main batch output) here.

5. **Add Set Node (Variables):**  
   - Type: Set  
   - Configure variables:  
     - `firstname` = `={{ $json.properties.firstname }}`  
     - `lastname` = `={{ $json.properties.lastname }}`  
     - `email` = `={{ $json.properties.email }}`  
     - `product_to_sell` = set to fixed string describing your product or service  
   - Connect output of Contact Ref to this node.

6. **Add Gmail Node (Get All Customer's Correspondence):**  
   - Type: Gmail  
   - Operation: Get All Emails  
   - Filter query: `from:{{ $json.email }}`  
   - Limit: 20 emails  
   - Authentication: OAuth2 with Gmail credentials  
   - Connect output of Variables node to this node.

7. **Add Information Extractor Node (Analyse and Build Persona):**  
   - Type: Information Extractor with AI language model support  
   - System Prompt: Instruct AI to analyze emails and extract detailed persona attributes (decision-making style, preferences, pain points, etc.)  
   - Input Text: Concatenate email subjects, dates, and bodies as described  
   - Connect output of Gmail node to this node.

8. **Add Google Gemini Chat Model Node:**  
   - Type: Language Model (Google Gemini)  
   - Model: `models/gemini-2.0-flash`  
   - Credentials: Google PaLM API key  
   - Connect this node as AI model for Information Extractor node (persona build).

9. **Add Information Extractor Node (Generate Sales Email):**  
   - Type: Information Extractor  
   - System Prompt: Instruct AI to draft a sales email subject and HTML body using the persona and product description variables.  
   - Input Text: Use persona JSON and variables containing product info.  
   - Connect output of persona build node to this node.

10. **Add Second Google Gemini Chat Model Node:**  
    - Same as step 8, to support email generation AI node.

11. **Add Gmail Node (Create Draft Email For Review):**  
    - Type: Gmail  
    - Operation: Create draft email  
    - Message: Use AI-generated body with HTML styling  
    - Subject: Use AI-generated subject  
    - Send to: Contact email from variables  
    - Authentication: OAuth2 with Gmail credentials  
    - Connect output of sales email generation node to this node.

12. **Connect output of Gmail draft node back to SplitInBatches node:**  
    - To continue processing next contact.

13. **Set up Credentials:**  
    - HubSpot OAuth2 credentials for HubSpot node.  
    - Gmail OAuth2 credentials for Gmail nodes.  
    - Google PaLM API credentials for Gemini model nodes.

14. **Add Sticky Notes (Optional):**  
    - Add descriptive sticky notes at each logical block for clarity and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                           | Context or Link                                                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow created to automate personalized outreach emails using AI-generated customer personas and sales pitches based on actual email conversations.                                                                                                                  | Workflow description.                                                                                                                                               |
| Use a small, focused list of decision-makers for initial testing to measure outreach effectiveness.                                                                                                                                                                    | Sticky Note5 content.                                                                                                                                               |
| AI persona builds are guided by detailed attribute extraction, including decision style, communication, pain points, and buying behavior.                                                                                                                             | Sticky Note7 content.                                                                                                                                               |
| Sales emails are tailored to the customer’s style and priorities, emphasizing clarity, tone matching, and a strong call to action.                                                                                                                                     | Sticky Note8 content.                                                                                                                                               |
| Draft emails are saved in Gmail for manual review, allowing SDRs to quickly personalize and send.                                                                                                                                                                       | Sticky Note9 content.                                                                                                                                               |
| For setup, ensure you have: HubSpot OAuth2 credentials, Gmail OAuth2 credentials, and Google PaLM API credentials for Gemini model access.                                                                                                                                 | Sticky Note10 content.                                                                                                                                              |
| Google Gemini (PaLM) Model used is “models/gemini-2.0-flash”, which requires an active Google API key with PaLM access.                                                                                                                                                  | Node configuration details.                                                                                                                                         |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.