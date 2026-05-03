Qualify & Nurture Website Demo Requests with Claude AI, Cal.com and Instantly.ai

https://n8nworkflows.xyz/workflows/qualify---nurture-website-demo-requests-with-claude-ai--cal-com-and-instantly-ai-6380


# Qualify & Nurture Website Demo Requests with Claude AI, Cal.com and Instantly.ai

### 1. Workflow Overview

This workflow automates the qualification and nurturing process for demo requests received through a website. It leverages AI models (Anthropic Claude) for lead qualification and personalized outreach, integrates with external services (Cal.com for booking checks, Clay.com for CRM data updates, and Instantly.ai for campaign lead additions), and uses enrichment data to improve decision-making.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Preprocessing:** Receiving demo request data via webhook, initial filtering based on source.
- **1.2 Lead Enrichment and AI Qualification:** Enriching lead data via a LinkedIn profile lookup, then running AI logic to qualify the lead based on business criteria.
- **1.3 Routing Qualified and Unqualified Leads:** Sending qualified leads to Clay CRM with a "Qualified" status and unqualified leads with "Not Booked" or "False" flags.
- **1.4 Demo Booking Verification:** Checking Cal.com API for actual booking confirmation and deciding if follow-up is needed.
- **1.5 AI-Powered Personalized Follow-Up Generation:** Using Claude AI models to research prospect companies, generate personalized follow-up sentences, and add qualified leads to Instantly.ai campaigns.
- **1.6 Final Response and Wait Handling:** Responding to the webhook with qualification outcome and waiting for a specified time before booking check.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Preprocessing

**Overview:**  
Receives incoming demo request data via HTTP POST webhook and filters based on the submission source.

**Nodes Involved:**  
- Webhook  
- If1  
- Route to Clay2  
- Trigify Enrichment

**Node Details:**  

- **Webhook**  
  - Type: HTTP Webhook (n8n-nodes-base.webhook)  
  - Role: Accepts incoming POST requests with demo request data (email, LinkedIn URL, referral source, use case, timestamp).  
  - Configuration: POST method, responseMode set to "responseNode" for synchronous reply.  
  - Input: External HTTP POST from website form.  
  - Output: JSON with body containing lead info.  
  - Edge cases: Invalid or missing fields, malformed requests.

- **If1**  
  - Type: Conditional Filter (n8n-nodes-base.if)  
  - Role: Checks if the source field in request contains "lead_capture_box".  
  - Configuration: String "contains" condition on `$json.body.source`.  
  - Input: Output from Webhook.  
  - Output: True branch leads to Route to Clay2, false branch to Trigify Enrichment.  
  - Edge cases: Missing or unexpected source values.

- **Route to Clay2**  
  - Type: HTTP Request node  
  - Role: Sends unqualified lead data to Clay CRM with status "NOT BOOKED".  
  - Configuration: POST request to Clay webhook URL, sending email, timestamp, and qualification status in body parameters.  
  - Input: True branch from If1.  
  - Output: None (endpoint call).  
  - Edge cases: HTTP errors, network issues, invalid response from Clay.

- **Trigify Enrichment**  
  - Type: HTTP Request node  
  - Role: Enriches lead data by querying Trigify API with LinkedIn URL.  
  - Configuration: GET request to `/v1/profile/track` with `url` query param set to lead’s LinkedIn URL.  
  - Input: False branch from If1.  
  - Output: Enriched JSON profile data containing company, position, geography, etc.  
  - Edge cases: API rate limits, invalid LinkedIn URL, missing data fields.

---

#### 2.2 Lead Enrichment and AI Qualification

**Overview:**  
Uses enriched data to qualify leads as B2B and senior-level prospects using an AI agent powered by Anthropic Claude. Parses AI output to a boolean for downstream logic.

**Nodes Involved:**  
- AI Qualification (AI Qulification)  
- Anthropic Chat Model  
- Structured Output Parser  
- If

**Node Details:**  

- **AI Qualification**  
  - Type: LangChain AI Agent (n8n-nodes-langchain.agent)  
  - Role: Runs a detailed prompt analyzing company industry, size, location, job title, and employment status to determine if the lead is qualified.  
  - Configuration: Custom prompt with embedded JSON expressions from enrichment node, specifying business rules for qualification (e.g., senior titles, allowed countries).  
  - Input: Enriched profile data from Trigify Enrichment.  
  - Output: AI-generated JSON with a boolean "result" field indicating qualification.  
  - Retry: Retries up to 5 times on failure.  
  - Edge cases: AI service downtime, parsing errors, ambiguous input data.

- **Anthropic Chat Model**  
  - Type: AI Language Model node (lmChatAnthropic)  
  - Role: Executes the prompt for AI Qualification using Claude Sonnet 4 model.  
  - Input: Text from AI Qualification node.  
  - Output: AI response for parsing.  
  - Credentials: Requires Anthropic API key.  
  - Edge cases: API rate limit, model version compatibility.

- **Structured Output Parser**  
  - Type: LangChain Output Parser (outputParserStructured)  
  - Role: Parses AI JSON string response into usable JSON object with "result" boolean.  
  - Configuration: Expects JSON schema `{ "result": true }`.  
  - Input: AI response from Anthropic Chat Model.  
  - Output: Parsed JSON for conditional logic.  
  - Edge cases: Malformed AI response, parsing failures.

- **If**  
  - Type: Conditional node  
  - Role: Branches workflow based on AI qualification result.  
  - Condition: `$json.output.result` boolean true or false.  
  - True branch leads to Route to Clay (qualified), false to Route to Clay1 (unqualified).  
  - Edge cases: Missing or invalid AI output.

---

#### 2.3 Routing Qualified and Unqualified Leads

**Overview:**  
Sends enriched lead data to Clay CRM with qualification status flags for record keeping and segmentation.

**Nodes Involved:**  
- Route to Clay  
- Edit Fields2  
- Respond to Webhook  
- Wait  
- Route to Clay1  
- Edit Fields3

**Node Details:**  

- **Route to Clay**  
  - Type: HTTP Request node  
  - Role: Sends qualified lead data to Clay CRM with "Qualified" = TRUE.  
  - Configuration: POST request with body parameters populated from enrichment and webhook data (names, company, job title, address).  
  - Input: True branch from If node.  
  - Output: Leads to Edit Fields2.  
  - Edge cases: HTTP request failures, missing enrichment fields.

- **Edit Fields2**  
  - Type: Set node  
  - Role: Adds `"output": true` field to data, indicating qualified status for response.  
  - Input: Output from Route to Clay.  
  - Output: Leads to Respond to Webhook.  
  - Edge cases: None significant.

- **Respond to Webhook**  
  - Type: Respond to Webhook node  
  - Role: Sends back response to the original webhook caller with qualification result.  
  - Input: From Edit Fields2.  
  - Output: Leads to Wait node.  
  - Edge cases: Client disconnects, timeout.

- **Wait**  
  - Type: Wait node  
  - Role: Pauses workflow for a configured number of minutes before checking booking status.  
  - Configuration: Wait unit is minutes, webhookId set for delayed processing.  
  - Input: From Respond to Webhook.  
  - Output: Leads to Check Cal.com For Booking.  
  - Edge cases: Workflow timeouts, unexpected delays.

- **Route to Clay1**  
  - Type: HTTP Request node  
  - Role: Sends unqualified lead data to Clay CRM with "Qualified" = FALSE.  
  - Configuration: Similar to Route to Clay but with "Qualified": FALSE in the body.  
  - Input: False branch from If node.  
  - Output: Leads to Edit Fields3.  
  - Edge cases: Same as Route to Clay.

- **Edit Fields3**  
  - Type: Set node  
  - Role: Adds `"output": false` field to data, indicating unqualified status.  
  - Input: From Route to Clay1.  
  - Output: No further nodes connected (ends path).  
  - Edge cases: None significant.

---

#### 2.4 Demo Booking Verification

**Overview:**  
After waiting, the workflow queries Cal.com to verify if the prospect actually booked a demo and decides if follow-up is needed.

**Nodes Involved:**  
- Check Cal.com For Booking  
- If3  
- AI Agent

**Node Details:**  

- **Check Cal.com For Booking**  
  - Type: HTTP Request node  
  - Role: Queries Cal.com bookings API filtering by prospect’s email.  
  - Configuration: GET request with query parameter `attendeeEmail` set from webhook email; authorization via Bearer token header.  
  - Input: From Wait node.  
  - Output: Leads to If3.  
  - Edge cases: API rate limits, no bookings found, network errors.

- **If3**  
  - Type: Conditional node  
  - Role: Checks if the booking data contains a non-empty start time (i.e., booking exists).  
  - Condition: `$json.data[0].start` is not empty.  
  - True branch: No further action (empty array).  
  - False branch: Leads to AI Agent node for follow-up email generation.  
  - Edge cases: Empty or malformed response from Cal.com.

- **AI Agent**  
  - Type: LangChain AI Agent (n8n-nodes-langchain.agent)  
  - Role: Generates a personalized follow-up email sentence to re-engage prospects who did not complete booking.  
  - Configuration: Complex prompt describing Trigify’s capabilities, prospect data, and instructions to research prospect’s company and craft personalized content.  
  - Input: From If3 false branch.  
  - Output: Leads to Add to Instantly node.  
  - Credentials: Uses Anthropic Claude 3.7 model via Anthropic Chat Model1 node and web search tool (Search Web) linked as AI tools for research.  
  - Edge cases: AI or search API failures, missing prospect data.

---

#### 2.5 AI-Powered Personalized Follow-Up Generation and Lead Addition

**Overview:**  
Uses AI and web search to generate personalized email sentences, then adds the qualified lead to an Instantly.ai campaign for outreach.

**Nodes Involved:**  
- Anthropic Chat Model1  
- Search Web  
- Structured Output Parser1  
- Add to Instantly

**Node Details:**  

- **Anthropic Chat Model1**  
  - Type: AI Language Model node  
  - Role: Runs Claude 3.7 Sonnet model to process AI Agent prompt for follow-up generation.  
  - Credentials: Anthropic API.  
  - Input: From AI Agent node as ai_languageModel.  
  - Output: AI text response for parsing.  
  - Edge cases: API limits, model errors.

- **Search Web**  
  - Type: LangChain HTTP Request Tool node  
  - Role: Provides web search capability to AI Agent to research prospect company online.  
  - Configuration: POST to OpenAI API endpoint with GPT-4o model and web_search_preview tool enabled.  
  - Credentials: OpenAI API key.  
  - Input: From AI Agent node as ai_tool.  
  - Output: Search results used internally in AI prompt.  
  - Edge cases: API rate limits, poor search results.

- **Structured Output Parser1**  
  - Type: Output Parser node  
  - Role: Parses AI follow-up sentence JSON response into structured object with "Sentence" field.  
  - Input: From Anthropic Chat Model1 ai_outputParser output.  
  - Output: Parsed sentence for use in Add to Instantly.  
  - Edge cases: Parsing failures, malformed JSON.

- **Add to Instantly**  
  - Type: HTTP Request node  
  - Role: Adds the lead to an Instantly.ai campaign with personalized follow-up sentence in custom variables.  
  - Configuration: POST to Instantly API with email, first/last names, company name, and custom variable "research" populated from AI output.  
  - Input: From AI Agent main output.  
  - Credentials: Instantly API key.  
  - Edge cases: API errors, invalid data causing campaign rejection.

---

#### 2.6 Final Response and Wait Handling

**Overview:**  
Sends synchronous response to the original webhook caller and triggers the delayed booking verification process.

**Nodes Involved:**  
- Respond to Webhook  
- Wait

**Node Details:**  

- **Respond to Webhook**  
  - See above in 2.3.

- **Wait**  
  - See above in 2.3.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                           | Input Node(s)               | Output Node(s)             | Sticky Note                              |
|-------------------------|----------------------------------|-----------------------------------------|-----------------------------|----------------------------|-----------------------------------------|
| Webhook                 | HTTP Webhook                     | Receive demo request data                | External HTTP POST          | If1                        |                                         |
| If1                     | Conditional                     | Filter requests by source                | Webhook                    | Route to Clay2, Trigify Enrichment |                                         |
| Route to Clay2           | HTTP Request                    | Send unqualified lead to Clay CRM       | If1 (true)                 |                            |                                         |
| Trigify Enrichment       | HTTP Request                    | Enrich lead data via LinkedIn profile   | If1 (false)                | AI Qualification           |                                         |
| AI Qulification          | LangChain AI Agent              | Qualify lead using AI analysis           | Trigify Enrichment         | If                         |                                         |
| Anthropic Chat Model     | AI Language Model (Claude Sonnet 4) | Run AI prompt for qualification         | AI Qulification (ai_languageModel) | AI Qulification (outputParser) | Requires Anthropic API credentials       |
| Structured Output Parser | Output Parser                   | Parse AI qualification JSON output      | Anthropic Chat Model       | AI Qulification (main)     |                                         |
| If                      | Conditional                    | Branch on AI qualification result       | AI Qulification            | Route to Clay, Route to Clay1 |                                         |
| Route to Clay            | HTTP Request                   | Send qualified lead to Clay CRM          | If (true)                  | Edit Fields2               |                                         |
| Edit Fields2             | Set                            | Mark output as true (qualified)          | Route to Clay              | Respond to Webhook         |                                         |
| Respond to Webhook       | Respond to Webhook             | Respond to HTTP caller with qualification | Edit Fields2               | Wait                      |                                         |
| Wait                    | Wait                           | Delay before booking verification        | Respond to Webhook          | Check Cal.com For Booking  |                                         |
| Check Cal.com For Booking | HTTP Request                   | Check if demo booking exists on Cal.com  | Wait                       | If3                       |                                         |
| If3                     | Conditional                    | Check booking presence                    | Check Cal.com For Booking  | AI Agent (false branch)    |                                         |
| AI Agent                | LangChain AI Agent              | Generate personalized follow-up sentence | If3 (false)                | Add to Instantly           | Uses Claude 3.7 model and web search tool |
| Anthropic Chat Model1    | AI Language Model (Claude 3.7) | Run AI follow-up generation prompt       | AI Agent (ai_languageModel)| Structured Output Parser1  | Requires Anthropic API credentials       |
| Search Web              | LangChain HTTP Request Tool      | Provide web search data for AI           | AI Agent (ai_tool)          | AI Agent                   | Requires OpenAI API credentials          |
| Structured Output Parser1 | Output Parser                 | Parse AI follow-up JSON output            | Anthropic Chat Model1      | AI Agent                   |                                         |
| Add to Instantly         | HTTP Request                   | Add lead to Instantly campaign            | AI Agent                   |                            | Requires Instantly API key                |
| Route to Clay1           | HTTP Request                   | Send unqualified lead to Clay with FALSE | If (false)                 | Edit Fields3               |                                         |
| Edit Fields3             | Set                            | Mark output as false (unqualified)       | Route to Clay1             |                            |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - Method: POST  
   - Path: Set unique webhook path for receiving demo requests.  
   - Response Mode: `responseNode` (for synchronous response).

2. **Create Conditional Node "If1"**  
   - Condition: Check if `$json.body.source` contains `"lead_capture_box"`.  
   - Connect Webhook output to If1.

3. **Create HTTP Request Node "Route to Clay2"**  
   - Purpose: Send unqualified leads with status "NOT BOOKED".  
   - Method: POST  
   - URL: Clay webhook URL (replace with your own).  
   - Body Parameters:  
     - `Data of Demo Booked`: `{{$json.body.timestamp}}`  
     - `Qualified`: "NOT BOOKED"  
     - `Email`: `{{$json.body.email}}`  
   - Connect If1 true output to this node.

4. **Create HTTP Request Node "Trigify Enrichment"**  
   - Purpose: Enrich lead data using LinkedIn URL.  
   - Method: GET  
   - URL: Your enrichment API endpoint `/v1/profile/track`.  
   - Query Parameter: `url` = `{{$json.body.linkedinUrl}}`  
   - Connect If1 false output to this node.

5. **Create LangChain AI Agent Node "AI Qulification"**  
   - Purpose: Use AI to qualify leads.  
   - Prompt: Use detailed prompt with embedded expressions referencing enrichment data (`{{$json.position[0].companyIndustry}}`, etc.) to analyze B2B status, seniority, location, etc.  
   - Set max retries to 5, enable output parser expecting JSON with boolean `result`.  
   - Connect Trigify Enrichment output to this node.

6. **Create Anthropic Chat Model Node**  
   - Model: `claude-sonnet-4-20250514` or latest compatible.  
   - Credentials: Set Anthropic API credentials.  
   - Connect AI Qualification's AI language model input to this node.

7. **Create Structured Output Parser Node**  
   - JSON Schema Example: `{ "result": true }`  
   - Connect Anthropic Chat Model output to this parser.  
   - Connect parser output back to AI Qualification node main output.

8. **Create Conditional Node "If"**  
   - Condition: Check if `$json.output.result` is true.  
   - Connect AI Qualification output to this node.

9. **Create HTTP Request Node "Route to Clay"**  
   - Purpose: Send qualified leads with "Qualified"=TRUE.  
   - Method: POST  
   - URL: Clay webhook URL.  
   - Body Parameters: Map enrichment and webhook fields (first/last name, company, job title, address, referral source, email).  
   - Connect If true output to this node.

10. **Create Set Node "Edit Fields2"**  
    - Add field `"output": true`.  
    - Connect Route to Clay output to this node.

11. **Create Respond to Webhook Node**  
    - Connect Edit Fields2 to this node (sends response to original caller).

12. **Create Wait Node**  
    - Configure delay in minutes (based on your business logic).  
    - Connect Respond to Webhook output to Wait node.

13. **Create HTTP Request Node "Check Cal.com For Booking"**  
    - Method: GET  
    - URL: `https://api.cal.com/v2/bookings`  
    - Query Param: `attendeeEmail` = `{{$node["Webhook"].json.body.email}}`  
    - Headers: Authorization Bearer token, `cal-api-version` header.  
    - Connect Wait output to this node.

14. **Create Conditional Node "If3"**  
    - Condition: Check if `$json.data[0].start` is not empty.  
    - Connect Check Cal.com output to this node.

15. **Create LangChain AI Agent Node "AI Agent"**  
    - Purpose: Generate personalized follow-up sentences for prospects with no bookings.  
    - Prompt: Use detailed prompt describing company, use case, and Trigify capabilities; instruct to research and generate personalized email content.  
    - Connect If3 false output to this node.

16. **Create Anthropic Chat Model Node "Anthropic Chat Model1"**  
    - Model: `claude-3-7-sonnet-20250219` or latest.  
    - Credentials: Anthropic API.  
    - Connect AI Agent's AI language model input here.

17. **Create LangChain HTTP Request Tool Node "Search Web"**  
    - Purpose: Web search tool for AI Agent to use.  
    - Endpoint: OpenAI API POST to `/v1/responses` with `gpt-4o` and web_search_preview tool.  
    - Headers: Content-Type JSON, Authorization Bearer OpenAI API key.  
    - Connect AI Agent's ai_tool input here.

18. **Create Structured Output Parser Node "Structured Output Parser1"**  
    - JSON Schema Example:  
      `{ "Sentence": "Personalized follow-up sentence here" }`  
    - Connect Anthropic Chat Model1 output here.

19. **Connect Structured Output Parser1 output back to AI Agent as outputParser**.

20. **Create HTTP Request Node "Add to Instantly"**  
    - Method: POST  
    - URL: Instantly.ai API endpoint for leads.  
    - Body: JSON with campaign ID, email, first name, last name, company name, and custom variable "research" set to AI Agent’s output sentence.  
    - Headers: Authorization Bearer Instantly API key, Content-Type application/json.  
    - Connect AI Agent main output to this node.

21. **Create HTTP Request Node "Route to Clay1"**  
    - Purpose: Send unqualified leads with "Qualified"=FALSE.  
    - Method: POST  
    - URL: Clay webhook.  
    - Body Parameters similar to Route to Clay but with `"Qualified": "FALSE"`.  
    - Connect If false output from node If to this node.

22. **Create Set Node "Edit Fields3"**  
    - Add field `"output": false`.  
    - Connect Route to Clay1 output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow uses Anthropic Claude AI models (Claude Sonnet 4 and Claude 3.7 Sonnet) for lead qualification and follow-up generation.| Anthropic API required: https://www.anthropic.com/                                                              |
| Web search integration uses OpenAI GPT-4o model with web_search_preview tool for real-time prospect company research.                 | OpenAI API required: https://platform.openai.com/                                                               |
| Lead enrichment relies on Trigify’s LinkedIn profile enrichment API to gather company and position data.                              | Trigify API documentation (internal or contact Trigify team).                                                   |
| Clay.com is used as the CRM system for storing lead qualification status and demo booking data.                                        | Clay API webhook integration: https://www.clay.com/docs/api                                                      |
| Cal.com API is queried to confirm if the prospect actually booked a demo.                                                             | Cal.com API docs: https://cal.com/docs/api                                                                       |
| Instantly.ai is used for lead campaign management and personalized outreach.                                                          | Instantly API docs: https://docs.instantly.ai/api                                                                |
| The use of wait node allows asynchronous confirmation of booking status post initial demo request reception and qualification.        |                                                                                                                 |
| The AI Qualification prompt includes complex logic for seniority, geography, and job title filtering with exceptions for certain countries and job titles. |                                                                                                                 |

---

This structured documentation enables advanced users and automated agents to fully comprehend, reproduce, and maintain the workflow, anticipating potential integration issues and failure modes.