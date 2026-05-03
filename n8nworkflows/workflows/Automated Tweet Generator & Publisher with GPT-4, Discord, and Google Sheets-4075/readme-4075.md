Automated Tweet Generator & Publisher with GPT-4, Discord, and Google Sheets

https://n8nworkflows.xyz/workflows/automated-tweet-generator---publisher-with-gpt-4--discord--and-google-sheets-4075


# Automated Tweet Generator & Publisher with GPT-4, Discord, and Google Sheets

### 1. Workflow Overview

This workflow automates the entire process of generating, refining, approving, and publishing tweets (now X posts) for personal brands or creators. It leverages AI models (GPT-4), conversational approval via Discord, and Google Sheets for storage and history tracking. The workflow is designed to produce consistent, bold, and high-quality content with minimal manual effort, incorporating brand voice and style alignment, duplicate avoidance, and human-in-the-loop approval.

**Logical Blocks:**

- **1.1 Input Initialization & Brand Brief Retrieval:** Triggering the workflow and fetching the brand brief from an external sub-workflow or Notion.
- **1.2 Idea Generation:** Creating potential tweet topic ideas based on the brand brief using AI.
- **1.3 Post Creation & Quality Assessment Loop:** Drafting the tweet, evaluating it via feedback sub-workflow, rewriting if necessary, and style refinement.
- **1.4 Duplicate Checking & Style Rewriting:** Verifying against Google Sheets to avoid reposting and rewriting posts to match user’s writing style.
- **1.5 Approval via Discord:** Sending the final post to a Discord channel for manual user approval or rejection with retry capability.
- **1.6 Publishing & Logging:** Posting approved tweets to Twitter (X) and updating Google Sheets logs for history and examples.
- **1.7 Sub-workflows:** Separate workflows for fetching brand brief and content feedback evaluation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization & Brand Brief Retrieval

- **Overview:** This block initiates the workflow manually or via another workflow and obtains the brand brief, which guides all content creation.
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)  
  - `Get Brief` (Execute Workflow)  
  - `Notion` (Optional brand brief retrieval)  
  - `Aggregate` (Combines Notion content)  
  - `Edit Fields1` (Prepares brand brief content)  
- **Node Details:**

| Node Name               | Details                                                                                                                                            |
|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual trigger node to start the workflow manually.                                                                                              |
| Get Brief               | Executes the sub-workflow `Get Brand Brief` to retrieve brand tone and voice. Inputs none, outputs the brand brief as text.                        |
| Notion                  | (Optional) Retrieves brand brief blocks from a Notion page by block ID; returns all blocks for aggregation.                                        |
| Aggregate               | Aggregates all Notion blocks’ content into a single concatenated string to form the brand brief text.                                              |
| Edit Fields1            | Sets a single field named `content` containing the aggregated brand brief text for downstream use.                                                 |

- **Edge Cases & Errors:**
  - Notion API rate limits or invalid block IDs may cause failures.
  - Sub-workflow `Get Brand Brief` must return valid text; otherwise, downstream AI nodes fail.
  - Manual trigger requires user intervention unless automated externally.

---

#### 2.2 Idea Generation

- **Overview:** Generates 10 tweet topic suggestions aligned with the brand brief using an AI language model.
- **Nodes Involved:**  
  - `Idea creator` (Chain LLM)  
  - `Structured Output Parser` (Parses AI JSON output)  
  - `Edit Fields` (Sets `Post Idea` array)  
- **Node Details:**

| Node Name              | Details                                                                                                                                            |
|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Idea creator           | Uses GPT-4o-mini model to generate 10 post topic suggestions formatted as JSON with a `suggestions` array, based on the brand brief content field. |
| Structured Output Parser | Parses the AI-generated JSON to extract the `suggestions` array cleanly for use in later nodes.                                                     |
| Edit Fields            | Assigns the parsed `suggestions` as the `Post Idea` field for downstream AI content creation.                                                      |

- **Key Expressions:**  
  - Prompt embeds the brand brief text dynamically (`{{ $json.content }}`) for context.
  - Output is strictly JSON-parsed to avoid format errors.

- **Edge Cases & Errors:**
  - AI model might produce malformed JSON, failing the parser.
  - Brand brief content quality directly impacts idea relevance.

---

#### 2.3 Post Creation & Quality Assessment Loop

- **Overview:** Creates the initial tweet draft, evaluates quality and brand alignment, and rewrites until a minimum quality threshold is met.
- **Nodes Involved:**  
  - `AI Content creator` (Langchain Agent)  
  - `Get_Brand_Brief` (Tool Workflow - fetches brand brief dynamically)  
  - `Get_Content_Fedback` (Tool Workflow - evaluates post quality)  
  - `Check History` (Google Sheets check for duplicates)  
  - `OpenAI Chat Model` (GPT-4.1-mini for agent’s language model)  
  - `Simple Memory` (Maintains session context for AI agent)  
- **Node Details:**

| Node Name           | Details                                                                                                                                           |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| AI Content creator  | Core agent node that:  <br>1. Fetches brand brief via sub-workflow `Get_Brand_Brief`. <br>2. Creates a post idea from available suggestions. <br>3. Gets post feedback and score from `Get_Content_Fedback`. <br>4. If score < 0.7, rewrites and repeats. <br>5. Checks Google Sheets history for duplicates. <br>6. Outputs approved post text. <br>Configured with max 10 iterations, spartan tone, offensive and casual style, lowercase only. |
| Get_Brand_Brief     | Invoked as a tool inside the agent to dynamically retrieve the latest brand brief during iterative post creation.                                  |
| Get_Content_Fedback | Invoked as a tool to evaluate the AI-generated post against brand alignment and quality metrics, returning a JSON with `description` and `score`. |
| Check History       | Google Sheets node that searches the history spreadsheet to ensure the generated post idea has not been previously published.                     |
| OpenAI Chat Model   | Provides the GPT-4.1-mini language model used by the agent for content generation and refinement.                                                 |
| Simple Memory       | Maintains conversational context over multiple iterations for the agent to build upon previous outputs and feedback.                              |

- **Edge Cases & Errors:**
  - Infinite loop if feedback scoring or duplicate checks malfunction.
  - Google Sheets API limits or sheet misconfiguration may block duplicate checks.
  - Agent may output inappropriate content unless prompt carefully designed.
  - OpenAI API quota or connectivity issues can cause failures.

---

#### 2.4 Duplicate Checking & Style Rewriting

- **Overview:** Rewrites the approved post to better match the user’s actual writing style using examples stored in Google Sheets.
- **Nodes Involved:**  
  - `Rewriter` (Langchain Agent)  
  - `Check Examples` (Google Sheets fetch of past posts for style)  
  - `Simple Memory1` (Agent session memory for style rewriting)  
  - `OpenAI Chat Model1` (GPT-4.1-mini for rewriter agent)  
- **Node Details:**

| Node Name         | Details                                                                                                                                           |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| Rewriter          | Agent that rewrites the post to match user style, using examples from the `Check Examples` Google Sheet. Maintains the same tone and style rules as creator. |
| Check Examples    | Retrieves examples of previously published posts from the `Examples` sheet in Google Sheets.                                                      |
| Simple Memory1    | Holds conversational context for the rewriting process.                                                                                           |
| OpenAI Chat Model1| Language model instance used by the rewriting agent.                                                                                            |

- **Key Expressions:**  
  - The prompt includes instructions to analyze examples and rewrite creatively, maintaining a spartan, confident, and controversial tone.

- **Edge Cases & Errors:**
  - If no examples exist, rewriting quality may degrade.
  - Google Sheets access errors or empty sheets cause failures.
  - AI may fail to align style if examples are inconsistent.

---

#### 2.5 Approval via Discord

- **Overview:** Sends the rewritten post to a Discord channel for manual approval, supporting approval or retry with user interaction.
- **Nodes Involved:**  
  - `Send post for approval` (Discord message with approval prompt)  
  - `If2` (Checks if approved)  
  - `Post on X` (Twitter publishing if approved)  
  - `Try Again ` (Discord message asking to retry if not approved)  
  - `If1` (Checks if user wants to try again)  
  - `Confirm of next try` (Discord confirmation to retry)  
  - `End of work` (Discord message signaling workflow end if not retrying)  
  - `Discord confirm` (Confirms successful posting)  
- **Node Details:**

| Node Name           | Details                                                                                                                                        |
|---------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Send post for approval | Sends the final post text to the configured Discord channel and waits for user interaction (approve/disapprove).                              |
| If2                 | Branch node evaluating Discord approval response; if true, proceeds to publish; if false, asks to retry.                                       |
| Post on X           | Publishes the post to Twitter (X) using OAuth2 credentials with the rewritten post text.                                                       |
| Try Again           | Sends a "Try Again?" prompt on Discord with approval options to retry editing or stop.                                                         |
| If1                 | Checks the retry response; if approved, restarts post creation; if disapproved, ends workflow.                                                  |
| Confirm of next try | Sends a confirmation message on Discord indicating the retry process is starting.                                                              |
| End of work         | Sends a Discord message indicating the workflow has ended with no tweet published.                                                             |
| Discord confirm     | Sends a Discord message confirming that the post was successfully sent to Twitter.                                                             |

- **Edge Cases & Errors:**
  - Discord API rate limits or missing permissions may block message sending or approval capture.
  - User indecision or no response may stall workflow.
  - Twitter OAuth2 token expiry or errors may prevent publishing.

---

#### 2.6 Publishing & Logging

- **Overview:** After approval, the post is published on Twitter and logged into Google Sheets for history and examples.
- **Nodes Involved:**  
  - `Post on X` (Twitter API node)  
  - `Add post to examples` (Appends post to Google Sheets Examples tab)  
  - `Update history` (Logs post with timestamp and published status)  
  - `Discord confirm` (Confirms posting)  
- **Node Details:**

| Node Name          | Details                                                                                                                                    |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Post on X          | Posts the finalized tweet text to Twitter (X) using OAuth2 credential, sourced from the rewritten post node output.                        |
| Add post to examples | Appends the published post to an `Examples` sheet in Google Sheets to build the example database for future rewriting.                     |
| Update history     | Appends the post, current timestamp, and published status (`TRUE`) to the `History` sheet in Google Sheets for duplicate tracking.        |
| Discord confirm    | Sends a confirmation message to the designated Discord channel notifying successful posting and workflow progression.                      |

- **Edge Cases & Errors:**
  - Google Sheets append failures can cause data loss for history/examples.
  - Twitter API errors, e.g. posting limits or content restrictions.
  - Discord confirmation failure does not affect publishing but may reduce user feedback.

---

#### 2.7 Sub-Workflows

- **Get Brand Brief:**  
  - Trigger: `When Executed by Another Workflow` node.  
  - Function: Returns the brand brief as a single text field for use by AI agents.  
  - Dependencies: Pulls data from Notion optionally, or other sources.  

- **Get Content Feedback:**  
  - Trigger: `When Executed by Another Workflow`.  
  - Function: Takes a post and brand brief as input and returns JSON with `description` and a `score` (0 to 1) evaluating brand alignment and content quality.  
  - Uses OpenAI GPT-4o-mini for evaluation.

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                                   | Input Node(s)                  | Output Node(s)                  | Sticky Note                                             |
|----------------------------|----------------------------------------|--------------------------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                         | Workflow manual start                             |                               | Get Brief                      |                                                         |
| Get Brief                  | Execute Workflow                       | Runs Get Brand Brief sub-workflow                 | When clicking ‘Test workflow’ | Idea creator                   |                                                         |
| Notion                     | Notion                                | Retrieves brand brief blocks                       | When Executed by Another Workflow | Aggregate                    |                                                         |
| Aggregate                  | Aggregate                             | Combines Notion content                            | Notion                        | Edit Fields1                  |                                                         |
| Edit Fields1               | Set                                  | Prepares brand brief text                          | Aggregate                     | Idea creator                   |                                                         |
| Idea creator               | Chain LLM                            | Generates tweet topic ideas                        | Get Brief / Edit Fields1      | Structured Output Parser        | # Idea creator - AI Agent finding relevant ideas.       |
| Structured Output Parser   | Output Parser Structured             | Parses AI JSON output                              | Idea creator                  | Edit Fields                    |                                                         |
| Edit Fields                | Set                                  | Sets `Post Idea` array                             | Structured Output Parser      | AI Content creator             |                                                         |
| AI Content creator         | Langchain Agent                      | Creates and refines post, checks duplicates & quality | Edit Fields / Get_Brand_Brief / Get_Content_Fedback / Check History / Simple Memory / OpenAI Chat Model | Rewriter                      | # Content creator - Writes first draft with feedback.   |
| Get_Brand_Brief            | Tool Workflow                       | Provides brand brief dynamically                   | AI Content creator            | AI Content creator             |                                                         |
| Get_Content_Fedback        | Tool Workflow                       | Evaluates post quality & alignment                 | AI Content creator            | AI Content creator             |                                                         |
| Check History              | Google Sheets Tool                  | Checks for duplicate posts                          | AI Content creator            | AI Content creator             |                                                         |
| OpenAI Chat Model          | Langchain LM Chat OpenAI             | GPT-4.1-mini model for AI content                  | AI Content creator            | AI Content creator             |                                                         |
| Simple Memory              | Langchain Memory Buffer              | Maintains AI agent session context                  | AI Content creator            | AI Content creator             |                                                         |
| Rewriter                   | Langchain Agent                      | Rewrites post to match user style                   | AI Content creator / Check Examples / Simple Memory1 / OpenAI Chat Model1 | Send post for approval      | # Rewriter - Rewrites posts matching user style.         |
| Check Examples             | Google Sheets Tool                  | Retrieves example posts for style analysis          | Rewriter                     | Rewriter                      |                                                         |
| Simple Memory1             | Langchain Memory Buffer              | Maintains rewriting session context                 | Rewriter                     | Rewriter                      |                                                         |
| OpenAI Chat Model1         | Langchain LM Chat OpenAI             | GPT-4.1-mini model for rewriting                     | Rewriter                     | Rewriter                      |                                                         |
| Send post for approval     | Discord                              | Sends post to Discord for approval                   | Rewriter                     | If2                          | # Additional approval - Sends post for approval on Discord. |
| If2                        | If                                   | Checks Discord approval response                     | Send post for approval        | Post on X / Try Again          |                                                         |
| Post on X                  | Twitter                             | Publishes approved post to Twitter (X)               | If2                          | Add post to examples           | # Post On X(Twitter)                                      |
| Try Again                  | Discord                             | Sends retry prompt on Discord                         | If2                          | If1                          |                                                         |
| If1                        | If                                   | Checks user decision to retry or stop                 | Try Again                    | Confirm of next try / End of work |                                                         |
| Confirm of next try        | Discord                             | Confirms retry start on Discord                        | If1                          | AI Content creator             |                                                         |
| End of work                | Discord                             | Sends end of workflow message on Discord              | If1                          |                              |                                                         |
| Add post to examples       | Google Sheets                       | Logs post to Examples tab                             | Post on X                    | Update history                | # Update History and Examples                             |
| Update history             | Google Sheets                       | Logs post history and published flag                  | Add post to examples         | Discord confirm               |                                                         |
| Discord confirm            | Discord                             | Confirms successful posting                            | Update history               |                              | # Confirm of End Workflow                                 |
| When Executed by Another Workflow | Execute Workflow Trigger          | Sub-workflow trigger for brand brief retrieval        |                              | Notion                       |                                                         |
| OpenAI                     | Langchain OpenAI                    | Evaluates post quality in sub-workflow                  | Get brand brief              | Edit Fields2                 |                                                         |
| Edit Fields2               | Set                                  | Extracts feedback and score from OpenAI response       | OpenAI                      |                              |                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger:**  
   - Node: `Manual Trigger` (When clicking ‘Test workflow’).  
   - No parameters needed.

2. **Set up Brand Brief Retrieval:**  
   - Create sub-workflow `Get Brand Brief` with an `Execute Workflow Trigger`.  
   - In main, add a `Notion` node (optional) to fetch blocks by URL or ID.  
   - Add `Aggregate` node to combine block contents.  
   - Add `Set` node (`Edit Fields1`) to assign aggregated content to a `content` field.  
   - Return this text as output for use in main workflow.  
   - In main workflow, add an `Execute Workflow` node (`Get Brief`) that calls the sub-workflow. Connect manual trigger to this node.

3. **Idea Generation:**  
   - Add a `Chain LLM` node (`Idea creator`) using GPT-4o-mini.  
   - Configure prompt: ask for 10 post topic suggestions in JSON format, including brand brief text (`{{ $json.content }}`).  
   - Enable JSON output parsing with `Structured Output Parser` node.  
   - Add `Set` node (`Edit Fields`) to assign parsed suggestions to `Post Idea` array.

4. **AI Content Creation Loop:**  
   - Add a `Langchain Agent` node (`AI Content creator`) with:  
     - System message instructing steps: get brand brief, create post, get feedback, rewrite if score < 0.7, check duplicates, finalize post.  
     - Connect tool workflows:  
       - `Get_Brand_Brief` (Execute Workflow node or Tool Workflow node)  
       - `Get_Content_Fedback` (sub-workflow that evaluates content quality, returns JSON score)  
     - Connect `Check History` Google Sheets node to verify duplicates.  
     - Use `OpenAI Chat Model` node with GPT-4.1-mini as language model.  
     - Use `Simple Memory` for session context.  
   - Connect inputs and outputs accordingly.

5. **Style Rewriting:**  
   - Add `Check Examples` Google Sheets node to fetch past posts for style examples.  
   - Add `Langchain Agent` node (`Rewriter`) configured to rewrite posts matching user style, using examples.  
   - Use `OpenAI Chat Model1` GPT-4.1-mini for rewriting.  
   - Use `Simple Memory1` for context.  
   - Connect AI Content creator output to Rewriter input.

6. **Discord Approval:**  
   - Add `Discord` node (`Send post for approval`) configured to send `{{ $json.output }}` to a specified Discord channel. Enable `sendAndWait` with double approval type.  
   - Add `If` node (`If2`) to check if Discord approval is true.  
   - If approved, proceed to `Post on X` Twitter node; if disapproved, send `Try Again` Discord message with approval options.  
   - Add another `If` node (`If1`) to check retry decision; if retry, send confirmation and loop back to content creation; if stop, send end of workflow message.

7. **Publishing and Logging:**  
   - `Post on X` node publishes post text to Twitter via OAuth2 credentials.  
   - `Add post to examples` Google Sheets appends to examples tab.  
   - `Update history` Google Sheets appends to history tab (post, timestamp, published flag).  
   - `Discord confirm` sends confirmation message.

8. **Credentials Setup:**  
   - Configure OpenAI API credentials for all OpenAI nodes.  
   - Set Google Sheets OAuth2 credentials for Google Sheets nodes.  
   - Configure Discord Bot API credentials and specify correct guild and channel IDs.  
   - Set Twitter OAuth2 credentials for posting.  
   - Configure Notion credentials if used.

9. **Sub-workflows:**  
   - Create `Get Brand Brief` sub-workflow returning brand brief text.  
   - Create `Get Content Feedback` sub-workflow that takes post + brief and returns JSON with feedback description and score (0–1).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| # Idea creator  \n## AI Agent who finds relevant ideas for posts on social media. He finds ideas with help of the brand brief.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note near `Idea creator` node                                                                                      |
| # Content creator  \n## Writes the first draft of a post based on the brand brief, with feedback on which post will fit best.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note near `AI Content creator` node                                                                                 |
| # Rewriter  \n## This agent rewrites posts to match your style using examples from a Google Sheet.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note near `Rewriter` node                                                                                           |
| # Additional approval  \n## Sends the post for approval in a Discord server, or you can use Telegram—whichever you prefer. This step is optional.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note near `Send post for approval` node                                                                             |
| # Post On X(Twitter)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note near `Post on X` node                                                                                          |
| # Update History and Examples                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note near `Add post to examples` and `Update history` nodes                                                        |
| # Confirm of End Workflow                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note near `Discord confirm` node                                                                                    |
| # Sub Workflow Get Brand Brief  \n## Move to a separate workflow, add the trigger "When Executed by Another Workflow", and connect it to the main workflow "AI Content Creator".                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note near `Get Brief` and `When Executed by Another Workflow` node                                                 |
| # Sub Workflow Get Content Feedback  \n## Move to a separate workflow, add the trigger "When Executed by Another Workflow", and connect it to the main workflow "AI Content Creator".                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Sticky Note near `Get_Content_Fedback` node                                                                               |
| # ✅ AI X Post Creator – Quick Start Guide  \nThis workflow automatically generates viral posts for your personal brand on X (Twitter).  \nIt creates a content idea based on your brand brief, writes a short post, evaluates the post’s alignment with your tone, checks if it was published before (via Google Sheets), rewrites if needed, sends it for approval on Discord, publishes to X, and updates history logs.  \n\n---\n\n## 🔧 What You Need to Replace\n\n| Placeholder              | Where to Update                                                       |\n|--------------------------|------------------------------------------------------------------------|\n| **[YOUR NAME]**          | Prompt nodes: `AI Content creator`, `Rewriter`, `OpenAI`              |\n| **[YOUR URL]**           | Node: `Notion` (if you use Notion for brief)                          |\n| **OpenAi API**           | All `OpenAI` nodes – add your actual OpenAI credentials               |\n| **Google Sheets **       | All Google Sheets nodes – connect your real spreadsheet               |\n| **Discord Bot **         | Discord nodes – replace with your bot token & correct channel IDs     |\n| **X**                    | Twitter node – add your real X (Twitter) OAuth2 credentials           |\n| **Get Brand Brief**      | Sub-workflow ID – must return brand brief as text                     |\n| **Get Content Feedback** | Sub-workflow ID – must return post evaluation in JSON (score + notes) |\n\n---\n\n## 🧩 Required Subworkflows\n\n### 1. **Get Brand Brief**  \nCreate a sub-workflow that returns a brand tone/voice as a single `text` field.\n\n### 2. **Get Content Feedback**  \nCreate another sub-workflow that takes a post and brand brief, then returns:\n\n```json\n{\n  \"description\": \"Short evaluation of the post\",\n  \"score\": 0.0 – 1.0\n}\n | See Sticky Note at workflow top right area                                    |

---

_Disclaimer: The provided text derives exclusively from an automated workflow created with n8n, respecting all current content policies, legal, and ethical guidelines. All data manipulated is legal and public._