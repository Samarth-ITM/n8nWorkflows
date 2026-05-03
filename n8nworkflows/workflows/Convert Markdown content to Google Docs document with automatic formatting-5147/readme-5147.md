Convert Markdown content to Google Docs document with automatic formatting

https://n8nworkflows.xyz/workflows/convert-markdown-content-to-google-docs-document-with-automatic-formatting-5147


# Convert Markdown content to Google Docs document with automatic formatting

### 1. Workflow Overview

This workflow automates the conversion of Markdown content into a fully formatted Google Docs document stored in a specified Google Drive folder. It is designed for use cases such as generating technical documentation, content publishing, blog/article preparation, or report generation where Markdown text needs to be rendered and collaboratively edited in Google Docs with automatic timestamping and organization.

The workflow is logically divided into these blocks:

- **1.1 Input Preparation**: Manual trigger and setting initial input data including Markdown content, target Google Drive folder URL, and document title.
- **1.2 Markdown to HTML Conversion**: Converts Markdown content into HTML with specific formatting options.
- **1.3 Content Augmentation and File Preparation**: Adds metadata about Google Drive location to the HTML content, converts it to a text file, and prepares an initial empty Google Docs file in the target folder.
- **1.4 File Merging and MIME Type Adjustment**: Combines the generated file data with document metadata and adjusts MIME type for correct Google Docs conversion.
- **1.5 Document Finalization**: Waits briefly to allow document creation, then updates the Google Docs file content with the correctly formatted HTML.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Preparation

- **Overview:**  
  Captures input parameters and initiates the workflow manually.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set Input Data

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual workflow execution.  
    - Config: No parameters; triggers workflow on demand.  
    - Connections: Outputs to "Set Input Data".  
    - Edge Cases: None specific; manual trigger depends on user action.

  - **Set Input Data**  
    - Type: Set  
    - Role: Defines static input values for the workflow to use.  
    - Config:  
      - Google Drive URL: Target folder URL where the document will be stored.  
      - Content Title: Document's title.  
      - Conten in Markdown: Raw Markdown text content to convert.  
    - Expressions: Uses direct string assignment with embedded Markdown text.  
    - Connections: Outputs to "Create Empty File" and "Change Markdown To HTML".  
    - Edge Cases: Invalid URL or empty Markdown would cause downstream errors.

#### 2.2 Markdown to HTML Conversion

- **Overview:**  
  Converts the Markdown input into HTML format with options to support emoji, tables, and heading spacing.

- **Nodes Involved:**  
  - Change Markdown To HTML

- **Node Details:**

  - **Change Markdown To HTML**  
    - Type: Markdown  
    - Role: Converts Markdown to HTML content.  
    - Config:  
      - Mode: markdownToHtml  
      - Options: emoji enabled, tables enabled, no header ID generation, require space before heading text.  
      - Source Markdown: Uses expression referencing `Conten in Markdown` from input JSON.  
      - Output Key: Content in HTML  
    - Connections: Outputs to "Add Information About Google Drive Location".  
    - Edge Cases: Malformed Markdown might produce unexpected HTML or errors.

#### 2.3 Content Augmentation and File Preparation

- **Overview:**  
  Adds Google Drive location metadata to the HTML content, converts the enriched HTML to a text file, and creates an initial empty Google Docs file in the specified Drive folder.

- **Nodes Involved:**  
  - Add Information About Google Drive Location  
  - Convert Content to Text File  
  - Create Empty File

- **Node Details:**

  - **Add Information About Google Drive Location**  
    - Type: Set  
    - Role: Prepends an HTML unordered list with a hyperlink to the Google Drive folder URL before the HTML content.  
    - Config:  
      - Assigns a new string field `Content` combining a formatted link with existing HTML content.  
      - Uses expressions to embed the Drive URL and HTML content.  
    - Connections: Outputs to "Convert Content to Text File".  
    - Edge Cases: Invalid Drive URL would result in broken links in the final document.

  - **Convert Content to Text File**  
    - Type: Convert To File  
    - Role: Converts the HTML content string into a binary text file named "content.html".  
    - Config:  
      - Operation: toText  
      - Source Property: Content (the augmented HTML content)  
      - Binary Property Name: file  
    - Connections: Outputs to "Change Mime Type of The File".  
    - Edge Cases: Missing content or conversion errors could interrupt the flow.

  - **Create Empty File**  
    - Type: Google Drive  
    - Role: Creates a new Google Docs file in the target Drive folder with a timestamped name.  
    - Config:  
      - Name: Template with prefix "_PUB" plus content title and current timestamp (format yyyy-MM-dd HH:mm:ss).  
      - Content: "Dummy File Content" (placeholder to create the file).  
      - Drive: "My Drive" selected explicitly.  
      - Folder: Uses the Drive folder URL from input, resolved by URL mode.  
      - Operation: createFromText with option to convert to Google Document.  
    - Credentials: Uses configured Google Drive OAuth2 credentials.  
    - Connections: Outputs to "Merge File With About Document ID".  
    - Edge Cases: Folder URL must be correct and accessible or creation fails; insufficient permissions cause auth errors.

#### 2.4 File Merging and MIME Type Adjustment

- **Overview:**  
  Adjusts the MIME type of the generated HTML file to ensure correct interpretation and merges this with the Google Docs file metadata.

- **Nodes Involved:**  
  - Change Mime Type of The File  
  - Merge File With About Document ID

- **Node Details:**

  - **Change Mime Type of The File**  
    - Type: Code  
    - Role: Changes MIME type of the binary file from default 'text/plain' to 'text/html' for proper formatting.  
    - Config:  
      - Runs JavaScript for each item.  
      - Checks if binary file exists and sets mimeType to 'text/html'.  
    - Connections: Outputs to "Merge File With About Document ID".  
    - Edge Cases: No binary file present would cause no change; malformed binary data may cause errors.

  - **Merge File With About Document ID**  
    - Type: Merge  
    - Role: Combines data from the file conversion and Google Docs creation nodes by position to unify metadata and file content.  
    - Config:  
      - Mode: Combine  
      - Combine By: position (index)  
    - Connections: Outputs to "Wait for Document Creation".  
    - Edge Cases: Mismatched item counts between inputs lead to data loss or errors.

#### 2.5 Document Finalization

- **Overview:**  
  Waits for a short delay to ensure Google Docs file creation completes, then updates the document content with the correctly formatted HTML file.

- **Nodes Involved:**  
  - Wait for Document Creation  
  - Update Document with Correct HTML Formatting

- **Node Details:**

  - **Wait for Document Creation**  
    - Type: Wait  
    - Role: Introduces a 0.5-minute delay to allow asynchronous Google Docs file creation to finalize before update.  
    - Config:  
      - Amount: 0.5 minutes (30 seconds)  
    - Webhook ID: Present but not externally triggered; used internally for delay.  
    - Connections: Outputs to "Update Document with Correct HTML Formatting".  
    - Edge Cases: Insufficient wait time might cause update failure; excessive delay slows workflow.

  - **Update Document with Correct HTML Formatting**  
    - Type: Google Drive  
    - Role: Updates the newly created Google Docs file by replacing its content with the HTML file content for formatted display.  
    - Config:  
      - File ID: Uses the Google Docs file ID from merged data.  
      - Change File Content: true (content replaced)  
      - Input Data Field Name: file (binary file with HTML content)  
      - Options: Requests webViewLink and id fields in response.  
    - Credentials: Uses same Google Drive OAuth2 credentials as file creation.  
    - Connections: Terminal node (no outputs).  
    - Edge Cases: Invalid file ID or access permissions cause update failure; malformed HTML can corrupt document.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                      | Input Node(s)                         | Output Node(s)                         | Sticky Note                                                                                                                             |
|-----------------------------------|---------------------|-----------------------------------------------------|-------------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger      | Entry point, manual workflow start                   | —                                   | Set Input Data                        | 📝 Markdown to Google Docs Converter ... (see full note in Sticky Note node)                                                            |
| Set Input Data                    | Set                 | Defines initial input variables                       | When clicking ‘Test workflow’        | Create Empty File, Change Markdown To HTML |                                                                                                                                        |
| Change Markdown To HTML           | Markdown            | Converts Markdown to HTML with formatting options    | Set Input Data                      | Add Information About Google Drive Location |                                                                                                                                        |
| Add Information About Google Drive Location | Set          | Adds Drive location metadata into HTML content       | Change Markdown To HTML             | Convert Content to Text File          |                                                                                                                                        |
| Convert Content to Text File      | Convert To File     | Converts HTML string content into a text file binary | Add Information About Google Drive Location | Change Mime Type of The File           |                                                                                                                                        |
| Change Mime Type of The File      | Code                | Changes MIME type from text/plain to text/html       | Convert Content to Text File        | Merge File With About Document ID      |                                                                                                                                        |
| Create Empty File                | Google Drive         | Creates new Google Docs file with timestamped name   | Set Input Data                      | Merge File With About Document ID      |                                                                                                                                        |
| Merge File With About Document ID | Merge               | Combines file content and Google Docs file metadata  | Create Empty File, Change Mime Type of The File | Wait for Document Creation             |                                                                                                                                        |
| Wait for Document Creation        | Wait                | Waits 30 seconds to allow document creation          | Merge File With About Document ID   | Update Document with Correct HTML Formatting |                                                                                                                                        |
| Update Document with Correct HTML Formatting | Google Drive  | Updates Google Docs file content with HTML file      | Wait for Document Creation          | —                                     |                                                                                                                                        |
| Sticky Note                      | Sticky Note          | Provides detailed purpose, setup, usage, and tips    | —                                   | —                                     | 📝 Markdown to Google Docs Converter ... (full content describing purpose, setup, usage, customization, and use cases)                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Add a Manual Trigger Node**  
   - Name: When clicking ‘Test workflow’  
   - No parameters.

2. **Add a Set Node for Input Data**  
   - Name: Set Input Data  
   - Define three string fields:  
     - `Google Drive URL`: e.g. `https://drive.google.com/drive/folders/1ObGBisk0j56rn93OFxYPd24t4Se6dElY`  
     - `Content Title`: e.g. `Łyżki do butów`  
     - `Conten in Markdown`: Paste raw Markdown text content here.  
   - Connect "When clicking ‘Test workflow’" output to this node.

3. **Add a Markdown Node to Convert Markdown to HTML**  
   - Name: Change Markdown To HTML  
   - Mode: markdownToHtml  
   - Options: Enable emoji, tables, noHeaderId, requireSpaceBeforeHeadingText  
   - Markdown input: Expression `={{ $json['Conten in Markdown'] }}`  
   - Destination key: `Content in HTML`  
   - Connect "Set Input Data" output to this node.

4. **Add a Set Node to Add Drive Location Info**  
   - Name: Add Information About Google Drive Location  
   - Assign new field `Content` with value:  
     ```
     <ul>
     <li>URL DYSKU: <a href="{{ $json['Google Drive URL'] }}">{{ $json['Google Drive URL'] }}</a></li>
     </ul>

     {{ $json['Content in HTML'] }}
     ```  
   - Use expressions to insert `Google Drive URL` and `Content in HTML`.  
   - Connect "Change Markdown To HTML" output to this node.

5. **Add a Convert To File Node**  
   - Name: Convert Content to Text File  
   - Operation: toText  
   - Source Property: `Content`  
   - File Name: `content.html`  
   - Binary Property Name: `file`  
   - Connect "Add Information About Google Drive Location" output to this node.

6. **Add a Code Node to Change MIME Type**  
   - Name: Change Mime Type of The File  
   - JavaScript Code (run once per item):  
     ```js
     const item = $input.item;
     if (item.binary && item.binary.file) {
       item.binary.file.mimeType = 'text/html';
     }
     return item;
     ```  
   - Connect "Convert Content to Text File" output to this node.

7. **Add a Google Drive Node to Create an Empty Google Docs File**  
   - Name: Create Empty File  
   - Operation: createFromText  
   - Name: Use expression `_PUB {{ $json['Content Title'] }} {{ $now.format('yyyy-MM-dd HH:mm:ss') }}`  
   - Content: `Dummy File Content` (placeholder)  
   - Drive: Select "My Drive" or your target drive  
   - Folder ID: Use URL mode with expression `={{ $json['Google Drive URL'] }}`  
   - Options: Enable "convertToGoogleDocument"  
   - Credentials: Configure Google Drive OAuth2 credentials  
   - Connect "Set Input Data" output to this node.

8. **Add a Merge Node to Combine File and Document Data**  
   - Name: Merge File With About Document ID  
   - Mode: Combine  
   - Combine By: Position (index)  
   - Connect "Create Empty File" node to input 2  
   - Connect "Change Mime Type of The File" node to input 1

9. **Add a Wait Node**  
   - Name: Wait for Document Creation  
   - Amount: 0.5 minutes (30 seconds)  
   - Connect "Merge File With About Document ID" output to this node.

10. **Add a Google Drive Node to Update the Document**  
    - Name: Update Document with Correct HTML Formatting  
    - Operation: update  
    - File ID: Use expression `={{ $json.id }}` (from previous merge output)  
    - Change File Content: true  
    - Input Data Field Name: `file` (binary file with HTML content)  
    - Options: Request fields `webViewLink` and `id`  
    - Credentials: Use the same Google Drive OAuth2 credentials as before  
    - Connect "Wait for Document Creation" output to this node.

11. **Connect the Nodes in Proper Order:**  
    - When clicking ‘Test workflow’ → Set Input Data  
    - Set Input Data → Create Empty File  
    - Set Input Data → Change Markdown To HTML  
    - Change Markdown To HTML → Add Information About Google Drive Location  
    - Add Information About Google Drive Location → Convert Content to Text File  
    - Convert Content to Text File → Change Mime Type of The File  
    - Change Mime Type of The File → Merge File With About Document ID (input 1)  
    - Create Empty File → Merge File With About Document ID (input 2)  
    - Merge File With About Document ID → Wait for Document Creation  
    - Wait for Document Creation → Update Document with Correct HTML Formatting  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| 📝 Markdown to Google Docs Converter\n\n🎯 PURPOSE: Convert Markdown content to formatted Google Docs documents with automatic timestamping and Drive organization\n\n⚙️ SETUP REQUIRED:\n• Google Drive OAuth2 credentials configured in n8n\n• Target Google Drive folder URL\n• Content title and Markdown text input\n\n📋 HOW TO USE:\n1. Configure Google Drive credentials in n8n settings\n2. Update "Set Input Data" node with:\n   - Google Drive URL (target folder)\n   - Content Title (document name)\n   - Content in Markdown (your text)\n3. Click "Test workflow" to execute\n4. Check your Google Drive for timestamped document\n\n🔧 CUSTOMIZE:\n• Modify HTML formatting options in "Change Markdown To HTML" node\n• Adjust file naming pattern in "Create Empty File" node  \n• Change Drive location metadata in "Add Information About Google Drive Location"\n• Update MIME type handling in "Change Mime Type of The File"\n\n💡 USE CASES:\n• Technical documentation (README → Google Docs)\n• Content publishing workflow (draft → review)\n• Blog/article preparation (Markdown → collaborative editing)\n• Report generation with automated formatting | Sticky Note attached to the workflow |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.