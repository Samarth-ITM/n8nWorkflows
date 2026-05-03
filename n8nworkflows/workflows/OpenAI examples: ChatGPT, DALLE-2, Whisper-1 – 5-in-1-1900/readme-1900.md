OpenAI examples: ChatGPT, DALLE-2, Whisper-1 – 5-in-1

https://n8nworkflows.xyz/workflows/openai-examples--chatgpt--dalle-2--whisper-1---5-in-1-1900


# OpenAI examples: ChatGPT, DALLE-2, Whisper-1 – 5-in-1

### 1. Workflow Overview

This workflow demonstrates five distinct examples of using OpenAI models—ChatGPT, DALLE-2, and Whisper—within n8n to showcase various AI capabilities for text generation, editing, transcription, image generation, and multi-response creation. It is designed as a primer for integrating OpenAI APIs into automation workflows, useful for developers, analysts, and AI enthusiasts who want to explore or build upon OpenAI model usage in n8n.

The workflow is logically divided into these blocks:

- **1.1 Whisper Transcription (Disabled by default)**: Transcribes voice from an audio file into text using the Whisper model.
- **1.2 Legacy Text Completion & Editing with Davinci-003**: Demonstrates old-style OpenAI completions and text edits using the text-davinci-003 model.
- **1.3 Basic ChatGPT Conversational Examples**: Simple ChatGPT API calls for text summarization and translation.
- **1.4 Advanced ChatGPT with System and User Content**: Shows how to include system-level instructions alongside user input for more controlled responses.
- **1.5 Programmatic Prompt Chaining & Image Generation with DALLE-2**: Uses code nodes to prepare message arrays for ChatGPT, then generates image prompts and calls DALLE-2 for image creation.
- **1.6 ChatGPT Code Generation and HTML Rendering**: Generates HTML code containing SVG graphics and renders it.
- **1.7 Multi-Answer Generation for Quick Replies**: Produces multiple short responses useful for email or chat quick replies.

The workflow is modular; users are advised to run individual branches/nodes to avoid long execution times.

---

### 2. Block-by-Block Analysis

#### 1.1 Whisper Transcription (Disabled)

- **Overview:** Transcribes an audio file (mp3) into text using OpenAI’s Whisper model.
- **Nodes Involved:** `LoadMP3`, `Whisper-transcribe`, `Text-example`
- **Node Details:**
  - **LoadMP3**
    - Type: Read Binary File
    - Role: Loads an mp3 audio file from local filesystem.
    - Config: File path set to a specific mp3 file; disabled by default (requires user to enable and provide own file).
    - Inputs: None (start node)
    - Outputs: Binary audio data to Whisper-transcribe.
    - Edge Cases: Missing or inaccessible file, file format errors.
  - **Whisper-transcribe**
    - Type: HTTP Request (OpenAI Whisper API)
    - Role: Sends audio data for transcription.
    - Config: POST to `/v1/audio/transcriptions` with model `whisper-1`, multipart-form data including audio file.
    - Credentials: OpenAI API Key.
    - Inputs: Binary audio from LoadMP3.
    - Outputs: JSON transcription result.
    - Edge Cases: API auth errors, large files, network timeouts.
  - **Text-example**
    - Type: Code Node
    - Role: Provides example text for downstream nodes.
    - Inputs: Transcribed text from Whisper-transcribe.
    - Outputs: JSON with example text for other OpenAI calls.
    - Edge Cases: If transcription fails, downstream nodes won’t have input.

#### 1.2 Legacy Text Completion & Editing (Davinci-003)

- **Overview:** Illustrates legacy usage of OpenAI API with text-davinci-003 model for completion and editing (translation).
- **Nodes Involved:** `Text-example`, `davinci-003-complete`, `davinci-003-edit`, `Sticky Note`
- **Node Details:**
  - **davinci-003-complete**
    - Type: OpenAI Node
    - Role: Text completion with prompt "Tl;dr:" appended to input text.
    - Config: Uses text-davinci-003 implicitly, max tokens 500.
    - Inputs: Text from Text-example.
    - Outputs: Text completion.
    - Edge Cases: Rate limits, long texts exceeding token limit.
  - **davinci-003-edit**
    - Type: OpenAI Node
    - Role: Edits input text with instruction "translate to German".
    - Config: Operation set to "edit".
    - Inputs: Output from davinci-003-complete.
    - Outputs: Translated text.
    - Edge Cases: API errors, unsupported editing instructions.
  - **Sticky Note**
    - Content warns that davinci-003 is expensive and encourages switching to ChatGPT.
    - Link: https://openai.com/blog/introducing-chatgpt-and-whisper-apis

#### 1.3 Basic ChatGPT Conversational Examples

- **Overview:** Demonstrates simple ChatGPT calls for summarizing and translating text using user-only input.
- **Nodes Involved:** `Text-example`, `ChatGPT-ex1.1`, `ChatGPT-ex1.2`, `Sticky Note3`
- **Node Details:**
  - **ChatGPT-ex1.1**
    - Type: OpenAI Node (Chat resource)
    - Role: Summarizes text (Tl;dr).
    - Config: Single user message with prompt to write Tl;dr of input text.
    - Inputs: Text-example output.
    - Outputs: Summary text.
  - **ChatGPT-ex1.2**
    - Type: OpenAI Node (Chat resource)
    - Role: Translates summary text to German.
    - Config: Single user message instructing translation.
    - Inputs: Output of ChatGPT-ex1.1.
    - Outputs: German translation.
  - **Sticky Note3**
    - Describes this block as user-only content ChatGPT examples for summarizing and translating.

#### 1.4 Advanced ChatGPT with System and User Content

- **Overview:** Shows usage of system instructions along with user content to guide ChatGPT behavior with emojis and customized response style.
- **Nodes Involved:** `Text-example`, `ChatGPT-ex2`, `Sticky Note4`
- **Node Details:**
  - **ChatGPT-ex2**
    - Type: OpenAI Node (Chat resource)
    - Role: Summarizes text with system instruction to add 5 emojis at the end.
    - Config: Messages array includes system role content and user content.
    - Inputs: Text-example output.
    - Outputs: Text with emojis added.
    - Edge Cases: Misinterpretation of system instructions, token limits.
  - **Sticky Note4**
    - Explains system content usage to provide instructions for ChatGPT responses.

#### 1.5 Programmatic Prompt Chaining & Image Generation with DALLE-2

- **Overview:** Programmatically creates a message array for ChatGPT, generates a text prompt for an image, then uses DALLE-2 to create comic-style images.
- **Nodes Involved:** `Text-example`, `Code-ex3.1`, `ChatGPT-ex3.1 (HTTP Request)`, `ChatGPT-ex3.2`, `DALLE-ex3.3`, `Sticky Note5`, `Sticky Note6`
- **Node Details:**
  - **Code-ex3.1**
    - Type: Code Node
    - Role: Constructs a messages array with system + user roles for ChatGPT prompt chaining.
    - Inputs: Text-example data.
    - Outputs: JSON object with `messages`.
    - Code: Builds system message for assistant role and appends user text.
  - **ChatGPT-ex3.1**
    - Type: HTTP Request Node
    - Role: Calls ChatGPT API directly with full messages array.
    - Config: POST to OpenAI chat completions endpoint with model `gpt-3.5-turbo`, temperature 0.8, max_tokens 500.
    - Inputs: messages array from Code-ex3.1.
    - Outputs: ChatGPT completion response.
    - Requires OpenAI API credentials.
  - **ChatGPT-ex3.2**
    - Type: OpenAI Node (Chat resource)
    - Role: Generates a DALLE-2 prompt for a comic-style cover image based on ChatGPT-ex3.1 output.
    - Config: System message instructs to create a comic style 60s cover image prompt.
    - Inputs: Output message content from ChatGPT-ex3.1.
    - Outputs: DALLE prompt text.
  - **DALLE-ex3.3**
    - Type: OpenAI Node (Image resource)
    - Role: Generates 4 images at 512x512 resolution using prompt from ChatGPT-ex3.2.
    - Inputs: Prompt text.
    - Outputs: Images URLs.
  - **Sticky Note5**
    - Explains prompt chaining technique and HTTP request usage.
  - **Sticky Note6**
    - Explains ChatGPT prompt generation for DALLE-2 images.

#### 1.6 ChatGPT Code Generation and HTML Rendering

- **Overview:** Generates HTML code with embedded SVG graphics using ChatGPT and renders the output as HTML.
- **Nodes Involved:** `When clicking "Execute Workflow"`, `Set-ex4`, `ChatGPT-ex4`, `HTML-ex4`, `Sticky Note7`
- **Node Details:**
  - **When clicking "Execute Workflow"**
    - Type: Manual Trigger
    - Role: Entry point for this branch.
  - **Set-ex4**
    - Type: Set Node
    - Role: Defines static parameters to generate HTML SVG code.
    - Parameters: Model `code-davinci-002`, prompt to create HTML with SVG shapes, suffix `</svg>`.
  - **ChatGPT-ex4**
    - Type: OpenAI Node (Chat resource)
    - Role: Generates code (HTML + SVG) from prompt.
    - Config: Uses `gpt-3.5-turbo-0301`, temperature 0.5.
  - **HTML-ex4**
    - Type: HTML Node
    - Role: Renders the generated HTML content.
    - Inputs: ChatGPT-ex4 output message content.
  - **Sticky Note7**
    - Indicates this block is for ChatGPT code generation of SVG HTML.

#### 1.7 Multi-Answer Generation for Quick Replies

- **Overview:** Generates multiple brief answers (3 short replies) for quick response scenarios like emails.
- **Nodes Involved:** `When clicking "Execute Workflow"`, `ChatGPT-ex`, `Sticky Note8`
- **Node Details:**
  - **ChatGPT-ex**
    - Type: OpenAI Node (Chat resource)
    - Role: Acts as an email client generating 3 short (5–8 words) replies to a sample message.
    - Config: Model `gpt-3.5-turbo-0301`, max tokens 15, temperature 0.8, n=3.
    - Inputs: Manual trigger initiates request.
    - Outputs: Multiple short reply options.
  - **Sticky Note8**
    - Describes this example as useful for quick replies in email clients.

---

### 3. Summary Table

| Node Name                    | Node Type              | Functional Role                                   | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                                                          |
|------------------------------|------------------------|-------------------------------------------------|------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger         | Entry point for workflow execution               | None                         | LoadMP3, Set-ex4, ChatGPT-ex        |                                                                                                    |
| LoadMP3                      | Read Binary File       | Loads mp3 audio file for transcription           | When clicking "Execute Workflow" | Whisper-transcribe                 | ## Whisper-1 example<br>### Prepare your audio file and send it to whisper-1 transcription model  |
| Whisper-transcribe           | HTTP Request           | Sends audio to Whisper model for transcription   | LoadMP3                      | Text-example                        |                                                                                                    |
| Text-example                 | Code                   | Provides example text input for AI models        | Whisper-transcribe            | davinci-003-complete, ChatGPT-ex1.1, ChatGPT-ex2, Code-ex3.1 | ## An example of transcribed text<br>### Please pause this node when using real audio files       |
| davinci-003-complete         | OpenAI                 | Legacy text completion with Davinci-003           | Text-example                 | davinci-003-edit                   | ## The old way of using text completion and text edit<br>### Davinci model is 10 times more expensive then ChatGPT, consider switching to the new API:<br>https://openai.com/blog/introducing-chatgpt-and-whisper-apis |
| davinci-003-edit             | OpenAI                 | Edits text by translating to German               | davinci-003-complete         | None                              |                                                                                                    |
| ChatGPT-ex1.1                | OpenAI                 | ChatGPT text summarization (user-only content)   | Text-example                 | ChatGPT-ex1.2                     | ## ChatGPT example 1.1 and 1.2<br>### Write a Tl;dr of the text input<br>### Translate it to German<br>### only user content provided |
| ChatGPT-ex1.2                | OpenAI                 | ChatGPT translation to German                      | ChatGPT-ex1.1                | None                              |                                                                                                    |
| ChatGPT-ex2                  | OpenAI                 | ChatGPT summary with system instruction and emojis | Text-example                 | None                              | ## ChatGPT example 2<br>### Use system content to provide general instruction<br>### Manual setup of system and user content |
| Code-ex3.1                  | Code                   | Prepares messages array for ChatGPT prompt chaining | Text-example                 | ChatGPT-ex3.1                     | ## ChatGPT example 3.1<br>### When using ChatGPT programmatically, create an array of system / user / assistant contents and append them one after another<br>### Call ChatGPT API via HTTP Request node to provide all messages at once |
| ChatGPT-ex3.1               | HTTP Request           | Calls ChatGPT API with full messages array        | Code-ex3.1                  | ChatGPT-ex3.2                     |                                                                                                    |
| ChatGPT-ex3.2               | OpenAI                 | Generates DALLE-2 prompt for comic style cover image | ChatGPT-ex3.1               | DALLE-ex3.3                      | ## ChatGPT example 3.2 & DALLE-2 example 3.3<br>### Use ChatGPT to create a prompt for a cover image of the Tl;dr message<br>### Use OpenAI node to generate 4 images using the auto-generated prompt |
| DALLE-ex3.3                 | OpenAI                 | Generates 4 comic style images using DALLE-2      | ChatGPT-ex3.2               | None                              |                                                                                                    |
| Set-ex4                     | Set                    | Sets parameters for code generation prompt        | When clicking "Execute Workflow" | ChatGPT-ex4                     |                                                                                                    |
| ChatGPT-ex4                 | OpenAI                 | Generates HTML code with SVG shapes                | Set-ex4                     | HTML-ex4                         | ## ChatGPT example 4<br>### Generate HTML code that contains SVG image                            |
| HTML-ex4                    | HTML                   | Renders generated HTML content                      | ChatGPT-ex4                 | None                              |                                                                                                    |
| ChatGPT-ex                  | OpenAI                 | Generates multiple short replies for email clients | When clicking "Execute Workflow" | None                          | ## ChatGPT example 5<br>### Provide several outputs. Useful for quick replies (i.e. in Gmail / Outlook) |
| Sticky Note                 | Sticky Note            | Provides notes on legacy Davinci model usage       | None                        | None                              | ## The old way of using text completion and text edit<br>### Davinci model is 10 times more expensive then ChatGPT, consider switching to the new API:<br>https://openai.com/blog/introducing-chatgpt-and-whisper-apis |
| Sticky Note1                | Sticky Note            | Whisper usage instruction                           | None                        | None                              | ## Whisper-1 example<br>### Prepare your audio file and send it to whisper-1 transcription model  |
| Sticky Note2                | Sticky Note            | Note on transcribed text usage                      | None                        | None                              | ## An example of transcribed text<br>### Please pause this node when using real audio files       |
| Sticky Note3                | Sticky Note            | ChatGPT example 1.1 and 1.2 summary and translate  | None                        | None                              | ## ChatGPT example 1.1 and 1.2<br>### Write a Tl;dr of the text input<br>### Translate it to German<br>### only user content provided |
| Sticky Note4                | Sticky Note            | ChatGPT example 2 system content usage              | None                        | None                              | ## ChatGPT example 2<br>### Use system content to provide general instruction<br>### Manual setup of system and user content |
| Sticky Note5                | Sticky Note            | ChatGPT programmatic message array and HTTP request | None                        | None                              | ## ChatGPT example 3.1<br>### When using ChatGPT programmatically, create an array of system / user / assistant contents and append them one after another<br>### Call ChatGPT API via HTTP Request node to provide all messages at once |
| Sticky Note6                | Sticky Note            | ChatGPT prompt for DALLE-2 image generation         | None                        | None                              | ## ChatGPT example 3.2 & DALLE-2 example 3.3<br>### Use ChatGPT to create a prompt for a cover image of the Tl;dr message<br>### Use OpenAI node to generate 4 images using the auto-generated prompt |
| Sticky Note7                | Sticky Note            | ChatGPT code generation example                      | None                        | None                              | ## ChatGPT example 4<br>### Generate HTML code that contains SVG image                            |
| Sticky Note8                | Sticky Note            | Multiple quick replies example                        | None                        | None                              | ## ChatGPT example 5<br>### Provide several outputs. Useful for quick replies (i.e. in Gmail / Outlook) |
| Sticky Note9                | Sticky Note            | Workflow execution advice                             | None                        | None                              | ## Do not run the whole workflow, it's rather slow<br>### Better execute the last node of each branch or simply disconnect branches that are not needed |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Name: `When clicking "Execute Workflow"`
   - Purpose: Start the workflow manually.

2. **Whisper Transcription Branch (Disabled by default)**
   - Create a **Read Binary File** node:
     - Name: `LoadMP3`
     - Parameters: Set file path to your mp3 audio file with voice.
     - Disabled: Yes (default)
   - Connect `When clicking "Execute Workflow"` → `LoadMP3`.
   - Create an **HTTP Request** node:
     - Name: `Whisper-transcribe`
     - HTTP Method: POST
     - URL: `https://api.openai.com/v1/audio/transcriptions`
     - Authentication: Use OpenAI API credentials.
     - Body Type: multipart-form-data
     - Body Parameters:
       - model: whisper-1
       - file: from binary data input (field name "data")
     - Disabled: Yes (default)
   - Connect `LoadMP3` → `Whisper-transcribe`.
   - Create a **Code** node:
     - Name: `Text-example`
     - Purpose: Provide example text or receive transcribed text.
     - JavaScript code returns JSON with example text content.
   - Connect `Whisper-transcribe` → `Text-example`.

3. **Legacy Davinci Text Completion & Edit**
   - Create an **OpenAI** node:
     - Name: `davinci-003-complete`
     - Model: text-davinci-003
     - Operation: Completion
     - Prompt: `={{ $json.text }}\n\nTl;dr:`
     - Max Tokens: 500
     - Credentials: OpenAI API
   - Connect `Text-example` → `davinci-003-complete`.
   - Create an **OpenAI** node:
     - Name: `davinci-003-edit`
     - Operation: Edit
     - Input: `={{ $json.text }}`
     - Instruction: `translate to German`
     - Credentials: OpenAI API
   - Connect `davinci-003-complete` → `davinci-003-edit`.
   - Add a Sticky Note near these nodes with the note on Davinci cost and link.

4. **Basic ChatGPT Examples (Summarize & Translate)**
   - Create an **OpenAI** node:
     - Name: `ChatGPT-ex1.1`
     - Resource: Chat Completion
     - Messages: Single user message with content `Write a Tl;dr of the following text: {{ $json.text }}`
     - Max Tokens: 500
     - Credentials: OpenAI API
   - Connect `Text-example` → `ChatGPT-ex1.1`.
   - Create an **OpenAI** node:
     - Name: `ChatGPT-ex1.2`
     - Resource: Chat Completion
     - Messages: Single user message with content `Translate to German the following text: {{ $json.message.content }}`
     - Max Tokens: 500
     - Credentials: OpenAI API
   - Connect `ChatGPT-ex1.1` → `ChatGPT-ex1.2`.
   - Add Sticky Note describing user-only content examples.

5. **Advanced ChatGPT Example with System Instructions**
   - Create an **OpenAI** node:
     - Name: `ChatGPT-ex2`
     - Resource: Chat Completion
     - Messages: Array with:
       - system role: "You are an assistant. Always add 5 emojis to the end of your answer."
       - user role: "Write tl;dr of the following text: {{ $json.text }}"
     - Max Tokens: 500
     - Temperature: 0.8
     - Credentials: OpenAI API
   - Connect `Text-example` → `ChatGPT-ex2`.
   - Add Sticky Note explaining system content usage.

6. **Programmatic Prompt Chaining and DALLE-2 Image Generation**
   - Create a **Code** node:
     - Name: `Code-ex3.1`
     - JavaScript code:
       ```
       var intext = $input.first().json;
       var messages = [
         {"role": "system", "content": "You are a helpful assistant. Write a Tl;dr of each user message"},
         {"role": "user", "content": intext.text}
       ];
       return { "messages": messages };
       ```
   - Connect `Text-example` → `Code-ex3.1`.
   - Create an **HTTP Request** node:
     - Name: `ChatGPT-ex3.1`
     - POST to `https://api.openai.com/v1/chat/completions`
     - Body parameters: model `gpt-3.5-turbo`, temperature 0.8, max_tokens 500, n=1, messages from `Code-ex3.1`.
     - Authentication: OpenAI API credentials.
   - Connect `Code-ex3.1` → `ChatGPT-ex3.1`.
   - Create an **OpenAI** node:
     - Name: `ChatGPT-ex3.2`
     - Resource: Chat Completion
     - Messages:
       - system role: "You are now a DALLE-2 prompt generation tool that will generate a suitable prompt. Write a prompt to create a cover image relevant to the user input. The image should be in a comic style of the 60-s."
       - user role: output content from `ChatGPT-ex3.1`
     - Max Tokens: 500
     - Temperature: 0.8
     - Credentials: OpenAI API
   - Connect `ChatGPT-ex3.1` → `ChatGPT-ex3.2`.
   - Create an **OpenAI** node:
     - Name: `DALLE-ex3.3`
     - Resource: Image Generation
     - Prompt: `={{ $json.message.content }}`
     - Number of images: 4
     - Size: 512x512
     - Credentials: OpenAI API
   - Connect `ChatGPT-ex3.2` → `DALLE-ex3.3`.
   - Add Sticky Notes describing prompt chaining and DALLE-2 usage.

7. **ChatGPT Code Generation and HTML Rendering**
   - Connect `When clicking "Execute Workflow"` → `Set-ex4`.
   - Create a **Set** node:
     - Name: `Set-ex4`
     - Set three string fields:
       - model: `code-davinci-002`
       - suffix: `</svg>`
       - prompt: `Create an HTML code with and SVG tag that contains random shapes of various colors. Include triangles, lines, ellipses and other shapes`
   - Connect `Set-ex4` → `ChatGPT-ex4`.
   - Create an **OpenAI** node:
     - Name: `ChatGPT-ex4`
     - Model: `gpt-3.5-turbo-0301`
     - Resource: Chat Completion
     - Messages: Single user content with prompt from `Set-ex4`
     - Max Tokens: 500
     - Temperature: 0.5
     - Credentials: OpenAI API
   - Connect `ChatGPT-ex4` → `HTML-ex4`.
   - Create an **HTML** node:
     - Name: `HTML-ex4`
     - HTML content source: `{{$json.message.content}}`
   - Add Sticky Note describing code generation example.

8. **Multi-Answer Quick Replies**
   - Connect `When clicking "Execute Workflow"` → `ChatGPT-ex`.
   - Create an **OpenAI** node:
     - Name: `ChatGPT-ex`
     - Model: `gpt-3.5-turbo-0301`
     - Resource: Chat Completion
     - Messages:
       - system role: "Act as an e-mail client. Provide five to eight word answers to a given user messages."
       - user role: Sample email text (e.g. from Jack)
     - Max Tokens: 15
     - Temperature: 0.8
     - Number of completions: 3
     - Credentials: OpenAI API
   - Add Sticky Note describing multi-answer usage.

9. **General Settings**
   - Workflow settings: Disable saving data success execution.
   - Apply OpenAI API credentials to all relevant nodes.
   - Add notes or sticky notes as per original.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The old way of using text completion and text edit. Davinci model is 10 times more expensive than ChatGPT. Consider switching to the new API: https://openai.com/blog/introducing-chatgpt-and-whisper-apis | Sticky Note on Davinci nodes.                                                                     |
| Whisper-1 example: Prepare your audio file and send it to Whisper-1 transcription model.                                   | Sticky Note near Whisper nodes.                                                                   |
| An example of transcribed text. Please pause this node when using real audio files.                                        | Sticky Note near Text-example node.                                                               |
| ChatGPT example 1.1 and 1.2: Write a Tl;dr of the text input, translate it to German, only user content provided.          | Sticky Note near ChatGPT examples 1.1 and 1.2 nodes.                                              |
| ChatGPT example 2: Use system content to provide general instruction. Manual setup of system and user content.             | Sticky Note near ChatGPT-ex2 node.                                                                |
| ChatGPT example 3.1: When using ChatGPT programmatically, create an array of system/user/assistant contents and append them one after another. Call ChatGPT API via HTTP Request node to provide all messages at once. | Sticky Note near Code-ex3.1 and ChatGPT-ex3.1 nodes.                                              |
| ChatGPT example 3.2 & DALLE-2 example 3.3: Use ChatGPT to create a prompt for a cover image of the Tl;dr message. Use OpenAI node to generate 4 images using the auto-generated prompt. | Sticky Note near ChatGPT-ex3.2 and DALLE-ex3.3 nodes.                                             |
| ChatGPT example 4: Generate HTML code that contains SVG image.                                                             | Sticky Note near Set-ex4, ChatGPT-ex4, and HTML-ex4 nodes.                                        |
| ChatGPT example 5: Provide several outputs. Useful for quick replies (e.g. in Gmail / Outlook).                             | Sticky Note near ChatGPT-ex node.                                                                 |
| Do not run the whole workflow, it’s rather slow. Better execute the last node of each branch or disconnect unused branches.| Sticky Note near the start node and general workflow advice.                                      |

---

This documentation provides a full structural and operational reference for the "OpenAI examples: ChatGPT, DALLE-2, Whisper-1 – 5-in-1" n8n workflow, enabling reproduction, modification, and integration with awareness of potential pitfalls.