NPR Listening Service MCP Server

https://n8nworkflows.xyz/workflows/npr-listening-service-mcp-server-5650


# NPR Listening Service MCP Server

### 1. Workflow Overview

This workflow, titled **NPR Listening Service MCP Server**, is designed to expose the NPR Listening Service API as a Multi-Channel Platform (MCP) server endpoint to AI agents. It enables AI-driven interaction with NPR’s audio recommendations and metadata by wrapping nine distinct NPR API endpoints into an MCP-compatible interface.

**Target Use Case:**  
The workflow serves AI agents that require access to NPR’s audio content recommendations, channel listings, user history, organization details, promo audios, ratings submission, and search functionalities. It is ideal for integrations where AI agents dynamically query NPR’s services for personalized and categorized audio content.

**Logical Blocks:**  
The workflow is organized into the following functional blocks, each representing a category of NPR API endpoints:

- **1.1 Setup and Overview**  
  Initialization and instructional notes for setup and usage.

- **1.2 MCP Trigger**  
  The single entry point that receives AI agent requests and routes them internally.

- **1.3 Aggregation Operations**  
  Fetch aggregation-based recommendations.

- **1.4 Channels Operations**  
  Retrieve available NPR channels.

- **1.5 History Operations**  
  Obtain the logged-in user’s media ratings history.

- **1.6 Organizations Operations**  
  Fetch organization-related recommendations and details.

- **1.7 Promo Operations**  
  Get recent promo audio for the logged-in user.

- **1.8 Ratings Operations**  
  Submit media ratings on behalf of the user.

- **1.9 Recommendations Operations**  
  Get user-specific media recommendations.

- **1.10 Search Operations**  
  Search NPR audio content based on terms.

Each block consists of one or more HTTP Request nodes configured to call NPR’s public API with parameters dynamically populated from the AI agent’s input.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Overview

- **Overview:** Provides detailed setup instructions, usage notes, customization tips, and support contact information. Also includes a high-level workflow overview describing the purpose, architecture, and available operations.
- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Guide users through importing, authenticating, activating, and using the workflow.  
    - Content Highlights:  
      - Configure OAuth2 credentials for authentication.  
      - Use MCP trigger webhook URL for AI agent connection.  
      - Notes on AI-powered parameter auto-population (`$fromAI()` expressions).  
      - Suggests adding custom error handling or logging.  
      - Provides Discord and n8n documentation links for help.  
    - Inputs/Outputs: None (informational node)

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Summarizes workflow purpose and lists all nine API operations by category.  
    - Content Highlights:  
      - Explains MCP Trigger’s role as the server endpoint.  
      - Mentions use of HTTP Request nodes for NPR API calls.  
      - Describes how AI expressions feed parameters.  
      - Lists the 9 API endpoints grouped by function (Aggregation, Channels, etc.).  
    - Inputs/Outputs: None (informational node)

---

#### 1.2 MCP Trigger

- **Overview:** Serves as the single webhook entry point for all AI agent requests, receiving and routing calls to the appropriate NPR API endpoint nodes.
- **Nodes Involved:**  
  - NPR Listening Service MCP Server (MCP Trigger)  

- **Node Details:**

  - **NPR Listening Service MCP Server**  
    - Type: MCP Trigger (Langchain)  
    - Role: Listens on webhook path `/npr-listening-service-mcp` for incoming AI agent requests.  
    - Key Configurations:  
      - Webhook path: `npr-listening-service-mcp`  
      - Acts as an AI tool that triggers downstream HTTP request nodes via AI tool connections.  
    - Inputs: External AI agent HTTP requests.  
    - Outputs: Routed to all HTTP request nodes representing different NPR API endpoints.  
    - Failure Modes: Webhook unavailability, network issues, malformed input from AI agent.

---

#### 1.3 Aggregation Operations

- **Overview:** Retrieves recommendations for a specific aggregation (e.g., program or podcast), optionally paginated.
- **Nodes Involved:**  
  - Sticky Note (Aggregation)  
  - Get Aggregation Recommendations (HTTP Request)  

- **Node Details:**

  - **Sticky Note (Aggregation)**  
    - Type: Sticky Note  
    - Role: Labels and visually groups the aggregation-related node.  
    - Inputs/Outputs: None  

  - **Get Aggregation Recommendations**  
    - Type: HTTP Request (Tool)  
    - Role: Calls NPR API endpoint to fetch aggregation recommendations.  
    - URL Template: `https://listening.api.npr.org/v2/aggregation/{{aggId}}/recommendations`  
    - Parameters:  
      - Path: `aggId` (number) dynamically injected from AI input using `$fromAI('aggId', ...)`.  
      - Query: `startNum` (number, optional, default 0), for pagination.  
    - Authentication: Generic HTTP Header Auth (OAuth2 set up externally).  
    - Outputs: NPR API response forwarded back to AI agent.  
    - Failure Modes: Invalid `aggId`, auth errors, API timeouts, malformed requests.

---

#### 1.4 Channels Operations

- **Overview:** Retrieves a list of NPR channels, optionally filtering to only those shown in the client’s Explore view.
- **Nodes Involved:**  
  - Sticky Note (Channels)  
  - List Available Channels (HTTP Request)  

- **Node Details:**

  - **Sticky Note (Channels)**  
    - Type: Sticky Note  
    - Role: Labels the block visually.  
    - Inputs/Outputs: None  

  - **List Available Channels**  
    - Type: HTTP Request (Tool)  
    - Role: Calls `/v2/channels` NPR API endpoint.  
    - Query Parameter:  
      - `exploreOnly` (boolean, optional, default false), controls filtering.  
    - Authentication: Generic HTTP Header Auth.  
    - Outputs: List of channels for AI agent.  
    - Failure Modes: Auth failures, network errors, invalid parameter formats.

---

#### 1.5 History Operations

- **Overview:** Fetches the logged-in user’s recent media rating history.
- **Nodes Involved:**  
  - Sticky Note (History)  
  - Get User Ratings History (HTTP Request)  

- **Node Details:**

  - **Sticky Note (History)**  
    - Type: Sticky Note  
    - Role: Visual label.  
    - Inputs/Outputs: None  

  - **Get User Ratings History**  
    - Type: HTTP Request (Tool)  
    - Role: Calls `/v2/history` to retrieve user ratings.  
    - Authentication: Generic HTTP Header Auth.  
    - Outputs: User rating history data.  
    - Failure Modes: Authorization issues (if user not authenticated), API response errors.

---

#### 1.6 Organizations Operations

- **Overview:** Provides two endpoints: one for category-based recommendations within an organization, and one for detailed organization information.
- **Nodes Involved:**  
  - Sticky Note (Organizations)  
  - Get Category Recommendations (HTTP Request)  
  - Get Organization Details (HTTP Request)  

- **Node Details:**

  - **Sticky Note (Organizations)**  
    - Type: Sticky Note  
    - Role: Visual grouping.  
    - Inputs/Outputs: None  

  - **Get Category Recommendations**  
    - Type: HTTP Request (Tool)  
    - Role: Fetches recommendations by category (newscast, story, podcast) for an organization.  
    - URL Template: `/v2/organizations/{{orgId}}/categories/{{category}}/recommendations`  
    - Parameters:  
      - `orgId` (number), from AI input.  
      - `category` (string, default 'story'), from AI input.  
    - Authentication: Generic HTTP Header Auth.  
    - Failure Modes: Incorrect category or orgId, auth issues.

  - **Get Organization Details**  
    - Type: HTTP Request (Tool)  
    - Role: Retrieves organization details including recent audio items.  
    - URL Template: `/v2/organizations/{{orgId}}/recommendations`  
    - Parameter:  
      - `orgId` (number), from AI input.  
    - Authentication: Generic HTTP Header Auth.  
    - Failure Modes: Invalid orgId, API errors.

---

#### 1.7 Promo Operations

- **Overview:** Fetches the most recent promotional audio for the authenticated user.
- **Nodes Involved:**  
  - Sticky Note (Promo)  
  - Get Recent Promo Audio (HTTP Request)  

- **Node Details:**

  - **Sticky Note (Promo)**  
    - Type: Sticky Note  
    - Role: Visual label.  
    - Inputs/Outputs: None  

  - **Get Recent Promo Audio**  
    - Type: HTTP Request (Tool)  
    - Role: Calls `/v2/promo/recommendations` endpoint.  
    - Authentication: Generic HTTP Header Auth.  
    - Outputs: Promo audio data.  
    - Failure Modes: Auth issues, empty promo data.

---

#### 1.8 Ratings Operations

- **Overview:** Allows submitting media ratings, optionally requesting a new recommendation object.
- **Nodes Involved:**  
  - Sticky Note (Ratings)  
  - Submit Media Ratings (HTTP Request)  

- **Node Details:**

  - **Sticky Note (Ratings)**  
    - Type: Sticky Note  
    - Role: Visual label.  
    - Inputs/Outputs: None  

  - **Submit Media Ratings**  
    - Type: HTTP Request (Tool)  
    - Role: POST to `/v2/ratings` to submit user ratings.  
    - Query Parameter:  
      - `recommend` (boolean, default true), controls response content.  
    - Authentication: Generic HTTP Header Auth.  
    - Outputs: New recommendation object or blank document.  
    - Failure Modes: Invalid payload, auth errors, API rejections.

---

#### 1.9 Recommendations Operations

- **Overview:** Retrieves personalized media recommendations for the logged-in user, optionally considering shared or notified media IDs.
- **Nodes Involved:**  
  - Sticky Note (Recommendations)  
  - Get User Recommendations (HTTP Request)  

- **Node Details:**

  - **Sticky Note (Recommendations)**  
    - Type: Sticky Note  
    - Role: Visual label.  
    - Inputs/Outputs: None  

  - **Get User Recommendations**  
    - Type: HTTP Request (Tool)  
    - Role: Calls `/v2/recommendations` endpoint.  
    - Query Parameters:  
      - `sharedMediaId` (string, optional)  
      - `notifiedMediaId` (string, optional)  
    - Authentication: Generic HTTP Header Auth.  
    - Outputs: Recommendations list.  
    - Failure Modes: Invalid media IDs, auth failures.

---

#### 1.10 Search Operations

- **Overview:** Searches NPR audio and aggregation content based on provided search terms.
- **Nodes Involved:**  
  - Sticky Note (Search)  
  - Get Search Recommendations (HTTP Request)  

- **Node Details:**

  - **Sticky Note (Search)**  
    - Type: Sticky Note  
    - Role: Visual label.  
    - Inputs/Outputs: None  

  - **Get Search Recommendations**  
    - Type: HTTP Request (Tool)  
    - Role: Calls `/v2/search/recommendations` endpoint.  
    - Query Parameter:  
      - `searchTerms` (string, required), dynamically from AI input.  
    - Authentication: Generic HTTP Header Auth.  
    - Outputs: Search results with audio and aggregation items.  
    - Failure Modes: Missing or invalid search terms, auth errors.

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                         | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                           |
|------------------------------|----------------------------|---------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| Setup Instructions            | Sticky Note                | Setup guidance and instructions       | None                             | None                            | ### ⚙️ Setup Instructions – detailed setup and usage guidance with links to Discord and n8n docs.    |
| Workflow Overview             | Sticky Note                | Workflow purpose and API summary      | None                             | None                            | Overview of workflow functionality and listing of all 9 NPR API operations.                          |
| NPR Listening Service MCP Server | MCP Trigger               | Main webhook entry point for AI agent | External HTTP request            | All HTTP Request nodes           |                                                                                                     |
| Sticky Note                  | Sticky Note                | Label for Aggregation block            | None                             | None                            | ## Aggregation                                                                                       |
| Get Aggregation Recommendations | HTTP Request (Tool)       | Fetch aggregation recommendations      | MCP Trigger                     | None                            |                                                                                                     |
| Sticky Note2                 | Sticky Note                | Label for Channels block               | None                             | None                            | ## Channels                                                                                         |
| List Available Channels       | HTTP Request (Tool)         | Fetch NPR channels list                 | MCP Trigger                     | None                            |                                                                                                     |
| Sticky Note3                 | Sticky Note                | Label for History block                | None                             | None                            | ## History                                                                                          |
| Get User Ratings History      | HTTP Request (Tool)         | Retrieve user rating history            | MCP Trigger                     | None                            |                                                                                                     |
| Sticky Note4                 | Sticky Note                | Label for Organizations block          | None                             | None                            | ## Organizations                                                                                     |
| Get Category Recommendations  | HTTP Request (Tool)         | Get recommendations by category/org    | MCP Trigger                     | None                            |                                                                                                     |
| Get Organization Details      | HTTP Request (Tool)         | Get organization detailed info         | MCP Trigger                     | None                            |                                                                                                     |
| Sticky Note5                 | Sticky Note                | Label for Promo block                   | None                             | None                            | ## Promo                                                                                           |
| Get Recent Promo Audio        | HTTP Request (Tool)         | Retrieve recent promo audio             | MCP Trigger                     | None                            |                                                                                                     |
| Sticky Note6                 | Sticky Note                | Label for Ratings block                 | None                             | None                            | ## Ratings                                                                                        |
| Submit Media Ratings          | HTTP Request (Tool)         | Submit media ratings                    | MCP Trigger                     | None                            |                                                                                                     |
| Sticky Note7                 | Sticky Note                | Label for Recommendations block        | None                             | None                            | ## Recommendations                                                                                |
| Get User Recommendations      | HTTP Request (Tool)         | Get personalized user recommendations  | MCP Trigger                     | None                            |                                                                                                     |
| Sticky Note8                 | Sticky Note                | Label for Search block                  | None                             | None                            | ## Search                                                                                         |
| Get Search Recommendations    | HTTP Request (Tool)         | Search NPR content by terms             | MCP Trigger                     | None                            |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Setup Instructions**  
   - Content: Detailed setup and usage instructions including OAuth2 configuration, activating the workflow, getting webhook URL, AI parameter auto-population, customization tips, and support links (Discord and n8n docs).  
   - Position: Left-top corner for visibility.

2. **Create Sticky Note: Workflow Overview**  
   - Content: Summarize workflow purpose, MCP trigger role, NPR API endpoints included (9 total), and AI integration notes.  
   - Position: Near setup instructions for reference.

3. **Create MCP Trigger Node**  
   - Name: `NPR Listening Service MCP Server`  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Webhook path: `npr-listening-service-mcp`  
   - Position: Center-left  
   - Credentials: Configure with appropriate OAuth2 credentials to authenticate HTTP requests.

4. **For Each NPR API Endpoint Block, Create:**

   - A **Sticky Note** as a visual label with the block name (e.g., Aggregation, Channels, History, etc.).

   - One or more **HTTP Request (Tool)** nodes configured as follows:

     - **General HTTP Request Node Settings:**  
       - Authentication: Use generic HTTP Header Auth with OAuth2 credentials.  
       - Send Query Parameters: Enabled when necessary.  
       - URL: Use NPR API endpoint URL with placeholders for dynamic parameters using the expression `$fromAI(parameterName, description, type, defaultValue)`.

     - **Specific Endpoint Configuration:**

       - **Aggregation:**  
         - URL: `https://listening.api.npr.org/v2/aggregation/{{ $fromAI('aggId', ..., 'number') }}/recommendations`  
         - Query: `startNum` (optional, default 0)

       - **Channels:**  
         - URL: `https://listening.api.npr.org/v2/channels`  
         - Query: `exploreOnly` (boolean, optional, default false)

       - **History:**  
         - URL: `https://listening.api.npr.org/v2/history`  
         - No query parameters.

       - **Organizations:**  
         - Get Category Recommendations:  
           - URL: `https://listening.api.npr.org/v2/organizations/{{ $fromAI('orgId', ..., 'number') }}/categories/{{ $fromAI('category', ..., 'string', 'story') }}/recommendations`  
         - Get Organization Details:  
           - URL: `https://listening.api.npr.org/v2/organizations/{{ $fromAI('orgId', ..., 'number') }}/recommendations`

       - **Promo:**  
         - URL: `https://listening.api.npr.org/v2/promo/recommendations`

       - **Ratings:**  
         - URL: `https://listening.api.npr.org/v2/ratings`  
         - Method: POST  
         - Query: `recommend` (boolean, default true)

       - **Recommendations:**  
         - URL: `https://listening.api.npr.org/v2/recommendations`  
         - Query: `sharedMediaId` and `notifiedMediaId` (optional strings)

       - **Search:**  
         - URL: `https://listening.api.npr.org/v2/search/recommendations`  
         - Query: `searchTerms` (string, required)

5. **Connect the MCP Trigger Node’s AI Tool Output to Each HTTP Request Node’s AI Tool Input.**  
   - This enables the MCP trigger to route requests dynamically to the appropriate endpoint nodes based on AI agent commands.

6. **Configure Credentials:**  
   - Create an OAuth2 credential for NPR API access.  
   - Assign this credential to all HTTP Request nodes under the generic HTTP header authentication.

7. **Position Sticky Notes and Nodes Visually:**  
   - Group nodes per functional block for clarity.  
   - Sticky Notes should precede their related HTTP Request nodes visually.

8. **Activate the Workflow:**  
   - Enable the workflow in n8n to start the MCP server and accept incoming AI agent requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Discord support channel for integration guidance and custom automation requests.                              | https://discord.me/cfomodz                                                                                        |
| Official n8n documentation on MCP Trigger and Langchain integration nodes.                                   | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                     |
| Parameters in HTTP requests are auto-populated using `$fromAI()` expression placeholders for dynamic inputs. | Promotes seamless AI integration by converting AI agent prompts into NPR API parameters automatically.            |
| OAuth2 credentials must be configured correctly to authenticate with NPR Listening Service API.              | Essential for securing API requests and avoiding unauthorized errors.                                            |
| The workflow maintains the original API response structure, enabling direct use or further transformation.   | Facilitates downstream processing without loss of data fidelity.                                                |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow, respecting all applicable content policies and containing no illegal, offensive, or protected material. All handled data is legal and publicly accessible.