Generate AI document titles for Paperless-ngx with GPT-4o Mini

https://n8nworkflows.xyz/workflows/generate-ai-document-titles-for-paperless-ngx-with-gpt-4o-mini-14381


# Generate AI document titles for Paperless-ngx with GPT-4o Mini

# 1. Workflow Overview

This workflow updates a document title in Paperless-ngx after receiving an external webhook request. It validates the incoming document URL, retrieves the target document from Paperless-ngx, sanitizes the OCR text, uses GPT-4o Mini to generate a better title, removes a specified tag, and finally patches the document in Paperless-ngx with the new title and filtered tag list.

Typical use cases:
- Automatically renaming newly ingested or OCR-processed Paperless-ngx documents
- Standardizing document titles using AI
- Removing a temporary processing tag after title generation
- Triggering title updates from another automation or from Paperless-related event logic

Logical blocks in this workflow:

## 1.1 Webhook Reception and URL Validation
The workflow starts from a secured webhook, checks that the incoming `x-doc-url` value is a valid URL, and stops with an error if invalid.

## 1.2 URL Parsing and Document Retrieval
The validated Paperless document URL is decomposed into scheme, server, and document ID, then used to fetch the document record from the Paperless API.

## 1.3 Content Presence Check and Guardrails
The workflow verifies the document actually has OCR/content text. If content exists, it is sanitized with guardrails before being sent to the AI model. If not, execution ends gracefully.

## 1.4 AI Title Generation
An AI Agent uses GPT-4o Mini and a constrained system prompt to generate a concise, standardized title from the OCR text.

## 1.5 Tag Resolution, Removal, and Document Update
The workflow looks up the ID of the tag to remove, filters that tag out of the document’s existing tag list, and patches the Paperless document with the AI-generated title and revised tags.

---

# 2. Block-by-Block Analysis

## 2.1 Webhook Reception and URL Validation

**Overview:**  
This block receives the external request that asks for a document title update. It validates that the supplied document URL is syntactically valid before any Paperless API call is made.

**Nodes Involved:**  
- When Title Update Requested
- Check URL Validity
- Interrupt on Invalid URL

### Node: When Title Update Requested
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for the workflow. Accepts an authenticated HTTP POST request.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `update-document-title`
  - Authentication: header auth
- **Key expressions or variables used:**  
  Downstream nodes reference:
  - `$json.body['x-doc-url']`
  - `$json.body['x-tag-to-remove']`
- **Input and output connections:**  
  - Input: none, entry node
  - Output: Check URL Validity
- **Version-specific requirements:**  
  Uses webhook node type version `2.1`
- **Edge cases or potential failure types:**  
  - Missing or invalid header auth credentials
  - Wrong HTTP method
  - Missing `body.x-doc-url`
  - Missing `body.x-tag-to-remove` may not break immediately, but will affect tag lookup later
- **Sub-workflow reference:**  
  None

### Node: Check URL Validity
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches execution based on whether the incoming URL is valid.
- **Configuration choices:**  
  - Evaluates `={{ $json.body['x-doc-url'].isUrl() }}`
  - Condition expects boolean `true`
  - Strict type validation enabled
- **Key expressions or variables used:**  
  - `$json.body['x-doc-url'].isUrl()`
- **Input and output connections:**  
  - Input: When Title Update Requested
  - True output: Set URL Components
  - False output: Interrupt on Invalid URL
- **Version-specific requirements:**  
  Uses IF node version `2.3`
- **Edge cases or potential failure types:**  
  - If `x-doc-url` is missing or null, the expression may evaluate unexpectedly depending on runtime behavior
  - Validation is syntactic; it does not guarantee the URL is a valid Paperless document endpoint
- **Sub-workflow reference:**  
  None

### Node: Interrupt on Invalid URL
- **Type and technical role:** `n8n-nodes-base.stopAndError`  
  Explicitly halts execution when the document URL is invalid.
- **Configuration choices:**  
  - Error message: `Input value for doc_url is not a url.`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: false branch of Check URL Validity
  - Output: none
- **Version-specific requirements:**  
  Uses node version `1`
- **Edge cases or potential failure types:**  
  - Intentionally throws an execution error
- **Sub-workflow reference:**  
  None

---

## 2.2 URL Parsing and Document Retrieval

**Overview:**  
This block extracts the server, URL scheme, and document ID from the incoming Paperless document URL. It then uses those parsed parts to fetch the document object from the Paperless-ngx API.

**Nodes Involved:**  
- Set URL Components
- Fetch Document Content

### Node: Set URL Components
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates normalized fields needed for subsequent API calls.
- **Configuration choices:**  
  Assigns three string fields:
  - `server` from domain extraction
  - `scheme` from splitting at `:`
  - `documentId` from the URL path
- **Key expressions or variables used:**  
  - `={{ $json.body['x-doc-url'].extractDomain() }}`
  - `={{ $json.body['x-doc-url'].split(':')[0] }}`
  - `={{ (() => { const path = $json.body['x-doc-url'].extractUrlPath().split('/'); const id = path[path.length - 2]; return id; })() }}`
- **Input and output connections:**  
  - Input: true branch of Check URL Validity
  - Output: Fetch Document Content
- **Version-specific requirements:**  
  Uses Set node version `3.4`
- **Edge cases or potential failure types:**  
  - Assumes a path structure where the document ID is the second-to-last path segment
  - URLs with trailing structure different from `/.../<id>/` may produce wrong IDs
  - `extractDomain()` and `extractUrlPath()` rely on n8n expression helpers being available in the running version
- **Sub-workflow reference:**  
  None

### Node: Fetch Document Content
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Paperless API to retrieve the full document resource.
- **Configuration choices:**  
  - Method defaults to GET
  - URL built as: `scheme://server/api/documents/{documentId}/`
  - Response configured as full response
  - Authentication: generic credential type using HTTP Basic Auth
  - Retry on fail enabled
- **Key expressions or variables used:**  
  - `={{ $json.scheme }}://{{ $json.server }}/api/documents/{{ $json.documentId }}/`
- **Input and output connections:**  
  - Input: Set URL Components
  - Output: If Document Content Exists
- **Version-specific requirements:**  
  Uses HTTP Request version `4.3`
- **Edge cases or potential failure types:**  
  - Basic auth failure
  - Invalid host or SSL issue
  - Non-existent document ID returns 404
  - API connectivity/timeouts
  - Unexpected response schema if Paperless version differs
  - Since full response is enabled, downstream logic expects content under `$json.body`
- **Sub-workflow reference:**  
  None

---

## 2.3 Content Presence Check and Guardrails

**Overview:**  
This block determines whether the fetched Paperless document includes OCR/content text worth processing. If content exists, it is sanitized for PII before AI use; otherwise, the workflow terminates without update.

**Nodes Involved:**  
- If Document Content Exists
- Apply Document Guardrails
- Terminate Process

### Node: If Document Content Exists
- **Type and technical role:** `n8n-nodes-base.if`  
  Tests whether `body.content` exists in the fetched Paperless response.
- **Configuration choices:**  
  - OR combinator with one condition
  - Checks existence of `={{ $json.body.content }}`
- **Key expressions or variables used:**  
  - `$json.body.content`
- **Input and output connections:**  
  - Input: Fetch Document Content
  - True output: Apply Document Guardrails
  - False output: Terminate Process
- **Version-specific requirements:**  
  Uses IF node version `2.3`
- **Edge cases or potential failure types:**  
  - Checks existence, not quality; empty strings may still need special handling depending on n8n semantics
  - If API schema changes, `body.content` may be absent even though OCR text exists elsewhere
- **Sub-workflow reference:**  
  None

### Node: Apply Document Guardrails
- **Type and technical role:** `@n8n/n8n-nodes-langchain.guardrails`  
  Sanitizes OCR text before passing it to the LLM.
- **Configuration choices:**  
  - Operation: `sanitize`
  - Text input: `={{ $json.body.content }}`
  - PII protection: all
- **Key expressions or variables used:**  
  - `$json.body.content`
  - Downstream output used as `$json.guardrailsInput`
- **Input and output connections:**  
  - Input: true branch of If Document Content Exists
  - Output: Generate Title AI Agent
- **Version-specific requirements:**  
  Uses Guardrails node version `2`
  - Requires n8n environment with LangChain nodes available
- **Edge cases or potential failure types:**  
  - Guardrails node unavailable on older/self-hosted builds without AI features
  - Over-sanitization could remove useful title cues such as names, dates, or account identifiers
  - Large content bodies may still be truncated later
- **Sub-workflow reference:**  
  None

### Node: Terminate Process
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Graceful end path when there is no usable content.
- **Configuration choices:**  
  No special parameters
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: false branch of If Document Content Exists
  - Output: none
- **Version-specific requirements:**  
  Uses No Op version `1`
- **Edge cases or potential failure types:**  
  - Does not raise an error, so upstream systems may interpret the run as successful with no change
- **Sub-workflow reference:**  
  None

---

## 2.4 AI Title Generation

**Overview:**  
This block sends sanitized OCR text to an AI Agent configured with a strict system message that enforces a naming convention. GPT-4o Mini is attached as the language model for title generation.

**Nodes Involved:**  
- Generate Title AI Agent
- OpenAI GPT-4 Mini

### Node: Generate Title AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrates the prompt and model call to produce the new document title.
- **Configuration choices:**  
  - Prompt mode: define
  - Input text: first 10,000 characters of sanitized content
  - System message instructs:
    - role as archivist/information architect
    - OCR-noise tolerance
    - output title only
    - max length 60 chars
    - naming convention `[YYYY-MM-DD] - [Document Type] - [Main Subject/Entity]`
    - omit date if absent
- **Key expressions or variables used:**  
  - `={{ $json.guardrailsInput.substring(0, 10000) }}`
  - Output later referenced as `$('Generate Title AI Agent').first().json.output`
- **Input and output connections:**  
  - Input: Apply Document Guardrails
  - AI language model input: OpenAI GPT-4 Mini
  - Main output: Fetch Removal Tag ID
- **Version-specific requirements:**  
  Uses Agent node version `3.1`
  - Requires AI-capable n8n setup and compatible LangChain package versions
- **Edge cases or potential failure types:**  
  - `guardrailsInput` may be undefined if previous node fails or output format changes
  - Truncation to 10,000 chars may omit relevant document headers if OCR text starts with noise or coversheets
  - Model may not always obey 60-character limit or exact naming format
  - OCR ambiguity may lead to generic or inaccurate titles
- **Sub-workflow reference:**  
  None

### Node: OpenAI GPT-4 Mini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the LLM used by the AI Agent.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
  - Max tokens: `4096`
  - OpenAI credentials required
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: none on main branch
  - AI language model output: Generate Title AI Agent
- **Version-specific requirements:**  
  Uses OpenAI chat model node version `1.3`
  - Requires valid OpenAI API credentials
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Rate limits
  - Model availability changes
  - Cost considerations on large volumes
- **Sub-workflow reference:**  
  None

---

## 2.5 Tag Resolution, Removal, and Document Update

**Overview:**  
This block resolves the tag named in the webhook payload, removes it from the document’s current tag array if present, and applies the final PATCH update to Paperless-ngx using the AI-generated title and updated tag list.

**Nodes Involved:**  
- Fetch Removal Tag ID
- Execute Tag Removal
- Apply Document Updates

### Node: Fetch Removal Tag ID
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries the Paperless tags endpoint for a case-insensitive exact tag-name match.
- **Configuration choices:**  
  - GET request
  - URL uses parsed scheme/server from Set URL Components
  - Query implemented directly in URL: `name__iexact=...`
  - Authentication: HTTP Basic Auth
- **Key expressions or variables used:**  
  - `={{ $('Set URL Components').first().json.scheme }}://{{ $('Set URL Components').first().json.server }}/api/tags/?name__iexact={{ $('When Title Update Requested').item.json.body['x-tag-to-remove'] }}`
- **Input and output connections:**  
  - Input: Generate Title AI Agent
  - Output: Execute Tag Removal
- **Version-specific requirements:**  
  Uses HTTP Request version `4.4`
- **Edge cases or potential failure types:**  
  - Missing `x-tag-to-remove`
  - Tag not found returns empty `results`
  - Lack of URL encoding could cause problems for tags containing special characters
  - Auth/network/API errors
- **Sub-workflow reference:**  
  None

### Node: Execute Tag Removal
- **Type and technical role:** `n8n-nodes-base.code`  
  Computes a new tag array with the target tag removed.
- **Configuration choices:**  
  JavaScript logic:
  - Reads first matched tag ID from tag lookup response
  - Reads current tags from fetched document
  - Filters matching tag ID out of current tags
  - Returns `{ updatedTags }`
- **Key expressions or variables used:**  
  Internally references:
  - `$('Fetch Removal Tag ID').first()?.json?.results?.[0]?.id`
  - `$('Fetch Document Content').first()?.json?.body?.tags ?? []`
- **Input and output connections:**  
  - Input: Fetch Removal Tag ID
  - Output: Apply Document Updates
- **Version-specific requirements:**  
  Uses Code node version `2`
- **Edge cases or potential failure types:**  
  - If tag lookup returns no result, `tagIdToRemove` is `undefined` and tags remain unchanged
  - If `body.tags` is not an array, filter logic may fail unless default is used
  - Assumes only one tag should be removed
- **Sub-workflow reference:**  
  None

### Node: Apply Document Updates
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the final PATCH request to update the document title and tags in Paperless-ngx.
- **Configuration choices:**  
  - Method: `PATCH`
  - URL: `scheme://server/api/documents/{documentId}/`
  - Sends body parameters:
    - `title` from AI output
    - `tags` from filtered tag array
  - Authentication: HTTP Basic Auth
  - Retry on fail enabled
- **Key expressions or variables used:**  
  - `={{ $('Set URL Components').first().json.scheme }}://{{ $('Set URL Components').first().json.server }}/api/documents/{{ $('Set URL Components').last().json.documentId }}/`
  - `={{ $('Generate Title AI Agent').first().json.output }}`
  - `={{ $json.updatedTags }}`
- **Input and output connections:**  
  - Input: Execute Tag Removal
  - Output: none
- **Version-specific requirements:**  
  Uses HTTP Request version `4.3`
- **Edge cases or potential failure types:**  
  - PATCH may fail if title format violates Paperless validation constraints
  - AI output may be empty or malformed
  - Auth failure, permission failure, 404 on document
  - If tag array is malformed, API may reject the update
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Title Update Requested | Webhook | Receives authenticated POST requests to start the title update flow |  | Check URL Validity | ## Trigger and validation<br>Starts workflow on webhook and validates the URL. |
| Check URL Validity | If | Validates that `x-doc-url` is a proper URL | When Title Update Requested | Set URL Components; Interrupt on Invalid URL | ## Trigger and validation<br>Starts workflow on webhook and validates the URL. |
| Interrupt on Invalid URL | Stop And Error | Stops execution if the provided URL is invalid | Check URL Validity |  | ## Trigger and validation<br>Starts workflow on webhook and validates the URL. |
| Set URL Components | Set | Extracts scheme, domain, and document ID from the incoming document URL | Check URL Validity | Fetch Document Content | ## Trigger and validation<br>Starts workflow on webhook and validates the URL. |
| Fetch Document Content | HTTP Request | Retrieves the target Paperless document object from the API | Set URL Components | If Document Content Exists | ## Fetch and evaluate document<br>Fetches document content and checks if it needs processing. |
| If Document Content Exists | If | Checks whether OCR/content text exists on the document | Fetch Document Content | Apply Document Guardrails; Terminate Process | ## Fetch and evaluate document<br>Fetches document content and checks if it needs processing. |
| Apply Document Guardrails | Guardrails | Sanitizes document content before AI processing | If Document Content Exists | Generate Title AI Agent | ## AI document name generation<br>Generates a new document name using AI. |
| Generate Title AI Agent | AI Agent | Produces a standardized document title from OCR text | Apply Document Guardrails | Fetch Removal Tag ID | ## AI document name generation<br>Generates a new document name using AI. |
| OpenAI GPT-4 Mini | OpenAI Chat Model | Provides the GPT-4o Mini language model to the AI Agent |  | Generate Title AI Agent | ## AI document name generation<br>Generates a new document name using AI. |
| Fetch Removal Tag ID | HTTP Request | Looks up the ID of the tag that should be removed | Generate Title AI Agent | Execute Tag Removal | ## Tag removal and update<br>Identifies and removes unnecessary tags, then updates the document. |
| Execute Tag Removal | Code | Filters the removal tag out of the current tag list | Fetch Removal Tag ID | Apply Document Updates | ## Tag removal and update<br>Identifies and removes unnecessary tags, then updates the document. |
| Apply Document Updates | HTTP Request | PATCHes the new title and updated tags back to Paperless | Execute Tag Removal |  | ## Tag removal and update<br>Identifies and removes unnecessary tags, then updates the document. |
| Terminate Process | No Op | Ends the workflow cleanly if no content exists | If Document Content Exists |  | ## Fetch and evaluate document<br>Fetches document content and checks if it needs processing. |
| Sticky Note | Sticky Note | Documentation note for the whole workflow |  |  | ## Set Paperless Document Title w/Tag Removal<br><br>### How it works<br><br>1. The workflow is triggered by a webhook when a title needs updating.<br>2. Validates the incoming URL provided in the webhook.<br>3. Fetches document content based on the valid URL.<br>4. Uses AI to generate a new document name.<br>5. Removes unnecessary tags and updates the document.<br><br>### Setup steps<br><br>- [ ] Configure the Webhook node to receive update requests.<br>- [ ] Set up the API authentication details for all HTTP Request nodes.<br>- [ ] Ensure the AI Agent has access to required models like OpenAI Chat.<br><br>### Customization<br><br>Adjustment of tags to remove can be customized in the "Filter Out Tag to Remove" code node. |
| Sticky Note1 | Sticky Note | Visual documentation for trigger/validation block |  |  | ## Trigger and validation<br><br>Starts workflow on webhook and validates the URL. |
| Sticky Note2 | Sticky Note | Visual documentation for document fetch/evaluation block |  |  | ## Fetch and evaluate document<br><br>Fetches document content and checks if it needs processing. |
| Sticky Note3 | Sticky Note | Visual documentation for AI title generation block |  |  | ## AI document name generation<br><br>Generates a new document name using AI. |
| Sticky Note4 | Sticky Note | Visual documentation for tag removal/update block |  |  | ## Tag removal and update<br><br>Identifies and removes unnecessary tags, then updates the document. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Set Paperless Document Title w/Tag Removal`.
   - Ensure your n8n instance has the AI/LangChain nodes enabled.

2. **Add a Webhook node**
   - Node type: **Webhook**
   - Name: `When Title Update Requested`
   - HTTP method: `POST`
   - Path: `update-document-title`
   - Authentication: **Header Auth**
   - Create/select header auth credentials, for example `N8N Webhook Auth`
   - Expected request body fields:
     - `x-doc-url`: full Paperless document URL
     - `x-tag-to-remove`: tag name to remove after successful retitling

3. **Add an If node for URL validation**
   - Node type: **If**
   - Name: `Check URL Validity`
   - Add one boolean condition:
     - Left value: `={{ $json.body['x-doc-url'].isUrl() }}`
     - Operation: `true`
   - Connect `When Title Update Requested` → `Check URL Validity`

4. **Add a Stop and Error node for invalid requests**
   - Node type: **Stop And Error**
   - Name: `Interrupt on Invalid URL`
   - Error message: `Input value for doc_url is not a url.`
   - Connect the **false** output of `Check URL Validity` → `Interrupt on Invalid URL`

5. **Add a Set node to parse the URL**
   - Node type: **Set**
   - Name: `Set URL Components`
   - Create these fields:
     - `server` as string:
       - `={{ $json.body['x-doc-url'].extractDomain() }}`
     - `scheme` as string:
       - `={{ $json.body['x-doc-url'].split(':')[0] }}`
     - `documentId` as string:
       - `={{ (() => { const path = $json.body['x-doc-url'].extractUrlPath().split('/'); const id = path[path.length - 2]; return id; })() }}`
   - Connect the **true** output of `Check URL Validity` → `Set URL Components`

6. **Create Paperless API credentials**
   - Credential type: **HTTP Basic Auth**
   - Use a Paperless user that has permission to read documents, read tags, and patch documents
   - Reuse this credential in all Paperless HTTP Request nodes

7. **Add an HTTP Request node to fetch the document**
   - Node type: **HTTP Request**
   - Name: `Fetch Document Content`
   - Method: `GET`
   - URL:
     - `={{ $json.scheme }}://{{ $json.server }}/api/documents/{{ $json.documentId }}/`
   - Authentication: **Generic Credential Type**
   - Generic auth type: **HTTP Basic Auth**
   - Enable **Full Response**
   - Enable **Retry on Fail**
   - Connect `Set URL Components` → `Fetch Document Content`

8. **Add an If node to check for OCR/content**
   - Node type: **If**
   - Name: `If Document Content Exists`
   - Condition:
     - Left value: `={{ $json.body.content }}`
     - Operation: `exists`
   - Connect `Fetch Document Content` → `If Document Content Exists`

9. **Add a No Operation node for empty-content documents**
   - Node type: **No Op**
   - Name: `Terminate Process`
   - Connect the **false** output of `If Document Content Exists` → `Terminate Process`

10. **Add a Guardrails node**
    - Node type: **Guardrails**
    - Name: `Apply Document Guardrails`
    - Operation: `sanitize`
    - Text:
      - `={{ $json.body.content }}`
    - Enable PII sanitization for **all**
    - Connect the **true** output of `If Document Content Exists` → `Apply Document Guardrails`

11. **Create OpenAI credentials**
    - Credential type: **OpenAI API**
    - Use a valid API key with access to `gpt-4o-mini`

12. **Add an OpenAI Chat Model node**
    - Node type: **OpenAI Chat Model**
    - Name: `OpenAI GPT-4 Mini`
    - Model: `gpt-4o-mini`
    - Max tokens: `4096`

13. **Add an AI Agent node**
    - Node type: **AI Agent**
    - Name: `Generate Title AI Agent`
    - Prompt mode: `Define`
    - User text/input:
      - `={{ $json.guardrailsInput.substring(0, 10000) }}`
    - System message:
      - Use the archivist-style prompt from the workflow:
        - Analyze OCR text
        - Generate a concise professional title
        - Ignore OCR noise
        - Output title only
        - Max 60 characters
        - Prefer format `[YYYY-MM-DD] - [Document Type] - [Main Subject/Entity]`
        - Omit date if not found
    - Connect `Apply Document Guardrails` → `Generate Title AI Agent`
    - Connect `OpenAI GPT-4 Mini` to the AI Agent through the **AI language model** port

14. **Add an HTTP Request node to look up the removable tag**
    - Node type: **HTTP Request**
    - Name: `Fetch Removal Tag ID`
    - Method: `GET`
    - URL:
      - `={{ $('Set URL Components').first().json.scheme }}://{{ $('Set URL Components').first().json.server }}/api/tags/?name__iexact={{ $('When Title Update Requested').item.json.body['x-tag-to-remove'] }}`
    - Authentication: **Generic Credential Type**
    - Generic auth type: **HTTP Basic Auth**
    - Use the same Paperless credential
    - Connect `Generate Title AI Agent` → `Fetch Removal Tag ID`

15. **Add a Code node to filter out the tag**
    - Node type: **Code**
    - Name: `Execute Tag Removal`
    - JavaScript:
      ```javascript
      const tagIdToRemove = $('Fetch Removal Tag ID').first()?.json?.results?.[0]?.id;
      const currentTags = $('Fetch Document Content').first()?.json?.body?.tags ?? [];
      const updatedTags = currentTags.filter(id => id !== tagIdToRemove);

      return {
        updatedTags: updatedTags
      };
      ```
    - Connect `Fetch Removal Tag ID` → `Execute Tag Removal`

16. **Add an HTTP Request node to patch the document**
    - Node type: **HTTP Request**
    - Name: `Apply Document Updates`
    - Method: `PATCH`
    - URL:
      - `={{ $('Set URL Components').first().json.scheme }}://{{ $('Set URL Components').first().json.server }}/api/documents/{{ $('Set URL Components').last().json.documentId }}/`
    - Authentication: **Generic Credential Type**
    - Generic auth type: **HTTP Basic Auth**
    - Send body: enabled
    - Add body parameters:
      - `title` = `={{ $('Generate Title AI Agent').first().json.output }}`
      - `tags` = `={{ $json.updatedTags }}`
    - Enable **Retry on Fail**
    - Connect `Execute Tag Removal` → `Apply Document Updates`

17. **Add optional documentation sticky notes**
    - One global note describing purpose, setup, and customization
    - One note over the trigger/validation area
    - One note over the fetch/evaluation area
    - One note over the AI generation area
    - One note over the tag-removal/update area

18. **Activate the workflow**
    - Save and activate
    - Copy the production webhook URL

19. **Test with a sample POST request**
    - Send a POST request to the webhook with the required auth header and JSON body similar to:
      ```json
      {
        "x-doc-url": "https://paperless.example.com/documents/123/",
        "x-tag-to-remove": "needs-title"
      }
      ```
    - Verify:
      - the document is fetched correctly
      - content exists
      - AI returns a title
      - the tag is removed if found
      - the document title is patched in Paperless

20. **Recommended hardening improvements**
    - URL-encode `x-tag-to-remove` before sending it in the query string
    - Add validation to ensure the URL belongs to your trusted Paperless host
    - Add a fallback title if AI output is empty
    - Add a response node or explicit success response if the caller requires structured acknowledgement
    - Consider preserving PII if your title format depends on names/entities that guardrails may redact

**Sub-workflow setup:**  
This workflow does not invoke any sub-workflow and is not itself modeled as a callable sub-workflow. It has a single entry point: the webhook.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Main workflow note: The workflow is triggered by webhook, validates the URL, fetches document content, generates a document name with AI, removes unnecessary tags, and updates the document. | Embedded workflow documentation |
| Setup note: Configure the Webhook node, set API authentication for HTTP Request nodes, and ensure AI Agent access to OpenAI Chat or equivalent models. | Embedded workflow documentation |
| Customization note: “Adjustment of tags to remove can be customized in the "Filter Out Tag to Remove" code node.” In the actual workflow JSON, the code node is named `Execute Tag Removal`. | Embedded workflow documentation; naming mismatch to be aware of |
| The workflow title supplied by the user is “Generate AI document titles for Paperless-ngx with GPT-4o Mini”, while the internal workflow name in JSON is `Set Paperless Document Title w/Tag Removal`. | Important naming context |