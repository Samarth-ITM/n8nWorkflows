Extract vocabulary flashcards from Spotify listening with Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/extract-vocabulary-flashcards-from-spotify-listening-with-gemini-and-google-sheets-14022


# Extract vocabulary flashcards from Spotify listening with Gemini and Google Sheets

# 1. Workflow Overview

This workflow extracts useful vocabulary flashcards from your recent Spotify listening history. It pulls recently played tracks, fetches lyrics from lrclib.net, combines and cleans those lyrics, asks Gemini to extract useful vocabulary with translations, removes words already learned, and writes the new flashcards into Google Sheets.

Typical use case:
- A language learner listens to songs on Spotify
- Once per week, the workflow collects recent songs
- AI extracts 40–60 useful words from the lyrics
- New words are stored in a master vocabulary sheet and a week-specific sheet

Logical blocks:

## 1.1 Scheduled Input and Spotify Song Collection
The workflow starts from a weekly schedule trigger, then retrieves the user’s recently played songs from Spotify.

## 1.2 Song Deduplication and Lyrics Retrieval
The workflow normalizes recent track data, removes duplicate songs, and calls lrclib.net to search for lyrics.

## 1.3 Lyrics Consolidation and Existing Vocabulary Fetch
Fetched lyrics are merged into one combined text corpus, while all previously stored vocabulary is read from Google Sheets.

## 1.4 AI Vocabulary Extraction
Gemini receives the consolidated lyrics and extracts a structured JSON list of vocabulary words and translations.

## 1.5 Deduplication, Weekly Sheet Naming, and Parsing
The AI output is parsed, cleaned, compared against previously learned words, and assigned to the current week sheet name.

## 1.6 Google Sheets Output
The workflow creates a weekly sheet if needed, appends new words to that weekly sheet, and also appends them to the master “All Vocabularies” sheet.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input and Spotify Song Collection

**Overview:**  
This block defines the workflow entry point and pulls the latest Spotify listening activity. It is the source block for all downstream processing.

**Nodes Involved:**
- Every Sunday at noon
- Get the recently played Songs

### Node: Every Sunday at noon
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow trigger node
- **Configuration choices:** Uses a cron expression `0 12 * * 7`, meaning every Sunday at 12:00
- **Key expressions or variables used:** None
- **Input and output connections:** No input; outputs to **Get the recently played Songs**
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - Node is currently **disabled**, so the workflow will not run automatically until enabled
  - Server timezone affects the actual execution time
- **Sub-workflow reference:** None

### Node: Get the recently played Songs
- **Type and technical role:** `n8n-nodes-base.spotify`; retrieves user listening history
- **Configuration choices:**
  - Operation: `recentlyPlayed`
  - Limit: `20`
- **Key expressions or variables used:** None
- **Input and output connections:** Input from **Every Sunday at noon**; output to **Cut out Duplicate Songs**
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Spotify OAuth credential failure
  - Missing permission scope for recently played tracks
  - Empty history if the account has little or no recent listening activity
  - Rate limiting or temporary Spotify API issues
- **Sub-workflow reference:** None

---

## 2.2 Song Deduplication and Lyrics Retrieval

**Overview:**  
This block transforms Spotify items into simplified song records and removes duplicates based on Spotify URL. It then queries lrclib.net for matching lyrics.

**Nodes Involved:**
- Cut out Duplicate Songs
- Getting Lyrics

### Node: Cut out Duplicate Songs
- **Type and technical role:** `n8n-nodes-base.code`; transforms and deduplicates song records
- **Configuration choices:**
  - Extracts:
    - `trackName` from `item.json.track.name`
    - `artistName` from first artist
    - `spotifyUrl` from `track.external_urls.spotify`
  - Uses a JavaScript `Set` to keep only one item per Spotify URL
- **Key expressions or variables used:**
  - `item.json.track`
  - `track.external_urls?.spotify || ''`
- **Input and output connections:** Input from **Get the recently played Songs**; output to **Getting Lyrics**
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Missing `track` object or empty artist array can break mapping
  - If `spotifyUrl` is empty for multiple items, they may be treated as duplicates unexpectedly
- **Sub-workflow reference:** None

### Node: Getting Lyrics
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls lrclib.net search API
- **Configuration choices:**
  - URL expression:
    `https://lrclib.net/api/search?track_name={{ $json.trackName }}&artist_name={{ $json.artistName }}`
  - `retryOnFail: true`
  - `onError: continueRegularOutput`
- **Key expressions or variables used:**
  - `$json.trackName`
  - `$json.artistName`
- **Input and output connections:** Input from **Cut out Duplicate Songs**; output to **Put all Lyrics together**
- **Version-specific requirements:** Type version `4.4`
- **Edge cases or potential failure types:**
  - lrclib.net may return no match
  - HTTP/network failures
  - Query encoding issues with special characters in song or artist names
  - Because `continueRegularOutput` is enabled, downstream nodes must tolerate incomplete/empty lyric responses
- **Sub-workflow reference:** None

---

## 2.3 Lyrics Consolidation and Existing Vocabulary Fetch

**Overview:**  
This block merges lyric results into one text payload and fetches the master list of previously learned words from Google Sheets. It prepares the data context required for AI extraction and duplicate filtering.

**Nodes Involved:**
- Put all Lyrics together
- Read all Vocabularies
- Combine existing Words with new Words

### Node: Put all Lyrics together
- **Type and technical role:** `n8n-nodes-base.code`; consolidates lyric search results into a single combined text
- **Configuration choices:**
  - Normalizes song titles by removing bracketed text and quote-like fragments
  - Normalizes artist names by removing `- Topic`
  - Builds a deduplication key as `cleanArtist + ' - ' + cleanTrack`
  - Keeps the version with the longest `plainLyrics`
  - Joins all songs into one field `allLyrics`
  - Outputs:
    - `allLyrics`
    - `songCount`
- **Key expressions or variables used:**
  - `data.trackName`
  - `data.artistName`
  - `data.plainLyrics`
- **Input and output connections:** Input from **Getting Lyrics**; output to **Read all Vocabularies**
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - lrclib search responses may not contain `plainLyrics` in the expected shape
  - If all lyric lookups fail, `allLyrics` may be empty and `songCount` may be `0`
  - Over-cleaning titles may occasionally merge distinct tracks
- **Sub-workflow reference:** None

### Node: Read all Vocabularies
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads the master vocabulary sheet
- **Configuration choices:**
  - Reads document `YOUR_GOOGLE_SHEET_ID`
  - Sheet: `gid=0`, named “All Vocabularies”
  - `alwaysOutputData: true` ensures downstream flow continues even when no rows are returned
- **Key expressions or variables used:** None
- **Input and output connections:** Input from **Put all Lyrics together**; output to **Combine existing Words with new Words**
- **Version-specific requirements:** Type version `4.7`
- **Edge cases or potential failure types:**
  - Google Sheets OAuth permission issues
  - Wrong sheet ID or missing tab
  - Empty sheet on first run
- **Sub-workflow reference:** None

### Node: Combine existing Words with new Words
- **Type and technical role:** `n8n-nodes-base.code`; packages existing vocabulary together with the lyric corpus
- **Configuration choices:**
  - Reads `allLyrics` and `songCount` from **Put all Lyrics together**
  - Collects all `Word` column values from incoming Google Sheets items
  - Returns a single item with:
    - `allLyrics`
    - `songCount`
    - `existingWords`
- **Key expressions or variables used:**
  - `$('Put all Lyrics together').first().json`
  - `item.json.Word`
- **Input and output connections:** Input from **Read all Vocabularies**; output to **AI Agent**
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If sheet rows do not contain a `Word` column, deduplication quality degrades
  - If `Put all Lyrics together` returns no item, the cross-node reference fails
- **Sub-workflow reference:** None

---

## 2.4 AI Vocabulary Extraction

**Overview:**  
This block sends the combined lyrics to a LangChain agent backed by Gemini. The model is instructed to extract a deduplicated JSON array of useful vocabulary words and German translations.

**Nodes Involved:**
- Google Gemini Chat Model
- AI Agent

### Node: Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; language model provider node
- **Configuration choices:** Uses default options with a Gemini chat model credential setup
- **Key expressions or variables used:** None
- **Input and output connections:** AI language model connection into **AI Agent**
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Missing or invalid Gemini API credentials
  - Model quota/rate-limit issues
  - Regional availability restrictions
- **Sub-workflow reference:** None

### Node: AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates prompt execution with the Gemini model
- **Configuration choices:**
  - Prompt text:
    - `All Lyrics:\n{{ $json.allLyrics }}`
  - Prompt type: defined manually
  - System message instructs the model to:
    - extract 40–60 useful English words across all songs
    - avoid duplicates
    - skip proper nouns, names, filler words, slang abbreviations, and very basic words
    - normalize slang to standard English if needed
    - translate to German
    - focus on B1–B2 everyday vocabulary
    - return only a valid JSON array
- **Key expressions or variables used:**
  - `$json.allLyrics`
- **Input and output connections:** Main input from **Combine existing Words with new Words**; main output to **Parse, Deduplicate, Week Code**; AI model input from **Google Gemini Chat Model**
- **Version-specific requirements:** Type version `3.1`
- **Edge cases or potential failure types:**
  - Model may still return invalid JSON or extra markdown
  - Empty lyrics can produce weak or empty output
  - Token/context limits if aggregated lyrics are very large
  - Translation language is hardcoded to German in the system prompt
- **Sub-workflow reference:** None

---

## 2.5 Deduplication, Weekly Sheet Naming, and Parsing

**Overview:**  
This block parses the AI response, removes markdown wrappers if present, strips parenthetical notes from translations, filters out already known words, and calculates the current week sheet name.

**Nodes Involved:**
- Parse, Deduplicate, Week Code

### Node: Parse, Deduplicate, Week Code
- **Type and technical role:** `n8n-nodes-base.code`; post-processes AI output and prepares final rows for storage
- **Configuration choices:**
  - Reads `item.json.output`
  - Removes code fences such as ```json
  - Parses the cleaned string as JSON
  - Maps each entry to:
    - `Word`
    - `Translation`
  - Removes trailing parenthetical notes from translations using regex
  - Retrieves `existingWords` from **Combine existing Words with new Words**
  - Performs case-insensitive deduplication against existing words
  - Computes week number from current date
  - Sets `sheetName` like `Week 11`
- **Key expressions or variables used:**
  - `item.json.output`
  - `$('Combine existing Words with new Words').first().json.existingWords`
- **Input and output connections:** Input from **AI Agent**; outputs to:
  - **Create sheet**
  - **Cut out SheetName Column**
  - **Append to All Vocabularies**
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Invalid AI JSON causes parsing failure; node logs errors and skips invalid item output
  - If a parsed word lacks expected fields, output rows may be malformed
  - Week number logic is simple calendar math, not ISO week numbering
  - Duplicate words within the AI output itself are not explicitly deduplicated unless already in Sheets
- **Sub-workflow reference:** None

---

## 2.6 Google Sheets Output

**Overview:**  
This block writes the final vocabulary into Google Sheets. It tries to create the weekly sheet, appends entries to that weekly sheet, and also appends them to the master sheet.

**Nodes Involved:**
- Create sheet
- Cut out SheetName Column
- Append to Weekly
- Append to All Vocabularies

### Node: Create sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; creates a new sheet/tab in the target spreadsheet
- **Configuration choices:**
  - Operation: `create`
  - Title: `={{ $json.sheetName }}`
  - Document: `YOUR_GOOGLE_SHEET_ID`
  - `onError: continueRegularOutput`
- **Key expressions or variables used:**
  - `$json.sheetName`
- **Input and output connections:** Input from **Parse, Deduplicate, Week Code**; no downstream connection
- **Version-specific requirements:** Type version `4.7`
- **Edge cases or potential failure types:**
  - If the weekly tab already exists, create may fail, but the workflow continues because errors are tolerated
  - Insufficient Google Drive/Sheets permissions
- **Sub-workflow reference:** None

### Node: Cut out SheetName Column
- **Type and technical role:** `n8n-nodes-base.code`; removes helper field before writing to weekly sheet
- **Configuration choices:**
  - Outputs only:
    - `Word`
    - `Translation`
- **Key expressions or variables used:**
  - `item.json.Word`
  - `item.json.Translation`
- **Input and output connections:** Input from **Parse, Deduplicate, Week Code**; output to **Append to Weekly**
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If Word or Translation is missing, blank rows may be appended
- **Sub-workflow reference:** None

### Node: Append to Weekly
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends rows to the current weekly sheet
- **Configuration choices:**
  - Operation: `append`
  - Sheet name is dynamic:
    `={{ $('Parse, Deduplicate, Week Code').first().json.sheetName }}`
  - Column mapping:
    - `Word = {{ $json.Word }}`
    - `Translation = {{ $json.Translation }}`
- **Key expressions or variables used:**
  - `$('Parse, Deduplicate, Week Code').first().json.sheetName`
  - `$json.Word`
  - `$json.Translation`
- **Input and output connections:** Input from **Cut out SheetName Column**
- **Version-specific requirements:** Type version `4.7`
- **Edge cases or potential failure types:**
  - Fails if the sheet was not created and does not already exist
  - Dynamic expression can fail if no parsed items are produced
  - Credential or spreadsheet access problems
- **Sub-workflow reference:** None

### Node: Append to All Vocabularies
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends rows to the master sheet
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `gid=0`, “All Vocabularies”
  - Column mapping:
    - `Word = {{ $json.Word }}`
    - `Translation = {{ $json.Translation }}`
- **Key expressions or variables used:**
  - `$json.Word`
  - `$json.Translation`
- **Input and output connections:** Input from **Parse, Deduplicate, Week Code**
- **Version-specific requirements:** Type version `4.7`
- **Edge cases or potential failure types:**
  - Wrong sheet mapping or missing headers
  - Appending duplicate entries if upstream logic fails to deduplicate
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Sunday at noon | Schedule Trigger | Weekly workflow entry point |  | Get the recently played Songs | ## How it works Every Sunday it fetches your recently played songs, finds the lyrics via lrclib.net, and uses Google Gemini to extract 40-60 useful vocabulary words (B1-B2 level). New words are deduplicated against all previously learned words and written to Google Sheets — both a master tab and a weekly tab. You review the flashcards using the free Flashcard Lab app (iOS + Android), which reads directly from Google Sheets with built-in spaced repetition. |
| Get the recently played Songs | Spotify | Fetch recent Spotify tracks | Every Sunday at noon | Cut out Duplicate Songs | ### 🎵 Fetches recently played songs from Spotify, removes duplicates, and gets lyrics from lrclib.net. |
| Cut out Duplicate Songs | Code | Simplify and deduplicate track list | Get the recently played Songs | Getting Lyrics | ### 🎵 Fetches recently played songs from Spotify, removes duplicates, and gets lyrics from lrclib.net. |
| Getting Lyrics | HTTP Request | Search lrclib.net for lyrics | Cut out Duplicate Songs | Put all Lyrics together | ### 🎵 Fetches recently played songs from Spotify, removes duplicates, and gets lyrics from lrclib.net. |
| Put all Lyrics together | Code | Merge unique lyric text into one corpus | Getting Lyrics | Read all Vocabularies | ### 📖 Reads all previously learned words from Google Sheets for duplicate checking. |
| Read all Vocabularies | Google Sheets | Read master vocabulary sheet | Put all Lyrics together | Combine existing Words with new Words | ### 📖 Reads all previously learned words from Google Sheets for duplicate checking. |
| Combine existing Words with new Words | Code | Package combined lyrics with known words | Read all Vocabularies | AI Agent | ### 📖 Reads all previously learned words from Google Sheets for duplicate checking. |
| Google Gemini Chat Model | LangChain Google Gemini Chat Model | Provide Gemini LLM to agent |  | AI Agent | ## 🤖 AI EXTRACTION ➡️ To change the language: edit the prompt. |
| AI Agent | LangChain Agent | Extract vocabulary and translations from lyrics | Combine existing Words with new Words; Google Gemini Chat Model | Parse, Deduplicate, Week Code | ## 🤖 AI EXTRACTION ➡️ To change the language: edit the prompt. |
| Parse, Deduplicate, Week Code | Code | Parse AI JSON, deduplicate, assign weekly tab name | AI Agent | Create sheet; Cut out SheetName Column; Append to All Vocabularies |  |
| Create sheet | Google Sheets | Create weekly tab if needed | Parse, Deduplicate, Week Code |  | ## 📝 SAVE TO SHEETS Creates a weekly tab (e.g. "Week 11") and writes new words there + to the master "All Vocabularies" tab. Use Flashcard Lab app to study from the weekly tabs. |
| Cut out SheetName Column | Code | Remove helper sheet name field before weekly append | Parse, Deduplicate, Week Code | Append to Weekly | ## 📝 SAVE TO SHEETS Creates a weekly tab (e.g. "Week 11") and writes new words there + to the master "All Vocabularies" tab. Use Flashcard Lab app to study from the weekly tabs. |
| Append to Weekly | Google Sheets | Append rows to the week-specific sheet | Cut out SheetName Column |  | ## 📝 SAVE TO SHEETS Creates a weekly tab (e.g. "Week 11") and writes new words there + to the master "All Vocabularies" tab. Use Flashcard Lab app to study from the weekly tabs. |
| Append to All Vocabularies | Google Sheets | Append rows to master vocabulary sheet | Parse, Deduplicate, Week Code |  | ## 📝 SAVE TO SHEETS Creates a weekly tab (e.g. "Week 11") and writes new words there + to the master "All Vocabularies" tab. Use Flashcard Lab app to study from the weekly tabs. |
| Sticky Note | Sticky Note | Visual documentation |  |  |  |
| Sticky Note1 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note2 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note4 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note5 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note6 | Sticky Note | Visual documentation |  |  | ⚠️ Set your Google Cloud app to "In Production" — in "Testing" mode, tokens expire after 7 days and break the weekly automation. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the trigger node**
   - Add a **Schedule Trigger** node.
   - Name it **Every Sunday at noon**.
   - Configure cron mode with expression: `0 12 * * 7`.
   - If you want to test manually first, keep it disabled until setup is complete.

2. **Add the Spotify node**
   - Add a **Spotify** node.
   - Name it **Get the recently played Songs**.
   - Operation: **Recently Played**
   - Limit: **20**
   - Create and attach Spotify OAuth credentials from the Spotify Developer Dashboard.
   - Connect **Every Sunday at noon → Get the recently played Songs**.

3. **Add a Code node for initial deduplication**
   - Add a **Code** node named **Cut out Duplicate Songs**.
   - Paste logic that:
     - reads `item.json.track`
     - extracts `trackName`, `artistName`, `spotifyUrl`
     - deduplicates rows by `spotifyUrl`
   - Connect **Get the recently played Songs → Cut out Duplicate Songs**.

4. **Add the lyrics lookup node**
   - Add an **HTTP Request** node named **Getting Lyrics**.
   - Method: GET
   - URL:
     `https://lrclib.net/api/search?track_name={{ $json.trackName }}&artist_name={{ $json.artistName }}`
   - Enable retry on failure.
   - Set error handling to continue on error.
   - Connect **Cut out Duplicate Songs → Getting Lyrics**.

5. **Add a Code node to merge lyric content**
   - Add a **Code** node named **Put all Lyrics together**.
   - Implement logic that:
     - normalizes track and artist names
     - builds a unique song key
     - keeps the lyric version with the longest `plainLyrics`
     - combines all lyrics into one text block
     - returns one item with `allLyrics` and `songCount`
   - Connect **Getting Lyrics → Put all Lyrics together**.

6. **Prepare Google Sheets**
   - Create a spreadsheet manually in Google Sheets.
   - Add a tab named **All Vocabularies**.
   - Add headers in row 1:
     - `Word`
     - `Translation`
   - Note the spreadsheet ID from the URL.

7. **Add the master vocabulary reader**
   - Add a **Google Sheets** node named **Read all Vocabularies**.
   - Select your spreadsheet.
   - Set the sheet to **All Vocabularies**.
   - Use the read/get rows behavior with default options.
   - Enable **Always Output Data**.
   - Create and attach Google Sheets OAuth2 credentials.
   - Connect **Put all Lyrics together → Read all Vocabularies**.

8. **Add a Code node to combine lyrics and known words**
   - Add a **Code** node named **Combine existing Words with new Words**.
   - Logic:
     - fetch `allLyrics` and `songCount` from **Put all Lyrics together**
     - collect all `Word` values from the sheet rows
     - return one item containing:
       - `allLyrics`
       - `songCount`
       - `existingWords`
   - Connect **Read all Vocabularies → Combine existing Words with new Words**.

9. **Add the Gemini model node**
   - Add a **Google Gemini Chat Model** node.
   - Name it **Google Gemini Chat Model**.
   - Configure Gemini API credentials using a key from Google AI Studio.
   - Leave model options at default unless you want to tune output.

10. **Add the AI Agent node**
    - Add an **AI Agent** node.
    - Name it **AI Agent**.
    - Set prompt type to manually defined text.
    - User text:
      `All Lyrics:\n{{ $json.allLyrics }}`
    - System message should instruct the model to:
      - extract 40–60 useful English vocabulary words
      - avoid duplicates
      - skip very basic words, names, filler words, slang abbreviations
      - translate into German
      - return only valid JSON array objects with `word` and `translation`
    - Connect:
      - **Combine existing Words with new Words → AI Agent** (main input)
      - **Google Gemini Chat Model → AI Agent** (AI language model input)

11. **Add the parsing/deduplication Code node**
    - Add a **Code** node named **Parse, Deduplicate, Week Code**.
    - Implement logic that:
      - reads `item.json.output`
      - strips code fences like ```json
      - parses JSON
      - maps `word` to `Word`
      - maps `translation` to `Translation`
      - removes trailing parenthetical translation notes
      - loads `existingWords` from **Combine existing Words with new Words**
      - removes case-insensitive duplicates against previously stored words
      - calculates current week name such as `Week 11`
      - outputs items with:
        - `Word`
        - `Translation`
        - `sheetName`
    - Connect **AI Agent → Parse, Deduplicate, Week Code**.

12. **Add the weekly sheet creation node**
    - Add a **Google Sheets** node named **Create sheet**.
    - Operation: **Create sheet**
    - Spreadsheet: your target spreadsheet
    - Title:
      `{{ $json.sheetName }}`
    - Set error handling to continue on error, because the tab may already exist.
    - Connect **Parse, Deduplicate, Week Code → Create sheet**.

13. **Add a Code node to remove helper fields**
    - Add a **Code** node named **Cut out SheetName Column**.
    - Configure it to output only:
      - `Word`
      - `Translation`
    - Connect **Parse, Deduplicate, Week Code → Cut out SheetName Column**.

14. **Add the weekly append node**
    - Add a **Google Sheets** node named **Append to Weekly**.
    - Operation: **Append**
    - Spreadsheet: your target spreadsheet
    - Sheet name expression:
      `{{ $('Parse, Deduplicate, Week Code').first().json.sheetName }}`
    - Map columns:
      - `Word` ← `{{ $json.Word }}`
      - `Translation` ← `{{ $json.Translation }}`
    - Connect **Cut out SheetName Column → Append to Weekly**.

15. **Add the master append node**
    - Add a **Google Sheets** node named **Append to All Vocabularies**.
    - Operation: **Append**
    - Spreadsheet: your target spreadsheet
    - Sheet: **All Vocabularies**
    - Map columns:
      - `Word` ← `{{ $json.Word }}`
      - `Translation` ← `{{ $json.Translation }}`
    - Connect **Parse, Deduplicate, Week Code → Append to All Vocabularies**.

16. **Add optional sticky notes for maintainability**
    - Add notes documenting:
      - Spotify + lyrics fetch section
      - vocabulary read section
      - AI extraction section
      - Google Sheets save section
      - setup guidance and credential reminders

17. **Configure credentials**
    - **Spotify OAuth2**
      - Create app in Spotify Developer Dashboard
      - Add redirect URI from n8n
      - Authorize with the account whose listening history you want
    - **Google Sheets / Google Drive OAuth2**
      - Enable Google Sheets API and Drive API in Google Cloud Console
      - Create OAuth app credentials
      - Set app to **In Production**
      - Authorize in n8n
    - **Gemini**
      - Create API key in Google AI Studio
      - Add to n8n Gemini credential

18. **Test the workflow**
    - Execute manually from the Spotify node or trigger node.
    - Confirm:
      - Spotify returns songs
      - lrclib returns lyric matches
      - combined lyrics are non-empty
      - AI returns valid JSON
      - weekly sheet is created or reused
      - rows are appended to both tabs

19. **Enable automation**
    - Re-enable the **Every Sunday at noon** node if it was disabled.
    - Activate the workflow.

### Sub-workflow setup
This workflow does **not** use any sub-workflow or Execute Workflow node. No sub-workflow configuration is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Every Sunday it fetches your recently played songs, finds the lyrics via lrclib.net, and uses Google Gemini to extract 40–60 useful vocabulary words (B1–B2 level). New words are deduplicated against all previously learned words and written to Google Sheets — both a master tab and a weekly tab. | Workflow purpose |
| You review the flashcards using the free Flashcard Lab app (iOS + Android), which reads directly from Google Sheets with built-in spaced repetition. | External app usage |
| Google Cloud Console: create project, enable Sheets API and Drive API, create OAuth2 credentials, set app to “In Production”. | Google setup |
| Get a free Gemini API key from Google AI Studio. | Gemini setup |
| Spotify Developer Dashboard: create app and note Client ID and Secret. | Spotify setup |
| Create a Google Sheet with tab “All Vocabularies” and headers “Word” + “Translation”. | Spreadsheet prerequisite |
| Import workflow, connect credentials, select your sheet in all Google Sheets nodes, test with Execute Workflow, then enable the schedule trigger. | Deployment guidance |
| To change the language: edit the prompt in the AI Agent node. | Customization |
| ⚠️ Set your Google Cloud app to “In Production” — in “Testing” mode, tokens expire after 7 days and break the weekly automation. | Reliability warning |
| lrclib.net lyrics search API | https://lrclib.net/ |
| Google AI Studio for Gemini API keys | https://aistudio.google.com/ |
| Spotify Developer Dashboard | https://developer.spotify.com/dashboard/ |
| Flashcard Lab app | App referenced in sticky note; no URL provided in workflow |