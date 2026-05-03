Automatically Create Trello Cards from Google Forms Lead Submissions

https://n8nworkflows.xyz/workflows/automatically-create-trello-cards-from-google-forms-lead-submissions-4946


# Automatically Create Trello Cards from Google Forms Lead Submissions

### 1. Workflow Overview

This workflow automates the creation of Trello cards from new lead submissions collected via a Google Form connected to a Google Sheets spreadsheet. Its target use case is lead management automation, where form responses are transformed into structured Trello cards with both standard and custom field data, facilitating organized lead tracking.

The workflow logic is divided into these main functional blocks:

- **1.1 Input Reception:** Detect new form submissions in Google Sheets.
- **1.2 Data Preparation:** Extract and map form fields into workflow variables.
- **1.3 Trello Card Creation:** Create a new Trello card using mapped form data.
- **1.4 Custom Fields Setup:** Update Trello card custom fields for detailed lead info.
- **1.5 Conditional Field Mapping:** Use Switch and Set nodes to map enumerated form values (Company Size, Lead Source, Company Industry) to Trello custom field option IDs.
- **1.6 Sequential Custom Field Updates:** Chain updates to Trello card custom fields for Company Size, Lead Source, and Industry.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new rows added to a specific Google Sheets spreadsheet linked to the Google Form responses.

**Nodes Involved:**  
- New Form Submission  
- New Form  

**Node Details:**  

- **New Form Submission**  
  - Type: Google Sheets Trigger  
  - Role: Triggers workflow on new row added (new form submission)  
  - Config: Polls every minute on the "Form Responses 1" sheet of the designated spreadsheet  
  - Credentials: Google Sheets OAuth2  
  - Inputs: None (trigger node)  
  - Outputs: New form response data as JSON  
  - Edge Cases: API rate limits, permission errors, sheet renaming or deletion  

- **New Form**  
  - Type: Google Sheets Trigger  
  - Role: Another trigger on the same spreadsheet and sheet, appearing to duplicate "New Form Submission" (likely for a parallel processing path)  
  - Config: Same as above  
  - Credentials: Same Google Sheets OAuth2  
  - Inputs: None  
  - Outputs: New form response data  
  - Edge Cases: Same as above  

---

#### 1.2 Data Preparation

**Overview:**  
Extracts specific form fields from the new submission and assigns them to workflow variables for easier referencing downstream.

**Nodes Involved:**  
- Set Fields  

**Node Details:**  

- **Set Fields**  
  - Type: Set  
  - Role: Maps form response fields to named variables such as Name, EmailAddress, Company Name, Company Size, Needs, Lead Source, Company Website, Company Industry, and listID (hardcoded Trello list)  
  - Configuration: Assigns string values from JSON response fields using expressions like `={{ $json['What\'s your name? '] }}`  
  - Inputs: From "New Form" trigger node  
  - Outputs: JSON with mapped fields for next nodes  
  - Edge Cases: Missing form fields, unexpected formats, trailing spaces in keys (e.g., "Email Address ") may cause issues if form structure changes  

---

#### 1.3 Trello Card Creation

**Overview:**  
Creates a new Trello card with a title composed of the lead's name and company, and a description containing key lead information.

**Nodes Involved:**  
- Create Card  
- Create Lead (appears in parallel path, see below)  

**Node Details:**  

- **Create Card**  
  - Type: Trello  
  - Role: Creates Trello card in specified list (using listID from Set Fields)  
  - Configuration: Card name formatted as `"Name | Company Name"`, description populated with Needs only (from Set Fields output)  
  - Inputs: From "Set Fields"  
  - Outputs: Trello card data including card ID  
  - Credentials: Trello API OAuth  
  - Edge Cases: Invalid list ID, API limits, authentication errors  

- **Create Lead**  
  - Type: Trello  
  - Role: Similar to Create Card but in the separate "New Form Submission" trigger path  
  - Configuration: Card name `"Name | Company Name"`, description includes multiple fields: Name, Phone Number, Email, Company Name, Company Size, Needs  
  - Inputs: From "New Form Submission"  
  - Outputs: Created Trello card data  
  - Credentials: Trello API OAuth  
  - Edge Cases: Same as Create Card; redundancy in workflow may cause duplicate cards if both triggers fire  

---

#### 1.4 Custom Fields Setup

**Overview:**  
Sets up the Trello card's custom fields by updating them via HTTP requests to Trello API using the card ID and custom field IDs mapped beforehand.

**Nodes Involved:**  
- Set IDs  
- Trello | Update Custom Fields  
- Trello | Update Company Size  
- Trello| Update Lead Source  
- Trello| Update Company Industry  

**Node Details:**  

- **Set IDs**  
  - Type: Set  
  - Role: Holds Trello custom field IDs for fields like Name, Phone Number, Email, Company Name, Company Size, Company Industry, Company Website, Lead Source  
  - Configuration: Assigns string values with placeholders `{field-id}` that must be replaced with actual Trello custom field IDs before use  
  - Inputs: From "Create Card"  
  - Outputs: JSON with custom field IDs for HTTP requests  
  - Edge Cases: Missing or incorrect field IDs will cause API requests to fail  

- **Trello | Update Custom Fields**  
  - Type: HTTP Request  
  - Role: Updates multiple Trello custom fields at once (Name, Phone Number, Email, Company Name, Company Website)  
  - Configuration: PUT request to Trello API endpoint `/cards/{cardId}/customFields` with JSON body containing `customFieldItems` array  
  - Inputs: From "Set IDs"  
  - Outputs: HTTP response from Trello  
  - Credentials: Trello API OAuth  
  - Edge Cases: API errors if field IDs or values are invalid, network issues, rate limits  

- **Trello | Update Company Size**  
  - Type: HTTP Request  
  - Role: Updates only the Company Size custom field on the card, setting the option ID selected from the Switch node results  
  - Configuration: PUT request with JSON body specifying `idCustomField` and `idValue` (option ID)  
  - Inputs: From "Solo (1)", "Small (2–10)", "Medium (11–50)", "Large (51–200)", or "Enterprise (200+)" nodes via Options for Company Size Switch  
  - Outputs: HTTP response  
  - Credentials: Trello API OAuth  
  - Edge Cases: Invalid option ID, incorrect card ID, API errors  

- **Trello| Update Lead Source**  
  - Type: HTTP Request  
  - Role: Updates Lead Source custom field similarly to Company Size  
  - Configuration: PUT request, uses option ID from Lead Source option nodes (Youtube, Facebook/Instagram, Google, Friend/Family)  
  - Inputs: From corresponding Set nodes after Options for Lead Source Switch  
  - Outputs: HTTP response  
  - Credentials: Trello API OAuth  
  - Edge Cases: Same as above  

- **Trello| Update Company Industry**  
  - Type: HTTP Request  
  - Role: Updates Company Industry custom field on the card  
  - Configuration: PUT request, uses option ID determined by a Code node mapping industry strings to Trello option IDs  
  - Inputs: From "Options for Company Industry" Code node  
  - Outputs: HTTP response  
  - Credentials: Trello API OAuth  
  - Edge Cases: Missing mapping for some industries logs warnings, failed API calls  

---

#### 1.5 Conditional Field Mapping

**Overview:**  
Maps textual form input values for Company Size, Lead Source, and Company Industry to Trello custom field option IDs using Switch and Set nodes or a Code node.

**Nodes Involved:**  
- Options for Company Size (Switch)  
- Solo (1), Small (2–10), Medium (11–50), Large (51–200), Enterprise (200+) (Set nodes)  
- Options for Lead Source (Switch)  
- Youtube, Facebook/Instagram, Google, Friendy/Family (Set nodes)  
- Options for Company Industry (Code)  

**Node Details:**  

- **Options for Company Size**  
  - Type: Switch  
  - Role: Routes based on exact match of Company Size string from Set Fields  
  - Outputs: One of five Set nodes assigning Trello option IDs for company size  

- **Solo (1), Small (2–10), Medium (11–50), Large (51–200), Enterprise (200+)**  
  - Type: Set  
  - Role: Assign corresponding Trello option ID to `sizeOption` variable  
  - Configuration: Placeholder `{option-id}` to be replaced  

- **Options for Lead Source**  
  - Type: Switch  
  - Role: Routes based on exact match of Lead Source string  
  - Outputs: One of four Set nodes assigning Trello option IDs for lead source  

- **Youtube, Facebook/Instagram, Google, Friendy/Family**  
  - Type: Set  
  - Role: Assign corresponding Trello option ID to `sourceOption` variable  

- **Options for Company Industry**  
  - Type: Code (JavaScript)  
  - Role: Maps Company Industry strings to Trello option IDs using a switch-case construct  
  - Configuration: Hardcoded `{option-id}` placeholders in each case to be replaced with actual Trello IDs  
  - Outputs: JSON containing `trelloIndustryOptionId` for next node  
  - Edge Cases: Logs warning if no mapping found for industry  

---

#### 1.6 Sequential Custom Field Updates

**Overview:**  
Executes the updates to Trello card custom fields in a sequence, ensuring Company Size update triggers Lead Source update, which triggers Company Industry update.

**Nodes Involved:**  
- Trello | Update Company Size  
- Trello| Update Lead Source  
- Trello| Update Company Industry  

**Node Details:**  

- Each node is an HTTP Request to Trello API updating one custom field at a time.  
- They are connected in a chain for sequential execution.  
- They use the card ID from the created card and option IDs from previous mapping nodes.  
- Credentials: Trello API OAuth  
- Edge Cases: If any update fails, subsequent updates may not execute; no explicit error handling shown  

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                               | Input Node(s)              | Output Node(s)                     | Sticky Note                                                                                      |
|-----------------------------|-----------------------|-----------------------------------------------|----------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note           | Informational note on no custom fields        |                            |                                   | ### No custom fields All the information will be saved in the description.                       |
| New Form Submission         | Google Sheets Trigger | Trigger workflow on new Google Form submission |                            | Create Lead                       |                                                                                                 |
| Create Lead                 | Trello                | Create Trello card with detailed description  | New Form Submission         | Set IDs                          |                                                                                                 |
| New Form                    | Google Sheets Trigger | Trigger workflow on new Google Form submission |                            | Set Fields                      |                                                                                                 |
| Set Fields                  | Set                   | Map Google Form fields to workflow variables  | New Form                    | Create Card                     |                                                                                                 |
| Create Card                 | Trello                | Create Trello card with basic description     | Set Fields                  | Set IDs                         |                                                                                                 |
| Set IDs                    | Set                   | Store Trello custom field IDs                  | Create Card / Create Lead   | Trello | Update Custom Fields       |                                                                                                 |
| Trello | Update Custom Fields | HTTP Request          | Update basic custom fields on Trello card     | Set IDs                    | Options for Company Size           |                                                                                                 |
| Options for Company Size    | Switch                | Route by Company Size text to option IDs      | Trello | Update Custom Fields | Solo (1), Small (2–10), Medium (11–50), Large (51–200), Enterprise (200+) |                                                                                                 |
| Solo (1)                   | Set                   | Assign Trello option ID for Solo size          | Options for Company Size    | Trello | Update Company Size         |                                                                                                 |
| Small (2–10)               | Set                   | Assign Trello option ID for Small size         | Options for Company Size    | Trello | Update Company Size         |                                                                                                 |
| Medium (11–50)             | Set                   | Assign Trello option ID for Medium size        | Options for Company Size    | Trello | Update Company Size         |                                                                                                 |
| Large (51–200)             | Set                   | Assign Trello option ID for Large size         | Options for Company Size    | Trello | Update Company Size         |                                                                                                 |
| Enterprise (200+)          | Set                   | Assign Trello option ID for Enterprise size    | Options for Company Size    | Trello | Update Company Size         |                                                                                                 |
| Trello | Update Company Size | HTTP Request          | Update Company Size custom field               | Solo, Small, Medium, Large, Enterprise | Options for Lead Source            |                                                                                                 |
| Options for Lead Source    | Switch                | Route by Lead Source to option IDs             | Trello | Update Company Size     | Youtube, Facebook/Instagram, Google, Friendy/Family |                                                                                                 |
| Youtube                   | Set                   | Assign Trello option ID for Youtube source     | Options for Lead Source     | Trello| Update Lead Source         |                                                                                                 |
| Facebook/Instagram        | Set                   | Assign Trello option ID for Facebook/Instagram | Options for Lead Source     | Trello| Update Lead Source         |                                                                                                 |
| Google                    | Set                   | Assign Trello option ID for Google source      | Options for Lead Source     | Trello| Update Lead Source         |                                                                                                 |
| Friendy/Family            | Set                   | Assign Trello option ID for Friend/Family source | Options for Lead Source     | Trello| Update Lead Source         |                                                                                                 |
| Trello| Update Lead Source | HTTP Request           | Update Lead Source custom field                | Youtube, Facebook/Instagram, Google, Friendy/Family | Options for Company Industry         |                                                                                                 |
| Options for Company Industry | Code                 | Map Company Industry text to Trello option ID | Trello| Update Lead Source     | Trello| Update Company Industry     |                                                                                                 |
| Trello| Update Company Industry | HTTP Request       | Update Company Industry custom field           | Options for Company Industry |                                   |                                                                                                 |
| Sticky Note1               | Sticky Note           | Informational note on custom fields update     |                            |                                   | ### Custom Fields This workflow will create the card and update the custom fields.               |
| Sticky Note2               | Sticky Note           | Setup instructions for Google Sheets and card mapping |                            |                                   | ## Set Up Steps Step 1: Connect to Google Sheets Select the Google Sheets spreadsheet that receives responses from your Google Form. This should be the sheet automatically created when you set up form responses to be collected in a spreadsheet. Step 2: Configure Card Details Map the form response data to create your card: Card Name: Select the form field that will serve as the card title. Card Description: Choose the form field(s) that will populate the card description. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node ("New Form Submission")**  
   - Node Type: Google Sheets Trigger  
   - Credentials: Setup Google Sheets OAuth2 with access to your form response spreadsheet  
   - Configuration:  
     - Event: Row Added  
     - Document ID: Your Google Sheets spreadsheet ID linked to the Google Form responses  
     - Sheet Name: Select the specific sheet collecting form responses (usually "Form Responses 1")  
     - Polling: Every minute  

2. **Create Google Sheets Trigger Node ("New Form")**  
   - Duplicate of step 1, same configuration (optional, if both parallel paths desired)  

3. **Create Set Node ("Set Fields")**  
   - Node Type: Set  
   - Connect input from "New Form" trigger node  
   - Assign variables for all relevant form fields using expressions, e.g.:  
     - Name → `={{ $json["What's your name? "] }}`  
     - EmailAddress → `={{ $json["Email Address "] }}`  
     - Company Name → `={{ $json["Company Name "] }}`  
     - Company Size → `={{ $json["Company Size "] }}`  
     - Needs → `={{ $json["What specific challenges or goals would you like assistance with?  "] }}`  
     - Lead Source → `={{ $json["How did you find us?"] }}`  
     - Company Website → `={{ $json["Company Website"] }}`  
     - Company Industry → `={{ $json["Company Industry"] }}`  
     - listID → hardcode your Trello list ID as a string  

4. **Create Trello Node ("Create Card")**  
   - Node Type: Trello  
   - Connect input from "Set Fields"  
   - Credentials: Setup Trello API OAuth2 with your account  
   - Configuration:  
     - List ID: Use `={{ $json.listID }}`  
     - Name: `={{ $json.Name }} | {{ $json["Company Name "] }}`  
     - Description: Use `={{ $json.Needs }}` (or other fields as desired)  

5. **Create Set Node ("Set IDs")**  
   - Node Type: Set  
   - Connect input from "Create Card" node  
   - Assign variables for all Trello custom field IDs with your actual Trello custom field IDs, e.g.:  
     - fieldName: `"your-field-id-for-name"`  
     - fieldPhoneNumber: `"your-field-id-for-phone"`  
     - fieldEmail: `"your-field-id-for-email"`  
     - fieldCompanyName: `"your-field-id-for-company-name"`  
     - fieldCompanySize: `"your-field-id-for-company-size"`  
     - fieldCompanyIndustry: `"your-field-id-for-company-industry"`  
     - fieldCompanyWebsite: `"your-field-id-for-company-website"`  
     - fieldLeadSource: `"your-field-id-for-lead-source"`  

6. **Create HTTP Request Node ("Trello | Update Custom Fields")**  
   - Node Type: HTTP Request  
   - Connect input from "Set IDs"  
   - Credentials: Trello API OAuth2  
   - Configuration:  
     - Method: PUT  
     - URL: `https://api.trello.com/1/cards/{{ $json["id"] }}/customFields` (card ID from "Create Card")  
     - Body Format: JSON  
     - Body Content: JSON with the array of customFieldItems for Name, Phone, Email, Company Name, Company Website using `$json` and expressions referencing "Set Fields" variables and "Set IDs" for field IDs  

7. **Create Switch Node ("Options for Company Size")**  
   - Node Type: Switch  
   - Connect input from "Trello | Update Custom Fields"  
   - Configure rules for exact string matches on `Company Size` variable (from "Set Fields"), e.g.:  
     - "Solo (1)" → output 1  
     - "Small (2–10)" → output 2  
     - "Medium (11–50)" → output 3  
     - "Large (51–200)" → output 4  
     - "Enterprise (200+)" → output 5  

8. **Create Set Nodes for Company Size Options**  
   - For each output of the Switch node, create a Set node (e.g., "Solo (1)")  
   - Assign variable `sizeOption` to Trello option ID string corresponding to that size (replace `{option-id}` with your actual Trello option IDs)  

9. **Create HTTP Request Node ("Trello | Update Company Size")**  
   - Connect inputs from all Company Size Set nodes  
   - Credentials: Trello API OAuth2  
   - Method: PUT  
   - URL: `https://api.trello.com/1/cards/{{ $json["id"] }}/customFields`  
   - Body: JSON updating company size custom field with `"idCustomField"` from "Set IDs" and `"idValue"` from `sizeOption` variable from Set nodes  

10. **Create Switch Node ("Options for Lead Source")**  
    - Node Type: Switch  
    - Connect input from "Trello | Update Company Size"  
    - Configure rules matching Lead Source strings (`Youtube`, `Facebook/Instagram`, `Google`, `Friend/Family`)  

11. **Create Set Nodes for Lead Source Options**  
    - For each output of the Lead Source Switch, create a Set node assigning `sourceOption` variable with corresponding Trello option ID (replace `{option-id}`)  

12. **Create HTTP Request Node ("Trello| Update Lead Source")**  
    - Connect inputs from all Lead Source Set nodes  
    - Configure similarly to Company Size update, updating lead source custom field  

13. **Create Code Node ("Options for Company Industry")**  
    - Connect input from "Trello| Update Lead Source"  
    - In JavaScript, map Company Industry strings to Trello option IDs using a switch-case, e.g.:  
      ```js
      let trelloIndustryOptionId = null;
      switch(companyIndustry) {
        case 'Consumer Goods': trelloIndustryOptionId = 'option-id-1'; break;
        case 'Education': trelloIndustryOptionId = 'option-id-2'; break;
        // Add all industry mappings here
        default: console.warn(`No matching Trello option for industry: ${companyIndustry}`); break;
      }
      return [{ json: { trelloIndustryOptionId } }];
      ```  
      Replace all `{option-id}` placeholders with your Trello custom field option IDs.  

14. **Create HTTP Request Node ("Trello| Update Company Industry")**  
    - Connect input from the Code node  
    - Configure PUT request to update the Company Industry custom field using card ID and returned `trelloIndustryOptionId`  

15. **Set up Credentials**  
    - Google Sheets OAuth2: Configure with access to your Google Form response spreadsheet  
    - Trello API OAuth2: Configure with permissions to create cards and update custom fields on your Trello board  

16. **Optional: Create "Create Lead" node path**  
    - This path duplicates card creation from "New Form Submission" trigger with more detailed description  
    - Connect "New Form Submission" → "Create Lead" → "Set IDs" → "Trello | Update Custom Fields" → "Options for Company Size" (rest of chain same as above)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| No custom fields: All the information will be saved in the description.                                                              | Sticky Note near "New Form Submission" and "Create Lead" nodes                                     |
| Custom Fields: This workflow will create the card and update the custom fields.                                                      | Sticky Note near custom field update HTTP Request nodes                                           |
| Setup Steps: Connect to Google Sheets and configure card details (title, description) mapped from form responses.                   | Sticky Note providing high-level setup instructions                                              |
| Trello API Reference: https://developer.atlassian.com/cloud/trello/rest/api-group-cards/#api-cards-id-customfields-put                | For custom field updates via HTTP requests                                                       |
| Google Sheets Trigger Docs: https://docs.n8n.io/integrations/builtin/triggers/google-sheets-trigger/                                 | For configuring the Google Sheets trigger node                                                   |
| Trello OAuth2 Setup: https://docs.n8n.io/integrations/builtin/credentials/trello/                                                     | For setting up Trello API credentials in n8n                                                     |

---

This completes the detailed analysis and reproduction instructions for the "Automatically Create Trello Cards from Google Forms Lead Submissions" workflow.