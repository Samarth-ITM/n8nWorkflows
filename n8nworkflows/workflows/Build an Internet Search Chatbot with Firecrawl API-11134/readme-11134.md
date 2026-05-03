Build an Internet Search Chatbot with Firecrawl API

https://n8nworkflows.xyz/workflows/build-an-internet-search-chatbot-with-firecrawl-api-11134


# Build an Internet Search Chatbot with Firecrawl API

### 1. Workflow Overview

This workflow implements a **Firecrawl-based Internet Search Chatbot** that enables users to submit web search queries via a chat interface and receive structured search results in real-time. It also includes an administrative utility to monitor Firecrawl API credit usage.

The workflow is logically divided into three main functional blocks:

- **1.1 Internet Search Chat Interface:**  
  Captures user chat inputs, sends these inputs to the backend search service, receives formatted search results, and replies back to the user in the chat.

- **1.2 Internet Search Service with Firecrawl:**  
  Acts as the backend search service responsible for receiving search queries via webhook, performing the web search through Firecrawl API, formatting the raw results into markdown summaries, and responding to the chat interface.

- **1.3 Firecrawl Account Credits Monitor:**  
  Provides manual triggering and querying of Firecrawl account credit usage, enabling administrators to check remaining credits and usage summaries.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Internet Search Chat Interface

**Overview:**  
This block manages the user-facing chat experience. It receives chat messages, forwards the search query to the backend webhook, formats the returned data, and delivers the final search summary to the user.

**Nodes Involved:**  
- Receive chat message (Chat Trigger)  
- Define constants (Set)  
- Query search server (HTTP Request)  
- Format search response (Python Code)  
- Reply to the user in the chat (Langchain Chat)  
- Terminate interface flow (NoOp)

**Node Details:**

- **Receive chat message**  
  - Type: Chat Trigger (Langchain)  
  - Role: Entry node for capturing user chat messages on a public chat widget.  
  - Configuration: Public access enabled, custom UI styling and placeholder text defined, initial greeting message set.  
  - Input: User chat input string (from widget)  
  - Output: JSON containing `chatInput` with user query  
  - Edge cases: Network issues affecting public webhook accessibility; user input missing or empty.  
  - Version: 1.4

- **Define constants**  
  - Type: Set  
  - Role: Defines a constant variable `webhookUrl` containing the backend webhook URL for forwarding queries.  
  - Configuration: Assigns the webhook URL as a string value.  
  - Input: Data from previous node  
  - Output: Passes data with added constant for use in HTTP request  
  - Edge cases: Incorrect webhook URL configuration causing request failures.  
  - Version: 3.4

- **Query search server (HTTP Request)**  
  - Type: HTTP Request  
  - Role: Sends the user’s search query as a POST request to the backend webhook URL.  
  - Configuration:  
    - URL dynamically set from `webhookUrl` variable.  
    - POST method with JSON body parameter `consultaPesquisa` set to user’s chat input.  
  - Input: JSON containing `webhookUrl` and user query  
  - Output: Raw JSON response from backend search service  
  - Edge cases: HTTP errors (timeouts, 404, 500), malformed queries, webhook unavailability.  
  - Version: 4.3

- **Format search response (Python Code)**  
  - Type: Code (Python)  
  - Role: Parses and formats the raw Firecrawl search response into user-friendly markdown blocks with clickable links and separators.  
  - Configuration:  
    - Reads JSON payload from HTTP Request node.  
    - Extracts search results list (`data.web`).  
    - For each result, constructs markdown with position, title, description, and clickable URL.  
    - Joins results with horizontal separators.  
  - Input: Raw data from HTTP Request  
  - Output: JSON with key `output` containing formatted markdown string  
  - Edge cases: Unexpected or empty search results, invalid JSON payloads, parsing errors.  
  - Version: 2

- **Reply to the user in the chat (Langchain Chat)**  
  - Type: Langchain Chat  
  - Role: Sends the formatted markdown output back to the chat interface as the chatbot’s reply.  
  - Configuration: Message set dynamically from `output` field of previous node.  
  - Input: Formatted markdown string  
  - Output: Chat message sent to user  
  - Edge cases: Message formatting issues; chat delivery failures.  
  - Version: 1

- **Terminate interface flow (NoOp)**  
  - Type: NoOp  
  - Role: Ends the chat interface execution path cleanly.  
  - Configuration: None  
  - Input: Output from chat reply node  
  - Output: None  
  - Edge cases: None  
  - Version: 1

---

#### 2.2 Internet Search Service with Firecrawl

**Overview:**  
Handles incoming search queries from the chat interface webhook, performs a Firecrawl web search, and returns structured search results.

**Nodes Involved:**  
- Receive search query (Webhook)  
- Search the web (Firecrawl)  
- Answer search query (Respond to Webhook)  
- Terminate service flow (NoOp)

**Node Details:**

- **Receive search query**  
  - Type: Webhook  
  - Role: Entry point for backend search queries sent as POST requests from chat interface HTTP node.  
  - Configuration:  
    - HTTP POST method, specific unique path defined for requests.  
    - Response mode set to respond via Respond to Webhook node.  
  - Input: Incoming JSON body with `consultaPesquisa` (search query)  
  - Output: JSON containing search query forwarded to Firecrawl node  
  - Edge cases: Unauthorized or malformed requests, webhook path misconfiguration, connectivity issues.  
  - Version: 2.1

- **Search the web (Firecrawl)**  
  - Type: Firecrawl API node  
  - Role: Executes the actual internet search query using the Firecrawl API.  
  - Configuration:  
    - Operation set to "search".  
    - Query parameter dynamically set from webhook input `consultaPesquisa`.  
    - Credentials linked to Firecrawl API account.  
  - Input: JSON with search query  
  - Output: Raw Firecrawl search results JSON  
  - Edge cases: API authentication errors, rate limiting, network timeouts, invalid queries.  
  - Version: 1

- **Answer search query (Respond to Webhook)**  
  - Type: Respond to Webhook  
  - Role: Sends the raw Firecrawl search results back as JSON response to the calling chat interface HTTP request.  
  - Configuration: Default response options.  
  - Input: Raw Firecrawl data from the search node  
  - Output: HTTP response payload containing search results  
  - Edge cases: Response formatting errors, webhook response timeouts.  
  - Version: 1.4

- **Terminate service flow (NoOp)**  
  - Type: NoOp  
  - Role: Ends the backend search service flow execution cleanly.  
  - Configuration: None  
  - Input: Output from Respond to Webhook node  
  - Output: None  
  - Edge cases: None  
  - Version: 1

---

#### 2.3 Firecrawl Account Credits Monitor

**Overview:**  
Provides a manual trigger that allows an administrator to check the current Firecrawl account credit usage and receive a summary response.

**Nodes Involved:**  
- Manual Trigger  
- Verify Firecrawl Account Credit Balance (Firecrawl API)  
- Terminate Monitor Flow (NoOp)

**Node Details:**

- **Manual Trigger**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the credits monitoring flow.  
  - Configuration: None  
  - Input: User manually triggers node execution  
  - Output: Triggers Firecrawl API credit usage query  
  - Edge cases: None  
  - Version: 1

- **Verify Firecrawl Account Credit Balance**  
  - Type: Firecrawl API node  
  - Role: Queries Firecrawl API for team credit usage and remaining credits.  
  - Configuration:  
    - Operation set to "teamCreditUsage".  
    - Credentials linked to Firecrawl API account.  
  - Input: Trigger from manual node  
  - Output: JSON with credit usage details  
  - Edge cases: API authentication failures, network issues, rate limits.  
  - Version: 1

- **Terminate Monitor Flow (NoOp)**  
  - Type: NoOp  
  - Role: Ends the credit monitoring flow cleanly after returning usage data.  
  - Configuration: None  
  - Input: Output from Firecrawl credit usage node  
  - Output: None  
  - Edge cases: None  
  - Version: 1

---

### 3. Summary Table

| Node Name                            | Node Type                 | Functional Role                             | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                      |
|------------------------------------|---------------------------|-------------------------------------------|------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note5                       | Sticky Note               | Description of Internet Search Service    |                              |                                  | ## Internet Search Service with Firecrawl - Backend search service responsible for processing search requests. |
| Sticky Note6                       | Sticky Note               | Description of Internet Search Chat Interface |                              |                                  | ## Internet Search Chat Interface - Handles user-facing chat interactions for Internet Search feature. |
| Sticky Note                       | Sticky Note               | Description of Firecrawl Account Credits Monitor |                              |                                  | ## Firecrawl Account Credits Monitor - Monitors Firecrawl account credit usage.                 |
| Receive search query              | Webhook                   | Receives search query from chat interface |                              | Search the web (Firecrawl)        |                                                                                                |
| Search the web (Firecrawl)        | Firecrawl API             | Executes web search via Firecrawl          | Receive search query           | Answer search query               |                                                                                                |
| Answer search query               | Respond to Webhook        | Returns raw search results to HTTP caller  | Search the web (Firecrawl)     | Terminate service flow            |                                                                                                |
| Terminate service flow            | NoOp                      | Ends backend search service flow           | Answer search query            |                                  |                                                                                                |
| Receive chat message              | Chat Trigger (Langchain)  | Captures user chat messages                 |                              | Define constants                 |                                                                                                |
| Define constants                 | Set                       | Defines webhook URL constant                | Receive chat message          | Query search server (HTTP)        |                                                                                                |
| Query search server (HTTP)        | HTTP Request              | Sends user query to backend webhook        | Define constants             | Format search response (Python)   |                                                                                                |
| Format search response (Python)   | Code (Python)             | Parses and formats raw search results       | Query search server (HTTP)    | Reply to the user in the chat     |                                                                                                |
| Reply to the user in the chat     | Langchain Chat            | Sends formatted search results to user     | Format search response (Python) | Terminate interface flow         |                                                                                                |
| Terminate interface flow          | NoOp                      | Ends chat interface flow                     | Reply to the user in the chat |                                  |                                                                                                |
| Manual Trigger                   | Manual Trigger            | Manually triggers Firecrawl credit check    |                              | Verify Firecrawl Account Credit Balance |                                                                                                |
| Verify Firecrawl Account Credit Balance | Firecrawl API             | Queries Firecrawl account credit usage      | Manual Trigger               | Terminate Monitor Flow            |                                                                                                |
| Terminate Monitor Flow            | NoOp                      | Ends credit monitoring flow                  | Verify Firecrawl Account Credit Balance |                                  |                                                                                                |
| Sticky Note1                     | Sticky Note               | Notes on workflow architecture and setup    |                              |                                  | ## Internet Search Chat with Firecrawl - How it works & Setup instructions                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Set `Public` to true to allow public access.  
   - Configure the chat interface title, subtitle, and custom CSS styling as per design preferences.  
   - Set the initial message to "🤖 Olá! Como posso ajudar você hoje?".  
   - This node will receive user chat input as `chatInput`.

2. **Create a Set Node to Define Constants**  
   - Type: `Set`  
   - Add a string field named `webhookUrl`.  
   - Set its value to the full URL of the backend webhook (e.g., `https://your-n8n-instance/webhook/your-webhook-path`).  
   - This node prepares the URL for the HTTP request node.

3. **Create HTTP Request Node to Query Backend Search Service**  
   - Type: `HTTP Request`  
   - Set method to POST.  
   - URL: Use expression to set to `{{$json.webhookUrl}}`.  
   - Under body parameters, add a parameter named `consultaPesquisa` with value set to `{{$node["Receive chat message"].json["chatInput"]}}`.  
   - Ensure "Send Body" is enabled and body format is JSON.

4. **Create Python Code Node to Format Search Response**  
   - Type: `Code` with Python selected.  
   - Paste the provided Python script that parses search results JSON, builds markdown with clickable links, and joins blocks with separators.  
   - Input: JSON from HTTP Request node.  
   - Output: JSON object with key `output` containing formatted markdown string.

5. **Create Langchain Chat Node to Reply to User**  
   - Type: `@n8n/n8n-nodes-langchain.chat`  
   - Set message text to `{{$json["output"]}}`.  
   - Set `Wait for User Reply` to false to send message immediately.

6. **Create NoOp Node to Terminate Interface Flow**  
   - Type: `NoOp`  
   - Connect from Langchain Chat node for clean termination.

7. **Create Webhook Node to Receive Backend Search Queries**  
   - Type: `Webhook`  
   - Set HTTP Method to POST.  
   - Define unique webhook path (e.g., `your-webhook-path`).  
   - Set Response Mode to respond via a dedicated Respond to Webhook node.

8. **Create Firecrawl API Node to Perform Web Search**  
   - Type: Firecrawl node (from Mendable Firecrawl integration).  
   - Set operation to `search`.  
   - Query parameter set to `{{$json["body"]["consultaPesquisa"]}}` from webhook JSON body.  
   - Link Firecrawl API credentials (create if not existing).

9. **Create Respond to Webhook Node**  
   - Type: `Respond to Webhook`  
   - Connect it after Firecrawl node to send raw search results as HTTP response.

10. **Create NoOp Node to Terminate Backend Service Flow**  
    - Type: `NoOp`  
    - Connect after Respond to Webhook node.

11. **Create Manual Trigger Node for Credit Monitoring**  
    - Type: `Manual Trigger`  
    - Used to manually initiate credit usage query.

12. **Create Firecrawl API Node for Credit Usage Query**  
    - Type: Firecrawl API node  
    - Set operation to `teamCreditUsage`.  
    - Link same Firecrawl API credentials.

13. **Create NoOp Node to Terminate Monitor Flow**  
    - Type: `NoOp`  
    - Connect after credit usage node.

14. **Connect all nodes as per flow logic:**  
    - Chat Trigger → Define constants → HTTP Request → Python formatter → Langchain Chat → Terminate interface flow  
    - Webhook → Firecrawl Search → Respond to Webhook → Terminate service flow  
    - Manual Trigger → Firecrawl Credit Usage → Terminate monitor flow

15. **Credentials Setup:**  
    - Add Firecrawl API credentials in n8n credentials section with API key and necessary access rights.  
    - Ensure Chat Trigger node is accessible publicly or behind authentication if desired.

16. **Testing:**  
    - Update webhook URL in Define constants node to match your n8n instance and webhook path.  
    - Send test queries via chat interface and verify formatted search results.  
    - Trigger manual credit check to verify account usage reporting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Internet Search Chat with Firecrawl — How it works and setup instructions included in Sticky Note1.                              | See sticky note content for detailed setup checklist and operational overview.                          |
| Chat interface styling uses custom CSS to customize colors, fonts, and layout for a polished user experience.                    | CSS is embedded within the Chat Trigger node parameters.                                               |
| Firecrawl API credential must be created and configured in n8n prior to workflow usage.                                           | Obtain API key from Firecrawl service portal.                                                          |
| Webhook URL must be reachable and correctly set in Define constants node to ensure communication between chat interface and backend. |                                                                                                        |
| For more information on Firecrawl API and usage, visit https://firecrawl.com or the Mendable n8n node documentation.            |                                                                                                        |
| Workflow designed by Alysson Neves, with UI assets referenced in chat header background image URL.                                | https://n8n.io/creators/alysson                                                                          |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.