Send SMS via ClickSend API Worldwide without a Phone number

https://n8nworkflows.xyz/workflows/send-sms-via-clicksend-api-worldwide-without-a-phone-number-3084


# Send SMS via ClickSend API Worldwide without a Phone number

### 1. Workflow Overview

This workflow enables sending SMS messages globally via the ClickSend API without requiring a physical phone number. It is designed for use cases such as notifications, alerts, or marketing campaigns where programmatic SMS delivery is needed.

The workflow is logically divided into three main blocks:

- **1.1 Input Trigger**: Initiates the workflow manually via a user action.
- **1.2 SMS Data Preparation**: Defines the SMS message content and recipient phone number.
- **1.3 SMS Sending via API**: Sends the SMS message using the ClickSend REST API with HTTP Basic Authentication.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block starts the workflow manually when the user clicks the "Test workflow" button in n8n. It serves as the entry point for the SMS sending process.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  

  - **Node Name:** When clicking ‘Test workflow’  
    - **Type:** Manual Trigger  
    - **Technical Role:** Initiates workflow execution on user command.  
    - **Configuration:** Default manual trigger with no parameters.  
    - **Expressions/Variables:** None.  
    - **Input Connections:** None (start node).  
    - **Output Connections:** Connects to "Set SMS data" node.  
    - **Version Requirements:** n8n version supporting manual trigger node (standard).  
    - **Potential Failures:** None expected; manual trigger is user-initiated.  
    - **Sub-workflow:** None.

#### 1.2 SMS Data Preparation

- **Overview:**  
  This block sets the SMS message content and the recipient's phone number, preparing the data payload for the API request.

- **Nodes Involved:**  
  - Set SMS data

- **Node Details:**  

  - **Node Name:** Set SMS data  
    - **Type:** Set  
    - **Technical Role:** Defines static variables for SMS content and recipient phone number.  
    - **Configuration:**  
      - Sets two string fields:  
        - `sms`: The message text, e.g., "Hi, this is my first message".  
        - `to`: The recipient's phone number including international prefix, e.g., "+39xxxxxxxx".  
    - **Expressions/Variables:** Static values assigned directly; no dynamic expressions.  
    - **Input Connections:** Receives input from "When clicking ‘Test workflow’".  
    - **Output Connections:** Passes data to "Send SMS".  
    - **Version Requirements:** Compatible with n8n Set node version 3.4 or higher.  
    - **Potential Failures:**  
      - Incorrect phone number format may cause API rejection.  
      - Empty or invalid message content may cause errors.  
    - **Sub-workflow:** None.

#### 1.3 SMS Sending via API

- **Overview:**  
  This block sends the SMS message using the ClickSend API via an HTTP POST request with Basic Authentication.

- **Nodes Involved:**  
  - Send SMS

- **Node Details:**  

  - **Node Name:** Send SMS  
    - **Type:** HTTP Request  
    - **Technical Role:** Sends a POST request to ClickSend’s SMS API endpoint to deliver the message.  
    - **Configuration:**  
      - URL: `https://rest.clicksend.com/v3/sms/send`  
      - Method: POST  
      - Authentication: HTTP Basic Auth using credentials stored in n8n (username and API key).  
      - Headers: Content-Type set to `application/json`.  
      - Body: JSON payload constructed dynamically using expressions:  
        ```json
        {
          "messages": [
            {
              "source": "php",
              "body": "{{ $json.sms }}",
              "to": "{{ $json.to }}"
            }
          ]
        }
        ```  
      - Sends JSON body and headers.  
    - **Expressions/Variables:** Uses `{{ $json.sms }}` and `{{ $json.to }}` to inject SMS content and recipient number from previous node.  
    - **Input Connections:** Receives data from "Set SMS data".  
    - **Output Connections:** None (end node).  
    - **Version Requirements:** HTTP Request node version 4.2 or higher recommended for full feature support.  
    - **Potential Failures:**  
      - Authentication errors if credentials are incorrect or expired.  
      - Network timeouts or connectivity issues.  
      - API errors due to invalid phone number format or message content.  
      - Rate limiting or quota exceeded on ClickSend account.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name               | Node Type       | Functional Role                  | Input Node(s)             | Output Node(s)       | Sticky Note                                                                                                         |
|-------------------------|-----------------|--------------------------------|---------------------------|----------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger  | Starts workflow manually         | —                         | Set SMS data          |                                                                                                                     |
| Set SMS data            | Set             | Defines SMS message and recipient | When clicking ‘Test workflow’ | Send SMS              | ## STEP 2  In the node "Set SMS data" add the text and recipient of the message including the international prefix (eg. +39) and the phone number without spaces |
| Send SMS                | HTTP Request    | Sends SMS via ClickSend API      | Set SMS data              | —                    | ## STEP 1  [Register here to ClickSend](https://clicksend.com/?u=586989) and obtain your API Key and 2 € of free credits  In the node "Send SMS" create a "Basic Auth" with the username you registered and the API Key provided as your password |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - No parameters needed. This node will start the workflow on manual execution.

2. **Create Set Node for SMS Data**  
   - Add a **Set** node named "Set SMS data".  
   - Configure two fields:  
     - `sms` (string): Set to your desired message text, e.g., "Hi, this is my first message".  
     - `to` (string): Set to the recipient's phone number including international prefix, e.g., "+39xxxxxxxx".  
   - Connect the output of "When clicking ‘Test workflow’" to this node.

3. **Create HTTP Request Node to Send SMS**  
   - Add an **HTTP Request** node named "Send SMS".  
   - Set the HTTP Method to **POST**.  
   - Set the URL to `https://rest.clicksend.com/v3/sms/send`.  
   - Under **Authentication**, select **HTTP Basic Auth**.  
   - Create or select credentials with:  
     - Username: Your ClickSend account username.  
     - Password: Your ClickSend API Key.  
   - Under **Headers**, add:  
     - `Content-Type`: `application/json` (note the space before application/json should be removed).  
   - Under **Body Parameters**, select **JSON** and enter the following JSON with expressions:  
     ```json
     {
       "messages": [
         {
           "source": "php",
           "body": "{{ $json.sms }}",
           "to": "{{ $json.to }}"
         }
       ]
     }
     ```  
   - Connect the output of "Set SMS data" to this node.

4. **Configure Credentials**  
   - In n8n, create HTTP Basic Auth credentials for ClickSend with your username and API key.  
   - Assign these credentials to the "Send SMS" node.

5. **Test the Workflow**  
   - Save the workflow.  
   - Click the "Test workflow" button to trigger the manual start.  
   - Verify that the SMS is sent and received on the specified phone number.

6. **Optional Customization**  
   - Modify the "Set SMS data" node to accept dynamic inputs from other nodes or external data sources for message content and recipient number.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                         |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Register at ClickSend to obtain API Key and receive 2 € free credits for testing.                             | https://clicksend.com/?u=586989                         |
| Ensure phone numbers include international prefix and no spaces for API compatibility.                       | SMS Data Preparation                                    |
| Use HTTP Basic Authentication with username and API Key in the "Send SMS" node for secure API access.        | ClickSend API Authentication                            |
| This workflow is ideal for automating SMS notifications, alerts, and marketing messages globally without hardware. | Workflow Purpose                                        |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the SMS sending workflow using ClickSend API in n8n.