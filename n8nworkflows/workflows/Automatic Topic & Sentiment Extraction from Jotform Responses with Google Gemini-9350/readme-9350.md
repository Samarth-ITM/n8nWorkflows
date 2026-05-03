Automatic Topic & Sentiment Extraction from Jotform Responses with Google Gemini

https://n8nworkflows.xyz/workflows/automatic-topic---sentiment-extraction-from-jotform-responses-with-google-gemini-9350


# Automatic Topic & Sentiment Extraction from Jotform Responses with Google Gemini

### 1. Workflow Overview

This workflow automates the extraction of topics, keywords, and sentiment analysis from responses submitted via a Jotform survey. It leverages Google Gemini AI models to analyze the textual feedback, providing structured insights including sentiment classification, key phrases, topic summaries, and priority alerts. The processed data is then appended or updated in a Google Sheets spreadsheet for further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Formatting:** Captures new submissions from a specified Jotform form and formats the raw JSON data into a string format suitable for AI processing.
- **1.2 Sentiment Analysis Block:** Uses Google Gemini to perform detailed sentiment analysis on the feedback text, parsing the output into a structured schema.
- **1.3 Topics & Keywords Extraction Block:** Uses Google Gemini to extract high-level topics, keywords, insights, and sentiment associated with topics, similarly parsing the structured output.
- **1.4 Data Aggregation and Storage:** Merges the results from sentiment and topic analysis, aggregates them, and appends or updates the data into a Google Sheets document for record-keeping and further analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Formatting

- **Overview:**  
  This block receives new Jotform submissions via a webhook trigger, then formats the form response JSON into a string to prepare for AI analysis.

- **Nodes Involved:**  
  - JotForm Trigger  
  - Format the Form Data

- **Node Details:**

  - **JotForm Trigger**  
    - Type: Trigger (Webhook)  
    - Role: Listens to new submissions on a specific Jotform form (ID: 252797914459475).  
    - Configuration: Uses Jotform API credentials; configured to trigger on new form submissions.  
    - Inputs: External webhook trigger (Jotform).  
    - Outputs: Raw JSON of form submission.  
    - Potential Failures: Authentication errors with Jotform API; webhook not receiving data if form ID is incorrect or form is inactive.  
    - No sub-workflows.

  - **Format the Form Data**  
    - Type: Code (JavaScript)  
    - Role: Converts the incoming JSON form response into a formatted JSON string under the key `body` for AI consumption.  
    - Configuration: Uses JavaScript to stringify the first input JSON with indentation for clarity.  
    - Key Expression: `const outputString = JSON.stringify($input.first().json, null, 2);`  
    - Inputs: Raw form JSON from JotForm Trigger.  
    - Outputs: JSON with a single field `body` containing formatted string.  
    - Potential Failures: Errors if input data is empty or malformed JSON (unlikely as input is from trigger).  
    - No sub-workflows.

---

#### 1.2 Sentiment Analysis Block

- **Overview:**  
  Analyzes the sentiment of the feedback text using Google Gemini AI, then parses the AI's response into a detailed structured schema including sentiment label, confidence, key phrases, and alerts.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Sentiment Analyzer (Chain LLM)  
  - Structured Output Parser

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: AI Language Model (Google Gemini via LangChain)  
    - Role: Provides AI model interface for sentiment analysis requests.  
    - Configuration: Uses model "models/gemini-2.0-flash-exp" with Google PaLM API credentials.  
    - Inputs: Receives prompt from Sentiment Analyzer node.  
    - Outputs: Generates raw AI response for sentiment analysis.  
    - Potential Issues: API quota limits, authentication failures, latency or timeout errors.  
    - Credentials: Google Palm API account.

  - **Sentiment Analyzer**  
    - Type: Chain LLM (LangChain)  
    - Role: Defines the prompt and manages the AI conversation for sentiment analysis.  
    - Configuration: Prompt instructs to "Perform sentiment analysis of the following {{ $json.body }}" with an expert sentiment analyzer message.  
    - Output parser enabled to parse structured output.  
    - Inputs: Formatted form data string from Format the Form Data node.  
    - Outputs: AI-generated sentiment analysis response.  
    - Potential Issues: Prompt formatting errors, model misinterpretation.  
    - Version: 1.7

  - **Structured Output Parser**  
    - Type: Output Parser (LangChain)  
    - Role: Parses AI output into a JSON object conforming to a manually defined JSON schema capturing customer info, sentiment, confidence score, key phrases, summary, alert priority, and timestamp.  
    - Configuration: JSON schema explicitly defines fields and constraints such as enum values for sentiment and alert priority, number ranges for confidence.  
    - Inputs: AI raw response from Sentiment Analyzer.  
    - Outputs: Structured sentiment analysis JSON for downstream processing.  
    - Potential Issues: Parsing failures if AI output does not conform to schema; schema mismatch; empty or incomplete AI responses.

---

#### 1.3 Topics & Keywords Extraction Block

- **Overview:**  
  Extracts survey topics, keywords, insights, and topic-level sentiment using Google Gemini, parsing the AI response into a JSON schema tailored for survey analytics.

- **Nodes Involved:**  
  - Google Gemini Chat Model for Topics and Keywords  
  - Topics & Keywords (Chain LLM)  
  - Structured Output Parser for Topics & Keywords

- **Node Details:**

  - **Google Gemini Chat Model for Topics and Keywords**  
    - Type: AI Language Model (Google Gemini via LangChain)  
    - Role: Provides AI model interface for topic and keyword extraction.  
    - Configuration: Model "models/gemini-2.0-flash-exp" with Google PaLM API credentials.  
    - Inputs: Prompt from Topics & Keywords node.  
    - Outputs: AI raw response with topic extraction.  
    - Potential Issues: Same as Google Gemini Chat Model above.

  - **Topics & Keywords**  
    - Type: Chain LLM (LangChain)  
    - Role: Defines prompt instructing expert sentiment analysis on the form data; in practice, handles topic extraction prompt.  
    - Configuration: Similar prompt text ("Perform sentiment analysis of the following {{ $json.body }}") and messages, output parser enabled.  
    - Inputs: Formatted form data string from Format the Form Data node.  
    - Outputs: AI response for topic and keyword extraction.  
    - Potential Issues: Prompt ambiguity, AI misunderstanding.

  - **Structured Output Parser for Topics & Keywords**  
    - Type: Output Parser (LangChain)  
    - Role: Parses AI output into a detailed JSON schema covering topics (array of topic objects with topic name, summary, keywords, sentiment, importance), global keywords, insights, and generation timestamp.  
    - Configuration: JSON schema manually defined with strict types and constraints including enums for sentiment and array structures.  
    - Inputs: AI output from Topics & Keywords node.  
    - Outputs: Structured topic and keyword extraction JSON.  
    - Potential Issues: Parsing errors if AI output is malformed or incomplete.

---

#### 1.4 Data Aggregation and Storage

- **Overview:**  
  Merges sentiment and topic extraction outputs, aggregates the combined data, and appends or updates a row in a Google Sheets spreadsheet for persistent storage.

- **Nodes Involved:**  
  - Merge  
  - Aggregate  
  - Append or update row in sheet

- **Node Details:**

  - **Merge**  
    - Type: Merge node  
    - Role: Combines two input streams (sentiment and topic analysis results) into one data stream.  
    - Configuration: Default merge settings (likely "Merge by Index" or "Merge by Append").  
    - Inputs: Two inputs — sentiment analysis output and topics & keywords output.  
    - Outputs: Single merged output combining both datasets.  
    - Potential Issues: Mismatched data arrays, data loss if inputs are not synchronised.

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Aggregates the merged data, specifically aggregating the "output" field to prepare for writing.  
    - Configuration: Aggregates field "output".  
    - Inputs: Merged output from Merge node.  
    - Outputs: Aggregated single JSON object combining the data.  
    - Potential Issues: Aggregation errors if input data is malformed or missing.

  - **Append or update row in sheet**  
    - Type: Google Sheets node  
    - Role: Writes the aggregated data into a Google Sheets document, appending or updating a row based on matching criteria.  
    - Configuration:  
      - Operation: "appendOrUpdate"  
      - Sheet Name: "Sheet1" (gid=0)  
      - Document ID: Spreadsheet with ID "13iJ2sOSaCEekzRNrhkUirZX2llBnpwGi3gdhI4PWIFM"  
      - Columns mapped:  
        - `feedback_analysis`: Entire aggregated JSON as string  
        - `topics_keywords`: Topics and keywords JSON string  
      - Matching column: "feedback_analysis" (used to determine row update)  
      - Credentials: Google Sheets OAuth2.  
    - Inputs: Aggregated JSON from Aggregate node.  
    - Outputs: Confirmation of row append/update.  
    - Potential Issues: API quota, permission errors, schema mismatch causing failed writes.

---

### 3. Summary Table

| Node Name                              | Node Type                             | Functional Role                         | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                          |
|--------------------------------------|-------------------------------------|---------------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------------|
| JotForm Trigger                      | Trigger (Webhook)                   | Receives new Jotform submissions      | —                           | Format the Form Data          |                                                                                                    |
| Format the Form Data                 | Code                               | Formats raw form JSON to string for AI| JotForm Trigger             | Sentiment Analyzer, Topics & Keywords |                                                                                                    |
| Sentiment Analyzer                  | Chain LLM (LangChain)               | Performs sentiment analysis AI prompt | Format the Form Data         | Merge                        | Sticky Note2 covers Sentiment Analysis block                                                        |
| Google Gemini Chat Model            | AI Language Model (Google Gemini)  | Provides AI model interface for sentiment analysis | Sentiment Analyzer (ai_languageModel) | Sentiment Analyzer (ai_languageModel) |                                                                                                    |
| Structured Output Parser            | Output Parser (LangChain)           | Parses sentiment AI output to JSON schema | Sentiment Analyzer (ai_outputParser) | Sentiment Analyzer           |                                                                                                    |
| Topics & Keywords                   | Chain LLM (LangChain)               | Performs topics and keywords AI prompt | Format the Form Data         | Merge                        | Sticky Note1 covers Topics & Keywords extraction block                                             |
| Google Gemini Chat Model for Topics and Keywords | AI Language Model (Google Gemini)  | Provides AI model interface for topic extraction | Topics & Keywords (ai_languageModel) | Topics & Keywords (ai_languageModel) |                                                                                                    |
| Structured Output Parser for Topics & Keywords | Output Parser (LangChain)           | Parses topic extraction AI output to JSON schema | Topics & Keywords (ai_outputParser) | Topics & Keywords             |                                                                                                    |
| Merge                              | Merge                              | Merges sentiment and topic outputs    | Sentiment Analyzer, Topics & Keywords | Aggregate                   |                                                                                                    |
| Aggregate                         | Aggregate                         | Aggregates merged data before storage  | Merge                       | Append or update row in sheet |                                                                                                    |
| Append or update row in sheet      | Google Sheets                      | Writes aggregated analysis to spreadsheet | Aggregate                   | —                            |                                                                                                    |
| Sticky Note                       | Sticky Note                       | Branding and overview note             | —                           | —                            | "Uses Google Gemini AI for the sentiment analysis and topics + keyword extraction of Jotform content" |
| Sticky Note1                      | Sticky Note                       | Topics & Keywords block label          | —                           | —                            | "## Topics and Keyword Extraction using Google Gemini"                                            |
| Sticky Note2                      | Sticky Note                       | Sentiment Analysis block label         | —                           | —                            | "## Sentiment Analysis using Google Gemini"                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node:**  
   - Type: JotForm Trigger  
   - Configure with JotForm API credentials.  
   - Set Form ID to `252797914459475`.  
   - Save.

2. **Add Code Node "Format the Form Data":**  
   - Connect output of JotForm Trigger to this node.  
   - Use JavaScript code:  
     ```javascript
     const outputString = JSON.stringify($input.first().json, null, 2);
     return [{ json: { body: outputString } }];
     ```  
   - This converts raw form submission JSON to a formatted string under `body`.

3. **Create Chain LLM Node "Sentiment Analyzer":**  
   - Connect output from Format the Form Data node.  
   - Configure prompt:  
     - Text: `Perform sentiment analysis of the following {{ $json.body }}`  
     - Add system message: "You are an expert sentiment analyzer".  
   - Enable output parser.  
   - Set version 1.7.

4. **Create Google Gemini Chat Model Node for Sentiment:**  
   - Connect as AI language model for Sentiment Analyzer node.  
   - Use model name: `models/gemini-2.0-flash-exp`.  
   - Authenticate with Google Palm API credentials.

5. **Create Structured Output Parser Node for Sentiment:**  
   - Connect as output parser for Sentiment Analyzer node.  
   - Paste JSON schema defining sentiment output structure: customer info, sentiment, confidence score, key phrases, summary, alert priority, timestamp.  
   - Set schema type to manual.

6. **Create Chain LLM Node "Topics & Keywords":**  
   - Connect output from Format the Form Data node.  
   - Configure prompt similar to Sentiment Analyzer:  
     - Text: `Perform sentiment analysis of the following {{ $json.body }}` (used for topic extraction prompt).  
     - System message: "You are an expert sentiment analyzer".  
   - Enable output parser.  
   - Set version 1.7.

7. **Create Google Gemini Chat Model Node for Topics & Keywords:**  
   - Connect as AI language model for Topics & Keywords node.  
   - Use same model name and credentials as Sentiment Gemini node.

8. **Create Structured Output Parser Node for Topics & Keywords:**  
   - Connect as output parser for Topics & Keywords node.  
   - Paste JSON schema for topic extraction: topics array with topic name, keywords, sentiment, importance score; global keywords; insights; generation timestamp.  
   - Set schema type to manual.

9. **Add Merge Node:**  
   - Connect Sentiment Analyzer output to first input.  
   - Connect Topics & Keywords output to second input.  
   - Use default merge mode.

10. **Add Aggregate Node:**  
    - Connect output of Merge node.  
    - Configure to aggregate field `output`.

11. **Add Google Sheets Node "Append or update row in sheet":**  
    - Connect output of Aggregate node.  
    - Set operation: `appendOrUpdate`.  
    - Set Document ID to your Google Sheets spreadsheet ID (e.g., "13iJ2sOSaCEekzRNrhkUirZX2llBnpwGi3gdhI4PWIFM").  
    - Set Sheet Name to `Sheet1` (gid=0).  
    - Define columns:  
      - `feedback_analysis`: map to `={{ $json.output.toJsonString() }}`  
      - `topics_keywords`: map to `={{ $json.output[0].toJsonString() }}`  
    - Set matching column to `feedback_analysis`.  
    - Authenticate with Google Sheets OAuth2 credentials.

12. **Optional:** Add Sticky Notes for documentation and branding as per convenience.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                           |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| ![Logo](https://www.jotform.com/resources/assets/logo-nb/min/jotform-logo-white-400x200.png) Uses Google Gemini AI for the sentiment analysis and topics + keyword extraction of Jotform content | Branding and overview sticky note in the workflow                                        |
| ## Topics and Keyword Extraction using Google Gemini                                                     | Sticky Note labeling the Topics & Keywords extraction block                              |
| ## Sentiment Analysis using Google Gemini                                                                | Sticky Note labeling the Sentiment Analysis block                                        |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data processed is lawful and publicly accessible.