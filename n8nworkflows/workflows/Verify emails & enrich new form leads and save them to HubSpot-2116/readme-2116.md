Verify emails & enrich new form leads and save them to HubSpot

https://n8nworkflows.xyz/workflows/verify-emails---enrich-new-form-leads-and-save-them-to-hubspot-2116


# Verify emails & enrich new form leads and save them to HubSpot

### 1. Workflow Overview

This workflow automates the verification, enrichment, and CRM integration of leads collected via web forms. It targets businesses that gather leads through online forms and face common challenges such as invalid email addresses, limited form data, and manual CRM data entry. The workflow is divided into the following logical blocks:

- **1.1 Input Reception:** Captures new form submissions containing business email addresses.
- **1.2 Email Verification:** Validates the submitted email address using Hunter.io to ensure deliverability.
- **1.3 Data Enrichment:** Enriches valid email leads with personal and company information via Clearbit.
- **1.4 CRM Integration:** Creates or updates leads in HubSpot CRM with enriched data.
- **1.5 Handling Invalid Emails:** Gracefully ignores leads with invalid emails.
- **1.6 Setup and Customization Notes:** Provides guidance and tips on configuration and potential adjustments.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow upon a new form submission, capturing the submitted business email.

**Nodes Involved:**  
- n8n Form Trigger

**Node Details:**  
- **n8n Form Trigger**  
  - Type: Form Trigger  
  - Role: Starts the workflow on form submission  
  - Configuration:  
    - Webhook path uniquely identifies this form trigger URL.  
    - Form titled "Contact us" with a single field: "What's your business email?"  
    - Description: "We'll get back to you soon"  
  - Expressions: Captures the submitted email under the field name "What's your business email?"  
  - Input: External form submission via webhook  
  - Output: JSON object with form data for downstream nodes  
  - Edge cases: Invalid or missing email field submissions could propagate downstream if not handled properly (addressed later)  
  - Version: 2

---

#### 2.2 Email Verification

**Overview:**  
Verifies if the submitted email is valid and deliverable using Hunter.io email verification service.

**Nodes Involved:**  
- Hunter  
- Check if the email is valid (If node)

**Node Details:**  
- **Hunter**  
  - Type: Hunter node  
  - Role: Email verification  
  - Configuration:  
    - Operation: "emailVerifier"  
    - Email: Dynamically taken from form submission field "What's your business email?"  
  - Credentials: Hunter API credentials set up  
  - Input: Form submission JSON  
  - Output: Verification result JSON including status (e.g., "valid")  
  - Edge cases: API rate limits, invalid API credentials, network timeout, malformed email input  
  - Version: 1

- **Check if the email is valid**  
  - Type: If node  
  - Role: Conditional branching based on verification status  
  - Configuration:  
    - Condition: Checks if `$json.status` equals "valid" (strict and case sensitive)  
  - Input: Output from Hunter node  
  - Output: Two branches  
    - True branch for valid emails  
    - False branch for invalid emails  
  - Edge cases: Missing or unexpected status field, case mismatches

---

#### 2.3 Data Enrichment

**Overview:**  
Enriches the lead’s data with personal and company information using Clearbit APIs if the email is verified valid.

**Nodes Involved:**  
- Enrich person  
- Enrich company

**Node Details:**  
- **Enrich person**  
  - Type: Clearbit node  
  - Role: Personal information enrichment  
  - Configuration:  
    - Resource: "person"  
    - Email: Uses `$json.email` from the valid email check  
  - Credentials: Clearbit API credentials required  
  - Input: Valid email JSON from If node's True branch  
  - Output: Person enrichment data including name, employment, location, etc.  
  - Edge cases: API failures, rate limits, no data found for email  
  - Version: 1  
  - Note: Always outputs data even if enrichment fails (`alwaysOutputData: true`), preventing dead ends

- **Enrich company**  
  - Type: Clearbit node  
  - Role: Company information enrichment  
  - Configuration:  
    - Domain: Taken from `employment.domain` field of the person enrichment result  
  - Credentials: Clearbit API credentials  
  - Input: Output from Enrich person node  
  - Output: Company enrichment data including metrics, industry, location, etc.  
  - Edge cases: Missing or invalid domain, API errors

---

#### 2.4 CRM Integration

**Overview:**  
Adds the enriched lead data into HubSpot CRM, populating fields such as email, name, job title, company, and company size.

**Nodes Involved:**  
- Add lead to Hubspot

**Node Details:**  
- **Add lead to Hubspot**  
  - Type: HubSpot node  
  - Role: Create or update contact in HubSpot CRM  
  - Configuration:  
    - Authentication: OAuth2  
    - Email: From "Check if the email is valid" node's JSON email field  
    - Additional fields mapped from Enrich person and Enrich company nodes, including:  
      - Job Title  
      - Last Name  
      - First Name  
      - Company Name  
      - Company Size (employees metric)  
  - Credentials: HubSpot OAuth2 credentials required  
  - Input: Company enrichment JSON  
  - Output: HubSpot contact creation/update response  
  - Edge cases: OAuth token expiration, API rate limits, missing mandatory fields in HubSpot, mapping errors  
  - Version: 2

---

#### 2.5 Handling Invalid Emails

**Overview:**  
Gracefully ends the workflow path for invalid emails without further processing.

**Nodes Involved:**  
- Email is not valid, do nothing (NoOp node)

**Node Details:**  
- **Email is not valid, do nothing**  
  - Type: NoOp (no operation) node  
  - Role: Terminates workflow branch for invalid emails silently  
  - Configuration: None  
  - Input: False branch from If node "Check if the email is valid"  
  - Output: None  
  - Edge cases: None significant; ensures no errors from invalid emails

---

#### 2.6 Setup and Customization Notes

**Overview:**  
Provides users with setup instructions and tips for customization via sticky notes.

**Nodes Involved:**  
- Sticky Note (Setup instructions)  
- Sticky Note1 (Form customization)  
- Sticky Note2 (HubSpot field adjustments)  
- Sticky Note3 (Adding criteria for lead acceptance)

**Node Details:**  
- **Sticky Note** (Setup instructions)  
  - Content:  
    1. Add Hunter, Clearbit, and HubSpot credentials  
    2. Test workflow with an email and verify HubSpot entry  
    3. Activate and use production form URL for leads  
- **Sticky Note1**  
  - Content: You can use any form (Typeform, Google Forms, Survey Monkey, etc.)  
- **Sticky Note2**  
  - Content: Adjust HubSpot fields as per your needs  
- **Sticky Note3**  
  - Content: Suggests adding criteria to filter leads before adding to HubSpot, with link to inspiration template:  
    [Reach out via Email to new form submissions that meet a certain criteria](https://n8n.io/workflows/2106-reach-out-via-email-to-new-form-submissions-that-meet-a-certain-criteria)

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                  | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                           |
|-------------------------------|--------------------|--------------------------------|--------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| n8n Form Trigger               | Form Trigger       | Receive new form submissions   | -                              | Hunter                          |                                                                                                                       |
| Hunter                        | Hunter             | Verify email deliverability    | n8n Form Trigger               | Check if the email is valid     |                                                                                                                       |
| Check if the email is valid   | If                 | Branch on email validity       | Hunter                        | Enrich person, Email is not valid, do nothing |                                                                                                                       |
| Enrich person                 | Clearbit           | Enrich person data             | Check if the email is valid   | Enrich company                  |                                                                                                                       |
| Enrich company                | Clearbit           | Enrich company data            | Enrich person                 | Add lead to Hubspot             |                                                                                                                       |
| Add lead to Hubspot           | HubSpot             | Add enriched lead to CRM       | Enrich company                | -                              | 👆 Adjust the fields you need in your Hubspot here                                                                    |
| Email is not valid, do nothing| NoOp               | Stop processing invalid emails | Check if the email is valid   | -                              |                                                                                                                       |
| Sticky Note                   | Sticky Note        | Setup instructions             | -                            | -                              | 1. Add you **Hunter**, **Clearbit** and **Hubspot** credentials. 2. Click the Test Workflow button, enter your email and check your Hubspot. 3. Activate the workflow and use the form trigger production URL to collect your leads in a smart way |
| Sticky Note1                  | Sticky Note        | Form customization tip         | -                            | -                              | 👆 You can exchange this with any form you like (*e.g. Typeform, Google forms, Survey Monkey...*)                      |
| Sticky Note2                  | Sticky Note        | HubSpot field customization tip| -                            | -                              | 👆 Adjust the fields you need in your Hubspot here                                                                    |
| Sticky Note3                  | Sticky Note        | Lead acceptance criteria tip   | -                            | -                              | 👇 Idea: You could add criteria on when to add a lead to your Hubspot here. For inspiration, take a look at [this template](https://n8n.io/workflows/2106-reach-out-via-email-to-new-form-submissions-that-meet-a-certain-criteria) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Webhook path: Unique identifier (e.g., “0bf8840f-1cc4-46a9-86af-a3fa8da80608”)  
   - Form Title: "Contact us"  
   - Form Fields: One field labeled "What's your business email?"  
   - Form Description: "We'll get back to you soon"  
   - No credentials needed  
   - Connect output to Hunter node

2. **Create Hunter Node**  
   - Type: Hunter  
   - Operation: "emailVerifier"  
   - Email: Expression referencing form field: `{{$json["What's your business email?"]}}`  
   - Credentials: Set up Hunter API credentials (register and obtain API key)  
   - Connect output to "Check if the email is valid" node

3. **Create If Node "Check if the email is valid"**  
   - Type: If  
   - Condition: Check if expression `$json.status` equals "valid" (case-sensitive, strict)  
   - True output to “Enrich person” node  
   - False output to “Email is not valid, do nothing” node

4. **Create Clearbit Node "Enrich person"**  
   - Type: Clearbit  
   - Resource: "person"  
   - Email: Expression: `{{$json.email}}` (taken from Hunter node output)  
   - Credentials: Set up Clearbit API credentials  
   - Connect output to “Enrich company” node  
   - Enable "Always output data" to avoid workflow breaks

5. **Create Clearbit Node "Enrich company"**  
   - Type: Clearbit  
   - Domain: Expression: `{{$json.employment.domain}}` (from Enrich person output)  
   - Credentials: Use same Clearbit API credentials  
   - Connect output to “Add lead to Hubspot” node

6. **Create HubSpot Node "Add lead to Hubspot"**  
   - Type: HubSpot  
   - Authentication: OAuth2 (set up OAuth2 credential in n8n with HubSpot app)  
   - Email: Expression referencing email from valid email check: `{{$node["Check if the email is valid"].json.email}}`  
   - Additional fields:  
     - jobTitle: `{{$node["Enrich person"].json.employment.title}}`  
     - lastName: `{{$node["Enrich person"].json.name.familyName}}`  
     - firstName: `{{$node["Enrich person"].json.name.givenName}}`  
     - companyName: `{{$node["Enrich person"].json.employment.name}}`  
     - companySize: `{{$json.metrics.employees}}` (from Enrich company node)  
   - Credentials: Use HubSpot OAuth2 credentials  
   - No further output required

7. **Create NoOp Node "Email is not valid, do nothing"**  
   - Type: No Operation  
   - Connect false output branch from If node here  
   - No configuration needed

8. **Add Sticky Notes for Documentation (Optional)**  
   - Setup instructions, customization tips, and lead filtering ideas as per the original workflow, including links.

9. **Final Connections:**  
   - n8n Form Trigger → Hunter → If Node  
   - If True → Enrich person → Enrich company → Add lead to HubSpot  
   - If False → NoOp node

10. **Test Workflow:**  
    - Use test mode in Form Trigger to submit email addresses  
    - Verify email validation, enrichment, and presence in HubSpot CRM

11. **Activate Workflow:**  
    - Deploy the webhook URL from the Form Trigger to production environment for live lead capture

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| This workflow can be adapted to any form collection service such as Typeform, Google Forms, SurveyMonkey, etc.                         | Sticky Note1                                                                                                                        |
| Adjust HubSpot field mappings to your CRM schema and requirements.                                                                     | Sticky Note2                                                                                                                        |
| Consider adding filtering criteria before adding leads to HubSpot to target specific industries, company sizes, or other attributes.  | Inspiration Template: [Reach out via Email to new form submissions that meet a certain criteria](https://n8n.io/workflows/2106-reach-out-via-email-to-new-form-submissions-that-meet-a-certain-criteria) |
| Credentials required: Hunter API key, Clearbit API key, HubSpot OAuth2 credentials                                                      | Setup instructions in Sticky Note                                                                                                  |
| Workflow handles invalid emails by silently ignoring them, preventing unnecessary CRM clutter                                          | Design choice for robust lead quality control                                                                                       |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and extending the "Verify emails & enrich new form leads and save them to HubSpot" workflow in n8n.