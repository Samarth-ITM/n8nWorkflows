Convert Workout Plan PDFs to Hevy App Routines with Gemini AI

https://n8nworkflows.xyz/workflows/convert-workout-plan-pdfs-to-hevy-app-routines-with-gemini-ai-6527


# Convert Workout Plan PDFs to Hevy App Routines with Gemini AI

### 1. Workflow Overview

This workflow automates the conversion of workout plan PDFs into structured workout routines compatible with the Hevy app, leveraging AI-powered OCR and semantic matching. It is designed for users who want to digitize their physical or scanned workout plans and seamlessly import them into their Hevy account without manual data entry.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Handles user input via a form where a workout plan PDF or image file is uploaded.
- **1.2 File Preparation & OCR Extraction:** Converts the uploaded file to Base64 and sends it to an AI model for OCR text extraction.
- **1.3 Hevy Exercise Data Retrieval:** Fetches the full list of exercises available in the Hevy app to provide context for AI matching.
- **1.4 AI Processing & Routine Structuring:** Uses Google’s Gemini AI to analyze OCR text, perform semantic exercise matching, and output a structured JSON adhering to Hevy’s routine schema.
- **1.5 Routine Creation in Hevy:** Sends the structured JSON to the Hevy API to create the routine in the user’s account.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow by receiving a user submission of a workout plan file via a web form.
- **Nodes Involved:**  
  - On form submission
  - Sticky Note (overview comment)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger node  
    - Role: Receives file upload from a web form titled "Upload Training Plan" with a required file field accepting `.jpg, .png, .pdf, .jpeg`.  
    - Configuration: Single file upload, required field.  
    - Inputs: Triggered externally by form submission.  
    - Outputs: Binary file data passed to next node.  
    - Edge Cases: File type other than accepted formats; upload failures.  
    - Sticky Note: Explains the workflow’s purpose and setup instructions.

  - **Sticky Note**  
    - Provides high-level explanation of the workflow’s purpose and setup.

---

#### 2.2 File Preparation & OCR Extraction

- **Overview:** Converts the uploaded workout plan file into Base64 text, then sends it to an AI model for OCR to extract text content.
- **Nodes Involved:**  
  - Convert PDF to Base64  
  - Extract Text from PDF via AI  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**

  - **Convert PDF to Base64**  
    - Type: Extract From File node  
    - Role: Converts binary file data into a Base64-encoded string required by the AI API.  
    - Configuration: Operation set to "binaryToProperty" to convert binary file to JSON property.  
    - Inputs: File binary data from form submission.  
    - Outputs: JSON with `data` property containing Base64 string.  
    - Edge Cases: Large files causing conversion delay; unsupported binary formats.  
    - Sticky Note: Notes this conversion is needed to send file content to AI.

  - **Extract Text from PDF via AI**  
    - Type: HTTP Request node  
    - Role: Sends Base64-encoded image/pdf to OpenRouter API (Google Gemini 2.5 flash model) to perform OCR and extract text.  
    - Configuration: POST request to `https://openrouter.ai/api/v1/chat/completions`, with JSON body including the Base64 image data in the specified format. Uses HTTP Header Auth credential for authentication.  
    - Inputs: Base64 file data from previous node.  
    - Outputs: AI response JSON containing extracted text.  
    - Edge Cases: API rate limits, authentication errors, OCR inaccuracies due to poor image quality.  
    - Sticky Note: Mentions this node requires OpenRouter.ai API credentials.

---

#### 2.3 Hevy Exercise Data Retrieval

- **Overview:** Retrieves all exercise templates from the user’s Hevy account across paginated responses and consolidates the data into one list.
- **Nodes Involved:**  
  - Get Exercise List from Hevy  
  - Format Exercise Names  
  - Combine Exercise List  
  - Sticky Note3

- **Node Details:**

  - **Get Exercise List from Hevy**  
    - Type: HTTP Request node  
    - Role: Sends authenticated GET requests to Hevy API’s `/v1/exercise_templates` endpoint with pagination to fetch all exercises.  
    - Configuration: Uses pagination with max 10 pages, 100 items per page, stops on HTTP 404 status. Authentication via HTTP Header Auth credential for Hevy.  
    - Inputs: Triggered after OCR extraction node.  
    - Outputs: Multiple paginated JSON responses with exercise templates.  
    - Edge Cases: Pagination errors, API rate limits, auth failures.  
    - Sticky Note: Requires Hevy API credentials.

  - **Format Exercise Names**  
    - Type: Set node  
    - Role: Extracts and assigns the combined exercise templates to a single JSON array property `exercise_templates`.  
    - Configuration: Assigns `$json.exercise_templates` array.  
    - Inputs: Exercise list pages from previous node.  
    - Outputs: JSON object with exercise list array.  
    - Edge Cases: Empty responses, malformed JSON.

  - **Combine Exercise List**  
    - Type: Aggregate node  
    - Role: Merges multiple exercise lists from paginated responses into a single aggregated array.  
    - Configuration: Merge lists enabled on `exercise_templates` field.  
    - Inputs: Multiple exercise template arrays from Format Exercise Names node.  
    - Outputs: Single array with all exercises combined.  
    - Edge Cases: Missing or partial pagination data.

---

#### 2.4 AI Processing & Routine Structuring

- **Overview:** Uses Google’s Gemini AI model with a detailed prompt and JSON schema to analyze the OCR-extracted text, match exercises semantically against Hevy’s exercise list, and output a structured routine JSON compliant with Hevy’s API.
- **Nodes Involved:**  
  - google/gemini-2.5-flash  
  - Structured Output Parser  
  - Match Exercises & Structure Routine  
  - Sticky Note4

- **Node Details:**

  - **google/gemini-2.5-flash**  
    - Type: LangChain LLM Chat OpenRouter node  
    - Role: Provides the AI language model environment to run the prompt chain.  
    - Configuration: Model set to `google/gemini-2.5-flash`, using OpenRouter API credential.  
    - Inputs: None directly; connected as AI language model for downstream chain node.  
    - Outputs: AI-generated content passed to output parser.  
    - Edge Cases: Model availability, API limits, authentication errors.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser node  
    - Role: Validates and parses AI output according to a strict JSON schema defining the Hevy routine structure: routine title, folder ID, notes, exercises array with sets and metadata.  
    - Configuration: Manual schema defined with required fields for routine and exercises, including nested properties for sets and exercise IDs.  
    - Inputs: Raw AI text from Gemini model node.  
    - Outputs: Structured JSON object matching Hevy’s API requirements.  
    - Edge Cases: Parsing failures if AI output deviates from schema, JSON validation errors.

  - **Match Exercises & Structure Routine**  
    - Type: LangChain Chain LLM node  
    - Role: Contains the detailed prompt and instructions for the AI to analyze OCR text and exercise data, performing semantic matching and building the final routine JSON.  
    - Configuration: Prompt includes goal, context, input data placeholders for OCR text and base exercise data, detailed extraction rules (e.g., set parsing, rest times, superset IDs, tempo notes), and output instructions to produce only valid JSON. Output parser enabled.  
    - Inputs: Receives OCR text, combined exercise list, AI model environment, output parser node.  
    - Outputs: Validated routine JSON.  
    - Edge Cases: Semantic matching errors, incomplete OCR text, ambiguous exercise names, incorrect set parsing, missing rest or tempo data.  
    - Sticky Note: Emphasizes this node as the workflow’s core for AI processing.

---

#### 2.5 Routine Creation in Hevy

- **Overview:** Takes the structured routine JSON and sends it to the Hevy API to create the workout routine in the user’s account.
- **Nodes Involved:**  
  - Create Hevy Routine  
  - Sticky Note8

- **Node Details:**

  - **Create Hevy Routine**  
    - Type: HTTP Request node  
    - Role: Sends a POST request to Hevy’s `/v1/routines` endpoint with the AI-generated routine JSON in the request body.  
    - Configuration: POST method, JSON body set dynamically from the AI output’s `output` property. Authentication via Hevy API HTTP Header Auth credential.  
    - Inputs: Structured routine JSON from AI processing node.  
    - Outputs: API response confirming routine creation or error messages.  
    - Edge Cases: API errors (validation, auth, rate limits), malformed JSON, network failures.  
    - Sticky Note: Describes this as the final node creating the workout routine.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                                 | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                                                             |
|----------------------------|---------------------------------------|------------------------------------------------|-----------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger                          | Receives workout plan file upload               | (Trigger)                   | Convert PDF to Base64            | ## Scan Any Workout Plan into the Hevy App with AI. This workflow extracts a workout plan from a PDF, uses AI to match exercises...      |
| Convert PDF to Base64      | Extract From File                     | Converts uploaded file to Base64 text            | On form submission          | Extract Text from PDF via AI     | This node converts the incoming PDF file into a Base64 text string. This format is required to send the file content to the AI model.    |
| Extract Text from PDF via AI| HTTP Request                         | Sends Base64 file to AI for OCR text extraction | Convert PDF to Base64        | Get Exercise List from Hevy      | This node sends the PDF content to an AI model to perform OCR and extract all the text from the document. Requires OpenRouter.ai API.    |
| Get Exercise List from Hevy| HTTP Request                         | Fetches exercise templates from Hevy API         | Extract Text from PDF via AI | Format Exercise Names            | These nodes fetch all available exercises from your Hevy account and combine them into a single list. Requires https://api.hevyapp.com/docs/ credentials. |
| Format Exercise Names      | Set                                  | Formats and assigns exercise templates array     | Get Exercise List from Hevy | Combine Exercise List            | These nodes fetch all available exercises from your Hevy account and combine them into a single list. Requires https://api.hevyapp.com/docs/ credentials. |
| Combine Exercise List      | Aggregate                           | Merges paginated exercise lists into one array   | Format Exercise Names        | Match Exercises & Structure Routine | These nodes fetch all available exercises from your Hevy account and combine them into a single list. Requires https://api.hevyapp.com/docs/ credentials. |
| google/gemini-2.5-flash   | LangChain LLM Chat OpenRouter       | Provides AI model environment for processing     | (Internal AI node)          | Match Exercises & Structure Routine | This is the core of the workflow. It uses Google's Gemini model to read text, compare exercises, and create structured JSON.            |
| Structured Output Parser   | LangChain Output Parser              | Parses AI output into structured JSON            | google/gemini-2.5-flash     | Match Exercises & Structure Routine | This is the core of the workflow. It uses Google's Gemini model to read text, compare exercises, and create structured JSON.            |
| Match Exercises & Structure Routine | LangChain Chain LLM           | AI prompt processing, semantic matching, JSON output | Combine Exercise List, Structured Output Parser, google/gemini-2.5-flash | Create Hevy Routine             | This is the core of the workflow. It uses Google's Gemini model to read text, compare exercises, and create structured JSON.            |
| Create Hevy Routine        | HTTP Request                        | Sends structured routine JSON to Hevy API         | Match Exercises & Structure Routine | (End)                        | This final node takes the structured data from the AI and makes a POST request to the Hevy API, creating the complete workout routine.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form title: "Upload Training Plan"  
   - Add a single required file upload field accepting `.jpg, .png, .pdf, .jpeg`  
   - This node listens for user uploads.

2. **Add an Extract From File Node**  
   - Name: "Convert PDF to Base64"  
   - Operation: `binaryToProperty`  
   - Connect input from the Form Trigger node’s binary file output  
   - This converts the uploaded file to a Base64 string.

3. **Add an HTTP Request Node for OCR Extraction**  
   - Name: "Extract Text from PDF via AI"  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Configure JSON body as follows (use expressions for Base64 data):  
     ```json
     {
       "model": "google/gemini-2.5-flash",
       "messages": [
         {
           "role": "user",
           "content": [
             { "type": "text", "text": "Please perform OCR on this document and provide the full extracted content in your response" },
             { "type": "image_url", "image_url": { "url": "data:image/jpeg;base64,{{ $json.data }}" } }
           ]
         }
       ]
     }
     ```
   - Authentication: HTTP Header Auth (OpenRouter API)  
   - Connect input from "Convert PDF to Base64" node.

4. **Add an HTTP Request Node to Fetch Hevy Exercises**  
   - Name: "Get Exercise List from Hevy"  
   - Method: GET  
   - URL: `https://api.hevyapp.com/v1/exercise_templates`  
   - Add Query Parameter: `pageSize=100`  
   - Enable Pagination: Limit to 10 pages, stop on 404 status  
   - Authentication: HTTP Header Auth (Hevy API)  
   - Connect input from "Extract Text from PDF via AI".

5. **Add a Set Node to Format Exercise Names**  
   - Name: "Format Exercise Names"  
   - Assign variable `exercise_templates` to `{{$json["exercise_templates"]}}` (or equivalent depending on pagination output)  
   - Connect input from "Get Exercise List from Hevy".

6. **Add an Aggregate Node to Combine Exercise Lists**  
   - Name: "Combine Exercise List"  
   - Operation: Merge Lists enabled on `exercise_templates` field  
   - Connect input from "Format Exercise Names".

7. **Add a LangChain LLM Chat Node (OpenRouter)**  
   - Name: "google/gemini-2.5-flash"  
   - Select Model: `google/gemini-2.5-flash`  
   - Use OpenRouter credentials  
   - No direct input, used as AI environment for chain node.

8. **Add a LangChain Output Parser Node**  
   - Name: "Structured Output Parser"  
   - Schema Type: Manual  
   - Input Schema: Use the JSON schema as defined in the workflow, describing the `routine` object structure with exercises, sets, and metadata.  
   - Connect input from the "google/gemini-2.5-flash" node’s output.

9. **Add a LangChain Chain LLM Node**  
   - Name: "Match Exercises & Structure Routine"  
   - Prompt: Include the full detailed prompt as in the workflow, referencing OCR text and base exercise data placeholders, with instructions for semantic matching, set parsing, supersets, tempo, and output JSON only.  
   - Enable Output Parser and link it to "Structured Output Parser" node.  
   - Connect inputs from:  
     - "Combine Exercise List" (exercise data)  
     - "Structured Output Parser" (output parser)  
     - "google/gemini-2.5-flash" (AI model environment)  
     - OCR text from "Extract Text from PDF via AI" node (mapped inside prompt).

10. **Add an HTTP Request Node to Create Hevy Routine**  
    - Name: "Create Hevy Routine"  
    - Method: POST  
    - URL: `https://api.hevyapp.com/v1/routines`  
    - Body: JSON set dynamically to `{{$json.output}}` from the AI chain node’s output  
    - Authentication: HTTP Header Auth (Hevy API)  
    - Connect input from "Match Exercises & Structure Routine".

11. **Optional: Add Sticky Notes**  
    - Add notes at various points explaining node roles and setup instructions as per the sticky notes in the original workflow.

12. **Credentials Setup**  
    - OpenRouter API credentials for AI nodes.  
    - Hevy API HTTP Header Auth credentials for Hevy API nodes.

13. **Activate and Test**  
    - Activate the workflow.  
    - Test by submitting a workout plan PDF or image through the form.  
    - Verify the routine appears correctly in the Hevy app.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow extracts a workout plan from a PDF, uses AI to match exercises to Hevy app templates, and creates the routine automatically. | Sticky Note on Input Reception block.                                                              |
| Requires OpenRouter API credentials for AI-powered OCR and language processing.                                | OpenRouter.ai API: https://openrouter.ai/                                                          |
| Requires Hevy API credentials to authenticate requests fetching exercises and creating routines.             | Hevy API docs: https://api.hevyapp.com/docs/                                                       |
| The core AI prompt includes detailed instructions to handle imperfect OCR data, semantic exercise matching, rest times, supersets, and tempo notes. | Sticky Note on AI Processing block.                                                                |
| Final node creates the workout routine in the Hevy app using the structured JSON output from AI.              | Sticky Note on Routine Creation block.                                                             |

---

This structured documentation enables understanding, reproduction, and modification of the workflow, with attention to potential failure points such as API authentication errors, OCR inaccuracies, and AI output parsing issues. It also preserves all sticky note content for contextual guidance and resource links.