Convert LLM Output into Rich Telegram Messages — Auto-Media & Smart Chunking

https://n8nworkflows.xyz/workflows/convert-llm-output-into-rich-telegram-messages---auto-media---smart-chunking-3961


# Convert LLM Output into Rich Telegram Messages — Auto-Media & Smart Chunking

### 1. Workflow Overview

This workflow is designed to convert long, potentially mixed-media output from a large language model (LLM) or any upstream source into Telegram-compatible messages for chatbots, AI assistants, or notification services. It processes text that may include URLs linking to images, audio, or videos, intelligently splitting, batching, and sending media or text messages through Telegram's API while respecting message size limits and media types.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered via an Execute Workflow node from a parent flow, receiving `chatId` and `output` string input.
- **1.2 Media Link Extraction & Deduplication:** Extracts and classifies media URLs from the input text.
- **1.3 Media Handling:** Routes each media URL by type (image, audio, video) and sends via Telegram's respective media-sending APIs.
- **1.4 Text Handling:** For text without media or residual text after link extraction, checks length; if too long, chunks and sends sequentially to respect Telegram's 1,000-character limit; otherwise sends directly.
- **1.5 Message Dispatch and Looping:** Uses batching and looping nodes to send media and text chunks sequentially, ensuring ordered delivery and no data loss.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives input parameters `chatId` (destination chat identifier) and `output` (the message string potentially containing media URLs) from a parent workflow that invokes this workflow.
- **Nodes Involved:** `When Executed by Another Workflow`
- **Node Details:**
  - **Type:** Execute Workflow Trigger
  - **Role:** Entry point for the workflow, accepting inputs from a parent workflow.
  - **Configuration:** Defines inputs expected: a number type `chatId` and a string `output`.
  - **Input/Output:** No input; outputs an item with `.json.chatId` and `.json.output`.
  - **Edge Cases:** If inputs are missing or malformed, subsequent nodes may fail or produce empty results.
  - **Version:** 1.1
  - **Notes:** This node is essential for integration; the parent workflow must provide correct inputs.

#### 2.2 Media Link Extraction & Deduplication

- **Overview:** Parses the `output` string to find all URLs ending with specific file extensions, classifies these URLs by media type (image/audio/video), and removes duplicates.
- **Nodes Involved:** `Extract Links`
- **Node Details:**
  - **Type:** Code (JavaScript)
  - **Role:** Extracts and classifies media links from text.
  - **Configuration:** Uses regex to find URLs with media file extensions; categorizes into `image`, `audio`, or `video` arrays based on extension; removes duplicate URLs.
  - **Key Expressions:** Regex pattern to identify URLs and extracted file extension; arrays of extensions per media type.
  - **Input:** Receives `.json.output` text from `When Executed by Another Workflow`.
  - **Output:** Emits an array under `.json.files` with objects `{ url, type }`.
  - **Edge Cases:** URLs without recognized extensions are tagged as 'other' and ignored downstream; malformed URLs or empty input yield empty arrays.
  - **Version:** 2
  - **Sticky Note:** None.

#### 2.3 Media Presence Check and Routing

- **Overview:** Determines if any media links were found; if none, routes to text processing; if media present, splits and loops over media links for individual sending.
- **Nodes Involved:** `If no links`, `Split Out the Links`, `Loop Over Links`
- **Node Details:**
  - **If no links:**
    - **Type:** If
    - **Role:** Checks if `.json.files` array is empty.
    - **Configuration:** Condition checks if the `files` array is empty.
    - **Output:** If true (no links), routes to text handling; else routes to splitting links.
    - **Edge Cases:** Empty or missing `files` property.
  - **Split Out the Links:**
    - **Type:** SplitOut
    - **Role:** Splits the array of media files into individual items for processing.
    - **Parameters:** Splits on `files` field.
  - **Loop Over Links:**
    - **Type:** SplitInBatches
    - **Role:** Loops over each media link item for sequential processing.
    - **Edge Cases:** Large number of media links may affect throughput; batch size defaults to 1 (not explicitly set).
  - **Version:** If node v2.2, SplitOut v1, SplitInBatches v3.

#### 2.4 Media Type Switch and Sending

- **Overview:** Routes each media link according to its type (`image`, `audio`, `video`) and sends it using the correct Telegram API operation.
- **Nodes Involved:** `Check Link Type`, `Send back an image`, `Send back an audio`, `Send back a video`, `Limit`
- **Node Details:**
  - **Check Link Type:**
    - **Type:** Switch
    - **Role:** Routes items based on `.json.type` field.
    - **Configuration:** Outputs named `image`, `audio`, `video` corresponding to the type.
  - **Send back an image/audio/video:**
    - **Type:** Telegram node
    - **Role:** Sends media to Telegram chat via API.
    - **Configuration:** 
      - Uses `sendPhoto` for images, `sendAudio` for audio, and `sendVideo` for videos.
      - Passes `.json.url` as media file.
      - Uses the `chatId` from initial input.
      - Sets `parse_mode` to HTML for caption formatting (captions can be added via `additionalFields`).
    - **Credentials:** Uses a Telegram API credential named "Telegram account (SCP)".
    - **Edge Cases:** 
      - Media URLs inaccessible or invalid cause API errors.
      - Telegram API limits or timeouts.
  - **Limit:**
    - **Type:** Limit node
    - **Role:** Controls throughput of messages sent to avoid Telegram API rate limits.
    - **Configuration:** Default limit (not customized) applies.
  - **Version:** Telegram nodes mostly v1.1 or v1.2; Switch v3.2; Limit v1.
  - **Sticky Note:** There is a sticky note covering the media sending nodes:  
    > "Send links  
    > Send links to chat as separate messages  
    > with appropriate type (image, audio, video)"

#### 2.5 Text Length Check and Processing

- **Overview:** For text without media or residual text, checks if the text length exceeds Telegram's 1,000-character limit; if yes, splits text into chunks by lines to avoid message truncation and sends sequentially.
- **Nodes Involved:** `If text too long`, `Split large text by chunks`, `Split Out the Chunks`, `Loop Over Text Chunks`, `Send Text Chunk`, `Send Text`
- **Node Details:**
  - **If text too long:**
    - **Type:** If
    - **Role:** Checks if `.json.output.length > 1000`.
    - **Output:** Routes to chunk splitting if true; else sends text directly.
  - **Split large text by chunks:**
    - **Type:** Code (JavaScript)
    - **Role:** Splits large text into chunks not exceeding 1000 characters, breaking at line boundaries to preserve readability.
    - **Configuration:** 
      - Splits input on newline `\n` characters.
      - Accumulates lines into chunks <= 1000 characters.
      - Returns array of chunk strings.
  - **Split Out the Chunks:**
    - **Type:** SplitOut
    - **Role:** Converts array of chunks into individual items for looping.
  - **Loop Over Text Chunks:**
    - **Type:** SplitInBatches
    - **Role:** Iteratively sends each text chunk sequentially.
  - **Send Text Chunk:**
    - **Type:** Telegram node
    - **Role:** Sends each chunk as a separate Telegram text message.
    - **Configuration:** 
      - Uses `parse_mode` HTML.
      - Uses `chatId` from input.
      - Sends `.json.chunk` as message text.
  - **Send Text:** (Used when text length ≤ 1000)
    - **Type:** Telegram node
    - **Role:** Sends the entire text in a single message.
    - **Configuration:** Same as `Send Text Chunk` but sends the whole `.json.output`.
  - **Version:** If node v2.2; Code v2; SplitOut v1; SplitInBatches v3; Telegram nodes v1.2.
  - **Sticky Notes:**
    - For the chunked sending section:  
      > "Send a long message  
      > Separate long message by chunks and send them one by one"
    - For the short message sending node:  
      > "Send a short message  
      > Deliver a short message (default flow)"

---

### 3. Summary Table

| Node Name                | Node Type                   | Functional Role                                    | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                   |
|--------------------------|-----------------------------|---------------------------------------------------|----------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger      | Entry point; receives `chatId` and `output` from parent workflow | None                             | Extract Links                        |                                                                                              |
| Extract Links            | Code                        | Extracts, deduplicates, and classifies media URLs | When Executed by Another Workflow | If no links                         |                                                                                              |
| If no links              | If                          | Checks if media links exist                        | Extract Links                    | If text too long, Split Out the Links |                                                                                              |
| Split Out the Links      | SplitOut                    | Splits media links array into individual items    | If no links                     | Loop Over Links                     |                                                                                              |
| Loop Over Links          | SplitInBatches              | Loops over each media link for sending             | Split Out the Links             | Limit, Check Link Type              |                                                                                              |
| Limit                   | Limit                       | Controls sending throughput to Telegram API       | Loop Over Links                 | If text too long                    |                                                                                              |
| Check Link Type          | Switch                      | Routes each link by media type (image/audio/video) | Loop Over Links                 | Send back an image/audio/video       |                                                                                              |
| Send back an image       | Telegram                    | Sends image media to Telegram                       | Check Link Type (image)          | Loop Over Links                    | Send links. Send links to chat as separate messages with appropriate type (image, audio, video) |
| Send back an audio       | Telegram                    | Sends audio media to Telegram                       | Check Link Type (audio)          | Loop Over Links                    | Send links. Send links to chat as separate messages with appropriate type (image, audio, video) |
| Send back a video        | Telegram                    | Sends video media to Telegram                       | Check Link Type (video)          | Loop Over Links                    | Send links. Send links to chat as separate messages with appropriate type (image, audio, video) |
| If text too long         | If                          | Checks if text length exceeds 1000 chars           | Limit, If no links               | Split large text by chunks, Send Text |                                                                                              |
| Split large text by chunks | Code                      | Splits long text into <= 1000-char chunks          | If text too long                | Split Out the Chunks               | Send a long message. Separate long message by chunks and send them one by one                |
| Split Out the Chunks     | SplitOut                    | Splits text chunk array into individual items       | Split large text by chunks       | Loop Over Text Chunks              | Send a long message. Separate long message by chunks and send them one by one                |
| Loop Over Text Chunks    | SplitInBatches              | Loops over text chunks for sequential sending       | Split Out the Chunks             | Send Text Chunk                   | Send a long message. Separate long message by chunks and send them one by one                |
| Send Text Chunk          | Telegram                    | Sends each text chunk as a Telegram message         | Loop Over Text Chunks            | Loop Over Text Chunks (continue) | Send a long message. Separate long message by chunks and send them one by one                |
| Send Text                | Telegram                    | Sends short text message directly                    | If text too long (false branch) | None                             | Send a short message. Deliver a short message (default flow)                                |
| Sticky Note              | Sticky Note                 | Annotation covering media sending nodes             | None                           | None                             | Send links. Send links to chat as separate messages with appropriate type (image, audio, video) |
| Sticky Note1             | Sticky Note                 | Annotation covering long text message sending nodes | None                           | None                             | Send a long message. Separate long message by chunks and send them one by one                |
| Sticky Note2             | Sticky Note                 | Annotation covering short text message node          | None                           | None                             | Send a short message. Deliver a short message (default flow)                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it** "TelegramRichOutput".

2. **Add an Execute Workflow Trigger node:**
   - Name: `When Executed by Another Workflow`
   - Configure workflow inputs:  
     - Add `chatId` as Number  
     - Add `output` as String  
   - This node will receive inputs from the parent workflow.

3. **Add a Code node to extract media links:**
   - Name: `Extract Links`
   - Paste this JavaScript code:
     ```js
     const text = $input.first().json.output ?? "";

     const IMG = ['jpg','jpeg','png','gif','bmp','webp','tif','tiff'];
     const AUD = ['mp3','wav','m4a','flac','ogg','aac','opus'];
     const VID = ['mp4','mov','mkv','webm','avi','mpeg','mpg'];

     const re = /(https?:\/\/[^\s"'<>]+\.(?<ext>[A-Za-z0-9]{2,5}))(?:\?[^\s"'<>]*)?/gi;

     const seen  = new Set();
     const files = [];

     for (const m of text.matchAll(re)) {
       const url = m[1];
       if (seen.has(url)) continue;
       seen.add(url);

       const ext = (m.groups.ext || '').toLowerCase();

       let type = 'other';
       if (IMG.includes(ext))  type = 'image';
       else if (AUD.includes(ext)) type = 'audio';
       else if (VID.includes(ext)) type = 'video';

       files.push({ url, type });
     }

     return [{ json: { files } }];
     ```
   - Connect `When Executed by Another Workflow` to this node.

4. **Add an If node to check if media links exist:**
   - Name: `If no links`
   - Condition: Check if `files` array is empty.
     - Expression: `{{ $json.files.length === 0 }}`
   - Connect `Extract Links` to this node.

5. **Add a SplitOut node to split media links:**
   - Name: `Split Out the Links`
   - Field to split out: `files`
   - Connect the FALSE output of `If no links` to this node.

6. **Add a SplitInBatches node to loop over links:**
   - Name: `Loop Over Links`
   - Connect `Split Out the Links` to this node.

7. **Add a Limit node to control sending rate:**
   - Name: `Limit`
   - Default parameters (can be customized later)
   - Connect `Loop Over Links` to `Limit`.

8. **Add a Switch node to route media by type:**
   - Name: `Check Link Type`
   - Add 3 outputs with conditions matching `.json.type`:  
     - `image` (endsWith "image")  
     - `audio` (equals "audio")  
     - `video` (equals "video")
   - Connect `Limit` to this node.

9. **Add three Telegram nodes for media sending:**
   - Names: `Send back an image`, `Send back an audio`, `Send back a video`
   - For each:
     - Credential: Use Telegram API credentials (set up Telegram Bot token in n8n credentials)
     - Operation:  
       - Image → `sendPhoto`  
       - Audio → `sendAudio`  
       - Video → `sendVideo`
     - File: `={{ $json.url }}`
     - Chat ID: `={{ $('When Executed by Another Workflow').item.json.chatId }}`
     - Additional Fields: Set `parse_mode` to `HTML`
   - Connect these to the corresponding outputs of `Check Link Type`.
   - Connect all three Telegram media nodes back to `Loop Over Links` for continued processing (to loop through all links).

10. **Add an If node to check if text is too long:**
    - Name: `If text too long`
    - Condition:  
      Expression: `{{ $('When Executed by Another Workflow').item.json.output.length > 1000 }}`
    - Connect the TRUE and FALSE outputs from `If no links` and `Limit` nodes to this node accordingly.

11. **Add a Code node to split long text into chunks:**
    - Name: `Split large text by chunks`
    - JavaScript code:
      ```js
      const inputText = $('When Executed by Another Workflow').first().json.output || "";
      const lines = inputText.split("\n");

      let chunks = [];
      let currentChunk = "";

      for (const line of lines) {
        if ((currentChunk + line + "\n").length > 1000) {
          if (currentChunk) chunks.push(currentChunk.trim());
          currentChunk = line + "\n";
        } else {
          currentChunk += line + "\n";
        }
      }

      if (currentChunk) chunks.push(currentChunk.trim());

      return { output: chunks.map(chunk => ({ chunk })) };
      ```
    - Connect the TRUE output of `If text too long` to this node.

12. **Add SplitOut node to split chunks:**
    - Name: `Split Out the Chunks`
    - Field to split out: `output`
    - Connect `Split large text by chunks` to this node.

13. **Add SplitInBatches node to loop over text chunks:**
    - Name: `Loop Over Text Chunks`
    - Connect `Split Out the Chunks` to this node.

14. **Add Telegram node to send each text chunk:**
    - Name: `Send Text Chunk`
    - Credential: Telegram API (same as media nodes)
    - Text: `={{ $json.chunk }}`
    - Chat ID: `={{ $('When Executed by Another Workflow').item.json.chatId }}`
    - Additional Fields: `parse_mode` = `HTML`, `appendAttribution` = false
    - Connect `Loop Over Text Chunks` to this node.
    - Connect `Send Text Chunk` back to `Loop Over Text Chunks` to iterate all chunks.

15. **Add Telegram node to send short text messages directly:**
    - Name: `Send Text`
    - Credential: Telegram API
    - Text: `={{ $('When Executed by Another Workflow').item.json.output }}`
    - Chat ID: `={{ $('When Executed by Another Workflow').item.json.chatId }}`
    - Additional Fields: `parse_mode` = `HTML`, `appendAttribution` = false
    - Connect FALSE output of `If text too long` to this node.

16. **Add Sticky Notes as desired for documentation within n8n UI:**
    - For media sending nodes: "Send links to chat as separate messages with appropriate type (image, audio, video)"
    - For chunked text sending nodes: "Send a long message. Separate long message by chunks and send them one by one"
    - For short text sending node: "Send a short message. Deliver a short message (default flow)"

17. **Set Telegram API credential:**
    - Create a Telegram API credential in n8n using the bot token from `@BotFather`.
    - Assign this credential to all Telegram nodes.

18. **Test the workflow:**
    - Trigger via the parent workflow or manual execution by providing `chatId` and `output` parameters.
    - Verify media links are correctly extracted, routed, and sent.
    - Verify text splitting and message sending as per text length.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| This workflow requires n8n version 1.0 or higher with the built-in Telegram node installed.                                                 | Prerequisite for running this workflow                                                  |
| Telegram Bot must be created with @BotFather, and its API token saved in n8n credentials.                                                   | Telegram API integration setup                                                          |
| The Telegram API limits text messages to 4096 characters, but this workflow uses a conservative 1000-character limit for chunking messages. | Telegram API documentation and best practices                                           |
| Customize character limits, regex, and captions by editing the relevant nodes (`If text too long`, `Extract Links`, Telegram nodes).       | Workflow customization tips                                                             |
| The workflow supports sending images, audio, and video inline with Telegram’s native preview/player support.                                | Enhances user experience in Telegram chats                                              |
| For throughput vs. message order control, adjust batch sizes in `SplitInBatches` nodes.                                                     | Performance tuning based on use case                                                    |
| Useful blog for Telegram Bot API: https://core.telegram.org/bots/api                                                                   | Telegram API official documentation                                                    |
| Demo and explanation of splitting long messages for Telegram: https://telegra.ph/How-to-Split-Long-Messages-in-Telegram-09-20           | Additional resource on message chunking                                                |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected material and only handles legal, public data.