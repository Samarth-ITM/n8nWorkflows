Automated URL Phishing & Threat Analysis with NixGuard AI

https://n8nworkflows.xyz/workflows/automated-url-phishing---threat-analysis-with-nixguard-ai-5937


# Automated URL Phishing & Threat Analysis with NixGuard AI

### 1. Workflow Overview

This n8n workflow named **"Automated URL Phishing & Threat Analysis with NixGuard AI"** acts as a **dispatcher** designed to initiate a comprehensive security analysis workflow focused on URL and IP threat intelligence. It integrates with the NixGuard AI platform and Wazuh security monitoring to provide real-time threat insights. The workflow’s architecture enables easy reuse and modularity by triggering a separate, more complex workflow that performs the heavy analysis.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Initialization:** Accepts inputs such as API key and initial prompt (e.g., URL to scan), preparing the parameters for the main analysis.
- **1.2 Main Analysis Workflow Invocation:** Calls the external, detailed NixGuard & Wazuh workflow using the prepared inputs.
- **1.3 Result Formatting:** Processes and formats the output from the main workflow for downstream consumption.
- **1.4 Optional Alerting:** Provides an optional Slack alert node that can be enabled to notify security teams about high-risk events.
- **1.5 Documentation & Guidance:** Sticky notes that document setup instructions, workflow overview, and recommended next steps for automation and integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

- **Overview:**  
  This block sets the foundational inputs, including the NixGuard API key and the initial prompt (a URL to scan). It also contains a webhook node ready to receive real-time inputs in production environments.

- **Nodes Involved:**  
  - `Set API Key & Initial Prompt`  
  - `Webhook Trigger (REAL-WORLD USE)1`

- **Node Details:**

  - **Set API Key & Initial Prompt**  
    - Type: Set  
    - Role: Initializes workflow variables, specifically the API key and the URL prompt for scanning.  
    - Configuration:  
      - `apiKey` set as a placeholder string `"PASTE_YOUR_NIXGUARD_API_KEY_HERE"` to be replaced by the user.  
      - `chatInput` preset with a sample URL `"Scan this url for me: https://thenex.world"`.  
    - Inputs: None (starting node)  
    - Outputs: Connects to `Execute NixGuard & Wazuh Workflow` node.  
    - Edge Cases: Failure if API key is missing or invalid; static prompt limits dynamic input unless webhook is used.  
    - Version: Compatible with n8n version supporting `Set` node v2.

  - **Webhook Trigger (REAL-WORLD USE)1**  
    - Type: Webhook  
    - Role: Provides an entry point to trigger the workflow from external HTTP requests (currently inactive).  
    - Configuration:  
      - Path set to unique webhook ID `"e74aeb1a-0659-4a89-8ede-17bb9fdbe317"`.  
      - Not active by default, implying manual activation is needed for production.  
    - Inputs: External HTTP calls  
    - Outputs: None connected in this workflow (standalone).  
    - Edge Cases: Timeout or malformed webhook requests; needs activation before use.

---

#### 2.2 Main Analysis Workflow Invocation

- **Overview:**  
  This block calls the core analysis workflow that performs the detailed threat detection combining NixGuard AI and Wazuh inputs.

- **Nodes Involved:**  
  - `Execute NixGuard & Wazuh Workflow`

- **Node Details:**

  - **Execute NixGuard & Wazuh Workflow**  
    - Type: Execute Workflow  
    - Role: Invokes the main security analysis workflow by passing current inputs and credentials.  
    - Configuration:  
      - References workflow with ID `"I0nUORqYTwDFZa51"` titled "Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration".  
      - Passes all current input data downstream without schema restrictions or type conversions.  
    - Input: Connects from `Set API Key & Initial Prompt` node.  
    - Output: Connects to `Format NixGuard AI Summary & Wazuh Insights`.  
    - Edge Cases: Failure if referenced workflow is missing or credentials invalid; error propagation if invoked workflow errors out.  
    - Version: Requires n8n supporting Execute Workflow node v1.2.

---

#### 2.3 Result Formatting

- **Overview:**  
  This block formats the output from the main analysis workflow to a usable string summary for alerting or further automation.

- **Nodes Involved:**  
  - `Format NixGuard AI Summary & Wazuh Insights`

- **Node Details:**

  - **Format NixGuard AI Summary & Wazuh Insights**  
    - Type: Set  
    - Role: Extracts the output from the invoked workflow and assigns it to a new field `ai_summary` for uniform downstream usage.  
    - Configuration:  
      - Sets `ai_summary` to the entire `output` field from the JSON response of the executed workflow. Expression: `={{ $json.output }}`  
    - Input: Connects from `Execute NixGuard & Wazuh Workflow`.  
    - Output: Connects to `(Optional) Send Slack Alert for High-Risk Events`.  
    - Edge Cases: Expression failure if `output` is missing or malformatted; null or empty results yield blank summaries.  
    - Version: Set node v2 compatible.

---

#### 2.4 Optional Alerting

- **Overview:**  
  This disabled node is an optional Slack alert mechanism to notify the security team about detected high-risk events using the formatted summary.

- **Nodes Involved:**  
  - `(Optional) Send Slack Alert for High-Risk Events`

- **Node Details:**

  - **(Optional) Send Slack Alert for High-Risk Events**  
    - Type: Slack  
    - Role: Sends a customizable alert message to a configured Slack channel with the AI-generated summary.  
    - Configuration:  
      - OAuth2 authentication required.  
      - Message text includes emoji alert and references the `ai_summary` field dynamically:  
        `"=🚨 *NixGuard IP Analysis* 🚨\n\n*AI Summary:*\n{{ $json.ai_summary }}"`  
      - Node is disabled by default, ready for user enablement.  
    - Input: Receives from `Format NixGuard AI Summary & Wazuh Insights`.  
    - Output: None (terminal).  
    - Edge Cases: Authentication errors, Slack API rate limits, disabled node (no effect).  
    - Version: Slack node v2.

---

#### 2.5 Documentation & Guidance

- **Overview:**  
  This block contains sticky notes providing meta-information about the workflow, critical setup steps, and recommended automation next steps.

- **Nodes Involved:**  
  - `Next Steps: Automate Response2`  
  - `Workflow Overview2`  
  - `Setup Instructions2`

- **Node Details:**

  - **Next Steps: Automate Response2**  
    - Type: Sticky Note  
    - Content Highlights:  
      - Guidance on enabling Slack alerts, creating Jira tickets, logging results, and triggering firewall remediation workflows.  
      - Encourages integration of this dispatcher with broader SOAR playbooks.  
    - Position: Adjacent to main logic flow for visibility.

  - **Workflow Overview2**  
    - Type: Sticky Note  
    - Content Highlights:  
      - Explains the dispatcher pattern for modular analysis workflow triggering.  
      - Emphasizes reusability and separation of concerns between triggering logic and core analysis.  
      - Provides link to learn more about NixGuard: https://thenex.world  
      - Lists relevant tags for context.

  - **Setup Instructions2**  
    - Type: Sticky Note  
    - Content Highlights:  
      - Stepwise instructions for replacing the API key placeholder.  
      - Guidance for connecting the main analysis workflow.  
      - Link to obtain API keys and source main workflow.  
      - Clear callout that this workflow alone is insufficient without the main analysis workflow.  
    - Provides URLs for subscribing and downloading the main workflow.

---

### 3. Summary Table

| Node Name                          | Node Type            | Functional Role                             | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                                                            |
|-----------------------------------|----------------------|--------------------------------------------|----------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Set API Key & Initial Prompt       | Set                  | Initializes API key and URL prompt          | None                             | Execute NixGuard & Wazuh Workflow  | Setup Instructions2 node explains replacing API key and connecting main workflow.                                                     |
| Execute NixGuard & Wazuh Workflow  | Execute Workflow     | Invokes main NixGuard & Wazuh analysis      | Set API Key & Initial Prompt     | Format NixGuard AI Summary & Wazuh Insights | Workflow Overview2 explains dispatcher pattern and reusability.                                                                       |
| Format NixGuard AI Summary & Wazuh Insights | Set          | Formats AI & Wazuh output into summary      | Execute NixGuard & Wazuh Workflow | (Optional) Send Slack Alert for High-Risk Events | Next Steps: Automate Response2 outlines automation extensions post-formatting.                                                        |
| (Optional) Send Slack Alert for High-Risk Events | Slack          | Sends alert to Slack with AI summary         | Format NixGuard AI Summary & Wazuh Insights | None                               | Next Steps: Automate Response2 suggests enabling alerts and further integrations.                                                     |
| Webhook Trigger (REAL-WORLD USE)1  | Webhook              | External trigger (disabled by default)       | None                             | None                               | Workflow Overview2 notes webhook usage for real-time triggering.                                                                        |
| Next Steps: Automate Response2     | Sticky Note          | Provides guidance on further automation      | None                             | None                               | Contains detailed recommendations for SOC/IR automation including Slack, Jira, logging, and remediation.                               |
| Workflow Overview2                 | Sticky Note          | Describes dispatcher workflow purpose        | None                             | None                               | Explains design pattern and includes learning resource link https://thenex.world                                                        |
| Setup Instructions2               | Sticky Note          | Critical setup instructions                   | None                             | None                               | Detailed two-step setup and links for API key registration and main workflow import.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Set` node named `Set API Key & Initial Prompt`:**  
   - Add two string fields:  
     - `apiKey`: Set value to `"PASTE_YOUR_NIXGUARD_API_KEY_HERE"` (replace with your actual key).  
     - `chatInput`: Set value to `"Scan this url for me: https://thenex.world"` (or customize).  
   - No input connections.

3. **Add an `Execute Workflow` node named `Execute NixGuard & Wazuh Workflow`:**  
   - Set the workflow to invoke as `"Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration"` (workflow ID `"I0nUORqYTwDFZa51"`).  
   - Configure input mapping mode as pass-through all fields.  
   - Connect the output of `Set API Key & Initial Prompt` to this node’s input.

4. **Add a `Set` node named `Format NixGuard AI Summary & Wazuh Insights`:**  
   - Add a string field named `ai_summary`.  
   - Set its value to the expression: `{{$json["output"]}}` to capture the output from the invoked workflow.  
   - Connect the output of `Execute NixGuard & Wazuh Workflow` to this node.

5. **(Optional) Add a `Slack` node named `(Optional) Send Slack Alert for High-Risk Events`:**  
   - Authenticate using OAuth2 with your Slack workspace credentials.  
   - Set the message text to:  
     ```
     🚨 *NixGuard IP Analysis* 🚨

     *AI Summary:*
     {{ $json.ai_summary }}
     ```  
   - Connect the output of `Format NixGuard AI Summary & Wazuh Insights` to this node.  
   - Disable this node by default; enable when ready to send alerts.

6. **(Optional) Add a `Webhook` node named `Webhook Trigger (REAL-WORLD USE)1`:**  
   - Set a unique webhook path (e.g., `e74aeb1a-0659-4a89-8ede-17bb9fdbe317`).  
   - Leave inactive by default; enable to accept real-time external HTTP triggers.  
   - (No connections needed unless you want to connect webhook output to `Set API Key & Initial Prompt` or other nodes for dynamic input.)

7. **Add Sticky Notes for Documentation:**  
   - Add three sticky notes with contents matching:  
     - **Setup Instructions2:** Explaining API key replacement and main workflow connection with URLs.  
     - **Workflow Overview2:** Explaining dispatcher pattern and key project tags.  
     - **Next Steps: Automate Response2:** Suggestions for extending automation with Slack, Jira, logging, and firewall remediation.

8. **Set Credentials:**  
   - Configure credentials for Slack OAuth2.  
   - Ensure you have valid NixGuard API key ready.  
   - The main analysis workflow referenced must be imported or created separately.

9. **Test the Workflow:**  
   - Replace the API key placeholder with a valid key.  
   - Run the workflow manually or trigger via webhook (if enabled).  
   - Confirm the main workflow executes and outputs appear in `ai_summary`.  
   - Optionally enable Slack node and verify alerts are sent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow acts as a dispatcher pattern enabling separation of triggering logic from core NixGuard & Wazuh analysis logic for modularity and reusability.                    | Sticky Note `Workflow Overview2` and general project design.                                    |
| To get a free NixGuard API key and subscribe, visit: https://thenex.world/security/subscribe                                                                                   | Setup Instructions2 sticky note.                                                                |
| The main referenced workflow "Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration" can be found/imported from: https://n8n.io/workflows/4693-get-real-time-security-insights-with-nixguard-rag-and-wazuh-integration/ | Setup Instructions2 sticky note.                                                                |
| Recommended next steps include enabling Slack alerts, automating Jira ticket creation, logging results for audit, and triggering firewall remediation workflows.                | Sticky Note `Next Steps: Automate Response2`.                                                   |
| The Slack node requires OAuth2 authentication credentials configured in n8n prior to usage.                                                                                    | Slack node configuration details.                                                               |
| The webhook node is disabled by default and requires manual activation to allow real-time triggering from external sources.                                                  | Webhook Trigger node details.                                                                    |

---

**Disclaimer:** The provided text stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All handled data are lawful and public.