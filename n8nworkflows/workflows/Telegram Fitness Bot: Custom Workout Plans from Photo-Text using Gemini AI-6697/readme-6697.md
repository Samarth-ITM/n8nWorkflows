Telegram Fitness Bot: Custom Workout Plans from Photo/Text using Gemini AI

https://n8nworkflows.xyz/workflows/telegram-fitness-bot--custom-workout-plans-from-photo-text-using-gemini-ai-6697


# Telegram Fitness Bot: Custom Workout Plans from Photo/Text using Gemini AI

---

### 1. Workflow Overview

This workflow, titled **"Telegram Fitness Bot: Custom Workout Plans from Photo/Text using Gemini AI"**, is designed to provide personalized workout plans to users on Telegram based on either text input or images they send. The core purpose is to leverage Google Gemini AI models to analyze user inputs—text descriptions or images—and generate tailored fitness guidance. The workflow automates the process of receiving Telegram messages, discerning input type, processing inputs with AI, formatting workout plans for readability, and delivering the plans back to users via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Classification:** Captures incoming Telegram messages and determines whether the input is text or a photo.
- **1.2 Image Processing Pipeline:** For photo inputs, handles image retrieval, MIME type adjustment, AI-powered analysis, and workout plan generation.
- **1.3 Text Processing Pipeline:** For text inputs, sends the text to AI to generate workout plans.
- **1.4 Workout Plan Formatting:** Cleans and formats AI-generated workout plans to be user-friendly and visually appealing.
- **1.5 Response Delivery:** Sends the final workout plans back to users on Telegram.
- **1.6 AI Memory Integration:** Maintains conversational context for both text and image-based interactions using a memory buffer.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Classification

- **Overview:**  
  This block listens for incoming Telegram messages and routes them based on whether the message contains text or an image.

- **Nodes Involved:**  
  - Receive a Message (Telegram Trigger)  
  - Text Vs Photo (Switch)

- **Node Details:**

  - **Receive a Message**  
    - *Type & Role:* Telegram Trigger node — initiates workflow on incoming Telegram messages.  
    - *Config:* Default parameters with webhook linked to Telegram bot.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Passes message data downstream.  
    - *Failures:* Possible webhook or Telegram API connectivity issues.  
    - *Notes:* Entry point of the workflow.

  - **Text Vs Photo**  
    - *Type & Role:* Switch node — routes flow depending on content type (text vs photo).  
    - *Config:* Condition checks message payload for presence of text or photo.  
    - *Inputs:* Output from Receive a Message.  
    - *Outputs:*  
      - 1st output: Text path  
      - 2nd output: Photo path  
    - *Failures:* Incorrect condition logic could misroute inputs.

#### 2.2 Image Processing Pipeline

- **Overview:**  
  For photo inputs, this block fetches the image, adjusts its MIME type, analyzes it via Google Gemini AI, and generates a workout plan based on the image content.

- **Nodes Involved:**  
  - Get a Image (Telegram)  
  - Fetch Image Path (HTTP Request)  
  - Change Mime Type (Code)  
  - Analyze image (Google Gemini)  
  - Workout Guide thorugh Image (Langchain Agent)  
  - Clean up Workout plane and Make User freindly And Visually appealing1 (Code)  
  - Send A Workout Planes1 (Telegram)

- **Node Details:**

  - **Get a Image**  
    - *Type & Role:* Telegram node — extracts image data from user message.  
    - *Config:* Linked to the same Telegram bot webhook.  
    - *Outputs:* Passes image metadata.  
    - *Failures:* Missing or corrupted image data.

  - **Fetch Image Path**  
    - *Type & Role:* HTTP Request — downloads the image using URL obtained from Telegram.  
    - *Config:* Uses HTTP GET to fetch the image file.  
    - *Inputs:* Image URL from Get a Image.  
    - *Outputs:* Binary image data.  
    - *Failures:* Network errors, invalid URLs, timeouts.

  - **Change Mime Type**  
    - *Type & Role:* Code node — modifies the MIME type to ensure compatibility with AI model.  
    - *Config:* Adjusts MIME to expected format (e.g., image/jpeg or image/png).  
    - *Inputs:* Binary image from Fetch Image Path.  
    - *Outputs:* Modified binary data with correct MIME.  
    - *Failures:* Code errors or invalid MIME data.

  - **Analyze image**  
    - *Type & Role:* Google Gemini node — performs AI-based image analysis to extract fitness-related insights.  
    - *Config:* Default Gemini parameters (assumed).  
    - *Inputs:* Image data from Change Mime Type.  
    - *Outputs:* AI-generated interpretation of image.  
    - *Failures:* AI service downtime, authentication errors.

  - **Workout Guide thorugh Image**  
    - *Type & Role:* Langchain Agent — generates structured workout plans based on analyzed image data.  
    - *Config:* Uses Google Gemini Chat Model as language model backend.  
    - *Inputs:* AI interpretation from Analyze image; memory context from Simple Memory.  
    - *Outputs:* Raw workout plan text.  
    - *Failures:* Model errors or timeout.

  - **Clean up Workout plane and Make User freindly And Visually appealing1**  
    - *Type & Role:* Code node — formats and enhances workout plan text for clarity and presentation.  
    - *Config:* Contains JavaScript logic for text cleanup and formatting.  
    - *Inputs:* Raw workout plan from Workout Guide thorugh Image.  
    - *Outputs:* Cleaned, user-friendly workout plan text.  
    - *Failures:* Code exceptions or malformed input text.

  - **Send A Workout Planes1**  
    - *Type & Role:* Telegram node — sends formatted workout plan back to user.  
    - *Config:* Uses Telegram bot credentials; sends message to user chat ID.  
    - *Inputs:* Formatted workout plan text.  
    - *Outputs:* Confirmation of message sent.  
    - *Failures:* Telegram API errors, user blocking bot.

#### 2.3 Text Processing Pipeline

- **Overview:**  
  For textual inputs, this block sends the user’s text to the AI model, generates a workout plan, cleans it up, and sends it back to the user.

- **Nodes Involved:**  
  - Workout Guide (Langchain Agent)  
  - Google Gemini Chat Model1 (AI Language Model)  
  - Clean up Workout plane and Make User freindly And Visually appealing (Code)  
  - Send A Workout Planes (Telegram)

- **Node Details:**

  - **Workout Guide**  
    - *Type & Role:* Langchain Agent — generates workout plans from text inputs.  
    - *Config:* Uses Google Gemini Chat Model1 as language model backend; connected to conversational memory.  
    - *Inputs:* Text from Text Vs Photo switch; memory context from Simple Memory.  
    - *Outputs:* Raw workout plan text.  
    - *Failures:* Model errors, input format issues.

  - **Google Gemini Chat Model1**  
    - *Type & Role:* AI Language Model node — Google Gemini chat interface for textual prompts.  
    - *Config:* Default parameters linked to Langchain Agent (Workout Guide).  
    - *Inputs:* Prompt from Workout Guide agent.  
    - *Outputs:* AI response text.  
    - *Failures:* Authentication, rate limits.

  - **Clean up Workout plane and Make User freindly And Visually appealing**  
    - *Type & Role:* Code node — formats AI output for readability and appearance.  
    - *Config:* JavaScript code for text formatting.  
    - *Inputs:* Raw workout plan from Workout Guide.  
    - *Outputs:* User-friendly formatted workout plan.  
    - *Failures:* Code errors.

  - **Send A Workout Planes**  
    - *Type & Role:* Telegram node — sends formatted workout plan back to the user.  
    - *Config:* Uses Telegram bot credentials and chat ID.  
    - *Inputs:* Formatted workout plan text.  
    - *Outputs:* Message sent confirmation.  
    - *Failures:* Telegram API issues.

#### 2.4 AI Memory Integration

- **Overview:**  
  Maintains a short-term memory buffer to provide conversational context to AI agents for both text and image inputs, enhancing response relevance.

- **Nodes Involved:**  
  - Simple Memory (Langchain memoryBufferWindow)

- **Node Details:**

  - **Simple Memory**  
    - *Type & Role:* Langchain memory buffer — stores recent conversation context.  
    - *Config:* Default window size for memory context.  
    - *Inputs:* Connected as ai_memory to both Workout Guide and Workout Guide thorugh Image nodes.  
    - *Outputs:* Provides memory context to AI agents.  
    - *Failures:* Memory overflow or data corruption unlikely.

---

### 3. Summary Table

| Node Name                                                     | Node Type                              | Functional Role                                     | Input Node(s)                     | Output Node(s)                                         | Sticky Note                                                                                  |
|---------------------------------------------------------------|--------------------------------------|----------------------------------------------------|----------------------------------|--------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Receive a Message                                             | Telegram Trigger                     | Entry point: receives Telegram messages            | None                             | Text Vs Photo                                          |                                                                                              |
| Text Vs Photo                                                | Switch                              | Routes flow to text or photo processing             | Receive a Message                | Workout Guide (text path), Get a Image (photo path)    |                                                                                              |
| Get a Image                                                  | Telegram                            | Extracts image info from Telegram message           | Text Vs Photo                   | Fetch Image Path                                       |                                                                                              |
| Fetch Image Path                                             | HTTP Request                       | Downloads image from Telegram URL                    | Get a Image                    | Change Mime Type                                      |                                                                                              |
| Change Mime Type                                             | Code                               | Adjusts image MIME type for AI model compatibility | Fetch Image Path               | Analyze image                                         |                                                                                              |
| Analyze image                                                | Google Gemini                      | AI analyzes image content                            | Change Mime Type               | Workout Guide thorugh Image                           |                                                                                              |
| Workout Guide thorugh Image                                  | Langchain Agent                   | Creates workout plan from image analysis             | Analyze image, Simple Memory    | Clean up Workout plane and Make User freindly And Visually appealing1 |                                                                                              |
| Clean up Workout plane and Make User freindly And Visually appealing1 | Code                               | Formats workout plan text for readability           | Workout Guide thorugh Image     | Send A Workout Planes1                                |                                                                                              |
| Send A Workout Planes1                                       | Telegram                           | Sends workout plan message to Telegram user         | Clean up Workout plane and Make User freindly And Visually appealing1 | None                                                   |                                                                                              |
| Workout Guide                                               | Langchain Agent                   | Creates workout plan from text input                 | Text Vs Photo                  | Clean up Workout plane and Make User freindly And Visually appealing |                                                                                              |
| Google Gemini Chat Model1                                   | AI Language Model                 | Provides AI language model for text input            | Workout Guide                 | Workout Guide                                        |                                                                                              |
| Clean up Workout plane and Make User freindly And Visually appealing | Code                               | Formats workout plan text for readability           | Workout Guide                | Send A Workout Planes                                |                                                                                              |
| Send A Workout Planes                                       | Telegram                           | Sends workout plan message to Telegram user         | Clean up Workout plane and Make User freindly And Visually appealing | None                                                   |                                                                                              |
| Simple Memory                                               | Langchain memoryBufferWindow      | Maintains short-term conversational memory          | None                         | Workout Guide, Workout Guide thorugh Image (as ai_memory) |                                                                                              |
| Google Gemini Chat Model                                   | AI Language Model                 | Provides AI language model for image processing      | Workout Guide thorugh Image   | Workout Guide thorugh Image                           |                                                                                              |
| Sticky Note                                                | Sticky Note                      | No content                                           | None                         | None                                                  |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Setup Telegram Bot Credentials:**  
   - Configure Telegram bot with proper webhook URL and credentials for trigger and send nodes.

2. **Create Telegram Trigger Node:**  
   - Add a **Telegram Trigger** node named "Receive a Message."  
   - Configure to listen for new messages.

3. **Add Switch Node for Input Type Detection:**  
   - Add a **Switch** node named "Text Vs Photo."  
   - Configure conditions to check if incoming message contains text or photo.

4. **Text Input Path:**  
   - Add a **Langchain Agent** node named "Workout Guide."  
     - Set its language model input to a **Google Gemini Chat Model** node named "Google Gemini Chat Model1."  
   - Link "Text Vs Photo" (text output) → "Workout Guide."  
   - Add a **Code** node named "Clean up Workout plane and Make User freindly And Visually appealing."  
     - Implement JavaScript code to format AI output text.  
   - Connect "Workout Guide" → "Clean up Workout plane and Make User freindly And Visually appealing."  
   - Add a **Telegram** node named "Send A Workout Planes."  
     - Configure to send message to user's chat ID.  
   - Connect "Clean up Workout plane and Make User freindly And Visually appealing" → "Send A Workout Planes."

5. **Photo Input Path:**  
   - Add a **Telegram** node named "Get a Image."  
     - Configure to extract photo info from message.  
   - Connect "Text Vs Photo" (photo output) → "Get a Image."  
   - Add an **HTTP Request** node named "Fetch Image Path."  
     - Configure for HTTP GET with the image URL from "Get a Image."  
   - Connect "Get a Image" → "Fetch Image Path."  
   - Add a **Code** node named "Change Mime Type."  
     - Adjust MIME type to expected format for AI processing.  
   - Connect "Fetch Image Path" → "Change Mime Type."  
   - Add a **Google Gemini** node named "Analyze image."  
     - Configure for image analysis.  
   - Connect "Change Mime Type" → "Analyze image."  
   - Add a **Langchain Agent** node named "Workout Guide thorugh Image."  
     - Set language model input to **Google Gemini Chat Model** node named "Google Gemini Chat Model."  
   - Connect "Analyze image" → "Workout Guide thorugh Image."  
   - Add a **Code** node named "Clean up Workout plane and Make User freindly And Visually appealing1."  
     - Include formatting JavaScript code for output text.  
   - Connect "Workout Guide thorugh Image" → "Clean up Workout plane and Make User freindly And Visually appealing1."  
   - Add a **Telegram** node named "Send A Workout Planes1."  
     - Configure to send message to user's chat ID.  
   - Connect "Clean up Workout plane and Make User freindly And Visually appealing1" → "Send A Workout Planes1."

6. **Add AI Memory Node:**  
   - Add a **Langchain memoryBufferWindow** node named "Simple Memory."  
   - Connect its output as `ai_memory` input to both "Workout Guide" and "Workout Guide thorugh Image" nodes.

7. **Configure Credentials:**  
   - Setup credentials for:  
     - Telegram (OAuth2 or Bot Token)  
     - Google Gemini AI (API keys / OAuth as required).

8. **Finalize Node Connections and Test:**  
   - Ensure all nodes are connected as per above.  
   - Test sending text and photo messages via Telegram to verify workflow operation.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow leverages Google Gemini AI models for both text and image understanding to generate plans. | Requires valid Google Gemini API credentials and Telegram Bot setup.                            |
| Node names are descriptive to clarify function, aiding maintenance and modification.                      | Useful naming conventions for complex workflows.                                               |
| Use the Langchain Agent and AI Memory nodes to maintain conversational context enhancing user experience. | Langchain integration docs: https://docs.n8n.io/nodes/agents/langchain/                        |
| Telegram webhook setup requires public URL and correct webhook registration with BotFather.             | Telegram Bot API docs: https://core.telegram.org/bots/api                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---