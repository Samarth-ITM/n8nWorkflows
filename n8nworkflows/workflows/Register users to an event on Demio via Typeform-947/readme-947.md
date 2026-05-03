Register users to an event on Demio via Typeform

https://n8nworkflows.xyz/workflows/register-users-to-an-event-on-demio-via-typeform-947


# Register users to an event on Demio via Typeform

### 1. Workflow Overview

This workflow automates the registration of users for an event on Demio based on responses submitted through a Typeform form. It is designed for use cases where an organization wants to streamline event sign-ups by capturing user details via Typeform and directly registering them to a Demio webinar or event. The workflow consists of two main logical blocks:

- **1.1 Input Reception:** Capturing form submissions from Typeform.
- **1.2 Event Registration:** Using the captured data to register the user on Demio.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new submissions from a specified Typeform form and triggers the workflow when a user completes the form.

- **Nodes Involved:**  
  - Typeform Trigger

- **Node Details:**  

  - **Typeform Trigger**  
    - Type: Trigger node (Typeform integration)  
    - Configuration:  
      - Listens to a specific Typeform form identified by `formId` (empty in the provided JSON, to be set by the user).  
      - Uses credentials named "Typeform Burner Account" for API access.  
      - Version: 1  
    - Key expressions/variables:  
      - The node outputs the full JSON response from Typeform, including answers keyed by question text, e.g., `{{$json["What's your email address?"]}}` and `{{$json["Let's start with your name."]}}`.  
    - Input/Output:  
      - Input: None (trigger node).  
      - Output: JSON object containing form response data.  
    - Edge cases / failures:  
      - Invalid or missing `formId` will prevent triggers.  
      - Authentication failure if credentials are invalid or expired.  
      - Network or webhook setup errors may cause missed triggers.  
    - Sub-workflow: None.

#### 1.2 Event Registration

- **Overview:**  
  This block registers the user to a Demio event using the data captured from Typeform.

- **Nodes Involved:**  
  - Demio

- **Node Details:**  

  - **Demio**  
    - Type: Action node (Demio integration)  
    - Configuration:  
      - Operation: `register` (to register a user for an event).  
      - Event ID: `357191` (hardcoded event identifier).  
      - Email: Extracted dynamically from Typeform response via expression `{{$json["What's your email address?"]}}`.  
      - First Name: Extracted dynamically via expression `{{$json["Let's start with your name."]}}`.  
      - Additional fields: None configured.  
      - Uses credentials named "Demio API Credentials" for API access.  
      - Version: 1  
    - Key expressions/variables:  
      - Email and First Name via dynamic expressions pulling from Typeform JSON.  
    - Input/Output:  
      - Input: JSON data from Typeform Trigger node (user response).  
      - Output: Result of registration attempt (success or error details).  
    - Edge cases / failures:  
      - Invalid or missing email or first name fields could cause registration failure.  
      - API authentication errors if Demio credentials are invalid or expired.  
      - Event ID mismatch or event not found errors.  
      - Network or API downtime causing timeouts or failures.  
      - Rate limiting or quota exceeded on Demio API.  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name         | Node Type               | Functional Role          | Input Node(s)    | Output Node(s) | Sticky Note                                                                                      |
|-------------------|-------------------------|-------------------------|------------------|----------------|------------------------------------------------------------------------------------------------|
| Typeform Trigger  | Trigger (Typeform)      | Capture form submissions | None             | Demio          | This node will trigger the workflow when a form response is submitted. Replace it if needed.    |
| Demio             | Action (Demio API)      | Register user to event   | Typeform Trigger | None           | This node registers a user for an event using details from the Typeform response.                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Typeform Trigger node:**  
   - Add a new node of type **Typeform Trigger**.  
   - Set the **Form ID** to the ID of your Typeform form used for event registration.  
   - Assign credentials for Typeform API access (e.g., "Typeform Burner Account").  
   - Leave the node at version 1 (default).  
   - This node will start the workflow whenever a user submits the form.

2. **Create the Demio node:**  
   - Add a new node of type **Demio**.  
   - Set the **Operation** to `register`.  
   - Enter the **Event ID** corresponding to your Demio event (e.g., `357191`).  
   - For **Email**, use expression mode and enter: `{{$json["What's your email address?"]}}` to dynamically map the email from Typeform response.  
   - For **First Name**, use expression mode and enter: `{{$json["Let's start with your name."]}}` to map the user's name.  
   - Assign credentials for Demio API access (e.g., "Demio API Credentials").  
   - Keep additional fields empty unless needed.  
   - Use version 1 (default).

3. **Connect the nodes:**  
   - Connect the output of **Typeform Trigger** node to the input of **Demio** node.

4. **Save and activate the workflow:**  
   - Ensure both credentials are correctly set up and have required API access.  
   - Test the workflow by submitting a test response in the Typeform form and verify registration on Demio.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                |
|----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Replace the Typeform Trigger node with an equivalent node if using another form platform (e.g., Google Forms, Jotform).           | Workflow overview and flexibility instructions |
| Ensure Demio event ID matches the intended event to avoid registration errors.                                                   | Demio API documentation                        |
| Verify API credentials periodically to prevent authentication failures.                                                         | n8n Credentials management                      |
| This workflow assumes the Typeform questions are named exactly as “What's your email address?” and “Let's start with your name.” | Mapping form fields to node expressions         |