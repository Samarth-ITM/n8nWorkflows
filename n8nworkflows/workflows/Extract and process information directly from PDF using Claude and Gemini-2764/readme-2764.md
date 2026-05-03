Extract and process information directly from PDF using Claude and Gemini

https://n8nworkflows.xyz/workflows/extract-and-process-information-directly-from-pdf-using-claude-and-gemini-2764


# Extract and process information directly from PDF using Claude and Gemini

### 1. Workflow Overview

This workflow is designed to extract and process information directly from a PDF document using two advanced AI models: Claude 3.5 Sonnet and Gemini 2.0 Flash. Its primary use case is to enable users to compare the extraction quality, latency, and cost of these two models when processing PDF content without the traditional two-step approach of OCR followed by language model processing. Instead, it sends the PDF content encoded in base64 directly to each AI model with a user-defined prompt, streamlining the extraction process into a single step.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Prompt Definition:** Manual trigger and prompt setup for extraction instructions.
- **1.2 PDF Retrieval and Conversion:** Downloading the PDF from Google Drive and converting it to a base64 string.
- **1.3 Parallel AI Processing:** Sending the base64 PDF and prompt to Claude 3.5 Sonnet and Gemini 2.0 Flash APIs simultaneously for extraction.
- **1.4 Result Comparison and Output:** (Implicit) The workflow outputs results from both AI calls for comparison.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Prompt Definition

**Overview:**  
This block initiates the workflow manually and defines the prompt that instructs the AI models on what information to extract from the PDF.

**Nodes Involved:**  
- When clicking 'Test workflow'  
- Define Prompt

**Node Details:**

- **When clicking 'Test workflow'**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually for testing or execution.  
  - *Configuration:* No parameters; triggers the workflow on user action.  
  - *Inputs:* None  
  - *Outputs:* Connects to "Define Prompt" node.  
  - *Edge Cases:* None significant; user must trigger manually.

- **Define Prompt**  
  - *Type:* Set  
  - *Role:* Defines a string variable `prompt` containing extraction instructions for the AI models.  
  - *Configuration:* Assigns the prompt string `"Extract the VAT numbers for each country"` to the variable `prompt`.  
  - *Inputs:* Receives trigger from manual node.  
  - *Outputs:* Passes prompt data to "Google Drive" node.  
  - *Edge Cases:* Prompt must be carefully constructed to ensure meaningful AI responses; improper prompts may yield irrelevant or incomplete data.

---

#### 1.2 PDF Retrieval and Conversion

**Overview:**  
This block downloads the specified PDF file from Google Drive and converts the binary file content into a base64-encoded string, which is required by both AI APIs for direct PDF processing.

**Nodes Involved:**  
- Google Drive  
- Extract from File

**Node Details:**

- **Google Drive**  
  - *Type:* Google Drive (Download operation)  
  - *Role:* Downloads the PDF file from Google Drive using a specified file ID.  
  - *Configuration:*  
    - `fileId` set to a specific Google Drive file (Invoice-798FE2FA-0004.pdf).  
    - OAuth2 credentials configured for Google Drive access.  
  - *Inputs:* Receives prompt data from "Define Prompt" (though prompt is not used here, this connection ensures sequential execution).  
  - *Outputs:* Passes binary PDF data to "Extract from File".  
  - *Edge Cases:*  
    - Authentication errors if OAuth2 token is invalid or expired.  
    - File not found or permission denied errors if file ID is incorrect or inaccessible.

- **Extract from File**  
  - *Type:* Extract From File (binaryToProperty operation)  
  - *Role:* Converts the binary PDF data into a base64 string stored in a JSON property (`data`).  
  - *Configuration:* Uses default options for binary to property conversion.  
  - *Inputs:* Receives binary PDF from "Google Drive".  
  - *Outputs:* Passes base64 string to both AI call nodes.  
  - *Edge Cases:*  
    - Failure if binary data is corrupted or improperly downloaded.  
    - Conversion errors if file format is unsupported.

---

#### 1.3 Parallel AI Processing

**Overview:**  
This block sends the base64-encoded PDF along with the user-defined prompt to two AI models — Claude 3.5 Sonnet and Gemini 2.0 Flash — to extract the requested information in parallel.

**Nodes Involved:**  
- Call Claude 3.5 Sonnet with PDF Capabilities  
- Call Gemini 2.0 Flash with PDF Capabilities

**Node Details:**

- **Call Claude 3.5 Sonnet with PDF Capabilities**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to Anthropic's Claude 3.5 Sonnet API with the PDF data and prompt for extraction.  
  - *Configuration:*  
    - URL: `https://api.anthropic.com/v1/messages`  
    - Method: POST  
    - Headers: Includes `anthropic-version: 2023-06-01` and `content-type: application/json`.  
    - Body: JSON containing model name `claude-3-5-sonnet-20241022`, max tokens 1024, and messages array with:  
      - Document object embedding the base64 PDF data with media type `application/pdf`.  
      - Text object containing the prompt from "Define Prompt".  
    - Authentication: Uses Anthropic API credentials.  
  - *Inputs:* Receives base64 PDF data from "Extract from File".  
  - *Outputs:* Returns Claude's extraction result.  
  - *Edge Cases:*  
    - API authentication failures (invalid or expired API key).  
    - Rate limiting or quota exceeded errors.  
    - Timeout or network errors.  
    - Response parsing errors if output format changes.  
  - *Version Requirements:* Requires n8n version supporting HTTP Request node v4.2 or higher for advanced JSON body expressions.

- **Call Gemini 2.0 Flash with PDF Capabilities**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to Google's Gemini 2.0 Flash API with the PDF data and prompt for extraction.  
  - *Configuration:*  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent`  
    - Method: POST  
    - Body: JSON with `contents` array containing:  
      - `inline_data` object with MIME type `application/pdf` and base64 data.  
      - Text part with the prompt from "Define Prompt".  
    - Authentication: Uses Google Palm API credentials.  
  - *Inputs:* Receives base64 PDF data from "Extract from File".  
  - *Outputs:* Returns Gemini's extraction result.  
  - *Edge Cases:*  
    - API key invalid or expired.  
    - Quota exceeded or rate limiting.  
    - Network or timeout issues.  
    - Changes in API endpoint or request format.  
  - *Version Requirements:* Requires n8n HTTP Request node v4.2+ for JSON body expressions.

---

### 3. Summary Table

| Node Name                                | Node Type           | Functional Role                                | Input Node(s)                 | Output Node(s)                                         | Sticky Note                                                                                                                |
|-----------------------------------------|---------------------|-----------------------------------------------|------------------------------|--------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow'            | Manual Trigger      | Initiates workflow manually                    | None                         | Define Prompt                                          |                                                                                                                            |
| Define Prompt                           | Set                 | Defines extraction prompt for AI models       | When clicking 'Test workflow' | Google Drive                                           | This prompt is used in both Gemini’s and Claude’s calls to define what information should be extracted and processed.       |
| Google Drive                           | Google Drive        | Downloads PDF file from Google Drive           | Define Prompt                | Extract from File                                      | These 2 steps first download the PDF file, and then convert it to base64. This is required by both APIs to process the file. |
| Extract from File                      | Extract From File   | Converts binary PDF to base64 string            | Google Drive                 | Call Claude 3.5 Sonnet with PDF Capabilities, Call Gemini 2.0 Flash with PDF Capabilities |                                                                                                                            |
| Call Claude 3.5 Sonnet with PDF Capabilities | HTTP Request        | Sends PDF and prompt to Claude 3.5 Sonnet API | Extract from File            | (Outputs Claude's response)                            | You can force Claude to output JSON with [Prefill response format](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency#prefill-claudes-response) |
| Call Gemini 2.0 Flash with PDF Capabilities | HTTP Request        | Sends PDF and prompt to Gemini 2.0 Flash API  | Extract from File            | (Outputs Gemini's response)                            | You can output the result as JSON by adding `"generationConfig": {"responseMimeType": "application/json"}` or use structured output. [Check the documentation](https://ai.google.dev/gemini-api/docs/structured-output?lang=rest) |
| Sticky Note1                          | Sticky Note         | Workflow overview and usage instructions       | None                         | None                                                  | See Workflow Overview section above.                                                                                        |
| Sticky Note                          | Sticky Note         | Gemini API JSON output tip                      | None                         | None                                                  | You can output the result as JSON by adding the following: `"generationConfig": {"responseMimeType": "application/json"}` or use structured output. [Check the documentation](https://ai.google.dev/gemini-api/docs/structured-output?lang=rest) |
| Sticky Note2                         | Sticky Note         | Claude JSON output tip                          | None                         | None                                                  | You can force Claude to output JSON with [Prefill response format](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency#prefill-claudes-response) |
| Sticky Note3                         | Sticky Note         | Explanation of PDF download and base64 conversion | None                         | None                                                  | These 2 steps first download the PDF file, and then convert it to base64. This is required by both APIs to process the file. |
| Sticky Note4                         | Sticky Note         | Explanation of prompt usage                      | None                         | None                                                  | This prompt is used in both Gemini’s and Claude’s calls to define what information should be extracted and processed.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking 'Test workflow'`  
   - Purpose: To manually start the workflow.

2. **Create a Set Node for Prompt Definition**  
   - Name: `Define Prompt`  
   - Add a string field named `prompt`.  
   - Set its value to the desired extraction instruction, e.g., `"Extract the VAT numbers for each country"`.  
   - Connect the output of the Manual Trigger node to this node.

3. **Create a Google Drive Node to Download PDF**  
   - Name: `Google Drive`  
   - Operation: `Download`  
   - Set the `fileId` to the Google Drive file you want to process (e.g., `18Ac2xorxirIBm9FNFDDB5aVUSPBCCg1U`).  
   - Configure Google Drive OAuth2 credentials with appropriate access rights.  
   - Connect the output of `Define Prompt` node to this node to maintain execution order.

4. **Create an Extract From File Node**  
   - Name: `Extract from File`  
   - Operation: `binaryToProperty` (convert binary data to base64 string)  
   - Connect the output of `Google Drive` node to this node.

5. **Create an HTTP Request Node for Claude 3.5 Sonnet API Call**  
   - Name: `Call Claude 3.5 Sonnet with PDF Capabilities`  
   - HTTP Method: POST  
   - URL: `https://api.anthropic.com/v1/messages`  
   - Headers:  
     - `anthropic-version`: `2023-06-01`  
     - `content-type`: `application/json`  
   - Body (JSON):  
     ```json
     {
       "model": "claude-3-5-sonnet-20241022",
       "max_tokens": 1024,
       "messages": [{
         "role": "user",
         "content": [
           {
             "type": "document",
             "source": {
               "type": "base64",
               "media_type": "application/pdf",
               "data": "{{$json.data}}"
             }
           },
           {
             "type": "text",
             "text": "{{ $('Define Prompt').item.json.prompt }}"
           }
         ]
       }]
     }
     ```
   - Authentication: Use Anthropic API credentials (API key).  
   - Connect the output of `Extract from File` node to this node.

6. **Create an HTTP Request Node for Gemini 2.0 Flash API Call**  
   - Name: `Call Gemini 2.0 Flash with PDF Capabilities`  
   - HTTP Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent`  
   - Body (JSON):  
     ```json
     {
       "contents": [
         {
           "parts": [
             {
               "inline_data": {
                 "mime_type": "application/pdf",
                 "data": "{{ $json.data }}"
               }
             },
             {
               "text": "{{ $('Define Prompt').item.json.prompt }}"
             }
           ]
         }
       ]
     }
     ```
   - Authentication: Use Google Palm API credentials (API key).  
   - Connect the output of `Extract from File` node to this node.

7. **Optional: Add Sticky Notes**  
   - Add sticky notes to document workflow overview, prompt usage, API output tips, and PDF processing explanation for user clarity.

8. **Test the Workflow**  
   - Trigger manually via the Manual Trigger node.  
   - Verify that the PDF is downloaded, converted, and sent to both AI APIs.  
   - Inspect the outputs for both Claude and Gemini calls to compare extraction results.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow extracts and processes PDF data in one step, bypassing the traditional OCR + LLM flow. | Overview of workflow purpose and design.                                                               |
| Modify the prompt in "Define Prompt" to customize extraction targets.                                | User customization instructions.                                                                       |
| Obtain API keys from: [Claude API keys](https://console.anthropic.com/settings/keys) and [Gemini API keys](https://aistudio.google.com/app/apikey) | API credential setup.                                                                                   |
| Claude JSON output can be forced using Prefill response format: [Claude Prefill docs](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency#prefill-claudes-response) | Enhancing Claude response consistency.                                                                 |
| Gemini supports structured JSON output: [Gemini Structured Output docs](https://ai.google.dev/gemini-api/docs/structured-output?lang=rest) | Improving Gemini response parsing and structure.                                                       |
| Ensure Google Drive OAuth2 credentials have access to the target file.                               | Credential and permission management.                                                                  |
| The workflow enables side-by-side comparison of AI extraction quality, latency, and cost.           | Use case and evaluation purpose.                                                                       |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Extract and process information directly from PDF using Claude and Gemini" workflow in n8n. It anticipates common failure modes and highlights configuration points critical for successful integration.