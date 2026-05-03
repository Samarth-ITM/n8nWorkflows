Generate GitLab release notes from Linear issues with Claude Opus

https://n8nworkflows.xyz/workflows/generate-gitlab-release-notes-from-linear-issues-with-claude-opus-14520


# Generate GitLab release notes from Linear issues with Claude Opus

# 1. Workflow Overview

This workflow automates the creation of release-note material for a GitLab merge request by combining:

- a GitLab merge request trigger,
- release-window inference from an RSS feed,
- issue retrieval from Linear through GraphQL,
- AI-generated changelog suggestions using Claude Opus,
- and final updates back into the GitLab merge request.

Its main purpose is to help teams build public-facing release notes from completed Linear issues associated with a release range. It is especially useful for product/documentation/release managers who want to collect candidate issues, summarize them, and attach draft release-note content directly to a merge request.

## 1.1 Trigger and merge-request eligibility checks

The workflow starts on a GitLab merge request event, then verifies whether the MR has already been processed and whether it carries the expected release label.

## 1.2 Release range discovery from RSS and Linear labels

If the MR is eligible, the workflow reads a release RSS feed, uses a date from that feed to query Linear issue labels, filters version labels, and builds a release label range such as `v2.231-v2.233`.

## 1.3 Label preparation and Linear issue retrieval

The workflow expands the version range into individual labels, adds custom labels, then queries Linear for completed issues matching those labels and at least one customer-facing signal such as Zendesk, Slack, or configured custom labels.

## 1.4 Branching based on whether issues exist

If no relevant Linear issues are found, the workflow posts a note to the GitLab MR and still updates the MR labels. If issues do exist, it generates two outputs:
- a structured issue summary with links for human review,
- a richer issue dump including descriptions for AI processing.

## 1.5 GitLab summary posting and AI draft generation

The human-readable summary is posted into the MR as a GitLab discussion. In parallel, the detailed issue content is sent to Claude Opus, which produces customer-facing release-note suggestions.

## 1.6 GitLab follow-up updates

The AI-generated suggestions are added as a follow-up note inside the previously created discussion. The workflow then fetches current MR labels and updates them to mark processing complete.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and initial condition checks

### Overview
This block receives GitLab merge request events and prevents duplicate or irrelevant processing. It only continues when the MR is not already marked as done and when it contains the expected release label.

### Nodes Involved
- When GitLab MR Event Occurs
- Check MR for RN Inputs
- Check Release Label

### Node Details

#### When GitLab MR Event Occurs
- **Type and technical role:** `n8n-nodes-base.gitlabTrigger`; webhook-based trigger for GitLab merge request events.
- **Configuration choices:**
  - Owner: `your-org`
  - Repository: `your-docs-repo`
  - Event: `merge_requests`
- **Key expressions or variables used:**
  - Downstream nodes read from `{{$json.body.object_attributes...}}`
- **Input and output connections:**
  - Entry point of the workflow
  - Outputs to `Check MR for RN Inputs`
- **Version-specific requirements:**
  - Type version `1`
  - Requires GitLab trigger/webhook support and working credentials in n8n
- **Edge cases or potential failure types:**
  - Webhook not registered or invalid
  - GitLab permission issues
  - Unexpected event payload shape
  - Merge request payload missing `object_attributes`
- **Sub-workflow reference:** None

#### Check MR for RN Inputs
- **Type and technical role:** `n8n-nodes-base.if`; skips MRs already marked as processed.
- **Configuration choices:**
  - Evaluates whether the MR labels do **not** contain `rn-done`
  - Uses boolean condition with strict validation
- **Key expressions or variables used:**
  - `={{ $json.body.object_attributes.labels.some(label => label.title === "rn-done") }}`
- **Input and output connections:**
  - Input from `When GitLab MR Event Occurs`
  - True output goes to `Check Release Label`
  - False branch is unconnected, so execution ends there
- **Version-specific requirements:**
  - Type version `2.2`
- **Edge cases or potential failure types:**
  - If `labels` is missing or not an array, expression may fail
  - If label naming convention changes, duplicate prevention stops working
- **Sub-workflow reference:** None

#### Check Release Label
- **Type and technical role:** `n8n-nodes-base.if`; ensures only release-note candidate MRs proceed.
- **Configuration choices:**
  - Evaluates whether MR labels contain `rn-release-n8n`
- **Key expressions or variables used:**
  - `={{ $json.body.object_attributes.labels.some(label => label.title === "rn-release-n8n") }}`
- **Input and output connections:**
  - Input from `Check MR for RN Inputs`
  - True output goes to `Read RSS Feed`
  - False branch is unconnected
- **Version-specific requirements:**
  - Type version `2.2`
- **Edge cases or potential failure types:**
  - Same label-array assumptions as above
  - If project uses a different release label, workflow silently stops
- **Sub-workflow reference:** None

---

## 2.2 Release range discovery from RSS and Linear labels

### Overview
This block uses an RSS feed as the source of a release date anchor, then queries Linear labels created after that date. It filters version labels and compresses them into a range string.

### Nodes Involved
- Read RSS Feed
- Post Linear Labels to GraphQL
- Filter Labels Starting With v2
- Set Various Fields

### Node Details

#### Read RSS Feed
- **Type and technical role:** `n8n-nodes-base.rssFeedRead`; fetches and parses a release RSS feed.
- **Configuration choices:**
  - RSS URL: `https://YOUR_DOCS_SITE.com/releases/saas/rss.xml`
- **Key expressions or variables used:**
  - Downstream node expects an `isoDate` field from the RSS item
- **Input and output connections:**
  - Input from `Check Release Label`
  - Output to `Post Linear Labels to GraphQL`
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - Invalid or unreachable RSS URL
  - Feed does not include `isoDate`
  - Feed returns multiple items but downstream logic effectively uses one execution context
- **Sub-workflow reference:** None

#### Post Linear Labels to GraphQL
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a GraphQL query to Linear to retrieve version labels created in a date range.
- **Configuration choices:**
  - URL: `https://api.linear.app/graphql`
  - Method: `POST`
  - Authentication: predefined credential type `linearApi`
  - JSON request body with query:
    - requests `issueLabels`
    - filters by:
      - `createdAt > afterDate`
      - `createdAt < beforeDate`
      - label name starts with `v2`
  - `afterDate` comes from RSS item `isoDate`
  - `beforeDate` is generated as today at `23:59:59.999Z`
  - `executeOnce: true`
- **Key expressions or variables used:**
  - `{{ $json.isoDate }}`
  - `{{ new Date().toISOString().split('T')[0] + 'T23:59:59.999Z' }}`
- **Input and output connections:**
  - Input from `Read RSS Feed`
  - Output to `Filter Labels Starting With v2`
- **Version-specific requirements:**
  - Type version `4.2`
  - Requires valid Linear API credential in n8n
- **Edge cases or potential failure types:**
  - Linear auth failure
  - GraphQL schema changes
  - Empty label result set
  - Invalid RSS date causing malformed GraphQL variables
- **Sub-workflow reference:** None

#### Filter Labels Starting With v2
- **Type and technical role:** `n8n-nodes-base.code`; converts Linear label results into an ordered version range and date metadata.
- **Configuration choices:**
  - Reads `items[0].json.data.issueLabels.nodes`
  - Keeps labels whose names start with `v`
  - Extracts numeric part using `node.name.split('.')[1]`
  - Sorts ascending
  - Builds:
    - `versionRange`
    - `oldestLabelDate`
    - `newestLabelDate`
- **Key expressions or variables used:**
  - No expression syntax; pure JavaScript
  - Output fields:
    - `versionRange`
    - `oldestLabelDate`
    - `newestLabelDate`
- **Input and output connections:**
  - Input from `Post Linear Labels to GraphQL`
  - Output to `Set Various Fields`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - If no labels are returned, accessing `versionLabels[0]` will throw
  - If label naming is not `v2.xxx`, numeric extraction may fail
  - Multi-dot versions are not robustly parsed
- **Sub-workflow reference:** None

#### Set Various Fields
- **Type and technical role:** `n8n-nodes-base.set`; creates base arrays for version labels and custom labels.
- **Configuration choices:**
  - Sets `versions_labels` to a JSON-string-like array containing the computed `versionRange`
  - Sets `custom_labels` to:
    - `Customer request`
    - `Release note public`
- **Key expressions or variables used:**
  - `=["{{ $json.versionRange }}"]`
  - literal string array for custom labels
- **Input and output connections:**
  - Input from `Filter Labels Starting With v2`
  - Output to `Parse Linear Version Labels`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - `versions_labels` is produced as a string and later parsed; malformed quoting would break the next node
  - Custom labels are hard-coded and environment-specific
- **Sub-workflow reference:** None

---

## 2.3 Label expansion and Linear issue retrieval

### Overview
This block expands label ranges into concrete labels and queries Linear for completed issues matching the release versions and customer-facing criteria.

### Nodes Involved
- Parse Linear Version Labels
- Post Linear Issues to GraphQL

### Node Details

#### Parse Linear Version Labels
- **Type and technical role:** `n8n-nodes-base.code`; normalizes string/array inputs and expands ranges such as `v2.231-v2.233` into full label lists.
- **Configuration choices:**
  - Parses:
    - `versions_labels`
    - `custom_labels`
  - Expands both through helper function `expandVersionRange`
  - Returns original input plus:
    - `versions_labels_expanded`
    - `custom_labels_expanded`
- **Key expressions or variables used:**
  - Input fields:
    - `$json['versions_labels']`
    - `$json['custom_labels']`
- **Input and output connections:**
  - Input from `Set Various Fields`
  - Output to `Post Linear Issues to GraphQL`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - JSON parse failure if upstream strings are malformed
  - Range regex only supports `v2.<start>-v2.<end>`
  - No guard against reverse ranges
- **Sub-workflow reference:** None

#### Post Linear Issues to GraphQL
- **Type and technical role:** `n8n-nodes-base.httpRequest`; queries Linear issues with release and customer-facing filters.
- **Configuration choices:**
  - URL: `https://api.linear.app/graphql`
  - Method: `POST`
  - Authentication: predefined credential `linearApi`
  - GraphQL filter includes:
    - issue state type equals `completed`
    - labels include any in `$labels`
    - OR condition requiring at least one of:
      - attachment URL contains `zendesk.com`
      - attachment URL contains `your-company.slack.com/archives/`
      - issue has one of the custom labels expanded from previous node
  - Pagination variable `after` is conditionally included if `endCursor` exists
  - Fetch size: `first: 250`
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json.versions_labels_expanded) }}`
  - dynamic injection of custom-label filter clauses:
    - `{{ $('Parse Linear Version Labels').first().json.custom_labels_expanded.filter(...)...join('') }}`
  - optional cursor:
    - `{{ $json["endCursor"] ? ... : "" }}`
- **Input and output connections:**
  - Input from `Parse Linear Version Labels`
  - Output to `Check Linear Issues Exist`
- **Version-specific requirements:**
  - Type version `4.2`
  - Requires Linear API credential
- **Edge cases or potential failure types:**
  - GraphQL string templating is fragile
  - If a custom label contains quotes/special chars, query generation may break
  - Pagination is only partially prepared; no loop exists to fetch subsequent pages
  - Attachment hostname placeholders must be customized
- **Sub-workflow reference:** None

---

## 2.4 Linear issue existence branching

### Overview
This block decides whether the workflow should continue into summary and AI generation or instead post a fallback note indicating that no matching issues were found.

### Nodes Involved
- Check Linear Issues Exist
- Post No Issues Note to GitLab

### Node Details

#### Check Linear Issues Exist
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the Linear GraphQL response contains any issues.
- **Configuration choices:**
  - Numeric condition: issue count not equal to `0`
- **Key expressions or variables used:**
  - `={{ $json["data"]["issues"]["nodes"].length }}`
- **Input and output connections:**
  - Input from `Post Linear Issues to GraphQL`
  - True branch goes to:
    - `Generate Summary with Links`
    - `Generate Summary with Linear Details`
  - False branch goes to:
    - `Post No Issues Note to GitLab`
- **Version-specific requirements:**
  - Type version `2.2`
- **Edge cases or potential failure types:**
  - If GraphQL returns errors or a different structure, expression fails
  - Empty `data` object causes runtime issue
- **Sub-workflow reference:** None

#### Post No Issues Note to GitLab
- **Type and technical role:** `n8n-nodes-base.httpRequest`; writes a note in the MR when no qualifying issues are found.
- **Configuration choices:**
  - GitLab API endpoint for MR notes
  - Method: `POST`
  - Authentication: predefined credential `gitlabApi`
  - Form body contains explanatory message including version range and custom labels
- **Key expressions or variables used:**
  - MR IID from trigger:
    - `{{ $('When GitLab MR Event Occurs').first().json.body.object_attributes.iid }}`
  - Version range:
    - `{{ $('Parse Linear Version Labels').item.json.versionRange }}`
  - Custom labels:
    - `{{ $('Parse Linear Version Labels').item.json.custom_labels }}`
- **Input and output connections:**
  - Input from false branch of `Check Linear Issues Exist`
  - Output to `Update GitLab MR Label`
- **Version-specific requirements:**
  - Type version `4.2`
- **Edge cases or potential failure types:**
  - GitLab auth issues
  - MR IID unavailable if trigger payload changes
  - `custom_labels` may render as raw JSON/string rather than clean text
- **Sub-workflow reference:** None

---

## 2.5 Human-readable issue summary generation and posting

### Overview
This block builds a Markdown summary of all selected Linear issues, grouped by team and enriched with Linear, Zendesk, and Slack links. It then posts that summary into the GitLab MR as a discussion.

### Nodes Involved
- Generate Summary with Links
- Post Summary to GitLab MR

### Node Details

#### Generate Summary with Links
- **Type and technical role:** `n8n-nodes-base.code`; transforms Linear issues into a Markdown checklist summary intended for reviewers.
- **Configuration choices:**
  - `APPEND_LINKS = true`
  - Reads metadata from:
    - `Filter Labels Starting With v2`
    - `Parse Linear Version Labels`
  - Reads issue data from input GraphQL response
  - Computes counts:
    - total tickets
    - tickets with Zendesk attachments
    - tickets with custom labels
  - Groups issues by team
  - Sorts teams and issue identifiers
  - Outputs one field:
    - `markdown`
- **Key expressions or variables used:**
  - Uses `$input.first().json.data`
  - Uses `$('Filter Labels Starting With v2').first()?.json`
  - Uses `$('Parse Linear Version Labels').first().json...`
- **Input and output connections:**
  - Input from true branch of `Check Linear Issues Exist`
  - Output to `Post Summary to GitLab MR`
- **Version-specific requirements:**
  - Type version `2`
  - `executeOnce: true`
- **Edge cases or potential failure types:**
  - Placeholder domains must be customized:
    - `YOUR_WORKSPACE`
    - `your-company.zendesk.com`
    - `your-company.slack.com`
  - Missing issue fields may produce partial Markdown
  - Assumes GraphQL shape `data.issues.nodes`
- **Sub-workflow reference:** None

#### Post Summary to GitLab MR
- **Type and technical role:** `n8n-nodes-base.httpRequest`; creates a GitLab discussion containing the generated issue summary.
- **Configuration choices:**
  - Endpoint:
    - `/projects/YOUR_PROJECT_ID/merge_requests/{iid}/discussions`
  - Method: `POST`
  - Authentication: predefined credential `gitlabApi`
  - Form-urlencoded body:
    - `body = generated markdown`
- **Key expressions or variables used:**
  - MR IID from trigger payload
  - Markdown from `Generate Summary with Links`
- **Input and output connections:**
  - Input from `Generate Summary with Links`
  - Output is referenced later by `Post AI Suggestions to GitLab`
- **Version-specific requirements:**
  - Type version `4.2`
- **Edge cases or potential failure types:**
  - Invalid GitLab project ID
  - GitLab permission or API rate-limit errors
  - Very large Markdown body could exceed API limits
- **Sub-workflow reference:** None

---

## 2.6 AI preparation and Claude release-note draft generation

### Overview
This block creates a richer Markdown payload including issue descriptions and sends it to Claude Opus with detailed prompting for customer-facing release-note output.

### Nodes Involved
- Generate Summary with Linear Details
- Claude AI Message

### Node Details

#### Generate Summary with Linear Details
- **Type and technical role:** `n8n-nodes-base.code`; similar to the human summary node, but includes issue descriptions to improve AI context.
- **Configuration choices:**
  - Very similar grouping/statistics logic to `Generate Summary with Links`
  - Each issue line is followed by:
    - `<description>{issue.description}</description>`
  - No checkbox prefix in the issue list line
  - Outputs `markdown`
- **Key expressions or variables used:**
  - Same cross-node references as the previous summary node
- **Input and output connections:**
  - Input from true branch of `Check Linear Issues Exist`
  - Output to `Claude AI Message`
- **Version-specific requirements:**
  - Type version `2`
  - `executeOnce: true`
- **Edge cases or potential failure types:**
  - Same placeholder and schema assumptions as above
  - Very long issue descriptions may increase token usage significantly
  - XML-like tags are plain text; model may still interpret them inconsistently
- **Sub-workflow reference:** None

#### Claude AI Message
- **Type and technical role:** `@n8n/n8n-nodes-langchain.anthropic`; sends a prompt to Anthropic Claude Opus for release-note generation.
- **Configuration choices:**
  - Model: `claude-opus-4-6`
  - Single message prompt instructing model to:
    - act as a technical writer,
    - use the supplied issue data,
    - output two sections: `Enhancements` and `Fixes`,
    - format each entry as changelog-ready Markdown,
    - include linked issue IDs,
    - infer feature area from issue/team/labels when necessary.
- **Key expressions or variables used:**
  - `{{ $json.markdown }}`
  - Output later consumed as `{{ $json.content[0].text }}`
- **Input and output connections:**
  - Input from `Generate Summary with Linear Details`
  - Output to `Post AI Suggestions to GitLab`
- **Version-specific requirements:**
  - Anthropic/LangChain community package installed in n8n
  - Valid Anthropic credentials configured
  - Type version `1`
- **Edge cases or potential failure types:**
  - Missing Anthropic credentials
  - Model availability or quota issues
  - Prompt too long for context window
  - Output format may vary despite instructions
- **Sub-workflow reference:** None

---

## 2.7 Posting AI suggestions and final MR label update

### Overview
This block posts the AI draft into the GitLab discussion created earlier, then fetches existing MR labels and updates them to mark release-note processing complete.

### Nodes Involved
- Post AI Suggestions to GitLab
- Fetch GitLab MR Labels
- Update GitLab MR Label

### Node Details

#### Post AI Suggestions to GitLab
- **Type and technical role:** `n8n-nodes-base.httpRequest`; appends Claude’s output as a note under the summary discussion.
- **Configuration choices:**
  - Endpoint:
    - `/merge_requests/{iid}/discussions/{discussion_id}/notes`
  - Method: `POST`
  - Authentication: predefined credential `gitlabApi`
  - Body adds heading:
    - `## 🤖 Suggestion from Claude AI ✨ to be verified:`
  - Then inserts `{{$json.content[0].text}}`
- **Key expressions or variables used:**
  - MR IID from trigger
  - Discussion ID from `Post Summary to GitLab MR`
  - Claude output from current item
- **Input and output connections:**
  - Input from `Claude AI Message`
  - Output to `Fetch GitLab MR Labels`
- **Version-specific requirements:**
  - Type version `4.2`
- **Edge cases or potential failure types:**
  - If summary posting failed, discussion ID reference fails
  - If Anthropic output shape changes, `content[0].text` may fail
  - GitLab note length or permission issues
- **Sub-workflow reference:** None

#### Fetch GitLab MR Labels
- **Type and technical role:** `n8n-nodes-base.httpRequest`; retrieves the current MR object before label mutation.
- **Configuration choices:**
  - GET MR endpoint
  - Authentication: predefined credential `gitlabApi`
- **Key expressions or variables used:**
  - MR IID from trigger payload
- **Input and output connections:**
  - Input from `Post AI Suggestions to GitLab`
  - Output to `Update GitLab MR Label`
- **Version-specific requirements:**
  - Type version `4.2`
- **Edge cases or potential failure types:**
  - GitLab auth failure
  - MR not found
  - API response shape mismatch if self-hosted GitLab differs
- **Sub-workflow reference:** None

#### Update GitLab MR Label
- **Type and technical role:** `n8n-nodes-base.httpRequest`; updates MR labels to mark processing complete while preserving a subset of existing labels.
- **Configuration choices:**
  - Endpoint:
    - `/merge_requests/{iid}`
  - Method: `PUT`
  - Authentication: predefined credential `gitlabApi`
  - Form body sets `labels` to:
    - `rn-done`
    - plus up to first 4 current labels
    - excluding `rn-release-n8n` and `rn-self-hosted-n8n`
- **Key expressions or variables used:**
  - `=rn-done{{ $json.labels.slice(0, 4).filter(l => l && !['rn-release-n8n', 'rn-self-hosted-n8n'].includes(l)).map(l => ',' + l).join('') }}`
- **Input and output connections:**
  - Inputs from:
    - `Fetch GitLab MR Labels`
    - `Post No Issues Note to GitLab`
  - Terminal node
- **Version-specific requirements:**
  - Type version `4.2`
- **Edge cases or potential failure types:**
  - Assumes GitLab GET MR returns `labels` as array of strings
  - Arbitrary `slice(0, 4)` may drop important labels
  - If node receives input from the “no issues” path only, it still relies on current item having `labels`; in practice, only the connected output from `Fetch GitLab MR Labels` provides the expected data, so the no-issues branch as wired may not update correctly unless n8n merges execution context as expected in this version
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When GitLab MR Event Occurs | GitLab Trigger | Starts workflow on merge request events |  | Check MR for RN Inputs | ## Trigger and initial condition<br>Starts the workflow when a merge request is opened or updated in GitLab, then checks if the MR already contains release note inputs. |
| Check MR for RN Inputs | If | Prevents reprocessing of MRs already marked done | When GitLab MR Event Occurs | Check Release Label | ## Trigger and initial condition<br>Starts the workflow when a merge request is opened or updated in GitLab, then checks if the MR already contains release note inputs. |
| Check Release Label | If | Confirms MR is intended for release-note generation | Check MR for RN Inputs | Read RSS Feed | ## Check and read RSS for labels<br>Evaluates if a release label exists and reads relevant RSS feed to get initial labels. |
| Read RSS Feed | RSS Feed Read | Reads release feed to derive date window | Check Release Label | Post Linear Labels to GraphQL | ## Check and read RSS for labels<br>Evaluates if a release label exists and reads relevant RSS feed to get initial labels. |
| Post Linear Labels to GraphQL | HTTP Request | Queries Linear issue labels within date range | Read RSS Feed | Filter Labels Starting With v2 | ## Retrieve and process Linear labels<br>Fetches labels from Linear, extracts specific versions, and sets parameters for further processing. |
| Filter Labels Starting With v2 | Code | Sorts version labels and creates a version range | Post Linear Labels to GraphQL | Set Various Fields | ## Retrieve and process Linear labels<br>Fetches labels from Linear, extracts specific versions, and sets parameters for further processing. |
| Set Various Fields | Set | Creates base version/custom label arrays | Filter Labels Starting With v2 | Parse Linear Version Labels | ## Retrieve and process Linear labels<br>Fetches labels from Linear, extracts specific versions, and sets parameters for further processing. |
| Parse Linear Version Labels | Code | Expands label ranges into explicit labels | Set Various Fields | Post Linear Issues to GraphQL | ## Retrieve and process Linear labels<br>Fetches labels from Linear, extracts specific versions, and sets parameters for further processing. |
| Post Linear Issues to GraphQL | HTTP Request | Queries completed Linear issues for release note candidates | Parse Linear Version Labels | Check Linear Issues Exist | ## Get Linear issues and manage response<br>Retrieves Linear issues and decides the subsequent action, including generating summaries or adding notes if issues are absent. |
| Check Linear Issues Exist | If | Branches depending on whether matching issues were found | Post Linear Issues to GraphQL | Generate Summary with Links; Generate Summary with Linear Details; Post No Issues Note to GitLab | ## Get Linear issues and manage response<br>Retrieves Linear issues and decides the subsequent action, including generating summaries or adding notes if issues are absent. |
| Generate Summary with Links | Code | Creates human-review Markdown summary with links | Check Linear Issues Exist | Post Summary to GitLab MR | ## Generate and add high level summary<br>Creates a high-level summary and adds it to the GitLab MR. |
| Post Summary to GitLab MR | HTTP Request | Posts issue summary as GitLab discussion | Generate Summary with Links |  | ## Generate and add high level summary<br>Creates a high-level summary and adds it to the GitLab MR. |
| Generate Summary with Linear Details | Code | Builds richer issue context for AI prompting | Check Linear Issues Exist | Claude AI Message | ## AI suggestions and issue management<br>Generates AI-driven suggestions and applies labels, finalizing MR modifications with notes if necessary. |
| Claude AI Message | Anthropic | Generates customer-facing changelog suggestions | Generate Summary with Linear Details | Post AI Suggestions to GitLab | ## AI suggestions and issue management<br>Generates AI-driven suggestions and applies labels, finalizing MR modifications with notes if necessary. |
| Post AI Suggestions to GitLab | HTTP Request | Adds Claude output as a note in GitLab discussion | Claude AI Message | Fetch GitLab MR Labels | ## AI suggestions and issue management<br>Generates AI-driven suggestions and applies labels, finalizing MR modifications with notes if necessary. |
| Fetch GitLab MR Labels | HTTP Request | Retrieves current MR labels before updating them | Post AI Suggestions to GitLab | Update GitLab MR Label | ## AI suggestions and issue management<br>Generates AI-driven suggestions and applies labels, finalizing MR modifications with notes if necessary. |
| Post No Issues Note to GitLab | HTTP Request | Posts fallback note if no relevant issues are found | Check Linear Issues Exist | Update GitLab MR Label | ## AI suggestions and issue management<br>Generates AI-driven suggestions and applies labels, finalizing MR modifications with notes if necessary. |
| Update GitLab MR Label | HTTP Request | Marks MR as processed and preserves selected labels | Fetch GitLab MR Labels; Post No Issues Note to GitLab |  | ## AI suggestions and issue management<br>Generates AI-driven suggestions and applies labels, finalizing MR modifications with notes if necessary. |
| Sticky Note | Sticky Note | Canvas documentation |  |  | ## Release Note Helper (Linear + GitLab + Claude AI)<br><br>### How it works<br><br>1. Triggers on a GitLab merge request (MR) event to start the process.<br>2. Checks existing MR conditions to determine if a release label is present.<br>3. Retrieves and formats labels from a Linear RSS feed.<br>4. Based on label presence, gathers Linear issues to generate summaries and suggestions.<br>5. Applies AI to enhance MR summaries and updates the MR with suggestions and labels.<br>6. Completes the MR process with additional notes if needed.<br><br>### Setup steps<br><br>- [ ] Configure GitLab credentials for API access.<br>- [ ] Set up Linear API credentials for issue retrieval.<br>- [ ] Ensure access to the AI model service for generating summaries.<br><br>### Customization<br><br>Adjust conditions and API endpoints for different GitLab projects or Linear workspaces. |
| Sticky Note1 | Sticky Note | Canvas documentation |  |  | ## Trigger and initial condition<br><br>Starts the workflow when a merge request is opened or updated in GitLab, then checks if the MR already contains release note inputs. |
| Sticky Note2 | Sticky Note | Canvas documentation |  |  | ## Check and read RSS for labels<br><br>Evaluates if a release label exists and reads relevant RSS feed to get initial labels. |
| Sticky Note3 | Sticky Note | Canvas documentation |  |  | ## Retrieve and process Linear labels<br><br>Fetches labels from Linear, extracts specific versions, and sets parameters for further processing. |
| Sticky Note4 | Sticky Note | Canvas documentation |  |  | ## Get Linear issues and manage response<br><br>Retrieves Linear issues and decides the subsequent action, including generating summaries or adding notes if issues are absent. |
| Sticky Note5 | Sticky Note | Canvas documentation |  |  | ## Generate and add high level summary<br><br>Creates a high-level summary and adds it to the GitLab MR. |
| Sticky Note6 | Sticky Note | Canvas documentation |  |  | ## AI suggestions and issue management<br><br>Generates AI-driven suggestions and applies labels, finalizing MR modifications with notes if necessary. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Release Note Helper (Linear + GitLab + Claude AI)`.
   - Keep execution order as default `v1` if you want behavior aligned with this workflow.

2. **Add a GitLab Trigger node**
   - Node type: **GitLab Trigger**
   - Name: `When GitLab MR Event Occurs`
   - Configure:
     - Owner: `your-org`
     - Repository: `your-docs-repo`
     - Events: `merge_requests`
   - Attach GitLab trigger credentials.
   - Ensure GitLab can reach the generated webhook URL.

3. **Add an If node to prevent duplicate processing**
   - Node type: **If**
   - Name: `Check MR for RN Inputs`
   - Condition:
     - Boolean
     - Expression:
       `{{ $json.body.object_attributes.labels.some(label => label.title === "rn-done") }}`
     - Check that this is **false**
   - Connect:
     - `When GitLab MR Event Occurs` → `Check MR for RN Inputs`

4. **Add an If node to require the release label**
   - Node type: **If**
   - Name: `Check Release Label`
   - Condition:
     - Boolean
     - Expression:
       `{{ $json.body.object_attributes.labels.some(label => label.title === "rn-release-n8n") }}`
     - Check that this is **true**
   - Connect:
     - `Check MR for RN Inputs` true output → `Check Release Label`

5. **Add an RSS Feed Read node**
   - Node type: **RSS Feed Read**
   - Name: `Read RSS Feed`
   - URL:
     `https://YOUR_DOCS_SITE.com/releases/saas/rss.xml`
   - Connect:
     - `Check Release Label` true output → `Read RSS Feed`

6. **Add an HTTP Request node to query Linear labels**
   - Node type: **HTTP Request**
   - Name: `Post Linear Labels to GraphQL`
   - Configure:
     - Method: `POST`
     - URL: `https://api.linear.app/graphql`
     - Authentication: predefined credential type
     - Credential: **Linear API**
     - Body type: JSON
   - JSON body:
     ```json
     {
       "query": "query ($afterDate: DateTimeOrDuration!, $beforeDate: DateTimeOrDuration!) { issueLabels(filter: { createdAt: { gt: $afterDate, lt: $beforeDate }, name: { startsWith: \"v2\" } }) { nodes { name, createdAt } } }",
       "variables": {
         "afterDate": "{{ $json.isoDate }}",
         "beforeDate": "{{ new Date().toISOString().split('T')[0] + 'T23:59:59.999Z' }}"
       }
     }
     ```
   - Enable **Execute Once**
   - Connect:
     - `Read RSS Feed` → `Post Linear Labels to GraphQL`

7. **Add a Code node to build the version range**
   - Node type: **Code**
   - Name: `Filter Labels Starting With v2`
   - Paste the logic that:
     - reads `data.issueLabels.nodes`,
     - keeps labels starting with `v`,
     - sorts by numeric suffix,
     - outputs:
       - `versionRange`
       - `oldestLabelDate`
       - `newestLabelDate`
   - Connect:
     - `Post Linear Labels to GraphQL` → `Filter Labels Starting With v2`

8. **Add a Set node for base label inputs**
   - Node type: **Set**
   - Name: `Set Various Fields`
   - Add string fields:
     - `versions_labels` = `["{{ $json.versionRange }}"]`
     - `custom_labels` = `["Customer request", "Release note public"]`
   - Connect:
     - `Filter Labels Starting With v2` → `Set Various Fields`

9. **Add a Code node to expand label ranges**
   - Node type: **Code**
   - Name: `Parse Linear Version Labels`
   - Implement logic to:
     - parse `versions_labels` and `custom_labels`,
     - expand ranges like `v2.231-v2.233`,
     - return:
       - original values,
       - `versions_labels_expanded`,
       - `custom_labels_expanded`
   - Connect:
     - `Set Various Fields` → `Parse Linear Version Labels`

10. **Add an HTTP Request node to query Linear issues**
    - Node type: **HTTP Request**
    - Name: `Post Linear Issues to GraphQL`
    - Configure:
      - Method: `POST`
      - URL: `https://api.linear.app/graphql`
      - Authentication: predefined credential type
      - Credential: **Linear API**
      - Body type: JSON
    - Use a GraphQL query that:
      - fetches completed issues,
      - filters by version labels from `versions_labels_expanded`,
      - requires at least one of:
        - Zendesk attachment,
        - Slack attachment,
        - custom labels,
      - returns issue metadata including labels, team, attachments, description.
    - Use the workflow’s dynamic variables:
      - `{{ JSON.stringify($json.versions_labels_expanded) }}`
      - dynamic custom label clause from `Parse Linear Version Labels`
    - Connect:
      - `Parse Linear Version Labels` → `Post Linear Issues to GraphQL`

11. **Add an If node to check whether issues exist**
    - Node type: **If**
    - Name: `Check Linear Issues Exist`
    - Condition:
      - Number
      - Expression:
        `{{ $json["data"]["issues"]["nodes"].length }}`
      - Operation: `notEquals`
      - Value: `0`
    - Connect:
      - `Post Linear Issues to GraphQL` → `Check Linear Issues Exist`

12. **Add a fallback HTTP Request node for the no-issues path**
    - Node type: **HTTP Request**
    - Name: `Post No Issues Note to GitLab`
    - Configure:
      - Method: `POST`
      - URL:
        `https://gitlab.example.com/api/v4/projects/YOUR_PROJECT_ID/merge_requests/{{ $('When GitLab MR Event Occurs').first().json.body.object_attributes.iid }}/notes`
      - Authentication: predefined credential type
      - Credential: **GitLab API**
      - Content type: `form-urlencoded`
      - Body field:
        - `body` = explanatory text referencing:
          - `{{ $('Parse Linear Version Labels').item.json.versionRange }}`
          - `{{ $('Parse Linear Version Labels').item.json.custom_labels }}`
    - Connect:
      - false output of `Check Linear Issues Exist` → `Post No Issues Note to GitLab`

13. **Add a Code node for the human-readable summary**
    - Node type: **Code**
    - Name: `Generate Summary with Links`
    - Use code that:
      - reads issue data from `data.issues.nodes`,
      - groups by team,
      - counts total/Zendesk/custom-label tickets,
      - formats Markdown checklist entries,
      - optionally adds links to:
        - Linear,
        - Zendesk,
        - Slack
      - outputs `markdown`
    - Important placeholders to replace:
      - `YOUR_WORKSPACE`
      - `your-company.zendesk.com`
      - `your-company.slack.com/archives/`
    - Enable **Execute Once**
    - Connect:
      - true output of `Check Linear Issues Exist` → `Generate Summary with Links`

14. **Add an HTTP Request node to post the summary discussion**
    - Node type: **HTTP Request**
    - Name: `Post Summary to GitLab MR`
    - Configure:
      - Method: `POST`
      - URL:
        `https://gitlab.example.com/api/v4/projects/YOUR_PROJECT_ID/merge_requests/{{ $('When GitLab MR Event Occurs').first().json.body.object_attributes.iid }}/discussions`
      - Authentication: predefined credential type
      - Credential: **GitLab API**
      - Content type: `form-urlencoded`
      - Body field:
        - `body` = `{{ $('Generate Summary with Links').item.json.markdown }}`
    - Connect:
      - `Generate Summary with Links` → `Post Summary to GitLab MR`

15. **Add a second Code node for AI input preparation**
    - Node type: **Code**
    - Name: `Generate Summary with Linear Details`
    - Duplicate the summary logic, but for each issue also include:
      - `<description>{{issue.description}}</description>`
    - Output field: `markdown`
    - Enable **Execute Once**
    - Connect:
      - true output of `Check Linear Issues Exist` → `Generate Summary with Linear Details`

16. **Add an Anthropic node**
    - Node type: **Anthropic** from the LangChain-compatible n8n package
    - Name: `Claude AI Message`
    - Configure credentials for Anthropic API.
    - Select model:
      - `claude-opus-4-6`
    - Add one message containing:
      - role/task as technical writer,
      - embedded data block with `{{ $json.markdown }}`,
      - formatting instructions for:
        - `### Enhancements`
        - `### Fixes`
        - checklist bullet format,
        - bold feature area,
        - linked Linear issue ID,
      - concise changelog style guidance.
    - Connect:
      - `Generate Summary with Linear Details` → `Claude AI Message`

17. **Add an HTTP Request node to post AI suggestions**
    - Node type: **HTTP Request**
    - Name: `Post AI Suggestions to GitLab`
    - Configure:
      - Method: `POST`
      - URL:
        `https://gitlab.example.com/api/v4/projects/YOUR_PROJECT_ID/merge_requests/{{ $('When GitLab MR Event Occurs').first().json.body.object_attributes.iid }}/discussions/{{ $('Post Summary to GitLab MR').first().json.id}}/notes`
      - Authentication: predefined credential type
      - Credential: **GitLab API**
      - Content type: `form-urlencoded`
      - Body field:
        - `body` = Markdown text that begins with:
          `## 🤖 Suggestion from Claude AI ✨ to be verified:`
          followed by `{{ $json.content[0].text }}`
    - Connect:
      - `Claude AI Message` → `Post AI Suggestions to GitLab`

18. **Add an HTTP Request node to fetch current MR labels**
    - Node type: **HTTP Request**
    - Name: `Fetch GitLab MR Labels`
    - Configure:
      - Method: `GET`
      - URL:
        `https://gitlab.example.com/api/v4/projects/YOUR_PROJECT_ID/merge_requests/{{ $('When GitLab MR Event Occurs').first().json.body.object_attributes.iid }}`
      - Authentication: predefined credential type
      - Credential: **GitLab API**
    - Connect:
      - `Post AI Suggestions to GitLab` → `Fetch GitLab MR Labels`

19. **Add an HTTP Request node to update MR labels**
    - Node type: **HTTP Request**
    - Name: `Update GitLab MR Label`
    - Configure:
      - Method: `PUT`
      - URL:
        `https://gitlab.example.com/api/v4/projects/YOUR_PROJECT_ID/merge_requests/{{ $('When GitLab MR Event Occurs').first().json.body.object_attributes.iid }}`
      - Authentication: predefined credential type
      - Credential: **GitLab API**
      - Content type: `form-urlencoded`
      - Body field:
        - `labels` =
          `rn-done{{ $json.labels.slice(0, 4).filter(l => l && !['rn-release-n8n', 'rn-self-hosted-n8n'].includes(l)).map(l => ',' + l).join('') }}`
    - Connect:
      - `Fetch GitLab MR Labels` → `Update GitLab MR Label`

20. **Connect the no-issues branch to the label update path**
    - Connect:
      - `Post No Issues Note to GitLab` → `Update GitLab MR Label`
    - Important implementation note:
      - As designed, `Update GitLab MR Label` expects a payload containing `labels`.
      - The no-issues branch does not fetch MR labels first.
      - To make this branch robust, add an extra `Fetch GitLab MR Labels` node before `Update GitLab MR Label` on the no-issues path, or merge both paths after fetching labels.

21. **Create credentials**
    - **GitLab API / Trigger credentials**
      - Personal access token or OAuth2 depending on your setup
      - Must allow webhook registration and MR API access
    - **Linear API credentials**
      - API key with permission to query GraphQL issues and labels
    - **Anthropic credentials**
      - API key with access to `claude-opus-4-6`

22. **Replace all placeholders**
    - `gitlab.example.com`
    - `YOUR_PROJECT_ID`
    - `YOUR_DOCS_SITE.com`
    - `YOUR_WORKSPACE`
    - `your-company.zendesk.com`
    - `your-company.slack.com/archives/`
    - `your-org`
    - `your-docs-repo`

23. **Test with a merge request**
    - Add `rn-release-n8n` to a test MR.
    - Ensure it does not already contain `rn-done`.
    - Trigger an MR event by updating the MR.
    - Verify:
      - summary discussion posted,
      - AI suggestion note added,
      - MR labels updated to include `rn-done`.

24. **Recommended hardening before production**
    - Add null checks around MR label access
    - Handle empty Linear label results safely
    - Add GraphQL error handling
    - Add pagination loop for Linear issues beyond 250
    - Normalize the no-issues branch so label update always fetches MR labels first

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Release Note Helper (Linear + GitLab + Claude AI): triggers on GitLab MR events, checks conditions, retrieves labels from RSS/Linear, gathers issues, uses AI for summary suggestions, updates the MR, and completes with notes if needed. | Workflow-level description |
| Setup steps: configure GitLab credentials, set up Linear API credentials, and ensure access to the AI model service. | Workflow-level setup guidance |
| Customization: adjust conditions and API endpoints for different GitLab projects or Linear workspaces. | Workflow-level customization note |