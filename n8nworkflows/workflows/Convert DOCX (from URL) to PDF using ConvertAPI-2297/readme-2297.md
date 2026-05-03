Convert DOCX (from URL) to PDF using ConvertAPI

https://n8nworkflows.xyz/workflows/convert-docx--from-url--to-pdf-using-convertapi-2297


# Convert DOCX (from URL) to PDF using ConvertAPI

### 1. Workflow Overview

This workflow enables developers and organizations to convert DOCX files hosted at a URL into PDF format using the ConvertAPI service. It addresses the common problem of file format conversion by automating the download, conversion, and storage of documents.

The workflow is organized into three main logical blocks:

- **1.1 Input Preparation & Trigger:** Receives manual trigger input and sets up the DOCX file URL.
- **1.2 Conversion via ConvertAPI:** Sends the DOCX file URL to ConvertAPI to perform the DOCX-to-PDF conversion.
- **1.3 Output Storage:** Saves the resulting PDF file to the local file system.

Supporting these blocks are configuration and documentation notes that guide the user on authentication and customization.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Preparation & Trigger

**Overview:**  
This block initiates the workflow via manual trigger and defines the DOCX file source URL to be converted.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Config  
- Sticky Note1  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point; starts the workflow on user command.  
  - *Configuration:* Default, no parameters.  
  - *Inputs:* None  
  - *Outputs:* Connects to `Config` node  
  - *Edge Cases:* None specific; user must manually trigger to start workflow.

- **Config**  
  - *Type:* Set  
  - *Role:* Defines the key variable `url_to_file` that points to the DOCX file URL.  
  - *Configuration:* Sets `url_to_file` string to `https://cdn.convertapi.com/cara/testfiles/document.docx` (default example URL).  
  - *Inputs:* From manual trigger  
  - *Outputs:* Connects to `HTTP Request` node  
  - *Edge Cases:* If the URL is invalid or unreachable, downstream HTTP Request will fail.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Provides instructions to change the `url_to_file` parameter to the user’s target file URL.  
  - *Inputs/Outputs:* None  
  - *Sticky Note Content:*  
    ```
    ## Configuration 
    Change the `url_to_file` parameter here to the file you want to convert
    ```

#### 2.2 Conversion via ConvertAPI

**Overview:**  
This block sends the DOCX file URL to the ConvertAPI endpoint to convert it into PDF format.

**Nodes Involved:**  
- HTTP Request  
- Sticky Note  

**Node Details:**  

- **HTTP Request**  
  - *Type:* HTTP Request (version 4.2)  
  - *Role:* Performs the POST request to ConvertAPI to convert DOCX to PDF.  
  - *Configuration:*  
    - URL: `https://v2.convertapi.com/convert/docx/to/pdf`  
    - Method: POST  
    - Authentication: HTTP Query Authentication using a credential named `Query Auth account` containing the secret key.  
    - Content Type: multipart/form-data  
    - Body Parameters: Sends a single parameter named `file` with value `={{ $json.url_to_file }}` dynamically referencing the URL from the previous node.  
    - Headers: Sets `Accept` header to `application/octet-stream` to receive the PDF as a binary stream.  
    - Response: Configured to interpret the response as a file (binary data).  
  - *Inputs:* Receives JSON with `url_to_file`  
  - *Outputs:* Binary PDF file data sent to `Read/Write Files from Disk` node  
  - *Edge Cases:*  
    - Authentication failure if secret is invalid or missing.  
    - Network or timeout errors.  
    - Invalid file URL or unsupported file format errors returned by ConvertAPI.  
    - Incorrect body parameter formatting.  
  - *Version Requirements:* Uses HTTP Request node version 4.2 for advanced authentication and file response handling.

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Provides authentication instructions and a link to create the required ConvertAPI account and secret.  
  - *Inputs/Outputs:* None  
  - *Sticky Note Content:*  
    ```
    ## Authentication
    Conversion requests must be authenticated. Please create 
    [ConvertAPI account to get authentication secret](https://www.convertapi.com/a/signin)

    Create a query auth credential with `secret` as name and your secret from the convertAPI dashboard as value
    ```

#### 2.3 Output Storage

**Overview:**  
This block saves the PDF file received from ConvertAPI to the local file system as `document.pdf`.

**Nodes Involved:**  
- Read/Write Files from Disk  

**Node Details:**  

- **Read/Write Files from Disk**  
  - *Type:* Read/Write File (version 1)  
  - *Role:* Writes the binary PDF data to a file on disk.  
  - *Configuration:*  
    - Operation: Write  
    - File Name: `document.pdf` (fixed)  
    - Data Property Name: `=data` (expects binary data property named `data` from input)  
  - *Inputs:* Receives binary PDF data from `HTTP Request` node  
  - *Outputs:* None (end of chain)  
  - *Edge Cases:*  
    - File write permission errors.  
    - Insufficient disk space.  
    - If received data is not properly binary PDF, file may be corrupted.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                    | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                      |
|----------------------------|-------------------------|----------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Starts workflow manually          | None                         | Config                      |                                                                                                                  |
| Config                     | Set                     | Sets DOCX file URL for conversion | When clicking ‘Test workflow’ | HTTP Request                | Change the `url_to_file` parameter here to the file you want to convert                                          |
| HTTP Request               | HTTP Request            | Sends DOCX URL to ConvertAPI to convert to PDF | Config                       | Read/Write Files from Disk  | Conversion requests must be authenticated. Please create [ConvertAPI account to get authentication secret](https://www.convertapi.com/a/signin). Create a query auth credential with `secret` as name and your secret from the convertAPI dashboard as value |
| Read/Write Files from Disk | Read/Write File         | Saves the PDF file locally        | HTTP Request                 | None                        |                                                                                                                  |
| Sticky Note1               | Sticky Note             | Instruction to update input file URL | None                         | None                        | Change the `url_to_file` parameter here to the file you want to convert                                          |
| Sticky Note                | Sticky Note             | Authentication instructions       | None                         | None                        | Conversion requests must be authenticated. Please create [ConvertAPI account to get authentication secret](https://www.convertapi.com/a/signin). Create a query auth credential with `secret` as name and your secret from the convertAPI dashboard as value |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Add a `Manual Trigger` node called `When clicking ‘Test workflow’`. No configuration needed.

2. **Add Set node named `Config`**  
   - Connect output of `When clicking ‘Test workflow’` to this node.  
   - Configure one string parameter named `url_to_file` with default value: `https://cdn.convertapi.com/cara/testfiles/document.docx`.  
   - This represents the DOCX file URL to convert.

3. **Add HTTP Request node named `HTTP Request`**  
   - Connect output of `Config` to this node.  
   - Set the following parameters:  
     - HTTP Method: POST  
     - URL: `https://v2.convertapi.com/convert/docx/to/pdf`  
     - Authentication: Select `Generic Credential` → Use HTTP Query Authentication  
     - Configure Query Auth Credential: Create or use existing credential named `Query Auth account` with one query parameter named `secret` set to your ConvertAPI secret key.  
     - Content Type: multipart/form-data  
     - Body Parameters: Add one parameter: Name = `file`, Value = expression `{{$json.url_to_file}}`  
     - Header Parameters: Add one header: Name = `Accept`, Value = `application/octet-stream`  
     - Options: Enable response format as file (to handle binary response)  
   - This node calls ConvertAPI to perform the conversion.

4. **Add Read/Write Files node named `Read/Write Files from Disk`**  
   - Connect output of `HTTP Request` to this node.  
   - Configure:  
     - Operation: Write  
     - File Name: `document.pdf`  
     - Data Property Name: Set to `=data` to use binary data from HTTP Request node output.  
   - This saves the converted PDF file locally.

5. **Add Sticky Notes (optional for documentation):**  
   - Add a Sticky Note near the `Config` node with content:  
     ```
     ## Configuration 
     Change the `url_to_file` parameter here to the file you want to convert
     ```  
   - Add a Sticky Note near the `HTTP Request` node with content:  
     ```
     ## Authentication
     Conversion requests must be authenticated. Please create 
     [ConvertAPI account to get authentication secret](https://www.convertapi.com/a/signin)

     Create a query auth credential with `secret` as name and your secret from the convertAPI dashboard as value
     ```

6. **Credential Setup:**  
   - In n8n credentials, create a new HTTP Query Auth credential:  
     - Name: `Query Auth account`  
     - Query Parameter: `secret` = your ConvertAPI secret key (obtained after account creation at https://www.convertapi.com/a/signin)

7. **Test the Workflow:**  
   - Click “Execute Workflow” or “Test workflow” to run the process.  
   - The workflow will fetch the DOCX file from the specified URL, convert it to PDF via ConvertAPI, and save `document.pdf` locally.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| The ConvertAPI documentation and available endpoints can be found here: https://www.convertapi.com/api                                                        | Official API documentation                    |
| To obtain an API secret, create a free account at ConvertAPI: https://www.convertapi.com/a/signin                                                           | Authentication setup                          |
| This workflow requires the `HTTP Request` node version 4.2 or later for binary file response handling                                                      | Node version requirement                       |
| The output filename is fixed as `document.pdf` in the workflow; modify the `Read/Write Files from Disk` node to change output filename if desired           | Customization note                            |
| Ensure the URL provided in `url_to_file` directly points to a publicly accessible DOCX file                                                                 | Input data requirement                        |

---

This documentation fully describes the workflow's structure, logic, and configuration, allowing reproduction, modification, and troubleshooting by both human users and automation agents.