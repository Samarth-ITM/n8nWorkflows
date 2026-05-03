Compare AI Models with Nvidia API: Qwen, DeepSeek, Seed-OSS & Nemotron

https://n8nworkflows.xyz/workflows/compare-ai-models-with-nvidia-api--qwen--deepseek--seed-oss---nemotron-9283


# Compare AI Models with Nvidia API: Qwen, DeepSeek, Seed-OSS & Nemotron

### 1. Workflow Overview

This workflow enables querying and comparing four different AI language models simultaneously via Nvidia's API to achieve parallel processing and faster response aggregation. It is ideal for use cases such as ensemble AI systems, A/B testing of models, redundancy to improve reliability, or research comparing model outputs.

The workflow consists of these logical blocks:

- **1.1 Input Reception:** Receives incoming HTTP POST requests containing user queries via a webhook.
- **1.2 AI Model Routing:** Routes the input query to four parallel HTTP request nodes, each querying a distinct Nvidia-hosted AI model.
- **1.3 Response Aggregation:** Merges the parallel responses into a single data stream, enabling partial results to continue on timeout.
- **1.4 Response Formatting:** Extracts and formats the primary response content from the aggregated data for output.
- **1.5 Output Delivery:** Sends the formatted combined response back as a JSON HTTP response to the original webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block captures incoming user queries through an HTTP webhook POST request. It triggers the workflow upon receiving a call at a specific webhook path.
- **Nodes Involved:**  
  - Webhook Trigger
- **Node Details:**

| Node Name       | Details                                                                                                                 |
|-----------------|-------------------------------------------------------------------------------------------------------------------------|
| Webhook Trigger | - Type: Webhook Trigger node (HTTP POST) <br> - Configured with path `"6737b4b1-3c2f-47b9-89ff-a012c1fa4f29"` <br> - HTTP method: POST <br> - Response mode: Uses Respond To Webhook node later for output <br> - Inputs: None (entry point) <br> - Outputs: AI Model Router <br> - Edge cases: Invalid HTTP method calls, missing payload, malformed JSON input, or network errors. <br> - Version: 2.1 |

#### 2.2 AI Model Routing

- **Overview:** Based on the incoming data, routes the query to one of four parallel branches, each targeting a distinct Nvidia AI model. Enables concurrent invocation of models for speed and comparison.
- **Nodes Involved:**  
  - AI Model Router (Switch node)  
  - Query Qwen3-next-80b-a3b-thinking (Alibaba) (HTTP Request)  
  - Query Bytedance/seed-oss-36b-instruct (Bytedance) (HTTP Request)  
  - Query DeepSeekv3_1 (HTTP Request)  
  - Query Nvidia-nemotron-nano-9b-v2 (Nvidia) (HTTP Request)
- **Node Details:**

| Node Name                              | Details                                                                                                                                |
|--------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| AI Model Router                      | - Type: Switch node <br> - Routes based on `$json['AI Model']` value to one of 5 conditions (1 to 5) <br> - Inputs: Webhook Trigger <br> - Outputs: 4 branches to HTTP request nodes <br> - Edge cases: Input missing `AI Model` field, invalid model number, or routing logic failure. <br> - Version: 3.2 |
| Query Qwen3-next-80b-a3b-thinking (Alibaba) | - Type: HTTP Request <br> - Method: POST <br> - URL: `https://integrate.api.nvidia.com/v1/chat/completions` <br> - Auth: Bearer token (Nvidia API key) <br> - Body: JSON with model `"qwen/qwen3-next-80b-a3b-thinking"` and user message from webhook input field `'Insert your Query'` <br> - Parameters: temperature 0.7, max_tokens 1024 <br> - Input: AI Model Router branch 1 <br> - Output: Merge AI Model (index 0) <br> - Edge cases: Auth failure, rate limits, timeout, malformed input, or Nvidia API errors. <br> - Version: 4.2 |
| Query Bytedance/seed-oss-36b-instruct (Bytedance) | - Type: HTTP Request <br> - Method: POST <br> - URL: same Nvidia API endpoint <br> - Auth: Bearer token <br> - Model: `"bytedance/seed-oss-36b-instruct"` <br> - Input message: from current node JSON `'Insert your Query'` <br> - Parameters: temperature 1.1, top_p 0.95, max_tokens 4096, no streaming <br> - Input: AI Model Router branch 2 <br> - Output: Merge AI Model (index 1) <br> - Edge cases: Same as above, plus handling large max_tokens <br> - Version: 4.2 |
| Query DeepSeekv3_1                   | - Type: HTTP Request <br> - Method: POST <br> - URL: Nvidia API endpoint <br> - Auth: Bearer token <br> - Model: `"deepseek-ai/deepseek-r1"` <br> - Message: user query from webhook input `'Insert your Query'` <br> - Parameters: temperature 0.6, top_p 0.7, max_tokens 4096, streaming enabled <br> - Input: AI Model Router branch 3 <br> - Output: Merge AI Model (index 2) <br> - Edge cases: Streaming response handling, auth, timeout <br> - Version: 4.2 |
| Query Nvidia-nemotron-nano-9b-v2 (Nvidia) | - Type: HTTP Request <br> - Method: POST <br> - URL: Nvidia API endpoint <br> - Auth: Bearer token <br> - Model: `"nvidia/nvidia-nemotron-nano-9b-v2"` <br> - Message: system role with content `"/think"` (preset prompt) <br> - Parameters: temperature 0.6, top_p 0.95, max_tokens 2048, min/max thinking tokens, streaming enabled <br> - Input: AI Model Router branch 4 <br> - Output: Merge AI Model (index 3) <br> - Edge cases: Streaming, auth, timeout, header param issues (empty header parameter present) <br> - Version: 4.2 |

#### 2.3 Response Aggregation

- **Overview:** Collects the four parallel model responses into a single data stream for unified processing. It waits for all or partial results in case of timeouts.
- **Nodes Involved:**  
  - Merge AI Model (Merge node)
- **Node Details:**

| Node Name      | Details                                                                                                         |
|----------------|-----------------------------------------------------------------------------------------------------------------|
| Merge AI Model | - Type: Merge node <br> - Configured to merge 4 inputs (from the 4 HTTP Request nodes) <br> - Input connections: 4 branches from HTTP Request nodes <br> - Output connection: Format Response <br> - Merge mode: Waits for all inputs (default behavior) but continues with partial results on timeout <br> - Edge cases: Missing or delayed responses leading to partial merges, node failure due to unexpected data format <br> - Version: 3.2 |

#### 2.4 Response Formatting

- **Overview:** Extracts the core AI-generated text response from the merged data structure, simplifying the output for client consumption.
- **Nodes Involved:**  
  - Format Response (Set node)
- **Node Details:**

| Node Name       | Details                                                                                                            |
|-----------------|--------------------------------------------------------------------------------------------------------------------|
| Format Response | - Type: Set node <br> - Extracts and assigns `choices[0].message.content` from JSON to a new key with the same name <br> - Input: Merge AI Model <br> - Output: Send Aggregated AI Model Responses <br> - Edge cases: Missing `choices` array, empty content, malformed JSON <br> - Version: 3.4 |

#### 2.5 Output Delivery

- **Overview:** Sends the aggregated and formatted AI model responses back to the caller as a JSON HTTP response.
- **Nodes Involved:**  
  - Send Aggregated AI Model Responses (Respond to Webhook node)
- **Node Details:**

| Node Name                     | Details                                                                                                            |
|-------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Send Aggregated AI Model Responses | - Type: Respond to Webhook <br> - Sends the final JSON response to the original webhook HTTP caller <br> - Input: Format Response <br> - Output: None (terminates workflow) <br> - Edge cases: Network failures, response formatting issues <br> - Version: 1.4 |

---

### 3. Summary Table

| Node Name                               | Node Type              | Functional Role                   | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                          |
|----------------------------------------|------------------------|---------------------------------|-------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook Trigger                        | Webhook Trigger        | Input Reception                 | None                          | AI Model Router                 |                                                                                                    |
| AI Model Router                       | Switch                 | AI Model Routing                | Webhook Trigger               | Query Qwen3-next-80b-a3b-thinking (Alibaba), Query Bytedance/seed-oss-36b-instruct, Query DeepSeekv3_1, Query Nvidia-nemotron-nano-9b-v2 |                                                                                                    |
| Query Qwen3-next-80b-a3b-thinking (Alibaba) | HTTP Request          | Query Qwen Model                | AI Model Router               | Merge AI Model (index 0)         |                                                                                                    |
| Query Bytedance/seed-oss-36b-instruct (Bytedance) | HTTP Request          | Query Seed-OSS Model            | AI Model Router               | Merge AI Model (index 1)         |                                                                                                    |
| Query DeepSeekv3_1                    | HTTP Request           | Query DeepSeek Model            | AI Model Router               | Merge AI Model (index 2)         |                                                                                                    |
| Query Nvidia-nemotron-nano-9b-v2 (Nvidia) | HTTP Request           | Query Nemotron Model            | AI Model Router               | Merge AI Model (index 3)         |                                                                                                    |
| Merge AI Model                       | Merge                  | Response Aggregation            | Query Qwen3-next-80b-a3b-thinking, Query Bytedance/seed-oss-36b-instruct, Query DeepSeekv3_1, Query Nvidia-nemotron-nano-9b-v2 | Format Response                 |                                                                                                    |
| Format Response                      | Set                    | Response Formatting             | Merge AI Model                | Send Aggregated AI Model Responses |                                                                                                    |
| Send Aggregated AI Model Responses   | Respond to Webhook     | Output Delivery                | Format Response              | None                            |                                                                                                    |
| Sticky Note                         | Sticky Note            | Documentation / Notes          | None                          | None                            | "# Compare AI Models with Nvidia API: Qwen, DeepSeek, Seed-OSS & Nemotron\n\n## Overview\n- Queries four AI models simultaneously via Nvidia's API in 2-3 seconds—4x faster than sequential processing. Perfect for ensemble intelligence, model comparison, or redundancy.\n\n## How It Works\n- Webhook Trigger receives queries\n- AI Router distributes to four parallel branches: Qwen2, SyncGenInstruct, DeepSeek-v3.1, and Nvidia Nemotron\n- Merge Node aggregates responses (continues with partial results on timeout)\n- Format Response structures output\n- Webhook Response returns JSON with all model outputs\n\n## Prerequisites\n- Nvidia API key from [build.nvidia.com](https://build.nvidia.com) (free tier available)\n- n8n v1.0.0+ with HTTP access\n- Model access in Nvidia dashboard\n\n## Setup\n1. Import workflow JSON\n2. Configure HTTP nodes: Authentication → Header Auth → `Authorization: Bearer YOUR_TOKEN_HERE`\n3. Activate workflow and test\n\n## Customization\nAdjust temperature/max_tokens in HTTP nodes, add/remove models by duplicating nodes, change primary response selection in Format node, or add Redis caching for frequent queries.\n\n## Use Cases\nMulti-model chatbots, A/B testing, code review, research assistance, and production systems with AI fallback." |
| Sticky Note4                        | Sticky Note            | Empty / Placeholder Note        | None                          | None                            |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook Trigger  
   - HTTP Method: POST  
   - Path: `6737b4b1-3c2f-47b9-89ff-a012c1fa4f29` (custom unique path)  
   - Response Mode: Set to wait for response from Respond to Webhook node  
   - No authentication required here  
   - Save node

2. **Create AI Model Router (Switch) Node**  
   - Type: Switch  
   - Add 5 rules with conditions checking if `$json["AI Model"]` equals `"1"`, `"2"`, `"3"`, `"4"`, or `"5"` respectively  
   - Connect Webhook Trigger node output to AI Model Router input

3. **Create HTTP Request Nodes for Each AI Model**

   For all HTTP Request nodes:  
   - URL: `https://integrate.api.nvidia.com/v1/chat/completions`  
   - Method: POST  
   - Authentication: HTTP Bearer with Nvidia API Key (create credentials with your Nvidia token). Use separate credentials or reuse as needed.

   a. **Query Qwen3-next-80b-a3b-thinking (Alibaba)**  
      - Body type: JSON  
      - Body content:  
        ```json
        {
          "model": "qwen/qwen3-next-80b-a3b-thinking",
          "messages": [
            { "role": "user", "content": "{{ $('On form submission').item.json['Insert your Query'] }}" }
          ],
          "temperature": 0.7,
          "max_tokens": 1024
        }
        ```  
      - Headers: Add `accept: application/json`  
      - Connect AI Model Router output #1 to this node's input

   b. **Query Bytedance/seed-oss-36b-instruct (Bytedance)**  
      - Body content:  
        ```json
        {
          "model": "bytedance/seed-oss-36b-instruct",
          "messages": [
            { "role": "user", "content": "{{ $json['Insert your Query'] }}" }
          ],
          "temperature": 1.1,
          "top_p": 0.95,
          "max_tokens": 4096,
          "thinking_budget": -1,
          "frequency_penalty": 0,
          "presence_penalty": 0,
          "stream": false
        }
        ```  
      - Connect AI Model Router output #2 to this node's input

   c. **Query DeepSeekv3_1**  
      - Body content:  
        ```json
        {
          "model": "deepseek-ai/deepseek-r1",
          "messages": [
            { "role": "user", "content": "{{ $('On form submission').item.json['Insert your Query'] }}" }
          ],
          "temperature": 0.6,
          "top_p": 0.7,
          "frequency_penalty": 0,
          "presence_penalty": 0,
          "max_tokens": 4096,
          "stream": true
        }
        ```  
      - Headers: Add `Accept: application/json`  
      - Connect AI Model Router output #3 to this node's input

   d. **Query Nvidia-nemotron-nano-9b-v2 (Nvidia)**  
      - Body content (raw JSON):  
        ```json
        {
          "model": "nvidia/nvidia-nemotron-nano-9b-v2",
          "messages": [
            { "role": "system", "content": "/think" }
          ],
          "temperature": 0.6,
          "top_p": 0.95,
          "max_tokens": 2048,
          "min_thinking_tokens": 1024,
          "max_thinking_tokens": 2048,
          "frequency_penalty": 0,
          "presence_penalty": 0,
          "stream": true
        }
        ```  
      - Connect AI Model Router output #4 to this node's input

4. **Create Merge Node**  
   - Type: Merge  
   - Number of Inputs: 4  
   - Connect each HTTP Request node's output to one of the merge node inputs (preserving index order)  
   - Connect Merge node output to next step

5. **Create Format Response Node (Set Node)**  
   - Type: Set  
   - Remove all other fields except assign a new field or overwrite `choices[0].message.content` to the expression `={{ $json.choices[0].message.content }}`  
   - Connect Merge node output to this node

6. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Input: Connect from Format Response node  
   - No parameters needed (default sends JSON response)  
   - This node completes the HTTP response for the initial webhook request

7. **Activate Workflow**  
   - Ensure all HTTP Request nodes have working Nvidia API Bearer Token credentials set up correctly  
   - Test by sending POST requests to the webhook URL with JSON payload containing the key `'Insert your Query'` and `'AI Model'` with values 1 to 4 as per routing  
   - Adjust parameters such as temperature, tokens, or add/remove models by duplicating or modifying HTTP Request nodes accordingly

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Queries four AI models simultaneously via Nvidia's API in 2-3 seconds—4x faster than sequential processing. Perfect for ensemble intelligence, model comparison, or redundancy.                                                                                                                                                                                                                                                                                   | Workflow purpose                                   |
| Nvidia API key is required, obtainable from [build.nvidia.com](https://build.nvidia.com) (free tier available).                                                                                                                                                                                                                                                                                                                                                  | Nvidia API registration                            |
| n8n version 1.0.0 or higher with HTTP access is needed.                                                                                                                                                                                                                                                                                                                                                                                                          | n8n version requirement                            |
| Customize temperature and max_tokens parameters in HTTP Request nodes to tune response creativity and length. Add or remove models by duplicating or deleting HTTP Request nodes and adjusting the switch routing accordingly. Adding Redis caching is recommended for frequent queries to reduce API calls and improve speed.                                                                                                                                      | Customization hints                               |
| Use cases include multi-model chatbots, A/B testing, code reviews, research assistance, and production systems with AI fallback for reliability.                                                                                                                                                                                                                                                                                                               | Example use cases                                 |
| Workflow contains sticky notes with detailed instructions and overview for end users inside n8n editor.                                                                                                                                                                                                                                                                                                                                                          | Embedded documentation                            |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.