Generate 360 social media reports with AI - Bright Data MCP

https://n8nworkflows.xyz/workflows/generate-360-social-media-reports-with-ai---bright-data-mcp-8653


# Generate 360 social media reports with AI - Bright Data MCP

### 1. Workflow Overview

This workflow automates the process of generating comprehensive 360-degree social media intelligence reports about individuals by integrating Bright Data MCP tools with advanced AI agents (OpenAI GPT-4). It targets professionals who require detailed social media presence analysis for lead qualification, due diligence, recruitment, or partnership evaluation.

The workflow is logically divided into the following functional blocks:

- **1.1 Data Input & Validation:** Collects person and search parameters via a web form and validates the input, extracting known social media handles and generating search queries.
- **1.2 Profile Discovery Phase:** Uses an AI-powered discovery agent that performs multi-platform searches, profile verification, and confidence scoring.
- **1.3 Cross-Platform Verification:** Validates and analyzes discovered profiles for consistency, trust scoring, and flags potential issues.
- **1.4 Platform-Specific Deep Analysis:** Routes validated profiles by platform to specialized prompt builders and agents that leverage Bright Data MCP PRO tools for in-depth analysis.
- **1.5 Intelligence Report Generation:** Aggregates platform analyses and synthesizes a comprehensive intelligence report using GPT-4.
- **1.6 Report Formatting and Delivery:** Converts the report from markdown to HTML, creates and updates a Google Doc with styled content, and redirects the user to the final document.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Input & Validation

- **Overview:**  
  Collects user-submitted person details and search preferences via a form trigger node, then validates and enriches this input data by extracting known social media handles, generating name variations, and building platform-specific search queries.

- **Nodes Involved:**  
  - Social Media Research Form  
  - Input Validator  

- **Node Details:**  

  - **Social Media Research Form**  
    - Type: Form Trigger  
    - Role: Entry point; collects full name, email, company, location, known social handles, search depth, and platforms to search.  
    - Configuration: Web form with required fields for full name, search depth, and platforms to search; supports multi-select for platforms and text area for known handles.  
    - Connections: Outputs to Input Validator.  
    - Edge Cases: Missing required fields triggers validation errors; user input errors for handle formats.

  - **Input Validator**  
    - Type: Code (JavaScript)  
    - Role: Processes form inputs, extracts known handles using regex patterns, generates name variations for name ambiguity, and builds detailed search queries per platform.  
    - Key Logic:  
      - Parses known social handles from free text input.  
      - Generates multiple name variants to improve search matching.  
      - Builds Google-style search queries tailored to each platform.  
      - Validates required fields and sets search depth parameters.  
    - Connections: Outputs validated and enriched data to the discovery agent.  
    - Edge Cases: Incorrect handle formats, empty required fields, malformed input data.  
    - Error Handling: Collects validation errors and sets an isValid flag.

#### 2.2 Profile Discovery Phase

- **Overview:**  
  Uses an AI-powered agent to search and verify social media profiles across selected platforms using generated queries and known handles, applying confidence scoring and cross-referencing.

- **Nodes Involved:**  
  - discovery agent  
  - Bright Data MCP  
  - OpenAI Chat Model (GPT-4.1-mini)  
  - Simple Memory  

- **Node Details:**  

  - **discovery agent**  
    - Type: Langchain AI Agent  
    - Role: Executes search and verification workflow using instructions based on input data; utilizes Bright Data MCP and OpenAI GPT-4.1-mini as tools.  
    - Configuration: System prompt defines agent’s role as a Social Media Discovery Agent, detailed instructions for search and verification phases, confidence scoring guidelines, and output format.  
    - Inputs: Validated data from Input Validator.  
    - Outputs: Structured JSON of discovered profiles with confidence scores and metadata.  
    - Connections: Output feeds into Output Validator.  
    - Edge Cases: Network issues, API rate limits, incomplete data affecting search quality, AI parsing errors.

  - **Bright Data MCP**  
    - Type: AI Tool (Bright Data MCP Client)  
    - Role: Provides data extraction and scraping capabilities via the Bright Data Market Client Platform for profile data enrichment.  
    - Configuration: Endpoint URL with placeholders for token and unlocker code (must be updated by user).  
    - Connections: Used as a tool by discovery agent and research agent.  
    - Edge Cases: Authentication failures if token/unlocker invalid, network timeouts.

  - **OpenAI Chat Model (GPT-4.1-mini)**  
    - Type: Language Model (OpenAI GPT-4 variant)  
    - Role: Supports the discovery agent’s reasoning and data parsing.  
    - Credentials: Requires OpenAI API key (configured in credentials).  
    - Edge Cases: API quota limits, model downtime.

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains session memory for discovery agent to improve context and continuity during searches.  
    - Configuration: Uses execution id as session key.  
    - Edge Cases: Memory overflow or session expiration.

#### 2.3 Cross-Platform Verification

- **Overview:**  
  Validates discovered profiles by analyzing consistency across platforms, calculates trust scores, flags data inconsistencies, and determines if profiles are reliable enough to proceed.

- **Nodes Involved:**  
  - Output validator  
  - Profiles validator  

- **Node Details:**  

  - **Output validator**  
    - Type: Code (JavaScript)  
    - Role: Parses AI agent output, normalizes platform names, extracts profiles, applies fallback URL extraction if needed, and prepares structured data for validation.  
    - Key Logic: JSON parsing with error handling, confidence score aggregation, recommended primary profile assignment.  
    - Outputs: Structured profiles and metadata.  
    - Edge Cases: Malformed AI output, missing profiles, inconsistent confidence data.

  - **Profiles validator**  
    - Type: Code (JavaScript)  
    - Role: Performs cross-platform consistency checks on names, companies, and locations; generates confidence and trust scores; creates verification matrices; flags potential issues; suggests further actions.  
    - Key Logic:  
      - Computes consistency scores for name, company, location across profiles.  
      - Identifies primary identity based on highest confidence profile.  
      - Flags warnings for inconsistent information or low confidence.  
      - Calculates overall trust score combining confidence, consistency, and platform coverage.  
    - Outputs: Validated profiles, trust scores, validation flags, suggested next steps, and a proceedToAnalysis boolean.  
    - Edge Cases: Sparse profile data, conflicting information, low confidence profiles.

#### 2.4 Platform-Specific Deep Analysis

- **Overview:**  
  Splits the validated profiles per platform, routes each to platform-specific prompt builders, and triggers specialized research agents using Bright Data MCP to perform detailed profile analysis.

- **Nodes Involved:**  
  - Split Platforms into Items  
  - Switch  
  - LinkedIn Prompt Builder  
  - Twitter/X Prompt Builder  
  - Instagram Prompt Builder  
  - YouTube Prompt Builder  
  - GitHub Prompt Builder  
  - TikTok Prompt Builder  
  - Research Agent  
  - OpenAI Chat Model1 (GPT-4.1-mini)  
  - Simple Memory1  
  - Merge  

- **Node Details:**  

  - **Split Platforms into Items**  
    - Type: Code (JavaScript)  
    - Role: Transforms the profiles object into individual items per platform for parallel processing.  
    - Outputs: One item per platform profile for analysis.  
    - Edge Cases: Profiles missing URLs skipped.

  - **Switch**  
    - Type: Switch (conditional routing)  
    - Role: Routes each platform item to the appropriate platform-specific prompt builder node.  
    - Conditions: Matches platform strings (linkedin, twitter, instagram, youtube, github, tiktok).  
    - Edge Cases: Unmatched platforms are not routed.

  - **Platform Prompt Builders (LinkedIn, Twitter/X, Instagram, YouTube, GitHub, TikTok)**  
    - Type: Set Nodes  
    - Role: Prepare platform-specific system and user prompts, define analysis fields and Bright Data MCP PRO tools to use, and pass relevant profile details.  
    - Configuration: Each has a tailored system prompt describing the analyst role and a detailed user prompt specifying analysis focus areas and MCP tools to leverage.  
    - Outputs: Structured prompt data for the Research Agent.  
    - Edge Cases: Missing profile data limits analysis depth.

  - **Research Agent**  
    - Type: Langchain AI Agent  
    - Role: Executes platform-specific analysis by invoking Bright Data MCP PRO tools and OpenAI language model to generate detailed profile insights.  
    - Inputs: Prompts from platform prompt builders.  
    - Outputs: Analysis text per platform.  
    - Edge Cases: API limits, tool errors.

  - **OpenAI Chat Model1 (GPT-4.1-mini)**  
    - Type: Language Model  
    - Role: Provides language understanding and generation for Research Agent.  
    - Credentials: OpenAI API key.  
    - Edge Cases: Model availability.

  - **Simple Memory1**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains session context for each platform’s research agent using a composite session key.  
    - Edge Cases: Memory size limits.

  - **Merge**  
    - Type: Merge (multiple inputs)  
    - Role: Combines outputs from all platform research agents into a single stream for aggregation.

#### 2.5 Intelligence Report Generation

- **Overview:**  
  Aggregates all platform analyses and synthesizes a structured, comprehensive intelligence report using an advanced GPT-4.1 LLM chain with a detailed prompt framework.

- **Nodes Involved:**  
  - Aggregate analyses  
  - Basic LLM Chain  
  - OpenAI Chat Model2 (GPT-4.1)  

- **Node Details:**  

  - **Aggregate analyses**  
    - Type: Aggregate  
    - Role: Combines all platform analysis items into a single array under the field "platforms".  
    - Output: Aggregated data for final report synthesis.

  - **Basic LLM Chain**  
    - Type: Langchain Chain LLM  
    - Role: Uses a complex, multi-section prompt instructing GPT-4.1 to generate a detailed social media intelligence report covering executive summary, platform breakdowns, cross-platform analysis, professional assessment, content strategy, influence metrics, personality analysis, recommendations, risk, and contact intelligence.  
    - Inputs: Aggregated platform analyses.  
    - Outputs: Full textual intelligence report in markdown format.

  - **OpenAI Chat Model2 (GPT-4.1)**  
    - Type: Language Model  
    - Role: Supports the generation of the comprehensive intelligence report.  
    - Credentials: OpenAI API key.  
    - Edge Cases: Model rate limits, timeouts.

#### 2.6 Report Formatting and Delivery

- **Overview:**  
  Converts the generated markdown report to styled HTML, creates or updates a Google Doc with this content, and redirects the user to the final formatted report.

- **Nodes Involved:**  
  - Set Content  
  - Markdown to HTML  
  - Convert HTML to File  
  - Set MIME Type to HTML  
  - Create Empty Google Doc  
  - Merge2  
  - Update Doc with HTML  
  - Form  

- **Node Details:**  

  - **Set Content**  
    - Type: Set  
    - Role: Prepares the markdown text and filename for the report.  
    - Inputs: Report text from Basic LLM Chain.  
    - Outputs: Markdown and filename for downstream nodes.

  - **Markdown to HTML**  
    - Type: Markdown Converter  
    - Role: Converts markdown report into HTML format preserving emojis, tables, and formatting.

  - **Convert HTML to File**  
    - Type: Convert To File  
    - Role: Converts HTML content into a text-based file with a binary property for Google Drive upload.

  - **Set MIME Type to HTML**  
    - Type: Code  
    - Role: Ensures the binary file has MIME type "text/html" for Google Docs compatibility.

  - **Create Empty Google Doc**  
    - Type: Google Drive  
    - Role: Creates an empty Google Doc in a specified folder as a container for the report.  
    - Configuration: User must specify target folder ID; uses OAuth2 credentials.

  - **Merge2**  
    - Type: Merge  
    - Role: Combines the Google Doc creation output and the HTML file for update.

  - **Update Doc with HTML**  
    - Type: Google Drive  
    - Role: Overwrites the empty Google Doc content with the formatted HTML file, enabling rich formatting not supported by direct upload.  
    - Outputs: Google Doc metadata including webViewLink.

  - **Form**  
    - Type: Form Completion  
    - Role: Redirects the user to the Google Doc URL upon workflow completion.  
    - Configuration: Redirects via webViewLink field.

- **Edge Cases:**  
  - Google Drive API quota limits or permission errors.  
  - HTML formatting issues causing document display errors.  
  - Missing or invalid OAuth2 credentials.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                            | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                                        |
|-------------------------|----------------------------------|--------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Social Media Research Form | Form Trigger                     | Collects user input for social media search |                             | Input Validator             | ## 1. Data Input & Validation<br>[Learn more about Form Triggers](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.formtrigger/) |
| Input Validator          | Code                             | Validates and enriches input data          | Social Media Research Form   | discovery agent             |                                                                                                                                   |
| discovery agent          | Langchain Agent                  | Searches & verifies social media profiles  | Input Validator             | Output validator            | ## 2. Profile Discovery Phase<br>[Learn about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) |
| Bright Data MCP          | MCP Client Tool                  | Provides data extraction and scraping tools | discovery agent, Research Agent | discovery agent, Research Agent | ## ⚠️ SETUP REQUIRED (1/3)<br>Replace YOUR_BRIGHT_DATA_TOKEN_HERE and UNLOCKER_CODE_HERE<br>[Get your Bright Data API key here](https://get.brightdata.com/qsg36y0kkq70) |
| OpenAI Chat Model        | Language Model (GPT-4.1-mini)    | Supports AI reasoning and generation       | discovery agent             | discovery agent             |                                                                                                                                   |
| Simple Memory            | Memory Buffer Window             | Maintains session context for discovery    |                             | discovery agent             |                                                                                                                                   |
| Output validator         | Code                             | Parses and normalizes discovery output     | discovery agent             | Profiles validator          |                                                                                                                                   |
| Profiles validator       | Code                             | Validates profiles cross-platform, scores  | Output validator            | Split Platforms into Items  | ## 3. Cross-Platform Verification<br>Only profiles with >50% trust proceed to analysis.                                            |
| Split Platforms into Items | Code                           | Splits profiles by platform for analysis   | Profiles validator          | Switch                     |                                                                                                                                   |
| Switch                   | Switch                          | Routes profiles to platform-specific prompt builders | Split Platforms into Items | LinkedIn Prompt Builder, Twitter/X Prompt Builder, Instagram Prompt Builder, YouTube Prompt Builder, GitHub Prompt Builder, TikTok Prompt Builder | ## ⚡ Smart Routing:<br>Add new platforms by adding Switch case and prompt builder node.                                             |
| LinkedIn Prompt Builder  | Set                             | Builds LinkedIn-specific analysis prompt   | Switch (LinkedIn output)    | Research Agent             | ## 4. Platform-Specific Deep Analysis<br>Bright Data MCP PRO tools per platform                                                    |
| Twitter/X Prompt Builder | Set                             | Builds Twitter/X-specific analysis prompt  | Switch (Twitter output)     | Research Agent             |                                                                                                                                   |
| Instagram Prompt Builder | Set                             | Builds Instagram-specific analysis prompt  | Switch (Instagram output)   | Research Agent             |                                                                                                                                   |
| YouTube Prompt Builder   | Set                             | Builds YouTube-specific analysis prompt    | Switch (YouTube output)     | Research Agent             |                                                                                                                                   |
| GitHub Prompt Builder    | Set                             | Builds GitHub-specific analysis prompt     | Switch (GitHub output)      | Research Agent             |                                                                                                                                   |
| TikTok Prompt Builder    | Set                             | Builds TikTok-specific analysis prompt     | Switch (TikTok output)      | Research Agent             |                                                                                                                                   |
| Research Agent           | Langchain Agent                 | Performs platform-specific profile analysis | Platform Prompt Builders    | Merge                      |                                                                                                                                   |
| OpenAI Chat Model1       | Language Model (GPT-4.1-mini)    | Supports Research Agent                     | Research Agent             | Research Agent             |                                                                                                                                   |
| Simple Memory1           | Memory Buffer Window             | Session memory for platform research agents |                             | Research Agent             |                                                                                                                                   |
| Merge                    | Merge                           | Combines all platform analyses              | Research Agent (all)        | Aggregate analyses         |                                                                                                                                   |
| Aggregate analyses       | Aggregate                       | Aggregates platform analysis data           | Merge                      | Basic LLM Chain            |                                                                                                                                   |
| Basic LLM Chain          | Langchain Chain LLM             | Synthesizes comprehensive intelligence report | Aggregate analyses          | Set Content                | ## 5. Intelligence Report Generation<br>GPT-4.1 synthesizes multi-platform data                                                    |
| OpenAI Chat Model2       | Language Model (GPT-4.1)          | Supports final report generation             | Basic LLM Chain            | Basic LLM Chain            |                                                                                                                                   |
| Set Content              | Set                             | Prepares markdown report content and filename | Basic LLM Chain            | Markdown to HTML           |                                                                                                                                   |
| Markdown to HTML         | Markdown Converter              | Converts markdown to styled HTML             | Set Content                | Convert HTML to File       |                                                                                                                                   |
| Convert HTML to File     | Convert To File                 | Converts HTML text to binary file            | Markdown to HTML           | Set MIME Type to HTML      |                                                                                                                                   |
| Set MIME Type to HTML    | Code                            | Sets MIME type for Google Docs compatibility | Convert HTML to File       | Merge2                     |                                                                                                                                   |
| Create Empty Google Doc  | Google Drive                   | Creates new empty Google Doc in target folder | Set Content                | Merge2                     | ## ⚠️ SETUP REQUIRED (3/3)<br>Replace YOUR_FOLDER_ID_HERE or select folder for reports                                               |
| Merge2                   | Merge                           | Combines Google Doc file and HTML file       | Set MIME Type to HTML, Create Empty Google Doc | Update Doc with HTML       |                                                                                                                                   |
| Update Doc with HTML     | Google Drive                   | Updates Google Doc content with formatted HTML | Merge2                     | Form                       | ## 6. Report Delivery to Google Drive<br>Uses advanced HTML injection technique<br>[Original Template](https://n8n.io/workflows/5147-convert-markdown-content-to-google-docs-document-with-automatic-formatting/) |
| Form                     | Form Completion                | Redirects user to final Google Doc            | Update Doc with HTML       |                             |                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Social Media Research Form"**  
   - Node Type: Form Trigger  
   - Configure fields:  
     - Full Name (text, required)  
     - Email Address (text)  
     - Company/Organization (text)  
     - Location (text)  
     - Known Social Handles (textarea)  
     - Search Depth (dropdown: Basic, Comprehensive, Deep; required)  
     - Platforms to Search (checkbox multiple select: LinkedIn, Twitter/X, Instagram, GitHub, YouTube, TikTok, Facebook, Reddit, Pinterest, Snapchat; required)  
   - Set webhook ID for form submission.

2. **Add Code Node: "Input Validator"**  
   - Type: Code  
   - Paste JavaScript that:  
     - Extracts handles from Known Social Handles using regex.  
     - Generates name variations from Full Name.  
     - Builds search queries per platform using name variations, company, and location.  
     - Validates required fields and sets search parameters (depth, max results).  
   - Connect output from Social Media Research Form.

3. **Add Langchain Agent Node: "discovery agent"**  
   - Type: Langchain Agent  
   - Configure system message as Social Media Discovery Agent with detailed instructions on searching, verifying, scoring profiles, and output format.  
   - Use Bright Data MCP and OpenAI Chat Model (GPT-4.1-mini) as AI tools and language model respectively.  
   - Connect output from Input Validator.

4. **Add Bright Data MCP Node**  
   - Type: MCP Client Tool  
   - Set endpoint URL with your Bright Data token and unlocker code.  
   - Use server transport as httpStreamable.  
   - Connect as tool to discovery agent.

5. **Add OpenAI Chat Model Node**  
   - Type: OpenAI GPT-4.1-mini  
   - Set with your OpenAI API credentials.  
   - Connect as language model to discovery agent.

6. **Add Simple Memory Node: "Simple Memory"**  
   - Type: Langchain Memory Buffer Window  
   - Set session key to execution ID + suffix for discovery agent.  
   - Connect AI memory to discovery agent.

7. **Add Code Node: "Output validator"**  
   - Type: Code  
   - Parse discovery agent output, normalize platform names, extract profiles and confidence, fallback regex extraction from raw text, prepare structured output.  
   - Connect from discovery agent output.

8. **Add Code Node: "Profiles validator"**  
   - Type: Code  
   - Cross-check profiles for name, company, location consistency; calculate confidence and trust scores; generate verification matrix; add validation flags and suggested actions; determine if proceed to analysis.  
   - Connect from Output validator.

9. **Add Code Node: "Split Platforms into Items"**  
   - Split profiles object into individual items per platform for parallel processing.  
   - Connect from Profiles validator.

10. **Add Switch Node: "Switch"**  
    - Route items by platform string (linkedin, twitter, instagram, youtube, github, tiktok).  
    - Connect from Split Platforms into Items.

11. **Create Platform-Specific Prompt Builder Set Nodes:**  
    - For each platform (LinkedIn, Twitter/X, Instagram, YouTube, GitHub, TikTok):  
      - Create a Set node that assigns: platform, profileUrl, analysisDepth, systemPrompt, userPrompt, proTools (Bright Data MCP tools), analysisFields, handle, confidence, profileData.  
    - Connect each respective output from Switch.

12. **Add Langchain Agent: "Research Agent"**  
    - Configure to use platform-specific prompts and Bright Data MCP PRO tools.  
    - Use OpenAI Chat Model1 (GPT-4.1-mini) as language model.  
    - Connect outputs from all platform prompt builders as separate inputs.

13. **Add Simple Memory Node: "Simple Memory1"**  
    - Configure session key to execution ID + platform for per-platform memory.  
    - Connect AI memory input to Research Agent.

14. **Add Merge Node**  
    - Merge all Research Agent outputs into one stream.  
    - Connect from Research Agent.

15. **Add Aggregate Node: "Aggregate analyses"**  
    - Aggregate all platform analyses into a single array named "platforms".  
    - Connect from Merge.

16. **Add Langchain Chain LLM Node: "Basic LLM Chain"**  
    - Configure with a detailed prompt to generate a multi-section intelligence report (executive summary, platform breakdown, cross-platform analysis, professional assessment, content strategy, digital influence, personality, recommendations, risk, contact intelligence).  
    - Use OpenAI Chat Model2 (GPT-4.1) as language model.  
    - Connect from Aggregate analyses.

17. **Add Set Node: "Set Content"**  
    - Assign markdown text from Basic LLM Chain output to "markdown" field.  
    - Assign formatted filename with date/time.  
    - Connect from Basic LLM Chain.

18. **Add Markdown to HTML Node**  
    - Convert markdown content to HTML with emoji and table support.  
    - Connect from Set Content.

19. **Add Convert HTML to File Node**  
    - Convert HTML to a binary file for upload.  
    - Connect from Markdown to HTML.

20. **Add Code Node: "Set MIME Type to HTML"**  
    - Set MIME type to "text/html" on binary file.  
    - Connect from Convert HTML to File.

21. **Add Google Drive Node: "Create Empty Google Doc"**  
    - Operation: createFromText  
    - Name: use filename from Set Content  
    - Content: temporary placeholder text  
    - Folder: set target Google Drive folder ID  
    - Credentials: Google Drive OAuth2  
    - Connect from Set Content.

22. **Add Merge Node: "Merge2"**  
    - Combine outputs from Create Empty Google Doc and Set MIME Type to HTML nodes.  
    - Connect their outputs accordingly.

23. **Add Google Drive Node: "Update Doc with HTML"**  
    - Operation: update  
    - File ID: use the ID of the newly created Google Doc  
    - Input file: use HTML file binary  
    - Credentials: Google Drive OAuth2  
    - Connect from Merge2.

24. **Add Form Completion Node: "Form"**  
    - Operation: completion  
    - Redirect URL: use webViewLink from updated Google Doc  
    - Connect from Update Doc with HTML.

25. **Add Sticky Notes for Documentation and Setup Guidance:**  
    - Include overview, input section, discovery phase, verification, platform analysis, report generation, report delivery, setup requirements, testing guide, platform routing logic, Bright Data tools, output formats, success metrics, template attribution, and customization guides as non-executable nodes for user reference.

**Important Setup Steps:**  
- Replace placeholders in Bright Data MCP node endpoint URL with your actual API token and unlocker code.  
- Set your Google Drive folder ID in the "Create Empty Google Doc" node.  
- Configure OpenAI API credentials for all OpenAI Chat Model nodes.  
- Test workflow with your own profile using "Basic" search depth and a few platforms first.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow was created by Romuald Członkowski from AiAdvisors; visit [AiAdvisors website](https://www.aiadvisors.pl/en) to collaborate.                                                                                                             | Template Attribution Sticky Note                                                                                                                                                                 |
| The workflow relies on Bright Data MCP PRO tools for data extraction; user must obtain a Bright Data API token and unlocker code from [Bright Data Quick Start Guide](https://get.brightdata.com/qsg36y0kkq70).                                         | Bright Data Tools Sticky Note                                                                                                                                                                   |
| The final report is generated as a Google Doc with advanced HTML injection to preserve complex formatting, adapted from Roman Rozenberger's approach: [Original Template](https://n8n.io/workflows/5147-convert-markdown-content-to-google-docs-document-with-automatic-formatting/) | Report Delivery Sticky Note                                                                                                                                                                     |
| Users should customize by adding new platforms in the Switch node, modifying prompt builders, adjusting confidence thresholds, or changing final report format.                                                                                       | Customization Guide Sticky Note                                                                                                                                                                 |
| Testing recommendations include using known profiles, verifying confidence scores, and testing edge cases like common names, single names, or inactive accounts.                                                                                         | Testing Guide Sticky Note                                                                                                                                                                       |
| The workflow uses n8n-mcp and Claude Desktop as co-pilots during development; learn more at [n8n-mcp website](https://www.n8n-mcp.com/).                                                                                                               | Built with n8n-mcp Sticky Note                                                                                                                                                                 |
| The workflow requires a self-hosted or cloud n8n instance with proper credentials for Bright Data, OpenAI, and Google Drive OAuth2 connections.                                                                                                         | Overview Sticky Note                                                                                                                                                                           |

---

**Disclaimer:** The provided text and workflow originates exclusively from an automated workflow created with n8n, adhering strictly to content policies and handling only legal and public data.