Telegram Interior Design Studio with Furniture Cataloging & Gemini AI Design Generation

https://n8nworkflows.xyz/workflows/telegram-interior-design-studio-with-furniture-cataloging---gemini-ai-design-generation-8772


# Telegram Interior Design Studio with Furniture Cataloging & Gemini AI Design Generation

### 1. Workflow Overview

This workflow implements a comprehensive Telegram-based Interior Design Studio assistant integrated with a furniture cataloging system and AI-powered interior design generation using Google Gemini AI. It is designed to manage user interactions via Telegram, analyze images and text messages related to interior design, and coordinate database operations for furniture and room documentation. It further supports generating new interior design concepts or modifying existing ones by leveraging AI-driven image generation.

The workflow logic can be grouped into these main blocks:

- **1.1 Input Reception & Preprocessing:**  
  Triggered by Telegram messages (text and/or images), this block handles the initial reception and preprocessing, including file uploads and conditional routing.

- **1.2 Image Analysis:**  
  Uses OpenAI image analysis to classify the image as either furniture or room and extract detailed descriptive data accordingly.

- **1.3 AI Agent Processing & Routing:**  
  A central AI Agent node interprets the user message combined with image analysis results to decide on the next steps: catalog furniture, document room, or prepare AI design generation/modification.

- **1.4 Furniture Cataloging:**  
  Specialized nodes and tools parse furniture analysis results and create structured entries in the furniture catalog database.

- **1.5 Room Documentation:**  
  Dedicated nodes parse room analysis results and create structured room entries in the room database.

- **1.6 AI Design Generation & Modification:**  
  This block handles the generation or modification of interior design images using Google Gemini API, including payload construction, API calling, image extraction, saving generated images, and sending results back to the user.

- **1.7 User Communication:**  
  Nodes dedicated to sending text and photo messages back to the user via Telegram, including confirmations and requests for clarifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

- **Overview:**  
  Captures incoming Telegram messages (with optional images), downloads images, uploads them to Supabase storage, and prepares data for analysis.

- **Nodes Involved:**  
  - Telegram Trigger  
  - If (checks if message contains binary/image data)  
  - HTTP Request1 (uploads image to Supabase)  
  - Code in JavaScript (constructs public URL for uploaded image)  
  - Edit Fields1 / Edit Fields (sets ai_agent_input and image_analysis fields)

- **Node Details:**

  - **Telegram Trigger:**  
    - Type: Telegram message listener  
    - Configured to trigger on new Telegram messages, downloading any attached images.  
    - Uses Telegram API credentials for bot access.  
    - Output: Raw Telegram message JSON including binary image data if any.  
    - Edge cases: No image in message, message format errors.

  - **If (Check for image):**  
    - Type: Conditional node  
    - Checks if binary data exists in the message to distinguish image vs text-only messages.  
    - Branches workflow accordingly.  
    - Edge cases: Binary data structure unexpected.

  - **HTTP Request1 (Supabase Upload):**  
    - Type: HTTP POST binary upload  
    - Uploads image binary data to Supabase storage under `catalog-images` bucket, naming files with current timestamp and filename.  
    - Sends appropriate headers including content type and upsert flag.  
    - Credentials: Supabase API.  
    - Failure points: Network errors, auth failures, invalid data.

  - **Code in JavaScript:**  
    - Extracts file key from upload response and constructs a public URL for the image.  
    - Passes original Telegram message for context.  
    - Edge cases: Upload response missing expected fields.

  - **Edit Fields1 / Edit Fields:**  
    - Sets JSON properties `ai_agent_input` and `image_analysis` based on whether the message is text-only or includes image analysis results.  
    - Ensures downstream nodes receive consistent structured data.

---

#### 2.2 Image Analysis

- **Overview:**  
  Analyzes uploaded images to determine if they depict furniture or rooms, and extracts detailed descriptive metadata tailored for interior design cataloging or room documentation.

- **Nodes Involved:**  
  - Analyze image (OpenAI image analysis model)

- **Node Details:**

  - **Analyze image:**  
    - Type: OpenAI image analysis (custom prompt)  
    - Configuration: Uses GPT-4o-latest with image URL input. The prompt instructs to classify images as FURNITURE or ROOM, and extract detailed metadata accordingly.  
    - Output includes furniture item details or room characteristics in a structured text format.  
    - Edge cases: Poor image quality, ambiguous content, multiple items in one image.

---

#### 2.3 AI Agent Processing & Routing

- **Overview:**  
  Central AI agent merges user text and image analysis to interpret intent, route requests to appropriate database operations or AI generation, and prepares structured outputs.

- **Nodes Involved:**  
  - AI Agent (Langchain agent)  
  - AI Agent Tool (Furniture catalog database interface)  
  - Room Organiser (Room database interface)  
  - Get AI Generated Images (retrieve previous designs)  
  - Send a text message in Telegram  
  - Send a photo message in Telegram

- **Node Details:**

  - **AI Agent:**  
    - Type: Langchain agent with custom system prompt describing interior design assistant role and detailed logic for handling various user message types.  
    - Inputs: User text (`ai_agent_input`) and image analysis (`image_analysis`).  
    - Outputs: Routed instructions, including calls to furniture or room database tools, AI generation requests, or user communications.  
    - Contains complex logic for request type detection (catalog, room doc, AI generation/modification).  
    - Version: Langchain agent v2.2.  
    - Edge cases: Ambiguous user requests, missing data, conflicting instructions.

  - **AI Agent Tool:**  
    - Type: Langchain agentTool specialized in furniture catalog database operations.  
    - Receives detailed furniture analysis text from AI Agent.  
    - Creates structured entries in Supabase `catalog_products` table via Create Catalogue Row node.  
    - Includes field formatting rules and tag generation.  
    - Edge cases: Missing required fields, invalid formatting.

  - **Room Organiser:**  
    - Type: Langchain agentTool specialized in room database operations.  
    - Parses room analysis text and maps fields precisely to Supabase `rooms` table columns.  
    - Generates tags and validates data consistency.  
    - Edge cases: Partial room data, missing image URL, malformed input.

  - **Get AI Generated Images:**  
    - Type: Supabase query tool  
    - Retrieves previously generated AI images for modification requests.  
    - Used when user references prior designs.  
    - Edge cases: No previous designs found.

  - **Send a text message in Telegram:**  
    - Type: Telegram message sender  
    - Used for all textual user communications, confirmations, clarifications.  
    - Chat ID dynamically retrieved from Telegram Trigger node.  
    - Credentials: Telegram API.  
    - Edge cases: Telegram API rate limits or failures.

  - **Send a photo message in Telegram:**  
    - Type: Telegram photo sender  
    - Sends images (product photos, room images, generated designs) to user.  
    - Uses HTTP URLs or binary data.  
    - Edge cases: Image delivery failures.

---

#### 2.4 Furniture Cataloging

- **Overview:**  
  Processes individual furniture items extracted from image analysis and creates catalog entries with detailed metadata in the Supabase database.

- **Nodes Involved:**  
  - AI Agent Tool (agentTool)  
  - Create Catalogue Row (Supabase insert)  
  - Get Many Catalogue Rows (Supabase query)  
  - Sticky Note (Catalog Tools annotation)

- **Node Details:**

  - **Create Catalogue Row:**  
    - Type: Supabase Tool node for inserting into `catalog_products` table.  
    - Fields: product_name, product_type, category, style, design_era, primary_color, materials, size_category, room_types, ai_description, image_url, brand, tags.  
    - Values dynamically assigned from AI Agent Tool outputs.  
    - Credentials: Supabase.  
    - Edge cases: Missing required fields, duplicate entries, API errors.

  - **Get Many Catalogue Rows:**  
    - Type: Supabase Tool node for retrieving multiple catalog entries.  
    - Used for querying existing products, e.g., during AI generation request.  
    - Edge cases: Large data sets, pagination.

---

#### 2.5 Room Documentation

- **Overview:**  
  Processes room analysis data from AI to create structured room entries in the Supabase database.

- **Nodes Involved:**  
  - Room Organiser (agentTool)  
  - Create Room Row (Supabase insert)  
  - Get Many Room Rows (Supabase query)  
  - Sticky Note (Room Organiser Tools annotation)

- **Node Details:**

  - **Create Room Row:**  
    - Type: Supabase Tool node for inserting into `rooms` table.  
    - Fields: room_name, room_type, room_size, style, design_era, color_palette, lighting_type, architectural_features, flooring_type, existing_furniture, ai_description, image_url, tags.  
    - Values extracted precisely from Room Organiser agentTool output.  
    - Credentials: Supabase.  
    - Edge cases: Partial data, missing image URLs.

  - **Get Many Room Rows:**  
    - Type: Supabase Tool node to retrieve room entries, e.g., for AI generation context.  
    - Edge cases: Large data sets.

---

#### 2.6 AI Design Generation & Modification

- **Overview:**  
  Constructs AI prompt payloads, calls Google Gemini API (via Nano Banana wrapper), extracts generated images, stores them, and sends results back to the user.

- **Nodes Involved:**  
  - Output Organiser (OpenAI Chat to parse AI Agent output)  
  - Split Out (splits image URLs array)  
  - Binary Downloader (downloads referenced images)  
  - Binary Decoder (aggregates images, creates Nano Banana payload)  
  - Nanobanana Caller (HTTP request to Gemini API)  
  - Gen AI Image (extracts base64 image from Gemini response)  
  - Gen Image Save (uploads generated image to Supabase)  
  - Create AI Image row (records generation metadata)  
  - Send a photo message (Telegram sends generated image)  
  - Sticky Note (AI Gen Image Tool annotation)

- **Node Details:**

  - **Output Organiser:**  
    - Parses the structured AI Agent output text to extract fields: image_name, gemini_prompt, image_urls_array, image_urls string.  
    - Uses GPT-4.1-mini for structured JSON output.  
    - Edge cases: Malformed or incomplete AI Agent outputs.

  - **Split Out:**  
    - Splits the array of image URLs into individual items for downloading.

  - **Binary Downloader:**  
    - Downloads each referenced image URL for inclusion in AI generation payload.

  - **Binary Decoder:**  
    - Aggregates downloaded images and AI prompt text into a single payload for Nano Banana API.  
    - Converts images into inline base64 data as required by Gemini API.  
    - Edge cases: Missing images, large payload.

  - **Nanobanana Caller:**  
    - Sends the generation request to Google Gemini API via Nano Banana endpoint.  
    - Uses Google Palm Api credentials.  
    - Handles JSON body and headers.  
    - Edge cases: API rate limits, errors, malformed requests.

  - **Gen AI Image:**  
    - Extracts base64 encoded image data from Gemini API response.  
    - Converts to n8n binary data for saving and sending.  
    - Edge cases: No image in response, extraction failures.

  - **Gen Image Save:**  
    - Uploads generated image binary data to Supabase storage.  
    - Uses image_name as file name.  
    - Credentials: Supabase.  
    - Edge cases: Upload failures.

  - **Create AI Image row:**  
    - Records metadata of generated image into Supabase `ai_generated_images` table.  
    - Fields: image_name, image_url, original_prompt, ai_description.  
    - Edge cases: Duplicate entries, missing metadata.

  - **Send a photo message:**  
    - Sends generated image back to the user on Telegram with caption.  
    - Credentials: Telegram API.

---

#### 2.7 User Communication

- **Overview:**  
  Sends messages and photos to users via Telegram, providing confirmations, clarifications, and results.

- **Nodes Involved:**  
  - Send a text message in Telegram  
  - Send a photo message in Telegram  
  - Sticky Note1 (Telegram Text and Image sending tools annotation)

- **Node Details:**

  - Both nodes use Telegram API credentials.  
  - Text messages used for success confirmations, clarifications, and general communication.  
  - Photo messages used for showing catalog items, rooms, and AI-generated images.  
  - Edge cases: API limits, message formatting issues.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                                        | Input Node(s)                   | Output Node(s)                                  | Sticky Note                              |
|--------------------------|----------------------------------|-------------------------------------------------------|---------------------------------|------------------------------------------------|------------------------------------------|
| Telegram Trigger          | telegramTrigger                  | Entry point: receives Telegram messages                | -                               | If                                             |                                          |
| If                       | if                              | Checks if message contains image data                  | Telegram Trigger                | Edit Fields1 (text only) / HTTP Request1 (image) |                                          |
| Edit Fields1             | set                             | Sets ai_agent_input and image_analysis for text-only   | If (no image)                  | AI Agent                                        |                                          |
| HTTP Request1            | httpRequest                     | Uploads image binary to Supabase Storage               | If (image present)             | Code in JavaScript                              |                                          |
| Code in JavaScript       | code                            | Constructs public URL for uploaded image               | HTTP Request1                  | Analyze image                                   |                                          |
| Analyze image            | openAi (image analysis)          | Analyzes image content and classifies as furniture/room| Code in JavaScript             | Edit Fields                                     |                                          |
| Edit Fields              | set                             | Sets ai_agent_input and image_analysis after analysis  | Analyze image                  | AI Agent                                        |                                          |
| AI Agent                 | langchain agent                 | Core logic: interprets input, routes to database tools or AI generation | Edit Fields / Edit Fields1     | If1                                             |                                          |
| If1                      | if                              | Checks if output contains AI generation prompt         | AI Agent                      | Output Organiser                                |                                          |
| Output Organiser         | openAi chat                    | Extracts structured fields from AI Agent output text   | If1                          | Split Out                                       |                                          |
| Split Out                | splitOut                       | Splits images URLs array for download                   | Output Organiser              | Binary Downloader                               |                                          |
| Binary Downloader        | httpRequest                    | Downloads referenced images for AI generation payload  | Split Out                    | Binary Decoder                                  |                                          |
| Binary Decoder           | code                           | Aggregates images and text into Nano Banana API payload| Binary Downloader             | Nanobanana Caller                               | AI GEN IMAGE TOOL                        |
| Nanobanana Caller        | httpRequest                    | Calls Google Gemini API for image generation            | Binary Decoder               | Gen AI Image                                    |                                          |
| Gen AI Image             | code                           | Extracts base64 image from Gemini API response          | Nanobanana Caller             | Send a photo message, Gen Image Save            |                                          |
| Send a photo message     | telegramTool                   | Sends generated image to Telegram user                  | Gen AI Image                 | -                                              | Telegram Text and Image sending tools   |
| Gen Image Save           | httpRequest                    | Uploads generated image to Supabase Storage             | Gen AI Image                 | Create AI Image row                             |                                          |
| Create AI Image row      | supabase                       | Records generated image metadata in database            | Gen Image Save               | -                                              |                                          |
| AI Agent Tool            | langchain agentTool            | Furniture catalog database operations                    | AI Agent                    | Create Catalogue Row                            | CATALOGUE ORGANISER TOOLS               |
| Create Catalogue Row     | supabaseTool                   | Inserts furniture item into catalog_products table      | AI Agent Tool               | -                                              |                                          |
| Get Many Catalogue Rows  | supabaseTool                   | Retrieves multiple furniture catalog entries            | AI Agent Tool               | AI Agent Tool                                  |                                          |
| Room Organiser           | langchain agentTool            | Room database operations                                 | OpenAI Chat / AI Agent       | Create Room Row                                 | ROOM ORGANISER TOOLS                    |
| Create Room Row          | supabaseTool                   | Inserts room entry into rooms table                      | Room Organiser              | -                                              |                                          |
| Get Many Room Rows       | supabaseTool                   | Retrieves multiple room entries                          | Room Organiser              | Room Organiser                                 |                                          |
| Send a text message in Telegram | telegramTool                   | Sends textual messages to Telegram users                 | AI Agent Tool               | AI Agent Tool                                  | Telegram Text and Image sending tools   |
| Send a photo message in Telegram | telegramTool                   | Sends photo messages to Telegram users                   | AI Agent Tool               | AI Agent Tool                                  | Telegram Text and Image sending tools   |
| Sticky Note               | stickyNote                    | Annotations / labels                                    | -                           | -                                              | Various: Cataloguer, Room Organiser, AI Gen, Telegram tools|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure to listen for messages (update type: message)  
   - Enable "Download" to true for image retrieval  
   - Connect Telegram Bot API credentials (OAuth token for AI DESIGNER BOT)  

2. **Add If node to check for images:**  
   - Condition: Check if `$binary` object length > 0 (indicating image presence)  
   - If true: proceed with image upload flow  
   - If false: proceed with text-only flow  

3. **Image upload flow:**  
   - **HTTP Request1:** POST binary to Supabase Storage bucket `catalog-images`  
     - URL: `https://[your-supabase-project].supabase.co/storage/v1/object/catalog-images/{{$now}}-{{$binary.data.fileName}}`  
     - Headers: `Content-Type` from binary mimeType, `x-upsert: true`  
     - Authentication: Supabase API credentials  

   - **Code in JavaScript:**  
     - Extract uploaded file key from response  
     - Construct public URL: `https://[your-supabase-project].supabase.co/storage/v1/object/public/${fileKey}`  
     - Also pass original Telegram message JSON for context  

   - **Analyze image node:**  
     - Use OpenAI model (chatgpt-4o-latest) with custom prompt for furniture vs room analysis  
     - Input: public URL of uploaded image  
     - Output: structured text describing furniture or room  

   - **Edit Fields node:**  
     - Set `ai_agent_input` to JSON from JavaScript node  
     - Set `image_analysis` to analysis text from OpenAI node  

4. **Text-only flow:**  
   - **Edit Fields1 node:**  
     - Set `ai_agent_input` from `Telegram Trigger` text message content  
     - Set `image_analysis` to `"null"`  

5. **AI Agent node:**  
   - Use Langchain agent with detailed prompt describing full interior design assistant logic, including handling text-only, furniture cataloging, room documentation, AI generation and modification requests.  
   - Inputs: `ai_agent_input` and `image_analysis`  
   - Output: structured text including routing commands and AI generation prompts as needed  

6. **If1 node:**  
   - Check if AI Agent output contains `GEMINI_PROMPT:` to detect AI generation requests  
   - If yes: proceed to Output Organiser  
   - Else: handle normal catalog or room responses  

7. **Output Organiser (OpenAI Chat):**  
   - Parse AI Agent output to extract JSON fields: image_name, gemini_prompt, image_urls_array, image_urls string  
   - Use GPT-4.1-mini or equivalent  

8. **Split Out node:**  
   - Split `image_urls_array` into individual items for download  

9. **Binary Downloader:**  
   - Download each image from URLs for inclusion in AI generation request  

10. **Binary Decoder (Code):**  
    - Aggregate images into inline base64 format for Nano Banana API  
    - Combine with Gemini prompt to form payload  

11. **Nanobanana Caller (HTTP Request):**  
    - POST payload to Google Gemini API endpoint  
    - Use Google Palm API credentials  

12. **Gen AI Image (Code):**  
    - Extract base64 image data and description from Gemini response  
    - Prepare output with binary data for saving and sending  

13. **Gen Image Save (HTTP Request):**  
    - Upload generated image to Supabase storage in `catalog-images` bucket  
    - Use image_name for file naming  

14. **Create AI Image row (Supabase):**  
    - Insert new record in `ai_generated_images` table with image metadata  

15. **Send a photo message in Telegram:**  
    - Send generated image back to user with caption showing image name  

16. **Furniture Cataloging:**  
    - **AI Agent Tool:** Receive furniture analysis text from AI Agent  
    - **Create Catalogue Row:** Insert furniture items into `catalog_products` Supabase table with all required fields  
    - **Get Many Catalogue Rows:** For querying catalog when needed  

17. **Room Documentation:**  
    - **Room Organiser:** Receive room analysis text from AI Agent or OpenAI Chat  
    - **Create Room Row:** Insert room data into `rooms` Supabase table  
    - **Get Many Room Rows:** For querying rooms when needed  

18. **Send a text message in Telegram:**  
    - Used throughout for confirmations and clarifications  

19. **Credential Setup:**  
    - Telegram API credentials for bot  
    - Supabase API credentials for DB and storage  
    - OpenAI API credentials (GPT-4, GPT-4o)  
    - Google Palm API credentials for Gemini AI  

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| The AI Agent prompt contains comprehensive logic for handling all user interaction types, including detailed instructions on database operations and AI generation formatting. | Core logic embedded in "AI Agent" node prompt.                                                               |
| Image analysis prompt is specialized for interior design workflows, distinguishing furniture vs room images with structured output. | Analyze image node prompt.                                                                                     |
| The workflow uses Google Gemini API via Nano Banana API wrapper for AI image generation with text+image payloads.                  | Nanobanana Caller node configuration.                                                                         |
| Supabase tables `catalog_products`, `rooms`, and `ai_generated_images` are critical for data persistence.                         | Supabase tool nodes and field mappings in agent tools.                                                        |
| Telegram bot interactions include sending texts and photos, with dynamic chat ID extraction from incoming messages.               | Telegram Trigger and Telegram Tool nodes.                                                                      |
| For detailed interior design terminology and standards, the AI models are prompted to maintain professional tone and domain accuracy. | Embedded in system messages for Langchain agents.                                                             |
| Workflow handles ambiguous user inputs by requesting clarifications and sending images for selection where applicable.             | Described in AI Agent prompt and Telegram photo message nodes.                                                |
| Workflow is designed for n8n version supporting Langchain agent and agentTool nodes (v2+), and requires configured credentials.    | Technical requirement for node versions and credentials.                                                      |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.