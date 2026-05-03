Generate AI-Powered Lease Renewal Offers with Ollama LLM, Supabase and Gmail

https://n8nworkflows.xyz/workflows/generate-ai-powered-lease-renewal-offers-with-ollama-llm--supabase-and-gmail-7201


# Generate AI-Powered Lease Renewal Offers with Ollama LLM, Supabase and Gmail

### 1. Workflow Overview

This workflow automates the generation and delivery of lease renewal offers using AI and cloud services. It targets property managers or landlords who want to streamline the process of creating personalized lease renewal letters, storing them, and emailing tenants efficiently. The workflow consists of three main logical blocks:

- **1.1 Input Reception and Customer Data Retrieval:** Receives customer input via a form, retrieves customer details from Supabase, and prepares landlord/company static data.

- **1.2 AI-Powered Offer Letter Generation and File Management:** Uses AI (Ollama LLM) to generate a formal lease renewal offer letter, converts the text to a file, manages duplicate files in Google Drive, and uploads the new offer letter file.

- **1.3 Email Composition and Sending:** Generates a professional email body referencing the generated letter, downloads the uploaded file, and sends the email with the file attached via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Customer Data Retrieval

- **Overview:**  
  This block handles form submission input, queries customer data from Supabase based on the provided customer ID, updates customer renewal info, and sets landlord/company contact details for subsequent use.

- **Nodes Involved:**  
  - On form submission  
  - Supabase-search_cust  
  - Supabase_cust_info  
  - Edit Fields  
  - Sticky Note (Get Customer Information)

- **Node Details:**

  1. **On form submission**  
     - Type: Form Trigger  
     - Role: Entry point capturing customer input such as Customer ID and Renewal Amount via a web form titled "Customer Renewal Form" with required numeric fields.  
     - Key Parameters: Form fields include "Customer id" and "Renewal Amount" (both required).  
     - Inputs: HTTP webhook trigger (webhookId provided)  
     - Outputs: Form data JSON  
     - Possible Failures: Form validation errors, webhook connectivity issues.

  2. **Supabase-search_cust**  
     - Type: Supabase node (database query)  
     - Role: Retrieves customer details from the "customer_details" table using the provided Customer ID.  
     - Parameters: Filter condition where `id` equals the submitted Customer ID.  
     - Inputs: Output from form submission node.  
     - Outputs: Customer record JSON including email, name, address, renewable_date, etc.  
     - Failures: Authentication errors, empty results if ID not found, query timeouts.

  3. **Supabase_cust_info**  
     - Type: Supabase node (update operation)  
     - Role: Updates the customer's renewable_date (set to current date/time) and Renewal Amount in Supabase.  
     - Parameters: Filter by `id` equal to the customer’s id; update fields "renewable_date" and "Renewal Amount" with current timestamp and form input respectively.  
     - Inputs: Customer data from Supabase-search_cust.  
     - Outputs: Updated customer record JSON.  
     - Failures: Auth errors, conflicts on update, invalid data.

  4. **Edit Fields**  
     - Type: Set node  
     - Role: Sets static landlord and company contact information fields into the workflow context for use in letter generation and email composition.  
     - Parameters: Sets landlord address, contact email, phone, company name, and landlord name as fixed JSON values.  
     - Inputs: Output from Supabase_cust_info.  
     - Outputs: JSON object enriched with static fields.  
     - Failures: Expression errors unlikely as static data used.

  5. **Sticky Note**  
     - Content: "## Get Customer Information"  
     - Role: Documentation block labeling this logical section for clarity.

---

#### 2.2 AI-Powered Offer Letter Generation and File Management

- **Overview:**  
  This block uses the Ollama large language model to generate a formal lease renewal letter based on customer and landlord data, converts the letter text into a file, checks for duplicate files on Google Drive, deletes duplicates if any, and uploads the new letter file.

- **Nodes Involved:**  
  - Ollama Chat Model  
  - Basic LLM Chain-offerLetter  
  - Convert to File  
  - Google Drive-search  
  - If-check_dup  
  - Google Drive-delete_dup  
  - Google Drive-upload  
  - Sticky Note1 (Generate offer letter)

- **Node Details:**

  1. **Ollama Chat Model**  
     - Type: Language Model Chat (Ollama)  
     - Role: AI engine providing language model capabilities feeding into offer letter and email generation chains.  
     - Parameters: Uses "llama3.1:latest" model at temperature 0.3 for controlled creativity.  
     - Credentials: Ollama API account linked.  
     - Inputs: None directly; triggers AI chains.  
     - Outputs: AI-generated textual responses.  
     - Failures: API key issues, timeout, rate limits, model errors.

  2. **Basic LLM Chain-offerLetter**  
     - Type: LangChain LLM Chain node  
     - Role: Generates the lease renewal offer letter text using an AI prompt that incorporates dynamic customer and landlord fields, including lease terms, dates, and rent amount.  
     - Parameters: Custom prompt template referencing multiple node data fields; outputs only the formatted letter text with no extraneous text.  
     - Inputs: AI model output from Ollama Chat Model and data from Edit Fields and Supabase_cust_info nodes.  
     - Outputs: Text of the offer letter.  
     - Failures: Expression errors in prompt, missing data, API failures.

  3. **Convert to File**  
     - Type: Convert to File  
     - Role: Converts the generated letter text into a text file suitable for storage and attachment.  
     - Parameters: Operation "toText" on property "text".  
     - Inputs: Text output from Basic LLM Chain-offerLetter.  
     - Outputs: Binary file data for upload.  
     - Failures: Conversion errors, large text handling issues.

  4. **Google Drive-search**  
     - Type: Google Drive node (file/folder search)  
     - Role: Searches a specific Google Drive folder ("OfferRenewal") for existing files matching the customer name and renewable date to detect duplicates.  
     - Parameters: Folder ID set, query string dynamically built from customer name and renewable date from Edit Fields.  
     - Inputs: Output from Edit Fields node.  
     - Outputs: List of matching files (if any).  
     - Failures: Auth errors, API quota exceeded, no matching files.

  5. **If-check_dup**  
     - Type: If condition node  
     - Role: Checks if any duplicate files were found (existence of file ID).  
     - Parameters: Condition "exists" on JSON field `id`.  
     - Inputs: Google Drive-search results.  
     - Outputs: Two branches: True (duplicates found), False (no duplicates).  
     - Failures: Logic errors if input malformed.

  6. **Google Drive-delete_dup**  
     - Type: Google Drive node (delete file)  
     - Role: Deletes any duplicate file found to ensure only the latest offer letter is stored.  
     - Parameters: Deletes file by ID found in search.  
     - Inputs: True branch from If-check_dup.  
     - Outputs: Confirmation of deletion (not used downstream).  
     - Failures: Permission errors, file locked, file not found.

  7. **Google Drive-upload**  
     - Type: Google Drive node (upload file)  
     - Role: Uploads the converted offer letter file to the "OfferRenewal" folder on Google Drive, naming it with customer name and renewable date.  
     - Parameters: Drive set to "My Drive", folder ID set for "OfferRenewal", file name dynamic.  
     - Inputs: Output from Convert to File node.  
     - Outputs: Metadata of uploaded file including file ID.  
     - Failures: Quota limits, permission errors.

  8. **Sticky Note1**  
     - Content: "## Generate offer letter"  
     - Role: Documentation label for this block.

---

#### 2.3 Email Composition and Sending

- **Overview:**  
  This final block composes a professional email referencing the offer letter, downloads the uploaded letter file from Google Drive, and sends the email with the letter attached to the customer.

- **Nodes Involved:**  
  - Basic LLM Chain-email  
  - Google Drive-get_file  
  - Gmail  
  - Sticky Note2 (Send Email)

- **Node Details:**

  1. **Basic LLM Chain-email**  
     - Type: LangChain LLM Chain node  
     - Role: Generates a concise, professional email to the client referring to the attached lease renewal offer letter, using AI based on the letter text.  
     - Parameters: Prompt enforces short business email format, no extra labels, starts with greeting, ends with signature. Input text is the offer letter text from Basic LLM Chain-offerLetter.  
     - Inputs: AI model output from Ollama and offer letter text.  
     - Outputs: Email body text.  
     - Failures: Expression or prompt errors, API failures.

  2. **Google Drive-get_file**  
     - Type: Google Drive node (file download)  
     - Role: Downloads the uploaded offer letter file from Google Drive to attach to the email.  
     - Parameters: Downloads by file ID from Google Drive-upload node output, binary data property named "down-data".  
     - Inputs: Output from Google Drive-upload.  
     - Outputs: Binary file data for attachment.  
     - Failures: File access errors, download failures.

  3. **Gmail**  
     - Type: Gmail node (send email)  
     - Role: Sends the composed email to the tenant’s email address with the offer letter attached.  
     - Parameters:  
       - To: Tenant email from Supabase-search_cust output.  
       - Subject: Dynamic "Offer Renewal [Customer Name] [Renewable Date (YYYY-MM)]".  
       - Message: Email body text from Basic LLM Chain-email.  
       - Attachment: Binary file data from Google Drive-get_file.  
       - Email type: Plain text.  
     - Credentials: OAuth2 Gmail account linked.  
     - Inputs: Email text and binary attachment.  
     - Outputs: Email send confirmation.  
     - Failures: Authentication, quota limits, invalid email address.

  4. **Sticky Note2**  
     - Content: "## Send Email"  
     - Role: Documentation label for this block.

---

### 3. Summary Table

| Node Name               | Node Type                               | Functional Role                       | Input Node(s)                     | Output Node(s)                  | Sticky Note                     |
|-------------------------|---------------------------------------|------------------------------------|----------------------------------|--------------------------------|--------------------------------|
| On form submission      | n8n-nodes-base.formTrigger             | Receive form input (Customer ID, Renewal Amount) | -                                | Supabase-search_cust            | ## Get Customer Information     |
| Supabase-search_cust     | n8n-nodes-base.supabase                | Fetch customer details from Supabase | On form submission               | Supabase_cust_info              | ## Get Customer Information     |
| Supabase_cust_info       | n8n-nodes-base.supabase                | Update customer renewal info in Supabase | Supabase-search_cust            | Edit Fields                    | ## Get Customer Information     |
| Edit Fields              | n8n-nodes-base.set                     | Set landlord and company static info | Supabase_cust_info               | Google Drive-search             | ## Get Customer Information     |
| Google Drive-search      | n8n-nodes-base.googleDrive             | Search for duplicate offer files in folder | Edit Fields                     | If-check_dup                   |                                |
| If-check_dup             | n8n-nodes-base.if                      | Check if duplicates exist           | Google Drive-search              | Google Drive-delete_dup (true), Basic LLM Chain-offerLetter (false) |                                |
| Google Drive-delete_dup  | n8n-nodes-base.googleDrive             | Delete duplicate files              | If-check_dup (true)              | Basic LLM Chain-offerLetter    |                                |
| Ollama Chat Model        | @n8n/n8n-nodes-langchain.lmChatOllama | Provide AI LLM model for text generation | -                              | Basic LLM Chain-offerLetter, Basic LLM Chain-email |                                |
| Basic LLM Chain-offerLetter | @n8n/n8n-nodes-langchain.chainLlm     | Generate lease renewal offer letter text | Ollama Chat Model, Edit Fields, Supabase_cust_info, Google Drive-delete_dup | Convert to File               | ## Generate offer letter        |
| Convert to File          | n8n-nodes-base.convertToFile           | Convert letter text to file binary  | Basic LLM Chain-offerLetter      | Google Drive-upload             | ## Generate offer letter        |
| Google Drive-upload      | n8n-nodes-base.googleDrive             | Upload offer letter file to Drive  | Convert to File                 | Basic LLM Chain-email           | ## Generate offer letter        |
| Basic LLM Chain-email    | @n8n/n8n-nodes-langchain.chainLlm     | Generate professional email content | Ollama Chat Model, Basic LLM Chain-offerLetter, Google Drive-upload | Google Drive-get_file          | ## Send Email                  |
| Google Drive-get_file    | n8n-nodes-base.googleDrive             | Download uploaded offer letter file | Basic LLM Chain-email           | Gmail                         | ## Send Email                  |
| Gmail                   | n8n-nodes-base.gmail                   | Send email to customer with attachment | Google Drive-get_file            | -                              | ## Send Email                  |
| Sticky Note              | n8n-nodes-base.stickyNote              | Documentation block                | -                                | -                              | ## Get Customer Information     |
| Sticky Note1             | n8n-nodes-base.stickyNote              | Documentation block                | -                                | -                              | ## Generate offer letter        |
| Sticky Note2             | n8n-nodes-base.stickyNote              | Documentation block                | -                                | -                              | ## Send Email                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form title: "Customer Renewal Form"  
   - Add two required number fields: "Customer id:" and "Renewal Amount:"  
   - Save webhook URL for form submissions.

2. **Add a Supabase node ("Supabase-search_cust")**  
   - Operation: Get (query)  
   - Table: customer_details  
   - Filter: `id` equals expression `{{$json["Customer id:"]}}` from form trigger output  
   - Connect input from "On form submission" node.  
   - Set Supabase credentials.

3. **Add a Supabase node ("Supabase_cust_info")**  
   - Operation: Update  
   - Table: customer_details  
   - Filter: `id` equals `{{$json.id}}` from previous output  
   - Fields to update:  
     - renewable_date = Current timestamp (`{{$now}}`)  
     - Renewal Amount = `{{$node["On form submission"].json["Renewal Amount:"]}}`  
   - Connect input from "Supabase-search_cust".  
   - Set Supabase credentials.

4. **Add a Set node ("Edit Fields")**  
   - Mode: Raw JSON  
   - Add static fields:  
     - landload_address: "Lakindu Siriwardana, Neuenlander Str. 28201 Bremen, Germany."  
     - contact_email: "lakithegreat99@gmail.com"  
     - contact_phone: "+491739XXXXXX"  
     - company_name: "HX GmbH"  
     - landload_name: "Lakindu Siriwardana"  
   - Connect input from "Supabase_cust_info".

5. **Add a Google Drive node ("Google Drive-search")**  
   - Operation: Search files/folders  
   - Folder ID: "1GGYne4bMaff9_1RvId2en38SPtmnInBA" (OfferRenewal folder)  
   - Query string: `{{$node["Edit Fields"].json.cust_name}}{{$node["Edit Fields"].json.renewable_date}}`  
   - Connect input from "Edit Fields".  
   - Set Google Drive OAuth2 credentials.

6. **Add an If node ("If-check_dup")**  
   - Condition: Check if `{{$json.id}}` exists (non-empty)  
   - Connect input from "Google Drive-search".

7. **Add a Google Drive node ("Google Drive-delete_dup")**  
   - Operation: Delete file  
   - File ID: `{{$json.id}}` from "If-check_dup" true branch  
   - Connect input from "If-check_dup" true output.  
   - Set credentials.

8. **Add an Ollama Chat Model node ("Ollama Chat Model")**  
   - Model: "llama3.1:latest"  
   - Temperature: 0.3  
   - Set Ollama API credentials.

9. **Add a LangChain LLM Chain node ("Basic LLM Chain-offerLetter")**  
   - Text prompt: Use template including landlord address, current date, customer name, address, lease terms, renewal amount, contact details, etc., referencing data from "Edit Fields", "Supabase_cust_info", and form input.  
   - Connect AI input from "Ollama Chat Model".  
   - Connect input from "Google Drive-delete_dup" (false branch) and "Ollama Chat Model" (ai_languageModel output).

10. **Connect "If-check_dup" false output to "Basic LLM Chain-offerLetter"**  
    - So either after deleting duplicates or if none found, generate letter.

11. **Add a Convert to File node ("Convert to File")**  
    - Operation: To Text  
    - Source property: "text" (output of offer letter chain)  
    - Connect input from "Basic LLM Chain-offerLetter".

12. **Add a Google Drive Upload node ("Google Drive-upload")**  
    - Operation: Upload file  
    - Drive: "My Drive"  
    - Folder ID: same as search folder ID  
    - File name: `{{$node["Edit Fields"].json.cust_name}}{{$node["Edit Fields"].json.renewable_date}}`  
    - Connect input from "Convert to File".  
    - Set credentials.

13. **Add a LangChain LLM Chain node ("Basic LLM Chain-email")**  
    - Text prompt: Compose short, professional email referencing the letter, instructing no extra text, starting with greeting and ending with signature, using offer letter text as input.  
    - Connect AI input from "Ollama Chat Model".  
    - Connect input from "Basic LLM Chain-offerLetter" and "Google Drive-upload" outputs.

14. **Add a Google Drive node ("Google Drive-get_file")**  
    - Operation: Download file  
    - File ID: `{{$node["Google Drive-upload"].json.id}}`  
    - Binary property: "down-data"  
    - Connect input from "Basic LLM Chain-email".  
    - Set credentials.

15. **Add a Gmail node ("Gmail")**  
    - Operation: Send email  
    - To: `{{$node["Supabase-search_cust"].json.cust_email}}`  
    - Subject: "Offer Renewal {{$node["Supabase_cust_info"].json.cust_name}} {{$node["Supabase-search_cust"].json.renewable_date.toString().slice(0,7)}}"  
    - Message: Email body from "Basic LLM Chain-email"  
    - Attachments: binary "down-data" from "Google Drive-get_file"  
    - Email type: Plain text  
    - Set Gmail OAuth2 credentials.  
    - Connect input from "Google Drive-get_file".

16. **Add Sticky Notes**  
    - Place appropriately for each block with content:  
      - "## Get Customer Information" near input nodes  
      - "## Generate offer letter" near AI and file nodes  
      - "## Send Email" near email nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow uses Ollama LLM (llama3.1:latest) for AI text generation.                                           | Ollama API credentials required.                                                                             |
| Google Drive folder with ID "1GGYne4bMaff9_1RvId2en38SPtmnInBA" is used to store lease offer letters.           | Ensure folder exists and OAuth2 credentials have write/delete permissions.                                   |
| Gmail node requires OAuth2 credentials with send email permission for the configured account.                    | Gmail API quota and security policies apply.                                                                 |
| The workflow updates Supabase "customer_details" table; ensure correct schema with fields: id, renewable_date, etc.| Supabase credentials and permissions must allow read/write on this table.                                    |
| Expressions in prompt templates rely on date manipulation and string splitting for address formatting.            | Be cautious of null or malformed data, which may cause expression failures.                                  |
| Duplicate file detection is based on file name combined with customer name and renewable date.                    | Naming collisions may cause unintended deletions; consider adding unique suffixes if needed.                 |
| Form Trigger webhook URL must be exposed and secured appropriately to avoid unauthorized submissions.             | Use HTTPS and authentication as needed.                                                                     |

---

**Disclaimer:** The provided text is exclusively from an n8n automated workflow, respecting content policies and