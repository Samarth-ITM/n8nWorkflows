Generate AI-Summarized Newsletter Drafts from RSS Feeds with GPT-4 and Gmail

https://n8nworkflows.xyz/workflows/generate-ai-summarized-newsletter-drafts-from-rss-feeds-with-gpt-4-and-gmail-7264


# Generate AI-Summarized Newsletter Drafts from RSS Feeds with GPT-4 and Gmail

### 1. Workflow Overview

This workflow automates the generation of AI-summarized newsletter drafts from RSS feeds using GPT-4 and Gmail. It targets content curators, newsletter creators, and marketing teams who want to streamline the process of extracting relevant news articles, summarizing them intelligently, personalizing newsletter texts, and preparing email drafts ready for sending.

The workflow is logically divided into the following blocks:

- **1.1 RSS Feed Input Trigger:** Periodically monitors an RSS feed for new articles.
- **1.2 Information Extraction:** Uses an AI information extractor to parse and summarize article snippets and content.
- **1.3 AI Content Personalization:** Employs GPT-4 to personalize the extracted information into newsletter-style text.
- **1.4 Draft Email Creation:** Creates a Gmail draft email containing the personalized newsletter content.

Sticky notes accompany each block to explain their purpose and configuration guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 RSS Feed Input Trigger

**Overview:**  
This block initiates the workflow by polling an RSS feed every hour to detect new articles. It continuously monitors the feed URL and passes new items downstream for processing.

**Nodes Involved:**  
- RSS Feed Trigger  
- Sticky Note (RSS feed instructions)

**Node Details:**

- **RSS Feed Trigger**  
  - *Type & Role:* RSS feed reader trigger node that initiates workflow runs on new RSS feed entries.  
  - *Configuration:*  
    - Feed URL set to `https://www.artificialintelligence-news.com/feed/`  
    - Polling mode set to every hour, ensuring hourly checks for new feed items.  
  - *Expressions/Variables:* None used; static URL.  
  - *Connections:* Outputs to "Information Extractor" node.  
  - *Version:* 1 (standard RSS feed trigger node).  
  - *Edge Cases / Failure Types:*  
    - RSS feed downtime or URL unavailability causing trigger failures.  
    - Malformed feed data causing parsing errors.  
  - *Sticky Note:* Explains to place the RSS feed URL here.

- **Sticky Note (RSS feed instructions)**  
  - Content: "## RSS feed\n**URL** \nPut a RSS feed into this node"  
  - Covers the RSS Feed Trigger node, providing setup guidance.

---

#### 2.2 Information Extraction

**Overview:**  
This block extracts relevant structured information from the RSS feed items using an AI-powered information extractor node. It condenses the article snippets and content into a clear, concise summary in the preferred language.

**Nodes Involved:**  
- Information Extractor  
- OpenAI Chat Model (connected as an AI language model to the extractor)  
- Sticky Note (Information extract instructions)

**Node Details:**

- **Information Extractor**  
  - *Type & Role:* AI-powered information extraction node (from LangChain integration) that summarizes input text fields.  
  - *Configuration:*  
    - Input text combines snippet, title, and full article content using expressions:  
      ```
      Snippet: {{ $json['content:encodedSnippet'] }}
      Title: {{ $json.title }}
      Content: {{ $json['content:encoded'] }}
      ```  
    - System prompt instructs the AI to extract information clearly and briefly, translating into the preferred language.  
    - Attributes defined for Snippet, Title, and Content with descriptions to guide extraction.  
  - *Expressions:* Uses JSON path expressions to dynamically pull article data from the RSS item.  
  - *Connections:*  
    - Input: RSS Feed Trigger output.  
    - AI Language Model: OpenAI Chat Model node (as a language model backend).  
    - Output: Message a model node.  
  - *Version:* 1.2 (supports LangChain integration and advanced prompt configuration).  
  - *Edge Cases / Failure Types:*  
    - Failures due to incomplete or missing article fields (snippet, title, content).  
    - AI extraction errors or timeouts.  
    - Language translation inaccuracies if preferred language is changed improperly.  
  - *Sub-workflow:* Uses OpenAI Chat Model node as AI backend.

- **OpenAI Chat Model**  
  - *Type & Role:* AI language model node providing GPT-4 capabilities to the Information Extractor.  
  - *Configuration:*  
    - Model set to "gpt-4.1-mini" (a version of GPT-4 optimized for smaller tasks).  
    - No additional options configured.  
  - *Credentials:* Uses stored OpenAI API credentials.  
  - *Connections:* Connected to Information Extractor's AI language model input.  
  - *Version:* 1.2.  
  - *Edge Cases / Failure Types:*  
    - API authentication or quota limits.  
    - Model unavailability or latency.  
    - Prompt interpretation inconsistencies.

- **Sticky Note (Information extract instructions)**  
  - Content:  
    ```
    ## Information extract

    This node will extract information about article.

    In system prompt change language to your prefered one!
    ```  
  - Positioned near the Information Extractor node to guide language preference and usage.

---

#### 2.3 AI Content Personalization

**Overview:**  
This block takes the extracted article information and uses GPT-4 to generate personalized newsletter content, applying a defined tone of voice and assistant persona.

**Nodes Involved:**  
- Message a model  
- Sticky Note (Message a model instructions)

**Node Details:**

- **Message a model**  
  - *Type & Role:* OpenAI GPT-4 chat node used to craft personalized newsletter messages based on extracted content.  
  - *Configuration:*  
    - Model: "gpt-4.1-mini".  
    - Messages:  
      - System role message instructs the assistant (named Patrik) to generate newsletter content from the article content provided by the extractor. Uses expression to inject extracted content:  
        ```
        Your task is get information about article, use assistant role named Patrik to give personalisation and create a newsletter content.
        Content si here:
        {{ $json.output.Content }}
        ```  
      - Assistant role message includes a placeholder for tone of voice: "Here give your tone of voice!" which can be customized.  
  - *Expressions:* Uses extracted content dynamically from previous node output.  
  - *Connections:* Outputs to "Create a draft" node.  
  - *Credentials:* Uses same OpenAI API credential as before.  
  - *Version:* 1.8 (supports advanced chat message formatting).  
  - *Edge Cases / Failure Types:*  
    - Content injection errors if extractor output is missing or malformed.  
    - API quota or authentication issues.  
    - Tone of voice customization missing or unclear.  
  - *Sticky Note:* Explains the purpose of system and assistant prompts for personalization.

- **Sticky Note (Message a model instructions)**  
  - Content:  
    ```
    ## Message a model

    System prompt is get data from extractor. 

    Asistant prompt should be your tone of voice to personalised content!
    ```  
  - Positioned next to "Message a model" node.

---

#### 2.4 Draft Email Creation

**Overview:**  
This block uses the Gmail node to create a draft email with the personalized newsletter content, ready to review and send.

**Nodes Involved:**  
- Create a draft  
- Sticky Note (Create a draft instructions)

**Node Details:**

- **Create a draft**  
  - *Type & Role:* Gmail node configured to create an email draft in the user's Gmail account.  
  - *Configuration:*  
    - Resource: "draft" (creates a draft email instead of sending immediately).  
    - Email type: HTML format.  
    - Subject: Static text "Newsletter".  
    - Message body: Injected dynamically from the previous node's message content (`={{ $json.message.content }}`).  
  - *Credentials:* Uses OAuth2 Gmail credentials configured for user `Patrik@descodino.studio`.  
  - *Connections:* Input from "Message a model" node.  
  - *Version:* 2.1.  
  - *Edge Cases / Failure Types:*  
    - OAuth token expiration or invalid credentials.  
    - Gmail API quota or connectivity issues.  
    - HTML content rendering problems in draft preview.  
  - *Sticky Note:* Explains the draft creation purpose.

- **Sticky Note (Create a draft instructions)**  
  - Content:  
    ```
    ## Create a draft

    Its created a draft, with ready to sent!
    ```  
  - Positioned adjacent to the "Create a draft" node.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                        | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                         |
|---------------------|----------------------------------|-------------------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------------|
| RSS Feed Trigger     | RSS Feed Read Trigger             | Polls RSS feed for new articles     | -                     | Information Extractor  | ## RSS feed\n**URL** \nPut a RSS feed into this node                                              |
| Information Extractor| LangChain Information Extractor  | Extracts and summarizes article info| RSS Feed Trigger       | Message a model        | ## Information extract\n\nThis node will extract information about article.\n\nIn system prompt change language to your prefered one! |
| OpenAI Chat Model    | LangChain OpenAI Chat Model      | AI backend for extraction            | -                     | Information Extractor (AI input) |                                                                                                   |
| Message a model      | LangChain OpenAI Chat Node       | Personalizes newsletter content      | Information Extractor  | Create a draft         | ## Message a model\n\nSystem prompt is get data from extractor. \n\nAsistant prompt should be your tone of voice to personalised content! |
| Create a draft       | Gmail Node                       | Creates Gmail draft email            | Message a model        | -                     | ## Create a draft\n\nIts created a draft, with ready to sent!                                     |
| Sticky Note          | Sticky Note                      | Guidance for RSS Feed node           | -                     | -                     | ## RSS feed\n**URL** \nPut a RSS feed into this node                                              |
| Sticky Note1         | Sticky Note                      | Guidance for Information Extractor   | -                     | -                     | ## Information extract\n\nThis node will extract information about article.\n\nIn system prompt change language to your prefered one! |
| Sticky Note2         | Sticky Note                      | Guidance for Message a model node    | -                     | -                     | ## Message a model\n\nSystem prompt is get data from extractor. \n\nAsistant prompt should be your tone of voice to personalised content! |
| Sticky Note3         | Sticky Note                      | Guidance for Create a draft node     | -                     | -                     | ## Create a draft\n\nIts created a draft, with ready to sent!                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger node**  
   - Type: RSS Feed Read Trigger  
   - Parameters:  
     - Feed URL: `https://www.artificialintelligence-news.com/feed/`  
     - Poll Times: set to every hour  
   - No credentials needed  
   - Connect output to "Information Extractor".

2. **Add Information Extractor node**  
   - Type: LangChain Information Extractor (version 1.2 or later)  
   - Parameters:  
     - Text:  
       ```
       Snippet: {{ $json['content:encodedSnippet'] }}
       Title: {{ $json.title }}
       Content: {{ $json['content:encoded'] }}
       ```  
     - System Prompt:  
       ```
       Your task is a get information from content snippet, and translate it in {your prefer} language.  
       Keep it clear and short about what is in content and title.
       ```  
     - Attributes: Define three attributes named Snippet, Title, and Content with their respective descriptions.  
   - Connect AI Language Model input to the "OpenAI Chat Model" node.  
   - Connect main output to "Message a model".

3. **Create OpenAI Chat Model node**  
   - Type: LangChain OpenAI Chat Model (version 1.2 or later)  
   - Parameters:  
     - Model: Select "gpt-4.1-mini" from the list  
   - Credentials: Configure with OpenAI API key  
   - Connect output as AI input to the Information Extractor node.

4. **Add Message a model node**  
   - Type: LangChain OpenAI Chat Node (version 1.8 or later)  
   - Parameters:  
     - Model ID: "gpt-4.1-mini"  
     - Messages:  
       - Role: system  
         Content (use expression):  
         ```
         Your task is get information about article, use assistant role named Patrik to give personalisation and create a newsletter content.
         Content si here:
         {{ $json.output.Content }}
         ```  
       - Role: assistant  
         Content: "Here give your tone of voice!" (customize as desired)  
   - Credentials: Use same OpenAI API credentials  
   - Connect main output to "Create a draft".

5. **Create Create a draft node (Gmail)**  
   - Type: Gmail node (version 2.1 or later)  
   - Parameters:  
     - Resource: draft  
     - Email Type: HTML  
     - Subject: "Newsletter"  
     - Message: Use expression `={{ $json.message.content }}` to pull from previous node output  
   - Credentials: Configure Gmail OAuth2 credentials for your Gmail account  
   - This node is the final step.

6. **Add Sticky Notes for guidance (optional but recommended)**  
   - Near RSS Feed Trigger: Note to place RSS feed URL  
   - Near Information Extractor: Note about changing preferred language in system prompt  
   - Near Message a model: Note about system and assistant prompts for personalization  
   - Near Create a draft: Note explaining draft creation

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses LangChain nodes integrated with OpenAI GPT-4 models for advanced NLP processing.                                      | LangChain documentation: https://js.langchain.com/docs/                                         |
| Gmail node creates drafts rather than sending emails immediately, allowing review and edits before sending.                            | Gmail API docs: https://developers.google.com/gmail/api                                           |
| Personalize the assistant’s tone of voice by editing the assistant role message in the "Message a model" node.                           |                                                                                                 |
| Ensure OpenAI API and Gmail OAuth2 credentials are properly set up with required API scopes and permissions for smooth execution.       |                                                                                                 |
| Use the system prompt in the Information Extractor node to specify the target language for summaries.                                    |                                                                                                 |
| Recommended to monitor API usage quotas to avoid interruptions due to limits or expired credentials.                                     |                                                                                                 |

---

**Disclaimer:**  
The text and configuration provided are exclusively extracted from an automated n8n workflow built with n8n, respecting all content policies. The data handled is legal and public.