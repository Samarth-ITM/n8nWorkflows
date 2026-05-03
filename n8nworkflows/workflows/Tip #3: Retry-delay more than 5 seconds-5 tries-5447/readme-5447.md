Tip #3: Retry/delay more than 5 seconds/5 tries

https://n8nworkflows.xyz/workflows/tip--3--retry-delay-more-than-5-seconds-5-tries-5447


# Tip #3: Retry/delay more than 5 seconds/5 tries

---

### 1. Workflow Overview

This workflow demonstrates a custom retry and delay mechanism in n8n that surpasses the platform’s default retry limits of 5 tries and 5 seconds between retries. Its main purpose is to reliably attempt an HTTP request multiple times (configured here as 6 tries) with a longer delay (30 seconds) between attempts, useful when calling APIs or services that require longer timeout intervals or more retry attempts than n8n's built-in retry system allows.

The workflow is logically divided into these blocks:

- **1.1 Manual Trigger & Initialization:** Starts the workflow manually and sets initial retry parameters.
- **1.2 HTTP Request Execution:** Performs the HTTP request with built-in retry disabled, relying on custom control.
- **1.3 Retry Logic and Delay:** Adjusts the retry counters, checks if max tries reached, delays if necessary, or stops with error.
- **1.4 Informational Sticky Note:** Provides reference and documentation embedded in the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger & Initialization

**Overview:**  
This block initiates the workflow manually and sets the initial parameters for the retry mechanism — specifically, the delay between retries and the maximum number of retry attempts.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Set Fields

**Node Details:**

- **When clicking ‘Execute workflow’**
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow on demand.  
  - Configuration: No parameters.  
  - Input: None (manual start)  
  - Output: Single output connected to Set Fields node.  
  - Edge Cases: None.  
  - Version: v1

- **Set Fields**
  - Type: Set  
  - Role: Defines the initial retry parameters:  
    - `delay_seconds` = 30 (numeric)  
    - `max_tries` = "6" (string) — initial max tries as string to be converted later.  
  - Input: From manual trigger  
  - Output: Passes data to HTTP Request node  
  - Edge Cases: Type mismatch if parameters are interpreted incorrectly downstream.  
  - Version: 3.4

---

#### 1.2 HTTP Request Execution

**Overview:**  
Executes the HTTP request to the specified URL. Built-in retry is disabled to allow custom retry logic to control the number of attempts and delay duration. The node is configured to continue on error, allowing the workflow to handle failed requests gracefully.

**Nodes Involved:**  
- HTTP Request

**Node Details:**

- **HTTP Request**
  - Type: HTTP Request  
  - Role: Calls external API endpoint `https://example.com/1`  
  - Configuration:  
    - `retryOnFail` is false — disables n8n's default retry mechanism.  
    - `maxTries` is 5, but since retryOnFail=false, this is not active.  
    - `waitBetweenTries` is 5000 ms (5 sec), but unused due to retryOnFail=false.  
    - `onError` set to `continueErrorOutput` — allows workflow to continue even if request fails.  
  - Input: Receives data from Set Fields or Wait node.  
  - Output:  
    - On success: no further downstream nodes (empty array).  
    - On error: passes output to the Edit Fields node via the second output.  
  - Edge Cases:  
    - Network failures, non-2xx HTTP responses.  
    - Timeout or invalid URL.  
    - Because onError is continue, failures must be handled downstream.  
  - Version: 4.2

---

#### 1.3 Retry Logic and Delay

**Overview:**  
This block reduces the retry counter, checks if the max number of tries has been exceeded, and either stops the workflow with an error or waits for the specified delay before retrying the HTTP request.

**Nodes Involved:**  
- Edit Fields  
- If  
- Stop and Error  
- Wait

**Node Details:**

- **Edit Fields**
  - Type: Set  
  - Role: Updates retry parameters after each HTTP request attempt:  
    - Copies `delay_seconds` as string (same value)  
    - Decrements `max_tries` by 1 (converted to number, decreased by 1)  
  - Input: From HTTP Request error output  
  - Output: Passes data to If node for evaluation  
  - Key Expressions:  
    - `delay_seconds` = `={{ $json.delay_seconds }}` (string passthrough)  
    - `max_tries` = `={{ $json.max_tries - 1 }}` (numeric decrement)  
  - Edge Cases:  
    - If `max_tries` is undefined or non-numeric, arithmetic will fail.  
  - Version: 3.4

- **If**
  - Type: If  
  - Role: Checks if max_tries has reached 0 or less to decide whether to stop or retry.  
  - Condition:  
    - `max_tries` <= 0 (strict number comparison)  
  - Input: From Edit Fields  
  - Output:  
    - True (max_tries ≤ 0): to Stop and Error  
    - False: to Wait node  
  - Edge Cases:  
    - If max_tries is not a number, condition may behave unexpectedly.  
  - Version: 2.2

- **Stop and Error**
  - Type: Stop and Error  
  - Role: Terminates the workflow with an error message after exhausting retries.  
  - Configuration:  
    - Error message dynamically includes original max tries from 'Set Fields' node via expression:  
      `Service unavailable after {{ $('Set Fields').item.json.max_tries }} tries`  
  - Input: From If (True branch)  
  - Output: Ends workflow  
  - Edge Cases: None  
  - Version: 1

- **Wait**
  - Type: Wait  
  - Role: Delays the workflow execution by the number of seconds specified in `delay_seconds` before retrying HTTP Request.  
  - Configuration:  
    - Delay amount set dynamically to `={{ $json.delay_seconds }}` seconds  
  - Input: From If (False branch)  
  - Output: To HTTP Request node to retry  
  - Edge Cases:  
    - If `delay_seconds` is zero or negative, wait time may be invalid or immediate.  
  - Version: 1.1

---

#### 1.4 Informational Sticky Note

**Overview:**  
Provides a textual reference inside the workflow UI with a summary and a link to a blog post explaining the retry/delay approach.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**
  - Type: Sticky Note  
  - Role: Documentation embedded in the workflow canvas  
  - Configuration:  
    - Content includes a title, brief description, and a clickable link:  
      `https://n8n-tips.blogspot.com/2025/06/tip-3-retrydelay-more-than-5-seconds5.html`  
    - Size: width 360px, height 180px  
  - Input/Output: None  
  - Edge Cases: N/A  
  - Version: 1

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                           | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                         |
|-----------------------------|---------------------|-----------------------------------------|-----------------------------|-------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Starts the workflow manually             | None                        | Set Fields              |                                                                                                   |
| Set Fields                  | Set                 | Initializes retry parameters              | When clicking ‘Execute workflow’ | HTTP Request            |                                                                                                   |
| HTTP Request                | HTTP Request        | Makes API call, disables built-in retry  | Set Fields, Wait            | Edit Fields (on error)   |                                                                                                   |
| Edit Fields                 | Set                 | Decrements retry count and copies delay  | HTTP Request (error output) | If                      |                                                                                                   |
| If                         | If                  | Checks if max retries exhausted           | Edit Fields                 | Stop and Error, Wait    |                                                                                                   |
| Stop and Error              | Stop and Error      | Stops workflow with error if retries exhausted | If (true branch)            | None                    |                                                                                                   |
| Wait                       | Wait                | Delays retry by configured seconds        | If (false branch)           | HTTP Request            |                                                                                                   |
| Sticky Note                | Sticky Note         | Embedded documentation and link           | None                        | None                    | ## Tip #3: Retry/delay more than 5 seconds/5 tries\nThis is the template with example of how you can implement retry/delay custom approach with more than 5 seconds/5 tries in n8n.\nMore details in my [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-3-retrydelay-more-than-5-seconds5.html) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name it `When clicking ‘Execute workflow’`.
   - Leave default parameters.
   - This node will start the workflow manually.

3. **Add a Set node:**
   - Name it `Set Fields`.
   - Connect the Manual Trigger node’s output to this node’s input.
   - Configure two fields to set initial retry parameters:  
     - `delay_seconds` (Number) = 30  
     - `max_tries` (String) = "6"

4. **Add an HTTP Request node:**
   - Name it `HTTP Request`.
   - Connect the `Set Fields` node output to this node input.
   - Set the URL to `https://example.com/1`.
   - Disable built-in retry:  
     - Set `Retry On Fail` to false.  
     - Set `Max Tries` to 5 (though disabled).  
     - Set `Wait Between Tries` to 5000 ms (5 sec, unused).  
   - Set `On Error` to `Continue Error Output` so that failures do not stop the workflow.

5. **Add a Set node:**
   - Name it `Edit Fields`.
   - Connect the error output (second output) of `HTTP Request` to this node.
   - Configure two fields to update retry parameters:  
     - `delay_seconds` (String) = `={{ $json.delay_seconds }}` (pass through)  
     - `max_tries` (Number) = `={{ $json.max_tries - 1 }}` (decrement retry count)

6. **Add an If node:**
   - Name it `If`.
   - Connect the output of `Edit Fields` to this node.
   - Configure condition:  
     - Check if `max_tries` (number) is less than or equal to 0.

7. **Add a Stop and Error node:**
   - Name it `Stop and Error`.
   - Connect the True output of `If` (max tries exhausted) to this node.
   - Configure error message:  
     - `Service unavailable after {{ $('Set Fields').item.json.max_tries }} tries`

8. **Add a Wait node:**
   - Name it `Wait`.
   - Connect the False output of `If` (retry allowed) to this node.
   - Configure delay amount:  
     - `={{ $json.delay_seconds }}` seconds (dynamic from JSON field)

9. **Connect the output of `Wait` back to the main input of `HTTP Request`** to retry the request.

10. **Add a Sticky Note node for documentation (optional):**
    - Place it somewhere visible.
    - Enter content similar to:  
      ```
      ## Tip #3: Retry/delay more than 5 seconds/5 tries
      This is the template with example of how you can implement retry/delay custom approach with more than 5 seconds/5 tries in n8n.
      More details in my [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-3-retrydelay-more-than-5-seconds5.html)
      ```
    - Adjust size to around 360x180 pixels.

11. **Activate the workflow and execute manually to test.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow demonstrates how to implement custom retry logic with delays longer than n8n’s default maximums (5 retries and 5 seconds).   | Workflow concept                                                                                        |
| More detailed explanation and additional tips available at the author’s blog: [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-3-retrydelay-more-than-5-seconds5.html) | External documentation                                                                                  |
| The HTTP Request node uses `continueErrorOutput` to allow retry logic to be handled manually instead of using n8n’s built-in retry system. | Important integration detail                                                                            |
| When decrementing `max_tries`, data type conversions between string and number must be handled carefully to avoid expression errors.       | Critical for robustness                                                                                  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---