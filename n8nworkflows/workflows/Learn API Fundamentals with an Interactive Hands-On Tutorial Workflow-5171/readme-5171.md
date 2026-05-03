Learn API Fundamentals with an Interactive Hands-On Tutorial Workflow

https://n8nworkflows.xyz/workflows/learn-api-fundamentals-with-an-interactive-hands-on-tutorial-workflow-5171


# Learn API Fundamentals with an Interactive Hands-On Tutorial Workflow

### 1. Workflow Overview

This workflow is an interactive, step-by-step tutorial designed to teach users the fundamentals of working with APIs (Application Programming Interfaces) through practical examples. It uses a restaurant metaphor where:

- The **Customer** represents the client making requests (modeled by HTTP Request nodes).
- The **Kitchen** represents the server responding to requests (modeled by Webhook nodes).
- The **API** is the interface including methods, URLs, headers, query parameters, and request bodies.

The workflow covers essential API concepts in five lessons, each illustrating a different HTTP interaction pattern:

- **1.1 Lesson 1: Basic GET request** — Retrieving data from the server using a simple GET method and URL.
- **1.2 Lesson 2: GET with Query Parameters** — Customizing requests by adding query parameters to influence the response.
- **1.3 Lesson 3: POST with Body** — Sending data to the server using a POST method with a request body.
- **1.4 Lesson 4: GET with Headers and Authentication** — Using headers for authorization and meta-information.
- **1.5 Lesson 5: Timeout Handling** — Managing slow responses and request timeouts.

Each lesson consists of paired nodes representing the customer request and the kitchen response, supplemented by explanatory sticky notes.

Logical blocks:

- **1.1 Initialization and Base URL Setup**
- **1.2 Lesson 1: Basic GET /menu**
- **1.3 Lesson 2: GET /order with Query Parameters**
- **1.4 Lesson 3: POST /review with Body**
- **1.5 Lesson 4: GET /secret-dish with Header Authentication**
- **1.6 Lesson 5: GET /slow-service with Timeout Handling**
- **1.7 Tutorial Guidance via Sticky Notes**

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Base URL Setup

**Overview:**  
Prepares the base URL for API calls dynamically using environment variables, ensuring subsequent HTTP Requests can target the local Webhook URLs regardless of deployment specifics.

**Nodes Involved:**  
- Start Tutorial  
- Base URL  
- Sticky Note (Tutorial Introduction)

**Node Details:**

- **Start Tutorial**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually.  
  - Config: No parameters; triggers workflow execution.  
  - Inputs: None  
  - Outputs: Connected to Base URL node  
  - Edge Cases: None; manual trigger.

- **Base URL**  
  - Type: Set  
  - Role: Constructs the base URL using environment variables `WEBHOOK_URL` and optional `N8N_ENDPOINT_WEBHOOK`.  
  - Config: Assigns `your_n8n_base_url` to `${WEBHOOK_URL}${N8N_ENDPOINT_WEBHOOK || "webhook"}/tutorial/api`  
  - Inputs: From Start Tutorial  
  - Outputs: To all Customer HTTP Request nodes  
  - Edge Cases: Environment variables must be correctly set; otherwise, URL will be invalid.

- **Sticky Note (Tutorial Introduction)**  
  - Type: Sticky Note  
  - Role: Provides an introduction to the tutorial explaining API basics and usage instructions.  
  - Inputs: None  
  - Outputs: None

---

#### 2.2 Lesson 1: Basic GET /menu

**Overview:**  
Demonstrates a simple GET request to retrieve the menu. The customer sends a GET request; the kitchen responds with fixed menu data.

**Nodes Involved:**  
- 1. The Customer (GET Menu Item)  
- 1. The Kitchen (GET /menu)  
- Respond with Menu  
- Sticky Note1

**Node Details:**

- **1. The Customer (GET Menu Item)**  
  - Type: HTTP Request  
  - Role: Sends a GET request to `/menu` endpoint to get available menu items.  
  - Config: URL is dynamically set from Base URL + `/menu`; method is GET.  
  - Inputs: Base URL node  
  - Outputs: None  
  - Edge Cases: Network issues or incorrect URL environment variables.

- **1. The Kitchen (GET /menu)**  
  - Type: Webhook  
  - Role: Receives GET requests at `/tutorial/api/menu`.  
  - Config: Path `/tutorial/api/menu`, method GET, response mode is last node.  
  - Inputs: HTTP Request node (triggered by GET request)  
  - Outputs: Respond with Menu node

- **Respond with Menu**  
  - Type: Set  
  - Role: Returns a static response representing menu items and prices.  
  - Config: Sets JSON with `item: "Pizza"` and `price: 12`.  
  - Inputs: Kitchen webhook node  
  - Outputs: Response sent back to HTTP request node.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Explains basic HTTP GET method and URL concepts, relating to the restaurant metaphor.  
  - Inputs: None

---

#### 2.3 Lesson 2: GET /order with Query Parameters

**Overview:**  
Teaches how to customize requests using query parameters. The customer orders pizza with or without extra cheese by toggling a query parameter.

**Nodes Involved:**  
- 2. The Customer (GET with Query Params)  
- 2. The Kitchen (GET /order)  
- IF extra cheese (Conditional)  
- Respond with Cheese  
- Respond with Plain  
- Sticky Note2

**Node Details:**

- **2. The Customer (GET with Query Params)**  
  - Type: HTTP Request  
  - Role: Sends a GET request to `/order` with query parameter `extra_cheese=true` or `false`.  
  - Config: URL from Base URL + `/order`, method GET, sends query parameter `extra_cheese`.  
  - Inputs: Base URL node  
  - Outputs: None

- **2. The Kitchen (GET /order)**  
  - Type: Webhook  
  - Role: Listens on `/tutorial/api/order` for GET requests.  
  - Config: Path `/tutorial/api/order`, method GET, response mode last node.  
  - Inputs: Triggered by HTTP Request node  
  - Outputs: IF extra cheese node

- **IF extra cheese**  
  - Type: IF  
  - Role: Checks if query parameter `extra_cheese` is `true`.  
  - Config: Condition `{{$json.query.extra_cheese}} == true` (boolean true)  
  - Inputs: Kitchen webhook node  
  - Outputs: Respond with Cheese (true branch), Respond with Plain (false branch)  
  - Edge Cases: Missing or malformed query parameter may cause false evaluation.

- **Respond with Cheese**  
  - Type: Set  
  - Role: Responds with "Pizza with extra cheese".  
  - Config: Sets JSON `order: "Pizza with extra cheese"`.  
  - Inputs: IF extra cheese true branch  
  - Outputs: Response sent

- **Respond with Plain**  
  - Type: Set  
  - Role: Responds with "Plain Pizza".  
  - Config: Sets JSON `order: "Plain Pizza"`.  
  - Inputs: IF extra cheese false branch  
  - Outputs: Response sent

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Explains query parameters as customization options appended to the URL.  
  - Inputs: None

---

#### 2.4 Lesson 3: POST /review with Body

**Overview:**  
Demonstrates sending data to the server using POST and request body. The customer posts a review comment; the kitchen acknowledges it.

**Nodes Involved:**  
- 3. The Customer (POST with Body)  
- 3. The Kitchen (POST /review)  
- Respond to Review  
- Sticky Note3

**Node Details:**

- **3. The Customer (POST with Body)**  
  - Type: HTTP Request  
  - Role: Sends a POST request to `/review` with JSON body containing a comment.  
  - Config: URL from Base URL + `/review`, method POST, body parameter `comment: "I'm so happy !!"` sent as JSON.  
  - Inputs: Base URL node  
  - Outputs: None

- **3. The Kitchen (POST /review)**  
  - Type: Webhook  
  - Role: Listens for POST requests at `/tutorial/api/review`.  
  - Config: Path `/tutorial/api/review`, method POST, response mode last node.  
  - Inputs: Triggered by HTTP Request node  
  - Outputs: Respond to Review node

- **Respond to Review**  
  - Type: Set  
  - Role: Returns a JSON response acknowledging receipt and echoes the comment.  
  - Config: Sets `status: "review_received"`, `your_comment: {{$json.body.comment}}` (dynamic extraction).  
  - Inputs: Kitchen webhook node  
  - Outputs: Response sent

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Explains POST method and sending data in the request body.  
  - Inputs: None

---

#### 2.5 Lesson 4: GET /secret-dish with Header Authentication

**Overview:**  
Shows how to use headers for authentication by sending an API key in the request header. The kitchen verifies the key and responds accordingly.

**Nodes Involved:**  
- 4. The Customer (GET with Headers/Auth)  
- 4. The Kitchen (GET /secret-dish)  
- IF Authorized  
- Respond with Secret  
- Respond with Error  
- Sticky Note4

**Node Details:**

- **4. The Customer (GET with Headers/Auth)**  
  - Type: HTTP Request  
  - Role: Sends a GET request to `/secret-dish` with header `x-api-key: your-api-key-for-example`.  
  - Config: URL from Base URL + `/secret-dish`, method GET, header parameter set.  
  - Inputs: Base URL node  
  - Outputs: None

- **4. The Kitchen (GET /secret-dish)**  
  - Type: Webhook  
  - Role: Listens for GET requests at `/tutorial/api/secret-dish`.  
  - Config: Path `/tutorial/api/secret-dish`, method GET, response mode last node.  
  - Inputs: Triggered by HTTP Request node  
  - Outputs: IF Authorized node

- **IF Authorized**  
  - Type: IF  
  - Role: Checks if header `x-api-key` equals `your-api-key-for-example`.  
  - Config: Condition `{{$json.headers['x-api-key']}} == 'your-api-key-for-example'`  
  - Inputs: Kitchen webhook node  
  - Outputs: Respond with Secret (true branch), Respond with Error (false branch)  
  - Edge Cases: Missing header or incorrect key leads to authorization failure.

- **Respond with Secret**  
  - Type: Set  
  - Role: Returns secret dish info `"The Chef's Special Truffle Pasta"`.  
  - Config: Sets `dish: "The Chef's Special Truffle Pasta"`.  
  - Inputs: IF Authorized true branch  
  - Outputs: Response sent

- **Respond with Error**  
  - Type: Set  
  - Role: Returns an error message for unauthorized access.  
  - Config: Sets `error: "You are not authorized"`.  
  - Inputs: IF Authorized false branch  
  - Outputs: Response sent

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Explains headers and authentication concepts.  
  - Inputs: None

---

#### 2.6 Lesson 5: GET /slow-service with Timeout Handling

**Overview:**  
Teaches handling of slow API responses by setting a timeout on the client side. The kitchen delays the response deliberately, causing the customer request to timeout.

**Nodes Involved:**  
- 5. The Customer (Request with Timeout)  
- 5. The Kitchen (GET /slow-service)  
- Wait 3 seconds  
- Respond Slowly  
- Sticky Note5

**Node Details:**

- **5. The Customer (Request with Timeout)**  
  - Type: HTTP Request  
  - Role: Sends GET request to `/slow-service` with a 2-second timeout.  
  - Config: URL from Base URL + `/slow-service`, timeout set to 2000 milliseconds (2 seconds), on error continue.  
  - Inputs: Base URL node  
  - Outputs: None  
  - Edge Cases: Designed to fail due to timeout before kitchen responds.

- **5. The Kitchen (GET /slow-service)**  
  - Type: Webhook  
  - Role: Listens for GET requests at `/tutorial/api/slow-service`.  
  - Config: Path `/tutorial/api/slow-service`, method GET, response mode last node.  
  - Inputs: Triggered by HTTP Request node  
  - Outputs: Wait 3 seconds node

- **Wait 3 seconds**  
  - Type: Wait  
  - Role: Delays workflow execution for 3 seconds simulating a slow kitchen.  
  - Config: Wait duration 3 seconds.  
  - Inputs: Kitchen webhook node  
  - Outputs: Respond Slowly node

- **Respond Slowly**  
  - Type: Set  
  - Role: Returns a response after delay `"Finally, your food is here!"`.  
  - Config: Sets `status: "Finally, your food is here!"`.  
  - Inputs: Wait node  
  - Outputs: Response sent

- **Sticky Note5**  
  - Type: Sticky Note  
  - Role: Explains timeout concept and shows how the customer gives up waiting before the kitchen responds.  
  - Inputs: None

---

### 3. Summary Table

| Node Name                         | Node Type         | Functional Role                        | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                                         |
|----------------------------------|-------------------|-------------------------------------|------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Start Tutorial                   | Manual Trigger    | Starts the workflow                  | None                         | Base URL                         |                                                                                                                     |
| Base URL                        | Set               | Builds base URL from env variables   | Start Tutorial               | 1. The Customer (GET Menu Item), 2. The Customer (GET with Query Params), 3. The Customer (POST with Body), 4. The Customer (GET with Headers/Auth), 5. The Customer (Request with Timeout) |                                                                                                                     |
| Sticky Note                     | Sticky Note       | Tutorial introduction and API basics | None                         | None                            | Tutorial - What is an API? Welcome! This workflow will teach you the basics of APIs (Application Programming Interfaces). |
| 1. The Customer (GET Menu Item)  | HTTP Request      | Sends GET /menu request               | Base URL                    | None                            | Lesson 1: The Basics (Method & URL) Explains basic GET request and URL concepts.                                    |
| 1. The Kitchen (GET /menu)        | Webhook           | Receives GET /menu                   | HTTP Request (GET Menu Item) | Respond with Menu                |                                                                                                                     |
| Respond with Menu               | Set               | Responds with fixed menu data         | Webhook (GET /menu)          | Response                       |                                                                                                                     |
| Sticky Note1                   | Sticky Note       | Explains HTTP GET basics              | None                         | None                            | Lesson 1: The Basics (Method & URL)                                                                                  |
| 2. The Customer (GET with Query Params) | HTTP Request      | Sends GET /order with query param    | Base URL                    | None                            | Lesson 2: Customizing a Request (Query Parameters) Explains query parameters and usage.                             |
| 2. The Kitchen (GET /order)       | Webhook           | Receives GET /order                   | HTTP Request (GET with Query Params) | IF extra cheese                 |                                                                                                                     |
| IF extra cheese                | IF                | Checks if extra_cheese query param is true | Webhook (GET /order)         | Respond with Cheese, Respond with Plain |                                                                                                                     |
| Respond with Cheese            | Set               | Responds with pizza with extra cheese | IF extra cheese (true)       | Response                       |                                                                                                                     |
| Respond with Plain             | Set               | Responds with plain pizza             | IF extra cheese (false)      | Response                       |                                                                                                                     |
| Sticky Note2                   | Sticky Note       | Explains query parameters             | None                         | None                            | Lesson 2: Customizing a Request (Query Parameters)                                                                   |
| 3. The Customer (POST with Body)  | HTTP Request      | Sends POST /review with body          | Base URL                    | None                            | Lesson 3: Sending Data (POST & Body) Explains POST method and request body.                                        |
| 3. The Kitchen (POST /review)     | Webhook           | Receives POST /review                 | HTTP Request (POST with Body) | Respond to Review               |                                                                                                                     |
| Respond to Review              | Set               | Responds acknowledging review         | Webhook (POST /review)       | Response                       |                                                                                                                     |
| Sticky Note3                   | Sticky Note       | Explains POST and sending data        | None                         | None                            | Lesson 3: Sending Data (POST & Body)                                                                                 |
| 4. The Customer (GET with Headers/Auth) | HTTP Request      | Sends GET /secret-dish with header    | Base URL                    | None                            | Lesson 4: Identification (Headers & Auth) Explains headers for authentication.                                     |
| 4. The Kitchen (GET /secret-dish) | Webhook           | Receives GET /secret-dish             | HTTP Request (GET with Headers/Auth) | IF Authorized                  |                                                                                                                     |
| IF Authorized                 | IF                | Checks authorization header            | Webhook (GET /secret-dish)   | Respond with Secret, Respond with Error |                                                                                                                     |
| Respond with Secret           | Set               | Responds with secret dish              | IF Authorized (true)         | Response                       |                                                                                                                     |
| Respond with Error            | Set               | Responds with error message            | IF Authorized (false)        | Response                       |                                                                                                                     |
| Sticky Note4                  | Sticky Note       | Explains headers and authentication   | None                         | None                            | Lesson 4: Identification (Headers & Auth)                                                                            |
| 5. The Customer (Request with Timeout) | HTTP Request      | Sends GET /slow-service with timeout  | Base URL                    | None                            | Lesson 5: Being Patient (Timeout) Explains timeout handling.                                                        |
| 5. The Kitchen (GET /slow-service) | Webhook           | Receives GET /slow-service             | HTTP Request (Request with Timeout) | Wait 3 seconds                 |                                                                                                                     |
| Wait 3 seconds               | Wait              | Delays response by 3 seconds           | Webhook (GET /slow-service)  | Respond Slowly                 |                                                                                                                     |
| Respond Slowly              | Set               | Sends delayed response                  | Wait 3 seconds              | Response                       |                                                                                                                     |
| Sticky Note5                | Sticky Note       | Explains timeout concept                | None                         | None                            | Lesson 5: Being Patient (Timeout)                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: Start Tutorial  
   - No parameters needed.

2. **Create Set Node for Base URL**  
   - Name: Base URL  
   - Add assignment:  
     - Name: your_n8n_base_url  
     - Type: String  
     - Value: `={{ $env.WEBHOOK_URL + ($env.N8N_ENDPOINT_WEBHOOK ?? "webhook") + "/tutorial/api" }}`  
   - Connect Start Tutorial → Base URL.

3. **Create Lesson 1 Nodes:**  
   - **HTTP Request Node:**  
     - Name: 1. The Customer (GET Menu Item)  
     - URL: `={{ $('Base URL').last().json.your_n8n_base_url }}/menu`  
     - Method: GET  
     - Connect Base URL → 1. The Customer (GET Menu Item).  
   - **Webhook Node:**  
     - Name: 1. The Kitchen (GET /menu)  
     - Path: `/tutorial/api/menu`  
     - HTTP Method: GET  
     - Response Mode: Last Node  
     - Connect to Set Node Respond with Menu.  
   - **Set Node:**  
     - Name: Respond with Menu  
     - Assignments:  
       - item: "Pizza" (string)  
       - price: 12 (number)  
   - Connect 1. The Kitchen (GET /menu) → Respond with Menu.

4. **Create Lesson 2 Nodes:**  
   - **HTTP Request Node:**  
     - Name: 2. The Customer (GET with Query Params)  
     - URL: `={{ $('Base URL').last().json.your_n8n_base_url }}/order`  
     - Method: GET  
     - Send Query Parameters: Enabled  
     - Add parameter: `extra_cheese` = `true` (string) initially  
     - Connect Base URL → 2. The Customer (GET with Query Params).  
   - **Webhook Node:**  
     - Name: 2. The Kitchen (GET /order)  
     - Path: `/tutorial/api/order`  
     - HTTP Method: GET  
     - Response Mode: Last Node  
     - Connect 2. The Customer (GET with Query Params) → 2. The Kitchen (GET /order).  
   - **IF Node:**  
     - Name: IF extra cheese  
     - Condition: Check if `$json.query.extra_cheese` is boolean true.  
     - Connect 2. The Kitchen (GET /order) → IF extra cheese.  
   - **Set Node (True branch):**  
     - Name: Respond with Cheese  
     - Assign `order` = "Pizza with extra cheese"  
   - **Set Node (False branch):**  
     - Name: Respond with Plain  
     - Assign `order` = "Plain Pizza"  
   - Connect IF extra cheese true → Respond with Cheese  
   - Connect IF extra cheese false → Respond with Plain

5. **Create Lesson 3 Nodes:**  
   - **HTTP Request Node:**  
     - Name: 3. The Customer (POST with Body)  
     - URL: `={{ $('Base URL').last().json.your_n8n_base_url }}/review`  
     - Method: POST  
     - Send Body: Enabled  
     - Body Parameters: `comment: "I'm so happy !!"`  
     - Connect Base URL → 3. The Customer (POST with Body)  
   - **Webhook Node:**  
     - Name: 3. The Kitchen (POST /review)  
     - Path: `/tutorial/api/review`  
     - HTTP Method: POST  
     - Response Mode: Last Node  
     - Connect 3. The Customer (POST with Body) → 3. The Kitchen (POST /review)  
   - **Set Node:**  
     - Name: Respond to Review  
     - Assignments:  
       - status: "review_received"  
       - your_comment: `={{ $json.body.comment }}` (dynamic)  
     - Connect 3. The Kitchen (POST /review) → Respond to Review

6. **Create Lesson 4 Nodes:**  
   - **HTTP Request Node:**  
     - Name: 4. The Customer (GET with Headers/Auth)  
     - URL: `={{ $('Base URL').last().json.your_n8n_base_url }}/secret-dish`  
     - Method: GET  
     - Send Headers: Enabled  
     - Header Parameters: `x-api-key: your-api-key-for-example`  
     - Connect Base URL → 4. The Customer (GET with Headers/Auth)  
   - **Webhook Node:**  
     - Name: 4. The Kitchen (GET /secret-dish)  
     - Path: `/tutorial/api/secret-dish`  
     - HTTP Method: GET  
     - Response Mode: Last Node  
     - Connect 4. The Customer (GET with Headers/Auth) → 4. The Kitchen (GET /secret-dish)  
   - **IF Node:**  
     - Name: IF Authorized  
     - Condition: Check if `$json.headers['x-api-key'] == 'your-api-key-for-example'` (string equals)  
     - Connect 4. The Kitchen (GET /secret-dish) → IF Authorized  
   - **Set Node (True branch):**  
     - Name: Respond with Secret  
     - Assign `dish` = "The Chef's Special Truffle Pasta"  
   - **Set Node (False branch):**  
     - Name: Respond with Error  
     - Assign `error` = "You are not authorized"  
   - Connect IF Authorized true → Respond with Secret  
   - Connect IF Authorized false → Respond with Error

7. **Create Lesson 5 Nodes:**  
   - **HTTP Request Node:**  
     - Name: 5. The Customer (Request with Timeout)  
     - URL: `={{ $('Base URL').last().json.your_n8n_base_url }}/slow-service`  
     - Method: GET  
     - Options → Timeout: 2000 ms (2 seconds)  
     - On Error: Continue Regular Output (to avoid workflow stopping)  
     - Connect Base URL → 5. The Customer (Request with Timeout)  
   - **Webhook Node:**  
     - Name: 5. The Kitchen (GET /slow-service)  
     - Path: `/tutorial/api/slow-service`  
     - HTTP Method: GET  
     - Response Mode: Last Node  
     - Connect 5. The Customer (Request with Timeout) → 5. The Kitchen (GET /slow-service)  
   - **Wait Node:**  
     - Name: Wait 3 seconds  
     - Wait Time: 3 seconds  
     - Connect 5. The Kitchen (GET /slow-service) → Wait 3 seconds  
   - **Set Node:**  
     - Name: Respond Slowly  
     - Assign `status` = "Finally, your food is here!"  
     - Connect Wait 3 seconds → Respond Slowly

8. **Add Sticky Notes for Each Lesson and Introduction**  
   - Create Sticky Notes with the exact content explaining each lesson's concepts as per the provided notes in the JSON.  
   - Position notes near corresponding nodes for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow teaches API fundamentals using a restaurant metaphor: Client = HTTP Request, Server = Webhook, API = Waiter & Menu. | Tutorial introduction sticky note.                                                                      |
| To dynamically get webhook URLs, environment variables `WEBHOOK_URL` and optionally `N8N_ENDPOINT_WEBHOOK` are used. | Base URL node setup.                                                                                     |
| The HTTP Request node timeout setting is critical to prevent workflow hang on slow or unresponsive services.  | Lesson 5 on timeout handling.                                                                            |
| Headers can be used for passing API keys or authentication tokens, demonstrated using `x-api-key`.            | Lesson 4 on headers and authentication.                                                                 |
| Conditional logic using IF nodes is used to branch responses based on query parameters or header values.      | Lessons 2 and 4 use IF nodes for branching.                                                             |
| This workflow is designed to be executed manually and explored node by node for learning purposes.            | Overall workflow usage instructions in tutorial introduction sticky note.                                |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow built with n8n, a no-code automation platform. It adheres strictly to content policies and does not contain illegal or protected materials. All data handled are legal and publicly accessible.