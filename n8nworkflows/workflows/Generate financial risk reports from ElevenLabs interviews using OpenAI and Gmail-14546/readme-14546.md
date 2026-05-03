Generate financial risk reports from ElevenLabs interviews using OpenAI and Gmail

https://n8nworkflows.xyz/workflows/generate-financial-risk-reports-from-elevenlabs-interviews-using-openai-and-gmail-14546


# Generate financial risk reports from ElevenLabs interviews using OpenAI and Gmail

# 1. Workflow Overview

This workflow automates the generation of a financial risk report from ElevenLabs interview webhooks. It accepts two possible event types from ElevenLabs:

- `post_call_audio`: the workflow decodes and stores the interview audio as an MP3 in Google Drive.
- `post_call_transcription`: the workflow converts the transcript into readable text, uses OpenAI to extract company identity data and assign a financial risk score, then generates an HTML report and emails it through Gmail.

The workflow is therefore split into two branches triggered by the same webhook endpoint. One branch handles audio archiving, and the other handles transcript-based analysis and report delivery.

## 1.1 Webhook Reception and Event Routing
The workflow starts with a single webhook and uses a Switch node to detect which ElevenLabs payload type has arrived.

## 1.2 Audio Archiving
If the event contains post-call audio, the workflow decodes the Base64 audio payload into binary MP3 format and uploads it to Google Drive.

## 1.3 Transcript Normalization
If the event contains a post-call transcription, the workflow extracts the transcript array and converts it into one continuous conversation text.

## 1.4 Parallel AI Analysis
The normalized transcript is processed in parallel by two AI branches:
- one extracts structured company details,
- the other calculates a financial score, verdict, and rationale as structured JSON.

## 1.5 Report Assembly and Email Delivery
The outputs of both AI branches are merged, transformed into an HTML financial report, and sent by Gmail to a fixed recipient.

---

# 2. Block-by-Block Analysis

## 2.1 Webhook Reception and Event Routing

### Overview
This block receives incoming ElevenLabs webhook events and routes them according to the event type found in the request body. It is the workflow’s only entry point and determines whether the execution follows the audio-storage path or the transcript-analysis path.

### Nodes Involved
- Webhook
- Switch

### Node Details

#### Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point node that exposes an HTTP endpoint for ElevenLabs.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `0a1e32dd-fb2d-450f-a9bd-e69c43bfdecf`
  - No extra webhook options configured
- **Key expressions or variables used:**
  - Incoming payload is later accessed as `$json.body`
- **Input and output connections:**
  - No input
  - Outputs to: `Switch`
- **Version-specific requirements:**
  - Type version `2.1`
  - Requires the workflow to be active for production webhook usage
- **Edge cases or potential failure types:**
  - ElevenLabs may send a payload with an unexpected structure
  - If the webhook is not active or publicly reachable, delivery fails
  - Signature validation is not implemented in this workflow
- **Sub-workflow reference:** None

#### Switch
- **Type and technical role:** `n8n-nodes-base.switch`  
  Conditional router that inspects the webhook payload type.
- **Configuration choices:**
  - Uses rules on `{{$json.body.type}}`
  - Output 1 renamed to `Post Call Audio` when value equals `post_call_audio`
  - Output 2 renamed to `Post call Transcription` when value equals `post_call_transcription`
  - Loose type validation enabled
- **Key expressions or variables used:**
  - `={{ $json.body.type }}`
- **Input and output connections:**
  - Input from: `Webhook`
  - Output 1 to: `Generate MP3`
  - Output 2 to: `Get Transcript`
- **Version-specific requirements:**
  - Type version `3.4`
  - Uses rule syntax from the newer Switch node format
- **Edge cases or potential failure types:**
  - If `body.type` is missing or misspelled, no branch will match
  - Unexpected event types are silently ignored unless an additional fallback route is added
- **Sub-workflow reference:** None

---

## 2.2 Audio Archiving

### Overview
This block handles ElevenLabs audio events. It converts the Base64-encoded audio into n8n binary data, preserves conversation metadata, and uploads the MP3 file to Google Drive for retention and traceability.

### Nodes Involved
- Generate MP3
- Upload audio

### Node Details

#### Generate MP3
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that converts Base64 audio into binary file content.
- **Configuration choices:**
  - Reads `body.data.full_audio`
  - Reads conversation metadata from `body.data`
  - Creates binary property `data`
  - Sets MIME type to `audio/mpeg`
  - Names the file as `conversation_<conversation_id>.mp3`
- **Key expressions or variables used:**
  - `$input.item.json.body.data.full_audio`
  - `$input.item.json.body.data.conversation_id`
  - `$input.item.json.body.data.agent_id`
  - `$input.item.json.body.data.agent_name`
  - `$input.item.json.body.data.user_id`
- **Input and output connections:**
  - Input from: `Switch` audio output
  - Output to: `Upload audio`
- **Version-specific requirements:**
  - Type version `2`
  - Assumes binary mode compatibility with the workflow setting `binaryMode: separate`
- **Edge cases or potential failure types:**
  - Invalid or empty Base64 content causes Buffer conversion issues or unusable files
  - Missing `conversation_id` leads to poor file naming
  - Large payloads may affect memory usage
- **Sub-workflow reference:** None

#### Upload audio
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the generated MP3 binary file to Google Drive.
- **Configuration choices:**
  - File name taken from `{{$binary.data.fileName}}`
  - Target drive: `My Drive`
  - Folder: root folder
  - Uses binary property generated upstream
- **Key expressions or variables used:**
  - `={{$binary.data.fileName}}`
- **Input and output connections:**
  - Input from: `Generate MP3`
  - No downstream node
- **Version-specific requirements:**
  - Type version `3`
  - Requires valid Google Drive OAuth2 credentials
- **Edge cases or potential failure types:**
  - OAuth token expiration or revoked consent
  - Upload permission errors
  - Root folder may not be the intended storage location in production
  - If binary property `data` is missing, upload fails
- **Sub-workflow reference:** None

---

## 2.3 Transcript Normalization

### Overview
This block handles the transcription event from ElevenLabs. It first isolates the transcript array, then converts the structured speaker/message items into a continuous text format suitable for LLM analysis.

### Nodes Involved
- Get Transcript
- Extract Fulltext

### Node Details

#### Get Transcript
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts the transcript array from the webhook payload into a dedicated field.
- **Configuration choices:**
  - Creates a field named `transcript`
  - Field type: array
  - Value: `{{$json.body.data.transcript}}`
- **Key expressions or variables used:**
  - `={{ $json.body.data.transcript }}`
- **Input and output connections:**
  - Input from: `Switch` transcription output
  - Output to: `Extract Fulltext`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - If `body.data.transcript` is absent or not an array, downstream code may fail
  - This node does not preserve all incoming metadata explicitly unless retained automatically in execution context
- **Sub-workflow reference:** None

#### Extract Fulltext
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts the transcript array into one formatted text block.
- **Configuration choices:**
  - Filters transcript items where `message` exists
  - Formats each line as `<role>: <message>`
  - Separates entries with blank lines
  - Returns:
    - `conversation_id`
    - `agent_name`
    - `transcript_text`
- **Key expressions or variables used:**
  - `const transcript = $json.transcript;`
  - `item.message`
  - `item.role`
  - `$json.conversation_id`
  - `$json.agent_name`
- **Input and output connections:**
  - Input from: `Get Transcript`
  - Outputs to:
    - `Information Extractor`
    - `Calculate Rating`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - If transcript items lack `role` or `message`, output quality degrades
  - If `conversation_id` and `agent_name` were not preserved upstream, they may become undefined
  - Empty transcript results in empty `transcript_text`, making AI outputs weak or invalid
- **Sub-workflow reference:** None

---

## 2.4 Parallel AI Analysis

### Overview
This block analyzes the transcript in parallel using OpenAI-backed LangChain nodes. One path extracts company metadata, while the other path scores the interview and produces a structured risk assessment.

### Nodes Involved
- Information Extractor
- OpenAI Chat Model1
- Calculate Rating
- OpenAI Chat Model
- Structured Output Parser

### Node Details

#### Information Extractor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.informationExtractor`  
  LLM-powered structured extraction node.
- **Configuration choices:**
  - Input text: `{{$json.transcript_text}}`
  - System prompt asks the model to extract only relevant information and omit unknown values
  - Required attributes:
    - `company_name`
    - `name`
    - `address`
    - `vat_number`
- **Key expressions or variables used:**
  - `={{ $json.transcript_text }}`
- **Input and output connections:**
  - Main input from: `Extract Fulltext`
  - AI model input from: `OpenAI Chat Model1`
  - Main output to: `Merge`
- **Version-specific requirements:**
  - Type version `1.2`
  - Requires compatible LangChain/OpenAI node availability in the n8n instance
- **Edge cases or potential failure types:**
  - Required fields may not actually be present in the transcript
  - Extracted values can be guessed or partially hallucinated if prompt quality is insufficient
  - Formatting or language mismatch in transcript may reduce extraction precision
- **Sub-workflow reference:** None

#### OpenAI Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used by `Information Extractor`.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - No custom options or built-in tools
  - Uses OpenAI credential named `OpenAi account (Eure)`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI output to: `Information Extractor`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI API access to the selected model
- **Edge cases or potential failure types:**
  - Authentication failure
  - Model availability or entitlement mismatch
  - API rate limits or quota exhaustion
- **Sub-workflow reference:** None

#### Calculate Rating
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Prompt-based LLM chain that evaluates the interview and returns a structured score, verdict, and rationale.
- **Configuration choices:**
  - Input text: `{{$json.transcript_text}}`
  - Prompt defines a financial and management analysis agent
  - Asks for:
    - final verdict
    - risk classification conceptually
    - valid JSON output with:
      - `score`
      - `verdict`
      - `reason`
  - Output parser enabled
- **Key expressions or variables used:**
  - `={{ $json.transcript_text }}`
- **Input and output connections:**
  - Main input from: `Extract Fulltext`
  - AI model input from: `OpenAI Chat Model`
  - Output parser input from: `Structured Output Parser`
  - Main output to: `Merge`
- **Version-specific requirements:**
  - Type version `1.9`
  - Depends on LangChain integration features present in recent n8n versions
- **Edge cases or potential failure types:**
  - Prompt asks for “risk classification” in prose, but parser schema only accepts `score`, `verdict`, `reason`; classification is not retained
  - Model may return invalid JSON despite parser guidance
  - Empty or poor transcript produces weak scoring
  - Long transcripts may hit token limits
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used by `Calculate Rating`.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - No custom options
  - Uses the same OpenAI credential
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI output to: `Calculate Rating`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - Same as the other OpenAI model node: auth, rate limits, quotas, model availability
- **Sub-workflow reference:** None

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a manual JSON schema on the `Calculate Rating` output.
- **Configuration choices:**
  - Manual schema:
    - `score`: number
    - `verdict`: string
    - `reason`: string
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI parser output to: `Calculate Rating`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - If the model output cannot be coerced into the schema, execution fails
  - No enum constraints are defined for `verdict`, so values may vary
- **Sub-workflow reference:** None

---

## 2.5 Report Assembly and Email Delivery

### Overview
This block combines the extracted company identity details with the risk evaluation, generates a styled HTML financial report, and sends it by Gmail.

### Nodes Involved
- Merge
- Financial Report Generator
- OpenAI Chat Model2
- Send report

### Node Details

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the outputs of the extraction and rating branches into one item.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input 0 from: `Information Extractor`
  - Input 1 from: `Calculate Rating`
  - Output to: `Financial Report Generator`
- **Version-specific requirements:**
  - Type version `3.2`
- **Edge cases or potential failure types:**
  - If one branch fails or returns no item, merge behavior may block downstream execution
  - Output structure must be checked carefully because downstream expressions expect nested `output` fields
- **Sub-workflow reference:** None

#### Financial Report Generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Generates the HTML body of the financial report email.
- **Configuration choices:**
  - Input text built from merged data:
    - company name
    - representative name
    - address
    - VAT number
    - score
    - verdict
    - reason
  - Prompt instructs model to:
    - output HTML only
    - start with `<div>`
    - use inline CSS only
    - use email-compatible table-based layout
    - include header, company details, risk summary, rationale, recommendation, and footer
- **Key expressions or variables used:**
  - `{{ $json.output.company_name }}`
  - `{{ $json.output.name }}`
  - `{{ $json.output.address }}`
  - `{{ $json.output.vat_number }}`
  - `{{ $json.output.score }}`
  - `{{ $json.output.verdict }}`
  - `{{ $json.output.reason }}`
- **Input and output connections:**
  - Main input from: `Merge`
  - AI model input from: `OpenAI Chat Model2`
  - Main output to: `Send report`
- **Version-specific requirements:**
  - Type version `1.9`
- **Edge cases or potential failure types:**
  - The prompt expects merged fields under `output`; if Merge produces a different structure, expressions break
  - Model may output HTML fragments that Gmail sanitizes
  - Missing extracted fields will create incomplete report sections
- **Sub-workflow reference:** None

#### OpenAI Chat Model2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used to generate the HTML report.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - Same OpenAI credential
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI output to: `Financial Report Generator`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - Auth/rate-limit/model-access issues
  - HTML quality depends on model consistency
- **Sub-workflow reference:** None

#### Send report
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated HTML report to a designated email address.
- **Configuration choices:**
  - Recipient: `xxx@xxx.com`
  - Subject: `[Financial Report] <company_name>`
  - Message body: `{{$json.text}}`
  - No extra send options configured
- **Key expressions or variables used:**
  - `={{ $json.text }}`
  - `=[Financial Report] {{ $('Merge').item.json.output.company_name }}`
- **Input and output connections:**
  - Input from: `Financial Report Generator`
  - No downstream node
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail OAuth2 credentials
- **Edge cases or potential failure types:**
  - Placeholder recipient must be replaced before real use
  - Subject expression depends on `Merge` output path being valid
  - OAuth token or Gmail sending restrictions may block delivery
  - If Gmail node is not configured to send HTML appropriately in the environment, rendering may vary
- **Sub-workflow reference:** None

---

## 2.6 Documentation and Visual Notes

### Overview
These nodes do not participate in execution logic but document the workflow visually inside n8n. They include setup instructions, process summaries, and a promotional link.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note9

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  General documentation block describing the workflow, setup steps, and credential requirements.
- **Configuration choices:** Large note covering the top-level process
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the audio upload step.
- **Configuration choices:** Color-coded note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the transcript extraction step.
- **Configuration choices:** Color-coded note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the AI extraction and rating step.
- **Configuration choices:** Color-coded note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the report generation and email step.
- **Configuration choices:** Color-coded note
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Promotional note linking to a YouTube channel.
- **Configuration choices:** Large note with image link and URL
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receives ElevenLabs webhook POST requests |  | Switch | ## Financial Risk Report Generator from ElevenLabs Interview audio Using AI<br>This workflow automates the process of receiving a post-call audio file and transcription from [ElevenLabs](https://try.elevenlabs.io/ahkbf00hocnu), processing them, and generating a financial risk report.<br><br>### How it works<br><br>This workflow receives ElevenLabs webhook events and routes them by payload type to handle either post-call audio or post-call transcription. Audio events are decoded from Base64, converted into an MP3 binary file, and uploaded to Google Drive for storage and traceability. Transcription events are transformed into a single readable conversation text, which is then analyzed in parallel by AI nodes: one extracts structured company details, and another assigns a financial risk score, verdict, and justification using structured JSON output.<br><br>The extracted company data and risk evaluation are merged and passed to a final report-generation step. That node creates a polished HTML financial risk report, which is then sent automatically by Gmail to the target recipient. The result is a fully automated pipeline from interview output to stakeholder-ready email delivery.<br><br>### Setup steps<br><br>Configure the required credentials first: OpenAI for the language model nodes, Google Drive for audio upload, and Gmail for sending the final report. Then verify the webhook path and ID, and register that endpoint in ElevenLabs so post-call audio and transcription events are sent into the workflow correctly.<br><br>Next, review node-specific parameters. Update the Google Drive destination folder if needed, confirm the extraction fields still match the business data you want to collect, and replace the placeholder Gmail recipient with the correct email address while checking the subject format. After credentials and node settings are validated, activate the workflow in n8n so it can start receiving webhook events and processing interviews automatically. |
| Switch | n8n-nodes-base.switch | Routes webhook events by ElevenLabs payload type | Webhook | Generate MP3; Get Transcript | ## Financial Risk Report Generator from ElevenLabs Interview audio Using AI<br>This workflow automates the process of receiving a post-call audio file and transcription from [ElevenLabs](https://try.elevenlabs.io/ahkbf00hocnu), processing them, and generating a financial risk report.<br><br>### How it works<br><br>This workflow receives ElevenLabs webhook events and routes them by payload type to handle either post-call audio or post-call transcription. Audio events are decoded from Base64, converted into an MP3 binary file, and uploaded to Google Drive for storage and traceability. Transcription events are transformed into a single readable conversation text, which is then analyzed in parallel by AI nodes: one extracts structured company details, and another assigns a financial risk score, verdict, and justification using structured JSON output.<br><br>The extracted company data and risk evaluation are merged and passed to a final report-generation step. That node creates a polished HTML financial risk report, which is then sent automatically by Gmail to the target recipient. The result is a fully automated pipeline from interview output to stakeholder-ready email delivery.<br><br>### Setup steps<br><br>Configure the required credentials first: OpenAI for the language model nodes, Google Drive for audio upload, and Gmail for sending the final report. Then verify the webhook path and ID, and register that endpoint in ElevenLabs so post-call audio and transcription events are sent into the workflow correctly.<br><br>Next, review node-specific parameters. Update the Google Drive destination folder if needed, confirm the extraction fields still match the business data you want to collect, and replace the placeholder Gmail recipient with the correct email address while checking the subject format. After credentials and node settings are validated, activate the workflow in n8n so it can start receiving webhook events and processing interviews automatically. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for risk scoring |  | Calculate Rating | ## STEP 3 - Extract Informations<br>The full transcript text is then used by two nodes in parallel: Information Extractor and Calculate Rating |
| Upload audio | n8n-nodes-base.googleDrive | Uploads MP3 audio to Google Drive | Generate MP3 |  | ## STEP 1 - Upload audio<br>Save audio call to Google Drive |
| Generate MP3 | n8n-nodes-base.code | Converts Base64 audio payload into MP3 binary | Switch | Upload audio | ## STEP 1 - Upload audio<br>Save audio call to Google Drive |
| Get Transcript | n8n-nodes-base.set | Extracts transcript array from ElevenLabs payload | Switch | Extract Fulltext | ## STEP 2 - Get Audio Transcript<br>Get audio transcript from Elevenlabs audio |
| Extract Fulltext | n8n-nodes-base.code | Builds readable transcript text from transcript entries | Get Transcript | Information Extractor; Calculate Rating | ## STEP 2 - Get Audio Transcript<br>Get audio transcript from Elevenlabs audio |
| Calculate Rating | @n8n/n8n-nodes-langchain.chainLlm | Produces structured financial score, verdict, and reason | Extract Fulltext | Merge | ## STEP 3 - Extract Informations<br>The full transcript text is then used by two nodes in parallel: Information Extractor and Calculate Rating |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema for rating output |  | Calculate Rating | ## STEP 3 - Extract Informations<br>The full transcript text is then used by two nodes in parallel: Information Extractor and Calculate Rating |
| Information Extractor | @n8n/n8n-nodes-langchain.informationExtractor | Extracts company identity fields from transcript | Extract Fulltext | Merge | ## STEP 3 - Extract Informations<br>The full transcript text is then used by two nodes in parallel: Information Extractor and Calculate Rating |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for information extraction |  | Information Extractor | ## STEP 3 - Extract Informations<br>The full transcript text is then used by two nodes in parallel: Information Extractor and Calculate Rating |
| OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for HTML report generation |  | Financial Report Generator | ## STEP 4 - Financial Report<br>Financial Report Generator and send it to email |
| Merge | n8n-nodes-base.merge | Combines extracted company data and rating data | Information Extractor; Calculate Rating | Financial Report Generator | ## STEP 3 - Extract Informations<br>The full transcript text is then used by two nodes in parallel: Information Extractor and Calculate Rating |
| Financial Report Generator | @n8n/n8n-nodes-langchain.chainLlm | Generates HTML financial risk report | Merge | Send report | ## STEP 4 - Financial Report<br>Financial Report Generator and send it to email |
| Send report | n8n-nodes-base.gmail | Emails the generated report via Gmail | Financial Report Generator |  | ## STEP 4 - Financial Report<br>Financial Report Generator and send it to email |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for setup and process |  |  | ## Financial Risk Report Generator from ElevenLabs Interview audio Using AI<br>This workflow automates the process of receiving a post-call audio file and transcription from [ElevenLabs](https://try.elevenlabs.io/ahkbf00hocnu), processing them, and generating a financial risk report.<br><br>### How it works<br><br>This workflow receives ElevenLabs webhook events and routes them by payload type to handle either post-call audio or post-call transcription. Audio events are decoded from Base64, converted into an MP3 binary file, and uploaded to Google Drive for storage and traceability. Transcription events are transformed into a single readable conversation text, which is then analyzed in parallel by AI nodes: one extracts structured company details, and another assigns a financial risk score, verdict, and justification using structured JSON output.<br><br>The extracted company data and risk evaluation are merged and passed to a final report-generation step. That node creates a polished HTML financial risk report, which is then sent automatically by Gmail to the target recipient. The result is a fully automated pipeline from interview output to stakeholder-ready email delivery.<br><br>### Setup steps<br><br>Configure the required credentials first: OpenAI for the language model nodes, Google Drive for audio upload, and Gmail for sending the final report. Then verify the webhook path and ID, and register that endpoint in ElevenLabs so post-call audio and transcription events are sent into the workflow correctly.<br><br>Next, review node-specific parameters. Update the Google Drive destination folder if needed, confirm the extraction fields still match the business data you want to collect, and replace the placeholder Gmail recipient with the correct email address while checking the subject format. After credentials and node settings are validated, activate the workflow in n8n so it can start receiving webhook events and processing interviews automatically. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual label for audio upload step |  |  | ## STEP 1 - Upload audio<br>Save audio call to Google Drive |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual label for transcript extraction step |  |  | ## STEP 2 - Get Audio Transcript<br>Get audio transcript from Elevenlabs audio |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual label for AI extraction and rating step |  |  | ## STEP 3 - Extract Informations<br>The full transcript text is then used by two nodes in parallel: Information Extractor and Calculate Rating |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual label for report and email step |  |  | ## STEP 4 - Financial Report<br>Financial Report Generator and send it to email |
| Sticky Note9 | n8n-nodes-base.stickyNote | Promotional note with YouTube link |  |  | ## MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new **YouTube channel**](https://youtube.com/@n3witalia). Here I’ll share videos and Shorts with practical tutorials and **FREE templates for n8n**.<br><br>[![image](https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg)](https://youtube.com/@n3witalia) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Financial Report from ElevenLabs Interview`.
   - In workflow settings, keep binary handling compatible with file processing. This workflow uses separate binary storage mode in the exported settings.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - HTTP method: `POST`
   - Path: `0a1e32dd-fb2d-450f-a9bd-e69c43bfdecf`
   - Leave extra options empty unless you need authentication or response customization.
   - This will be the endpoint to register in ElevenLabs.

3. **Add a Switch node**
   - Node type: `Switch`
   - Connect `Webhook -> Switch`
   - Configure two outputs using rules on `{{$json.body.type}}`
     1. Output name: `Post Call Audio`
        - Condition: equals `post_call_audio`
     2. Output name: `Post call Transcription`
        - Condition: equals `post_call_transcription`
   - Leave loose type validation enabled.

4. **Build the audio-storage branch**
   - Add a `Code` node named `Generate MP3`
   - Connect `Switch (Post Call Audio) -> Generate MP3`
   - Paste this logic conceptually:
     - Read `body.data.full_audio`
     - Read `conversation_id`, `agent_id`, `agent_name`, `user_id`
     - Convert Base64 audio into a binary buffer
     - Return JSON metadata plus binary field `data`
     - Set:
       - MIME type: `audio/mpeg`
       - File name: `conversation_<conversation_id>.mp3`
   - The implemented code in the source workflow creates the binary property `data`, so preserve that property name.

5. **Add Google Drive upload**
   - Node type: `Google Drive`
   - Connect `Generate MP3 -> Upload audio`
   - Configure it to upload the binary file
   - Set file name to `{{$binary.data.fileName}}`
   - Destination drive: `My Drive`
   - Folder: `/ (Root folder)` unless you want a dedicated archive folder
   - Create or select a **Google Drive OAuth2** credential
   - Confirm the account has write permission to the destination.

6. **Build the transcription branch**
   - Add a `Set` node named `Get Transcript`
   - Connect `Switch (Post call Transcription) -> Get Transcript`
   - Add one field:
     - Name: `transcript`
     - Type: `Array`
     - Value: `{{$json.body.data.transcript}}`

7. **Add transcript formatting logic**
   - Add a `Code` node named `Extract Fulltext`
   - Connect `Get Transcript -> Extract Fulltext`
   - Configure the code to:
     - Read `$json.transcript`
     - Filter entries with a non-empty `message`
     - Map each entry into `role: message`
     - Join entries with blank lines
     - Return:
       - `conversation_id`
       - `agent_name`
       - `transcript_text`
   - Important: verify the node still has access to `conversation_id` and `agent_name` from upstream data. If not, explicitly preserve them in the `Set` node as additional fields.

8. **Add the company information extraction branch**
   - Add an `Information Extractor` node
   - Connect `Extract Fulltext -> Information Extractor`
   - Set input text to `{{$json.transcript_text}}`
   - Set the system prompt to instruct the model to extract only relevant information and omit unknown values
   - Define these attributes:
     - `company_name` required
     - `name` required
     - `address` required
     - `vat_number` required

9. **Add the AI model for extraction**
   - Add an `OpenAI Chat Model` node and name it `OpenAI Chat Model1`
   - Select model: `gpt-5-mini`
   - Create/select an **OpenAI API** credential
   - Connect the model node to the `Information Extractor` through the AI language-model port

10. **Add the financial scoring branch**
    - Add a `Chain LLM` node named `Calculate Rating`
    - Connect `Extract Fulltext -> Calculate Rating`
    - Set input text to `{{$json.transcript_text}}`
    - Use a custom prompt that:
      - defines the model as a financial and management analysis agent
      - asks it to evaluate the interview on a 1–5 scale framework
      - asks for a final verdict
      - requires JSON output with:
        - `score`
        - `verdict`
        - `reason`
      - instructs it not to output anything outside the JSON object

11. **Add the AI model for scoring**
    - Add another `OpenAI Chat Model` node named `OpenAI Chat Model`
    - Select model: `gpt-5-mini`
    - Reuse the same **OpenAI API** credential
    - Connect it to `Calculate Rating` via the AI language-model port

12. **Add a Structured Output Parser**
    - Add a `Structured Output Parser` node
    - Connect it to `Calculate Rating` via the AI output-parser port
    - Use a manual schema:
      - object
      - properties:
        - `score`: number
        - `verdict`: string
        - `reason`: string

13. **Merge the AI outputs**
    - Add a `Merge` node
    - Connect:
      - `Information Extractor -> Merge` on input 1
      - `Calculate Rating -> Merge` on input 2
    - Configure:
      - Mode: `Combine`
      - Combine By: `Combine All`
    - Test the merged output structure. The original workflow expects merged values under an `output` object. If your n8n version structures Merge output differently, adjust downstream expressions.

14. **Add the HTML report generator**
    - Add a `Chain LLM` node named `Financial Report Generator`
    - Connect `Merge -> Financial Report Generator`
    - Set the text input to include:
      - company name
      - representative name
      - address
      - VAT number
      - score
      - verdict
      - reason
    - In the source workflow, these are referenced as:
      - `{{$json.output.company_name}}`
      - `{{$json.output.name}}`
      - `{{$json.output.address}}`
      - `{{$json.output.vat_number}}`
      - `{{$json.output.score}}`
      - `{{$json.output.verdict}}`
      - `{{$json.output.reason}}`
    - Use a prompt that instructs the model to generate:
      - HTML only
      - complete fragment starting with `<div>`
      - inline CSS only
      - email-compatible table layout
      - sections for header, details, risk summary, rationale, recommendation, and footer

15. **Add the AI model for report generation**
    - Add a third `OpenAI Chat Model` node named `OpenAI Chat Model2`
    - Select model: `gpt-5-mini`
    - Reuse the same **OpenAI API** credential
    - Connect it to `Financial Report Generator` via the AI language-model port

16. **Add Gmail sending**
    - Add a `Gmail` node named `Send report`
    - Connect `Financial Report Generator -> Send report`
    - Set:
      - To: replace `xxx@xxx.com` with the actual recipient
      - Subject: `[Financial Report] {{ $('Merge').item.json.output.company_name }}`
      - Message body: `{{$json.text}}`
    - Create/select a **Gmail OAuth2** credential
    - Ensure the Gmail account is allowed to send mail in your environment

17. **Optional but recommended: add defensive checks**
    - Before the AI nodes, consider an `IF` node to stop executions when transcript text is empty.
    - Before sending email, consider verifying that company name and verdict are present.
    - Consider adding an error branch or an `Error Trigger` workflow for failed LLM or Gmail operations.

18. **Register the webhook in ElevenLabs**
    - Use the production webhook URL from the `Webhook` node after activating the workflow
    - Configure ElevenLabs to send at least:
      - `post_call_audio`
      - `post_call_transcription`
    - Verify that the payload fields match:
      - `body.type`
      - `body.data.full_audio`
      - `body.data.transcript`

19. **Activate and test**
    - Test with a sample `post_call_audio` payload to confirm Google Drive upload
    - Test with a sample `post_call_transcription` payload to confirm:
      - transcript flattening
      - company data extraction
      - rating output schema
      - merge structure
      - HTML generation
      - Gmail delivery

20. **Add visual documentation if desired**
    - Recreate sticky notes for maintainability:
      - overall workflow description
      - Step 1 upload audio
      - Step 2 transcript
      - Step 3 extraction and rating
      - Step 4 report email

### Credential setup summary

- **OpenAI API credential**
  - Used by:
    - `OpenAI Chat Model`
    - `OpenAI Chat Model1`
    - `OpenAI Chat Model2`
  - Must support model `gpt-5-mini`

- **Google Drive OAuth2 credential**
  - Used by:
    - `Upload audio`

- **Gmail OAuth2 credential**
  - Used by:
    - `Send report`

### Input/output expectations

- **Webhook expected input**
  - `body.type` determines routing
  - Audio branch expects `body.data.full_audio`
  - Transcript branch expects `body.data.transcript`

- **Audio branch output**
  - MP3 file uploaded to Google Drive

- **Transcript branch output**
  - Email containing HTML financial risk report

### Important implementation notes
- The workflow has **one entry point** only: the `Webhook` node.
- There are **no sub-workflows** invoked.
- The current workflow stores audio and analyzes transcription as **separate webhook events**. It does not explicitly correlate the two branches afterward beyond shared conversation metadata.
- The placeholder email address in `Send report` must be changed.
- The merge/output field paths should be validated carefully in your n8n version, because AI and Merge nodes can differ in output shape.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ElevenLabs is the source of the webhook events used by this workflow. | https://try.elevenlabs.io/ahkbf00hocnu |
| Promotional note: “Subscribe to my new YouTube channel. Here I’ll share videos and Shorts with practical tutorials and FREE templates for n8n.” | https://youtube.com/@n3witalia |
| Promotional image link used inside the visual note. | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |
| The workflow includes built-in visual setup guidance reminding the user to configure OpenAI, Google Drive, and Gmail credentials, verify the webhook path, and replace the placeholder recipient email. | Internal workflow note |
| The workflow is currently inactive in the exported JSON (`active: false`), so it must be activated before production webhook calls will work. | Internal workflow state |