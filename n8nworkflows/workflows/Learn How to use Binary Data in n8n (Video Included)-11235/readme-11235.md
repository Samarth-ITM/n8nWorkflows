Learn How to use Binary Data in n8n (Video Included)

https://n8nworkflows.xyz/workflows/learn-how-to-use-binary-data-in-n8n--video-included--11235


# Learn How to use Binary Data in n8n (Video Included)

### 1. Workflow Overview

This workflow demonstrates multiple practical examples of how to handle binary data in n8n, including uploading files via forms, manipulating binary file properties, extracting data from binary files, analyzing images, and converting data formats. It is designed as an educational template with embedded video and detailed sticky notes to guide users in understanding and leveraging binary data nodes in n8n.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**  
  Nodes that trigger the workflow and receive binary data input, mainly through form submissions.

- **1.2 Binary Data Manipulation**  
  Nodes that edit fields by assigning or converting binary data, including setting binary properties and converting base64 strings.

- **1.3 Binary File Processing and Analysis**  
  Nodes that perform file extraction (e.g., from XLSX), image analysis using AI services, and text extraction from binary files.

- **1.4 File Upload and Storage**  
  Nodes responsible for uploading processed binary files to Google Drive or handling FTP operations.

- **1.5 User Guidance and Documentation**  
  Sticky Note nodes containing instructions, example descriptions, embedded video links, and credits.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block handles the reception of binary data via form triggers, initiating the workflow when a user submits a file.

- **Nodes Involved:**  
  - On form submission  
  - On form submission1  
  - When clicking ‘Execute workflow’  

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Receives initial binary data input from an external form submission webhook.  
    - Configurations:  
      - No additional options set.  
      - Webhook ID: "2d519e51-16a5-45b4-af16-0db2b0d8cc14"  
    - Inputs: External HTTP webhook trigger upon form submission.  
    - Outputs: Connected nodes not defined in connections, possibly isolated or for illustrative purposes.  
    - Edge cases: Possible webhook timeout or missing file in submission.

  - **On form submission1**  
    - Type: Form Trigger  
    - Role: Receives binary file input from a form containing a file field labeled "binary_file".  
    - Configurations:  
      - Form Title: "file uploader"  
      - Form Fields: Single file upload field named "binary_file".  
      - Webhook ID: "1b450f78-a9a6-41b3-ac9e-2fb89e625726"  
    - Inputs: HTTP webhook triggered on form file upload.  
    - Outputs: Connected to "Edit Fields1" node.  
    - Edge cases: File upload failure, unsupported file types, or empty submissions.

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow for testing or demonstration purposes.  
    - Configurations: None.  
    - Inputs: Manual trigger by user interaction in n8n UI.  
    - Outputs: Connected to "Edit Fields2" node.  
    - Edge cases: None significant; manual initiation.

---

#### 2.2 Binary Data Manipulation

- **Overview:**  
  This block demonstrates editing and assigning binary data using Set nodes and converting base64 strings to binary data.

- **Nodes Involved:**  
  - Edit Fields  
  - Edit Fields1  
  - Edit Fields2  
  - Convert to File2  

- **Node Details:**

  - **Edit Fields**  
    - Type: Set  
    - Role: Assigns binary data from a previous node's binary file to a new binary field.  
    - Configurations:  
      - Assignment of binary data type field "new_image" using expression referencing binary file from "On form submission1".  
      - Expression: `={{ $('On form submission1').item.binary.binary_file }}`  
    - Inputs: Expected to receive data from form submission with binary file.  
    - Outputs: Not explicitly connected, possibly for example purposes.  
    - Edge cases: Expression failure if binary data is missing.

  - **Edit Fields1**  
    - Type: Set  
    - Role: Adds a string field "id_name" with value "testers" to the data.  
    - Configurations:  
      - Adds a string field "id_name" with fixed value "testers".  
    - Inputs: Receives data from "On form submission1".  
    - Outputs: Connected to "Edit Image".  
    - Edge cases: None significant.

  - **Edit Fields2**  
    - Type: Set  
    - Role: Assigns a base64 string to a string field "image" for conversion.  
    - Configurations:  
      - Field "image" set to a long base64 encoded PNG image string.  
    - Inputs: Triggered manually by "When clicking ‘Execute workflow’".  
    - Outputs: Connected to "Convert to File2".  
    - Edge cases: Base64 string corruption or size too large for processing.

  - **Convert to File2**  
    - Type: Convert To File  
    - Role: Converts the string base64 image data in "image" field into binary data.  
    - Configurations:  
      - Operation: toBinary  
      - Source Property: "image"  
    - Inputs: Receives string field from "Edit Fields2".  
    - Outputs: Not connected further, demonstrating conversion.  
    - Edge cases: Invalid base64 data causing conversion failure.

---

#### 2.3 Binary File Processing and Analysis

- **Overview:**  
  This block contains nodes to extract content from binary files and analyze binary images using AI services.

- **Nodes Involved:**  
  - Extract from File  
  - Extract text  
  - Analyze an image  

- **Node Details:**

  - **Extract from File**  
    - Type: Extract From File  
    - Role: Extracts data from XLSX files in binary format.  
    - Configurations:  
      - Operation: xlsx  
      - Binary Property Name: binary_file  
    - Inputs: Expected binary file data from external trigger (e.g., Google Drive Trigger).  
    - Outputs: Not connected further, for demonstration.  
    - Edge cases: Unsupported file format, corrupted files, or empty binary data.

  - **Extract text**  
    - Type: Mistral AI  
    - Role: Extracts text content from a binary file using AI service.  
    - Configurations:  
      - Binary Property: binary_file  
      - Credential: Mistral Cloud account provided.  
    - Inputs: Binary file from trigger or prior node.  
    - Outputs: Not connected, example of AI text extraction.  
    - Edge cases: API credential errors, binary data format issues, or AI processing limits.

  - **Analyze an image**  
    - Type: Google Gemini (PaLM) via Langchain  
    - Role: Performs image analysis on binary image data to identify company logos.  
    - Configurations:  
      - Model: "models/gemini-2.5-flash"  
      - Resource: image  
      - Operation: analyze  
      - Input Type: binary  
      - Binary Property Name: binary_file  
      - Credential: Google Gemini (PaLM) API account provided.  
      - Prompt: "What are the company logos present in this image\nOutput as JSON only"  
    - Inputs: Binary image data.  
    - Outputs: Not connected further; demonstrates image analysis.  
    - Edge cases: API quota limits, auth failures, unsupported image formats, or empty binary data.

---

#### 2.4 File Upload and Storage

- **Overview:**  
  This block uploads binary files to Google Drive and includes an FTP node (unconnected) suggesting potential file transfer.

- **Nodes Involved:**  
  - Upload file  
  - FTP  

- **Node Details:**

  - **Upload file**  
    - Type: Google Drive  
    - Role: Uploads binary files to Google Drive root folder.  
    - Configurations:  
      - Drive ID: "My Drive"  
      - Folder ID: "root"  
      - Credentials: Google Drive OAuth2 API account configured.  
    - Inputs: Expected binary data input (not explicitly connected in provided JSON).  
    - Outputs: None connected, demonstrating upload capability.  
    - Edge cases: Authentication failure, upload quota exceeded, or invalid folder ID.

  - **FTP**  
    - Type: FTP  
    - Role: Placeholder or example node for potential FTP operations.  
    - Configurations: None specified.  
    - Inputs: None connected.  
    - Outputs: None connected.  
    - Edge cases: None active; unconnected node.

---

#### 2.5 User Guidance and Documentation

- **Overview:**  
  Sticky Note nodes provide tutorial-style explanations, instructions, example descriptions, and embedded video links related to binary data handling.

- **Nodes Involved:**  
  - Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note8, Sticky Note9, Sticky Note10, Sticky Note11, Sticky Note12  

- **Node Details:**

  Each Sticky Note node contains textual content to explain concepts such as:

  - How to access binary data using expressions.  
  - New features of the Set node for assigning binary data.  
  - Examples demonstrating extraction from files, image analysis, and base64 conversion.  
  - Instructions with embedded YouTube video link: https://youtu.be/0Vefm8vXFxE  
  - Credits to template creators Ryan & Matt Data Science on YouTube.  

  - Sticky Note positions and sizes are configured for clarity and grouping.

  - Edge cases: None; purely informational.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                   |
|----------------------------|--------------------------------|---------------------------------------|-------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note                | Sticky Note                    | Informational note about binary data  | -                       | -                       | Can now access binary data from earlier nodes using expressions and the set node includes a new entry type to assign binary data directly. |
| Sticky Note1               | Sticky Note                    | Example 6 explanation                  | -                       | -                       | ## Example 6\nAccess binary data from previous nodes using expression:                        |
| Sticky Note2               | Sticky Note                    | Example 5 explanation                  | -                       | -                       | ## Example 5\nNew type of entry in Set node to assign binary data:\n\nRename Example          |
| Edit Fields               | Set                            | Assign binary data to new field        | -                       | -                       |                                                                                                |
| Sticky Note3               | Sticky Note                    | How to access binary data              | -                       | -                       | How to access Binary Data\n\n(NODE).item.binary.BINARYNAME                                   |
| Edit Fields1              | Set                            | Add string field "id_name"             | On form submission1      | Edit Image               |                                                                                                |
| Edit Image                | Edit Image                     | Retrieve image info from binary data   | Edit Fields1             | -                       |                                                                                                |
| Sticky Note5               | Sticky Note                    | Explanation of binary data and conversion nodes | -                       | -                       | Binary data is any file-type data, such as image files or documents\n\nConvert to File to take input data and output it as a file.\nExtract From File to get data from a binary format and convert it to JSON. |
| Sticky Note6               | Sticky Note                    | Example 1 description                  | -                       | -                       | ## Example 1\nGetting Binary Data in n8n (Triggers/Form)\n\n4th data to show                  |
| Sticky Note7               | Sticky Note                    | Example 2 description                  | -                       | -                       | ## Example 2\nExtract from file node -> When doable                                          |
| Extract from File          | Extract From File              | Extract data from XLSX binary file     | -                       | -                       |                                                                                                |
| Sticky Note8               | Sticky Note                    | Example 3 description                  | -                       | -                       | ## Example 3\nExtract from file node -> When doable                                          |
| Extract text              | Mistral AI                    | Extract text from binary file using AI | -                       | -                       |                                                                                                |
| Sticky Note9               | Sticky Note                    | Example 4 description                  | -                       | -                       | ## Example 4\nAnalyze an Image                                                               |
| Analyze an image           | Google Gemini (Langchain)     | Analyze image binary data for logos    | -                       | -                       |                                                                                                |
| Sticky Note10              | Sticky Note                    | Example 7 description and video link | -                       | -                       | ## Example 7\nConvert to File to take input data and output it as a file.\n\nConvert to file Video: https://youtu.be/Rpw3D4MjPo8\n |
| Convert to File            | Convert To File               | Convert input data to file binary      | -                       | -                       |                                                                                                |
| Google Drive Trigger       | Google Drive Trigger          | Trigger on file creation in Google Drive (unconnected in this workflow) | -                       | -                       |                                                                                                |
| Sticky Note11              | Sticky Note                    | Example 8 description                  | -                       | -                       | ## Example 8\nBase64 allows us to change format \nand bring it back to its original format\n\nAPIs give image as base64\n\nLook for example of a website with it\n\nPut in a variable\n\nhttps://www.base64-image.de/\n\nREMOVE: data:image/png;base64, |
| Edit Fields2              | Set                            | Assign a base64 string to a field      | When clicking ‘Execute workflow’ | Convert to File2           |                                                                                                |
| Convert to File2           | Convert To File               | Convert base64 string to binary        | Edit Fields2             | -                       |                                                                                                |
| FTP                       | FTP                          | Placeholder for FTP operations          | -                       | -                       |                                                                                                |
| Upload file               | Google Drive                 | Upload binary file to Google Drive      | -                       | -                       |                                                                                                |
| Sticky Note4               | Sticky Note                    | YouTube video embedded link            | -                       | -                       | @[youtube](0Vefm8vXFxE)                                                                      |
| Sticky Note12              | Sticky Note                    | Template credit and general description | -                       | -                       | ## Template & Tutorial created by Ryan & Matt Data Science on YouTube\nThanks for checking this template out. We have a video embedded going over how to use Binary files through multiple examples  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node "On form submission1"**  
   - Type: Form Trigger  
   - Webhook ID: auto-generated or specify "1b450f78-a9a6-41b3-ac9e-2fb89e625726"  
   - Form Title: "file uploader"  
   - Add one form field: Type = File, Label = "binary_file"

2. **Create Set node "Edit Fields1"**  
   - Add a string field assignment: `id_name` = "testers"  
   - Connect input from "On form submission1"

3. **Create Edit Image node "Edit Image"**  
   - Operation: "information"  
   - Data Property Name: `={{ $('On form submission1').item.binary.binary_file }}`  
   - Connect input from "Edit Fields1"

4. **Create Manual Trigger node "When clicking ‘Execute workflow’"**  
   - No parameters needed

5. **Create Set node "Edit Fields2"**  
   - Add string assignment field "image" with the provided base64 PNG string (as in the workflow)  
   - Connect input from "When clicking ‘Execute workflow’"

6. **Create Convert to File node "Convert to File2"**  
   - Operation: "toBinary"  
   - Source Property: "image"  
   - Connect input from "Edit Fields2"

7. **Create Set node "Edit Fields"** (for binary assignment example)  
   - Add binary field assignment "new_image" with expression: `={{ $('On form submission1').item.binary.binary_file }}`

8. **Create Extract From File node "Extract from File"**  
   - Operation: "xlsx"  
   - Binary Property Name: "binary_file"  
   - Connect input appropriate to your trigger or file input node

9. **Create Mistral AI node "Extract text"**  
   - Binary Property: "binary_file"  
   - Set credentials for Mistral Cloud API with valid account

10. **Create Google Gemini (Langchain) node "Analyze an image"**  
    - Text Prompt: "What are the company logos present in this image\nOutput as JSON only"  
    - Model ID: "models/gemini-2.5-flash"  
    - Resource: "image"  
    - Operation: "analyze"  
    - Input Type: "binary"  
    - Binary Property Name: "binary_file"  
    - Set credentials for Google Gemini (PaLM) API account

11. **Create Convert to File node "Convert to File"**  
    - Default settings to convert input data to file

12. **Create Google Drive node "Upload file"**  
    - Drive ID: "My Drive"  
    - Folder ID: "root"  
    - Set credentials for Google Drive OAuth2 API

13. **Create FTP node "FTP"** (optional, as example)  
    - Configure FTP credentials if needed

14. **Create Sticky Note nodes**  
    - Add each sticky note with content from the workflow to provide documentation and guidance.  
    - For the YouTube video, add a sticky note with content: `@[youtube](0Vefm8vXFxE)`  
    - Add the credit note with content:  
      "## Template & Tutorial created by Ryan & Matt Data Science on YouTube  
      Thanks for checking this template out. We have a video embedded going over how to use Binary files through multiple examples"

15. **Connect nodes as per the described connections:**  
    - "On form submission1" → "Edit Fields1" → "Edit Image"  
    - "When clicking ‘Execute workflow’" → "Edit Fields2" → "Convert to File2"  
    - Other nodes are standalone or for demonstration; connect triggers and actions as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Can now access binary data from earlier nodes using expressions and the set node includes a new entry type to assign binary data directly. | General feature explanation                       |
| Binary data is any file-type data, such as image files or documents. Convert to File to take input data and output it as a file. Extract From File to get data from a binary format and convert it to JSON. | Conceptual explanation of binary data handling   |
| ## Template & Tutorial created by Ryan & Matt Data Science on YouTube Thanks for checking this template out. We have a video embedded going over how to use Binary files through multiple examples | Credit and video tutorial link                    |
| @[youtube](0Vefm8vXFxE)                                                                              | Embedded video link on binary data usage in n8n  |
| ## Example 6 Access binary data from previous nodes using expression:                                | Example description                              |
| ## Example 5 New type of entry in Set node to assign binary data: Rename Example                     | Example description                              |
| ## Example 1 Getting Binary Data in n8n (Triggers/Form) 4th data to show                              | Example description                              |
| ## Example 2 Extract from file node -> When doable                                                   | Example description                              |
| ## Example 3 Extract from file node -> When doable                                                   | Example description                              |
| ## Example 4 Analyze an Image                                                                        | Example description                              |
| ## Example 7 Convert to File to take input data and output it as a file. Convert to file Video: https://youtu.be/Rpw3D4MjPo8 | Example description and video link                |
| ## Example 8 Base64 allows us to change format and bring it back to its original format APIs give image as base64 Look for example of a website with it Put in a variable https://www.base64-image.de/ REMOVE: data:image/png;base64, | Example description and base64 image resource link |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.