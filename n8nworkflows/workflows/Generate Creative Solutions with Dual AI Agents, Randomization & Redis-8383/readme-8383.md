Generate Creative Solutions with Dual AI Agents, Randomization & Redis

https://n8nworkflows.xyz/workflows/generate-creative-solutions-with-dual-ai-agents--randomization---redis-8383


# Generate Creative Solutions with Dual AI Agents, Randomization & Redis

### 1. Workflow Overview

This workflow is designed to generate creative, innovative solutions to user-provided problems by leveraging dual AI agents, a randomization mechanism based on the Mersenne Twister algorithm, and temporary data storage using Redis. It targets brainstorming and ideation scenarios where diverse semantic input and critical evaluation are essential for high-quality output.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user problem statements via a chat interface.
- **1.2 Random Number Generation:** Uses a Mersenne Twister implementation to produce high-entropy random seeds.
- **1.3 Random Word Generation:** Converts random numbers into diverse, semantically distributed English words through an AI agent.
- **1.4 Redis Temporary Storage:** Stores generated words and tracks counts with TTL for transient semantic buffering.
- **1.5 Brainstorming AI Agent:** Consumes problem statements and accumulated random words to create five innovative ideas.
- **1.6 Idea Queue Management:** Controls the flow of ideas from Redis, ensuring the threshold of words before brainstorming is met.
- **1.7 Critic AI Agent:** Evaluates the five generated ideas and synthesizes a single optimal solution.
- **1.8 Configuration and Documentation:** Notes for setup, credentials, and operational guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives user input problem statements through a chat webhook, triggering the workflow start.
- **Nodes Involved:** `chat`
- **Node Details:**
  - Type: Chat Trigger (LangChain)
  - Role: Entry point for user problem input
  - Configuration: Uses webhook with ID `brainstorm-generator-webhook`
  - Inputs: HTTP chat messages
  - Outputs: Passes chat input JSON downstream
  - Edge Cases: Webhook failures, invalid inputs, network errors

#### 2.2 Random Number Generation (Mersenne Twister)

- **Overview:** Implements a native Mersenne Twister MT19937 algorithm in code to generate high-entropy random numbers with deterministic seeding.
- **Nodes Involved:** `mersenne_twister`, `get_random_number`
- **Node Details:**
  - `mersenne_twister`:
    - Type: Code node (JavaScript)
    - Role: Generates random integers using MT19937 seeded by current timestamp
    - Key Logic: Generates two random numbers per input item (`randomNumber`, `customRandomNumber`)
    - Outputs: Adds numbers and seed metadata to item JSON
    - Failure Types: JS execution errors, timestamp retrieval failure
  - `get_random_number`:
    - Type: Set node
    - Role: Calculates a derived random number as `(randomNumber * customRandomNumber * seed) % 1,000,000`
    - Outputs: Named value `number` as string
    - Edge Cases: Numeric overflow, expression evaluation errors

#### 2.3 Random Word Generation

- **Overview:** Converts the generated random number into a single English word with high entropy and balanced lexical distribution.
- **Nodes Involved:** `Random Word Generator`, `OpenAI Chat Model - Word Generator`
- **Node Details:**
  - `Random Word Generator`:
    - Type: LangChain Agent
    - Role: Produces one random English word based on input numeric seed
    - Configuration: System prompt defines detailed generation rules emphasizing randomness, lexical diversity, and exclusion of offensive content
    - Input Expression: `={{ $json.number }}`
    - Output: Single word per request
  - `OpenAI Chat Model - Word Generator`:
    - Type: OpenAI Chat Model node
    - Role: Provides GPT-4 language model backend for Random Word Generator agent
    - Parameters: Temperature=1, TopP=1, frequencyPenalty=1 (to reduce repetition)
    - Edge Cases: API rate limits, invalid credentials, model unavailability

#### 2.4 Redis Temporary Storage

- **Overview:** Stores each generated word in a Redis list with a counter and TTL for ephemeral semantic buffer management.
- **Nodes Involved:** `store_idea`, `count_ideas`, `get_count`
- **Node Details:**
  - `store_idea`:
    - Type: Redis node
    - Operation: Pushes sanitized word (removes markdown and backticks) to Redis list `brainstorm`
    - Inputs: Word from Random Word Generator output
    - Failure Types: Redis connection/auth errors
  - `count_ideas`:
    - Type: Redis node
    - Operation: Increments key `brainstorm_count` with TTL 30 seconds (auto-expire)
    - Role: Tracks number of words stored
  - `get_count`:
    - Type: Redis node
    - Operation: Retrieves current value of `brainstorm_count`
    - Outputs: Word count as property `count`
    - Edge Cases: Redis key missing, TTL expiry before read

#### 2.5 Idea Queue Management & Threshold Checking

- **Overview:** Ensures at least 36 words have been accumulated before triggering brainstorming; otherwise, continues random word generation.
- **Nodes Involved:** `check_number_of_ideas`, `extract_ideas`, `check_queue_is_empty`, `get_idea`, `set_idea`
- **Node Details:**
  - `check_number_of_ideas`:
    - Type: If node
    - Condition: Checks if `count` >= 36
    - True: Proceeds to extract ideas
    - False: Loops back to generate more random numbers
  - `extract_ideas`:
    - Type: Redis node
    - Operation: Pops one word from Redis list `brainstorm`
    - Outputs: Word as `text`
  - `check_queue_is_empty`:
    - Type: If node
    - Condition: Checks if popped word is empty (i.e., queue empty)
    - True: Proceeds to filtering and brainstorming
    - False: Loops back to extract another word
  - `get_idea`, `set_idea`:
    - Redis nodes to get/set cached messages keyed by Redis string `message`
    - Used for interim storage during idea extraction
    - TTL set to 5 seconds on set
    - Edge Cases: Redis availability, key conflicts

#### 2.6 Brainstorming AI Agent

- **Overview:** Receives the user problem statement and concatenated random words to generate five innovative ideas using creative AI prompting.
- **Nodes Involved:** `Brainstorming`, `Google Gemini Chat Model - Brainstorming`, `filtering`
- **Node Details:**
  - `filtering`:
    - Type: Set node
    - Role: Cleans up the concatenated message string (replaces double newlines with single newline)
    - Output: Provides cleaned keywords to Brainstorming node
  - `Brainstorming`:
    - Type: LangChain Agent
    - Role: Generates exactly five distinct, creative solutions integrating problem and keywords
    - System Prompt Highlights:
      - Ideation Specialist persona
      - Requires titles, summaries, detailed descriptions, and keywords per idea
      - Encourages divergent thinking, creative synthesis, and feasibility
    - Inputs: Problem statement and filtered keywords
  - `Google Gemini Chat Model - Brainstorming`:
    - Type: Google Gemini language model node
    - Provides AI backend for brainstorming (alternative to OpenAI)
    - Configured with temperature=1, topP=1
    - Edge Cases: API rate limits, auth errors

#### 2.7 Critic AI Agent

- **Overview:** Critically evaluates and synthesizes the five brainstorming ideas into a single optimal, actionable solution.
- **Nodes Involved:** `Critic`, `OpenAI Chat Model - Critic`
- **Node Details:**
  - `Critic`:
    - Type: LangChain Agent
    - Role: Analyzes five proposals on impact, viability, innovation, scalability, and risks
    - Outputs a unified refined solution with title, executive summary, detailed description, justification, and risk mitigation
    - Input: Output from Brainstorming node
  - `OpenAI Chat Model - Critic`:
    - Type: OpenAI GPT-4 Chat Model
    - Supports Critic node with high-quality language model
    - Parameters: temperature=1, topP=1
    - Edge Cases: API connectivity, auth failures

#### 2.8 Configuration and Documentation Notes

- **Overview:** Provides setup instructions, credential requirements, and operational tips for the workflow.
- **Nodes Involved:** `Welcome & Overview`, `Mersenne Twister Explanation`, `Word Generator Note`, `Redis Storage Note`, `Brainstorming Agent Note`, `Critic Agent Note`, `Configuration Notes`
- **Node Details:**
  - Sticky Notes:
    - Explain concepts like Mersenne Twister pseudo-random generation, Redis in-memory storage, AI agent roles, and configuration tips.
    - Document required credentials: Redis connection, OpenAI or Google Gemini API keys.
    - Outline customization options such as word count thresholds, TTL, temperature settings.
    - Author and version metadata included.

---

### 3. Summary Table

| Node Name                       | Node Type                           | Functional Role                                | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                          |
|--------------------------------|-----------------------------------|-----------------------------------------------|-----------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Welcome & Overview              | Sticky Note                       | Documentation overview of workflow            | -                           | -                             | ## 🚀 AI Brainstorm Generator Workflow ... (full content)                                          |
| Mersenne Twister Explanation   | Sticky Note                       | Documentation on Mersenne Twister RNG         | -                           | -                             | ## 🎲 Mersenne Twister Implementation ...                                                          |
| Word Generator Note             | Sticky Note                       | Documentation on random word generation        | -                           | -                             | ## 📝 Random Word Generator ...                                                                     |
| Redis Storage Note              | Sticky Note                       | Documentation on Redis storage usage           | -                           | -                             | ## 💾 Redis Storage ...                                                                             |
| Brainstorming Agent Note        | Sticky Note                       | Documentation on brainstorming AI agent        | -                           | -                             | ## 🧠 Brainstorming Agent ...                                                                       |
| Critic Agent Note              | Sticky Note                       | Documentation on critic AI agent                | -                           | -                             | ## 🎯 Critic Agent ...                                                                              |
| Configuration Notes            | Sticky Note                       | Documentation on required credentials and setup| -                           | -                             | ## ⚙️ Configuration Notes ...                                                                       |
| chat                          | LangChain Chat Trigger             | Entry point for user problem input             | -                           | mersenne_twister              |                                                                                                    |
| mersenne_twister              | Code                             | Generates high-entropy random numbers          | chat                        | get_random_number             |                                                                                                    |
| get_random_number             | Set                              | Calculates derived random number                | mersenne_twister            | Random Word Generator         |                                                                                                    |
| Random Word Generator         | LangChain Agent                   | Converts random number to random English word  | get_random_number           | store_idea                   |                                                                                                    |
| OpenAI Chat Model - Word Generator | OpenAI Chat Model               | Provides GPT-4 backend for Random Word Generator | Random Word Generator       | Random Word Generator         |                                                                                                    |
| store_idea                   | Redis                            | Pushes generated word to Redis list             | Random Word Generator       | count_ideas                  |                                                                                                    |
| count_ideas                  | Redis                            | Increments word count with TTL                   | store_idea                  | get_count                    |                                                                                                    |
| get_count                   | Redis                            | Gets current word count from Redis                | count_ideas                 | check_number_of_ideas        |                                                                                                    |
| check_number_of_ideas        | If                               | Checks if word count threshold (≥36) reached    | get_count                   | extract_ideas / mersenne_twister |                                                                                                 |
| extract_ideas               | Redis                            | Pops one word from Redis list                     | check_number_of_ideas       | get_idea / check_queue_is_empty |                                                                                                    |
| get_idea                    | Redis                            | Retrieves cached message by key                    | extract_ideas               | set_idea                    |                                                                                                    |
| set_idea                    | Redis                            | Sets message cache with TTL                         | get_idea                    | check_queue_is_empty        |                                                                                                    |
| check_queue_is_empty        | If                               | Checks if extracted word is empty                  | set_idea                    | filtering / extract_ideas    |                                                                                                    |
| filtering                   | Set                              | Cleans concatenated keyword string                 | check_queue_is_empty        | Brainstorming               |                                                                                                    |
| Brainstorming               | LangChain Agent                   | Generates 5 innovative ideas based on problem and keywords | filtering                   | Critic                      |                                                                                                    |
| Google Gemini Chat Model - Brainstorming | Google Gemini Chat Model      | AI backend for Brainstorming agent                | Brainstorming               | Brainstorming               |                                                                                                    |
| Critic                     | LangChain Agent                   | Synthesizes 5 ideas into one refined solution     | Brainstorming / OpenAI Chat Model - Critic | -                           |                                                                                                    |
| OpenAI Chat Model - Critic | OpenAI Chat Model                 | GPT-4 backend for Critic agent                      | Critic                     | Critic                      |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**
   - Type: LangChain Chat Trigger
   - Name: `chat`
   - Configure webhook with ID `brainstorm-generator-webhook`
   - No special parameters

2. **Create Code Node for Mersenne Twister**
   - Type: Code (JavaScript)
   - Name: `mersenne_twister`
   - Paste MT19937 JS implementation using current timestamp seed
   - Generate two random numbers (`randomNumber`, `customRandomNumber`) per input
   - Add `seed` and timestamp metadata
   - Connect output from `chat`

3. **Create Set Node to Calculate Derived Number**
   - Type: Set
   - Name: `get_random_number`
   - Expression: `={{ ($json.randomNumber * $json.customRandomNumber * $json.seed) % 1000000 }}`
   - Output field: `number` (string)
   - Connect output from `mersenne_twister`

4. **Create LangChain Agent for Random Word Generation**
   - Type: LangChain Agent
   - Name: `Random Word Generator`
   - Input expression: `={{ $json.number }}`
   - System prompt: Define agent as random English word generator with detailed rules for entropy and lexical distribution
   - Connect output from `get_random_number`

5. **Create OpenAI Chat Node for Word Generator Backend**
   - Type: OpenAI Chat Model
   - Name: `OpenAI Chat Model - Word Generator`
   - Model: GPT-4
   - Parameters: temperature=1, topP=1, frequencyPenalty=1
   - Connect to AI Language Model input of `Random Word Generator`

6. **Create Redis Node to Push Word to List**
   - Type: Redis
   - Name: `store_idea`
   - Operation: Push to list `brainstorm`
   - Message: sanitized output from `Random Word Generator` (remove markdown and backticks)
   - Connect output from `Random Word Generator`

7. **Create Redis Node to Increment Word Count**
   - Type: Redis
   - Name: `count_ideas`
   - Operation: Increment key `brainstorm_count` with TTL 30 seconds (expire=true)
   - Connect output from `store_idea`

8. **Create Redis Node to Get Word Count**
   - Type: Redis
   - Name: `get_count`
   - Operation: Get key `brainstorm_count`
   - Output property: `count`
   - Connect output from `count_ideas`

9. **Create If Node to Check Word Count Threshold**
   - Type: If
   - Name: `check_number_of_ideas`
   - Condition: `count` (converted to number) >= 36
   - True branch: Connect to `extract_ideas`
   - False branch: Connect back to `mersenne_twister` for more random numbers
   - Connect from `get_count`

10. **Create Redis Node to Pop One Word**
    - Type: Redis
    - Name: `extract_ideas`
    - Operation: Pop from list `brainstorm` (tail)
    - Output property: `text`
    - Connect from True branch of `check_number_of_ideas`

11. **Create Redis Node to Get Cached Message**
    - Type: Redis
    - Name: `get_idea`
    - Operation: Get key from Redis string `message`
    - Output property: `message`
    - Connect from `extract_ideas`

12. **Create Redis Node to Set Cached Message**
    - Type: Redis
    - Name: `set_idea`
    - Operation: Set key `message` with concatenated previous message and popped word
    - TTL: 5 seconds
    - Connect from `get_idea`

13. **Create If Node to Check if Extracted Word is Empty**
    - Type: If
    - Name: `check_queue_is_empty`
    - Condition: Check if `text` (extracted word) is empty string
    - True branch: Connect to `filtering`
    - False branch: Connect back to `extract_ideas` for more popping
    - Connect from `set_idea`

14. **Create Set Node to Clean Up Concatenated Message**
    - Type: Set
    - Name: `filtering`
    - Expression: `={{ $json.message.replaceAll('\n\n','\n') }}`
    - Output field: `message`
    - Connect from True branch of `check_queue_is_empty`

15. **Create LangChain Agent for Brainstorming**
    - Type: LangChain Agent
    - Name: `Brainstorming`
    - Input expression:  
      ```
      Problem: {{ $('chat').first().json.chatInput }}
      Keywords: {{ $json.message }}
      ```
    - System prompt: Define "Ideation Specialist" persona to generate exactly 5 innovative ideas integrating keywords and problem, each with title, summary, details, and keywords
    - Connect from `filtering`

16. **Create Google Gemini Chat Model Node (Optional)**
    - Type: Google Gemini Chat Model
    - Name: `Google Gemini Chat Model - Brainstorming`
    - Model: `models/gemini-2.0-flash-exp`
    - Parameters: temperature=1, topP=1
    - Connect to AI language model input of `Brainstorming`

17. **Create LangChain Agent for Critic**
    - Type: LangChain Agent
    - Name: `Critic`
    - Input: `={{ $json.output }}`
    - System prompt: Define "Innovation Strategist" persona to critically evaluate and synthesize five ideas into one refined solution with structured output (title, summary, details, justification, risks)
    - Connect from `Brainstorming`

18. **Create OpenAI Chat Model for Critic Backend**
    - Type: OpenAI Chat Model
    - Name: `OpenAI Chat Model - Critic`
    - Model: GPT-4
    - Parameters: temperature=1, topP=1
    - Connect to AI language model input of `Critic`

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| Workflow author: Einar César Santos; Version 1.0.0                                                                                                                                                                                    | Welcome & Overview node                                   |
| Uses Mersenne Twister MT19937 algorithm for random number generation for high entropy and deterministic seeding when needed                                                                                                          | Mersenne Twister Explanation                             |
| Random Word Generator AI agent designed to produce exactly one English word with controlled lexical distribution (nouns, verbs, adjectives, adverbs, etc.) maximizing entropy and semantic diversity                                    | Word Generator Note                                      |
| Redis chosen for ultra-fast, in-memory temporary storage with TTL for automatic cleanup, serving as a "semantic buffer"                                                                                                              | Redis Storage Note                                       |
| Brainstorming Agent instructed to generate 5 ideas incorporating random keywords and problem statement, focusing on creativity and feasibility                                                                                       | Brainstorming Agent Note                                 |
| Critic Agent evaluates ideas on impact, viability, novelty, scalability, and risk, producing one refined solution                                                                                                                     | Critic Agent Note                                       |
| Configuration requires credentials for Redis and either OpenAI API (GPT-4) or Google Gemini API; recommends monitoring rate limits and token consumption                                                                              | Configuration Notes                                     |
| The workflow uses LangChain agents extensively, linking AI prompts with code and Redis data flow                                                                                                                                     | General workflow architecture                            |
| For performance, Redis persistence is recommended to be disabled and TTL values adjusted as needed                                                                                                                                   | Configuration Notes                                     |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.