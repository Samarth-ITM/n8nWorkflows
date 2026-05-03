WhatsApp Dietitian AI Chatbot Workflow

https://n8nworkflows.xyz/workflows/whatsapp-dietitian-ai-chatbot-workflow-4858


# WhatsApp Dietitian AI Chatbot Workflow

### 1. Workflow Overview

This workflow enables a WhatsApp-based AI dietitian chatbot, which processes user messages sent via WhatsApp, interprets both text and images, generates contextual dietary advice using Google Gemini AI models, and responds back through WhatsApp. It is designed to handle mixed media input (text and images), parse AI responses into structured formats, and provide personalized, AI-driven health and diet recommendations.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Receive incoming WhatsApp messages, determine if they contain images or text.
- **1.2 Media Handling:** If images are present, retrieve and download the image for analysis.
- **1.3 AI Processing:** Use Google Gemini LLM to generate responses based on either text messages or image+caption input, employing structured output parsing.
- **1.4 Response Dispatch:** Send the generated AI response back to the user on WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow when a WhatsApp message is received and decides the processing path depending on whether the message contains an image.

**Nodes Involved:**  
- WhatsApp Message Received  
- Check if message contains image

**Node Details:**  

- **WhatsApp Message Received**  
  - *Type:* WhatsApp Trigger  
  - *Role:* Entry point, listens for incoming WhatsApp messages via webhook.  
  - *Configuration:* Uses a webhook with ID `e90274bf-7949-436a-8429-2844ce22cf2d`. No additional parameters specified.  
  - *Input/Output:* No inputs; outputs raw message data. Connected to "Check if message contains image".  
  - *Edge Cases:* Possible webhook misconfiguration or network downtime causing missed messages.

- **Check if message contains image**  
  - *Type:* Switch node  
  - *Role:* Routes workflow based on message content type (image vs text).  
  - *Configuration:* Conditions based on presence of image data in incoming message payload.  
  - *Input/Output:* Input from WhatsApp Message Received; outputs to either "Get whatsapp image" (image present) or "Generate Response from text" (text only).  
  - *Edge Cases:* Incorrect detection of image presence, leading to wrong processing path.

---

#### 2.2 Media Handling

**Overview:**  
Handles retrieval and downloading of WhatsApp images for AI processing.

**Nodes Involved:**  
- Get whatsapp image  
- Download Image

**Node Details:**  

- **Get whatsapp image**  
  - *Type:* WhatsApp node  
  - *Role:* Retrieves image media attached to the WhatsApp message.  
  - *Configuration:* Uses webhook ID `378c59ca-2b57-460f-94d2-38cd115f296d`. No further parameters specified.  
  - *Input/Output:* Input from "Check if message contains image"; output to "Download Image".  
  - *Edge Cases:* Failures in media retrieval due to expired media URL or permissions.

- **Download Image**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the image from the URL obtained in the previous node.  
  - *Configuration:* Standard HTTP GET request to image URL.  
  - *Input/Output:* Input from "Get whatsapp image"; output to "Generate Response from Image and caption".  
  - *Edge Cases:* Network errors, invalid or expired URLs, large image sizes causing timeouts.

---

#### 2.3 AI Processing

**Overview:**  
Uses Google Gemini LLM and output parsers to generate AI responses based on message content type: text-only or image with caption.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser  
- Generate Response from text  
- Generate Response from Image and caption

**Node Details:**  

- **Google Gemini Chat Model**  
  - *Type:* Langchain Google Gemini LLM node  
  - *Role:* Provides the core AI language model interaction for both text and image caption inputs.  
  - *Configuration:* No explicit parameters, uses default or pre-configured credentials for Google Gemini.  
  - *Input/Output:* Serves as AI language model for "Generate Response from text" and "Generate Response from Image and caption".  
  - *Edge Cases:* Authentication failures, API rate limits, model unavailability.

- **Structured Output Parser**  
  - *Type:* Langchain structured output parser  
  - *Role:* Parses raw AI output into structured data formats for consistency and easier downstream handling.  
  - *Configuration:* No explicit parameters shown.  
  - *Input/Output:* Used by both "Generate Response from text" and "Generate Response from Image and caption" as output parser.  
  - *Edge Cases:* Parsing errors if AI output deviates from expected structure.

- **Generate Response from text**  
  - *Type:* Langchain chain LLM node  
  - *Role:* Generates AI response based solely on incoming text messages.  
  - *Configuration:* Uses Google Gemini as language model and structured output parser.  
  - *Input/Output:* Input from "Check if message contains image" (text path); output to "Send Response to client".  
  - *Edge Cases:* Failures in model invocation or output parsing.

- **Generate Response from Image and caption**  
  - *Type:* Langchain chain LLM node  
  - *Role:* Generates AI response based on image data and associated caption text.  
  - *Configuration:* Uses Google Gemini and structured output parser similarly.  
  - *Input/Output:* Input from "Download Image"; output to "Send Response to client".  
  - *Edge Cases:* Failures in image processing context or parsing.

---

#### 2.4 Response Dispatch

**Overview:**  
Sends the AI-generated response back to the WhatsApp user.

**Nodes Involved:**  
- Send Response to client

**Node Details:**  

- **Send Response to client**  
  - *Type:* WhatsApp node  
  - *Role:* Delivers the AI-generated message back to the user via WhatsApp.  
  - *Configuration:* Uses webhook ID `8faff3bf-f15e-476d-99ec-d69c2a9c8304`. No other parameters specified.  
  - *Input/Output:* Input from either "Generate Response from text" or "Generate Response from Image and caption".  
  - *Edge Cases:* Failures in message delivery due to WhatsApp API issues or connectivity.

---

### 3. Summary Table

| Node Name                     | Node Type                             | Functional Role                        | Input Node(s)                | Output Node(s)                      | Sticky Note                          |
|-------------------------------|-------------------------------------|-------------------------------------|-----------------------------|-----------------------------------|------------------------------------|
| WhatsApp Message Received      | WhatsApp Trigger                    | Entry point, receives WhatsApp msgs | -                           | Check if message contains image    |                                    |
| Check if message contains image| Switch                             | Routes based on presence of image   | WhatsApp Message Received    | Get whatsapp image, Generate Response from text |                                    |
| Get whatsapp image             | WhatsApp node                      | Retrieves image media from WhatsApp | Check if message contains image | Download Image                   |                                    |
| Download Image                | HTTP Request                       | Downloads the image file             | Get whatsapp image           | Generate Response from Image and caption |                                    |
| Google Gemini Chat Model       | Langchain Google Gemini LLM        | Core AI model for both text and image | -                           | Generate Response from Image and caption, Generate Response from text |                                    |
| Structured Output Parser       | Langchain Output Parser            | Parses AI output into structured data | -                           | Generate Response from Image and caption, Generate Response from text |                                    |
| Generate Response from text    | Langchain Chain LLM                | Generates AI response from text     | Check if message contains image | Send Response to client           |                                    |
| Generate Response from Image and caption | Langchain Chain LLM          | Generates AI response from image+caption | Download Image              | Send Response to client           |                                    |
| Send Response to client        | WhatsApp node                     | Sends AI response back to user      | Generate Response from text, Generate Response from Image and caption | -                             |                                    |
| Sticky Note                   | Sticky Note                       | (No content)                        | -                           | -                                 |                                    |
| Sticky Note1                  | Sticky Note                       | (No content)                        | -                           | -                                 |                                    |
| Sticky Note2                  | Sticky Note                       | (No content)                        | -                           | -                                 |                                    |
| Sticky Note3                  | Sticky Note                       | (No content)                        | -                           | -                                 |                                    |
| Sticky Note4                  | Sticky Note                       | (No content)                        | -                           | -                                 |                                    |
| Sticky Note5                  | Sticky Note                       | (No content)                        | -                           | -                                 |                                    |
| Sticky Note6                  | Sticky Note                       | (No content)                        | -                           | -                                 |                                    |
| Sticky Note7                  | Sticky Note                       | (No content)                        | -                           | -                                 |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Name: `WhatsApp Message Received`  
   - Type: WhatsApp Trigger  
   - Configure webhook ID (e.g., `e90274bf-7949-436a-8429-2844ce22cf2d`) to receive incoming messages.  
   - Connect output to next node.

2. **Add a Switch Node to Check for Image Presence**  
   - Name: `Check if message contains image`  
   - Type: Switch  
   - Configure condition to check if the incoming message payload contains an image attachment.  
   - Create two outputs: one for messages with images, one for text-only messages.  
   - Connect WhatsApp Message Received to this node.

3. **For Image Path: Add WhatsApp Node to Retrieve Image**  
   - Name: `Get whatsapp image`  
   - Type: WhatsApp node  
   - Configure webhook ID for media retrieval (e.g., `378c59ca-2b57-460f-94d2-38cd115f296d`).  
   - Connect the "image present" output of switch to this node.

4. **Download Image via HTTP Request Node**  
   - Name: `Download Image`  
   - Type: HTTP Request  
   - Configure to perform GET request to the image URL obtained from the previous node.  
   - Connect output of `Get whatsapp image` to this node.

5. **Add Langchain Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Type: Langchain Google Gemini LLM  
   - Configure with valid Google Gemini credentials.  
   - No additional parameters unless needed for custom prompts.

6. **Add Langchain Structured Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Type: Langchain Output Parser Structured  
   - Configure output parser settings as needed for parsing AI responses into structured format.  

7. **Create Langchain Chain LLM Node for Image + Caption**  
   - Name: `Generate Response from Image and caption`  
   - Type: Langchain Chain LLM  
   - Configure to use `Google Gemini Chat Model` as language model and `Structured Output Parser` as output parser.  
   - Input connected from `Download Image`.  

8. **Create Langchain Chain LLM Node for Text**  
   - Name: `Generate Response from text`  
   - Type: Langchain Chain LLM  
   - Configure to use `Google Gemini Chat Model` and `Structured Output Parser`.  
   - Input connected from the "text only" output of `Check if message contains image`.  

9. **Add WhatsApp Node to Send Response**  
   - Name: `Send Response to client`  
   - Type: WhatsApp node  
   - Configure webhook ID for sending messages back to user (e.g., `8faff3bf-f15e-476d-99ec-d69c2a9c8304`).  
   - Connect outputs from both `Generate Response from text` and `Generate Response from Image and caption` to this node.

10. **Link Nodes Properly**  
    - `WhatsApp Message Received` → `Check if message contains image`  
    - `Check if message contains image` (image path) → `Get whatsapp image` → `Download Image` → `Generate Response from Image and caption` → `Send Response to client`  
    - `Check if message contains image` (text path) → `Generate Response from text` → `Send Response to client`  
    - `Google Gemini Chat Model` and `Structured Output Parser` connected as AI model and parser references in both LLM chain nodes.

11. **Credential Setup**  
    - Ensure WhatsApp trigger and message nodes are authenticated with valid WhatsApp integration credentials.  
    - Set up Google Gemini API credentials and permissions for Langchain nodes.

12. **Defaults and Constraints**  
    - Use reasonable timeouts on HTTP Request node for image download.  
    - Set proper error handling on switch node to manage unexpected message types.  
    - Validate AI response parsing logic to handle unexpected outputs gracefully.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                          |
|-----------------------------------------------------------------------------------------------|----------------------------------------|
| Workflow leverages Google Gemini Chat Model via Langchain integration for AI response generation | Langchain documentation: https://js.langchain.com/docs/ |
| WhatsApp integration requires proper webhook setup and permissions from WhatsApp Business API | WhatsApp Business API docs: https://developers.facebook.com/docs/whatsapp/ |
| Structured Output Parser ensures AI outputs conform to expected JSON or structured format     | Useful for downstream processing and reliability |
| Workflow is designed for health/diet chatbot scenarios with multimedia input support          | Can be adapted to other conversational AI use cases |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. All processing complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.