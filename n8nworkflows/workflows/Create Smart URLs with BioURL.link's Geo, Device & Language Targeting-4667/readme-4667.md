Create Smart URLs with BioURL.link's Geo, Device & Language Targeting

https://n8nworkflows.xyz/workflows/create-smart-urls-with-biourl-link-s-geo--device---language-targeting-4667


# Create Smart URLs with BioURL.link's Geo, Device & Language Targeting

---

### 1. Workflow Overview

This workflow creates smart, shortened URLs using the BioURL.link API with advanced targeting features such as geolocation, device type, and language. It is designed to accept a POST request containing URL parameters, forward these details to the BioURL.link API to generate a customized short link, and respond with the API's output.

**Target Use Cases:**  
- Marketing campaigns requiring URL shortening with customized redirection based on user geography, device, or language.  
- Automation of URL generation for multi-channel distribution with integrated tracking and expiry options.  

**Logical Blocks:**  
- **1.1 Input Reception:** Accepts incoming HTTP POST requests with URL creation parameters.  
- **1.2 API Request to BioURL.link:** Sends a detailed POST request to the BioURL.link API to create a smart URL with advanced targeting capabilities.  
- **1.3 Response Handling:** Sends back the API response to the requester, confirming the creation of the shortened URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block waits for incoming HTTP POST requests at a specific webhook endpoint and triggers the workflow with the provided data.

**Nodes Involved:**  
- Receive Link Webhook

**Node Details:**  

- **Receive Link Webhook**  
  - **Type:** Webhook  
  - **Technical Role:** Entry point for the workflow; listens for HTTP POST requests at the path `/shorten-link`.  
  - **Configuration Choices:**  
    - HTTP Method: POST  
    - Path: `shorten-link`  
    - Response Mode: Uses a response node to send back data after processing (Respond with Shortened URL node).  
  - **Expressions/Variables:** None explicitly used here; expects JSON or form payload from the client.  
  - **Input/Output Connections:** No input; outputs data to the HTTP Request node.  
  - **Edge Cases/Potential Failures:**  
    - Missing or malformed POST data could cause downstream failures.  
    - Unauthorized access if the webhook is public and not secured externally.  
  - **Version Specifics:** v2 webhook node used, supporting response modes.  

---

#### 1.2 API Request to BioURL.link

**Overview:**  
This block constructs and sends a detailed POST request to the BioURL.link API endpoint to generate a smart, shortened URL including geotargeting, devicetargeting, languagetargeting, and other metadata.

**Nodes Involved:**  
- HTTP Request

**Node Details:**  

- **HTTP Request**  
  - **Type:** HTTP Request node  
  - **Technical Role:** Sends a POST request to the BioURL.link API to create the smart URL.  
  - **Configuration Choices:**  
    - URL: `https://biourl.link/api/url/add`  
    - Method: POST  
    - Body Type: JSON  
    - Headers: Includes an `Authorization` header with a Bearer token placeholder (`YOURAPIKEY`) for authentication.  
    - Request Body: JSON containing parameters such as:  
      - `url`: Target URL (hardcoded to https://google.com for this sample)  
      - `status`: "private"  
      - `custom`: "google" (custom alias)  
      - `password`: "mypass" (password protecting the link)  
      - `expiry`: Link expiration datetime  
      - `type`: "splash" (redirect type)  
      - Metadata fields: title, description, image  
      - Pixels: Array of tracking pixel IDs  
      - Campaign and channel IDs  
      - Deeplink configurations for Apple and Google Play stores  
      - Geotarget: Array mapping countries to specific URLs  
      - Devicetarget: Array mapping devices to URLs  
      - Languagetarget: Array mapping languages to URLs  
      - Parameters: Additional query parameters for the link  
    - Options: Follows redirects automatically.  
  - **Expressions/Variables:** None dynamically used; all values are static in this example JSON. In production, these could be parameterized with incoming webhook data.  
  - **Input/Output Connections:** Receives data from the Receive Link Webhook node; outputs to Respond with Shortened URL node.  
  - **Edge Cases/Potential Failures:**  
    - Authentication errors if API key is invalid or missing.  
    - Network timeouts or API endpoint unavailability.  
    - Invalid or incomplete JSON body causing API errors.  
    - Rate limiting or API quota exceeded.  
  - **Version Specifics:** Uses version 4.2 of the HTTP Request node.  

---

#### 1.3 Response Handling

**Overview:**  
This block returns the response from the BioURL.link API back to the original requester, completing the workflow.

**Nodes Involved:**  
- Respond with Shortened URL

**Node Details:**  

- **Respond with Shortened URL**  
  - **Type:** Respond to Webhook node  
  - **Technical Role:** Sends the entire response data from the HTTP Request node back to the calling client.  
  - **Configuration Choices:**  
    - Respond with: All incoming items (full API response).  
    - No additional formatting or filtering applied.  
  - **Expressions/Variables:** Passes the output data from the HTTP Request node directly.  
  - **Input/Output Connections:** Input from HTTP Request node; output is the HTTP response to the webhook caller.  
  - **Edge Cases/Potential Failures:**  
    - If upstream node fails or returns error, this node will transmit that error response.  
    - Potentially large response payloads might cause delays or client timeouts.  
  - **Version Specifics:** Version 1.2 of the Respond to Webhook node.  

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                  | Input Node(s)             | Output Node(s)               | Sticky Note                                            |
|-----------------------------|---------------------------|---------------------------------|---------------------------|-----------------------------|--------------------------------------------------------|
| Receive Link Webhook         | Webhook                   | Accepts incoming POST requests  | None                      | HTTP Request                |                                                        |
| HTTP Request                | HTTP Request              | Sends URL creation request to API | Receive Link Webhook       | Respond with Shortened URL   | Requires Bearer token in header Authorization          |
| Respond with Shortened URL  | Respond to Webhook        | Sends API response back to client | HTTP Request              | None                        | Responds with all incoming items for complete feedback |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Receive Link Webhook Node**  
   - Node Type: Webhook  
   - Set HTTP Method: POST  
   - Set Path: `shorten-link`  
   - Set Response Mode: Use a response node (to respond after processing)  
   - Save the node.

2. **Create HTTP Request Node**  
   - Node Type: HTTP Request  
   - Set Request URL: `https://biourl.link/api/url/add`  
   - Method: POST  
   - Under "Headers", add:  
     - Name: `Authorization`  
     - Value: `Bearer YOURAPIKEY` (replace with your actual API key)  
   - Set Body Content Type: JSON  
   - Add JSON body with the following keys and example values (customize as needed):  
     ```json
     {
       "url": "https://google.com",
       "status": "private",
       "custom": "google",
       "password": "mypass",
       "expiry": "2020-11-11 12:00:00",
       "type": "splash",
       "metatitle": "Not Google",
       "metadescription": "Not Google description",
       "metaimage": "https://www.mozilla.org/media/protocol/img/logos/firefox/browser/og.4ad05d4125a5.png",
       "description": "For facebook",
       "pixels": [1, 2, 3, 4],
       "channel": 1,
       "campaign": 1,
       "deeplink": {
         "auto": true,
         "apple": "https://apps.apple.com/us/app/google/id284815942",
         "google": "https://play.google.com/store/apps/details?id=com.google.android.googlequicksearchbox&hl=en_CA&gl=US"
       },
       "geotarget": [
         {"location": "Canada", "link": "https://google.ca"},
         {"location": "United States", "link": "https://google.us"}
       ],
       "devicetarget": [
         {"device": "iPhone", "link": "https://google.com"},
         {"device": "Android", "link": "https://google.com"}
       ],
       "languagetarget": [
         {"language": "en", "link": "https://google.com"},
         {"language": "fr", "link": "https://google.ca"}
       ],
       "parameters": [
         {"name": "aff", "value": "3"},
         {"device": "gtm_source", "link": "api"}
       ]
     }
     ```  
   - Enable options to follow redirects automatically if needed.  
   - Connect output of Receive Link Webhook node to this HTTP Request node.

3. **Create Respond with Shortened URL Node**  
   - Node Type: Respond to Webhook  
   - Set to respond with all incoming items (full data from HTTP Request).  
   - Connect output of HTTP Request node to this node.

4. **Finalize Connections**  
   - Connect nodes in this order:  
     Receive Link Webhook → HTTP Request → Respond with Shortened URL

5. **Credentials Setup**  
   - Ensure you have an HTTP credential or API key for the BioURL.link API.  
   - Insert the correct Bearer token in the Authorization header in the HTTP Request node.

6. **Activation & Testing**  
   - Activate the workflow.  
   - Test by sending a POST request to your n8n webhook URL at `/shorten-link` with appropriate body parameters (or use the static example).  
   - Confirm the response contains the shortened URL data from BioURL.link.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------|
| Ensure your BioURL.link API key is kept secure and not exposed publicly in the workflow JSON.       | Security best practices                         |
| BioURL.link API documentation for URL creation with targeting: https://biourl.link/api-docs         | Useful for extending or customizing API calls |
| This workflow’s webhook path `shorten-link` can be customized to fit your naming conventions.       | n8n webhook configuration                       |
| Advanced targeting includes geo, device, and language parameters that can be dynamically set.       | Marketing and analytics use cases               |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.

---