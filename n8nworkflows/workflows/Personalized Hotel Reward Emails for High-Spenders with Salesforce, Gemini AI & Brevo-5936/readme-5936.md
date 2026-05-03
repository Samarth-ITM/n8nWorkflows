Personalized Hotel Reward Emails for High-Spenders with Salesforce, Gemini AI & Brevo

https://n8nworkflows.xyz/workflows/personalized-hotel-reward-emails-for-high-spenders-with-salesforce--gemini-ai---brevo-5936


# Personalized Hotel Reward Emails for High-Spenders with Salesforce, Gemini AI & Brevo

### 1. Workflow Overview

This workflow automates the process of identifying high-spending hotel guests after their checkout and sending them personalized reward offers via email. It targets guests who have recently updated checkout records in Salesforce and analyzes their spending across various paid hotel amenities. If the total spend exceeds a defined threshold, the system uses AI to generate a customized, realistic, and enticing one-time free perk offer for an amenity the guest has not yet used or has used less frequently. The personalized offer is then sent through a branded HTML email via Brevo (formerly Sendinblue).

**Target Use Cases:**  
- Hospitality CRM automation for rewarding VIP guests  
- Personalized marketing communications based on customer spend behavior  
- Integration of Salesforce data with AI-driven content generation and email delivery

**Logical Blocks:**  
- **1.1 Input Reception & Data Retrieval:** Trigger on Salesforce guest record updates and extract detailed spend data.  
- **1.2 VIP Identification:** Calculate total spend and filter guests exceeding the high-spender threshold.  
- **1.3 AI Offer Generation:** Use Google Vertex AI and LangChain to create personalized reward offers based on spend patterns.  
- **1.4 Email Delivery:** Format and send the personalized offer via Brevo email node.  
- **1.5 Workflow Annotation:** A sticky note node documents workflow purpose and usage for maintainability.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Retrieval

**Overview:**  
This block listens for updates on the Salesforce `Guest__c` custom object, then fetches detailed spend data for the updated guests.

**Nodes Involved:**  
- Check for any latest checkouts (Salesforce Trigger)  
- Extract Checkout information (Salesforce)  
- Look For VIP Clients (Code)

**Node Details:**  

- **Check for any latest checkouts**  
  - Type: Salesforce Trigger  
  - Role: Watches the `Guest__c` custom object for updates once a week.  
  - Config: Polls weekly for changes on the custom object.  
  - Inputs: None (trigger node)  
  - Outputs: Passes updated guest records downstream.  
  - Edge Cases: Salesforce API limits, auth token expiry, no updated records found.  
  - Credentials: Uses OAuth2 Salesforce account.  

- **Extract Checkout information**  
  - Type: Salesforce Node (Get All operation)  
  - Role: Retrieves specified guest fields including spend amounts and contact info for all guests.  
  - Config: Fetches fields like Name, guest_id__c, phone__c, Email__c, and various spend fields.  
  - Inputs: Trigger from previous node  
  - Outputs: List of guest records with spend details.  
  - Edge Cases: API timeouts, large data volume handling, missing fields.  
  - Credentials: Same Salesforce OAuth2 account as above.  

- **Look For VIP Clients**  
  - Type: Code (JavaScript)  
  - Role: Calculates total spend across all optional amenities for each guest.  
  - Config: Adds values of Room Service, Minibar, Laundry, Late Checkout, Airport Transfer spends; sets default to 0 if undefined.  
  - Inputs: Guest records from Salesforce node  
  - Outputs: Guest records augmented with a `total` spend field.  
  - Edge Cases: Missing or null spend fields, non-numeric values causing calculation errors.

---

#### 1.2 VIP Identification

**Overview:**  
Filters guests to identify those exceeding a spend threshold ($50), marking them as VIPs eligible for the reward offer.

**Nodes Involved:**  
- Check the threshold exceedings (IF node)

**Node Details:**  

- **Check the threshold exceedings**  
  - Type: IF  
  - Role: Checks if the calculated total spend is greater than or equal to $50.  
  - Config: Condition: total >= 50  
  - Inputs: Guests with total spend from previous code node  
  - Outputs: Two branches — 'true' (VIP guests) and 'false' (non-VIP, ignored)  
  - Edge Cases: Missing or malformed total values, numeric type validation issues.

---

#### 1.3 AI Offer Generation

**Overview:**  
Generates a personalized and realistic one-time free perk offer for the guest using Gemini AI via Google Vertex AI and LangChain.

**Nodes Involved:**  
- Give Away Personalised Offers (LangChain AI Chat Google Vertex)  
- Structured Output Parser (LangChain Output Parser Structured)  
- Basic LLM Chain (LangChain Chain LLM)

**Node Details:**  

- **Give Away Personalised Offers**  
  - Type: LangChain LM Chat Google Vertex  
  - Role: Calls Gemini 2.5 AI model hosted on Google to generate an offer text based on guest spend data.  
  - Config: Uses project `slidingpuzzlepro1` and Gemini-2.5-flash model.  
  - Inputs: Guest spend data from IF node (VIP guests)  
  - Outputs: Raw AI chat response  
  - Credentials: Google Service Account  
  - Edge Cases: API quota limits, latency, unexpected AI output formats.  

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI JSON output to extract the `suggested_offer` field strictly.  
  - Config: JSON schema example expects `{ "suggested_offer": "..." }`  
  - Inputs: AI raw output from previous node  
  - Outputs: Clean JSON object with offer text  
  - Edge Cases: AI output not matching schema, parsing errors.  

- **Basic LLM Chain**  
  - Type: LangChain Chain LLM  
  - Role: Defines the AI prompt that instructs the model on how to select and generate the personalized offer.  
  - Config: AI prompt includes guest spend variables for six amenities, rules to pick an unused service or fallback logic, and output JSON format enforcement.  
  - Inputs: Structured output from parser (via ai_outputParser connection), also sends output to email node  
  - Outputs: Final structured offer JSON  
  - Edge Cases: Expression evaluation failures on spend fields, prompt misconfiguration.

---

#### 1.4 Email Delivery

**Overview:**  
Sends the finalized personalized reward offer to the guest’s email address using a branded HTML email template via Brevo.

**Nodes Involved:**  
- Send offer via email (SendInBlue Node)

**Node Details:**  

- **Send offer via email**  
  - Type: SendInBlue (Brevo)  
  - Role: Sends an HTML email with a personalized subject and offer content to the guest’s email address.  
  - Config:  
    - Sender: placeholder email (should be replaced with valid sender)  
    - Subject: Uses guest name dynamically (e.g., "John, We Have Something Special for Your Next Stay")  
    - HTML content: Styled email with brand colors, including the AI-generated offer text interpolated in the body.  
    - Recipient: guest’s Email__c field from Salesforce data  
  - Inputs: Offer JSON from Basic LLM Chain node  
  - Outputs: Email send status  
  - Credentials: Brevo API Key  
  - Edge Cases: Invalid email addresses, API send failures, HTML rendering issues.

---

#### 1.5 Workflow Annotation

**Overview:**  
A sticky note node documents the workflow purpose, key nodes, example outputs, and rationale behind the automation for maintainers and stakeholders.

**Nodes Involved:**  
- Sticky Note

**Node Details:**  

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Provides detailed documentation inside the workflow editor.  
  - Content: Describes the workflow steps, key nodes, example outputs, and motivation behind the automation to increase guest loyalty via personalized rewards.  
  - Inputs/Outputs: None  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                                    | Input Node(s)                 | Output Node(s)             | Sticky Note                                                                                                                 |
|-----------------------------|---------------------------------|---------------------------------------------------|------------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Check for any latest checkouts | Salesforce Trigger              | Trigger on guest record updates                    | None                         | Extract Checkout information | See general sticky note for workflow purpose and overview.                                                                  |
| Extract Checkout information | Salesforce                      | Retrieve guest spend and contact details           | Check for any latest checkouts | Look For VIP Clients         | See general sticky note for workflow purpose and overview.                                                                  |
| Look For VIP Clients         | Code (JavaScript)               | Calculate total spend                               | Extract Checkout information | Check the threshold exceedings | See general sticky note for workflow purpose and overview.                                                                  |
| Check the threshold exceedings | IF                            | Filter guests exceeding $50 total spend             | Look For VIP Clients          | Basic LLM Chain             | See general sticky note for workflow purpose and overview.                                                                  |
| Give Away Personalised Offers | LangChain LM Chat Google Vertex | Generate AI-based personalized reward offers       | Check the threshold exceedings | Basic LLM Chain             | See general sticky note for workflow purpose and overview.                                                                  |
| Structured Output Parser     | LangChain Output Parser Structured | Parse AI JSON output into structured data          | Give Away Personalised Offers | Basic LLM Chain (ai_outputParser) | See general sticky note for workflow purpose and overview.                                                                  |
| Basic LLM Chain             | LangChain Chain LLM             | Define AI prompt and finalize offer JSON           | Structured Output Parser (ai_outputParser) | Send offer via email        | See general sticky note for workflow purpose and overview.                                                                  |
| Send offer via email         | SendInBlue (Brevo)              | Send personalized reward email to guest            | Basic LLM Chain              | None                       | See general sticky note for workflow purpose and overview.                                                                  |
| Sticky Note                 | Sticky Note                    | Documentation of workflow purpose and key nodes    | None                         | None                       | ## 📩 **High-Spender Guest Reward Automation**... (full sticky note content as per section 1)                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Salesforce Trigger node**  
   - Type: Salesforce Trigger  
   - Configure to trigger on updates to the custom object `Guest__c`  
   - Poll interval: Every week  
   - Connect Salesforce OAuth2 credentials.  

2. **Add a Salesforce node to get guest details**  
   - Type: Salesforce (Get All operation)  
   - Resource: Custom Object `Guest__c`  
   - Fields to retrieve: `Name`, `guest_id__c`, `phone__c`, `Total_Room_Service_Spend__c`, `Total_Minibar_Spend__c`, `Total_Laundry_Spend__c`, `Total_Late_Checkout_Fees__c`, `Total_Extra_Bed_Fees__c`, `Total_Airport_Transfer_Spend__c`, `Email__c`  
   - Connect credentials same as trigger node.  
   - Connect trigger output to this node input.  

3. **Add a Code node to calculate total spend**  
   - Type: Code (JavaScript)  
   - Script: Sum all spend fields, defaulting missing values to 0, output JSON with total field added.  
   - Connect Salesforce Get node output to this node.  

4. **Add an IF node to filter VIP guests**  
   - Condition: `total >= 50`  
   - Input: Output from Code node  
   - Configure true branch for guests exceeding threshold.  

5. **Add a LangChain LM Chat Google Vertex node**  
   - Type: LangChain LLM Chat (Google Vertex AI)  
   - Model: Gemini-2.5-flash  
   - Project ID: your Google Cloud project (e.g., `slidingpuzzlepro1`)  
   - Connect true output of IF node to this node.  
   - Provide Google Service Account credentials.  

6. **Add a Structured Output Parser node**  
   - Type: LangChain Output Parser Structured  
   - JSON Schema: `{ "suggested_offer": "..." }`  
   - Connect output of LangChain LM Chat node to this parser’s input.  

7. **Add a LangChain Chain LLM node**  
   - Type: LangChain Chain LLM  
   - Prompt:  
     - Include guest spend fields as variables  
     - Instructions to pick one unused amenity or fallback to an offer on a less used one  
     - Output strictly JSON with `suggested_offer` field  
   - Connect Structured Output Parser node output to this node as ai_outputParser input.  
   - Connect this node output to the email node in the next step.  

8. **Add a SendInBlue (Brevo) node**  
   - Type: SendInBlue  
   - Set sender email (replace placeholder with valid sender)  
   - Subject: Use guest name dynamically, e.g., `={{ $('Check the threshold exceedings').item.json.Name }}, We Have Something Special for Your Next Stay`  
   - Email recipient: guest email from Salesforce field `Email__c` dynamically  
   - Email content: HTML template with embedded `{{ $json.output.suggested_offer }}` for the personalized offer  
   - Connect output of LangChain Chain LLM node to this node input.  
   - Provide Brevo API credentials.  

9. **Add a Sticky Note**  
   - Content: Document workflow purpose, key steps, example output, and rationale as per section 1.  
   - Position for visibility within the workflow canvas.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow rewards guests who show willingness to spend by offering personalized, one-time free perks. This approach increases loyalty and repeat bookings without blanket discounts, making the offer feel exclusive and relevant.                                                                                                                                      | Workflow purpose and marketing rationale.                                                                |
| AI prompt logic strictly enforces output as JSON with a specific structure to ensure integration compatibility.                                                                                                                                                                                                                                                         | See Basic LLM Chain node prompt configuration.                                                           |
| The branded HTML email template uses inline CSS and hotel brand colors for professional presentation. Replace the sender placeholder with a valid sender email before production use.                                                                                                                                                                                    | Send offer via email node HTML content.                                                                   |
| Google Vertex AI model "gemini-2.5-flash" is used for generating natural language personalized offers. Ensure billing and API quota is managed in Google Cloud Console.                                                                                                                                                                                                   | LangChain LM Chat Google Vertex node.                                                                    |
| Brevo (formerly Sendinblue) is used for email delivery. Monitor email sending limits and handle bounce or invalid email errors appropriately in production.                                                                                                                                                                                                             | Send offer via email node.                                                                                 |
| Salesforce OAuth2 tokens must be regularly refreshed to prevent workflow failure due to expired credentials.                                                                                                                                                                                                                                                             | Salesforce nodes.                                                                                          |
| The workflow processes guests weekly; adjust polling frequency in the Salesforce Trigger node if near real-time updates are preferred, considering API limits.                                                                                                                                                                                                         | Check for any latest checkouts node configuration.                                                        |
| Official n8n docs for LangChain nodes and SendInBlue node provide further detailed setup instructions and examples: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/, https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.sendInBlue/                                                                                                    | n8n node documentation.                                                                                    |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.