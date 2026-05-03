🉑 Generate Anki Flash Cards for Language Learning with Google Translate and GPT-4o

https://n8nworkflows.xyz/workflows/---generate-anki-flash-cards-for-language-learning-with-google-translate-and-gpt-4o-3195


# 🉑 Generate Anki Flash Cards for Language Learning with Google Translate and GPT-4o

### 1. Workflow Overview

This workflow automates the creation of language learning flashcards, specifically designed for Mandarin but adaptable to any language. It listens for new vocabulary entries in a Google Sheet, translates the word using Google Translate, generates phonetic transcription (pinyin) and an example sentence via an AI agent powered by GPT-4o, retrieves a relevant illustrative image from the Pexels free image database, uploads the image to Google Drive, and finally updates the Google Sheet with all enriched data. This enables seamless import into Anki flashcard software, streamlining language study preparation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Trigger on new row addition in Google Sheet and verify input.
- **1.2 Translation Processing:** Translate the input word into the target language (Simplified Chinese).
- **1.3 AI Processing:** Generate phonetic transcription and example sentence using GPT-4o.
- **1.4 Image Retrieval and Upload:** Search Pexels for an image, download it, and upload to Google Drive.
- **1.5 Data Aggregation and Sheet Update:** Combine all generated data and update the original Google Sheet row.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
This block triggers the workflow when a new word is added to the Google Sheet and ensures the input is not empty before proceeding.

**Nodes Involved:**  
- Trigger Added Row  
- initialText is empty?  
- No Operation, do nothing  
- Take initialText

**Node Details:**

- **Trigger Added Row**  
  - Type: Google Sheets Trigger  
  - Role: Watches for new rows added to a specified Google Sheet tab.  
  - Configuration: Monitors a specific Google Sheet and tab (user must provide Sheet ID and Tab ID). Polls every minute.  
  - Inputs: None (trigger node)  
  - Outputs: Emits new row data including the initial English word under `initialText`.  
  - Edge Cases: Failure if Google Sheets API credentials are invalid or quota exceeded.  
  - Notes: Sticky Note1 provides setup instructions and links.

- **initialText is empty?**  
  - Type: If Node  
  - Role: Checks if the `initialText` field is not empty to avoid processing blank rows.  
  - Configuration: Condition tests if `initialText` is not empty (strict, case sensitive).  
  - Inputs: From Trigger Added Row  
  - Outputs:  
    - True branch: proceeds if `initialText` is present  
    - False branch: routes to No Operation node  
  - Edge Cases: Expression failure if `initialText` is missing or malformed.

- **No Operation, do nothing**  
  - Type: NoOp Node  
  - Role: Ends the workflow gracefully if input is empty.  
  - Inputs: From False branch of If node  
  - Outputs: None  
  - Edge Cases: None

- **Take initialText**  
  - Type: Set Node  
  - Role: Extracts and assigns the `initialText` value to a variable `entry` for downstream use.  
  - Configuration: Sets `entry` = `initialText` from the trigger data.  
  - Inputs: From True branch of If node  
  - Outputs: Passes `entry` forward.  
  - Edge Cases: None

---

#### 1.2 Translation Processing

**Overview:**  
This block translates the input English word into Simplified Chinese using Google Translate API.

**Nodes Involved:**  
- Google Translate  
- Extract Fields

**Node Details:**

- **Google Translate**  
  - Type: Google Translate Node  
  - Role: Translates the English word to Simplified Chinese (`zh-CN`).  
  - Configuration:  
    - Text input: `initialText` from trigger  
    - Target language: `zh-CN` (Simplified Mandarin)  
  - Inputs: From `initialText is empty?` True branch  
  - Outputs: Provides translated text under `translatedText`.  
  - Edge Cases: API quota limits, invalid credentials, or network timeouts.  
  - Notes: Sticky Note1 explains setup and API credential requirements.

- **Extract Fields**  
  - Type: Set Node  
  - Role: Extracts and prepares key fields for downstream nodes.  
  - Configuration:  
    - `initialText` from trigger  
    - `translatedText` from Google Translate output  
    - `image_name` constructed as `<initialText>.jpeg` for image naming  
  - Inputs: From Google Translate  
  - Outputs: Passes these fields forward.  
  - Edge Cases: Missing translation or malformed data.

---

#### 1.3 AI Processing

**Overview:**  
This block uses an AI agent (GPT-4o) to generate the phonetic transcription (pinyin) and a concise example sentence illustrating the translated word.

**Nodes Involved:**  
- OpenAI Chat Model  
- AI Agent  
- Structured Output Parser  
- Extract Pinyin and Example  
- Merge

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model Node  
  - Role: Provides GPT-4o-mini model for AI processing.  
  - Configuration: Model set to `gpt-4o-mini`.  
  - Inputs: Connected as AI language model for AI Agent node.  
  - Outputs: AI-generated response.  
  - Edge Cases: API key issues, rate limits, or model unavailability.

- **AI Agent**  
  - Type: Langchain Agent Node  
  - Role: Processes translated text to extract pinyin and generate example sentence.  
  - Configuration:  
    - Input text: `translatedText` from Google Translate  
    - System prompt instructs to output JSON with keys `pinyin` and `sentence`  
    - Output parser enabled for structured JSON extraction  
  - Inputs: From Google Translate and OpenAI Chat Model  
  - Outputs: JSON with `pinyin` and `sentence`.  
  - Edge Cases: Parsing errors if AI output is malformed or incomplete.  
  - Notes: Sticky Note2 details setup and prompt design.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser Node  
  - Role: Parses AI output to structured JSON format.  
  - Configuration: Example JSON schema provided for validation.  
  - Inputs: From OpenAI Chat Model (AI output)  
  - Outputs: Parsed JSON for downstream use.  
  - Edge Cases: Parsing failure if AI output deviates from schema.

- **Extract Pinyin and Example**  
  - Type: Set Node  
  - Role: Extracts `pinyin` and `sentence` from AI Agent output JSON into separate fields.  
  - Configuration:  
    - `phonetic` = `output.pinyin`  
    - `example` = `output.sentence`  
  - Inputs: From AI Agent  
  - Outputs: Passes extracted fields forward.  
  - Edge Cases: Missing keys in AI output.

- **Merge**  
  - Type: Merge Node  
  - Role: Combines AI output with translation and initial text data.  
  - Configuration: Mode `combineBySql` to merge by SQL-like keys.  
  - Inputs: From Extract Fields and Extract Pinyin and Example  
  - Outputs: Combined data object.  
  - Edge Cases: Merge conflicts or missing data.

---

#### 1.4 Image Retrieval and Upload

**Overview:**  
This block searches for an image related to the input word on Pexels, downloads the first image, uploads it to Google Drive, and extracts the image link.

**Nodes Involved:**  
- Call API Pexels  
- Get Picture  
- Upload Picture  
- Extract Image Link  
- Merge

**Node Details:**

- **Call API Pexels**  
  - Type: HTTP Request Node  
  - Role: Calls Pexels API to search images matching the input word.  
  - Configuration:  
    - URL: `https://api.pexels.com/v1/search`  
    - Query parameter: `query` = `initialText`  
    - Header: `Authorization` with Pexels API key (user must provide)  
  - Inputs: From Take initialText node  
  - Outputs: JSON response with photos array.  
  - Edge Cases: API key invalid, quota exceeded, no images found.  
  - Notes: Sticky Note3 explains API key setup and usage.

- **Get Picture**  
  - Type: HTTP Request Node  
  - Role: Downloads the first image from Pexels search results.  
  - Configuration: URL set to first photo’s medium-sized image URL (`photos[0].src.medium`).  
  - Inputs: From Call API Pexels  
  - Outputs: Binary image data.  
  - Edge Cases: No photos in response, network errors.

- **Upload Picture**  
  - Type: Google Drive Node  
  - Role: Uploads the downloaded image to Google Drive.  
  - Configuration:  
    - File name: `<initialText>.jpeg`  
    - Drive: User’s selected Google Drive (e.g., "My Drive")  
    - Folder: User-selected folder ID or URL (must be configured)  
  - Inputs: From Get Picture (binary data)  
  - Outputs: Metadata including `webContentLink` for the uploaded image.  
  - Edge Cases: Google Drive API errors, permission issues, quota limits.

- **Extract Image Link**  
  - Type: Set Node  
  - Role: Extracts the public web content link of the uploaded image for use in the sheet.  
  - Configuration: Sets `image_link` = `webContentLink` from Upload Picture output.  
  - Inputs: From Upload Picture  
  - Outputs: Passes `image_link` forward.  
  - Edge Cases: Missing or inaccessible link.

- **Merge** (second input)  
  - Type: Merge Node  
  - Role: Combines image link data with previously merged translation and AI data.  
  - Configuration: Mode `combineByPosition` to combine data arrays by position.  
  - Inputs: From Extract Image Link and previous Merge node  
  - Outputs: Fully combined data object for final sheet update.  
  - Edge Cases: Data alignment issues.

---

#### 1.5 Data Aggregation and Sheet Update

**Overview:**  
This block updates the original Google Sheet row with all generated data: initial text, translation, phonetic transcription, example sentence, image name, and image link.

**Nodes Involved:**  
- Final Merge  
- Add Results in Sheet

**Node Details:**

- **Final Merge**  
  - Type: Merge Node  
  - Role: Combines all data streams (translation, AI output, image link) into one unified dataset.  
  - Configuration: Mode `combine` by position.  
  - Inputs: From Merge nodes of AI and image data.  
  - Outputs: Complete data object for sheet update.  
  - Edge Cases: Data mismatch or missing fields.

- **Add Results in Sheet**  
  - Type: Google Sheets Node  
  - Role: Updates the Google Sheet row with enriched flashcard data.  
  - Configuration:  
    - Operation: Update row  
    - Columns mapped:  
      - `initialText`  
      - `translatedText`  
      - `phonetic` (pinyin)  
      - `sentence` (example)  
      - `image_name` (filename)  
      - `image_link` (Google Drive URL)  
    - Sheet and document IDs must be configured by user.  
  - Inputs: From Final Merge  
  - Outputs: None (final step)  
  - Edge Cases: API errors, permission issues, row mismatch.  
  - Notes: Sticky Note4 explains setup and mapping.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                                  | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                         |
|------------------------|----------------------------------|-------------------------------------------------|------------------------------|------------------------------|---------------------------------------------------------------------------------------------------|
| Trigger Added Row       | Google Sheets Trigger             | Detect new vocabulary word added in Google Sheet| None                         | initialText is empty?, Take initialText | Sticky Note1: Setup Google Sheet trigger and Translate API                                         |
| initialText is empty?   | If Node                         | Validate input is not empty                      | Trigger Added Row             | Google Translate, No Operation |                                                                                                   |
| No Operation, do nothing| NoOp Node                       | End workflow if input empty                       | initialText is empty?         | None                         |                                                                                                   |
| Take initialText        | Set Node                       | Extract initialText for downstream use           | initialText is empty?         | Call API Pexels              |                                                                                                   |
| Google Translate        | Google Translate Node            | Translate English word to Simplified Chinese     | initialText is empty?         | Extract Fields, AI Agent      | Sticky Note1: Setup Google Translate API                                                          |
| Extract Fields          | Set Node                       | Prepare fields for AI and image retrieval         | Google Translate              | Merge                        |                                                                                                   |
| OpenAI Chat Model       | Langchain OpenAI Chat Model      | Provide GPT-4o model for AI agent                 | AI Agent (ai_languageModel)   | AI Agent                     |                                                                                                   |
| AI Agent               | Langchain Agent Node             | Generate pinyin and example sentence              | Google Translate, OpenAI Chat Model | Extract Pinyin and Example | Sticky Note2: Setup AI Agent with prompt for phonetic transcription and example sentence          |
| Structured Output Parser| Langchain Structured Output Parser| Parse AI output JSON                               | OpenAI Chat Model             | AI Agent                     |                                                                                                   |
| Extract Pinyin and Example| Set Node                     | Extract pinyin and example sentence from AI output| AI Agent                     | Merge                        |                                                                                                   |
| Merge                  | Merge Node                      | Combine translation and AI data                    | Extract Fields, Extract Pinyin and Example | Final Merge               |                                                                                                   |
| Call API Pexels         | HTTP Request Node               | Search for image on Pexels                         | Take initialText              | Get Picture                  | Sticky Note3: Setup Pexels API key and usage                                                     |
| Get Picture             | HTTP Request Node               | Download first image from Pexels                   | Call API Pexels               | Upload Picture               |                                                                                                   |
| Upload Picture          | Google Drive Node               | Upload image to Google Drive                        | Get Picture                  | Extract Image Link           | Sticky Note3: Setup Google Drive upload node                                                     |
| Extract Image Link      | Set Node                       | Extract Google Drive image link                     | Upload Picture               | Final Merge                  |                                                                                                   |
| Final Merge             | Merge Node                      | Combine all data streams (translation, AI, image) | Merge, Extract Image Link     | Add Results in Sheet         |                                                                                                   |
| Add Results in Sheet    | Google Sheets Node              | Update Google Sheet row with all flashcard data   | Final Merge                  | None                        | Sticky Note4: Setup Google Sheets update node                                                    |
| Sticky Note1            | Sticky Note                    | Instructions for Google Sheet trigger and translation setup | None                         | None                         |                                                                                                   |
| Sticky Note2            | Sticky Note                    | Instructions for AI Agent setup                    | None                         | None                         |                                                                                                   |
| Sticky Note3            | Sticky Note                    | Instructions for Pexels API and Google Drive upload| None                         | None                         |                                                                                                   |
| Sticky Note4            | Sticky Note                    | Instructions for Google Sheets update              | None                         | None                         |                                                                                                   |
| Sticky Note3            | Sticky Note                    | Step-by-step tutorial link and image               | None                         | None                         | [🎥 Watch My Tutorial](https://youtu.be/2mRZJATUTDw)                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure credentials for Google Sheets API.  
   - Select your Google Sheet document and the specific tab where vocabulary is entered.  
   - Set event to "Row Added".  
   - Poll interval: every minute.

2. **Add If Node to Check Empty Input**  
   - Type: If  
   - Condition: Check if `initialText` (from trigger data) is not empty (string not empty).  
   - True branch: proceed; False branch: connect to No Operation node.

3. **Add No Operation Node**  
   - Type: NoOp  
   - Connect False branch of If node here to end workflow gracefully if input is empty.

4. **Add Set Node to Extract initialText**  
   - Type: Set  
   - Assign variable `entry` = `initialText` from trigger data.  
   - Connect True branch of If node here.

5. **Add HTTP Request Node to Call Pexels API**  
   - Type: HTTP Request  
   - URL: `https://api.pexels.com/v1/search`  
   - Query parameter: `query` = `entry` (word to translate)  
   - Header: Add `Authorization` with your Pexels API key.  
   - Method: GET.

6. **Add HTTP Request Node to Download Image**  
   - Type: HTTP Request  
   - URL: Set dynamically to first photo's medium image URL from Pexels response (`photos[0].src.medium`).  
   - Method: GET.  
   - Expect binary data output.

7. **Add Google Drive Node to Upload Image**  
   - Type: Google Drive  
   - Credentials: Google Drive API OAuth2 credentials.  
   - File name: `{{entry}}.jpeg`  
   - Select Drive (e.g., "My Drive") and target folder ID or URL for image storage.  
   - Input: binary data from previous node.

8. **Add Set Node to Extract Image Link**  
   - Type: Set  
   - Assign `image_link` = `webContentLink` from Google Drive upload output.

9. **Add Google Translate Node**  
   - Type: Google Translate  
   - Credentials: Google Translate API OAuth2 credentials.  
   - Text: `initialText` from trigger.  
   - Target language: `zh-CN` (Simplified Chinese).

10. **Add Set Node to Extract Translation Fields**  
    - Type: Set  
    - Assign:  
      - `initialText` = from trigger  
      - `translatedText` = from Google Translate output  
      - `image_name` = `{{entry}}.jpeg`

11. **Add OpenAI Chat Model Node**  
    - Type: Langchain OpenAI Chat Model  
    - Credentials: OpenAI API key with access to GPT-4o-mini.  
    - Model: `gpt-4o-mini`.

12. **Add AI Agent Node**  
    - Type: Langchain Agent  
    - Input text: `translatedText` from Google Translate.  
    - System prompt: instruct to generate JSON with `pinyin` and `sentence` keys (see prompt in original workflow).  
    - Connect OpenAI Chat Model as AI language model.  
    - Enable structured output parser.

13. **Add Structured Output Parser Node**  
    - Type: Langchain Structured Output Parser  
    - Provide example JSON schema for validation:  
      ```json
      {
        "pinyin": "Cāngkù",
        "sentence": "货物存放在仓库里。"
      }
      ```

14. **Add Set Node to Extract Pinyin and Example**  
    - Type: Set  
    - Assign:  
      - `phonetic` = `output.pinyin`  
      - `example` = `output.sentence`

15. **Add Merge Node to Combine Translation and AI Data**  
    - Type: Merge  
    - Mode: `combineBySql`  
    - Inputs: From Extract Fields and Extract Pinyin and Example nodes.

16. **Add Merge Node to Combine Image Link Data**  
    - Type: Merge  
    - Mode: `combineByPosition`  
    - Inputs: From previous Merge node and Extract Image Link node.

17. **Add Final Merge Node**  
    - Type: Merge  
    - Mode: `combine` (combine by position)  
    - Inputs: From the two Merge nodes above.

18. **Add Google Sheets Node to Update Sheet**  
    - Type: Google Sheets  
    - Credentials: Google Sheets API OAuth2 credentials.  
    - Operation: Update row.  
    - Map columns:  
      - `initialText`  
      - `translatedText`  
      - `phonetic`  
      - `sentence`  
      - `image_name`  
      - `image_link`  
    - Select the same Google Sheet and tab as trigger.

19. **Connect all nodes according to the flow:**  
    - Trigger → If → (True) → Google Translate → Extract Fields → Merge → Final Merge → Add Results in Sheet  
    - Trigger → If → (True) → Take initialText → Call API Pexels → Get Picture → Upload Picture → Extract Image Link → Final Merge  
    - Google Translate → AI Agent (with OpenAI Chat Model and Structured Output Parser) → Extract Pinyin and Example → Merge

20. **Test the workflow** by adding a new English word in the Google Sheet and verify all fields populate correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow designed by Samir, a Supply Chain Data Scientist with experience learning Mandarin in China.                            | Context and motivation for workflow creation.                                                            |
| Workflow supports any language by adjusting AI prompt from "pinyin" to "phonetic transcription".                                 | Adaptable for other languages beyond Mandarin.                                                           |
| Requires Google Drive, Google Sheets, Google Translate API credentials, and Pexels API key.                                       | Prerequisites for operation.                                                                              |
| Tutorial video available for detailed step-by-step setup and usage.                                                               | [🎥 Watch My Tutorial](https://youtu.be/2mRZJATUTDw)                                                     |
| Blog article on using Anki flashcards for Mandarin learning automation.                                                           | [Blog Article about Anki Flash Cards](https://www.samirsaci.com/automate-flash-cards-creation-for-language-learning-with-python/) |
| Sticky notes in workflow provide detailed setup instructions and links for each major block.                                      | Embedded in workflow nodes for user guidance.                                                            |

---

This documentation provides a comprehensive, structured reference to understand, reproduce, and modify the "Generate Anki Flash Cards for Language Learning with Google Translate and GPT-4o" workflow. It anticipates potential failure points such as API credential issues, empty inputs, and parsing errors, and includes all necessary configuration details for seamless integration.