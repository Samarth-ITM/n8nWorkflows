Convert Radiology Images to Patient-Friendly Reports with GPT-4 Vision & PDF Email

https://n8nworkflows.xyz/workflows/convert-radiology-images-to-patient-friendly-reports-with-gpt-4-vision---pdf-email-7275


# Convert Radiology Images to Patient-Friendly Reports with GPT-4 Vision & PDF Email

### 1. Workflow Overview

This workflow automates the conversion of radiology images into detailed, patient-friendly medical reports using AI vision capabilities (GPT-4 Vision). It targets medical imaging providers, radiologists, or healthcare systems wishing to streamline report generation and communication with patients. The workflow includes the following logical blocks:

- **1.1 Input Reception and Extraction:** Listens for incoming radiology image data and extracts relevant patient and scan details.
- **1.2 AI-Powered Image Analysis:** Sends the image and scan metadata to GPT-4 Vision for expert analysis and patient-friendly explanation.
- **1.3 AI Response Processing:** Parses and structures the AI’s textual analysis into report sections.
- **1.4 PDF Report Generation:** Creates an HTML report template and converts it into a PDF document.
- **1.5 Report Storage and Notification:** Saves report metadata and PDF link to Google Sheets, emails the PDF report to the patient, and returns a success response.
- **1.6 Workflow Metadata and Setup Instructions:** Provides sticky notes describing workflow features and setup requirements.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Extraction

**Overview:**  
This block receives radiology image upload requests via a webhook, then extracts and standardizes patient and scan information for subsequent processing.

**Nodes Involved:**  
- Upload Image Trigger  
- Extract Image Data

**Node Details:**  

- **Upload Image Trigger**  
  - *Type:* Webhook Trigger  
  - *Role:* Entry point that accepts POST requests containing radiology image data and metadata.  
  - *Configuration:* HTTP POST method on path `radiology-upload`.  
  - *Inputs:* External POST request with JSON payload including patient_name, patient_id, scan_type, body_part, image_url/base64, doctor_name, scan_date, urgency.  
  - *Outputs:* Forwards raw data to extraction node.  
  - *Failures:* Missing or malformed payload; webhook authentication not configured (public endpoint).  

- **Extract Image Data**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Parses incoming JSON, applies defaults if fields are missing, structures patient and scan info for downstream use.  
  - *Key Variables:* patient_name, patient_id, scan_type, body_part, image_url, image_base64, doctor_name, scan_date, urgency. Defaults provided for missing fields (e.g., 'Patient', 'X-Ray', current date).  
  - *Inputs:* JSON from webhook.  
  - *Outputs:* Standardized patient and scan data object.  
  - *Failures:* Expression errors if input missing expected fields, but fallback defaults mitigate this risk.

---

#### 2.2 AI-Powered Image Analysis

**Overview:**  
This block submits the radiology image and context to OpenAI’s GPT-4 Vision model to generate a patient-friendly interpretation of the scan.

**Nodes Involved:**  
- Analyze Radiology Image with AI

**Node Details:**  

- **Analyze Radiology Image with AI**  
  - *Type:* HTTP Request Node  
  - *Role:* Calls OpenAI API chat completion endpoint using GPT-4 Vision to analyze the image and describe findings in plain language.  
  - *Configuration:*  
    - URL: `https://api.openai.com/v1/chat/completions`  
    - Method: POST  
    - Headers: Authorization with OpenAI API key, Content-Type: application/json  
    - Body: JSON with system prompt defining radiology expert role, user prompt including scan type, body part, and image (base64 or URL).  
    - Model: `gpt-4-vision-preview`  
    - Temperature: 0.3 (low randomness for consistent results)  
    - Max tokens: 1000  
  - *Inputs:* Standardized patient and scan data.  
  - *Outputs:* Raw AI response JSON containing message content with analysis text.  
  - *Failures:* API key invalid, rate limits, network timeouts, malformed image data, unsupported image formats, response parsing errors.

---

#### 2.3 AI Response Processing

**Overview:**  
Processes the AI-generated text to extract structured sections: findings, explanations, and next steps. Prepares data for report generation.

**Nodes Involved:**  
- Process AI Analysis

**Node Details:**  

- **Process AI Analysis**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Parses AI text response by splitting on paragraph breaks, categorizes segments into findings, explanation, and next steps based on keywords. Fallback logic merges half of the text for findings and explanation if categorization fails.  
  - *Key Expressions:*  
    - `aiResponse.choices[0].message.content` to get AI text  
    - String methods to detect keywords like 'findings', 'means', 'next'  
  - *Inputs:* AI response JSON, patient data JSON from Extract Image Data node.  
  - *Outputs:* JSON with combined patient info and structured report fields: ai_analysis, findings, explanation, next_steps, report_generated timestamp, confidence_note disclaimer.  
  - *Failures:* Unexpected AI response format, missing message content, text parsing issues.

---

#### 2.4 PDF Report Generation

**Overview:**  
Creates a styled HTML report based on processed data, then converts it to PDF format via an external API.

**Nodes Involved:**  
- Generate PDF Report  
- Convert to PDF  
- Wait For PDF

**Node Details:**  

- **Generate PDF Report**  
  - *Type:* Code Node  
  - *Role:* Builds a detailed HTML template embedding patient info, AI findings, explanation, next steps, and disclaimers with styled sections using inline CSS. Also creates a dynamic filename based on patient name and scan date.  
  - *Inputs:* Processed AI analysis and patient data JSON.  
  - *Outputs:* JSON with html_report string and report_filename.  
  - *Failures:* Template rendering errors if data fields missing or contain invalid characters for filenames.

- **Convert to PDF**  
  - *Type:* HTTP Request Node  
  - *Role:* Sends HTML report to a third-party HTML-to-PDF conversion API, specifying A4 format and margins, to generate a downloadable PDF file.  
  - *Configuration:*  
    - URL: `https://api.html-css-to-pdf.com/v1/generate`  
    - Method: POST  
    - Headers: Authorization with PDF API key, Content-Type: application/json  
    - Body: JSON with HTML content and layout options.  
  - *Inputs:* html_report from previous node.  
  - *Outputs:* JSON including `download_url` for the PDF.  
  - *Failures:* API limits, invalid HTML, network issues, authorization errors.

- **Wait For PDF**  
  - *Type:* Wait Node  
  - *Role:* Delays workflow to ensure PDF generation completes and download URL is ready before next steps.  
  - *Inputs:* Triggered after PDF conversion.  
  - *Outputs:* Passes data to database storage.  
  - *Failures:* Workflow timeout or misconfiguration delaying downstream processes.

---

#### 2.5 Report Storage and Notification

**Overview:**  
Stores report metadata and PDF link in Google Sheets, emails the PDF report to the patient, and returns a success response to the webhook caller.

**Nodes Involved:**  
- Save Report to Database  
- Email Report to Patient  
- Return Response

**Node Details:**  

- **Save Report to Database**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends a new row to a Google Sheets document logging report details including PDF URL, patient and scan data, timestamp, and status.  
  - *Configuration:*  
    - Operation: Append  
    - Sheet Name: `Reports_Log`  
    - Document ID: User must replace `YOUR_REPORTS_SHEET_ID` with actual Google Sheets ID  
    - Authentication: Google Service Account credentials  
    - Mapped Columns: pdf_url, body_part, scan_date, scan_type, timestamp, patient_id, patient_name, report_status  
  - *Inputs:* PDF download URL, report metadata.  
  - *Outputs:* Passes data to next node.  
  - *Failures:* Invalid credentials, sheet access issues, incorrect document ID, API quota exceeded.

- **Email Report to Patient**  
  - *Type:* Gmail Node  
  - *Role:* Sends an email to the patient with a friendly message, key scan details, findings summary, and attaches the PDF report.  
  - *Configuration:*  
    - Recipient: `patient_email` from webhook input or fallback to `patient@example.com`  
    - Subject: Includes patient name  
    - HTML Message: Personalized with scan details and disclaimers, referencing doctor’s name.  
    - Authentication: OAuth2 Gmail credentials  
  - *Inputs:* Patient email, report data, PDF attachment link.  
  - *Outputs:* Passes data to final response node.  
  - *Failures:* Invalid email address, Gmail API quota, OAuth token expiration, attachment size limits.

- **Return Response**  
  - *Type:* Code Node  
  - *Role:* Sends a JSON success response back to the original webhook caller summarizing report generation status, patient name, scan type, PDF URL, findings snippet, and disclaimers.  
  - *Inputs:* Data from report generation and PDF conversion nodes.  
  - *Outputs:* Final output of the workflow.  
  - *Failures:* Unexpected missing data or malformed output.

---

#### 2.6 Workflow Metadata and Setup Instructions

**Overview:**  
Provides visual documentation within the workflow for users to understand features and prerequisites.

**Nodes Involved:**  
- Workflow Overview (Sticky Note)  
- Required Setup (Sticky Note)

**Node Details:**  

- **Workflow Overview**  
  - *Type:* Sticky Note  
  - *Content:* Describes key workflow features such as AI-powered analysis, PDF report generation, email delivery, database logging, and webhook trigger usage.  
  - *Role:* Informational only.  

- **Required Setup**  
  - *Type:* Sticky Note (Colored)  
  - *Content:* Lists required credentials and configuration steps: OpenAI API, PDF conversion API, Gmail OAuth2, Google Sheets setup, and updating the sheet ID placeholder.  
  - *Role:* Informational only.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                              | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                    |
|----------------------------|---------------------|----------------------------------------------|---------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| Upload Image Trigger        | Webhook Trigger     | Receives HTTP POST with radiology image data | —                         | Extract Image Data        |                                                                                                               |
| Extract Image Data          | Code                | Parses and standardizes input data            | Upload Image Trigger       | Analyze Radiology Image with AI |                                                                                                               |
| Analyze Radiology Image with AI | HTTP Request       | Calls OpenAI GPT-4 Vision for image analysis  | Extract Image Data         | Process AI Analysis       |                                                                                                               |
| Process AI Analysis         | Code                | Parses AI text into structured report sections| Analyze Radiology Image with AI | Generate PDF Report      |                                                                                                               |
| Generate PDF Report         | Code                | Creates HTML report template with data        | Process AI Analysis        | Convert to PDF            |                                                                                                               |
| Convert to PDF              | HTTP Request        | Converts HTML report to PDF via external API  | Generate PDF Report        | Wait For PDF              |                                                                                                               |
| Wait For PDF                | Wait                | Delays workflow post PDF generation            | Convert to PDF             | Save Report to Database   |                                                                                                               |
| Save Report to Database     | Google Sheets       | Logs report metadata and PDF link              | Wait For PDF               | Email Report to Patient   |                                                                                                               |
| Email Report to Patient     | Gmail               | Sends report PDF and summary to patient email | Save Report to Database    | Return Response           |                                                                                                               |
| Return Response            | Code                | Sends final success response back to caller   | Email Report to Patient    | —                        |                                                                                                               |
| Workflow Overview           | Sticky Note         | Describes workflow purpose and features        | —                         | —                        | Describes AI analysis, PDF generation, email delivery, logging, webhook trigger                                |
| Required Setup              | Sticky Note         | Lists credentials and configuration needed    | —                         | —                        | Setup requires OpenAI API, PDF API, Gmail OAuth2, Google Sheets, and updating YOUR_REPORTS_SHEET_ID           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Name: "Upload Image Trigger"  
   - HTTP Method: POST  
   - Path: `radiology-upload`  
   - Purpose: Receive radiology image data and metadata.

2. **Add Code Node to Extract Input Data**  
   - Name: "Extract Image Data"  
   - JavaScript code to extract patient_name, patient_id, scan_type, body_part, image_url/base64, doctor_name, scan_date, urgency, with defaults.  
   - Connect output of webhook to this node.

3. **Add HTTP Request Node to Call OpenAI GPT-4 Vision**  
   - Name: "Analyze Radiology Image with AI"  
   - URL: `https://api.openai.com/v1/chat/completions`  
   - Method: POST  
   - Headers: Authorization with `Bearer {{ $credentials.openaiApi.apiKey }}`, Content-Type: application/json  
   - Body (JSON):  
     ```json
     {
       "model": "gpt-4-vision-preview",
       "messages": [
         {
           "role": "system",
           "content": "You are a radiology expert who explains medical scans in simple, patient-friendly language..."
         },
         {
           "role": "user",
           "content": [
             {
               "type": "text",
               "text": "Please analyze this {{ $json.scan_type }} scan of the {{ $json.body_part }} and explain the findings in patient-friendly terms."
             },
             {
               "type": "image_url",
               "image_url": {
                 "url": "{{ $json.image_base64 ? 'data:image/jpeg;base64,' + $json.image_base64 : $json.image_url }}"
               }
             }
           ]
         }
       ],
       "max_tokens": 1000,
       "temperature": 0.3
     }
     ```
   - Connect output of Extract Image Data node here.

4. **Add Code Node to Process AI Response**  
   - Name: "Process AI Analysis"  
   - JavaScript to parse AI text response into findings, explanation, next steps; fallback if parsing fails.  
   - Combine with patient data from Extract Image Data.  
   - Connect output of AI node here.

5. **Add Code Node to Generate HTML Report**  
   - Name: "Generate PDF Report"  
   - Write JavaScript that builds a detailed HTML report embedding all data fields (patient info, findings, explanation, next steps, disclaimers) with inline CSS styling.  
   - Generate a filename based on patient name and scan date.  
   - Connect output of Process AI Analysis node here.

6. **Add HTTP Request Node to Convert HTML to PDF**  
   - Name: "Convert to PDF"  
   - URL: `https://api.html-css-to-pdf.com/v1/generate`  
   - Method: POST  
   - Headers: Authorization with `Bearer {{ $credentials.pdfApi.apiKey }}`, Content-Type: application/json  
   - Body JSON: includes HTML content and page format (A4, margins).  
   - Connect output of Generate PDF Report here.

7. **Add Wait Node to Pause Until PDF is Ready**  
   - Name: "Wait For PDF"  
   - No parameters needed; default short wait to ensure PDF generation completion.  
   - Connect output of Convert to PDF here.

8. **Add Google Sheets Node to Save Report Metadata**  
   - Name: "Save Report to Database"  
   - Operation: Append  
   - Sheet Name: `Reports_Log`  
   - Document ID: Replace placeholder with actual Google Sheets document ID  
   - Map columns: pdf_url, body_part, scan_date, scan_type, timestamp (current ISO datetime), patient_id, patient_name, report_status = "Generated Successfully"  
   - Use Google Service Account credentials configured for Sheets API.  
   - Connect output of Wait For PDF node here.

9. **Add Gmail Node to Email Report to Patient**  
   - Name: "Email Report to Patient"  
   - Recipient: Use patient_email from input, fallback to `patient@example.com`  
   - Subject: Include patient name and scan type  
   - HTML Body: Personalized message with scan details, key findings, disclaimers, PDF attachment link  
   - Authentication: OAuth2 Gmail credentials  
   - Connect output of Save Report to Database node here.

10. **Add Code Node to Return Final Response**  
    - Name: "Return Response"  
    - Return JSON status "success" with patient name, scan type, body part, report generated timestamp, PDF URL, summary of findings, next steps, and disclaimer.  
    - Connect output of Email Report to Patient node here.

11. **Add Sticky Notes for Documentation**  
    - Name: "Workflow Overview" with content describing features and usage instructions.  
    - Name: "Required Setup" listing required credentials and configuration steps.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                      |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Workflow triggered by POST to `/radiology-upload` with image data and patient metadata.          | Webhook trigger path.                                               |
| Requires OpenAI API key with access to GPT-4 Vision model.                                       | OpenAI API credentials setup.                                      |
| Uses third-party HTML-to-PDF API; requires API key for PDF generation.                           | `https://api.html-css-to-pdf.com` API key setup.                   |
| Emails sent via Gmail OAuth2 credentials for secure access to sending mailbox.                   | Gmail OAuth2 credential configuration.                             |
| Google Sheets used as a lightweight database for report logs; replace `YOUR_REPORTS_SHEET_ID`.   | Google Sheets document setup and Service Account configuration.    |
| AI-generated reports are informational only; patients instructed to consult healthcare providers.| Important medical disclaimer included in report and email content. |

---

**Disclaimer:**  
The provided workflow is an automated n8n integration designed for legal, public data processing. It does not replace professional medical advice or diagnosis. Users must secure appropriate credentials and comply with data privacy regulations (e.g., HIPAA, GDPR) when handling patient data.