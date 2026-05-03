CV Resume PDF Parsing with Multimodal Vision AI

https://n8nworkflows.xyz/workflows/cv-resume-pdf-parsing-with-multimodal-vision-ai-2416


# CV Resume PDF Parsing with Multimodal Vision AI

### 1. Workflow Overview

This workflow demonstrates an advanced use case of multimodal Large Language Models (LLMs) for parsing and analyzing candidate resumes in PDF format. Its primary goal is to filter out unqualified candidates, especially those attempting to bypass AI screening with hidden prompts (e.g., white font text invisible to conventional text extraction). The workflow achieves this by converting the PDF resume into an image, which is then processed by a vision-capable LLM (Google Gemini 1.5 Pro model in this case). This approach ensures that hidden text prompts do not influence the AI’s decision.

Logical blocks:

- **1.1 Input Reception:** Trigger the workflow and download the candidate's resume PDF from Google Drive.
- **1.2 PDF Processing:** Convert the PDF resume into an image using the Stirling PDF API and resize the image.
- **1.3 AI Resume Analysis:** Send the image to a multimodal LLM (Google Gemini) for qualification analysis.
- **1.4 Decision Making:** Parse the AI output and decide whether to proceed with the candidate.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block starts the workflow upon manual trigger and obtains the candidate resume PDF from Google Drive for subsequent processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Download Resume (Google Drive)  
- Sticky Note1 (Documentation)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates workflow execution manually for testing or demo purposes.  
  - Configuration: No parameters; triggers on user action.  
  - Input: None  
  - Output: Triggers "Download Resume" node.  
  - Edge cases: None specific, but manual trigger requires user interaction.

- **Download Resume**  
  - Type: Google Drive node  
  - Role: Downloads the PDF file of the candidate resume from Google Drive by file ID.  
  - Configuration:  
    - Operation: Download  
    - File ID: Set to a specific Google Drive file (the example resume with hidden prompt).  
    - Credentials: Uses Google Drive OAuth2 API credential.  
  - Input: Triggered by Manual Trigger node.  
  - Output: Binary file data of the PDF resume.  
  - Edge cases:  
    - Authentication failure with Google Drive OAuth2 credentials.  
    - File not found or access denied.  
    - Network timeout.  
  - Notes: The PDF is known to contain a hidden prompt to bypass AI screening.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Provides context and instructions about this block and the Google Drive integration.  
  - Content: Explains the test resume, hidden prompt, and Google Drive usage.

---

#### 1.2 PDF Processing

**Overview:**  
This block converts the downloaded PDF resume into an image format suitable for vision-based AI processing and resizes it to optimize processing speed.

**Nodes Involved:**  
- PDF-to-Image API (HTTP Request)  
- Resize Converted Image (Edit Image)  
- Sticky Note2, Sticky Note4 (Documentation)

**Node Details:**

- **PDF-to-Image API**  
  - Type: HTTP Request  
  - Role: Sends the PDF binary to the Stirling PDF API to convert it into a single JPG image at 300 DPI.  
  - Configuration:  
    - Method: POST  
    - URL: https://stirlingpdf.io/api/v1/convert/pdf/img  
    - Content-Type: multipart/form-data  
    - Body Parameters:  
      - fileInput: PDF binary from previous node  
      - imageFormat: jpg  
      - singleOrMultiple: single (one image output for the entire PDF)  
      - dpi: 300 (resolution)  
  - Input: PDF file binary from "Download Resume" node.  
  - Output: JPG image binary representing the resume.  
  - Edge cases:  
    - API unavailability or timeout.  
    - Invalid PDF file input.  
    - Rate limiting or authentication issues if API requires it.  
  - Notes: Public API used for demo; recommended to self-host for privacy.

- **Resize Converted Image**  
  - Type: Edit Image  
  - Role: Resizes the converted image to 75% of its original width and height to speed up downstream processing.  
  - Configuration:  
    - Operation: Resize  
    - Resize Option: Percent  
    - Width: 75%  
    - Height: 75%  
  - Input: Image binary from "PDF-to-Image API" node.  
  - Output: Resized image binary.  
  - Edge cases:  
    - Invalid image format or corrupted data.  
    - Processing failure or resource constraints.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Explains the need for PDF-to-image conversion and introduces Stirling PDF API usage.  
  - Content: Details about AI vision models and why conversion is necessary.

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Provides data privacy warning about using the public Stirling PDF API instance.  
  - Content: Recommends self-hosting for production use.

---

#### 1.3 AI Resume Analysis

**Overview:**  
This block sends the resized image of the resume to a multimodal Google Gemini LLM for evaluation and qualification determination.

**Nodes Involved:**  
- Google Gemini Chat Model (Google Gemini LLM)  
- Candidate Resume Analyser (Chain LLM)  
- Structured Output Parser (Output Parser Structured)  
- Sticky Note3 (Documentation)

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat Model  
  - Role: Provides the underlying AI model for multimodal processing.  
  - Configuration:  
    - Model Name: models/gemini-1.5-pro-latest  
    - Credentials: Google Palm API credentials.  
  - Input: Connected as the language model for the "Candidate Resume Analyser" node.  
  - Output: AI-generated response.  
  - Edge cases:  
    - API key or quota issues.  
    - Model unavailability or endpoint errors.  
    - Latency or timeout.

- **Candidate Resume Analyser**  
  - Type: LangChain Chain LLM  
  - Role: Sends a prompt with the candidate’s resume image to the multimodal LLM to assess if the candidate qualifies for the role of Plumber.  
  - Configuration:  
    - Prompt includes instructions to evaluate the resume and decide qualification.  
    - Messages:  
      - Text message instructing assessment.  
      - HumanMessagePromptTemplate expecting an image binary input.  
    - Output parser enabled (connected to Structured Output Parser).  
  - Input: Image binary from "Resize Converted Image" node and AI model from "Google Gemini Chat Model".  
  - Output: JSON structured response indicating qualification status and reason.  
  - Edge cases:  
    - AI response parsing failures.  
    - Unexpected or malformed AI output.  
    - Model misinterpretation or hallucination.

- **Structured Output Parser**  
  - Type: Output Parser Structured (LangChain)  
  - Role: Parses the AI response into a structured JSON format with fields like `is_qualified` (boolean) and `reason` (string).  
  - Configuration:  
    - JSON Schema Example provided for expected output format.  
  - Input: AI output from "Candidate Resume Analyser" node.  
  - Output: Parsed JSON data used for decision making.  
  - Edge cases:  
    - Parsing errors if AI output is not compliant with schema.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Explains the usage of multimodal LLMs and the rationale for image input to avoid hidden prompt bypasses.  
  - Content: Overview of multimodal LLM benefits and usage of Google Gemini model.

---

#### 1.4 Decision Making

**Overview:**  
This block evaluates the AI’s decision on candidate qualification and determines whether to proceed to the next stage of the pipeline.

**Nodes Involved:**  
- Should Proceed To Stage 2? (If node)

**Node Details:**

- **Should Proceed To Stage 2?**  
  - Type: If  
  - Role: Checks if the candidate is qualified based on the AI's parsed output and conditionally routes the workflow.  
  - Configuration:  
    - Condition: `$json.output.is_qualified === true` (strict boolean check)  
  - Input: Parsed JSON from "Candidate Resume Analyser" node.  
  - Output: Branches "true" or "false" depending on qualification.  
  - Edge cases:  
    - Missing or malformed `is_qualified` field.  
    - Type mismatch in condition evaluation.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                      | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                                                  |
|----------------------------|------------------------------------|------------------------------------|-----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                     | Start workflow manually             | None                        | Download Resume               |                                                                                                                              |
| Download Resume            | Google Drive                       | Download candidate resume PDF       | When clicking ‘Test workflow’ | PDF-to-Image API             | See Sticky Note1: Download candidate resume with hidden prompt explanation and Google Drive usage.                            |
| Sticky Note1               | Sticky Note                       | Documentation                      | None                        | None                         | Explains downloading resume from Google Drive and hidden prompt scenario.                                                   |
| PDF-to-Image API           | HTTP Request                      | Convert PDF to image via Stirling PDF API | Download Resume             | Resize Converted Image        | See Sticky Note2 and Sticky Note4: PDF to image conversion and data privacy warning.                                         |
| Resize Converted Image     | Edit Image                       | Resize image to 75% of original size | PDF-to-Image API            | Candidate Resume Analyser     |                                                                                                                              |
| Sticky Note2               | Sticky Note                       | Documentation                      | None                        | None                         | Explains PDF to image necessity and Stirling PDF usage.                                                                     |
| Sticky Note4               | Sticky Note                       | Documentation                      | None                        | None                         | Data privacy warning about public Stirling PDF API use.                                                                      |
| Google Gemini Chat Model   | LangChain Google Gemini Chat Model | Multimodal LLM model provider      | None                        | Candidate Resume Analyser (as language model) |                                                                                                                              |
| Candidate Resume Analyser  | LangChain Chain LLM               | Analyze resume image for qualification | Resize Converted Image, Google Gemini Chat Model | Should Proceed To Stage 2?, Structured Output Parser | See Sticky Note3: Explains multimodal LLM usage for resume parsing and hidden prompt mitigation.                              |
| Structured Output Parser   | LangChain Output Parser Structured | Parse AI output into JSON structure | Candidate Resume Analyser   | Candidate Resume Analyser (parsed output) |                                                                                                                              |
| Should Proceed To Stage 2? | If                               | Evaluate candidate qualification   | Candidate Resume Analyser   | (Conditional branch output)  |                                                                                                                              |
| Sticky Note3               | Sticky Note                       | Documentation                      | None                        | None                         | Explains multimodal LLM and rationale for image-based resume processing.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing.

2. **Create Google Drive Node ("Download Resume")**  
   - Type: Google Drive  
   - Operation: Download  
   - Set File ID to the candidate resume PDF file (example: `1MORAdeev6cMcTJBV2EYALAwll8gCDRav`)  
   - Configure Google Drive OAuth2 credentials.  
   - Connect the Manual Trigger node output to this node’s input.

3. **Create HTTP Request Node ("PDF-to-Image API")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://stirlingpdf.io/api/v1/convert/pdf/img`  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - `fileInput`: Attach binary data from "Download Resume" node.  
     - `imageFormat`: `jpg`  
     - `singleOrMultiple`: `single`  
     - `dpi`: `300`  
   - Connect "Download Resume" node output to this node’s input.

4. **Create Edit Image Node ("Resize Converted Image")**  
   - Type: Edit Image  
   - Operation: Resize  
   - Resize Option: Percent  
   - Width: 75  
   - Height: 75  
   - Connect "PDF-to-Image API" output to this node.

5. **Create LangChain Google Gemini Chat Model Node ("Google Gemini Chat Model")**  
   - Type: LangChain Google Gemini Chat Model  
   - Model Name: `models/gemini-1.5-pro-latest`  
   - Configure Google Palm API credentials with valid API key.  
   - No input connection (used as language model in next node).

6. **Create LangChain Chain LLM Node ("Candidate Resume Analyser")**  
   - Type: Chain LLM  
   - Prompt Type: Define  
   - Messages:  
     - Plain text: "Assess the given Candidate Resume for the role of Plumber. Determine if the candidate's skills match the role and if they qualify for an in-person interview."  
     - HumanMessagePromptTemplate with message type set to `imageBinary` (to accept image input).  
   - Enable output parser.  
   - Connect:  
     - Image input from "Resize Converted Image" node.  
     - Language model: select "Google Gemini Chat Model" node.  

7. **Create LangChain Output Parser Structured Node ("Structured Output Parser")**  
   - Type: Output Parser Structured  
   - JSON Schema Example:  
     ```json
     {
       "is_qualified": true,
       "reason": ""
     }
     ```  
   - Connect AI output of "Candidate Resume Analyser" node to this parser’s input.

8. **Create If Node ("Should Proceed To Stage 2?")**  
   - Type: If  
   - Condition: Check if `{{$json["output"]["is_qualified"]}}` is boolean `true` (strict type).  
   - Connect main output of "Candidate Resume Analyser" node to this node.

9. **Connect Workflow Start to End:**  
   - Manual Trigger → Download Resume → PDF-to-Image API → Resize Converted Image → Candidate Resume Analyser → Structured Output Parser  
   - Candidate Resume Analyser → Should Proceed To Stage 2?

10. **Add Sticky Notes (Optional but Recommended):**  
    - Add contextual sticky notes to explain each block as per the documented content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow combats AI resume screening bypass attempts by converting PDF resumes into images and processing them with multimodal LLMs.                                                                                                         | Overall workflow rationale.                                                                                      |
| The example candidate resume with hidden prompt is downloadable here: https://drive.google.com/file/d/1MORAdeev6cMcTJBV2EYALAwll8gCDRav/view?usp=sharing                                                                                            | Candidate resume with hidden prompt used in this demo.                                                          |
| Stirling PDF API is used for PDF-to-image conversion; recommended to self-host for data privacy. Public API is used here for demonstration only.                                                                                                   | https://github.com/Stirling-Tools/Stirling-PDF                                                                  |
| Google Gemini 1.5 Pro model is used as an example multimodal LLM. GPT-4 can also be used for similar use cases.                                                                                                                                      | Google Gemini API key required; alternatively GPT4.                                                              |
| For help and community support, join the n8n Discord (https://discord.com/invite/XPKeKXeB7d) or ask in the n8n Forum (https://community.n8n.io/).                                                                                                   | Community support channels.                                                                                       |
| Multimodal LLMs can accept binary inputs such as images and audio, enabling richer data understanding beyond text-only models.                                                                                                                     | Sticky Note3 explanation.                                                                                        |
| The hidden prompt embedded in the resume uses white font to evade text extraction, but becomes visible and analyzable when converted to an image and processed by the vision-capable LLM.                                                           | Workflow core concept.                                                                                            |