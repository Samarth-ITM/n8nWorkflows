Pre-Meeting Lead Research Agent with Calendly, Perplexity & RapidAPI

https://n8nworkflows.xyz/workflows/pre-meeting-lead-research-agent-with-calendly--perplexity---rapidapi-8957


# Pre-Meeting Lead Research Agent with Calendly, Perplexity & RapidAPI

### 1. Workflow Overview

This workflow automates pre-meeting lead research triggered by a Calendly booking. Its primary purpose is to enrich meeting attendee data with detailed company and person profiles plus relevant business signals, then deliver a comprehensive research summary to Slack for meeting preparation. It is designed for sales teams, BDRs, and account managers to save 30–45 minutes per meeting by automating research tasks.

**Logical Blocks:**

- **1.1 Input Reception:** Listens to Calendly webhook events for new invitees.
- **1.2 AI Orchestration Agent:** Coordinates specialized AI sub-agents to perform company research, person research, and signal detection.
- **1.3 Company Research Agent:** Enriches company data via Perplexity AI.
- **1.4 Person Research Agent:** Uses LinkedIn enrichment APIs and Perplexity AI to profile the invitee.
- **1.5 Signal Research Agent:** Detects hiring, growth, and risk signals for the company.
- **1.6 Output Parsing & Delivery:** Parses AI output into structured JSON and sends a summary message to Slack.
- **1.7 Documentation & Setup Notes:** Sticky notes provide operational context and setup instructions.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block listens for new meeting invitee creation events from Calendly via webhook, serving as the workflow trigger.

**Nodes Involved:**  
- Calendly Trigger

**Node Details:**  
- **Calendly Trigger**  
  - *Type:* Calendly webhook trigger node.  
  - *Configuration:* Subscribed to event `invitee.created` to detect new Calendly bookings.  
  - *Credentials:* Uses configured Calendly API credentials.  
  - *Input:* External webhook call from Calendly.  
  - *Output:* JSON payload with invitee and event details.  
  - *Edge cases:* Webhook delivery failure, API auth errors, malformed payloads.  
  - *Version:* v1  

---

#### 2.2 AI Orchestration Agent

**Overview:**  
Main agent node that orchestrates calls to three specialist research sub-agents, aggregates their outputs, and composes a unified meeting research summary.

**Nodes Involved:**  
- AI Agent  
- Structured Output Parser  
- Send a message (Slack)

**Node Details:**  
- **AI Agent**  
  - *Type:* Langchain AI agent node.  
  - *Configuration:* Receives invitee data, extracts company domain, and coordinates sub-agents: Company Research, Person Research, Signal Research.  
  - *Prompt:* Detailed system message specifying orchestration rules, input data schema, and output JSON schema.  
  - *Input:* Output from Calendly Trigger with invitee and meeting info.  
  - *Output:* Single JSON object summarizing company, person, signals, and meeting prep.  
  - *Edge cases:* AI model timeout, prompt errors, incomplete input data.  
  - *Version:* v2.2  
- **Structured Output Parser**  
  - *Type:* AI output parser node.  
  - *Configuration:* Validates and auto-fixes AI JSON output according to strict manual schema.  
  - *Input:* AI Agent output text.  
  - *Output:* Structured JSON object matching expected schema.  
  - *Edge cases:* Parsing failures, schema mismatches.  
  - *Version:* v1.3  
- **Send a message (Slack)**  
  - *Type:* Slack node.  
  - *Configuration:* Sends the `output.summary` field to a specific Slack user ID.  
  - *Credentials:* Slack API credentials required.  
  - *Input:* Parsed AI output JSON.  
  - *Output:* Slack message posted confirmation.  
  - *Edge cases:* Slack API rate limits, auth issues, user ID misconfiguration.  
  - *Version:* v2.3  

---

#### 2.3 Company Research Agent

**Overview:**  
AI tool node specialized in company profiling, leveraging Perplexity Sonar tool to validate identity and collect detailed company data including recent news.

**Nodes Involved:**  
- Company Research Agent  
- OpenAI Chat Model1  
- Perplexity Research

**Node Details:**  
- **Company Research Agent**  
  - *Type:* Langchain AI tool node.  
  - *Configuration:* Uses system prompt focused on company data enrichment tasks.  
  - *Input:* Text prompt derived from main AI Agent or user message.  
  - *Output:* Normalized company profile JSON with confidence and data quality notes.  
  - *Edge cases:* Missing company info, ambiguous company matches, Perplexity API errors.  
  - *Version:* v2.2  
- **OpenAI Chat Model1**  
  - *Type:* Langchain OpenAI GPT model node.  
  - *Configuration:* Model `gpt-4o` used to assist Company Research Agent.  
  - *Credentials:* OpenAI API key.  
  - *Input/Output:* Connected as language model for Company Research Agent.  
  - *Edge cases:* API quota, latency, model availability.  
  - *Version:* v1.2  
- **Perplexity Research**  
  - *Type:* Perplexity API tool node.  
  - *Configuration:* Uses Sonar model to query company data as instructed by AI prompt.  
  - *Credentials:* Perplexity API key.  
  - *Input:* AI-generated query text.  
  - *Output:* Research results to enrich company profile.  
  - *Edge cases:* API rate limits, incomplete data, response delays.  
  - *Version:* v1  

---

#### 2.4 Person Research Agent

**Overview:**  
AI tool node that enriches invitee’s professional profile by validating and normalizing LinkedIn data, collecting recent activity, and deriving themes.

**Nodes Involved:**  
- Person Research Agent  
- OpenAI Chat Model2  
- Research a person (Perplexity)  
- Enrich LinkedIn Profile (RapidAPI)  
- Get LinkedIn Posts (RapidAPI)  
- Get LinkedIn Comments (RapidAPI)  
- Get LinkedIn Reactions (RapidAPI)

**Node Details:**  
- **Person Research Agent**  
  - *Type:* Langchain AI tool node.  
  - *Configuration:* System prompt instructs validation and enrichment using LinkedIn and Perplexity.  
  - *Input:* Text prompt with invitee details and LinkedIn URL.  
  - *Output:* Detailed person profile JSON with confidence scores.  
  - *Edge cases:* Invalid LinkedIn URL, API errors, no recent activity found.  
  - *Version:* v2.2  
- **OpenAI Chat Model2**  
  - *Type:* Langchain OpenAI GPT model node (`gpt-4o`).  
  - *Credentials:* OpenAI API key.  
  - *Role:* Assists Person Research Agent in natural language processing.  
  - *Edge cases:* API limits, latency.  
- **Research a person (Perplexity)**  
  - *Type:* Perplexity Sonar tool node.  
  - *Role:* Supplements person profile with external background info.  
  - *Credentials:* Perplexity API key.  
- **Enrich LinkedIn Profile (RapidAPI)**  
  - *Type:* HTTP request node to RapidAPI LinkedIn enrichment endpoint.  
  - *Parameters:* Query parameter `username` from AI prompt variables.  
  - *Authentication:* HTTP header with RapidAPI key.  
  - *Edge cases:* API throttling, missing username, HTTP errors.  
- **Get LinkedIn Posts (RapidAPI)**  
  - *Type:* HTTP request node to fetch profile posts from RapidAPI.  
  - *Parameters:* `username`, pagination start=0.  
- **Get LinkedIn Comments (RapidAPI)**  
  - *Type:* HTTP request node to fetch profile comments from RapidAPI.  
- **Get LinkedIn Reactions (RapidAPI)**  
  - *Type:* HTTP request node to fetch profile likes/reactions from RapidAPI.  
- All HTTP nodes require valid RapidAPI credentials.  
- Potential failures: API quota exceeded, network issues, invalid username.  

---

#### 2.5 Signal Research Agent

**Overview:**  
AI tool node that detects actionable business signals (hiring, growth, risk) to provide relevant talking points for meeting prep.

**Nodes Involved:**  
- Signal Research Agent  
- OpenAI Chat Model3  
- Message a model in Perplexity

**Node Details:**  
- **Signal Research Agent**  
  - *Type:* Langchain AI tool node.  
  - *Configuration:* System prompt focused on signal detection and validation.  
  - *Input:* Company domain, LinkedIn URL, news hints.  
  - *Output:* JSON arrays of signals with confidence and sources.  
  - *Edge cases:* No signals found, ambiguous data, API failures.  
  - *Version:* v2.2  
- **OpenAI Chat Model3**  
  - *Type:* Langchain OpenAI GPT model node (`gpt-4o`).  
  - *Role:* Supports signal analysis.  
- **Message a model in Perplexity**  
  - *Type:* Perplexity Sonar tool node.  
  - *Role:* Queries Perplexity for recent business signals verification.  
  - *Credentials:* Perplexity API key.  

---

#### 2.6 Output Parsing & Delivery

**Overview:**  
Ensures the AI's JSON output conforms to the required schema and delivers the final meeting research summary to Slack.

**Nodes Involved:**  
- Structured Output Parser  
- Send a message (Slack)

**Node Details:**  
- Described under block 2.2.

---

#### 2.7 Documentation & Setup Notes

**Overview:**  
Sticky notes provide workflow purpose, setup checklist, and AI research agent descriptions to guide users.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**  
- Simple note nodes with textual content.  
- Contain links, instructions, and credits.  
- No inputs or outputs.  
- Serve only for user guidance.

---

### 3. Summary Table

| Node Name              | Node Type                              | Functional Role                          | Input Node(s)                 | Output Node(s)          | Sticky Note                                                                                         |
|------------------------|--------------------------------------|----------------------------------------|------------------------------|-------------------------|---------------------------------------------------------------------------------------------------|
| Calendly Trigger       | Calendly Trigger                     | Workflow trigger on new Calendly invitee | —                            | AI Agent                | Setup Checklist: Required APIs include Calendly; ensure LinkedIn URLs captured in Calendly form.  |
| AI Agent              | Langchain AI Agent                   | Orchestrates sub-agents and compiles report | Calendly Trigger             | Send a message          | 3 AI Research Agents: Main Agent orchestrates company, person, and signal agents.                  |
| Company Research Agent | Langchain AI Tool                   | Enriches company profile and news       | OpenAI Chat Model1, Perplexity Research | AI Agent                | 3 AI Research Agents: Company Agent uses Perplexity for company intel.                            |
| Person Research Agent  | Langchain AI Tool                   | Enriches invitee’s LinkedIn profile and activity | Research a person, Enrich LinkedIn Profile, Get LinkedIn Posts, Get LinkedIn Comments, Get LinkedIn Reactions, OpenAI Chat Model2 | AI Agent                | 3 AI Research Agents: Person Agent uses LinkedIn tools and Perplexity.                            |
| Signal Research Agent  | Langchain AI Tool                   | Detects business signals for meeting prep | Message a model in Perplexity, OpenAI Chat Model3 | AI Agent                | 3 AI Research Agents: Signal Agent identifies hiring, growth, and risk signals.                   |
| OpenAI Chat Model      | Langchain OpenAI GPT Model          | Language model for AI Agent             | —                            | AI Agent                | Setup Checklist: OpenAI GPT-4 required.                                                           |
| OpenAI Chat Model1     | Langchain OpenAI GPT Model          | Language model for Company Research Agent | —                            | Company Research Agent  |                                                                                                |
| OpenAI Chat Model2     | Langchain OpenAI GPT Model          | Language model for Person Research Agent | —                            | Person Research Agent   |                                                                                                |
| OpenAI Chat Model3     | Langchain OpenAI GPT Model          | Language model for Signal Research Agent | —                            | Signal Research Agent   |                                                                                                |
| OpenAI Chat Model4     | Langchain OpenAI GPT Model          | Language model for Structured Output Parser | —                            | Structured Output Parser |                                                                                                |
| Perplexity Research    | Perplexity Sonar Tool               | Provides company data for Company Research Agent | —                            | Company Research Agent  |                                                                                                |
| Research a person      | Perplexity Sonar Tool               | Provides background info for Person Research Agent | —                            | Person Research Agent   |                                                                                                |
| Enrich LinkedIn Profile| HTTP Request (RapidAPI)             | Enriches LinkedIn profile info          | —                            | Person Research Agent   |                                                                                                |
| Get LinkedIn Posts     | HTTP Request (RapidAPI)             | Retrieves recent LinkedIn posts          | —                            | Person Research Agent   |                                                                                                |
| Get LinkedIn Comments  | HTTP Request (RapidAPI)             | Retrieves LinkedIn comments              | —                            | Person Research Agent   |                                                                                                |
| Get LinkedIn Reactions | HTTP Request (RapidAPI)             | Retrieves LinkedIn likes/reactions       | —                            | Person Research Agent   |                                                                                                |
| Message a model in Perplexity | Perplexity Sonar Tool               | Provides business signals info for Signal Research Agent | —                            | Signal Research Agent   |                                                                                                |
| Structured Output Parser | Langchain Output Parser Structured | Validates and structures AI JSON output | OpenAI Chat Model4            | AI Agent                |                                                                                                |
| Send a message         | Slack Node                        | Sends meeting research summary to Slack | AI Agent                     | —                       | Setup Checklist: Update Slack user/channel ID.                                                   |
| Sticky Note            | Sticky Note                       | Workflow overview and tutorial link      | —                            | —                       | AI Meeting Research Workflow: Purpose, time savings, and target users.                           |
| Sticky Note1           | Sticky Note                       | Setup checklist and API requirements     | —                            | —                       | Setup Checklist: Lists required APIs and key configuration steps.                               |
| Sticky Note2           | Sticky Note                       | Describes 3 AI Research Agents            | —                            | —                       | 3 AI Research Agents: Detailed roles and orchestration instructions.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Calendly Trigger node:**  
   - Type: Calendly Trigger  
   - Configure to listen to event `invitee.created`  
   - Set credentials for Calendly API  
   - Position roughly at start.

3. **Add AI Agent node:**  
   - Type: Langchain AI Agent (`@n8n/n8n-nodes-langchain.agent`)  
   - Configure prompt to orchestrate three sub-agents (Company, Person, Signal) based on invitee data.  
   - Use GPT-4 or equivalent model.  
   - Connect Calendly Trigger output to AI Agent input.

4. **Add Company Research Agent (Langchain AI Tool):**  
   - Type: `@n8n/n8n-nodes-langchain.agentTool`  
   - System prompt: company profile enrichment using Perplexity Sonar as primary info source.  
   - Connect OpenAI Chat Model1 and Perplexity Research node outputs to this agent.  
   - Connect Company Research Agent output to AI Agent as a tool.

5. **Add OpenAI Chat Model1 node:**  
   - Type: Langchain OpenAI GPT model  
   - Select model `gpt-4o` or equivalent  
   - Connect to Company Research Agent as language model.

6. **Add Perplexity Research node:**  
   - Type: Perplexity Sonar tool node  
   - Configure with Perplexity API credentials  
   - Connect output to Company Research Agent.

7. **Add Person Research Agent (Langchain AI Tool):**  
   - Type: Langchain AI tool node  
   - System prompt: enrich LinkedIn profile and activity.  
   - Connect OpenAI Chat Model2 and various LinkedIn enrichment HTTP nodes outputs.  
   - Connect Research a person (Perplexity) output.  
   - Connect Person Research Agent output to AI Agent.

8. **Add OpenAI Chat Model2 node:**  
   - Langchain OpenAI GPT model, model `gpt-4o`  
   - Connect to Person Research Agent.

9. **Add Research a person node:**  
   - Type: Perplexity Sonar tool node  
   - Connect to Person Research Agent.

10. **Add LinkedIn enrichment HTTP request nodes:**  
    - Enrich LinkedIn Profile  
    - Get LinkedIn Posts  
    - Get LinkedIn Comments  
    - Get LinkedIn Reactions  
    - Configure each with RapidAPI HTTP Header Auth credentials.  
    - Set query parameter `username` dynamically from AI prompt variables.  
    - Connect their outputs to Person Research Agent.

11. **Add Signal Research Agent (Langchain AI Tool):**  
    - System prompt: detect hiring, growth, and risk signals.  
    - Connect OpenAI Chat Model3 and Message a model in Perplexity outputs.  
    - Connect output to AI Agent.

12. **Add OpenAI Chat Model3 node:**  
    - Langchain OpenAI GPT model, model `gpt-4o`  
    - Connect to Signal Research Agent.

13. **Add Message a model in Perplexity:**  
    - Perplexity Sonar tool node  
    - Configure with Perplexity API credentials  
    - Connect output to Signal Research Agent.

14. **Add Structured Output Parser node:**  
    - Type: Langchain output parser structured  
    - Paste the JSON schema matching the expected output format.  
    - Enable auto-fix for JSON parsing.  
    - Connect OpenAI Chat Model4 output to this node.

15. **Add OpenAI Chat Model4 node:**  
    - Langchain OpenAI GPT model, model `gpt-4.1-mini` or equivalent  
    - Connect output to Structured Output Parser.

16. **Connect Structured Output Parser output to AI Agent as output parser.**

17. **Add Slack node ("Send a message"):**  
    - Configure Slack API credentials.  
    - Set text to `{{$json.output.summary}}` (from structured parser output).  
    - Set recipient user ID or channel ID.  
    - Connect AI Agent output to Slack node input.

18. **Optional: Add Sticky Note nodes:**  
    - Add notes for workflow overview, setup checklist, and AI agent descriptions for user guidance.

19. **Save and activate the workflow.**

20. **Test by creating a Calendly booking with LinkedIn URL included.**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                          |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------|
| AI Meeting Research Workflow — Purpose, time saved, target users (Sales, BDRs, Account Managers).                    | https://youtu.be/K8E003tw2v8 (Tutorial)|
| Required APIs: Calendly, OpenAI GPT-4, Perplexity, RapidAPI (LinkedIn), Slack.                                       | Setup Checklist Sticky Note             |
| 3 AI Research Agents: Company, Person, Signal plus Main Orchestration Agent.                                         | Sticky Note describing AI Research Agents |
| Slack node requires updating user/channel ID to target correct recipient.                                            | Setup Checklist Sticky Note             |
| Calendly form must capture LinkedIn URLs for best person research results.                                           | Setup Checklist Sticky Note             |
| Perplexity and RapidAPI credentials must be valid and have sufficient quota.                                        | Credential setup notes                   |
| Strict JSON schema enforcement ensures output consistency and downstream usability.                                  | Structured Output Parser notes           |

---

**Disclaimer:**  
The text provided is derived exclusively from an n8n automated workflow. It complies strictly with content policies, contains no illegal or private data, and handles only lawful, public information.