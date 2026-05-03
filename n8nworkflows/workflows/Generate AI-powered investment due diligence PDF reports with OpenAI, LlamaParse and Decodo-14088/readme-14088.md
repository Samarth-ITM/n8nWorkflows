Generate AI-powered investment due diligence PDF reports with OpenAI, LlamaParse and Decodo

https://n8nworkflows.xyz/workflows/generate-ai-powered-investment-due-diligence-pdf-reports-with-openai--llamaparse-and-decodo-14088


# Generate AI-powered investment due diligence PDF reports with OpenAI, LlamaParse and Decodo

# 1. Workflow Overview

This workflow receives one or more uploaded deal documents, parses and embeds their content into Pinecone, enriches the deal with external website evidence discovered via Decodo, runs an OpenAI-powered due diligence analysis over the combined evidence, generates a styled PDF report, uploads that PDF to S3, and returns a public URL plus structured analysis data via webhook response.

Typical use cases:
- Automated first-pass M&A screening
- AI-assisted investment memo preparation
- Due diligence report generation from CIMs, decks, and supporting files
- Knowledge-base creation for a specific deal namespace in Pinecone

## 1.1 Input Reception and Cache Check
The workflow starts with a webhook that accepts multipart file uploads. It splits uploaded binaries into one item per file, computes a shared `dealId`, and checks whether the corresponding Pinecone namespace already exists so it can skip document ingestion when possible.

## 1.2 Document Parsing and Vector Ingestion
If the namespace is not already populated, each file is uploaded to LlamaParse, polled until parsing completes, normalized into plain text, converted into embeddings, and stored in Pinecone under the deal namespace.

## 1.3 Analysis Context Preparation
Whether coming from a cache hit or a fresh ingestion, the workflow standardizes a common analysis context centered on `dealId`, processing counts, cache state, and timestamps.

## 1.4 Company Discovery and Official Website Validation
The workflow infers a probable company name from uploaded filenames, runs a Decodo Google search, builds domain candidates, verifies them by scraping candidate websites, scores them, and selects the most likely canonical domain.

## 1.5 External Evidence Collection and Indexing
If a domain is acceptable, the workflow scrapes one or more site pages, classifies evidence into profile vs. commercial/risk content, measures evidence coverage, and upserts the external evidence into the same Pinecone namespace as the parsed deal documents.

## 1.6 AI Due Diligence Analysis
The workflow prepares an enriched AI input summary, then runs an OpenAI agent that uses Pinecone as a retrieval tool. The prompt forces multiple retrieval queries and structured JSON output covering company profile, financials, risks, customer concentration, and investment thesis.

## 1.7 Report Rendering, PDF Delivery, and API Response
The AI output is mapped into report fields, rendered as HTML, converted to PDF with Puppeteer, uploaded to S3, transformed into a public URL, merged with the analysis payload, and returned through the webhook response node.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Cache Check

**Overview:**  
This block accepts uploaded files through a webhook, creates a shared deal identifier, and checks Pinecone for an existing namespace. If vectors already exist for the deal, the workflow skips parsing and uses the cached namespace.

**Nodes Involved:**  
- Receive Upload Request
- Split Uploaded Files + Build Deal ID
- Get Pinecone Index Stats
- Check Deal Namespace Cache
- Cache Hit?

### Receive Upload Request
- **Type and technical role:** `n8n-nodes-base.webhook`; main entry point receiving HTTP POST uploads.
- **Configuration choices:**  
  - Method: `POST`
  - Response mode: `responseNode`, meaning the workflow replies later via `Respond to Webhook`
  - Error mode: `continueRegularOutput`
- **Key expressions or variables used:** none internally; downstream nodes consume `body` and `binary`.
- **Input and output connections:**  
  - No input node
  - Output to `Split Uploaded Files + Build Deal ID`
- **Version-specific requirements:** typeVersion `2.1`
- **Edge cases or potential failure types:**  
  - Missing multipart binary data
  - Incorrect content type
  - Caller timeout if downstream processing is slow
- **Sub-workflow reference:** none

### Split Uploaded Files + Build Deal ID
- **Type and technical role:** `Code`; transforms webhook payload into one item per uploaded file and computes a stable `dealId`.
- **Configuration choices:**  
  - Parses `body.filenames` if present
  - Falls back to binary metadata if parsing body fails
  - Builds `dealId` from sorted filenames, base64-encodes, strips non-alphanumeric chars, truncates to 20 chars
  - Emits each binary as `binary.data`
- **Key expressions or variables used:**  
  - `item.json.body`
  - `item.binary`
  - `binaryKeys.filter(k => k.startsWith('data'))`
- **Input and output connections:**  
  - Input from `Receive Upload Request`
  - Output to `Get Pinecone Index Stats`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Throws `No binary files found in webhook request`
  - `body.filenames` malformed JSON
  - Potential `dealId` collisions if different filename sets normalize similarly
- **Sub-workflow reference:** none

### Get Pinecone Index Stats
- **Type and technical role:** `HTTP Request`; calls Pinecone index stats API to inspect namespaces.
- **Configuration choices:**  
  - POST to `/describe_index_stats`
  - Uses predefined Pinecone API credentials
  - Sends empty JSON body
  - Retries enabled with 2-second wait
  - `onError: continueRegularOutput`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input from `Split Uploaded Files + Build Deal ID`
  - Output to `Check Deal Namespace Cache`
- **Version-specific requirements:** typeVersion `4.3`
- **Edge cases or potential failure types:**  
  - Auth failure
  - Wrong Pinecone base URL
  - Namespace stats unavailable
  - Rate limiting
- **Sub-workflow reference:** none

### Check Deal Namespace Cache
- **Type and technical role:** `Code`; compares computed `dealId` with Pinecone namespace stats and determines cache hit/miss.
- **Configuration choices:**  
  - Reads original split items from `Split Uploaded Files + Build Deal ID`
  - If stats unavailable, marks cache status as `unknown`
  - If namespace exists and `vectorCount > 0`, emits a single cache-hit item
  - Otherwise re-emits original file items with `cacheHit: false`
- **Key expressions or variables used:**  
  - `$('Split Uploaded Files + Build Deal ID').all()`
  - `stats.namespaces[dealId].vectorCount`
- **Input and output connections:**  
  - Input from `Get Pinecone Index Stats`
  - Output to `Cache Hit?`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Expression access fails if upstream structure changes
  - Namespace stats schema differs from expected
- **Sub-workflow reference:** none

### Cache Hit?
- **Type and technical role:** `If`; branches based on whether namespace already contains vectors.
- **Configuration choices:**  
  - Checks `{{$json.cacheHit}} === true`
- **Key expressions or variables used:** `={{ $json.cacheHit }}`
- **Input and output connections:**  
  - Input from `Check Deal Namespace Cache`
  - True output to `Prepare Analysis Context`
  - False output to `Iterate Files for Parsing`
- **Version-specific requirements:** typeVersion `2.3`
- **Edge cases or potential failure types:**  
  - Non-boolean values may fail strict comparison
- **Sub-workflow reference:** none

---

## 2.2 Document Parsing and Vector Ingestion

**Overview:**  
This block ingests files only when cache is missed. Each file is uploaded to LlamaParse, polled until the job succeeds, converted into normalized text, embedded with OpenAI embeddings, and inserted into Pinecone.

**Nodes Involved:**  
- Iterate Files for Parsing
- Upload File to LlamaParse
- Check LlamaParse Job Status
- Is Parsing Job Complete?
- Wait 10s Before Recheck
- Retrieve Parsed Content
- Normalize Parsed Text Payload
- Generate Embeddings (Ingest)
- Prepare Parsed Text Document
- Upsert Chunks to Pinecone
- Collect Ingested Deal IDs

### Iterate Files for Parsing
- **Type and technical role:** `Split In Batches`; iterates uploaded file items one at a time.
- **Configuration choices:** default batch behavior
- **Key expressions or variables used:** downstream nodes reference `$('Iterate Files for Parsing').item.json`
- **Input and output connections:**  
  - Input from `Cache Hit?` false branch
  - Main loop output to `Upload File to LlamaParse`
  - Completion path to `Collect Ingested Deal IDs`
- **Version-specific requirements:** typeVersion `3`
- **Edge cases or potential failure types:**  
  - Loop behavior depends on downstream returning control to this node
- **Sub-workflow reference:** none

### Upload File to LlamaParse
- **Type and technical role:** `HTTP Request`; uploads a file binary to LlamaParse.
- **Configuration choices:**  
  - POST multipart form-data
  - Sends binary field `data` as `file`
  - Uses HTTP header auth credential named `llamaindex`
  - Accept header set to `application/json`
- **Key expressions or variables used:** multipart binary input field `data`
- **Input and output connections:**  
  - Input from `Iterate Files for Parsing`
  - Output to `Check LlamaParse Job Status`
- **Version-specific requirements:** typeVersion `4.2`
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Unsupported file format
  - File too large
  - Upload timeout
- **Sub-workflow reference:** none

### Check LlamaParse Job Status
- **Type and technical role:** `HTTP Request`; polls LlamaParse job status by `id`.
- **Configuration choices:**  
  - GET-like request to `/parsing/job/{{ $json.id }}`
  - Header auth with `accept: application/json`
- **Key expressions or variables used:** `={{ $json.id }}`
- **Input and output connections:**  
  - Input from `Upload File to LlamaParse` and `Wait 10s Before Recheck`
  - Output to `Is Parsing Job Complete?`
- **Version-specific requirements:** typeVersion `4.2`
- **Edge cases or potential failure types:**  
  - Job ID missing
  - LlamaParse processing delays
  - Temporary service error
- **Sub-workflow reference:** none

### Is Parsing Job Complete?
- **Type and technical role:** `If`; determines whether parsing status is `SUCCESS`.
- **Configuration choices:**  
  - Strict string equality on `status`
- **Key expressions or variables used:** `={{ $json.status }}`
- **Input and output connections:**  
  - Input from `Check LlamaParse Job Status`
  - True output to `Retrieve Parsed Content`
  - False output to `Wait 10s Before Recheck`
- **Version-specific requirements:** typeVersion `2.2`
- **Edge cases or potential failure types:**  
  - Infinite polling if job never succeeds and does not hard-fail
  - No explicit handling for terminal failure statuses
- **Sub-workflow reference:** none

### Wait 10s Before Recheck
- **Type and technical role:** `Wait`; pauses before polling again.
- **Configuration choices:** wait amount `10` seconds
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input from `Is Parsing Job Complete?` false branch
  - Output back to `Check LlamaParse Job Status`
- **Version-specific requirements:** typeVersion `1.1`
- **Edge cases or potential failure types:**  
  - Long-running workflows
  - Execution retention considerations
- **Sub-workflow reference:** none

### Retrieve Parsed Content
- **Type and technical role:** `HTTP Request`; fetches completed markdown result from LlamaParse.
- **Configuration choices:**  
  - Calls `/parsing/job/{{ $json.id }}/result/markdown`
  - Header auth with `accept: application/json`
- **Key expressions or variables used:** `={{ $json.id }}`
- **Input and output connections:**  
  - Input from `Is Parsing Job Complete?` true branch
  - Output to `Normalize Parsed Text Payload`
- **Version-specific requirements:** typeVersion `4.2`
- **Edge cases or potential failure types:**  
  - Result unavailable despite success status
  - Markdown payload missing expected field
- **Sub-workflow reference:** none

### Normalize Parsed Text Payload
- **Type and technical role:** `Code`; standardizes parsed result into a simple JSON shape.
- **Configuration choices:**  
  - Pulls file metadata from current `Iterate Files for Parsing` item
  - Uses `markdown` or fallback `text`
- **Key expressions or variables used:**  
  - `$('Iterate Files for Parsing').item.json`
  - `$json.markdown || $json.text || ''`
- **Input and output connections:**  
  - Input from `Retrieve Parsed Content`
  - Output to `Upsert Chunks to Pinecone`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Empty parsed content
  - Lost loop context if node naming changes
- **Sub-workflow reference:** none

### Generate Embeddings (Ingest)
- **Type and technical role:** `OpenAI Embeddings`; provides embedding model for vector insertion.
- **Configuration choices:** default options; uses OpenAI credential `Sumopod`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - AI embedding connection to `Upsert Chunks to Pinecone`
- **Version-specific requirements:** typeVersion `1.2`
- **Edge cases or potential failure types:**  
  - OpenAI auth errors
  - Token or size limitations on chunked content
- **Sub-workflow reference:** none

### Prepare Parsed Text Document
- **Type and technical role:** `Document Default Data Loader`; turns normalized text into a document with metadata.
- **Configuration choices:**  
  - `jsonData` is `{{$json.parsedText || ""}}`
  - Metadata includes `deal_id`, `source_file`, `file_type`, `timestamp`
- **Key expressions or variables used:**  
  - `={{ $json.dealId }}`
  - `={{ $json.sourceFile }}`
  - `={{ $json.fileType }}`
  - `={{ $now.toUTC() }}`
- **Input and output connections:**  
  - AI document connection to `Upsert Chunks to Pinecone`
- **Version-specific requirements:** typeVersion `1.1`
- **Edge cases or potential failure types:**  
  - Empty string document may still be inserted unless filtered elsewhere
- **Sub-workflow reference:** none

### Upsert Chunks to Pinecone
- **Type and technical role:** `Vector Store Pinecone`; inserts embedded parsed documents into Pinecone.
- **Configuration choices:**  
  - Mode: `insert`
  - Namespace: `={{ $('Iterate Files for Parsing').item.json.dealId }}`
  - `clearNamespace: false`
  - Index: `poc`
- **Key expressions or variables used:** `={{ $('Iterate Files for Parsing').item.json.dealId }}`
- **Input and output connections:**  
  - Main input from `Normalize Parsed Text Payload`
  - AI embedding input from `Generate Embeddings (Ingest)`
  - AI document input from `Prepare Parsed Text Document`
  - Output back to `Iterate Files for Parsing`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**  
  - Pinecone dimension mismatch with embedding model
  - Namespace write failure
  - Credential/index misconfiguration
- **Sub-workflow reference:** none

### Collect Ingested Deal IDs
- **Type and technical role:** `Aggregate`; collects `metadata.deal_id` values after ingestion loop completes.
- **Configuration choices:** aggregates field `metadata.deal_id`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input from `Iterate Files for Parsing`
  - Output to `Prepare Analysis Context`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases or potential failure types:**  
  - Aggregated structure differs if upstream Pinecone node response changes
- **Sub-workflow reference:** none

---

## 2.3 Analysis Context Preparation

**Overview:**  
This block normalizes the starting context for downstream company discovery and AI analysis, regardless of whether the flow used cached vectors or freshly ingested files.

**Nodes Involved:**  
- Prepare Analysis Context

### Prepare Analysis Context
- **Type and technical role:** `Code`; unifies cache-hit and fresh-ingest paths into a single payload.
- **Configuration choices:**  
  - If `cacheHit === true`, uses upstream `dealId`
  - Otherwise extracts first `deal_id` from aggregated array
  - Emits `filesProcessed`, `fromCache`, and timestamp
- **Key expressions or variables used:**  
  - `$input.all()`
  - `items[0]?.json?.cacheHit`
  - `items[0]?.json?.deal_id`
- **Input and output connections:**  
  - Inputs from `Cache Hit?` true branch and `Collect Ingested Deal IDs`
  - Output to `Derive Company Seed`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Aggregated `deal_id` may be missing
  - `vectorCount` reused as `filesProcessed` on cache hit, which is semantically not file count
- **Sub-workflow reference:** none

---

## 2.4 Company Discovery and Official Website Validation

**Overview:**  
This block infers the likely company name from uploaded filenames, searches for its website, scores likely domains, verifies candidates via page scraping, and chooses a canonical website for enrichment.

**Nodes Involved:**  
- Derive Company Seed
- Decodo Search Official Site
- Build Domain Candidates
- Iterate Domain Candidates
- Decodo Verify Official Domain
- Score Domain Match
- Collect Domain Scores
- Select Canonical Domain
- Canonical Domain Acceptable?

### Derive Company Seed
- **Type and technical role:** `Code`; extracts company-name clues from filenames and derives search metadata.
- **Configuration choices:**  
  - Builds country normalization map
  - Reads original uploaded filenames from `Split Uploaded Files + Build Deal ID`
  - Removes generic words and date/version noise
  - Generates `companyNameGuess`, `searchQuery`, `fallbackSearchQuery`, locale and country code
- **Key expressions or variables used:**  
  - `$('Split Uploaded Files + Build Deal ID').all()`
  - `input.location || input.country || input.hqCountry`
- **Input and output connections:**  
  - Input from `Prepare Analysis Context`
  - Output to `Decodo Search Official Site`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Filename-only inference can be wrong
  - Generic filenames lead to low-confidence guesses
  - Non-Latin filenames may degrade token extraction
- **Sub-workflow reference:** none

### Decodo Search Official Site
- **Type and technical role:** `Decodo`; runs Google search for the inferred company.
- **Configuration choices:**  
  - Operation: `google_search`
  - Geo: `={{ $json.searchCountryCode || "us" }}`
  - Locale: `={{ $json.searchLocale || "en-US" }}`
  - Query: `searchQuery` or fallback
- **Key expressions or variables used:**  
  - `={{ $json.searchQuery || $json.fallbackSearchQuery || $json.companyNameGuess }}`
- **Input and output connections:**  
  - Input from `Derive Company Seed`
  - Output to `Build Domain Candidates`
- **Version-specific requirements:** Decodo node typeVersion `1`
- **Edge cases or potential failure types:**  
  - Search API quota or network issues
  - SERP layout changes
  - Poor query quality from noisy filenames
- **Sub-workflow reference:** none

### Build Domain Candidates
- **Type and technical role:** `Code`; creates up to 3 website candidates from document-mentioned domains, search results, and guessed domains.
- **Configuration choices:**  
  - Blocks social/media/directory domains
  - Scores search hits using title/snippet/domain token matching
  - Adds guessed TLDs based on country
- **Key expressions or variables used:**  
  - `$('Derive Company Seed').first().json`
  - `payload.results?.[0]?.content?.results?.results?.organic`
- **Input and output connections:**  
  - Input from `Decodo Search Official Site`
  - Output to `Iterate Domain Candidates`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - SERP schema may differ
  - Candidate filtering may reject valid domains
  - Guessed domains may be false positives
- **Sub-workflow reference:** none

### Iterate Domain Candidates
- **Type and technical role:** `Split In Batches`; loops over candidate domains.
- **Configuration choices:** default iterative behavior
- **Key expressions or variables used:** downstream nodes read `$('Iterate Domain Candidates').item.json`
- **Input and output connections:**  
  - Input from `Build Domain Candidates`
  - Loop body to `Decodo Verify Official Domain`
  - Completion to `Collect Domain Scores`
- **Version-specific requirements:** typeVersion `3`
- **Edge cases or potential failure types:** loop control dependent on downstream return path
- **Sub-workflow reference:** none

### Decodo Verify Official Domain
- **Type and technical role:** `Decodo`; fetches candidate page content for verification.
- **Configuration choices:**  
  - URL from candidate
  - `headless: false`
  - `markdown: true`
  - Retries: 3
  - `onError: continueRegularOutput`
- **Key expressions or variables used:**  
  - `={{ $json.url }}`
  - `={{ $json.hqCountry || 'United States' }}`
- **Input and output connections:**  
  - Input from `Iterate Domain Candidates`
  - Output to `Score Domain Match`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases or potential failure types:**  
  - Candidate site blocks scraping
  - DNS or SSL errors
  - Redirects to unrelated content
- **Sub-workflow reference:** none

### Score Domain Match
- **Type and technical role:** `Code`; evaluates scraped candidate content and computes a final verification score.
- **Configuration choices:**  
  - Rewards exact/partial company-name matches, country matches, corporate-page signals, and sufficient content
  - Penalizes errors, thin content, social/listing sites, parked domains, 404-like patterns
  - Classifies candidates into `verified`, `probable`, `reject`
- **Key expressions or variables used:**  
  - `$('Derive Company Seed').first().json`
  - `$('Iterate Domain Candidates').item.json`
- **Input and output connections:**  
  - Input from `Decodo Verify Official Domain`
  - Output back to `Iterate Domain Candidates`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Heuristic scoring can accept wrong domains
  - Missing content lowers confidence
- **Sub-workflow reference:** none

### Collect Domain Scores
- **Type and technical role:** `Aggregate`; collects all scored candidates into one array.
- **Configuration choices:**  
  - Aggregate all item data into `domainCandidates`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input from `Iterate Domain Candidates`
  - Output to `Select Canonical Domain`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases or potential failure types:**  
  - Empty candidate set
- **Sub-workflow reference:** none

### Select Canonical Domain
- **Type and technical role:** `Code`; selects the best-scoring domain and emits one or more enrichment target URLs.
- **Configuration choices:**  
  - Sorts candidates by `finalScore`
  - If none exist, emits reject result
  - Verified domains yield `''` and `/about` paths
  - Probable domains yield only root path
- **Key expressions or variables used:**  
  - `$json.domainCandidates`
  - `$('Derive Company Seed').first().json`
- **Input and output connections:**  
  - Input from `Collect Domain Scores`
  - Output to `Canonical Domain Acceptable?`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Best candidate may still be wrong
  - Root/about assumptions may not fit all sites
- **Sub-workflow reference:** none

### Canonical Domain Acceptable?
- **Type and technical role:** `If`; gates external evidence enrichment based on canonical status.
- **Configuration choices:**  
  - Continues only when `canonicalStatus != reject`
- **Key expressions or variables used:** `={{ $json.canonicalStatus }}`
- **Input and output connections:**  
  - Input from `Select Canonical Domain`
  - True output to `Iterate Enrichment URLs`
  - False output to `Prepare AI Input with Evidence`
- **Version-specific requirements:** typeVersion `2.2`
- **Edge cases or potential failure types:**  
  - `probable` domains still may introduce noisy evidence
- **Sub-workflow reference:** none

---

## 2.5 External Evidence Collection and Indexing

**Overview:**  
This block scrapes selected website pages, normalizes them into evidence records, computes evidence quality metrics, and, if sufficient evidence exists, inserts it into Pinecone under the same deal namespace.

**Nodes Involved:**  
- Iterate Enrichment URLs
- Profile Page?
- Decodo Scrape Company Profile
- Decodo Scrape Commercial and Risk
- Normalize External Evidence
- Collect External Evidence
- Evidence Coverage Metrics
- Has External Evidence?
- Generate Embeddings (External)
- Prepare External Evidence Document
- Upsert External Evidence to Pinecone

### Iterate Enrichment URLs
- **Type and technical role:** `Split In Batches`; iterates over emitted target URLs for enrichment.
- **Configuration choices:** default
- **Key expressions or variables used:** downstream nodes use `$('Iterate Enrichment URLs').item.json`
- **Input and output connections:**  
  - Input from `Canonical Domain Acceptable?`
  - Loop body to `Profile Page?`
  - Completion to `Collect External Evidence`
- **Version-specific requirements:** typeVersion `3`
- **Edge cases or potential failure types:** loop control depends on downstream return
- **Sub-workflow reference:** none

### Profile Page?
- **Type and technical role:** `If`; routes `/about` URLs to the profile scraper and other URLs to the commercial/risk scraper.
- **Configuration choices:**  
  - Case-insensitive contains check on `targetUrl` for `about`
- **Key expressions or variables used:** `={{ $json.targetUrl }}`
- **Input and output connections:**  
  - Input from `Iterate Enrichment URLs`
  - True to `Decodo Scrape Company Profile`
  - False to `Decodo Scrape Commercial and Risk`
- **Version-specific requirements:** typeVersion `2.2`
- **Edge cases or potential failure types:**  
  - Non-standard profile pages may not include `/about`
- **Sub-workflow reference:** none

### Decodo Scrape Company Profile
- **Type and technical role:** `Decodo`; scrapes profile-oriented page content.
- **Configuration choices:**  
  - URL from `targetUrl`
  - `markdown: true`
  - Retries enabled
  - `onError: continueRegularOutput`
- **Key expressions or variables used:**  
  - `={{ $json.targetUrl }}`
  - `={{ $json.hqCountry || 'United States' }}`
- **Input and output connections:**  
  - Input from `Profile Page?` true branch
  - Output to `Normalize External Evidence`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases or potential failure types:**  
  - Anti-bot blocks
  - Empty page body
- **Sub-workflow reference:** none

### Decodo Scrape Commercial and Risk
- **Type and technical role:** `Decodo`; scrapes root or business pages for commercial and risk content.
- **Configuration choices:** similar to profile scraper
- **Key expressions or variables used:** same pattern as above
- **Input and output connections:**  
  - Input from `Profile Page?` false branch
  - Output to `Normalize External Evidence`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases or potential failure types:** same as above
- **Sub-workflow reference:** none

### Normalize External Evidence
- **Type and technical role:** `Code`; turns raw scrape payloads into standardized evidence records.
- **Configuration choices:**  
  - Pulls source metadata from `Iterate Enrichment URLs`
  - Detects `evidenceType` based on URL pattern
  - Calculates confidence from canonical status and content length
  - Stores excerpt in `claim` and full text in `markdown`
- **Key expressions or variables used:**  
  - `$('Iterate Enrichment URLs').item.json`
  - `payload.results?.[0]?.content || payload.content || ...`
- **Input and output connections:**  
  - Inputs from both Decodo scrape nodes
  - Output back to `Iterate Enrichment URLs`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Content structure may vary by Decodo response
  - Very large pages truncated to 4000 chars for `claim`
- **Sub-workflow reference:** none

### Collect External Evidence
- **Type and technical role:** `Aggregate`; gathers all normalized evidence items.
- **Configuration choices:** aggregates all item data into `externalEvidence`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input from `Iterate Enrichment URLs`
  - Output to `Evidence Coverage Metrics`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases or potential failure types:** empty evidence list
- **Sub-workflow reference:** none

### Evidence Coverage Metrics
- **Type and technical role:** `Code`; computes evidence success/failure counts, coverage ratio, confidence summary, and gate flag.
- **Configuration choices:**  
  - Reject canonical domain disables external evidence usage
  - `evidenceGatePass` requires at least 1 successful scrape
  - Counts profile vs risk/commercial evidence
- **Key expressions or variables used:**  
  - `$json.externalEvidence`
  - `$('Select Canonical Domain').first().json`
  - `$('Prepare Analysis Context').first().json.dealId`
- **Input and output connections:**  
  - Input from `Collect External Evidence`
  - Output to `Has External Evidence?`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Mixed success/failure responses
  - Missing canonical context
- **Sub-workflow reference:** none

### Has External Evidence?
- **Type and technical role:** `If`; decides whether to index external evidence.
- **Configuration choices:**  
  - Checks `evidenceGatePass === true`
- **Key expressions or variables used:** `={{ $json.evidenceGatePass }}`
- **Input and output connections:**  
  - Input from `Evidence Coverage Metrics`
  - True to `Upsert External Evidence to Pinecone`
  - False to `Prepare AI Input with Evidence`
- **Version-specific requirements:** typeVersion `2.3`
- **Edge cases or potential failure types:** strict boolean handling
- **Sub-workflow reference:** none

### Generate Embeddings (External)
- **Type and technical role:** `OpenAI Embeddings`; embedding provider for web evidence insertion.
- **Configuration choices:** default options, OpenAI credential
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - AI embedding to `Upsert External Evidence to Pinecone`
- **Version-specific requirements:** typeVersion `1.2`
- **Edge cases or potential failure types:** same as ingestion embeddings
- **Sub-workflow reference:** none

### Prepare External Evidence Document
- **Type and technical role:** `Document Default Data Loader`; converts combined external evidence into one document.
- **Configuration choices:**  
  - Concatenates all external evidence items with separators
  - Metadata includes `deal_id`, `source_type`, evidence counts, coverage, timestamp
- **Key expressions or variables used:**  
  - `={{ $json.dealId }}`
  - `={{ ($json.externalEvidence || []).map(...).join(...) }}`
- **Input and output connections:**  
  - AI document to `Upsert External Evidence to Pinecone`
- **Version-specific requirements:** typeVersion `1.1`
- **Edge cases or potential failure types:**  
  - Large combined evidence may create oversized document chunks
- **Sub-workflow reference:** none

### Upsert External Evidence to Pinecone
- **Type and technical role:** `Vector Store Pinecone`; inserts external evidence into the deal namespace.
- **Configuration choices:**  
  - Mode: `insert`
  - Namespace: `={{ $json.dealId }}`
  - Index: `poc`
  - `clearNamespace: false`
- **Key expressions or variables used:** `={{ $json.dealId }}`
- **Input and output connections:**  
  - Main input from `Has External Evidence?` true branch
  - AI embedding from `Generate Embeddings (External)`
  - AI document from `Prepare External Evidence Document`
  - Output to `Prepare AI Input with Evidence`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**  
  - Namespace pollution if wrong `dealId`
  - Duplicate inserts across reruns
- **Sub-workflow reference:** none

---

## 2.6 AI Due Diligence Analysis

**Overview:**  
This block prepares the final retrieval context, invokes an OpenAI agent with Pinecone as a tool, and enforces structured JSON output using a parser schema.

**Nodes Involved:**  
- Prepare AI Input with Evidence
- OpenAI Chat Model (5-mini)
- Generate Embeddings (Retrieval)
- Pinecone Vector Store
- Parse Structured Analysis JSON
- Run Due Diligence AI Analysis

### Prepare AI Input with Evidence
- **Type and technical role:** `Code`; combines base deal context with evidence metrics and canonical domain metadata.
- **Configuration choices:**  
  - Starts from `Prepare Analysis Context`
  - Attempts to merge `Evidence Coverage Metrics`
  - If canonical status is reject, zeroes external evidence metrics
  - Emits `externalEvidenceSummary`
- **Key expressions or variables used:**  
  - `$('Prepare Analysis Context').first().json`
  - `$('Evidence Coverage Metrics').first().json`
  - `$('Select Canonical Domain').first().json`
- **Input and output connections:**  
  - Inputs from `Canonical Domain Acceptable?` false branch, `Has External Evidence?` false branch, and `Upsert External Evidence to Pinecone`
  - Output to `Run Due Diligence AI Analysis`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Relies on `.first()` access patterns across branches
  - Missing canonical node output could throw unless caught
- **Sub-workflow reference:** none

### OpenAI Chat Model (5-mini)
- **Type and technical role:** `LM Chat OpenAI`; language model backend for the agent.
- **Configuration choices:**  
  - Model: `gpt-5-mini`
  - Built-in tools disabled
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - AI language model connection to `Run Due Diligence AI Analysis`
- **Version-specific requirements:** typeVersion `1.3`; requires model availability in connected OpenAI account
- **Edge cases or potential failure types:**  
  - Model availability or access restrictions
  - Rate limiting
- **Sub-workflow reference:** none

### Generate Embeddings (Retrieval)
- **Type and technical role:** `OpenAI Embeddings`; embedding model used by Pinecone retrieval tool.
- **Configuration choices:** default
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - AI embedding to `Pinecone Vector Store`
- **Version-specific requirements:** typeVersion `1.2`
- **Edge cases or potential failure types:** embedding auth/model mismatch
- **Sub-workflow reference:** none

### Pinecone Vector Store
- **Type and technical role:** `Vector Store Pinecone`; retrieval tool exposed to the AI agent.
- **Configuration choices:**  
  - Mode: `retrieve-as-tool`
  - Namespace: `={{ $json.dealId }}`
  - Index: `poc`
  - Tool description explicitly states it contains parsed docs plus validated external evidence
- **Key expressions or variables used:** `={{ $json.dealId }}`
- **Input and output connections:**  
  - Main input context inherited from agent input
  - AI embedding from `Generate Embeddings (Retrieval)`
  - AI tool output to `Run Due Diligence AI Analysis`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**  
  - Empty namespace
  - Retrieval returns low-quality chunks if ingestion was poor
- **Sub-workflow reference:** none

### Parse Structured Analysis JSON
- **Type and technical role:** `Structured Output Parser`; enforces JSON schema shape for AI output.
- **Configuration choices:**  
  - Example schema includes `company_profile`, `financials`, and `analysis`
- **Key expressions or variables used:** schema example only
- **Input and output connections:**  
  - AI output parser connection to `Run Due Diligence AI Analysis`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**  
  - If model emits invalid JSON, parsing can fail
  - Example schema does not fully enforce every output rule in prompt
- **Sub-workflow reference:** none

### Run Due Diligence AI Analysis
- **Type and technical role:** `LangChain Agent`; orchestrates LLM + tool retrieval to produce the final structured analysis.
- **Configuration choices:**  
  - Prompt mandates 6 Pinecone retrieval queries before answering
  - Requires inclusion of all years 2021–2025 if available
  - Forces missing strings to `"Not Available"` and missing numerics to `0`
  - Explicitly instructs use of external web evidence from Pinecone namespace
  - `hasOutputParser: true`
- **Key expressions or variables used:** large prompt text; no dynamic expression in prompt body
- **Input and output connections:**  
  - Main input from `Prepare AI Input with Evidence`
  - AI language model from `OpenAI Chat Model (5-mini)`
  - AI tool from `Pinecone Vector Store`
  - AI output parser from `Parse Structured Analysis JSON`
  - Main outputs to `Map Analysis to Report Fields` and ` Merge Analysis + Report URL`
- **Version-specific requirements:** typeVersion `3.1`
- **Edge cases or potential failure types:**  
  - Agent may not perfectly comply with “all 6 queries”
  - Output parser failure on malformed JSON
  - Hallucination risk if Pinecone retrieval quality is weak
- **Sub-workflow reference:** none

---

## 2.7 Report Rendering, PDF Delivery, and API Response

**Overview:**  
This block converts the AI result into presentation-friendly fields, renders a styled HTML report, converts it into PDF, uploads it to S3, creates a public URL, and returns a final JSON payload to the original webhook caller.

**Nodes Involved:**  
- Map Analysis to Report Fields
- Augment Report with Evidence
- Render DD Report HTML
- Render PDF from HTML
- Convert PDF Base64 to Binary File
- Prepare S3 File Metadata
- Upload Report PDF to S3
- Build Public Report URL
- Merge Analysis + Report URL
- Prepare API Response Payload
- Return API Response

### Map Analysis to Report Fields
- **Type and technical role:** `Code`; transforms structured analysis JSON into escaped and formatted HTML-ready fields.
- **Configuration choices:**  
  - Handles both `raw.output` and raw JSON forms
  - Converts numbers to locale strings
  - Builds revenue table rows and risk list HTML
  - Pulls `dealId` from `Prepare Analysis Context`
  - Uses `DateTime.now()` for report date
- **Key expressions or variables used:**  
  - `$('Prepare Analysis Context').first()`
  - `DateTime.now().toFormat(...)`
- **Input and output connections:**  
  - Input from `Run Due Diligence AI Analysis`
  - Output to `Augment Report with Evidence`
- **Version-specific requirements:** typeVersion `2`; assumes Luxon `DateTime` is available in n8n Code node runtime
- **Edge cases or potential failure types:**  
  - `DateTime` availability can vary by runtime
  - HTML escaping covers most text but final template still inserts generated HTML fragments directly
- **Sub-workflow reference:** none

### Augment Report with Evidence
- **Type and technical role:** `Code`; injects evidence coverage metrics into the report payload.
- **Configuration choices:**  
  - Adds `evidenceCoverage`, `externalSourcesCount`, `confidenceSummary`
  - Adds disclaimer if coverage is low
- **Key expressions or variables used:** `$('Prepare AI Input with Evidence').first().json`
- **Input and output connections:**  
  - Input from `Map Analysis to Report Fields`
  - Output to `Render DD Report HTML`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:** missing upstream evidence context
- **Sub-workflow reference:** none

### Render DD Report HTML
- **Type and technical role:** `HTML`; compiles a complete styled HTML report from mapped fields.
- **Configuration choices:**  
  - Full HTML/CSS template
  - Sections: company overview, financial summary, business model, investment thesis, risk analysis, footer
- **Key expressions or variables used:**  
  - `{{ $json.companyName }}`, `{{ $json.revenueTableRows }}`, etc.
- **Input and output connections:**  
  - Input from `Augment Report with Evidence`
  - Output to `Render PDF from HTML`
- **Version-specific requirements:** typeVersion `1.2`
- **Edge cases or potential failure types:**  
  - Large content may impact PDF rendering
  - Injected HTML fragments must already be safe
- **Sub-workflow reference:** none

### Render PDF from HTML
- **Type and technical role:** `Puppeteer`; loads HTML into a browser page and exports a PDF.
- **Configuration choices:**  
  - Custom script uses `$page.setContent(...)` with `networkidle0`
  - A4 format, background printing, margins, scale `0.95`
  - Returns `pdfBase64`
- **Key expressions or variables used:** `$json.html`
- **Input and output connections:**  
  - Input from `Render DD Report HTML`
  - Output to `Convert PDF Base64 to Binary File`
- **Version-specific requirements:** Puppeteer community node; requires browser runtime support
- **Edge cases or potential failure types:**  
  - Empty HTML throws explicit error
  - Browser launch failures
  - Timeout on heavy pages
- **Sub-workflow reference:** none

### Convert PDF Base64 to Binary File
- **Type and technical role:** `Convert to File`; converts base64 string to binary.
- **Configuration choices:**  
  - Operation: `toBinary`
  - Source property: `pdfBase64`
  - Filename: `={{ $('Map Analysis to Report Fields').item.json.companyName }}-Analysis.pdf`
- **Key expressions or variables used:** `={{ $('Map Analysis to Report Fields').item.json.companyName }}-Analysis.pdf`
- **Input and output connections:**  
  - Input from `Render PDF from HTML`
  - Output to `Prepare S3 File Metadata`
- **Version-specific requirements:** typeVersion `1.1`
- **Edge cases or potential failure types:**  
  - Invalid base64
  - Company name contains S3-unfriendly characters
- **Sub-workflow reference:** none

### Prepare S3 File Metadata
- **Type and technical role:** `Code`; generates a timestamped output filename and preserves binary.
- **Configuration choices:**  
  - Filename format: `${companyName}-assessment-${timestamp}.pdf`
- **Key expressions or variables used:** `$('Map Analysis to Report Fields').first().json.companyName`
- **Input and output connections:**  
  - Input from `Convert PDF Base64 to Binary File`
  - Output to `Upload Report PDF to S3`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Invalid characters in filename
  - Missing binary from upstream
- **Sub-workflow reference:** none

### Upload Report PDF to S3
- **Type and technical role:** `S3`; uploads the generated PDF file to object storage.
- **Configuration choices:**  
  - Operation: `upload`
  - Bucket: `poc`
  - Uses binary input file and dynamic filename
- **Key expressions or variables used:** `={{ $json.fileName }}`
- **Input and output connections:**  
  - Input from `Prepare S3 File Metadata`
  - Output to `Build Public Report URL`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases or potential failure types:**  
  - Invalid credentials
  - Bucket missing
  - ACL/public access not configured externally
- **Sub-workflow reference:** none

### Build Public Report URL
- **Type and technical role:** `Code`; constructs a public URL for the uploaded report.
- **Configuration choices:**  
  - Hardcoded base URL `https://poc.atlr.dev`
  - URL-encodes filename from `Prepare S3 File Metadata`
- **Key expressions or variables used:** `$('Prepare S3 File Metadata').first().json.fileName`
- **Input and output connections:**  
  - Input from `Upload Report PDF to S3`
  - Output to ` Merge Analysis + Report URL`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Base URL must match actual public file-serving setup
  - URL may not work if S3 bucket is private or proxied differently
- **Sub-workflow reference:** none

### Merge Analysis + Report URL
- **Type and technical role:** `Merge`; combines AI analysis branch and PDF URL branch.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by position
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input 0 from `Run Due Diligence AI Analysis`
  - Input 1 from `Build Public Report URL`
  - Output to `Prepare API Response Payload`
- **Version-specific requirements:** typeVersion `3.2`
- **Edge cases or potential failure types:**  
  - Position-based merge can misalign if one branch produces unexpected item counts
- **Sub-workflow reference:** none

### Prepare API Response Payload
- **Type and technical role:** `Code`; builds the final structured API response returned to the webhook caller.
- **Configuration choices:**  
  - Includes `success`, `responseStatus`, `dealId`, `fileName`, `publicUrl`
  - Repackages selected AI output fields
  - Adds evidence metadata and disclaimer
  - Returns `partial_success` if no trusted external evidence was used
- **Key expressions or variables used:**  
  - `$('Prepare AI Input with Evidence').first().json`
  - `$('Run Due Diligence AI Analysis').first().json`
- **Input and output connections:**  
  - Input from ` Merge Analysis + Report URL`
  - Output to `Return API Response`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases or potential failure types:**  
  - Some financial margins are omitted from final API response except EBITDA margin
  - Upstream malformed AI output may degrade payload
- **Sub-workflow reference:** none

### Return API Response
- **Type and technical role:** `Respond to Webhook`; sends the final response back to the caller.
- **Configuration choices:** default response behavior
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input from `Prepare API Response Payload`
  - Terminal node
- **Version-specific requirements:** typeVersion `1.5`
- **Edge cases or potential failure types:**  
  - If workflow exceeds webhook waiting limits, client may time out before response
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Upload Request | Webhook | Receives multipart uploaded deal files |  | Split Uploaded Files + Build Deal ID | ## Intake + Cache Check<br>Receives file uploads, splits them into per-file items, builds a deal ID, and checks if this deal was already ingested. |
| Split Uploaded Files + Build Deal ID | Code | Splits binaries into one item per file and computes shared dealId | Receive Upload Request | Get Pinecone Index Stats | ## Intake + Cache Check<br>Receives file uploads, splits them into per-file items, builds a deal ID, and checks if this deal was already ingested. |
| Get Pinecone Index Stats | HTTP Request | Queries Pinecone namespace stats | Split Uploaded Files + Build Deal ID | Check Deal Namespace Cache | ## Intake + Cache Check<br>Receives file uploads, splits them into per-file items, builds a deal ID, and checks if this deal was already ingested. |
| Check Deal Namespace Cache | Code | Detects cache hit/miss for deal namespace | Get Pinecone Index Stats | Cache Hit? | ## Intake + Cache Check<br>Receives file uploads, splits them into per-file items, builds a deal ID, and checks if this deal was already ingested. |
| Cache Hit? | If | Routes execution to cached analysis or fresh parsing | Check Deal Namespace Cache | Prepare Analysis Context; Iterate Files for Parsing | ## Intake + Cache Check<br>Receives file uploads, splits them into per-file items, builds a deal ID, and checks if this deal was already ingested. |
| Iterate Files for Parsing | Split In Batches | Loops through files for LlamaParse ingestion | Cache Hit?; Upsert Chunks to Pinecone | Collect Ingested Deal IDs; Upload File to LlamaParse | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Upload File to LlamaParse | HTTP Request | Uploads file binary to LlamaParse | Iterate Files for Parsing | Check LlamaParse Job Status | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Check LlamaParse Job Status | HTTP Request | Polls LlamaParse job status | Upload File to LlamaParse; Wait 10s Before Recheck | Is Parsing Job Complete? | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Is Parsing Job Complete? | If | Checks whether parsing finished successfully | Check LlamaParse Job Status | Retrieve Parsed Content; Wait 10s Before Recheck | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Wait 10s Before Recheck | Wait | Delays before polling LlamaParse again | Is Parsing Job Complete? | Check LlamaParse Job Status | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Retrieve Parsed Content | HTTP Request | Fetches final parsed markdown from LlamaParse | Is Parsing Job Complete? | Normalize Parsed Text Payload | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Normalize Parsed Text Payload | Code | Standardizes LlamaParse output with source metadata | Retrieve Parsed Content | Upsert Chunks to Pinecone | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Generate Embeddings (Ingest) | OpenAI Embeddings | Supplies embeddings for parsed document ingestion |  | Upsert Chunks to Pinecone | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Prepare Parsed Text Document | Document Default Data Loader | Converts parsed text into document object with metadata |  | Upsert Chunks to Pinecone | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Upsert Chunks to Pinecone | Pinecone Vector Store | Inserts parsed document chunks into Pinecone namespace | Normalize Parsed Text Payload; Generate Embeddings (Ingest); Prepare Parsed Text Document | Iterate Files for Parsing | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Collect Ingested Deal IDs | Aggregate | Aggregates ingested deal IDs after parsing loop | Iterate Files for Parsing | Prepare Analysis Context |  |
| Prepare Analysis Context | Code | Creates normalized base context for analysis | Cache Hit?; Collect Ingested Deal IDs | Derive Company Seed |  |
| Derive Company Seed | Code | Infers company name and search parameters from filenames | Prepare Analysis Context | Decodo Search Official Site | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Decodo Search Official Site | Decodo | Runs web search for official company site | Derive Company Seed | Build Domain Candidates | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Build Domain Candidates | Code | Builds and scores possible company domains | Decodo Search Official Site | Iterate Domain Candidates | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Iterate Domain Candidates | Split In Batches | Loops through domain candidates for verification | Build Domain Candidates; Score Domain Match | Collect Domain Scores; Decodo Verify Official Domain | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Decodo Verify Official Domain | Decodo | Scrapes candidate website for verification | Iterate Domain Candidates | Score Domain Match | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Score Domain Match | Code | Applies heuristics to accept or reject domain candidate | Decodo Verify Official Domain | Iterate Domain Candidates | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Collect Domain Scores | Aggregate | Collects all domain verification results | Iterate Domain Candidates | Select Canonical Domain | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Select Canonical Domain | Code | Selects best verified/probable domain and target URLs | Collect Domain Scores | Canonical Domain Acceptable? | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Canonical Domain Acceptable? | If | Gates external enrichment based on canonical status | Select Canonical Domain | Iterate Enrichment URLs; Prepare AI Input with Evidence | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Iterate Enrichment URLs | Split In Batches | Loops through target URLs for scraping | Canonical Domain Acceptable?; Normalize External Evidence | Collect External Evidence; Profile Page? | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Profile Page? | If | Routes about/profile pages vs general pages | Iterate Enrichment URLs | Decodo Scrape Company Profile; Decodo Scrape Commercial and Risk | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Decodo Scrape Company Profile | Decodo | Scrapes profile/about content | Profile Page? | Normalize External Evidence | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Decodo Scrape Commercial and Risk | Decodo | Scrapes commercial/risk-facing website content | Profile Page? | Normalize External Evidence | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Normalize External Evidence | Code | Standardizes scraped website content into evidence records | Decodo Scrape Company Profile; Decodo Scrape Commercial and Risk | Iterate Enrichment URLs | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Collect External Evidence | Aggregate | Gathers all scraped evidence items | Iterate Enrichment URLs | Evidence Coverage Metrics | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Evidence Coverage Metrics | Code | Computes coverage, counts, and gating decision for external evidence | Collect External Evidence | Has External Evidence? | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Has External Evidence? | If | Decides whether external evidence should be indexed | Evidence Coverage Metrics | Upsert External Evidence to Pinecone; Prepare AI Input with Evidence | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Generate Embeddings (External) | OpenAI Embeddings | Supplies embeddings for external evidence indexing |  | Upsert External Evidence to Pinecone | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Prepare External Evidence Document | Document Default Data Loader | Converts external evidence bundle into document object |  | Upsert External Evidence to Pinecone | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Upsert External Evidence to Pinecone | Pinecone Vector Store | Inserts external web evidence into deal namespace | Has External Evidence?; Generate Embeddings (External); Prepare External Evidence Document | Prepare AI Input with Evidence | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Prepare AI Input with Evidence | Code | Combines deal context with external evidence metrics | Canonical Domain Acceptable?; Has External Evidence?; Upsert External Evidence to Pinecone | Run Due Diligence AI Analysis |  |
| OpenAI Chat Model (5-mini) | OpenAI Chat Model | LLM backend for due diligence agent |  | Run Due Diligence AI Analysis | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Generate Embeddings (Retrieval) | OpenAI Embeddings | Embeddings backend for Pinecone retrieval tool |  | Pinecone Vector Store | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Pinecone Vector Store | Pinecone Vector Store | Retrieval tool for agent over deal namespace | Generate Embeddings (Retrieval) | Run Due Diligence AI Analysis | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Parse Structured Analysis JSON | Structured Output Parser | Enforces JSON schema for AI output |  | Run Due Diligence AI Analysis | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Run Due Diligence AI Analysis | LangChain Agent | Runs structured AI diligence over retrieved evidence | Prepare AI Input with Evidence; OpenAI Chat Model (5-mini); Pinecone Vector Store; Parse Structured Analysis JSON | Map Analysis to Report Fields; Merge Analysis + Report URL | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Map Analysis to Report Fields | Code | Formats AI output into HTML-ready report fields | Run Due Diligence AI Analysis | Augment Report with Evidence | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Augment Report with Evidence | Code | Adds evidence coverage details to report payload | Map Analysis to Report Fields | Render DD Report HTML | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Render DD Report HTML | HTML | Renders styled due diligence HTML report | Augment Report with Evidence | Render PDF from HTML | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Render PDF from HTML | Puppeteer | Converts HTML report into PDF base64 | Render DD Report HTML | Convert PDF Base64 to Binary File | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Convert PDF Base64 to Binary File | Convert to File | Converts PDF base64 into binary file | Render PDF from HTML | Prepare S3 File Metadata | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Prepare S3 File Metadata | Code | Builds timestamped S3 filename for report | Convert PDF Base64 to Binary File | Upload Report PDF to S3 | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Upload Report PDF to S3 | S3 | Uploads generated PDF to S3 bucket | Prepare S3 File Metadata | Build Public Report URL | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Build Public Report URL | Code | Constructs public URL for uploaded report | Upload Report PDF to S3 | Merge Analysis + Report URL | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Merge Analysis + Report URL | Merge | Combines AI result with uploaded report URL | Run Due Diligence AI Analysis; Build Public Report URL | Prepare API Response Payload | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Prepare API Response Payload | Code | Builds final webhook response JSON | Merge Analysis + Report URL | Return API Response | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Return API Response | Respond to Webhook | Sends final API response to caller | Prepare API Response Payload |  | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |
| Sticky Note | Sticky Note | Workspace documentation note |  |  | ## Due Diligence AI: Ingest, Enrich & Report<br><br>### How it works<br>1. Receive uploaded deal files via webhook, split multiple files, and generate a unified dealId.<br>2. Parse documents with LlamaParse, normalize text, create embeddings and upsert content chunks into a Pinecone namespace for the deal.<br>3. Derive a company seed from filenames, run web searches and domain verification, then scrape profile and risk pages to collect external evidence and index it in Pinecone.<br>4. Run the AI due-diligence agent (OpenAI) that queries Pinecone for internal documents + external evidence, synthesizes company profile, financial history, risks, customer concentration and an investment thesis.<br>5. Render a structured JSON analysis and a styled HTML report, convert to PDF, upload the PDF to S3, and return a public report URL in the webhook response.<br><br>### Setup<br>- [ ] Add LlamaParse / LlamaIndex API key (HTTP header auth)<br>- [ ] Connect OpenAI API key for embeddings and the analysis model<br>- [ ] Configure Pinecone credentials and ensure index "poc" exists<br>- [ ] Add Decodo (or web-scrape) API credentials for domain verification and scraping<br>- [ ] Configure S3 credentials and target bucket for report uploads<br>- [ ] Expose the webhook URL publicly and test a sample file upload<br>- [ ] (Optional) Verify Pinecone namespace retention and index quotas for large deals |
| Sticky Note1 | Sticky Note | Workspace note for intake/cache section |  |  | ## Intake + Cache Check<br>Receives file uploads, splits them into per-file items, builds a deal ID, and checks if this deal was already ingested. |
| Sticky Note2 | Sticky Note | Workspace note for parsing/vector section |  |  | ## Parse Documents + Store Vectors<br>Uploads each file for parsing, polls until ready, normalizes the extracted text, then embeds and stores chunks for later retrieval. |
| Sticky Note3 | Sticky Note | Workspace note for domain discovery section |  |  | ## Find + Validate Official Website<br>Infers a company name from filenames, searches for the official domain, verifies candidates, and picks the best canonical site to use for enrichment. |
| Sticky Note4 | Sticky Note | Workspace note for external evidence section |  |  | ## Scrape External Evidence + Index It<br>Scrapes key pages from the selected site, tags the content as profile or risk evidence, measures coverage, and stores the web evidence for the deal |
| Sticky Note5 | Sticky Note | Workspace note for AI/reporting section |  |  | ## AI Due Diligence + Report Delivery<br>Queries the deal knowledge base, generates a structured diligence summary, builds a PDF report, uploads it, and returns the report link and key fields via API. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the webhook trigger**
   - Add a **Webhook** node named **Receive Upload Request**.
   - Set:
     - HTTP Method: `POST`
     - Response Mode: `Using Respond to Webhook node`
     - Path: use a fixed path or generate one
   - Expect multipart form uploads with one or more binary files, typically in fields like `data`, `data1`, etc.
   - Optional request body field: `filenames` containing a JSON array string.

2. **Add a Code node to split files and build the deal ID**
   - Add **Code** node named **Split Uploaded Files + Build Deal ID**.
   - Logic:
     - Read `item.json.body` and `item.binary`
     - Parse `body.filenames` if present, else derive names from binary metadata
     - Create a deterministic `dealId` from sorted filenames
     - Emit one item per binary file with:
       - `dealId`
       - `sourceFile`
       - `fileType`
       - `mimeType`
       - `fileIndex`
       - `totalFiles`
     - Keep each file in `binary.data`
   - Connect from the webhook.

3. **Add a Pinecone stats request**
   - Add **HTTP Request** node named **Get Pinecone Index Stats**.
   - Configure:
     - Method: `POST`
     - URL: your Pinecone index stats endpoint, e.g. `https://<index-host>/describe_index_stats`
     - Body: empty JSON
     - Headers: `Accept: application/json`, `Content-Type: application/json`
     - Authentication: **Pinecone API credential**
     - Enable retry on fail
     - Set `onError` to continue
   - Connect from the split node.

4. **Add a cache-check Code node**
   - Add **Code** node named **Check Deal Namespace Cache**.
   - Logic:
     - Read Pinecone namespace stats
     - Read original items from the split node
     - If `stats.namespaces[dealId].vectorCount > 0`, emit a single item with:
       - `dealId`
       - `cacheHit: true`
       - `vectorCount`
     - Else re-emit original file items with `cacheHit: false`
   - Connect from Pinecone stats.

5. **Add the cache branch**
   - Add **If** node named **Cache Hit?**
   - Condition:
     - Boolean equals `{{$json.cacheHit}}` = `true`
   - Connect from the cache-check node.

6. **Add the file iteration loop**
   - Add **Split In Batches** node named **Iterate Files for Parsing**.
   - Keep default options.
   - Connect the **false** branch of **Cache Hit?** to this node.

7. **Add LlamaParse upload**
   - Add **HTTP Request** node named **Upload File to LlamaParse**.
   - Configure:
     - Method: `POST`
     - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`
     - Content Type: `multipart-form-data`
     - Form field:
       - `file` as binary from input field `data`
     - Authentication: **HTTP Header Auth** with LlamaIndex/LlamaParse API key
     - Header `accept: application/json`
   - Connect from **Iterate Files for Parsing**.

8. **Add LlamaParse status polling**
   - Add **HTTP Request** node named **Check LlamaParse Job Status**.
   - URL expression:
     - `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}`
   - Authentication: same header auth
   - Header `accept: application/json`
   - Connect from upload node.

9. **Add parsing completion check**
   - Add **If** node named **Is Parsing Job Complete?**
   - Condition:
     - String equals `{{$json.status}}` = `SUCCESS`
   - Connect from status node.

10. **Add polling wait**
    - Add **Wait** node named **Wait 10s Before Recheck**.
    - Wait 10 seconds.
    - Connect **false** output of the status check to this wait node.
    - Connect the wait node back to **Check LlamaParse Job Status**.

11. **Add parsed result retrieval**
    - Add **HTTP Request** node named **Retrieve Parsed Content**.
    - URL expression:
      - `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`
    - Same auth and accept header as above.
    - Connect **true** output of the status check to this node.

12. **Normalize parsed content**
    - Add **Code** node named **Normalize Parsed Text Payload**.
    - Logic:
      - Pull current loop file metadata from `Iterate Files for Parsing`
      - Use `$json.markdown` or `$json.text`
      - Output:
        - `dealId`
        - `sourceFile`
        - `fileType`
        - `parsedText`
   - Connect from parsed content retrieval.

13. **Add OpenAI embeddings for ingestion**
    - Add **Embeddings OpenAI** node named **Generate Embeddings (Ingest)**.
    - Use your **OpenAI API credential**.
    - Keep default options.

14. **Add parsed text document loader**
    - Add **Document Default Data Loader** node named **Prepare Parsed Text Document**.
    - Configure:
      - JSON data: `{{$json.parsedText || ""}}`
      - Metadata:
        - `deal_id = {{$json.dealId}}`
        - `source_file = {{$json.sourceFile}}`
        - `file_type = {{$json.fileType}}`
        - `timestamp = {{$now.toUTC()}}`

15. **Add Pinecone upsert for parsed documents**
    - Add **Pinecone Vector Store** node named **Upsert Chunks to Pinecone**.
    - Configure:
      - Mode: `Insert`
      - Index: `poc`
      - Namespace expression:
        - `{{ $('Iterate Files for Parsing').item.json.dealId }}`
      - `clearNamespace = false`
    - Connect:
      - Main input from **Normalize Parsed Text Payload**
      - AI Embedding from **Generate Embeddings (Ingest)**
      - AI Document from **Prepare Parsed Text Document**
    - Connect node output back to **Iterate Files for Parsing** to continue the loop.

16. **Collect ingested deal IDs**
    - Add **Aggregate** node named **Collect Ingested Deal IDs**.
    - Aggregate field: `metadata.deal_id`
    - Connect completion output of **Iterate Files for Parsing** to this node.

17. **Create common analysis context**
    - Add **Code** node named **Prepare Analysis Context**.
    - Logic:
      - If input has `cacheHit: true`, use the cache path data
      - Else read first aggregated `deal_id`
      - Output:
        - `dealId`
        - `filesProcessed`
        - `fromCache`
        - `timestamp`
    - Connect both:
      - **true** branch of **Cache Hit?**
      - **Collect Ingested Deal IDs**
      to this node.

18. **Infer company seed from filenames**
    - Add **Code** node named **Derive Company Seed**.
    - Recreate the filename analysis logic:
      - Pull uploaded filenames from **Split Uploaded Files + Build Deal ID**
      - Strip common generic deal words and version/date noise
      - Build:
        - `companyNameGuess`
        - `sourceFilenames`
        - `mentionedDomains`
        - `seedConfidence`
        - `hqCountry`
        - `searchCountryCode`
        - `searchLocale`
        - `searchQuery`
        - `fallbackSearchQuery`
    - Connect from **Prepare Analysis Context**.

19. **Search for official website**
    - Add **Decodo** node named **Decodo Search Official Site**.
    - Configure:
      - Operation: `google_search`
      - Geo: `{{$json.searchCountryCode || "us"}}`
      - Locale: `{{$json.searchLocale || "en-US"}}`
      - Query: `{{$json.searchQuery || $json.fallbackSearchQuery || $json.companyNameGuess}}`
    - Connect from **Derive Company Seed**.
    - Configure **Decodo API credentials**.

20. **Build domain candidates**
    - Add **Code** node named **Build Domain Candidates**.
    - Logic:
      - Use mentioned document domains, SERP results, and guessed TLDs
      - Exclude social/media/listing sites
      - Produce up to 3 candidates with rank and base score
    - Connect from the search node.

21. **Iterate domain candidates**
    - Add **Split In Batches** node named **Iterate Domain Candidates**.
    - Connect from **Build Domain Candidates**.

22. **Verify each domain**
    - Add **Decodo** node named **Decodo Verify Official Domain**.
    - Configure:
      - URL: `{{$json.url}}`
      - Geo: `{{$json.hqCountry || 'United States'}}`
      - Markdown: enabled
      - Headless: false
      - Retry on fail: enabled
      - Continue on error
    - Connect from **Iterate Domain Candidates**.

23. **Score candidate domains**
    - Add **Code** node named **Score Domain Match**.
    - Logic:
      - Compare scraped content with `companyNameGuess`
      - Reward exact name match, country match, corporate signals
      - Penalize scraping errors, parked domains, thin pages, social sites
      - Emit `finalScore` and `verifyStatus` (`verified`, `probable`, `reject`)
    - Connect from **Decodo Verify Official Domain**.
    - Connect this node back into **Iterate Domain Candidates** to continue the loop.

24. **Collect domain scores**
    - Add **Aggregate** node named **Collect Domain Scores**.
    - Aggregate all item data into `domainCandidates`.
    - Connect completion output of **Iterate Domain Candidates** to this node.

25. **Select canonical domain**
    - Add **Code** node named **Select Canonical Domain**.
    - Logic:
      - Sort by `finalScore`
      - If none, emit rejected canonical result
      - If verified, emit root and `/about` URLs
      - If probable, emit root only
    - Connect from **Collect Domain Scores**.

26. **Gate enrichment**
    - Add **If** node named **Canonical Domain Acceptable?**
    - Condition:
      - `{{$json.canonicalStatus}} != "reject"`
    - Connect from **Select Canonical Domain**.

27. **Iterate enrichment URLs**
    - Add **Split In Batches** node named **Iterate Enrichment URLs**.
    - Connect the true branch of **Canonical Domain Acceptable?** to it.

28. **Detect profile pages**
    - Add **If** node named **Profile Page?**
    - Condition:
      - `{{$json.targetUrl}}` contains `about`
    - Connect from **Iterate Enrichment URLs**.

29. **Add external scrapers**
    - Add **Decodo** node named **Decodo Scrape Company Profile**.
    - Add **Decodo** node named **Decodo Scrape Commercial and Risk**.
    - Both use:
      - URL: `{{$json.targetUrl}}`
      - Geo: `{{$json.hqCountry || 'United States'}}`
      - Markdown: enabled
      - Headless: false
      - Retry on fail
      - Continue on error
    - Connect:
      - True branch of **Profile Page?** to profile scraper
      - False branch to commercial/risk scraper

30. **Normalize external evidence**
    - Add **Code** node named **Normalize External Evidence**.
    - Logic:
      - Use current `Iterate Enrichment URLs` item
      - Read scraped text from Decodo response
      - Determine `evidenceType`
      - Compute `confidence`
      - Output standardized evidence record
    - Connect both Decodo scrape nodes to this node.
    - Connect this node back to **Iterate Enrichment URLs**.

31. **Collect external evidence**
    - Add **Aggregate** node named **Collect External Evidence**.
    - Aggregate all item data into `externalEvidence`.
    - Connect completion output of **Iterate Enrichment URLs** to this node.

32. **Compute evidence metrics**
    - Add **Code** node named **Evidence Coverage Metrics**.
    - Logic:
      - Separate success and failure evidence
      - Compute `externalSourcesCount`, `evidenceCoverage`, `confidenceSummary`
      - Set `evidenceGatePass`
      - If canonical was reject, force evidence metrics to zero
    - Connect from **Collect External Evidence**.

33. **Add external evidence gate**
    - Add **If** node named **Has External Evidence?**
    - Condition:
      - `{{$json.evidenceGatePass}} === true`
    - Connect from **Evidence Coverage Metrics**.

34. **Add OpenAI embeddings for external evidence**
    - Add **Embeddings OpenAI** node named **Generate Embeddings (External)**.
    - Use the same OpenAI credential.

35. **Prepare external evidence document**
    - Add **Document Default Data Loader** node named **Prepare External Evidence Document**.
    - JSON data should concatenate each evidence item’s source URL, evidence type, confidence, and markdown.
    - Metadata:
      - `deal_id`
      - `source_type = external_web`
      - `evidence_count`
      - `coverage`
      - `timestamp`

36. **Upsert external evidence**
    - Add **Pinecone Vector Store** node named **Upsert External Evidence to Pinecone**.
    - Configure:
      - Mode: `Insert`
      - Namespace: `{{$json.dealId}}`
      - Index: `poc`
      - `clearNamespace = false`
    - Connect:
      - Main input from true branch of **Has External Evidence?**
      - AI embedding from **Generate Embeddings (External)**
      - AI document from **Prepare External Evidence Document**

37. **Prepare final AI input**
    - Add **Code** node named **Prepare AI Input with Evidence**.
    - Logic:
      - Merge:
        - `Prepare Analysis Context`
        - `Evidence Coverage Metrics` if present
        - `Select Canonical Domain`
      - Force external evidence counts to zero if canonical status is reject
      - Build `externalEvidenceSummary`
    - Connect:
      - false branch of **Canonical Domain Acceptable?**
      - false branch of **Has External Evidence?**
      - output of **Upsert External Evidence to Pinecone**

38. **Add LLM node**
    - Add **OpenAI Chat Model** node named **OpenAI Chat Model (5-mini)**.
    - Select model `gpt-5-mini`.
    - Use your OpenAI credential.

39. **Add retrieval embeddings**
    - Add **Embeddings OpenAI** node named **Generate Embeddings (Retrieval)**.
    - Use the same OpenAI credential.

40. **Add Pinecone retrieval tool**
    - Add **Pinecone Vector Store** node named **Pinecone Vector Store**.
    - Configure:
      - Mode: `Retrieve as Tool`
      - Namespace: `{{$json.dealId}}`
      - Index: `poc`
      - Tool description should explain that the namespace contains parsed docs and validated external evidence.
    - Connect retrieval embeddings to this node via AI embedding input.

41. **Add structured output parser**
    - Add **Structured Output Parser** node named **Parse Structured Analysis JSON**.
    - Provide a JSON example schema with:
      - `company_profile`
      - `financials`
      - `analysis`

42. **Add AI agent**
    - Add **AI Agent** node named **Run Due Diligence AI Analysis**.
    - Set prompt type to define custom text.
    - Paste a prompt that:
      - forces multiple Pinecone queries
      - requests company profile, financial history, margins, risks, customer concentration, business model, and investment thesis
      - requires JSON-only output
      - instructs use of `"Not Available"` and `0` defaults
      - explicitly includes external evidence usage guidance
    - Enable output parser.
    - Connect:
      - Main input from **Prepare AI Input with Evidence**
      - AI language model from **OpenAI Chat Model (5-mini)**
      - AI tool from **Pinecone Vector Store**
      - AI output parser from **Parse Structured Analysis JSON**

43. **Map AI output into report fields**
    - Add **Code** node named **Map Analysis to Report Fields**.
    - Logic:
      - Normalize `raw.output`
      - Build report fields like:
        - `companyName`
        - `industry`
        - `location`
        - `employeeCount`
        - `revenueTableRows`
        - `ebitda`
        - `grossMargin`
        - `netMargin`
        - `ebitdaMargin`
        - `businessModel`
        - `investmentThesis`
        - `riskListItems`
        - `customerConcentration`
        - `dealId`
        - `reportDate`
      - Escape HTML in inserted values
    - Connect from AI agent.

44. **Add evidence-aware report augmentation**
    - Add **Code** node named **Augment Report with Evidence**.
    - Pull metrics from **Prepare AI Input with Evidence**.
    - Add disclaimer for low external evidence coverage.
    - Connect from **Map Analysis to Report Fields**.

45. **Render HTML**
    - Add **HTML** node named **Render DD Report HTML**.
    - Build a full HTML document with inline CSS using the mapped report fields.
    - Include sections for company overview, financial summary, business model, investment thesis, risk analysis, and footer.
    - Connect from **Augment Report with Evidence**.

46. **Render PDF**
    - Add **Puppeteer** node named **Render PDF from HTML**.
    - Operation: run custom script.
    - Script should:
      - read `$json.html`
      - call `$page.setContent`
      - generate A4 PDF with margins and print background
      - return `pdfBase64`
    - Connect from HTML node.

47. **Convert PDF to binary**
    - Add **Convert to File** node named **Convert PDF Base64 to Binary File**.
    - Operation: `toBinary`
    - Source property: `pdfBase64`
    - Filename expression:
      - `{{ $('Map Analysis to Report Fields').item.json.companyName }}-Analysis.pdf`
    - Connect from Puppeteer node.

48. **Prepare S3 metadata**
    - Add **Code** node named **Prepare S3 File Metadata**.
    - Build final filename like:
      - `${companyName}-assessment-${Date.now()}.pdf`
    - Pass through upstream binary.
    - Connect from convert node.

49. **Upload to S3**
    - Add **S3** node named **Upload Report PDF to S3**.
    - Configure:
      - Operation: `upload`
      - Bucket: `poc` or your bucket
      - Filename: `{{$json.fileName}}`
    - Attach **S3 credentials**
    - Connect from metadata node.

50. **Build public file URL**
    - Add **Code** node named **Build Public Report URL**.
    - Logic:
      - Read `fileName` from **Prepare S3 File Metadata**
      - URL-encode it
      - Prefix with your public base URL, e.g. `https://poc.atlr.dev`
    - Connect from S3 node.

51. **Merge analysis and report URL**
    - Add **Merge** node named **Merge Analysis + Report URL**.
    - Configure:
      - Mode: `Combine`
      - Combine by: `Position`
    - Connect:
      - Input 1 from **Run Due Diligence AI Analysis**
      - Input 2 from **Build Public Report URL**

52. **Prepare final API payload**
    - Add **Code** node named **Prepare API Response Payload**.
    - Build a final JSON object containing:
      - `success`
      - `responseStatus`
      - `dealId`
      - `fileName`
      - `publicUrl`
      - selected AI output fields
      - evidence metadata and disclaimer
    - Connect from merge node.

53. **Return webhook response**
    - Add **Respond to Webhook** node named **Return API Response**.
    - Connect from **Prepare API Response Payload**.

54. **Configure credentials**
    - Create and attach:
      - **HTTP Header Auth** for LlamaParse/LlamaIndex API
      - **OpenAI API** credential for all embeddings and chat model nodes
      - **Pinecone API** credential for both Pinecone vector store nodes and stats request
      - **Decodo API** credential for search/verification/scraping nodes
      - **S3** credential for report upload

55. **Create the Pinecone index**
    - Ensure index `poc` exists before running.
    - Its vector dimension must match the embedding model used by your OpenAI embeddings nodes.

56. **Validate public file delivery**
    - Make sure your S3 bucket and `https://poc.atlr.dev` routing really expose uploaded files publicly.
    - If not, replace the base URL logic with your actual CDN, S3 website endpoint, or signed URL generation.

57. **Test with sample upload**
    - Send a multipart POST request to the webhook containing at least one file.
    - Optional field:
      - `filenames=["NorthStar_CIM_Jan2026.docx"]`
    - Confirm:
      - Pinecone namespace created or reused
      - External evidence optionally scraped
      - AI output returned
      - PDF uploaded and URL accessible

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Due Diligence AI: Ingest, Enrich & Report — receives uploaded deal files via webhook, parses with LlamaParse, embeds into Pinecone, enriches with external web evidence, runs OpenAI due diligence analysis, renders HTML, converts to PDF, uploads to S3, and returns a public report URL. | Workspace note |
| Setup checklist: Add LlamaParse / LlamaIndex API key, OpenAI API key, Pinecone credentials with index `poc`, Decodo or equivalent web scraping credentials, S3 credentials, expose webhook publicly, optionally verify Pinecone quota and namespace retention. | Workspace note |
| Intake + Cache Check — receives file uploads, splits files, builds deal ID, and checks if deal was already ingested. | Workspace note |
| Parse Documents + Store Vectors — uploads files for parsing, polls until ready, normalizes text, embeds and stores chunks. | Workspace note |
| Find + Validate Official Website — infers a company name from filenames, searches for official domain, verifies candidates, and selects a canonical site. | Workspace note |
| Scrape External Evidence + Index It — scrapes key pages from selected site, tags profile/risk evidence, measures coverage, and stores external evidence for the deal. | Workspace note |
| AI Due Diligence + Report Delivery — queries the knowledge base, generates the diligence summary, builds a PDF, uploads it, and returns the report link and structured output. | Workspace note |

## Additional implementation notes
- The workflow has a **single entry point**: `Receive Upload Request`.
- There are **no Execute Workflow / sub-workflow invocation nodes** in this JSON.
- The workflow relies heavily on **cross-node references by name** inside Code nodes. Renaming nodes without updating expressions will break execution.
- The polling loop for LlamaParse handles only `SUCCESS` explicitly. You may want to add explicit handling for failed/cancelled parsing jobs.
- The public URL generation is **hardcoded** and assumes a separate file-serving layer over S3.
- The merge step is **position-based**; preserve one-item outputs across the analysis and PDF branches to avoid response corruption.