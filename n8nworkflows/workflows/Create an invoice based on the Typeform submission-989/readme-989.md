Create an invoice based on the Typeform submission

https://n8nworkflows.xyz/workflows/create-an-invoice-based-on-the-typeform-submission-989


# Create an invoice based on the Typeform submission

### 1. Workflow Overview

This workflow automates the creation of a PDF invoice based on data submitted through a Typeform form. When a user submits the Typeform, the workflow triggers, collects the form data, and uses the APITemplate.io service to generate a customized invoice PDF with the provided details.

The workflow contains two main logical blocks:

- **1.1 Input Reception:** Receiving and processing submission data from Typeform.
- **1.2 Invoice Generation:** Using APITemplate.io to create and download a PDF invoice populated with the submitted data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming submissions from a specific Typeform and prepares the data for downstream processing.

**Nodes Involved:**  
- Typeform Trigger

**Node Details:**

- **Node Name:** Typeform Trigger  
- **Type:** Trigger node — Typeform Trigger  
- **Configuration:**  
  - Listens to a Typeform form with ID `dpr2kxSL`.  
  - Does not simplify answers (raw response data structure preserved).  
  - Uses credentials named "Typeform Burner Account" for API access.  
  - Webhook ID assigned by n8n to receive form submissions.  
- **Key Expressions / Variables:** The node outputs the entire form submission JSON, which includes an array of answers indexed by question order.  
- **Input Connections:** None (trigger node).  
- **Output Connections:** Connects to the "APITemplate.io" node for invoice generation.  
- **Version Specifics:** Uses typeVersion 1 of the Typeform node, which supports webhook-based triggers.  
- **Potential Failure Modes:**  
  - Authentication failure due to invalid or expired Typeform credentials.  
  - Network or webhook delivery issues causing missed triggers.  
  - Changes in Typeform form structure could affect data indexing.  
- **Sub-workflow Reference:** None.

#### 1.2 Invoice Generation

**Overview:**  
This block takes the Typeform submission data and passes it to APITemplate.io to generate a PDF invoice. The invoice is dynamically populated using data extracted from the form answers.

**Nodes Involved:**  
- APITemplate.io

**Node Details:**

- **Node Name:** APITemplate.io  
- **Type:** Action node — APITemplate.io PDF generation  
- **Configuration:**  
  - Resource: PDF  
  - PDF Template ID: `96c77b2b1ab6ac88` (predefined invoice template stored in APITemplate.io)  
  - Download enabled: The generated PDF is downloaded automatically.  
  - Output file name: `invoice.pdf`  
  - Input JSON parameters: A JSON object with fields for company details, invoice numbers, dates, address, website, document ID, and a list of items.  
- **Key Expressions / Variables:**  
  - Extracts email from the second answer in the form submission (`{{$json["1"]["email"]}}`).  
  - Pulls billing name from the first answer's text (`{{$json["0"]["text"]}}`).  
  - Uses item names and prices from subsequent answers (`{{$json["2"]["text"]}}`, `{{$json["3"]["number"]}}`, etc.).  
  - Hardcoded values for company name (`n8n`), invoice number, dates, address, website, and document_id.  
- **Input Connections:** Receives data from the "Typeform Trigger" node.  
- **Output Connections:** None (end of workflow).  
- **Credentials:** Uses "APITemplate Credentials" for authentication with APITemplate.io API.  
- **Version Specifics:** Uses typeVersion 1 of the APITemplate.io node.  
- **Potential Failure Modes:**  
  - API authentication failure due to invalid credentials.  
  - Incorrect template ID or template misconfiguration causing generation errors.  
  - Malformed JSON parameter expressions leading to runtime errors.  
  - Network or API downtime preventing PDF generation.  
- **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name         | Node Type               | Functional Role       | Input Node(s)    | Output Node(s)    | Sticky Note                                                                                 |
|-------------------|-------------------------|----------------------|------------------|-------------------|---------------------------------------------------------------------------------------------|
| Typeform Trigger  | Typeform Trigger        | Input Reception      | None             | APITemplate.io    | This node triggers the workflow. Whenever the form is submitted, the node triggers the workflow. We will use the information received in this node to generate the invoice. |
| APITemplate.io    | APITemplate.io          | Invoice Generation   | Typeform Trigger | None              | This node generates the invoice using the information from the previous node.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Typeform Trigger" node:**
   - Set the node name to "Typeform Trigger".
   - Configure the node to listen to the form with ID `dpr2kxSL`.
   - Set "Simplify Answers" to `false` to keep raw data format.
   - Add credentials for Typeform API (create or select a credential named, for example, "Typeform Burner Account").
   - Save the node; n8n will create a webhook URL for the trigger.
   
3. **Add an "APITemplate.io" node:**
   - Set the node name to "APITemplate.io".
   - Set resource to `PDF`.
   - Select or enter the PDF Template ID: `96c77b2b1ab6ac88`.
   - Enable "Download" to save the PDF file automatically.
   - Set output file name to `invoice.pdf`.
   - Enable `jsonParameters` to allow a JSON object as input.
   - Configure the JSON parameters for invoice generation with the following structure, using expressions to pull from the Typeform submission data:

     ```json
     {
       "company": "n8n",
       "email": "{{$json[\"1\"][\"email\"]}}",
       "invoice_no": "213223444",
       "invoice_date": "18-03-2021",
       "invoice_due_date": "17-04-2021",
       "address": "Berlin, Germany",
       "company_bill_to": "{{$json[\"0\"][\"text\"]}}",
       "website": "https://n8n.io",
       "document_id": "889856789012",
       "items": [
         {
           "item_name": "{{$json[\"2\"][\"text\"]}}",
           "price": "EUR {{$json[\"3\"][\"number\"]}}"
         },
         {
           "item_name": "{{$json[\"4\"][\"text\"]}}",
           "price": "EUR {{$json[\"5\"][\"number\"]}}"
         }
       ]
     }
     ```

   - Add API credentials for APITemplate.io (create or select credentials, e.g., "APITemplate Credentials").
   
4. **Connect the nodes:**
   - Connect the main output of the "Typeform Trigger" node to the main input of the "APITemplate.io" node.

5. **Save and activate the workflow.**

6. **Test the workflow:**
   - Submit a test response to the Typeform form.
   - Confirm the PDF invoice is generated and downloaded with the submitted data filled in.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                             |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| The workflow screenshot provides a visual overview of node layout and connections.                         | Attached as `workflow-screenshot` file in the original data. |
| APITemplate.io template ID `96c77b2b1ab6ac88` must correspond to an invoice PDF template pre-configured in APITemplate.io account. | APITemplate.io documentation: https://apitemplate.io/docs    |
| Typeform form ID `dpr2kxSL` is specific to the form used; replacing it requires updating the form ID in the trigger node. | Typeform API docs: https://developer.typeform.com/           |
| The invoice data includes hardcoded static fields (invoice number, dates) which may be parameterized for dynamic use cases. | Consider adding dynamic generation for invoice numbers/dates.|

---

This completes the detailed, structured reference document for the workflow "Create an invoice based on the Typeform submission."