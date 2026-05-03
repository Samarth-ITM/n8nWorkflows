n8n Enterprise AI Security Firewall — Guardrails for Secure Agents

https://n8nworkflows.xyz/workflows/n8n-enterprise-ai-security-firewall---guardrails-for-secure-agents-11025


# n8n Enterprise AI Security Firewall — Guardrails for Secure Agents

### 1. Workflow Overview

This workflow, titled **n8n Enterprise AI Security Firewall — Guardrails for Secure Agents**, is designed to enforce multiple layers of security and content moderation for AI agents by applying a series of guardrails. It targets scenarios where AI models interact with potentially sensitive, inappropriate, or malicious content, ensuring compliance and safe outputs.

The workflow logically breaks down into these main blocks:

- **1.1 Input and Trigger Reception:** Starts the workflow manually and fetches test data to be analyzed.
- **1.2 Content Classification and Guardrail Enforcement:** Uses a switch node to route input through multiple guardrails (e.g., Jailbreak detection, PII detection, Secret Keys, NSFW content, URL validation, Topical Alignment, and Keyword checks).
- **1.3 Sanitization Blocks:** After classification, separate nodes sanitize PII, secret keys, and URLs to obfuscate sensitive information.
- **1.4 AI Model Integration:** The Google Gemini Chat Model node integrates with the AI language model, applying all guardrails in real-time to monitor or filter content.
- **1.5 Data Preparation and Setting:** Various 'Set' nodes prepare sanitized data for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input and Trigger Reception

**Overview:**  
This block triggers the workflow manually and retrieves data from a Google Sheets node named "Guadrails Test Data" to be processed.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Guadrails Test Data (Google Sheets)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual trigger  
  - Configuration: No parameters, simply triggers workflow execution on demand  
  - Input: None (trigger node)  
  - Output: Connects to Guadrails Test Data  
  - Edge cases: None specific; manual trigger may require user interaction.

- **Guadrails Test Data**  
  - Type: Google Sheets node  
  - Configuration: Reads test data from a specific sheet (details not specified)  
  - Input: From Manual Trigger  
  - Output: Connects to Switch node  
  - Edge cases: Possible API errors, authentication issues, or empty data sets.

---

#### 2.2 Content Classification and Guardrail Enforcement

**Overview:**  
This core security block routes input data through a Switch node, which directs it to multiple guardrail nodes designed to detect and prevent various content and security risks.

**Nodes Involved:**  
- Switch  
- Jailbreak  
- PII  
- Secret Keys  
- NSFW  
- URLs  
- Topical Alignment  
- Keywords

**Node Details:**  

- **Switch**  
  - Type: Switch node  
  - Configuration: Routes based on conditions (not explicitly detailed in JSON) to different guardrail nodes  
  - Input: From Guadrails Test Data  
  - Output: Routes to each guardrail node as separate outputs  
  - Edge cases: Misconfiguration could cause incorrect routing or dropped data.

- **Jailbreak**  
  - Type: LangChain Guardrails node  
  - Configuration: Detects attempts to bypass AI restrictions (jailbreak content)  
  - Input: From Switch output 1  
  - Output: No further connections shown  
  - Edge cases: False positives or negatives possible depending on detection sensitivity.

- **PII**  
  - Type: LangChain Guardrails node  
  - Configuration: Detects Personally Identifiable Information in content  
  - Input: From Switch output 2  
  - Output: No further connections shown  
  - Edge cases: May miss obfuscated PII or produce false positives.

- **Secret Keys**  
  - Type: LangChain Guardrails node  
  - Configuration: Detects secret or private keys in content  
  - Input: From Switch output 3  
  - Output: No further connections shown  
  - Edge cases: May require regular updates to key patterns.

- **NSFW**  
  - Type: LangChain Guardrails node  
  - Configuration: Detects Not Safe For Work content  
  - Input: From Switch output 4  
  - Output: No further connections shown  
  - Edge cases: Cultural context may affect detection accuracy.

- **URLs**  
  - Type: LangChain Guardrails node  
  - Configuration: Detects and validates URLs in content  
  - Input: From Switch output 5  
  - Output: Connects to "Sanitize URLs" node  
  - Edge cases: Malformed URLs or encoded URLs might evade detection.

- **Topical Alignment**  
  - Type: LangChain Guardrails node  
  - Configuration: Checks content relevance or topical alignment  
  - Input: From Switch output 6  
  - Output: No further connections shown  
  - Edge cases: Ambiguous content may produce unreliable alignment results.

- **Keywords**  
  - Type: LangChain Guardrails node  
  - Configuration: Keyword detection for monitoring or filtering  
  - Input: From Switch output 7  
  - Output: Shows two outputs, but none connected further  
  - Edge cases: Keyword lists may need frequent maintenance.

---

#### 2.3 Sanitization Blocks

**Overview:**  
Sanitize sensitive data flagged by guardrails, notably PII, secret keys, and URLs, to prevent leakage or misuse.

**Nodes Involved:**  
- Personal Data (Set)  
- Sanitize PII  
- API Keys (Set)  
- Sanitize Keys  
- URLS (Set)  
- Sanitize URLs

**Node Details:**  

- **Personal Data**  
  - Type: Set node  
  - Configuration: Prepares or stores PII data before sanitization  
  - Input: Not explicitly connected; likely used for context  
  - Output: Connects to Sanitize PII  

- **Sanitize PII**  
  - Type: LangChain Guardrails node  
  - Configuration: Obfuscates or removes PII data from content  
  - Input: From Personal Data  
  - Output: No further connections shown  
  - Edge cases: Over-sanitization could remove useful data.

- **API Keys**  
  - Type: Set node  
  - Configuration: Prepares secret keys before sanitization  
  - Input: Not explicitly connected; likely used for context  
  - Output: Connects to Sanitize Keys  

- **Sanitize Keys**  
  - Type: LangChain Guardrails node  
  - Configuration: Obfuscates or removes secret keys  
  - Input: From API Keys node  
  - Output: No further connections shown  
  - Edge cases: New key patterns might require updates.

- **URLS**  
  - Type: Set node  
  - Configuration: Prepares URLs before sanitization  
  - Input: Not explicitly connected; likely used for context  
  - Output: Connects to Sanitize URLs  

- **Sanitize URLs**  
  - Type: LangChain Guardrails node  
  - Configuration: Obfuscates or validates URLs to prevent abuse  
  - Input: From URLS node and also receives output from URLs guardrail  
  - Output: No further connections shown  
  - Edge cases: Complex URLs might not sanitize correctly.

---

#### 2.4 AI Model Integration

**Overview:**  
This block integrates the Google Gemini Chat Model with all guardrails attached as ai_languageModel inputs, enforcing security and content restrictions during AI processing.

**Nodes Involved:**  
- Google Gemini Chat Model  
- All Guardrail nodes (Jailbreak, PII, Secret Keys, NSFW, URLs, Topical Alignment, Keywords) connected as ai_languageModel inputs

**Node Details:**  

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat Model node  
  - Configuration: Executes AI chat with Google Gemini; all guardrails monitor or filter input/output  
  - Input: Receives ai_languageModel inputs from all guardrail nodes simultaneously  
  - Output: Not connected further, likely final processing step  
  - Edge cases: Requires valid Google credentials; API limits and latency may impact performance.

---

#### 2.5 Sticky Notes

**Overview:**  
Multiple sticky notes are spread throughout the workflow, presumably for documentation or annotation purposes. Most contain no content, except for the few possibly covering clusters of guardrail nodes or sections, but no textual content was provided.

**Nodes Involved:**  
All nodes of type Sticky Note (e.g., Sticky Note, Sticky Note1, Sticky Note2, etc.)

**Node Details:**  

- Type: Sticky Note nodes for annotation only  
- Configuration: Empty content in all provided nodes  
- No input/output connections  
- Used for visual grouping or comments (empty here)

---

### 3. Summary Table

| Node Name                | Node Type                           | Functional Role                         | Input Node(s)               | Output Node(s)               | Sticky Note                              |
|--------------------------|-----------------------------------|---------------------------------------|----------------------------|-----------------------------|-----------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                    | Starts the workflow manually          | -                          | Guadrails Test Data          |                                         |
| Guadrails Test Data       | Google Sheets                     | Fetches test data                     | When clicking ‘Execute workflow’ | Switch                      |                                         |
| Switch                   | Switch                           | Routes data to appropriate guardrails | Guadrails Test Data         | Jailbreak, PII, Secret Keys, NSFW, URLs, Topical Alignment, Keywords |                                         |
| Jailbreak                | LangChain Guardrails             | Detects jailbreak attempts            | Switch                     | -                           |                                         |
| PII                      | LangChain Guardrails             | Detects Personally Identifiable Information | Switch                     | -                           |                                         |
| Secret Keys              | LangChain Guardrails             | Detects secret keys                   | Switch                     | -                           |                                         |
| NSFW                     | LangChain Guardrails             | Detects NSFW content                  | Switch                     | -                           |                                         |
| URLs                     | LangChain Guardrails             | Detects and validates URLs            | Switch                     | Sanitize URLs               |                                         |
| Topical Alignment        | LangChain Guardrails             | Checks topical relevance              | Switch                     | -                           |                                         |
| Keywords                 | LangChain Guardrails             | Keyword detection                     | Switch                     | -                           |                                         |
| Personal Data            | Set                             | Prepares PII for sanitization         | -                          | Sanitize PII                |                                         |
| Sanitize PII             | LangChain Guardrails             | Sanitizes PII                        | Personal Data              | -                           |                                         |
| API Keys                 | Set                             | Prepares secret keys for sanitization | -                          | Sanitize Keys               |                                         |
| Sanitize Keys            | LangChain Guardrails             | Sanitizes secret keys                | API Keys                   | -                           |                                         |
| URLS                     | Set                             | Prepares URLs for sanitization        | -                          | Sanitize URLs               |                                         |
| Sanitize URLs            | LangChain Guardrails             | Sanitizes URLs                       | URLS, URLs guardrail output | -                           |                                         |
| Google Gemini Chat Model | LangChain Google Gemini Chat Model | Executes AI chat with guardrails monitoring | Guardrail nodes (Jailbreak, PII, Secret Keys, NSFW, URLs, Topical Alignment, Keywords) | -                           |                                         |
| Sticky Note              | Sticky Note                      | Annotation / documentation            | -                          | -                           | Empty notes found throughout workflow  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add Google Sheets Node**  
   - Name: `Guadrails Test Data`  
   - Configure to connect to Google Sheets with correct credentials.  
   - Set to read the dataset containing test inputs for guardrail evaluation.

3. **Add Switch Node**  
   - Name: `Switch`  
   - Configure routing conditions to send incoming data to one of the guardrail nodes based on content type or flags.  
   - Conditions should correspond to: Jailbreak, PII, Secret Keys, NSFW, URLs, Topical Alignment, Keywords.

4. **Create Guardrail Nodes (LangChain Guardrails)**  
   - Add nodes for each guardrail:  
     - `Jailbreak`  
     - `PII`  
     - `Secret Keys`  
     - `NSFW`  
     - `URLs`  
     - `Topical Alignment`  
     - `Keywords`  
   - Configure each with relevant guardrail rules or policies (specifics depend on LangChain Guardrails implementation).

5. **Connect Switch Outputs**  
   - Connect each Switch output to the corresponding guardrail node.

6. **Setup Sanitization Nodes (Set + Guardrails)**  
   - Create `Personal Data` (Set node), set fields for PII content.  
   - Connect `Personal Data` to `Sanitize PII` (Guardrails node).  
   - Create `API Keys` (Set node), set fields for secret key content.  
   - Connect `API Keys` to `Sanitize Keys` (Guardrails node).  
   - Create `URLS` (Set node), set fields for URLs.  
   - Connect `URLS` to `Sanitize URLs` (Guardrails node).  
   - Connect output of `URLs` guardrail node to `Sanitize URLs` as well.

7. **Add Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Configure with valid Google AI credentials.  
   - Attach all guardrail nodes as `ai_languageModel` inputs to this node, so it applies their checks in real-time.

8. **Link Workflow Flow**  
   - From `When clicking ‘Execute workflow’` → `Guadrails Test Data` → `Switch`  
   - Switch outputs → respective guardrail nodes  
   - Guardrail nodes → where applicable to sanitization nodes  
   - Sanitization nodes prepared by Set nodes as above  
   - Guardrail nodes connected as ai_languageModel inputs → `Google Gemini Chat Model`

9. **Add Sticky Notes** (Optional)  
   - Add empty or descriptive sticky notes near logical blocks for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow leverages the LangChain Guardrails nodes for AI content moderation and security.    | LangChain Guardrails documentation and n8n integration details are required for customization.     |
| Google Gemini Chat Model integration requires valid Google Cloud AI credentials and API access.   | https://cloud.google.com/vertex-ai/docs/generative-ai/chat-models                                   |
| Guardrail conditions in the Switch node must be tailored to the expected input data schema.       | Proper testing with real inputs is recommended to minimize false positives and routing errors.      |
| The workflow is designed for enterprise AI deployments to enforce security and compliance guardrails. | Suitable for regulated environments requiring rigorous data privacy and content safety measures.    |

---

**Disclaimer:**  
This document is based exclusively on an n8n automated workflow exported as JSON. It respects all current content policies and contains no illegal or offensive elements. All data processed are legal and public.