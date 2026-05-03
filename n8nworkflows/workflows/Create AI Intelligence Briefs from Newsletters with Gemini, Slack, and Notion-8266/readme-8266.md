Create AI Intelligence Briefs from Newsletters with Gemini, Slack, and Notion

https://n8nworkflows.xyz/workflows/create-ai-intelligence-briefs-from-newsletters-with-gemini--slack--and-notion-8266


# Create AI Intelligence Briefs from Newsletters with Gemini, Slack, and Notion

### 1. Workflow Overview

This n8n workflow automates the generation of an AI-powered intelligence brief from your daily newsletters. It is designed for professionals who need to efficiently extract actionable insights from multiple newsletter emails, filtering signal from noise through AI analysis. The workflow fetches labeled Gmail newsletters received in the last 24 hours, combines them, processes their content using advanced AI models with a custom operational lens, and delivers a formatted brief to Slack. Optionally, it saves content-inspiring questions extracted by AI into Notion, and can leverage an external research tool (Perplexity) for additional context.

Logical blocks:

- **1.1 Input Reception & Preparation:** Collects recent labeled newsletters from Gmail and filters empty results.
- **1.2 AI Analysis & Processing:** Aggregates newsletter content, sends it to a Gemini-based AI agent with a detailed prompt and context, applies structured output parsing, and maintains conversation memory for context continuity.
- **1.3 Output Formatting & Delivery:** Formats the AI-generated insights into Slack Block Kit structure, sends the brief to Slack, splits out content questions, and optionally saves them to Notion.
- **1.4 Scheduling & Configuration:** Defines the daily trigger for the workflow and contains customizable business and audience configuration parameters.
- **1.5 Optional Research Integration:** Invokes Perplexity research tool if additional context is needed by the AI.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preparation

**Overview:**  
This block fetches Gmail emails labeled with a user-defined label from the past 24 hours and ensures the workflow proceeds only if emails were found.

**Nodes Involved:**  
- Daily Morning Trigger  
- Configuration  
- Get Labeled Newsletters  
- Filter Out Empty Results  

**Node Details:**

- **Daily Morning Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow daily at 8am UTC (cron: `0 8 * * *`)  
  - Configuration: Runs once every day at the set time; timezone can be adjusted by modifying the cron expression  
  - Input: None (trigger node)  
  - Output: Starts the workflow by triggering Configuration node  
  - Potential Failures: Cron misconfiguration or system downtime may cause missed runs  

- **Configuration**  
  - Type: Set node  
  - Role: Holds customizable parameters for business context, audience, filtering criteria, and output preferences  
  - Configuration: User sets values like recipient_name, company_name, relevance_filter, Slack channel name, notion database ID, content pillars, etc.  
  - Input: Trigger from Daily Morning Trigger  
  - Output: Feeds configuration data downstream  
  - Edge Cases: Missing or incorrect configuration values may cause AI prompt or output formatting errors  

- **Get Labeled Newsletters**  
  - Type: Gmail node  
  - Role: Fetches up to 20 emails from the last 24 hours with a specified Gmail label  
  - Configuration: Uses OAuth2 credentials; label ID must be set to the Gmail label containing newsletters; filtering by receivedAfter (last 1 day)  
  - Inputs: Configuration node output  
  - Outputs: Array of emails with fields including headers.from, subject, and textAsHtml  
  - Potential Failures: Authentication errors, Gmail API quota limits, or incorrect label ID can cause failure or empty results  

- **Filter Out Empty Results**  
  - Type: Filter node  
  - Role: Checks if any emails were retrieved; blocks workflow if none found  
  - Configuration: Condition checks if the fetched emails array has at least one item with a valid ID  
  - Input: Output of Get Labeled Newsletters  
  - Output: Passes data onward only if non-empty  
  - Edge Cases: Strict filter may block workflow unnecessarily if emails have malformed or missing IDs  

---

#### 1.2 AI Analysis & Processing

**Overview:**  
Aggregates the newsletter emails into a single data structure, sends the content to a Gemini AI agent with a complex prompt defining operational and business context, parses the structured AI response, and maintains memory for continuity.

**Nodes Involved:**  
- Combine All Newsletters  
- OpenRouter Chat Model  
- AI Newsletter Analyst  
- Perplexity Research Tool (optional)  
- Structured Output Parser  
- Conversation Memory  
- Output Parser Model  

**Node Details:**

- **Combine All Newsletters**  
  - Type: Aggregate node  
  - Role: Combines multiple email items into a single item for AI input  
  - Configuration: Aggregates fields `headers.from`, `subject`, and `textAsHtml` into a single `newsletter` field  
  - Input: Filter Out Empty Results node  
  - Output: Single aggregated item containing all newsletters  
  - Failure Modes: If input data is malformed or missing fields, aggregation may be incomplete  

- **OpenRouter Chat Model**  
  - Type: LangChain Chat Model (OpenRouter)  
  - Role: Provides the AI language model (Google Gemini 2.5 Flash) used for newsletter content analysis  
  - Configuration: Uses OpenRouter API credentials; model set to `google/gemini-2.5-flash`  
  - Input: Feeds AI prompt and context from AI Newsletter Analyst node  
  - Output: AI-generated raw text response  
  - Edge Cases: API rate limits, network errors, or quota exhaustion can cause failures  

- **AI Newsletter Analyst**  
  - Type: LangChain Agent node  
  - Role: Core AI node that analyzes combined newsletters with a detailed prompt including business context, audience, and filtering instructions; generates executive summary, content questions, and hidden patterns  
  - Configuration:  
    - Uses dynamic expressions pulling values from the Configuration node for personalized context  
    - Custom system message defines operational lenses and filtering criteria  
    - Retries enabled for robustness  
    - Connected to OpenRouter Chat Model for generation, Perplexity Research Tool for optional context, Structured Output Parser for JSON parsing, and Conversation Memory for context persistence  
  - Input: Aggregated newsletter data  
  - Output: Structured AI analysis JSON containing executive_summary, content_questions, hidden_patterns, and metadata  
  - Edge Cases: Prompt expression failures, AI hallucination, incomplete parsing results, or empty AI output  

- **Perplexity Research Tool**  
  - Type: Custom research API node  
  - Role: Optional external tool providing additional context if AI agent requests it  
  - Configuration: Uses Perplexity API with sonar-pro model  
  - Input: Triggered as AI tool by AI Newsletter Analyst node  
  - Output: Research results feeding back to AI agent  
  - Edge Cases: API limits, connectivity issues, or irrelevant results  

- **Structured Output Parser**  
  - Type: LangChain Output Parser  
  - Role: Ensures AI output adheres to a strict JSON schema for consistent downstream processing  
  - Configuration: Auto-fixes minor format issues; schema enforces presence of executive_summary, content_questions, hidden_patterns, and metadata fields  
  - Input: AI raw output from OpenRouter Chat Model  
  - Output: Parsed and validated structured JSON  
  - Edge Cases: Parser may fail if AI output is too divergent or unstructured  

- **Conversation Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains AI conversation context across runs, preserving last 7 interactions for better continuity in analysis  
  - Configuration: Context window length = 7  
  - Input: Connected as AI memory to AI Newsletter Analyst  
  - Output: Updated memory context  
  - Edge Cases: Memory overflow or corruption could confuse AI context  

- **Output Parser Model**  
  - Type: LangChain Chat Model (OpenRouter)  
  - Role: Secondary model instance used for output parsing and validation  
  - Configuration: Same Gemini 2.5 Flash model and credentials as primary AI model  
  - Input: Used in conjunction with Structured Output Parser  
  - Output: Parsed structured output for AI Newsletter Analyst node  

---

#### 1.3 Output Formatting & Delivery

**Overview:**  
Formats the AI analysis into Slack Block Kit JSON, sends the intelligence brief to Slack, splits out content questions for separate processing, and optionally saves these questions as pages in Notion.

**Nodes Involved:**  
- Format for Slack  
- Send to Slack  
- Split Out Questions  
- Save Questions to Notion  

**Node Details:**

- **Format for Slack**  
  - Type: Code node (JavaScript)  
  - Role: Converts structured AI output into Slack Block Kit message format, including header, TL;DR, key developments, content questions, emerging patterns, stats, and footer  
  - Configuration:  
    - Runs once per execution (mode: runOnceForEachItem)  
    - Uses helper functions to format dates and truncate text to Slack limits  
    - Dynamically builds rich text sections and quote blocks for developments, questions, and patterns  
    - Reads configuration values for branding and footer messages  
  - Input: AI Newsletter Analyst output (structured JSON)  
  - Output: JSON stringified Slack blocks and meta info (counts, flags)  
  - Edge Cases: Slack block size limits, malformed AI data, and empty content gracefully handled with fallback text  

- **Send to Slack**  
  - Type: Slack node  
  - Role: Posts the formatted intelligence brief as a Slack message with blocks  
  - Configuration:  
    - Uses OAuth2 Slack credentials  
    - Sends to a configured Slack channel ID  
    - Message type is block to leverage Slack Block Kit formatting  
  - Input: Block Kit JSON from Format for Slack  
  - Output: Slack API response  
  - Edge Cases: Slack API rate limits, invalid channel ID, or auth tokens may cause message failure  

- **Split Out Questions**  
  - Type: Split Out node  
  - Role: Separates content questions array into individual items for processing and saving  
  - Configuration: Splits on `output.content_questions` field from AI output  
  - Input: AI Newsletter Analyst output  
  - Output: Individual question JSON objects  
  - Edge Cases: Empty or missing content_questions array results in no downstream processing  

- **Save Questions to Notion**  
  - Type: Notion node  
  - Role: Creates new pages in a configured Notion database for each content question  
  - Configuration:  
    - Uses Notion API credentials  
    - Maps question text to page title, adds context and content pillar as rich text blocks, and records generation date  
    - Database ID must be configured  
  - Input: Individual questions from Split Out Questions node  
  - Output: Notion API response per page created  
  - Edge Cases: Missing database ID, API rate limits, or permission errors in Notion  

---

#### 1.4 Scheduling & Configuration

**Overview:**  
Defines workflow execution frequency and centralizes all customizable parameters.

**Nodes Involved:**  
- Daily Morning Trigger  
- Configuration  

**Node Details:**  
Already detailed above in section 1.1.

---

#### 1.5 Optional Research Integration

**Overview:**  
Provides supplementary research context to AI analyses if requested during processing.

**Nodes Involved:**  
- Perplexity Research Tool  

**Node Details:**  
Already detailed above in section 1.2.

---

### 3. Summary Table

| Node Name              | Node Type                       | Functional Role                                      | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                                  |
|------------------------|--------------------------------|-----------------------------------------------------|---------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Daily Morning Trigger   | Schedule Trigger               | Triggers workflow daily at 8am UTC                   | -                         | Configuration               | Adjust cron expression for your timezone (e.g. 8am Pacific, Eastern)                                         |
| Configuration          | Set                           | Holds customizable business and audience parameters | Daily Morning Trigger      | Get Labeled Newsletters      | 📝 CUSTOMIZE THESE VALUES for your business and use case                                                     |
| Get Labeled Newsletters | Gmail                         | Fetches recent labeled newsletters                    | Configuration             | Filter Out Empty Results     | Replace YOUR_GMAIL_LABEL_ID with your actual Gmail label ID                                                  |
| Filter Out Empty Results| Filter                        | Filters out empty email results                        | Get Labeled Newsletters    | Combine All Newsletters      | Only proceeds if emails were found                                                                           |
| Combine All Newsletters | Aggregate                     | Aggregates multiple emails into single item           | Filter Out Empty Results   | AI Newsletter Analyst       | Combines all newsletter emails into a single item for AI processing                                          |
| OpenRouter Chat Model   | LangChain Chat Model          | Primary AI language model (Gemini)                    | AI Newsletter Analyst      | AI Newsletter Analyst       |                                                                                                              |
| AI Newsletter Analyst   | LangChain Agent               | Core AI node analyzing newsletters and generating brief | Combine All Newsletters, OpenRouter Chat Model, Perplexity Research Tool, Structured Output Parser, Conversation Memory | Format for Slack, Split Out Questions | AI agent that analyzes newsletters and generates insights based on your configuration                        |
| Perplexity Research Tool| Perplexity API integration    | Optional tool for additional AI research context      | AI Newsletter Analyst     | AI Newsletter Analyst       | Optional tool for the AI agent to gather additional context                                                  |
| Structured Output Parser| LangChain Output Parser       | Parses and validates AI output JSON                    | Output Parser Model       | AI Newsletter Analyst       | Ensures AI output follows the expected JSON structure                                                        |
| Output Parser Model     | LangChain Chat Model          | AI model instance used for output parsing             | -                         | Structured Output Parser    |                                                                                                              |
| Conversation Memory     | LangChain Memory Buffer Window| Maintains AI context across interactions               | AI Newsletter Analyst     | AI Newsletter Analyst       | Maintains context across multiple AI interactions                                                            |
| Format for Slack        | Code                         | Formats AI analysis into Slack Block Kit JSON          | AI Newsletter Analyst     | Send to Slack               | Converts AI analysis to rich Slack Block Kit format                                                         |
| Send to Slack           | Slack                        | Sends formatted brief to Slack channel                 | Format for Slack          | -                           | Replace YOUR_SLACK_CHANNEL_ID with your channel ID                                                           |
| Split Out Questions     | Split Out                    | Splits content questions array for individual handling | AI Newsletter Analyst     | Save Questions to Notion    | Separates content questions for individual processing                                                       |
| Save Questions to Notion| Notion                       | Saves individual content questions as Notion pages    | Split Out Questions       | -                           | Replace YOUR_NOTION_DATABASE_ID with your database ID (optional)                                            |
| Sticky Note             | Sticky Note                  | AI-powered newsletter intelligence brief overview      | -                         | -                           | ## 🤖 AI-powered Newsletter Intelligence Brief (see detailed notes in workflow)                             |
| Sticky Note1            | Sticky Note                  | Setup instructions for Gmail and credentials           | -                         | -                           | ## ⚙️ Setup Steps (Gmail, Configuration, Credentials)                                                        |
| Sticky Note2            | Sticky Note                  | Required configuration updates and timezone notes      | -                         | -                           | ## 🔧 Required Configuration (Gmail label, Slack channel, Notion DB ID, trigger timezone)                    |
| Sticky Note3            | Sticky Note                  | Important notes on security, cost, and customization    | -                         | -                           | ## 🚨 Important Notes (API key security, cost estimates, customization tips)                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set cron expression to `0 8 * * *` (runs daily at 8am UTC)  
   - Connect output to Configuration node  

2. **Create Configuration Node**  
   - Type: Set  
   - Define these string parameters with your business values:  
     - recipient_name (e.g., "Tech Team")  
     - company_name  
     - business_context  
     - target_audience  
     - target_audience_description  
     - company_size_range  
     - relevance_filter (key question to filter relevant news)  
     - content_platforms (e.g., "LinkedIn posts, internal updates")  
     - content_pillars (main content areas, 3 pillars)  
     - additional_include_criteria  
     - additional_exclude_criteria  
     - brief_title (Slack brief title)  
     - footer_message (brief footer text)  
   - Connect output to Get Labeled Newsletters node  

3. **Create Get Labeled Newsletters Node**  
   - Type: Gmail  
   - Operation: getAll emails  
   - Limit: 20  
   - Filters: labelIds set to your Gmail newsletter label ID  
   - Filter emails receivedAfter `={{ $today.minus(1, 'days') }}`  
   - Credentials: Configure OAuth2 Gmail account  
   - Connect output to Filter Out Empty Results node  

4. **Create Filter Out Empty Results Node**  
   - Type: Filter  
   - Condition: Check if `{{$json.id}}` exists (string operation "exists")  
   - Connect output to Combine All Newsletters node  

5. **Create Combine All Newsletters Node**  
   - Type: Aggregate  
   - Aggregate all items into single item  
   - Include fields: headers.from, subject, textAsHtml  
   - Destination field: `newsletter`  
   - Connect output to AI Newsletter Analyst node  

6. **Create OpenRouter Chat Model Node**  
   - Type: LangChain Chat Model (OpenRouter)  
   - Model: google/gemini-2.5-flash  
   - Credentials: OpenRouter API key  
   - No direct connections; connected as AI language model input to AI Newsletter Analyst node  

7. **Create Perplexity Research Tool Node (optional)**  
   - Type: Perplexity API  
   - Model: sonar-pro  
   - Credentials: Perplexity API key  
   - Connected as AI tool input to AI Newsletter Analyst node  

8. **Create Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - AutoFix: enabled  
   - Provide JSON schema example matching AI expected output structure  
   - Connected as AI output parser input to AI Newsletter Analyst node  

9. **Create Output Parser Model Node**  
   - Type: LangChain Chat Model (OpenRouter)  
   - Model: google/gemini-2.5-flash  
   - Credentials: OpenRouter API key  
   - Connected to Structured Output Parser node  

10. **Create Conversation Memory Node**  
    - Type: LangChain Memory Buffer Window  
    - Context Window Length: 7  
    - Connected as AI memory input to AI Newsletter Analyst node  

11. **Create AI Newsletter Analyst Node**  
    - Type: LangChain Agent  
    - Parameters:  
      - Text prompt includes dynamic expressions referencing Configuration node values (business context, audience, relevance filter, etc.)  
      - System message defines AI's operational perspective and filtering logic  
      - Enable retry on failure  
    - Connect inputs:  
      - AI language model: OpenRouter Chat Model  
      - AI tool: Perplexity Research Tool (optional)  
      - AI output parser: Structured Output Parser  
      - AI memory: Conversation Memory  
    - Connect outputs to Format for Slack and Split Out Questions nodes  

12. **Create Format for Slack Node**  
    - Type: Code (JavaScript)  
    - Paste the provided JS code that formats AI analysis into Slack Block Kit JSON (see node details)  
    - Connect output to Send to Slack node  

13. **Create Send to Slack Node**  
    - Type: Slack  
    - Authentication: OAuth2 Slack account  
    - Message type: Block  
    - Channel ID: your Slack channel ID  
    - Text: "Daily Newsletter Brief" (fallback)  
    - Blocks UI: use `{{$json.blocksUi}}` from Format for Slack node  
    - Connect no further outputs  

14. **Create Split Out Questions Node**  
    - Type: Split Out  
    - Field to split: `output.content_questions` (from AI Newsletter Analyst output)  
    - Connect outputs to Save Questions to Notion node  

15. **Create Save Questions to Notion Node (optional)**  
    - Type: Notion  
    - Resource: Database Page  
    - Database ID: your Notion database ID  
    - Title: `{{$json.question}}`  
    - Blocks: Add rich text blocks for context, content pillar, and generation date  
    - Credentials: Notion API key  
    - Connect no further outputs  

16. **Validation and Testing**  
    - Verify credentials for Gmail, OpenRouter, Slack, Notion, and Perplexity (if used) are configured in n8n  
    - Run the workflow manually to test end-to-end execution  
    - Adjust Configuration parameters and cron schedule as needed  
    - Monitor for errors in API calls and expression evaluations  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This template automatically analyzes newsletters and creates an AI-powered intelligence brief to filter signal from noise, ideal for busy professionals, content creators, and team leaders. Requires Gmail label setup, OpenRouter API key, Slack workspace, and optionally Notion and Perplexity accounts. See detailed setup and customization instructions in the sticky notes within the workflow.                                                                                                                                                                                                                                                                                                                                                  | Overview and user guidance embedded as Sticky Notes in the workflow                                         |
| Gmail label creation instructions: <https://support.google.com/mail/thread/208327636/how-do-i-automatically-label-emails?hl=en>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Gmail labeling tutorial                                                                                      |
| Security note: Never hardcode API keys; always use n8n credential system for OAuth and API integrations. Costs per run: OpenRouter approx. $0.01-0.05; Perplexity approx. $0.01 per query; Slack and Notion have free tier limits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Security and cost considerations                                                                             |
| Customization tips: Start with 3-5 newsletters for initial tests, refine Configuration values for relevance, adjust AI prompt as needed, and review Slack formatting for clarity. AI output quality improves with clear, specific instructions and iterative tuning.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Best practices for tuning and using the workflow                                                             |

---

This completes the comprehensive analysis and reference documentation for the "Create AI Intelligence Briefs from Newsletters with Gemini, Slack, and Notion" n8n workflow.