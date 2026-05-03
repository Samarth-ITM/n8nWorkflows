Process Contact Form Submissions with Validation and MongoDB Storage

https://n8nworkflows.xyz/workflows/process-contact-form-submissions-with-validation-and-mongodb-storage-8071


# Process Contact Form Submissions with Validation and MongoDB Storage

---

### 1. Workflow Overview

This n8n workflow, titled **"Contact Form Workflow with Data Validation and MongoDB Integration,"** is designed to securely process contact form submissions. It targets use cases where user-submitted contact data must be validated for correctness and security, consistently formatted, and reliably stored in a MongoDB Atlas database.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Captures user inputs from a web-based contact form with fields for Name, Last Name, Email, and Phone Number.
- **1.2 Validation Layer:** Performs data validation including format checking (especially for phone numbers) and security checks to prevent injection attacks by filtering dangerous patterns.
- **1.3 Data Formatting:** Converts validated input field names into snake_case to maintain a consistent naming convention for database storage.
- **1.4 Database Insertion:** Inserts the sanitized and formatted contact data into a MongoDB collection.
- **1.5 Workflow Completion:** Sends a confirmation message back to the user indicating successful submission.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Acts as the workflow’s entry point. It collects user-submitted contact details via a form trigger node configured with required fields.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  - **On form submission**  
    - **Type:** Form Trigger  
    - **Technical Role:** Entry point that listens for contact form submissions via a webhook  
    - **Configuration:**  
      - Webhook enabled with ID "a87e98a4-aa64-4939-b6da-10213d7f7d57"  
      - Form titled "📩 Get in Touch with Us"  
      - Fields: Name, Last Name, Email (email type), Phone Number — all required  
      - Bot submissions ignored, no attribution appended  
      - Form description guides user interaction  
    - **Inputs:** HTTP request with form data  
    - **Outputs:** JSON object containing user-submitted data with keys as per form fields  
    - **Edge Cases / Failures:**  
      - Missing required fields leads to no submission (handled by frontend)  
      - Webhook downtime or connectivity issues may cause data loss  
      - Bots ignored via configuration, but sophisticated bots may bypass  
    - **Version Requirements:** n8n version supporting formTrigger v2.3  

- **Sticky Notes:**  
  - Sticky Note1 explains this block: acts as entry point, lists form fields collected.

---

#### 2.2 Validation Layer

- **Overview:**  
  Validates the form input to ensure data integrity and security. It validates phone number format and screens text fields for potentially malicious patterns such as SQL injection or script tags.

- **Nodes Involved:**  
  - Validate Pattern

- **Node Details:**  

  - **Validate Pattern**  
    - **Type:** Code Node (JavaScript)  
    - **Technical Role:** Custom JS code execution to validate and flag invalid inputs  
    - **Configuration:**  
      - Validates "Phone Number" against regex for international and local phone formats  
      - Applies regex pattern to detect dangerous characters or keywords (e.g., SQL commands, script tags) in Name, Last Name, and Email fields  
      - If invalid, replaces the field value with the string "Is not Valid"  
    - **Inputs:** JSON from form submission node  
    - **Outputs:** JSON with validated and possibly sanitized fields  
    - **Key Expressions:**  
      - Phone regex: `/^(\+?\d{1,4}[\s-]?)?(\(?\d{1,4}\)?[\s-]?)?[\d\s-]{5,}$/`  
      - Dangerous pattern regex: `/('|;|--|\/\*|\*\/|xp_|exec|drop|select|insert|delete|update|union|script|<|>)/i`  
    - **Edge Cases / Failures:**  
      - Misformatted phone numbers flagged but not rejected (field overwritten)  
      - Malicious inputs replaced but workflow continues; no outright rejection or error thrown  
      - Expression errors if input fields missing or malformed JSON (unlikely)  
    - **Version Requirements:** Code node v2  

- **Sticky Notes:**  
  - Sticky Note2 details the validation logic and security rationale.

---

#### 2.3 Data Formatting

- **Overview:**  
  Converts validated fields into a snake_case naming convention consistent with MongoDB schema expectations. Adds a timestamp field for submission time.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**  

  - **Edit Fields**  
    - **Type:** Set Node  
    - **Technical Role:** Renames and assigns fields for MongoDB insertion  
    - **Configuration:**  
      - Maps:  
        - "Name" → "name"  
        - "Last Name" → "last_name"  
        - "Email" → "email"  
        - "Phone Number" → "phone"  
      - Adds "submitted_at" with timestamp from `$json.submittedAt`  
    - **Inputs:** Validated JSON from the code node  
    - **Outputs:** JSON ready for database insertion with consistent field names  
    - **Edge Cases / Failures:**  
      - Missing input fields would result in empty strings or null values if not handled upstream  
      - Timestamp must be present; otherwise, "submitted_at" may be empty or invalid  
    - **Version Requirements:** Set node v3.4  

- **Sticky Notes:**  
  - Sticky Note3 describes the purpose of field renaming and formatting.

---

#### 2.4 Database Insertion

- **Overview:**  
  Stores the formatted contact data securely into a MongoDB Atlas collection named "contact."

- **Nodes Involved:**  
  - Insert documents

- **Node Details:**  

  - **Insert documents**  
    - **Type:** MongoDB Node  
    - **Technical Role:** Inserts a single document into MongoDB collection  
    - **Configuration:**  
      - Operation: insert  
      - Collection: "contact"  
      - Fields explicitly listed: name, last_name, email, phone, submitted_at  
      - Uses dot notation and dateFields options (though dateFields configured with all fields, likely an oversight, but non-critical)  
    - **Credentials:** MongoDB account connected via secure credentials (MongoDB Atlas)  
    - **Inputs:** JSON from Edit Fields node  
    - **Outputs:** Confirmation of insertion operation (inserted document metadata)  
    - **Edge Cases / Failures:**  
      - MongoDB connectivity issues (network, auth failures)  
      - Invalid data types causing insertion errors  
      - Duplicate entries not checked or prevented here  
    - **Version Requirements:** MongoDB node v1.2  

- **Sticky Notes:**  
  - Sticky Note4 explains the database persistence role.

---

#### 2.5 Workflow Completion

- **Overview:**  
  Sends a completion response to the user confirming successful form submission and storage.

- **Nodes Involved:**  
  - Form Ending

- **Node Details:**  

  - **Form Ending**  
    - **Type:** Form Node (completion)  
    - **Technical Role:** Returns a success message to the frontend form UI  
    - **Configuration:**  
      - Operation: completion  
      - Completion Title: "✅ Message Sent Successfully"  
      - Completion Message: "Thank you for reaching out! Your message has been received, and our team will get back to you shortly. We appreciate your interest and look forward to connecting with you."  
    - **Inputs:** Confirmation from MongoDB insertion node  
    - **Outputs:** HTTP response to user’s browser or form client  
    - **Edge Cases / Failures:**  
      - If upstream nodes fail, this may not be reached, leaving user without feedback  
      - Network or webhook issues may prevent delivery of confirmation  
    - **Version Requirements:** Form node v2.3  

- **Sticky Notes:**  
  - Sticky Note5 clarifies the role of this final step.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role              | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                                   |
|---------------------|-------------------|-----------------------------|------------------------|-----------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger      | Entry point, form data input| -                      | Validate Pattern       | ### 1. On Form Submission: entry point capturing Name, Last Name, Email, Phone Number                         |
| Validate Pattern     | Code              | Validation and sanitization | On form submission     | Edit Fields           | ### 2. Validation Layer: input validation, security checks, phone number format, flags invalid data          |
| Edit Fields         | Set               | Data formatting, renaming   | Validate Pattern       | Insert documents       | ### 3. Edit Fields Node: converts fields to snake_case for MongoDB                                           |
| Insert documents     | MongoDB           | Data persistence            | Edit Fields            | Form Ending            | ### 4. MongoDB Node: inserts validated and formatted data into MongoDB Atlas collection                      |
| Form Ending          | Form              | Completion confirmation     | Insert documents       | -                      | ### 5. Form Ending Node: confirms successful submission to user                                             |
| Sticky Note          | Sticky Note       | Documentation               | -                      | -                      | # n8n Workflow: Contact Form Workflow with Data Validation and MongoDB Integration (author & description)    |
| Sticky Note1         | Sticky Note       | Documentation               | -                      | -                      | ### 1. On Form Submission: entry point and form fields                                                      |
| Sticky Note2         | Sticky Note       | Documentation               | -                      | -                      | ### 2. Code Node (Validation Layer): validates input and prevents injection                                |
| Sticky Note3         | Sticky Note       | Documentation               | -                      | -                      | ### 3. Edit Fields Node (Data Formatting): snake_case renaming                                              |
| Sticky Note4         | Sticky Note       | Documentation               | -                      | -                      | ### 4. MongoDB Node (Insert Documents): database persistence                                                |
| Sticky Note5         | Sticky Note       | Documentation               | -                      | -                      | ### 5. Form Ending Node: confirmation message                                                               |
| Sticky Note8         | Sticky Note       | Documentation (block number)| -                      | -                      | ## 1                                                                                                        |
| Sticky Note9         | Sticky Note       | Documentation (block number)| -                      | -                      | ## 2                                                                                                        |
| Sticky Note11        | Sticky Note       | Documentation (block number)| -                      | -                      | ## 3                                                                                                        |
| Sticky Note10        | Sticky Note       | Documentation (block number)| -                      | -                      | ## 4                                                                                                        |
| Sticky Note12        | Sticky Note       | Documentation (block number)| -                      | -                      | ## 5                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" Node**  
   - Type: Form Trigger (v2.3)  
   - Configure webhook with unique ID (auto-generated)  
   - Form title: "📩 Get in Touch with Us"  
   - Add required fields:  
     - Name (text)  
     - Last Name (text)  
     - Email (email type)  
     - Phone Number (text)  
   - Enable "Ignore Bots" option  
   - Set form description accordingly  

2. **Create "Validate Pattern" Node**  
   - Type: Code Node (JavaScript, v2)  
   - Paste the validation JS code:  
     - Validate "Phone Number" with regex `/^(\+?\d{1,4}[\s-]?)?(\(?\d{1,4}\)?[\s-]?)?[\d\s-]{5,}$/`  
     - Validate "Name", "Last Name", and "Email" for dangerous patterns (SQL injection, scripting)  
     - Replace invalid fields with "Is not Valid"  
   - Connect "On form submission" node output to this node input  

3. **Create "Edit Fields" Node**  
   - Type: Set Node (v3.4)  
   - Create assignments for output fields:  
     - name = `{{$json.Name}}`  
     - last_name = `{{$json["Last Name"]}}`  
     - email = `{{$json.Email}}`  
     - phone = `{{$json["Phone Number"]}}`  
     - submitted_at = `{{$json.submittedAt}}` (ensure timestamp is passed or set in form trigger)  
   - Connect "Validate Pattern" output to this node input  

4. **Create "Insert documents" Node**  
   - Type: MongoDB (v1.2)  
   - Operation: Insert  
   - Collection: "contact"  
   - Fields: name, last_name, email, phone, submitted_at  
   - Enable "Use Dot Notation"  
   - (Optional) Configure "Date Fields" if needed  
   - Connect "Edit Fields" output to this node input  
   - Set up MongoDB credentials using MongoDB Atlas connection string  

5. **Create "Form Ending" Node**  
   - Type: Form Node (v2.3)  
   - Operation: Completion  
   - Completion Title: "✅ Message Sent Successfully"  
   - Completion Message: "Thank you for reaching out! Your message has been received, and our team will get back to you shortly. We appreciate your interest and look forward to connecting with you."  
   - Connect "Insert documents" output to this node input  

6. **Connect all nodes sequentially:**  
   - On form submission → Validate Pattern → Edit Fields → Insert documents → Form Ending  

7. **Test the workflow** by submitting data via the form URL generated by the form trigger node webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Author: Samuel Heredia - LinkedIn profile                                                                        | https://www.linkedin.com/in/samuel-heredia-2b5b7a98/                                                    |
| Workflow purpose: Securely handle contact form submissions with validation and MongoDB storage                   | N/A                                                                                                     |
| Regex patterns used for phone number validation and injection prevention                                         | Embedded in "Validate Pattern" code node                                                                |
| MongoDB Atlas recommended for secure and scalable database storage                                              | Credentials must be securely configured in n8n credentials                                               |
| Form trigger and form nodes support multi-version functionality; ensure n8n version compatibility               | Form trigger v2.3 and form node v2.3 used                                                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.

---