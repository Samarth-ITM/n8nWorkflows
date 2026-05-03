Turn book PDFs into audio with OpenAI and Google Drive

https://n8nworkflows.xyz/workflows/turn-book-pdfs-into-audio-with-openai-and-google-drive-14574


# Turn book PDFs into audio with OpenAI and Google Drive

# 1. Workflow Overview

**Purpose:**  
This workflow turns an uploaded **book PDF** into a collection of **audio files**. It accepts a PDF through an n8n form, extracts the text, uses OpenAI to infer the book’s chapter structure and generate parsing logic, splits the text into chapter-based chunks, converts each chunk to audio, and uploads the resulting files into a dedicated Google Drive folder.

**Primary use cases:**
- Converting long-form books or documents into spoken audio files
- Automatically organizing generated audio by upload/book
- Handling books with unknown or inconsistent chapter formats by using AI-generated parsing logic

## 1.1 Input Reception and Storage Preparation
The workflow starts when a user uploads a PDF via a form. In parallel, it:
- extracts the text from the PDF
- creates a Google Drive folder named after the uploaded file

## 1.2 AI-Based Structure Detection
The extracted text is sent to an AI agent. The AI inspects the beginning of the book, identifies likely chapter/section markers, and returns structured output containing:
- a description or regex-like pattern
- JavaScript code intended for an n8n Code node

## 1.3 Dynamic Structuring and Chunk Generation
A Code node executes the AI-generated JavaScript against the full extracted text. This produces an array of items representing book chunks with metadata such as chapter, part, title, text, and filename.

## 1.4 Folder/Text Merge and Iterative Processing
The created Google Drive folder information is merged with the structured content so every chunk knows where it should be uploaded. The workflow then loops over each chunk one by one.

## 1.5 Audio Generation and Delivery
Each text chunk is converted to audio using OpenAI’s audio generation capability, then uploaded to the Google Drive folder created for the uploaded book.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Initial Preparation

### Overview
This block receives the PDF from the user and prepares the two essential resources needed downstream: the extracted text and the destination Drive folder. It is the workflow’s entry point and fan-out stage.

### Nodes Involved
- Book Pdf Upload
- Extract Book Content
- Create folder

### Node Details

#### 1. Book Pdf Upload
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point webhook form that receives a PDF file upload from the user.
- **Configuration choices:**
  - Form title: **Turn your Book into Audio**
  - One required file field labeled **Book pdf**
- **Key expressions or variables used:**
  - Downstream nodes reference the uploaded file through:
    - `$('Book Pdf Upload').item.json["Book pdf"][0].filename`
- **Input and output connections:**
  - No input; this is a trigger node
  - Outputs to:
    - `Extract Book Content`
    - `Create folder`
- **Version-specific requirements:**
  - Uses **typeVersion 2.3**
  - Requires n8n support for Form Trigger file uploads
- **Edge cases or potential failure types:**
  - User submits no file
  - Non-PDF file uploaded despite intent
  - Large file uploads may hit webhook or instance limits
  - Binary field naming may differ if manually changed later
- **Sub-workflow reference:** None

#### 2. Extract Book Content
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts text from the uploaded PDF binary.
- **Configuration choices:**
  - Operation: **pdf**
  - Binary property name: `=Book_pdf`
- **Key expressions or variables used:**
  - Produces text at `item.json.text`
- **Input and output connections:**
  - Input from `Book Pdf Upload`
  - Output to `AI Agent`
- **Version-specific requirements:**
  - Uses **typeVersion 1**
- **Edge cases or potential failure types:**
  - Binary property mismatch: this node expects `Book_pdf`; if the incoming binary field uses a different key, extraction fails
  - Scanned/image-only PDFs may produce poor or empty extraction
  - Very large PDFs may cause memory or execution-time issues
  - Encrypted or malformed PDFs may fail to parse
- **Sub-workflow reference:** None

#### 3. Create folder
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Creates a destination folder in Google Drive for this book’s generated audio files.
- **Configuration choices:**
  - Resource: **folder**
  - Parent folder: **root**
  - Folder name from uploaded file name:
    - `={{ $('Book Pdf Upload').item.json["Book pdf"][0].filename }}`
  - Execute once: **true**
- **Key expressions or variables used:**
  - Uploaded filename used as folder name
- **Input and output connections:**
  - Input from `Book Pdf Upload`
  - Output to `Merge`
- **Version-specific requirements:**
  - Uses **typeVersion 3**
  - Requires valid Google Drive OAuth2 credentials
- **Edge cases or potential failure types:**
  - Google Drive authentication failure
  - Duplicate folder names may create ambiguity depending on Drive behavior and later assumptions
  - Invalid characters in file name can affect folder naming
  - Rate limiting or API quota issues
- **Sub-workflow reference:** None

---

## Block 2 — AI Structure Detection

### Overview
This block uses an LLM-driven agent to inspect a sample of the extracted text and infer how the book is structured. The output is normalized through a structured parser so the next Code node can consume it reliably.

### Nodes Involved
- AI Agent
- OpenAI Chat Model
- Structured Output Parser

### Node Details

#### 4. AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Coordinates the prompt, language model, and structured parser to generate chapter-splitting logic.
- **Configuration choices:**
  - Prompt type: **define**
  - Custom prompt instructs the model to:
    - inspect the first 3000 characters of the extracted book text
    - detect chapter/section formatting
    - generate full JavaScript for an n8n Code node
    - split chapters into chunks of max 3900 characters
    - return fields: `chapter`, `part`, `title`, `text`, `filename`
  - Output parser enabled
- **Key expressions or variables used:**
  - Sample text:
    - `{{ $json.text.slice(0, 3000) }}`
- **Input and output connections:**
  - Main input from `Extract Book Content`
  - AI language model input from `OpenAI Chat Model`
  - AI output parser input from `Structured Output Parser`
  - Main output to `Structure The Content`
- **Version-specific requirements:**
  - Uses **typeVersion 3.1**
  - Requires compatible LangChain nodes in the installed n8n version
- **Edge cases or potential failure types:**
  - Model may mis-detect book structure if the first 3000 characters are preface/front matter rather than chapter content
  - Model may generate invalid JavaScript
  - Structured parser may fail if model output does not match expected schema
  - Token or rate-limit issues on OpenAI side
- **Sub-workflow reference:** None

#### 5. OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM used by the AI Agent.
- **Configuration choices:**
  - Model: **gpt-4.1-mini**
  - No additional tools enabled
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Connected to `AI Agent` via `ai_languageModel`
- **Version-specific requirements:**
  - Uses **typeVersion 1.3**
  - Requires OpenAI credentials
  - Requires model availability in the connected OpenAI account
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model unavailable in region/account
  - Rate limiting
  - Context length limitations for very large prompts
- **Sub-workflow reference:** None

#### 6. Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the AI response into a structured object so the code string can be extracted predictably.
- **Configuration choices:**
  - Expected schema example:
    - `pattern`
    - `code`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Connected to `AI Agent` via `ai_outputParser`
- **Version-specific requirements:**
  - Uses **typeVersion 1.3**
- **Edge cases or potential failure types:**
  - Parser failure if model returns malformed structure
  - Escaping problems if the returned code includes characters that break JSON formatting
- **Sub-workflow reference:** None

---

## Block 3 — Dynamic Structuring and Chunk Production

### Overview
This block executes AI-generated JavaScript inside a Code node to transform the full book text into structured, chapter-aware chunks. It is the most dynamic part of the workflow because it relies on generated code rather than static splitting rules.

### Nodes Involved
- Structure The Content

### Node Details

#### 7. Structure The Content
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes generated JavaScript to split the book text into chunk items suitable for text-to-audio conversion.
- **Configuration choices:**
  - Retrieves AI output from:
    - `$('AI Agent').item.json.output`
  - Retrieves full extracted text from:
    - `$('Extract Book Content').item.json.text`
  - Logs the detected pattern with `console.log`
  - Builds a new executable function using `new Function(...)`
  - Replaces the generated code’s expected input source:
    - from `const text = $input.item.json.text;`
    - to `const text = bookText;`
  - Removes `return output;` and re-wraps result as n8n items:
    - `return output.map(item => ({ json: item }));`
- **Key expressions or variables used:**
  - `$('AI Agent').item.json.output`
  - `$('Extract Book Content').item.json.text`
  - `agentOutput.pattern`
  - `agentOutput.code`
- **Input and output connections:**
  - Main input from `AI Agent`
  - Main output to `Merge`
- **Version-specific requirements:**
  - Uses **typeVersion 2**
  - Requires Code node support for JS execution
- **Edge cases or potential failure types:**
  - AI-generated code may be syntactically invalid
  - The generated code may not contain the exact expected strings being replaced
  - If the generated code does not define `output`, the wrapper fails
  - If chapter parsing is wrong, chunking and filenames may be poor or inconsistent
  - Executing AI-generated code introduces maintainability and security considerations
  - Very large book content can create slow execution or memory pressure
- **Sub-workflow reference:** None

---

## Block 4 — Merge Folder Metadata with Structured Chunks

### Overview
This block combines the single folder creation result with every structured text chunk so each item includes both chunk metadata and the Drive folder ID. This is necessary before iterating through items for audio generation and upload.

### Nodes Involved
- Merge

### Node Details

#### 8. Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the output from folder creation and content structuring.
- **Configuration choices:**
  - Mode: **combineBySql**
  - SQL query:
    - Select all fields from input 2 and input 1
    - `CROSS JOIN` so the single folder item is attached to every chunk item
- **Key expressions or variables used:**
  - SQL aliasing:
    - `input1` = folder data from `Create folder`
    - `input2` = chunk data from `Structure The Content`
- **Input and output connections:**
  - Input 1 from `Create folder`
  - Input 2 from `Structure The Content`
  - Output to `Loop Over Items`
- **Version-specific requirements:**
  - Uses **typeVersion 3.2**
  - Requires n8n version with SQL-based merge mode
- **Edge cases or potential failure types:**
  - If folder creation fails, no merged records are produced
  - If structuring outputs zero chunks, downstream processing stops
  - Field name collisions between inputs may overwrite or shadow values depending on merge semantics
- **Sub-workflow reference:** None

---

## Block 5 — Iterative Audio Generation and Google Drive Upload

### Overview
This block processes one structured text chunk at a time, generates an audio file using OpenAI, uploads it to Google Drive, and continues until all chunks are handled.

### Nodes Involved
- Loop Over Items
- Generate audio
- Upload file

### Node Details

#### 9. Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through merged chunk items.
- **Configuration choices:**
  - No special options configured
  - Default batch handling pattern used with loop-back connection
- **Key expressions or variables used:**
  - Downstream references use:
    - `$('Loop Over Items').item.json.filename`
    - `$('Loop Over Items').item.json.id`
- **Input and output connections:**
  - Input from `Merge`
  - Output 1: completion path unused
  - Output 2: goes to `Generate audio`
  - Receives loop-back input from `Upload file`
- **Version-specific requirements:**
  - Uses **typeVersion 3**
- **Edge cases or potential failure types:**
  - If no items are received, nothing is processed
  - Long books may produce many iterations and long runtime
  - Misunderstanding split-in-batches output ordering can lead to rebuild mistakes
- **Sub-workflow reference:** None

#### 10. Generate audio
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Uses OpenAI audio generation to produce spoken audio from each text chunk.
- **Configuration choices:**
  - Resource: **audio**
  - Input text:
    - `={{ $json.text }}`
  - Speed: **1**
- **Key expressions or variables used:**
  - Reads chunk text from `$json.text`
- **Input and output connections:**
  - Input from `Loop Over Items`
  - Output to `Upload file`
- **Version-specific requirements:**
  - Uses **typeVersion 1.8**
  - Requires OpenAI credentials with access to audio generation
- **Edge cases or potential failure types:**
  - Text chunk too long for the selected audio endpoint/model
  - API errors or unsupported voice/model defaults depending on node defaults and account setup
  - Rate limits on many chunks
  - Binary audio output format assumptions must match upload expectations
- **Sub-workflow reference:** None

#### 11. Upload file
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads each generated audio file to the created Drive folder.
- **Configuration choices:**
  - File name:
    - `={{ $('Loop Over Items').item.json.filename }}`
  - Drive: **My Drive**
  - Folder ID:
    - `={{ $('Loop Over Items').item.json.id }}`
- **Key expressions or variables used:**
  - Filename from structured chunk metadata
  - Folder ID from merged Google Drive folder result
- **Input and output connections:**
  - Input from `Generate audio`
  - Output loops back to `Loop Over Items`
- **Version-specific requirements:**
  - Uses **typeVersion 3**
  - Requires Google Drive OAuth2 credentials
- **Edge cases or potential failure types:**
  - Wrong folder ID if merge output shape changes
  - Missing or duplicate filenames
  - Binary payload not present or not mapped as expected
  - Google Drive upload quota/rate-limit errors
- **Sub-workflow reference:** None

---

## Block 6 — Documentation and In-Canvas Notes

### Overview
These nodes are non-executable documentation elements placed on the canvas. They describe the workflow purpose, setup resources, and logical sections.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### 12. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation node containing the overall workflow description and external resources.
- **Configuration choices:**
  - Includes overview, step summary, and links
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:**
  - Uses **typeVersion 1**
- **Edge cases or potential failure types:** None operational
- **Sub-workflow reference:** None

#### 13. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents input/setup area
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** **typeVersion 1**
- **Edge cases or potential failure types:** None operational
- **Sub-workflow reference:** None

#### 14. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents AI processing area
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** **typeVersion 1**
- **Edge cases or potential failure types:** None operational
- **Sub-workflow reference:** None

#### 15. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents audio generation and delivery area
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** **typeVersion 1**
- **Edge cases or potential failure types:** None operational
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Book Pdf Upload | n8n-nodes-base.formTrigger | Receives uploaded PDF via form |  | Extract Book Content, Create folder | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>📥 INPUT & SETUP<br><br>The workflow begins with a user uploading a book in PDF format via a form trigger. |
| Create folder | n8n-nodes-base.googleDrive | Creates a Drive folder for this book | Book Pdf Upload | Merge | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>📥 INPUT & SETUP<br><br>The workflow begins with a user uploading a book in PDF format via a form trigger. |
| Extract Book Content | n8n-nodes-base.extractFromFile | Extracts text from uploaded PDF | Book Pdf Upload | AI Agent | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>📥 INPUT & SETUP<br><br>The workflow begins with a user uploading a book in PDF format via a form trigger. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies LLM to AI Agent |  | AI Agent | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>🧠 AI PROCESSING ENGINE<br><br>An AI agent analyzes the book content to detect chapter patterns and generate dynamic parsing logic. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured AI output with pattern and code fields |  | AI Agent | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>🧠 AI PROCESSING ENGINE<br><br>An AI agent analyzes the book content to detect chapter patterns and generate dynamic parsing logic. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Analyzes book sample and generates chapter-splitting JS | Extract Book Content | Structure The Content | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>🧠 AI PROCESSING ENGINE<br><br>An AI agent analyzes the book content to detect chapter patterns and generate dynamic parsing logic. |
| Structure The Content | n8n-nodes-base.code | Executes generated JS and emits chunk items | AI Agent | Merge | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>🧠 AI PROCESSING ENGINE<br><br>An AI agent analyzes the book content to detect chapter patterns and generate dynamic parsing logic. |
| Merge | n8n-nodes-base.merge | Combines Drive folder metadata with all text chunks | Create folder, Structure The Content | Loop Over Items | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>🔊 AUDIO GENERATION & DELIVERY<br><br>Structured text chunks are converted into audio and stored in Google Drive. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterates through each structured chunk | Merge, Upload file | Generate audio | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>🔊 AUDIO GENERATION & DELIVERY<br><br>Structured text chunks are converted into audio and stored in Google Drive. |
| Generate audio | @n8n/n8n-nodes-langchain.openAi | Converts chunk text into audio | Loop Over Items | Upload file | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>🔊 AUDIO GENERATION & DELIVERY<br><br>Structured text chunks are converted into audio and stored in Google Drive. |
| Upload file | n8n-nodes-base.googleDrive | Uploads generated audio to the created Drive folder | Generate audio | Loop Over Items | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>🔊 AUDIO GENERATION & DELIVERY<br><br>Structured text chunks are converted into audio and stored in Google Drive. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation with overview and links |  |  | # Book2Audio Pro<br>## Workflow brief<br>Book2Audio Pro turns an uploaded book PDF into organized audio files. The workflow starts with a file upload form, extracts the book text, uses AI to detect the chapter structure and generate splitting logic, converts each chunk into audio, and then saves the final MP3 files into a Google Drive folder created for that upload.<br>## How it works<br>1. The user uploads a PDF through the form trigger.<br>2. The PDF text is extracted from the uploaded binary file.<br>3. An AI agent analyzes the first part of the book to detect chapter/section patterns and generates JavaScript code for structuring the full text.<br>4. A code node executes the generated logic to split the book into chapters and smaller sentence-safe chunks.<br>5. The text chunks are passed to OpenAI audio generation.<br>6. Each generated audio file is uploaded to a Google Drive folder named after the uploaded book.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for input/setup block |  |  | 📥 INPUT & SETUP<br><br>The workflow begins with a user uploading a book in PDF format via a form trigger. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for AI block |  |  | 🧠 AI PROCESSING ENGINE<br><br>An AI agent analyzes the book content to detect chapter patterns and generate dynamic parsing logic. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for audio/upload block |  |  | 🔊 AUDIO GENERATION & DELIVERY<br><br>Structured text chunks are converted into audio and stored in Google Drive. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like **Book2Audio Pro**.
   - Leave the workflow inactive until testing is complete.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Name: **Book Pdf Upload**
   - Set form title to **Turn your Book into Audio**
   - Add one field:
     - Type: **File**
     - Label: **Book pdf**
     - Required: **true**
   - This becomes the workflow entry point.

3. **Add a Google Drive node to create a folder**
   - Node type: **Google Drive**
   - Name: **Create folder**
   - Connect from **Book Pdf Upload**
   - Set resource to **Folder**
   - Set parent folder to **root**
   - Set folder name to this expression:  
     `{{ $('Book Pdf Upload').item.json["Book pdf"][0].filename }}`
   - Enable **Execute Once**
   - Configure **Google Drive OAuth2** credentials with access to the target Google Drive account.

4. **Add an Extract From File node**
   - Node type: **Extract From File**
   - Name: **Extract Book Content**
   - Connect from **Book Pdf Upload**
   - Operation: **PDF**
   - Set binary property name to `Book_pdf`
   - Important: verify that the uploaded binary field from the Form Trigger is actually available under that key in your n8n version. If not, adapt the binary property name accordingly.

5. **Add an AI Agent node**
   - Node type: **AI Agent**
   - Name: **AI Agent**
   - Connect from **Extract Book Content**
   - Set prompt type to **Define**
   - Paste this prompt:

     ```text
     You are an expert at parsing book text.

     Here is a sample from the beginning of a book (first 3000 characters):
     {{ $json.text.slice(0, 3000) }}

     Analyze the structure and detect how chapters/sections are marked (e.g. "CHAPTER I.", "Chapter 1:", "1.", "Part One", etc.).

     Then write a complete n8n Code node JavaScript function that:
     1. Accepts `$input.item.json.text` as the full book text
     2. Detects and splits the text into chapters/sections using the pattern you found
     3. Splits each chapter into chunks of max 3900 characters, breaking on sentence boundaries
     4. Returns an array of items with fields: chapter, part, title, text, filename (as .mp3)

     Return ONLY the raw JavaScript code. No explanation, no markdown fences.
     ```

   - Enable structured output handling.

6. **Add an OpenAI Chat Model node**
   - Node type: **OpenAI Chat Model**
   - Name: **OpenAI Chat Model**
   - Set model to **gpt-4.1-mini**
   - Configure **OpenAI API** credentials
   - Connect this node to the **AI Agent** through the **Language Model** connector, not the main connector.

7. **Add a Structured Output Parser node**
   - Node type: **Structured Output Parser**
   - Name: **Structured Output Parser**
   - Set the schema example to:

     ```json
     {
       "pattern": "regex or description of what was detected",
       "code": "the actual JS code string to execute"
     }
     ```

   - Connect this node to the **AI Agent** through the **Output Parser** connector.

8. **Add a Code node to execute the generated parsing logic**
   - Node type: **Code**
   - Name: **Structure The Content**
   - Connect from **AI Agent**
   - Paste this JavaScript:

     ```javascript
     // Get the structured JSON output from AI Agent
     const agentOutput = $('AI Agent').item.json.output;

     // Capture the text BEFORE entering the new Function scope
     const bookText = $('Extract Book Content').item.json.text;

     // Log the detected pattern for debugging
     console.log("Detected pattern:", agentOutput.pattern);

     // Wrap the code
     const wrappedCode = `
       ${agentOutput.code.replace('return output;', '')}
       
       // Wrap output in n8n format
       return output.map(item => ({ json: item }));
     `;

     // Pass bookText directly instead of $input
     const fn = new Function('bookText', wrappedCode.replace(
       'const text = $input.item.json.text;',
       'const text = bookText;'
     ));

     return fn(bookText);
     ```

   - This node expects the AI output to contain a valid `code` string and to produce a variable named `output`.

9. **Add a Merge node**
   - Node type: **Merge**
   - Name: **Merge**
   - Connect **Create folder** to input 1
   - Connect **Structure The Content** to input 2
   - Set mode to **Combine by SQL**
   - Use this SQL query:

     ```sql
     SELECT 
         i2.*,
         i1.*
     FROM input2 i2
     CROSS JOIN input1 i1;
     ```

   - This duplicates the folder metadata across all text chunk items.

10. **Add a Loop Over Items node**
    - Node type: **Loop Over Items** / **Split In Batches**
    - Name: **Loop Over Items**
    - Connect from **Merge**
    - Leave default options unless your n8n version requires explicit batch size
    - Use the iterative branch output for per-item processing

11. **Add an OpenAI node for audio generation**
    - Node type: **OpenAI** (LangChain/OpenAI audio-capable node)
    - Name: **Generate audio**
    - Connect from the iterative output of **Loop Over Items**
    - Set resource to **Audio**
    - Set input to:
      `{{ $json.text }}`
    - Set speed to **1**
    - Configure **OpenAI API** credentials
    - Confirm your account and installed node version support audio generation.

12. **Add a Google Drive node to upload each generated file**
    - Node type: **Google Drive**
    - Name: **Upload file**
    - Connect from **Generate audio**
    - Set file name to:
      `{{ $('Loop Over Items').item.json.filename }}`
    - Set destination folder ID to:
      `{{ $('Loop Over Items').item.json.id }}`
    - Select **My Drive**
    - Configure the same **Google Drive OAuth2** credentials
    - Ensure the node is set to upload the binary output generated by the audio node.

13. **Close the loop**
    - Connect **Upload file** back into **Loop Over Items**
    - This allows the loop node to request the next item after each upload finishes.

14. **Optionally add sticky notes**
    - Add one general note with:
      - workflow purpose
      - processing summary
      - setup links
    - Add block labels for:
      - **INPUT & SETUP**
      - **AI PROCESSING ENGINE**
      - **AUDIO GENERATION & DELIVERY**

15. **Test the form trigger**
    - Run the workflow manually or open the form URL
    - Upload a text-based PDF
    - Confirm:
      - text extraction succeeds
      - a Drive folder is created
      - AI returns `pattern` and `code`
      - Code node emits multiple chunk items
      - audio files are generated
      - MP3 or audio outputs appear in the target Drive folder

16. **Validate field assumptions**
    - Check that:
      - extracted text exists at `json.text`
      - folder ID exists at `json.id` after merge
      - filenames are unique enough to avoid overwrites
      - the generated audio node outputs binary data in a format Google Drive accepts

17. **Credential setup requirements**
    - **OpenAI**
      - API key with access to:
        - chat model `gpt-4.1-mini`
        - audio generation endpoint used by the OpenAI node
    - **Google Drive OAuth2**
      - Permission to create folders and upload files to My Drive
    - Reconnect credentials if importing into another workspace or environment.

18. **Recommended hardening after rebuild**
    - Add an **If** or validation node after text extraction to stop on empty text
    - Add error handling if AI output is missing `code`
    - Sanitize filenames before upload
    - Add retry logic or rate limiting for long books
    - Consider replacing dynamic code execution with a safer deterministic parser if book formats are known

**Sub-workflow setup:**  
This workflow does **not** invoke any sub-workflow nodes and does not require a separate child workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | https://drive.google.com/file/d/1PkH1D164Xuju1hHLHayl8xTvYNJrhU34/view?usp=sharing |
| Course | https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |
| Workflow branding: Book2Audio Pro | Overall workflow concept |
| Form title: Turn your Book into Audio | User-facing entry form |
| Important implementation note: the workflow executes AI-generated JavaScript using `new Function(...)`, which is powerful but fragile and should be reviewed carefully in production. | Applies to the Code node and overall architecture |
| Important implementation note: the Extract From File node expects a binary property named `Book_pdf`; verify this matches the actual uploaded binary field in your n8n instance. | Applies to PDF ingestion |