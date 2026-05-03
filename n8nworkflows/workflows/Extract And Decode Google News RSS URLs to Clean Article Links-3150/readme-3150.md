Extract And Decode Google News RSS URLs to Clean Article Links

https://n8nworkflows.xyz/workflows/extract-and-decode-google-news-rss-urls-to-clean-article-links-3150


# Extract And Decode Google News RSS URLs to Clean Article Links

### 1. Workflow Overview

This workflow automates the extraction and decoding of Google News RSS feed URLs to produce clean, direct article links. It is designed for developers, journalists, and content aggregators who need to process Google News RSS feeds by removing URL encoding and tracking parameters, enabling easier sharing, analysis, and further automated processing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Fetch and Limit RSS Feed:** Reads Google News RSS feed with configurable language and region parameters, then limits the number of articles processed to reduce API load.
- **1.3 Retrieve Encoded News URLs:** For each limited article, fetches the encoded Google News URL.
- **1.4 Extract Decoding Keys:** Parses the HTML content of each encoded URL to extract necessary decoding parameters (signature, timestamp).
- **1.5 Prepare Decoding Request:** Constructs the request payload using extracted keys and article identifiers.
- **1.6 Decode URLs:** Sends a POST request to Google’s internal decoding endpoint to obtain decoded URLs.
- **1.7 Clean and Format URLs:** Extracts clean URLs from the decoding response, removing unwanted characters.
- **1.8 Aggregate Results:** Collects all cleaned URLs into a single output object for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’
- **Node Details:**

| Node Name                    | Details                                                                                          |
|------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Type: Manual Trigger<br>Role: Initiates workflow execution manually.<br>Config: No parameters.<br>Input: None<br>Output: Triggers next node.<br>Edge cases: None.<br>Version: 1 |

---

#### 2.2 Fetch and Limit RSS Feed

- **Overview:** Reads Google News RSS feed with specified language and region parameters, then limits the number of articles processed to avoid excessive HTTP requests.
- **Nodes Involved:**  
  - Reading Google News RSS  
  - Limit  
  - Sticky Note1 (info on RSS parameters)  
  - Sticky Note2 (recommendation on limit)  
  - Sticky Note3 (general info and disclaimer)  
  - Sticky Note4 (comment on encoded content retrieval)
- **Node Details:**

| Node Name             | Details                                                                                              |
|-----------------------|-----------------------------------------------------------------------------------------------------|
| Reading Google News RSS | Type: RSS Feed Read<br>Role: Fetches RSS feed from Google News.<br>Config: URL set to `https://news.google.com/rss?hl=it&gl=IT&ceid=IT:it` (Italian language and region).<br>Input: Trigger from manual node.<br>Output: RSS items with encoded links.<br>Edge cases: Network errors, invalid RSS format, SSL issues.<br>Version: 1.1 |
| Limit                 | Type: Limit<br>Role: Restricts number of items processed.<br>Config: maxItems = 5.<br>Input: RSS feed items.<br>Output: Limited subset of RSS items.<br>Edge cases: If fewer than 5 items available, processes all.<br>Version: 1 |

- **Sticky Notes Context:**
  - Sticky Note1 explains how to change RSS parameters (hl, gl, ceid) for different languages/regions.
  - Sticky Note2 recommends limiting results to 3 to reduce HTTP requests.
  - Sticky Note3 warns about Google blocking IPs if run too often and notes the decoding is based on reverse engineering.
  - Sticky Note4 clarifies that this block retrieves encoded HTML content.

---

#### 2.3 Retrieve Encoded News URLs

- **Overview:** For each limited RSS item, fetches the encoded Google News URL (which contains tracking and encoded parameters).
- **Nodes Involved:**  
  - Get encoded news URL  
- **Node Details:**

| Node Name           | Details                                                                                              |
|---------------------|-----------------------------------------------------------------------------------------------------|
| Get encoded news URL | Type: HTTP Request<br>Role: Retrieves the HTML content of the encoded news URL.<br>Config: URL dynamically set from the `link` field of the limited RSS item.<br>Input: Limited RSS items.<br>Output: HTML content containing decoding keys.<br>Edge cases: HTTP errors, invalid URLs, timeouts.<br>Version: 4.2 |

---

#### 2.4 Extract Decoding Keys

- **Overview:** Parses the HTML content retrieved to extract decoding keys required for URL decoding.
- **Nodes Involved:**  
  - Extract decoding keys  
  - Map needed keys  
  - Sticky Note5 (explains decoding keys)  
  - Sticky Note6 (mapping variables)  
- **Node Details:**

| Node Name           | Details                                                                                              |
|---------------------|-----------------------------------------------------------------------------------------------------|
| Extract decoding keys | Type: HTML Extract<br>Role: Extracts `signature` and `timestamp` attributes from HTML `<div>` elements.<br>Config: CSS selector `div`, attributes `data-n-a-sg` (signature) and `data-n-a-ts` (timestamp).<br>Input: HTML content from encoded URL.<br>Output: JSON with signature and timestamp.<br>Edge cases: Missing attributes, malformed HTML.<br>Version: 1.2 |
| Map needed keys      | Type: Set<br>Role: Maps extracted keys plus base64 string from RSS item into a structured JSON.<br>Config: Assigns `signature`, `timestamp` from extracted data, and `base64Str` from RSS item GUID.<br>Input: Extracted keys and RSS item.<br>Output: JSON with all decoding variables.<br>Edge cases: Missing or invalid keys.<br>Version: 3.4 |

- **Sticky Notes Context:**
  - Sticky Note5 clarifies the decoding keys extracted: signature, timestamp, and base64 string.
  - Sticky Note6 notes the mapping of these variables for easier use downstream.

---

#### 2.5 Prepare Decoding Request

- **Overview:** Builds the POST request body required by Google’s internal decoding endpoint using the decoding keys.
- **Nodes Involved:**  
  - Prepare decoding variables  
  - Sticky Note7 (explains request preparation)  
- **Node Details:**

| Node Name              | Details                                                                                              |
|------------------------|-----------------------------------------------------------------------------------------------------|
| Prepare decoding variables | Type: Code (JavaScript)<br>Role: Constructs the `f.req` JSON payload for the decoding HTTP request.<br>Config: Uses `base64Str`, `timestamp`, and `signature` to build a nested JSON array string.<br>Input: JSON with decoding keys.<br>Output: JSON with `f_req` string.<br>Edge cases: JSON parsing errors, missing keys.<br>Version: 2 |

- **Sticky Notes Context:**
  - Sticky Note7 explains that this node prepares the specific body content needed for decoding.

---

#### 2.6 Decode URLs

- **Overview:** Sends the decoding request to Google’s internal endpoint and receives a response containing decoded URLs.
- **Nodes Involved:**  
  - Call decoding URL  
  - Decoded url  
  - Sticky Note8 (decoding step)  
  - Sticky Note9 (cleaning URL)  
- **Node Details:**

| Node Name         | Details                                                                                              |
|-------------------|-----------------------------------------------------------------------------------------------------|
| Call decoding URL | Type: HTTP Request<br>Role: Sends POST request to `https://news.google.com/_/DotsSplashUi/data/batchexecute` with `f.req` payload.<br>Config: Content-Type `application/x-www-form-urlencoded;charset=UTF-8`, custom User-Agent and Referer headers.<br>Input: JSON with `f_req`.<br>Output: Raw text response containing decoded URLs.<br>Edge cases: HTTP errors, rate limiting, malformed response.<br>Version: 4.2 |
| Decoded url       | Type: Set<br>Role: Parses the decoding response to extract the clean Google News URL.<br>Config: Uses complex JSON parsing expression to extract URL from nested response.<br>Input: HTTP response text.<br>Output: Clean URL string assigned to `google_news_url`.<br>Edge cases: Parsing errors if response format changes.<br>Version: 3.4 |

- **Sticky Notes Context:**
  - Sticky Note8 describes this as the decoding step sending the request.
  - Sticky Note9 notes that Google adds unwanted characters at the beginning of URLs, which are removed here.

---

#### 2.7 Aggregate Results

- **Overview:** Collects all decoded URLs into a single aggregated object for output.
- **Nodes Involved:**  
  - Aggregate results in a single object  
  - Sticky Note10 (output and further processing suggestions)  
- **Node Details:**

| Node Name                   | Details                                                                                              |
|-----------------------------|-----------------------------------------------------------------------------------------------------|
| Aggregate results in a single object | Type: Aggregate<br>Role: Aggregates all decoded URL items into one combined JSON object.<br>Config: Aggregates all item data.<br>Input: Decoded URL items.<br>Output: Single aggregated JSON object.<br>Edge cases: Large data sets may cause memory issues.<br>Version: 1 |

- **Sticky Notes Context:**
  - Sticky Note10 suggests further processing options like fetching article text, extracting content, or using AI nodes for classification or publishing.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                     |
|-------------------------------|---------------------|----------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger      | Starts workflow execution               | None                        | Reading Google News RSS      |                                                                                                |
| Reading Google News RSS        | RSS Feed Read       | Fetches Google News RSS feed            | When clicking ‘Test workflow’| Limit                       | Sticky Note1: Change language parameters (hl, gl, ceid)                                        |
| Limit                         | Limit               | Limits number of RSS items processed    | Reading Google News RSS      | Get encoded news URL         | Sticky Note2: Limit results to max 3 to reduce HTTP requests                                   |
| Get encoded news URL           | HTTP Request        | Retrieves encoded news article HTML     | Limit                       | Extract decoding keys        | Sticky Note4: Retrieves HTML content                                                          |
| Extract decoding keys          | HTML Extract        | Extracts decoding keys from HTML        | Get encoded news URL         | Map needed keys              | Sticky Note5: Extracts signature, timestamp, base64 string                                    |
| Map needed keys               | Set                 | Maps decoding keys and base64 string    | Extract decoding keys        | Prepare decoding variables   | Sticky Note6: Maps variables for easier use                                                   |
| Prepare decoding variables     | Code (JavaScript)   | Builds decoding request payload          | Map needed keys              | Call decoding URL            | Sticky Note7: Prepares POST body for decoding request                                        |
| Call decoding URL             | HTTP Request        | Sends decoding request to Google         | Prepare decoding variables   | Decoded url                 | Sticky Note8: Sends decoding request<br>Sticky Note9: Cleans unwanted characters from URL    |
| Decoded url                   | Set                 | Parses and extracts clean decoded URL    | Call decoding URL            | Aggregate results in a single object |                                                                                                |
| Aggregate results in a single object | Aggregate          | Aggregates all decoded URLs into one object | Decoded url                 | None                        | Sticky Note10: Suggests further processing options (content extraction, AI classification)    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually  
   - No parameters needed

2. **Add RSS Feed Read Node**  
   - Type: RSS Feed Read  
   - Parameters:  
     - URL: `https://news.google.com/rss?hl=it&gl=IT&ceid=IT:it` (adjust `hl`, `gl`, `ceid` for target language/region)  
     - Options: Leave SSL verification enabled unless needed otherwise  
   - Connect Manual Trigger output to this node input

3. **Add Limit Node**  
   - Type: Limit  
   - Parameters:  
     - maxItems: 5 (recommended to reduce HTTP requests)  
   - Connect RSS Feed Read output to Limit input

4. **Add HTTP Request Node to Get Encoded News URL**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: Set dynamically to `={{ $('Limit').item.json.link }}`  
     - Method: GET (default)  
     - Options: Default  
   - Connect Limit output to this node input

5. **Add HTML Extract Node to Extract Decoding Keys**  
   - Type: HTML Extract  
   - Parameters:  
     - Operation: Extract HTML content  
     - Extraction Values:  
       - Key: `signature`, CSS Selector: `div`, Attribute: `data-n-a-sg`  
       - Key: `timestamp`, CSS Selector: `div`, Attribute: `data-n-a-ts`  
   - Connect HTTP Request output to this node input

6. **Add Set Node to Map Needed Keys**  
   - Type: Set  
   - Parameters: Assignments:  
     - `signature` = `={{ $json.signature }}`  
     - `timestamp` = `={{ $json.timestamp }}`  
     - `base64Str` = `={{ $('Limit').item.json.guid }}`  
   - Connect HTML Extract output to this node input

7. **Add Code Node to Prepare Decoding Variables**  
   - Type: Code (JavaScript)  
   - Parameters: Use this JS code:

```javascript
return $input.all().map(item => {
    const gn_art_id = item.json.base64Str;
    const timestamp = item.json.timestamp;
    const signature = item.json.signature;

    const articlesReq = [
        'Fbv4je',
        `[\"garturlreq\",[[\"X\",\"X\",[\"X\",\"X\"],null,null,1,1,\"US:en\",null,1,null,null,null,null,null,0,1],\"X\",\"X\",1,[1,1,1],1,1,null,0,0,null,0],\"${gn_art_id}\",${timestamp},\"${signature}\"]`,
    ];

    return {
        json: {
            f_req: JSON.stringify([[articlesReq]])
        }
    };
});
```
   - Connect Set node output to this node input

8. **Add HTTP Request Node to Call Decoding URL**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://news.google.com/_/DotsSplashUi/data/batchexecute`  
     - Method: POST  
     - Content-Type: `application/x-www-form-urlencoded;charset=UTF-8`  
     - Headers:  
       - User-Agent: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36`  
       - Referer: `https://www.google.com/`  
     - Body Parameters:  
       - Name: `f.req`  
       - Value: `={{ $json.f_req }}`  
     - Response Format: Text (full response)  
   - Connect Code node output to this node input

9. **Add Set Node to Decode URL from Response**  
   - Type: Set  
   - Parameters:  
     - Assign `google_news_url` with expression:  
       ```javascript
       ={{ JSON.parse(JSON.parse($json.data.split('\n\n')[1])[0][2])[1] }}
       ```
     - This extracts the clean URL from the nested response text  
   - Connect HTTP Request output to this node input

10. **Add Aggregate Node to Collect All Results**  
    - Type: Aggregate  
    - Parameters:  
      - Aggregate: Aggregate all item data  
    - Connect Set node output to this node input

11. **Connect the nodes in the following order:**  
    Manual Trigger → RSS Feed Read → Limit → Get encoded news URL → Extract decoding keys → Map needed keys → Prepare decoding variables → Call decoding URL → Decoded url → Aggregate results

12. **Optional:**  
    - Add sticky notes for documentation and parameter explanations.  
    - Adjust RSS URL parameters (`hl`, `gl`, `ceid`) to target different languages or countries.  
    - Adjust Limit node to control number of articles processed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| You can add a cron trigger but avoid running too frequently to prevent Google IP blocking.                     | Sticky Note3                                                                                        |
| The decoding procedure is based on reverse engineering and may break if Google changes their internal API.    | Sticky Note3                                                                                        |
| After obtaining clean URLs, you can extend the workflow by fetching article text, extracting HTML content, or using AI nodes for classification and publishing. | Sticky Note10                                                                                       |
| Example of a custom node for content extraction: https://www.npmjs.com/package/n8n-nodes-webpage-content-extractor | Sticky Note10                                                                                       |
| Change RSS feed language and region by modifying `hl`, `gl`, and `ceid` parameters in the RSS Feed Read node. | Sticky Note1                                                                                        |

---

This documentation provides a detailed, stepwise understanding of the workflow, enabling users and AI agents to reproduce, modify, and extend it confidently.