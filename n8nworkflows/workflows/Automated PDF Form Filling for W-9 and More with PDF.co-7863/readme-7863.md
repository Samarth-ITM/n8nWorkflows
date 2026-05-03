Automated PDF Form Filling for W-9 and More with PDF.co

https://n8nworkflows.xyz/workflows/automated-pdf-form-filling-for-w-9-and-more-with-pdf-co-7863


# Automated PDF Form Filling for W-9 and More with PDF.co

### 1. Workflow Overview

This workflow automates the process of filling out a standard PDF form—in this case, the IRS W-9—using structured input data within n8n and the PDF.co API. It is designed for scenarios where user or business data (e.g., Name, Business, Address, City/State) needs to be programmatically inserted into fixed PDF form fields, supporting automation for contracts, invoices, HR forms, and tax documents.

**Logical Blocks:**

- **1.1 Input Data Preparation:** Captures and structures user data that will be inserted into the PDF form.
- **1.2 PDF Form Filling:** Uses the PDF.co API node to fill the specified PDF form with the prepared data.
- **1.3 Manual Trigger:** Initiates the workflow execution manually.
- **1.4 Setup and Instructional Notes:** Provides setup instructions and contextual information to assist users in configuring and understanding the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Data Preparation

- **Overview:**  
  This block defines and sets the data fields (Name, Business, Address, CityState) that will populate the PDF form fields.

- **Nodes Involved:**  
  - W9 Data

- **Node Details:**  

  - **W9 Data**  
    - *Type & Role:* Set node; prepares structured input data.  
    - *Configuration:* Four string fields assigned with static example values:  
      - Name = "Jim"  
      - Business = "Bobs Appliances"  
      - Address = "general ave"  
      - CityState = "Atlanta GA"  
    - *Expressions / Variables:* Static values assigned directly; no dynamic expressions used.  
    - *Connections:* Input from manual trigger node; output to PDF form filling node.  
    - *Version:* Uses typeVersion 3.4 (latest stable Set node version).  
    - *Potential Failure Points:* Unlikely to fail unless workflow is triggered without this node feeding required data downstream.  
    - *Notes:* This node should be modified or replaced to dynamically fetch or input user data in real use cases.

#### 1.2 PDF Form Filling

- **Overview:**  
  This block takes the structured data and fills the IRS W-9 PDF form by mapping each data field to the corresponding PDF form field using PDF.co API.

- **Nodes Involved:**  
  - Fill in PDF Form

- **Node Details:**  

  - **Fill in PDF Form**  
    - *Type & Role:* PDF.co API node; performs the PDF form filling operation.  
    - *Configuration:*  
      - Operation: "Fill a PDF Form"  
      - PDF URL: "https://www.irs.gov/pub/irs-pdf/fw9.pdf" (IRS W-9 form)  
      - Fields: Metadata array mapping JSON fields to PDF form field names:  
        - Name => topmostSubform[0].Page1[0].f1_01[0]  
        - Business => topmostSubform[0].Page1[0].f1_02[0]  
        - Address => topmostSubform[0].Page1[0].Address_ReadOrder[0].f1_07[0]  
        - CityState => topmostSubform[0].Page1[0].Address_ReadOrder[0].f1_08[0]  
      - Credentials: Uses stored PDF.co API credentials (requires API key setup).  
    - *Expressions / Variables:* Uses expressions to dynamically insert data from the previous node (`{{$json.<FieldName>}}`).  
    - *Connections:* Receives input from "W9 Data" node; output is the filled PDF document (usually a URL or file object).  
    - *Version:* typeVersion 1 (PDF.co integration).  
    - *Potential Failure Points:*  
      - Invalid or missing PDF.co API credentials (authentication error).  
      - Incorrect field names in the PDF form mapping causing data not to populate.  
      - Network or timeout issues when accessing the external PDF URL or API.  
      - Rate limiting or API quota exceeded on PDF.co side.  
    - *Notes:* Requires prior credential configuration with valid PDF.co API key.

#### 1.3 Manual Trigger

- **Overview:**  
  Provides a manual way to start the workflow execution from the n8n editor or UI.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - *Type & Role:* Manual Trigger node; initiates workflow execution on demand.  
    - *Configuration:* Default, no parameters required.  
    - *Connections:* Output connects to "W9 Data" node.  
    - *Version:* typeVersion 1.  
    - *Potential Failure Points:* None typical; relies on user to manually trigger.  

#### 1.4 Setup and Instructional Notes

- **Overview:**  
  These nodes provide users with instructions and context about configuring the workflow and using PDF.co API.

- **Nodes Involved:**  
  - Sticky Note3  
  - Sticky Note56  
  - Sticky Note63

- **Node Details:**  

  - **Sticky Note3**  
    - *Type & Role:* Sticky Note; detailed step-by-step setup instructions for preparing data and connecting PDF.co.  
    - *Content Highlights:*  
      - Advises adding a Set node with specific fields.  
      - Guides on creating a PDF.co account and setting up credentials in n8n.  
      - Provides contact info for customization support.  
    - *Connections:* None (informational only).  
    - *Version:* typeVersion 1.  

  - **Sticky Note56**  
    - *Type & Role:* Sticky Note; overview of the workflow’s purpose and adaptability for other forms.  
    - *Content Highlights:*  
      - Explains workflow use case for auto-filling PDF forms with PDF.co inside n8n.  
      - Emphasizes adaptability for other standardized PDF forms.  
    - *Connections:* None (informational only).  
    - *Version:* typeVersion 1.  

  - **Sticky Note63**  
    - *Type & Role:* Sticky Note; concise instructions for connecting PDF.co credentials.  
    - *Content Highlights:*  
      - Steps to create PDF.co account and API key.  
      - Instructions to add PDF.co credentials in n8n.  
      - Reminder to select correct operation in PDF.co node.  
    - *Connections:* None (informational only).  
    - *Version:* typeVersion 1.  

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role               | Input Node(s)              | Output Node(s)         | Sticky Note                                                                                                            |
|----------------------------|----------------------------|------------------------------|----------------------------|------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Initiates workflow execution  | —                          | W9 Data                |                                                                                                                        |
| W9 Data                    | Set                        | Prepares input data fields    | When clicking ‘Execute workflow’ | Fill in PDF Form       |                                                                                                                        |
| Fill in PDF Form           | PDF.co API                 | Fills PDF form with data      | W9 Data                    | —                      |                                                                                                                        |
| Sticky Note3               | Sticky Note                | Setup instructions and contact info | —                          | —                      | Detailed setup instructions including PDF.co API key setup and data preparation steps. Contact info for customization. |
| Sticky Note56              | Sticky Note                | Workflow overview and use case | —                          | —                      | Explains workflow purpose and adaptability for other PDF forms.                                                        |
| Sticky Note63              | Sticky Note                | PDF.co connection instructions | —                          | —                      | Steps to create PDF.co account, API key, and add credentials in n8n.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’".  
   - No parameters needed. This node will start the workflow manually.

2. **Create a Set Node for Input Data**  
   - Add a **Set** node named "W9 Data".  
   - Add the following fields with static string values (for initial testing):  
     - Name: "Jim"  
     - Business: "Bobs Appliances"  
     - Address: "general ave"  
     - CityState: "Atlanta GA"  
   - Connect output of "When clicking ‘Execute workflow’" to "W9 Data".

3. **Configure PDF.co API Credentials in n8n**  
   - Go to **Credentials** → **New** → **PDF.co API**.  
   - Sign up or log in at [PDF.co](https://pdf.co/).  
   - Copy your API key from the PDF.co dashboard.  
   - Paste the API key into the n8n credential form and save.

4. **Add PDF.co API Node to Fill PDF Form**  
   - Add a **PDF.co API** node named "Fill in PDF Form".  
   - Set **Operation** to "Fill a PDF Form".  
   - Set **URL** to the IRS W-9 PDF: `https://www.irs.gov/pub/irs-pdf/fw9.pdf`.  
   - Configure the **Fields** parameter as an array of metadataValues, mapping:  
     - `topmostSubform[0].Page1[0].f1_01[0]` → `{{$json.Name}}`  
     - `topmostSubform[0].Page1[0].f1_02[0]` → `{{$json.Business}}`  
     - `topmostSubform[0].Page1[0].Address_ReadOrder[0].f1_07[0]` → `{{$json.Address}}`  
     - `topmostSubform[0].Page1[0].Address_ReadOrder[0].f1_08[0]` → `{{$json.CityState}}`  
   - Select the PDF.co API credential created earlier.  
   - Connect output of "W9 Data" node to "Fill in PDF Form" node.

5. **Add Sticky Notes for Documentation (optional but recommended)**  
   - Add three **Sticky Note** nodes with the following content:  
     - Setup instructions for data preparation and API key setup.  
     - Workflow overview and use cases.  
     - PDF.co connection instructions with links and steps.

6. **Test the Workflow**  
   - Execute the workflow manually via the "When clicking ‘Execute workflow’" node.  
   - Confirm the output from the PDF.co node contains the filled PDF or link to the filled document.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Create a free account at PDF.co and obtain your API Key to enable PDF form filling.                            | https://pdf.co/                                                                                 |
| The workflow is designed for IRS W-9 form but can be adapted to other standardized PDF forms by changing URL and field mappings. | IRS W-9 PDF URL: https://www.irs.gov/pub/irs-pdf/fw9.pdf                                       |
| For customization help on contracts, invoices, or HR forms, contact Robert Breen via email or LinkedIn.       | Email: robert@ynteractive.com <br> LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ |
| Refer to sticky notes in the workflow for step-by-step setup instructions and credential configuration.        | Embedded in the workflow as Sticky Note nodes                                                   |

---

**Disclaimer:** This document is generated exclusively from an n8n workflow automation. All data manipulated is legal and public. The workflow does not include any illegal or protected content.