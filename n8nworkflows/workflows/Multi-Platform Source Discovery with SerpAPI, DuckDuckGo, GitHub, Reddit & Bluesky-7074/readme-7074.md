Multi-Platform Source Discovery with SerpAPI, DuckDuckGo, GitHub, Reddit & Bluesky

https://n8nworkflows.xyz/workflows/multi-platform-source-discovery-with-serpapi--duckduckgo--github--reddit---bluesky-7074


# Multi-Platform Source Discovery with SerpAPI, DuckDuckGo, GitHub, Reddit & Bluesky

### 1. Workflow Overview

This workflow, titled **"Multi-Platform Source Discovery with SerpAPI, DuckDuckGo, GitHub, Reddit & Bluesky"**, is designed to automate the discovery of up-to-date, relevant information sources across multiple platforms. Instead of manually searching for news or articles, it identifies potential websites and repositories related to specified search themes by querying popular platforms, filtering out unwanted or existing sources, and evaluating relevance through content extraction.

**Target Use Cases:**
- Academic and educational resource compilation
- Journalism and research source discovery
- Content marketing and competitor analysis
- Automated curation of useful websites and repositories related to specific topics

**Logical Blocks:**

- **1.1 Setup and Input Configuration**  
  Handles workflow triggering, input parameter configuration, and preparation of search themes and source lists.

- **1.2 Search on Web Platforms**  
  Executes searches on DuckDuckGo, Bluesky, SerpAPI (Google News), Reddit, and GitHub to gather potential source URLs.

- **1.3 Data Preparation and Cleaning**  
  Extracts, formats, and filters search results; removes duplicates, empty links, and undesired or existing sources.

- **1.4 Content Retrieval and Relevance Evaluation**  
  Fetches website content, limits content size, and assesses relevance and recency using a language model information extractor.

- **1.5 Final Output Preparation**  
  Aggregates and prepares filtered, evaluated sources for user review.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Input Configuration

**Overview:**  
This block initializes the workflow, sets input arguments such as search themes, known sources, and sources to ignore, and manages scheduling or manual start.

**Nodes Involved:**  
- Schedule  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Configure Workflow Args (Global Constants Node)  
- Configure Workflow Args (Manual)  
- Configure Workflow Args (Prep)  
- Edit Fields  
- Add Other Fields  
- Split Out  

**Node Details:**

- **Schedule** (Schedule Trigger)  
  - Triggers workflow execution on specified days of the week at 12:00 PM.  
  - Used for automated periodic runs.  
  - Edge cases: incorrect time zone settings or missed triggers.

- **When clicking ‘Execute workflow’** (Manual Trigger)  
  - Allows manual execution without Global Variables Node.  
  - No credentials needed.  
  - No outputs, triggers downstream nodes.

- **Configure Workflow Args** (Global Constants Node)  
  - Stores constant variables: search themes, existing sources, sources to ignore, RSS feeds.  
  - Requires credential for Global Constants API.  
  - Helps centralize configuration.

- **Configure Workflow Args (Manual)** (Set Node)  
  - Alternative input setup without Global Constants Node.  
  - Defines arrays for `search_themes`, `existing_sources`, and `sources_to_ignore`.  
  - Useful for quick testing or variant workflows.

- **Configure Workflow Args (Prep)** (Set Node)  
  - Prepares and passes workflow arguments downstream.  
  - Assigns arrays `search_themes`, `existing_sources`, and `sources_to_ignore` from input JSON.  
  - Input from either manual or global constants node.

- **Edit Fields** (Set Node)  
  - Assigns `rss_feeds` from constants for later use.  
  - No transformation, just forwarding data.

- **Add Other Fields** (Set Node)  
  - Combines existing sources from constants and aggregated lists.  
  - Prepares final lists for filtering.  
  - Uses concatenation expressions.  
  - Edge cases: empty arrays handled gracefully.

- **Split Out** (SplitOut)  
  - Splits input array `search_themes` to process each theme separately.  
  - Enables parallel or iterative handling in subsequent nodes.

---

#### 1.2 Search on Web Platforms

**Overview:**  
This block performs searches on multiple platforms to find URLs related to the specified themes.

**Nodes Involved:**  
- DuckDuckGo  
- Bluesky Search  
- Google_news search (SerpAPI)  
- Get list of Subreddits (Reddit)  
- RSS Read (Reddit RSS feed)  
- HTTP Request (GitHub Search Repos)  
- Loop Over Items (Search)  
- Loop Over Items (Search) 2  

**Node Details:**

- **DuckDuckGo** (DuckDuckGo Search)  
  - Searches news with query from `search_themes`.  
  - Parameters: max 100 results, strict safe search enabled.  
  - Outputs news items with title, date, and URL.  
  - Edge cases: API unavailability, network timeouts.

- **Bluesky Search** (Custom Bluesky Node)  
  - Searches posts with query from `search_themes`.  
  - Limit set to 10 results.  
  - Requires Bluesky API credentials.  
  - Returns posts with author displayName and external links.  
  - Edge cases: auth failure, rate limits.

- **Google_news search** (SerpAPI Node)  
  - Searches Google News with query from `search_themes`.  
  - Requires SerpAPI credential.  
  - Outputs news articles with title, date, and links.  
  - Edge cases: API quota limits or errors.

- **Get list of Subreddits** (Reddit Node)  
  - Retrieves subreddits matching `search_themes`.  
  - Limit 1 subreddit (only first considered).  
  - Edge cases: 403 forbidden handled by workflow continuation.

- **RSS Read** (RSS Feed Reader)  
  - Reads subreddit RSS feed constructed from subreddit name.  
  - Parses posts for content.

- **HTTP Request** (GitHub API Search)  
  - Searches GitHub repositories with query `awesome links for <search_themes>`.  
  - Retrieves up to 10 repos per request, paginated (1 page).  
  - Edge cases: rate limits without credentials, network errors.

- **Loop Over Items (Search)** and **Loop Over Items (Search) 2** (SplitInBatches)  
  - Controls batch size for iterative processing of search queries.  
  - Supports splitting large input arrays for manageable search requests.

---

#### 1.3 Data Preparation and Cleaning

**Overview:**  
This block extracts relevant fields, filters out empty or duplicate links, removes known or ignored sources, and prepares the data for content evaluation.

**Nodes Involved:**  
- Filter Fields  
- Edit Fields (Bluesky)  
- Remove Elements with Empty Links  
- Remove Elements with Empty Links 2  
- Remove Empty  
- Remove Duplicates  
- Remove Duplicates Reddit Sources  
- Remove Duplicates Sources  
- Remove Duplicates Urls  
- Remove Existing Content  
- Remove Ignored Sources  
- Get Base URL  
- Get Base URL of Sources  
- Get Base URL of Reddit Sources  
- Get Base URL of Github Sources  
- Grab Links from Threads (Reddit)  
- Grab Links from README (GitHub)  
- Prepare Reddit Source Results  
- Prepare Github Source Results  
- Split Out Feed List  
- Split Out News Results  
- Split Out Sources  
- Split Out Github Sources  
- Split Out Repo List  
- Split Out  

**Node Details:**

- **Filter Fields** (Set Node)  
  - Extracts and renames fields `title`, `pubDate`, and `link` from DuckDuckGo results.  
  - Streamlines data for downstream processing.

- **Edit Fields (Bluesky)** (Set Node)  
  - Constructs title by combining "Bluesky Search" with author and external title.  
  - Assigns `pubDate` and `link` from indexedAt and embed fields.

- **Remove Elements with Empty Links** (Filter Node)  
  - Removes entries where the `link` field is empty or missing.  
  - Ensures data quality before merging.

- **Remove Empty** (Filter Node)  
  - Filters out entries missing `link` or with titles that are too short (less than 3 words).  
  - Avoids irrelevant or malformed entries.

- **Remove Duplicates, Remove Duplicates Reddit Sources, Remove Duplicates Sources, Remove Duplicates Urls** (RemoveDuplicates Nodes)  
  - Removes duplicate entries based on all fields except `pubDate` or specific fields like `baseUrl`.  
  - Helps reduce redundancy.

- **Remove Existing Content** (Filter Node)  
  - Filters out sources whose base URL is in the `existing_sources` list.  
  - Prevents rediscovery of known sources.

- **Remove Ignored Sources** (Filter Node)  
  - Filters out sources whose base URL is in the `sources_to_ignore` list.  
  - Avoids undesired sources.

- **Get Base URL, Get Base URL of Sources, Get Base URL of Reddit Sources, Get Base URL of Github Sources** (Code Nodes)  
  - Extracts the base URL (protocol + domain) from full URLs to normalize links for comparison.  
  - Prevents treating different URLs of the same domain as separate entries.

- **Grab Links from Threads** (Code Node)  
  - Extracts URLs from Reddit post content using regex matching.  
  - Adds extracted links as `sources` array.

- **Grab Links from README** (Code Node)  
  - Extracts URLs from GitHub README file content using regex.  
  - Adds as `sources` array.

- **Prepare Reddit Source Results, Prepare Github Source Results** (Set Nodes)  
  - Formats the extracted links into uniform objects with `title`, `pubDate` (current date), and `link` (base URL).  
  - Prepares data for merging.

- **Split Out Feed List, Split Out News Results, Split Out Sources, Split Out Github Sources, Split Out Repo List, Split Out** (SplitOut Nodes)  
  - Splits arrays into individual items for processing.  
  - Enables iterative or parallel operations.

---

#### 1.4 Content Retrieval and Relevance Evaluation

**Overview:**  
This block fetches website content for filtered sources, limits content size for efficiency, and uses a language model-based information extractor to evaluate relevance, update recency, and accessibility.

**Nodes Involved:**  
- Limit  
- HTTP Request to get URL Content  
- Code (Limiting number of Chars)  
- Mistral Cloud Chat Model  
- Information Extractor  
- Remove Errors  

**Node Details:**

- **Limit** (Limit Node)  
  - Restricts the number of sources processed in this block to 36 by default.  
  - Prevents overloading the content retrieval and LLM evaluation steps.

- **HTTP Request to get URL Content** (HTTP Request Node)  
  - Performs GET requests on each source's base URL to retrieve website content.  
  - Timeout configured to 2100 ms (2.1 seconds).  
  - Continues on error to avoid workflow halt.

- **Code (Limiting number of Chars)** (Code Node)  
  - Truncates retrieved content to first 1500 characters to fit LLM context limits.  
  - Handles cases where content is missing or not a string.

- **Mistral Cloud Chat Model** (Language Model Node)  
  - Uses Mistral Cloud API to process data.  
  - Connected as AI language model for Information Extractor.

- **Information Extractor** (LangChain Information Extractor)  
  - Uses a custom JSON schema to extract from website content:  
    - URL  
    - Relevance (boolean)  
    - Updated (string describing recency)  
    - Accessible (boolean)  
  - System prompt instructs expert extraction and omission of unknown attributes.  
  - Continues on error to allow partial results.

- **Remove Errors** (Filter Node)  
  - Filters out any items with errors from HTTP requests or extraction steps.  
  - Ensures only valid data continues.

---

#### 1.5 Final Output Preparation

**Overview:**  
This block merges data from all search sources, filters and deduplicates final results, and prepares them for user consumption or further automation.

**Nodes Involved:**  
- Merge  
- Merge1  
- Aggregate  
- Leave Remaining Sources  
- Remove Existing Content  
- Remove Ignored Sources  
- Remove Duplicates  
- Remove Empty  
- Remove Duplicates Urls  
- Sticky Notes (for annotations)  

**Node Details:**

- **Merge, Merge1** (Merge Nodes)  
  - Merge various search results into single data streams.  
  - Merge1 combines DuckDuckGo and Bluesky results; Merge combines all platform results.

- **Aggregate** (Aggregate Node)  
  - Aggregates base URLs from filtered sources for reuse in filtering steps.

- **Leave Remaining Sources** (Set Node)  
  - Passes forward base URLs after filtering.  
  - Helps finalize data for output.

- **Remove Existing Content, Remove Ignored Sources, Remove Duplicates, Remove Empty, Remove Duplicates Urls**  
  - Final filtering and deduplication for high-quality, unique source list.

- **Sticky Note Nodes**  
  - Provide detailed comments, observations, and usage notes throughout the workflow for user clarity.

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                                 | Input Node(s)                       | Output Node(s)                                 | Sticky Note                                                                                                               |
|--------------------------------|---------------------------------|------------------------------------------------|-----------------------------------|------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Schedule                       | Schedule Trigger                | Scheduled workflow start                        |                                   | Configure Workflow Args                        | Setup and Start: Schedule is the standard method for operation.                                                            |
| When clicking ‘Execute workflow’ | Manual Trigger                 | Manual workflow start                           |                                   | Configure Workflow Args (Manual)               | Setup and Start: Manual alternative to Schedule.                                                                           |
| Configure Workflow Args        | Global Constants                | Provide global input variables                  | Schedule                           | Edit Fields                                    | Configure Workflow Args: useful for quick setup and sharing.                                                               |
| Configure Workflow Args (Manual) | Set                          | Manual input variables alternative              | When clicking ‘Execute workflow’  | Configure Workflow Args (Prep)                  | Replace Global Constants Node if needed.                                                                                   |
| Configure Workflow Args (Prep) | Set                            | Prepare inputs for downstream processing        | Configure Workflow Args (Manual)  | Split Out                                       | Assigns search themes and source lists.                                                                                    |
| Edit Fields                   | Set                            | Assign rss_feeds from constants                  | Configure Workflow Args           | Split Out Feed List                             |                                                                                                                           |
| Add Other Fields              | Set                            | Combine existing sources and constants           | Aggregate                        | Configure Workflow Args (Prep)                  |                                                                                                                           |
| Split Out                    | SplitOut                       | Split search themes array for processing         | Configure Workflow Args (Prep)    | Loop Over Items (Search), Loop Over Items (Search) 2, Get list of Subreddits, HTTP Request |                                                                                                                           |
| DuckDuckGo                   | DuckDuckGo Search              | Search news on DuckDuckGo                         | Loop Over Items (Search)           | Filter Fields                                   | Search Queries with DuckDuckGo: max 100 results, Safe Search Strict.                                                       |
| Bluesky Search              | Bluesky Enhanced               | Search posts on Bluesky                            | Loop Over Items (Search)           | Edit Fields (Bluesky)                           | Search Queries with Bluesky: limit 10 results.                                                                             |
| Filter Fields                | Set                            | Extract relevant fields from DuckDuckGo results | DuckDuckGo                       | Merge1                                          |                                                                                                                           |
| Edit Fields (Bluesky)        | Set                            | Format Bluesky search results                     | Bluesky Search                   | Remove Elements with Empty Links                 |                                                                                                                           |
| Remove Elements with Empty Links | Filter                      | Remove entries with empty links                   | Edit Fields (Bluesky)             | Merge1                                          |                                                                                                                           |
| Merge1                      | Merge                          | Merge DuckDuckGo and Bluesky results             | Filter Fields, Remove Elements with Empty Links | Loop Over Items (Search)                       |                                                                                                                           |
| Google_news search           | SerpAPI                        | Search Google News                                | Loop Over Items (Search) 2         | Split Out News Results                          | Search Queries with SerpAPI: uses Google Search.                                                                           |
| Split Out News Results       | SplitOut                       | Split news results array                           | Google_news search               | Remove Elements with Empty Links 2               |                                                                                                                           |
| Remove Elements with Empty Links 2 | Filter                   | Remove entries with empty links                   | Split Out News Results           | Filter Fields for Loop                           | SerpAPI results sometimes contain empty links.                                                                            |
| Filter Fields for Loop       | Set                            | Extract fields for looping                         | Remove Elements with Empty Links 2 | Loop Over Items (Search) 2                       |                                                                                                                           |
| HTTP Request                | HTTP Request                   | GitHub repository search                          | Split Out                       | Split Out Repo List                             | Search Queries with GitHub: recommended to use credentials to avoid rate limits.                                          |
| Split Out Repo List          | SplitOut                       | Split GitHub repos array                           | HTTP Request                    | Add README field                                |                                                                                                                           |
| Add README field             | Set                            | Prepare README URL for GitHub repos               | Split Out Repo List             | HTTP Request the READMEs                         |                                                                                                                           |
| HTTP Request the READMEs     | HTTP Request                   | Get GitHub repo README content                     | Add README field               | Grab READMEs                                    |                                                                                                                           |
| Grab READMEs                | HTTP Request                   | Fetch README file data                             | HTTP Request the READMEs         | Remove Errors                                   |                                                                                                                           |
| Remove Errors               | Filter                         | Filter out errored GitHub README fetches          | Grab READMEs                   | Grab Links from README                           |                                                                                                                           |
| Grab Links from README       | Code                          | Extract URLs from README text                      | Remove Errors                   | Split Out Github Sources                        |                                                                                                                           |
| Split Out Github Sources     | SplitOut                      | Split extracted GitHub links array                | Grab Links from README          | Get Base URL of Github Sources                  |                                                                                                                           |
| Get Base URL of Github Sources | Code                         | Normalize GitHub URLs to base URLs                 | Split Out Github Sources        | Prepare Github Source Results                    |                                                                                                                           |
| Prepare Github Source Results | Set                           | Format GitHub sources for merging                  | Get Base URL of Github Sources  | Merge                                          |                                                                                                                           |
| Get list of Subreddits       | Reddit                        | Get subreddit matching search themes               | Split Out                      | Prepare Reddit RSS                              | Search Queries with Reddit: only first subreddit used; 403 errors tolerated.                                              |
| Prepare Reddit RSS           | Set                           | Construct Reddit RSS feed URL from subreddit name  | Get list of Subreddits          | RSS Read                                        |                                                                                                                           |
| RSS Read                    | RSS Feed Read                  | Read posts from Reddit RSS feed                     | Prepare Reddit RSS             | Grab Links from Threads                         |                                                                                                                           |
| Grab Links from Threads      | Code                          | Extract URLs from Reddit post content               | RSS Read                      | Split Out Sources                               |                                                                                                                           |
| Split Out Sources            | SplitOut                      | Split Reddit extracted links array                  | Grab Links from Threads         | Get Base URL of Reddit Sources                   |                                                                                                                           |
| Get Base URL of Reddit Sources | Code                        | Normalize Reddit URLs to base URLs                   | Split Out Sources              | Remove Duplicates Reddit Sources                 |                                                                                                                           |
| Remove Duplicates Reddit Sources | RemoveDuplicates            | Deduplicate Reddit sources by base URL              | Get Base URL of Reddit Sources  | Prepare Reddit Source Results                     |                                                                                                                           |
| Prepare Reddit Source Results | Set                          | Format Reddit sources for merging                    | Remove Duplicates Reddit Sources | Merge                                          |                                                                                                                           |
| Merge                       | Merge                         | Merge results from GitHub, Reddit, DuckDuckGo, Bluesky | Prepare Github Source Results, Prepare Reddit Source Results, Merge1 | Remove Empty                                   |                                                                                                                           |
| Remove Empty                | Filter                        | Remove entries with empty or invalid links/titles  | Merge                         | Remove Duplicates                               |                                                                                                                           |
| Remove Duplicates            | RemoveDuplicates              | Deduplicate merged results (excluding pubDate)     | Remove Empty                 | Get Base URL                                    |                                                                                                                           |
| Get Base URL                 | Code                          | Normalize merged URLs to base URLs                   | Remove Duplicates             | Remove Existing Content                          |                                                                                                                           |
| Remove Existing Content      | Filter                        | Remove sources already known (existing_sources list) | Get Base URL                 | Leave Remaining Sources                          |                                                                                                                           |
| Leave Remaining Sources      | Set                           | Pass remaining base URLs                             | Remove Existing Content       | Remove Duplicates Urls                           |                                                                                                                           |
| Remove Duplicates Urls       | RemoveDuplicates              | Deduplicate URLs by base URL                          | Leave Remaining Sources       | Remove Ignored Sources                           |                                                                                                                           |
| Remove Ignored Sources       | Filter                        | Remove sources in sources_to_ignore list             | Remove Duplicates Urls        | Limit                                           |                                                                                                                           |
| Limit                       | Limit                         | Limit number of sources sent for content analysis    | Remove Ignored Sources        | HTTP Request to get URL Content                  | Final Step: Source Selection; limits default to 36 sources.                                                               |
| HTTP Request to get URL Content | HTTP Request                | Retrieve website content for each source             | Limit                        | Code (Limiting number of Chars)                  | May face errors such as site blocking GET requests.                                                                       |
| Code (Limiting number of Chars) | Code                       | Truncate website content to 1500 characters           | HTTP Request to get URL Content | Information Extractor                            |                                                                                                                           |
| Mistral Cloud Chat Model    | Language Model                | Provides LLM processing for information extraction    |                              | Information Extractor                            |                                                                                                                           |
| Information Extractor       | LangChain Information Extractor | Extracts relevance, update recency, accessibility info from content | Code (Limiting number of Chars), Mistral Cloud Chat Model |                                                    |                                                                                                                           |
| Remove Errors               | Filter                        | Filters out any errored content retrieval or extraction | Grab READMEs, HTTP Request the READMEs | Grab Links from README                          |                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node named `Schedule` with weekly intervals on Monday-Friday at 12:00 PM.  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’` for manual runs.

2. **Configure Input Variables:**  
   - Add a **Global Constants** node named `Configure Workflow Args` to store input arrays: `search_themes`, `existing_sources`, `sources_to_ignore`, and `rss_feeds`. Provide associated credentials for Global Constants API.  
   - Add a **Set** node `Configure Workflow Args (Manual)` as alternative, defining default arrays for `search_themes` (e.g., `["technology"]`), `existing_sources` (empty), and `sources_to_ignore` (empty).  
   - Connect `Schedule` to `Configure Workflow Args` and `When clicking ‘Execute workflow’` to `Configure Workflow Args (Manual)`.  
   - Add a **Set** node `Configure Workflow Args (Prep)` to assign those arrays from input JSON, connect both `Configure Workflow Args` and `Configure Workflow Args (Manual)` to it.  
   - Add **Edit Fields** node to assign `rss_feeds` from constants. Connect `Configure Workflow Args` to it.

3. **Split Search Themes:**  
   - Add a **SplitOut** node named `Split Out` on field `search_themes`. Connect from `Configure Workflow Args (Prep)`.

4. **Search Queries on Platforms:**  
   - Add **DuckDuckGo Search** node named `DuckDuckGo` with operation `searchNews`, query from `search_themes`, max results 100, Safe Search set to strict. Connect from `Split Out`.  
   - Add **Bluesky Search** node named `Bluesky Search` with query from `search_themes`, limit 10, Bluesky API credentials required. Connect from `Split Out`.  
   - Add **SerpAPI** node named `Google_news search` configured for Google News search with query from `search_themes`, SerpAPI credentials required.  
   - Add **Reddit** node `Get list of Subreddits` to get subreddits matching `search_themes`, limit 1. Connect from `Split Out`.  
   - Add **Set** node `Prepare Reddit RSS` to build RSS URL in `reddit_rss` field from subreddit name. Connect from `Get list of Subreddits`.  
   - Add **RSS Feed Read** node `RSS Read` to read subreddit RSS feed from `reddit_rss`. Connect from `Prepare Reddit RSS`.  
   - Add **HTTP Request** node named `HTTP Request` to search GitHub repositories with query `awesome links for <search_themes>`. Pagination max 1 page, 10 per page. Connect from `Split Out`.

5. **Process DuckDuckGo & Bluesky Results:**  
   - Add **Set** node `Filter Fields` to extract `title`, `pubDate` (from `date`), and `link` (from `url`) from DuckDuckGo results. Connect from `DuckDuckGo`.  
   - Add **Set** node `Edit Fields (Bluesky)` to build `title` as `"Bluesky Search - {author} - {external.title}"`, assign `pubDate` and `link` fields from Bluesky result. Connect from `Bluesky Search`.  
   - Add **Filter** node `Remove Elements with Empty Links` to remove items where `link` is empty from Bluesky results. Connect from `Edit Fields (Bluesky)`.  
   - Add **Merge** node `Merge1` merging DuckDuckGo filtered results and cleaned Bluesky results. Connect from `Filter Fields` and `Remove Elements with Empty Links`.  

6. **Process SerpAPI Results:**  
   - Add **SplitOut** node `Split Out News Results` splitting `news_results` array. Connect from `Google_news search`.  
   - Add **Filter** node `Remove Elements with Empty Links 2` to remove empty links in SerpAPI results. Connect from `Split Out News Results`.  
   - Add **Set** node `Filter Fields for Loop` to extract `title`, `pubDate`, and `link`. Connect from `Remove Elements with Empty Links 2`.  

7. **Process GitHub Repositories:**  
   - Add **SplitOut** node `Split Out Repo List` on `body.items` from GitHub search. Connect from `HTTP Request`.  
   - Add **Set** node `Add README field` to add `readme_url` by appending `/readme` to repo URL. Connect from `Split Out Repo List`.  
   - Add **HTTP Request** node `HTTP Request the READMEs` to fetch README content with timeout 10s and continue on error. Connect from `Add README field`.  
   - Add **HTTP Request** node `Grab READMEs` to fetch raw README content. Connect from `HTTP Request the READMEs`.  
   - Add **Filter** node `Remove Errors` to exclude errored README fetches. Connect from `Grab READMEs`.  
   - Add **Code** node `Grab Links from README` with regex to extract URLs from README text. Connect from `Remove Errors`.  
   - Add **SplitOut** node `Split Out Github Sources` on extracted `sources`. Connect from `Grab Links from README`.  
   - Add **Code** node `Get Base URL of Github Sources` to normalize URLs to base URLs. Connect from `Split Out Github Sources`.  
   - Add **Set** node `Prepare Github Source Results` to set `title`, `pubDate` (today), and `link` (baseUrl). Connect from `Get Base URL of Github Sources`.  

8. **Process Reddit Sources:**  
   - Add **Code** node `Grab Links from Threads` extracting URLs from Reddit post content with regex. Connect from `RSS Read`.  
   - Add **SplitOut** node `Split Out Sources` on `sources`. Connect from `Grab Links from Threads`.  
   - Add **Code** node `Get Base URL of Reddit Sources` to normalize to base URLs. Connect from `Split Out Sources`.  
   - Add **RemoveDuplicates** node `Remove Duplicates Reddit Sources` to deduplicate by baseUrl. Connect from `Get Base URL of Reddit Sources`.  
   - Add **Set** node `Prepare Reddit Source Results` to format Reddit sources with `title`, `pubDate` (today), and `link`. Connect from `Remove Duplicates Reddit Sources`.  

9. **Merge and Clean All Results:**  
   - Add **Merge** node `Merge` to combine GitHub, Reddit, and `Merge1` (DuckDuckGo + Bluesky) results and SerpAPI results (connected via `Loop Over Items (Search) 2`).  
   - Add **Filter** node `Remove Empty` to remove empty or invalid entries.  
   - Add **RemoveDuplicates** node `Remove Duplicates` to deduplicate results.  
   - Add **Code** node `Get Base URL` to normalize URLs.  
   - Add **Filter** node `Remove Existing Content` to exclude known sources (`existing_sources`).  
   - Add **Set** node `Leave Remaining Sources` to pass filtered sources.  
   - Add **RemoveDuplicates** node `Remove Duplicates Urls` to deduplicate again.  
   - Add **Filter** node `Remove Ignored Sources` to exclude unwanted sources (`sources_to_ignore`).  

10. **Content Fetch and Relevance Evaluation:**  
    - Add **Limit** node `Limit` to limit number of sources processed (default max 36).  
    - Add **HTTP Request** node `HTTP Request to get URL Content` to GET website content with 2.1s timeout, continue on error.  
    - Add **Code** node `Code (Limiting number of Chars)` to truncate content to 1500 characters.  
    - Add **Mistral Cloud Chat Model** node with valid credentials for LLM processing.  
    - Add **Information Extractor** node with schema for relevance, update recency, and accessibility extraction, connected to Mistral node.  
    - Add **Filter** node `Remove Errors` to exclude errored items post extraction.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is **not a news curator** but a source discovery tool aiming to find relevant websites rather than news articles. For news curation, see: [Multi-Source News Curator Workflow](https://n8n.io/workflows/6157-multi-source-news-curator-with-mistral-ai-analysis-summaries-and-custom-channels/) | Sticky Note12                                                                                       |
| Recommended to install custom nodes: SerpApi, Bluesky, DuckDuckGo Search, and optionally Global Variables Node for easier configuration.                                                                                              | Sticky Note12                                                                                       |
| GitHub searches do not require credentials but recommended to avoid strict rate limits. Bluesky and SerpAPI require credentials.                                                                                                      | Sticky Note12 and Sticky Note                                                                      |
| The workflow is designed for n8n version 1.105.3 and specific versions of custom nodes: serapi 0.1.6, bluesky-enhanced 1.6.0, duckduckgo-search 30.0.4, globals 1.1.0. Running on Linux via Podman 4.3.1.                                  | Sticky Note32                                                                                       |
| The workflow includes extensive filtering steps to avoid duplicates, empty links, and known or undesired sources, improving data quality.                                                                                              | Sticky Note1                                                                                       |
| The final evaluation uses Mistral LLM for content relevance extraction with a custom JSON schema.                                                                                                                                    | Sticky Note2                                                                                       |
| Potential limitations include API rate limits, network timeouts, website blocking GET requests, and LLM misclassification of relevance.                                                                                                | Sticky Note2                                                                                       |
| The workflow includes example JSON for input variables for quick setup.                                                                                                                                                               | Sticky Note31                                                                                      |
| Future improvements might include monitoring newly registered websites and advanced source discovery techniques.                                                                                                                     | Sticky Note7                                                                                       |
| Hybroht branding and contact: https://hybroht.com, contact@hybroht.com                                                                                                                                                                | Sticky Note12 and Sticky Note30 (logo image link)                                                 |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.