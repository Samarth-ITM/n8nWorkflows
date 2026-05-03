Automatically issue training certificates and send via Gmail

https://n8nworkflows.xyz/workflows/automatically-issue-training-certificates-and-send-via-gmail-3114


# Automatically issue training certificates and send via Gmail

### 1. Workflow Overview

This n8n workflow automates the generation and delivery of student completion certificates for a training program. It is designed primarily for organizations running training sessions who want to efficiently issue personalized certificates of completion and email them directly to students.

The main logical blocks and their purposes are:

- **1.1 Trigger Input**: A manual trigger to start the workflow for testing or individual certificate issuance.
- **1.2 Student Data Retrieval**: Fetches student data (name, email) from the n8n Training Customer Datastore.
- **1.3 Data Preparation**: Extracts and sets the student’s name and email, then generates a unique UUID to uniquely identify the certificate.
- **1.4 Certificate Image Preparation**: Loads the certificate template image, then sequentially writes the student’s name and UUID onto the image.
- **1.5 Certificate Delivery**: Sends the personalized image certificate attached via Gmail to the student’s email.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger Input

**Overview:**  
This block initiates the workflow manually, enabling users to test or generate certificates on demand.

**Nodes Involved:**  
- When clicking ‘Test workflow’

**Node Details:**

- **Node:** When clicking ‘Test workflow’  
  - Type: Manual Trigger  
  - Role: Entry point to manually start workflow execution for testing or ad hoc use.  
  - Configuration: Default manual trigger, no parameter required.  
  - Inputs: None (start node)  
  - Outputs: Connects to Customer Datastore (n8n training) node  
  - Version-specific: Standard n8n manual trigger behavior  
  - Edge Cases: None significant; manual trigger fails if user does not trigger or system interrupted.  

---

#### 2.2 Student Data Retrieval

**Overview:**  
Retrieves relevant student data such as name and email for certificate personalization.

**Nodes Involved:**  
- Customer Datastore (n8n training)  
- Get Email & Name

**Node Details:**

- **Node:** Customer Datastore (n8n training)  
  - Type: n8n-Inhouse Training Customer Datastore node  
  - Role: Retrieves student data records from the n8n training customer datastore  
  - Configuration: Default configuration—fetches data without filters (can be adjusted in real usage)  
  - Input: From manual trigger node  
  - Output: Student data passed to “Get Email & Name” node  
  - Version-specific: Requires datastore integration setup to n8n Training Customer Datastore  
  - Edge Cases: Possible failures include datastore connectivity issues or empty result sets  

- **Node:** Get Email & Name  
  - Type: Set node  
  - Role: Extracts and defines two key fields: student’s name and email, simplifying downstream use  
  - Configuration: Sets variables/parameters to store “name” and “email” fields extracted from datastore result  
  - Input: Student data from Customer Datastore node  
  - Output: Data enriched with extracted name and email, passed to UUID generation  
  - Version-specific: Uses Set node v3.4 features  
  - Edge Cases: Missing or malformed student email/name fields require validation to prevent errors downstream  

---

#### 2.3 Data Preparation

**Overview:**  
Generates a universally unique identifier (UUID) for each certificate to uniquely associate certificates with students.

**Nodes Involved:**  
- Generate Crypto (UUID Generation)

**Node Details:**

- **Node:** Generate Crypto  
  - Type: Crypto node  
  - Role: Generates a UUID (universally unique identifier) used as a certificate ID  
  - Configuration: Set to output UUID format  
  - Input: Student info with name/email from previous node  
  - Output: Appends UUID to the data object passed along  
  - Version-specific: Standard crypto node; ensure n8n supports UUID generation in version used  
  - Edge Cases: Failures can happen if crypto module errors or unsupported version; retry or error handling recommended  

---

#### 2.4 Certificate Image Preparation

**Overview:**  
Loads the certificate template image and sequentially annotates it with the student’s name and unique UUID.

**Nodes Involved:**  
- Load Image  
- Get Info  
- Write Text(name)  
- Write Text(uuid)

**Node Details:**

- **Node:** Load Image  
  - Type: HTTP Request node  
  - Role: Downloads certificate template image from a URL or server  
  - Configuration: Configured with the URL to the certificate template image  
  - Input: UUID and student data from crypto node  
  - Output: Image binary data passed to editing nodes  
  - Version-specific: HTTP Request node v4.2 supports binary data for images  
  - Edge Cases: Timeout, invalid URL, or inaccessible image resource cause workflow failures  

- **Node:** Get Info  
  - Type: Edit Image node  
  - Role: Loads or prepares the image for editing operations such as adding text  
  - Configuration: Defaults or specifically set to handle the binary image input from Load Image  
  - Input: Image data from Load Image node  
  - Output: Prepared image passed to writing text nodes  
  - Version-specific: Edit Image node with error continuation enabled  
  - Edge Cases: Unsupported image format or corrupted input  

- **Node:** Write Text(name)  
  - Type: Edit Image node  
  - Role: Writes the student’s name onto the certificate template at a configured position  
  - Configuration: Configured with font, size, color (likely Courier New or Comic Sans MS as per prerequisites), coordinates for name placement  
  - Input: Image from Get Info node  
  - Output: Image with name written, sent to Write Text(uuid) node  
  - Version-specific: Use fonts installed (Courier New, Comic Sans MS) and n8n version that supports editImage node  
  - Edge Cases: Expression errors if name variable missing or font not found  

- **Node:** Write Text(uuid)  
  - Type: Edit Image node  
  - Role: Writes the unique UUID onto the certificate at a designated position, ideally separate from the name  
  - Configuration: Similar to Write Text(name), tailored for UUID text styling and position  
  - Input: Image with name from previous node  
  - Output: Final annotated image passed to email sending node  
  - Version-specific: Same as Write Text(name)  
  - Edge Cases: Missing UUID variable or output image corruption  

---

#### 2.5 Certificate Delivery

**Overview:**  
Sends the completed personalized certificate via Gmail as an email attachment.

**Nodes Involved:**  
- Send Email

**Node Details:**

- **Node:** Send Email  
  - Type: Gmail node  
  - Role: Sends an email with the generated certificate attached to the student’s email address  
  - Configuration:  
    - Uses Gmail OAuth2 credential for authentication  
    - To: student’s email from dataset  
    - Subject and Body: Customized with placeholders for name and completion message  
    - Attachment: the final image binary output from the last Edit Image node  
  - Input: Completed certificate image with metadata from Write Text(uuid) node  
  - Output: Email sent; workflow ends here  
  - Version-specific: Gmail node v2.1 with OAuth2 support  
  - Edge Cases: Authentication errors with Gmail OAuth2, attachment size limits, invalid recipient email resulting in send failure  

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                           | Input Node(s)                 | Output Node(s)             | Sticky Note                      |
|----------------------------|----------------------------------|-----------------------------------------|------------------------------|----------------------------|---------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts workflow manually for testing    | None                         | Customer Datastore (n8n training) |                                 |
| Customer Datastore (n8n training) | n8n Training Customer Datastore | Retrieves student data                   | When clicking ‘Test workflow’ | Get Email & Name            |                                 |
| Get Email & Name            | Set                              | Extracts student name and email          | Customer Datastore (n8n training) | Generate Crypto            |                                 |
| Generate Crypto             | Crypto                           | Generates unique certificate UUID        | Get Email & Name             | Load Image                  |                                 |
| Load Image                 | HTTP Request                      | Loads certificate template image         | Generate Crypto              | Get Info                   |                                 |
| Get Info                   | Edit Image                       | Prepares image for editing                | Load Image                   | Write Text(name)            |                                 |
| Write Text(name)           | Edit Image                       | Inserts student name into certificate     | Get Info                    | Write Text(uuid)            |                                 |
| Write Text(uuid)           | Edit Image                       | Inserts certificate UUID into image       | Write Text(name)             | Send Email                  |                                 |
| Send Email                 | Gmail                            | Sends certificate email to student       | Write Text(uuid)             | None                       |                                 |
| Sticky Note2               | Sticky Note                      |                                         |                              |                            |                                 |
| Sticky Note8               | Sticky Note                      |                                         |                              |                            |                                 |
| Sticky Note9               | Sticky Note                      |                                         |                              |                            |                                 |
| Sticky Note10              | Sticky Note                      |                                         |                              |                            |                                 |
| Sticky Note11              | Sticky Note                      |                                         |                              |                            |                                 |
| Sticky Note12              | Sticky Note                      |                                         |                              |                            |                                 |
| Sticky Note13              | Sticky Note                      |                                         |                              |                            |                                 |
| Sticky Note                | Sticky Note                      |                                         |                              |                            |                                 |
| Sticky Note1               | Sticky Note                      |                                         |                              |                            |                                 |

*Note:* Sticky notes are present but contain no content, hence no additional comments applied.

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a “Manual Trigger” node named `When clicking ‘Test workflow’`.

2. **Add Customer Datastore Node**  
   - Add “n8n Training Customer Datastore” node named `Customer Datastore (n8n training)`.  
   - No special parameters configured by default.  
   - Connect output of manual trigger node to this node.

3. **Add Set Node to Extract Email and Name**  
   - Add a “Set” node named `Get Email & Name`.  
   - Configure to extract and set fields:  
     - `name` (from customer datastore result)  
     - `email` (from customer datastore result)  
   - Connect output of Customer Datastore node to this node.

4. **Add Crypto Node for UUID Generation**  
   - Add a “Crypto” node named `Generate Crypto`.  
   - Set operation to generate a UUID.  
   - Connect output of Set node (Get Email & Name) to this node.

5. **Add HTTP Request Node to Load Image**  
   - Add “HTTP Request” node named `Load Image`.  
   - Configure:  
     - HTTP Method: GET  
     - URL: specify the URL of the certificate template image.  
     - Response Format: File or Binary data (to get image as binary).  
   - Connect output of Crypto node to this node.

6. **Add Edit Image Node to Prepare Image**  
   - Add “Edit Image” node named `Get Info`.  
   - Configure to accept binary image data from previous node.  
   - Connect output of Load Image node to this node.

7. **Add Edit Image Node to Write Student Name**  
   - Add “Edit Image” node named `Write Text(name)`.  
   - Configure:  
     - Text: Use expression to set student’s name (e.g. `{{$json["name"]}}`).  
     - Font: Courier New or Comic Sans MS (ensure installed).  
     - Font size, color, and position according to certificate design.  
   - Connect output of Get Info node to this node.

8. **Add Edit Image Node to Write UUID**  
   - Add “Edit Image” node named `Write Text(uuid)`.  
   - Configure:  
     - Text: The generated UUID (`{{$json["uuid"]}}`).  
     - Font, size, color, position distinct from name label.  
   - Connect output of Write Text(name) node to this node.

9. **Add Gmail Node to Send Email**  
   - Add “Gmail” node named `Send Email`.  
   - Configure:  
     - Set OAuth2 Gmail credentials with proper access.  
     - To: Use expression to set recipient as student email (`{{$json["email"]}}`).  
     - Subject: e.g., “Your Training Completion Certificate”  
     - Text Body: Compose message including student’s name and completion confirmation with date.  
     - Attachments: Attach the final certificate image from Write Text(uuid) node binary data.  
   - Connect output of Write Text(uuid) node to this node.

10. **Finalize Connections**  
    - Ensure connections flow exactly as:  
      Manual Trigger -> Customer Datastore -> Get Email & Name -> Generate Crypto -> Load Image -> Get Info -> Write Text(name) -> Write Text(uuid) -> Send Email

11. **Credential Preparation**  
    - Set up Gmail OAuth2 credentials under n8n credentials.  
    - Confirm n8n Training Customer Datastore access configured.  
    - Ensure Google Fonts (Courier New and Comic Sans MS) are installed on worker or server for font rendering.

12. **Test Workflow**  
    - Trigger workflow manually using the manual trigger node.  
    - Confirm email receipt of personalized certificate with correct name and UUID.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Google Fonts Courier New and Comic Sans MS must be installed on the host running n8n for proper font rendering.      | Font installation prerequisite for Edit Image nodes                                          |
| Gmail node requires OAuth2 credentials configured in n8n for sending emails securely.                                | Gmail OAuth2 setup in n8n credentials                                                         |
| Ensure the certificate template image URL is accessible and returns a proper image file for editing.                  | URL configuration in HTTP Request node                                                        |
| Example email body text:                                                                                             | Embedded in Gmail node parameter configuration                                                |
| ```Dear John Doe,                                                                                                    |                                                                                                |
| You have successfully completed this training program. Please find your completion certificate attached.            |                                                                                                |
| Completion Date: 2025.02.22                                                                                          |                                                                                                |
| Best regards,                                                                                                        |                                                                                                |
| Data Popcorn Team                                                                                                    |                                                                                                |
| ```                                                                                                                  |                                                                                                |

---

This documentation provides a detailed overview, stepwise node analysis, and complete reproduction instructions suitable for both advanced n8n users and AI-based automation agents.