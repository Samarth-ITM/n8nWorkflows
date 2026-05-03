Automated Stock Sentiment Analysis with Google Gemini and EODHD News API

https://n8nworkflows.xyz/workflows/automated-stock-sentiment-analysis-with-google-gemini-and-eodhd-news-api-5369


# Automated Stock Sentiment Analysis with Google Gemini and EODHD News API

### 1. Workflow Overview

This workflow automates daily sentiment analysis of stock market news for a list of stock tickers. It is designed to:

- Retrieve stock tickers from a Google Sheet.
- Fetch recent news articles for each ticker from the EODHD News API.
- Use Google Gemini’s large language model (via an AI Agent node) to analyze the sentiment of the news articles related to each stock.
- Output detailed sentiment scores and rationale back into a Google Sheet.
- Handle invalid or ticker symbols with no news by logging them separately.

The workflow can be logically divided into the following blocks:

- **1.1 Daily Trigger and Stock Ticker Retrieval:** Scheduling and fetching ticker list.
- **1.2 News Article Retrieval and Validation:** Fetching news and validating ticker presence.
- **1.3 Sentiment Analysis with AI:** Combining articles and running sentiment evaluation.
- **1.4 Output Formatting and Error Handling:** Parsing AI output and retry logic.
- **1.5 Storing the Results:** Writing sentiment results or invalid ticker logs back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger and Stock Ticker Retrieval

**Overview:**  
This block triggers the workflow daily at 4:00 PM Asia/Jerusalem time, reads stock tickers from a Google Sheet, and iterates over them individually for further processing.

**Nodes Involved:**  
- Schedule Trigger  
- Read_tickers_from_Sheet  
- loop_over_tickers  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule trigger node  
  - Configuration: Runs once daily at hour 16 (4 PM), timezone Asia/Jerusalem  
  - Inputs: None (trigger node)  
  - Outputs: Triggers the next node to start the workflow  
  - Edge Cases: Missed triggers if n8n instance is down; timezone misconfiguration can cause timing issues.

- **Read_tickers_from_Sheet**  
  - Type: Google Sheets node (read operation)  
  - Configuration: Reads from Google Sheets document "Stock Sentiment" (document ID: `1tQDBVDqn5v08GOsjupjV8o3Jzqd4-fKoSoGhWLEOTww`), sheet named "stocks" (gid: 470128021). Retrieves the list of stock tickers.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: An array of ticker objects  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Google API quotas, authentication failures, empty or malformed sheets.

- **loop_over_tickers**  
  - Type: SplitInBatches node  
  - Configuration: Processes each ticker individually by splitting the array of tickers into single items (batch size likely default 1)  
  - Inputs: List of tickers from Read_tickers_from_Sheet  
  - Outputs: Each ticker item to next node for processing  
  - Edge Cases: Empty ticker list leads to no downstream processing.

---

#### 1.2 News Article Retrieval and Validation

**Overview:**  
For each ticker, this block fetches recent news articles from EODHD API and validates ticker presence by checking if articles exist. Invalid tickers or those with no news are logged separately.

**Nodes Involved:**  
- Get articles from EODHD  
- If_ticker_not_valid  
- Write_in_google_sheets_invalid_ticker  
- join_articles_into_1  

**Node Details:**

- **Get articles from EODHD**  
  - Type: HTTP Request node  
  - Configuration: Calls `https://eodhd.com/api/news` with query parameters:
    - `s`: ticker symbol from current batch item
    - `offset`: 0
    - `limit`: 10 (fetch latest 10 news articles)
    - `fmt`: json (response format)  
  - Authentication: HTTP Query Authentication (API key or token in query parameters)  
  - Inputs: Current ticker from loop_over_tickers  
  - Outputs: API response with news articles array  
  - Edge Cases: API rate limits, network timeouts, invalid ticker returns empty or error response.

- **If_ticker_not_valid**  
  - Type: If node (conditional)  
  - Configuration: Checks if articles exist or ticker is valid by evaluating if input data is empty or not.  
  - Inputs: Output of Get articles from EODHD  
  - Outputs:  
    - True path: No articles found or invalid ticker  
    - False path: Valid ticker with articles  
  - Edge Cases: Misinterpretation if API changes structure; empty data might be due to network issues.

- **Write_in_google_sheets_invalid_ticker**  
  - Type: Google Sheets node (append operation)  
  - Configuration: Appends a row to "Sheet1" (gid=0) in the same Google Sheets document.  
  - Columns written:
    - `date`: current date (today)
    - `stock`: ticker symbol
    - `sentimentScore`: "Invalid Ticker"  
  - Inputs: If_ticker_not_valid (True path)  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Google API limits, sheet access errors.

- **join_articles_into_1**  
  - Type: Code node (JavaScript)  
  - Configuration: Takes all news article JSON objects for the ticker and combines them into one JSON string under property `fullString`.  
  - Inputs: If_ticker_not_valid (False path - valid articles)  
  - Outputs: Single item with `fullString` containing combined news articles JSON string.  
  - Edge Cases: Large combined string may hit size limits; malformed article content can cause issues.

---

#### 1.3 Sentiment Analysis with AI

**Overview:**  
This block sends the combined news articles along with the ticker symbol to a configured AI Agent node that leverages Google Gemini's Chat Model to analyze stock sentiment and produce a scored output with rationale.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model1  

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent node  
  - Configuration:  
    - Prompt instructs the AI to analyze stock sentiment based on news content.
    - Input format specifies symbol, title, content.
    - Output format expects JSON with: symbol, sentiment_score (float -1 to 1), rationale (text).
    - Uses data from:
      - Current ticker symbol (`loop_over_tickers`)
      - Combined article JSON string (`join_articles_into_1`)  
  - Inputs: Output of join_articles_into_1  
  - Outputs: Raw AI model response string (includes JSON inside markdown-style formatting)  
  - Edge Cases: Model timeout, API quota, malformed prompt or response, unexpected output format.

- **Google Gemini Chat Model1**  
  - Type: LangChain Chat Model node using Google Gemini (PaLM) API  
  - Configuration:
    - Model: "models/gemini-2.0-flash"
    - Max output tokens: 2048
    - Credentials: Google Palm API account  
  - Inputs: Connected as language model backend to AI Agent node  
  - Edge Cases: API quota limits, transient errors from Google, model version changes.

---

#### 1.4 Output Formatting and Error Handling

**Overview:**  
This block parses the AI Agent’s raw output string to extract the clean JSON object with sentiment data, then verifies success. If parsing fails or an error flag is set, the workflow loops back to retry.

**Nodes Involved:**  
- format_output_as_json  
- if_format_succesful  

**Node Details:**

- **format_output_as_json**  
  - Type: Code node (JavaScript)  
  - Configuration:  
    - Extracts JSON object embedded inside a markdown code block from AI output string.
    - Parses JSON string to object with keys: symbol, sentiment_score, rationale.
    - Returns parsed JSON for downstream usage.  
  - Inputs: Output of AI Agent  
  - Outputs: Parsed JSON object  
  - Edge Cases: Parsing errors if AI output format changes, missing or corrupted JSON, runtime exceptions.

- **if_format_succesful**  
  - Type: If node (conditional)  
  - Configuration: Checks existence of `$json.error` to detect parsing or AI errors.  
  - Inputs: Output of format_output_as_json  
  - Outputs:
    - True path (error detected): loops back to AI Agent to retry sentiment analysis.
    - False path (success): proceeds to store results.  
  - Edge Cases: Infinite retry loops if error condition persists, logic misinterpretation of error state.

---

#### 1.5 Storing the Results

**Overview:**  
This final block records the successful sentiment analysis results into the Google Sheet, then loops back to process the next ticker in the batch.

**Nodes Involved:**  
- write_sentiment_to_sheets  

**Node Details:**

- **write_sentiment_to_sheets**  
  - Type: Google Sheets node (append operation)  
  - Configuration:  
    - Appends a new row to "Sheet1" (gid=0) of the Google Sheet "Stock Sentiment".  
    - Columns:
      - `date`: current date
      - `stock`: ticker symbol (with ".US" suffix removed)
      - `sentimentScore`: sentiment score from AI
      - `rational`: rationale text from AI  
  - Inputs: if_format_succesful (False path - success)  
  - Outputs: Loops back to `loop_over_tickers` to process next ticker  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Google API quota, malformed data, sheet write conflicts.

---

### 3. Summary Table

| Node Name                      | Node Type                             | Functional Role                         | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                         |
|--------------------------------|-------------------------------------|---------------------------------------|-----------------------------|------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                    | Daily automatic start at 4 PM         | None                        | Read_tickers_from_Sheet            | # 1. Daily Trigger and Stock Ticker Retrieval                                                     |
| Read_tickers_from_Sheet        | Google Sheets                      | Reads list of tickers                  | Schedule Trigger            | loop_over_tickers                  | # 1. Daily Trigger and Stock Ticker Retrieval                                                     |
| loop_over_tickers              | SplitInBatches                    | Iterates over each ticker individually| Read_tickers_from_Sheet     | Get articles from EODHD (2nd output) | # 1. Daily Trigger and Stock Ticker Retrieval                                                     |
| Get articles from EODHD        | HTTP Request                      | Fetches latest news articles           | loop_over_tickers (2nd output) | If_ticker_not_valid                | # 2. News Article Retrieval and Validation                                                        |
| If_ticker_not_valid            | If (Conditional)                  | Checks if ticker is valid / has news   | Get articles from EODHD      | Write_in_google_sheets_invalid_ticker (True), join_articles_into_1 (False) | # 2. News Article Retrieval and Validation                                                        |
| Write_in_google_sheets_invalid_ticker | Google Sheets                | Logs invalid tickers                   | If_ticker_not_valid (True)   | None                             | # 2. News Article Retrieval and Validation                                                        |
| join_articles_into_1           | Code (JavaScript)                 | Combines news articles into one string | If_ticker_not_valid (False)  | AI Agent                         | # 2. News Article Retrieval and Validation                                                        |
| AI Agent                      | LangChain Agent                   | Runs sentiment analysis via AI         | join_articles_into_1         | format_output_as_json              | # 3. Sentiment Analysis with AI                                                                  |
| Google Gemini Chat Model1      | LangChain Chat Model (Google Gemini) | Provides language model backend for AI Agent | AI Agent (as LLM)            | AI Agent                         | # 3. Sentiment Analysis with AI                                                                  |
| format_output_as_json          | Code (JavaScript)                 | Extracts clean JSON from raw AI output | AI Agent                   | if_format_succesful               | # 4. Output Formatting and Error Handling                                                        |
| if_format_succesful            | If (Conditional)                  | Checks if JSON formatting was successful | format_output_as_json       | AI Agent (True - error), write_sentiment_to_sheets (False - success) | # 4. Output Formatting and Error Handling                                                        |
| write_sentiment_to_sheets      | Google Sheets                    | Writes sentiment results to Google Sheet | if_format_succesful (False) | loop_over_tickers                | # 5. Storing the Results                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to run daily at 4:00 PM (16:00) in timezone Asia/Jerusalem.

2. **Add Google Sheets node (Read_tickers_from_Sheet)**  
   - Set operation to "Read" rows.  
   - Configure document ID: `1tQDBVDqn5v08GOsjupjV8o3Jzqd4-fKoSoGhWLEOTww`  
   - Set sheet name or GID to "stocks" (gid: 470128021).  
   - Connect Schedule Trigger output to this node input.  
   - Use Google Sheets OAuth2 credentials.

3. **Add SplitInBatches node (loop_over_tickers)**  
   - Default batch size (1).  
   - Connect Read_tickers_from_Sheet output to this node input.

4. **Add HTTP Request node (Get articles from EODHD)**  
   - Set method to GET.  
   - URL: `https://eodhd.com/api/news`  
   - Add query parameters:  
     - `s` = current ticker symbol from `{{$json.ticker}}`  
     - `offset` = `0`  
     - `limit` = `10`  
     - `fmt` = `json`  
   - Use HTTP Query Authentication with your EODHD API key.  
   - Connect loop_over_tickers second output (batch items) to this node input.

5. **Add If node (If_ticker_not_valid)**  
   - Condition: Check if the output of Get articles from EODHD is empty or no articles returned.  
   - Connect Get articles from EODHD output to this node input.

6. **Add Google Sheets node (Write_in_google_sheets_invalid_ticker)**  
   - Operation: Append row to "Sheet1" (gid=0) of the same Google Sheet document.  
   - Columns:  
     - `date` = current date (`{{$today}}`)  
     - `stock` = ticker symbol (`{{$json.ticker}}`)  
     - `sentimentScore` = "Invalid Ticker"  
   - Connect If_ticker_not_valid True path to this node input.  
   - Use same Google Sheets OAuth2 credentials.

7. **Add Code node (join_articles_into_1)**  
   - JavaScript code: combine array of article JSONs into one JSON string under `fullString`.  
   - Connect If_ticker_not_valid False path to this node input.

8. **Add LangChain Agent node (AI Agent)**  
   - Configure prompt template to instruct stock sentiment analysis with detailed input/output format as per the prompt in the original workflow.  
   - Input variables:  
     - `symbol` = `{{$('loop_over_tickers').all()[0].json.ticker}}`  
     - `news content` = `{{$('join_articles_into_1').all()[0].json.fullString}}`  
   - Connect join_articles_into_1 output to AI Agent input.

9. **Add LangChain Chat Model node (Google Gemini Chat Model1)**  
   - Model: "models/gemini-2.0-flash"  
   - Max output tokens: 2048  
   - Connect this node as the language model backend of AI Agent.  
   - Use Google Palm API credentials.

10. **Add Code node (format_output_as_json)**  
    - JavaScript to extract JSON string from AI Agent raw output and parse it.  
    - Connect AI Agent output to this node input.

11. **Add If node (if_format_succesful)**  
    - Condition: Check if `$json.error` exists.  
    - Connect format_output_as_json output to this node input.

12. **Connect if_format_succesful**  
    - True path (error): connect back to AI Agent input to retry.  
    - False path (success): connect to write_sentiment_to_sheets node.

13. **Add Google Sheets node (write_sentiment_to_sheets)**  
    - Operation: Append row to "Sheet1" in the same Google Sheet.  
    - Columns:  
      - `date` = current date (`{{$today}}`)  
      - `stock` = ticker symbol with ".US" suffix removed (`{{$('loop_over_tickers').all()[0].json.ticker.replace(".US","")}}`)  
      - `sentimentScore` = sentiment score from AI output  
      - `rational` = rationale text from AI output  
    - Connect if_format_succesful False path to this node input.  
    - Connect output of this node back to loop_over_tickers input to process next ticker.

14. **Configure credentials**  
    - Google Sheets OAuth2 for all Google Sheets nodes.  
    - HTTP Query Authentication for EODHD API.  
    - Google Palm API credentials for Google Gemini Chat Model node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Workflow automates sentiment analysis of stock news using Google Gemini and EODHD API, writing results to Google Sheets.    | Workflow Overview Sticky Note                                                                                        |
| Scheduled daily at 4:00 PM Asia/Jerusalem time for timely stock market sentiment updates.                                    | Sticky Note1                                                                                                         |
| Uses EODHD API to fetch latest 10 news articles per ticker.                                                                 | Sticky Note2                                                                                                         |
| AI Agent prompt is detailed to produce sentiment scores between -1 and 1 with an explanation in JSON format.                | Sticky Note3                                                                                                         |
| Includes robust error handling by retrying AI sentiment analysis if output JSON parsing fails.                              | Sticky Note4                                                                                                         |
| Results and invalid tickers are logged back into the same Google Sheets document for easy monitoring and record keeping.   | Sticky Note5                                                                                                         |
| Google Sheets document used: https://docs.google.com/spreadsheets/d/1tQDBVDqn5v08GOsjupjV8o3Jzqd4-fKoSoGhWLEOTww           | Reference for documentId                                                                                             |
| EODHD API docs: https://eodhd.com/api                                                                                      | For API authentication and usage details                                                                             |
| Google Gemini (PaLM) models require a valid Google Cloud API account with PaLM access configured in n8n credentials.         | Credential setup note                                                                                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly available.