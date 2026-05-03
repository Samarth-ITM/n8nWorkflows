Get DNS entries of any domain with uProc

https://n8nworkflows.xyz/workflows/get-dns-entries-of-any-domain-with-uproc-862


# Get DNS entries of any domain with uProc

### 1. Workflow Overview

This workflow is designed to retrieve DNS records for any specified domain using the uProc platform's "Get Domain DNS records" tool. It targets users who need to monitor or validate DNS setups in real-time for domains they manage, such as customer domains or server hostnames. The workflow is structured into three logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Domain Preparation:** Creation and preparation of the domain item to be queried.
- **1.3 DNS Records Retrieval:** Execution of the uProc node to fetch DNS records for the given domain.

This structure allows flexibility to replace the domain input node with other sources like Google Sheets or databases, and the output can be integrated into various downstream applications.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This initial block allows manual initiation of the workflow. It waits for a user to trigger the process to start DNS record retrieval.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Technical Role:** Starts workflow execution on demand via manual user action.  
  - **Configuration:** Default manual trigger with no special parameters.  
  - **Key Expressions:** None.  
  - **Input:** None (start node).  
  - **Output:** Triggers "Create Domain Item" node.  
  - **Version Requirements:** Compatible with all n8n versions supporting manual triggers (v0.100+).  
  - **Potential Failures:** None typical; workflow simply does not start if not triggered.  
  - **Sub-workflows:** None.

#### 2.2 Domain Preparation

- **Overview:**  
  This block creates the domain data item to be used for querying DNS records. It currently hardcodes the domain as "n8n.io" but is designed for replacement with dynamic domain sources.

- **Nodes Involved:**  
  - Create Domain Item

- **Node Details:**  
  - **Name:** Create Domain Item  
  - **Type:** Function Item  
  - **Technical Role:** Sets up the domain string in the item JSON data for downstream consumption.  
  - **Configuration:** The JavaScript function sets `item.domain = "n8n.io"` and returns the item.  
  - **Key Expressions:** None beyond the function code itself.  
  - **Input:** Receives trigger from "On clicking 'execute'".  
  - **Output:** Passes item with domain property to "Get DNS records" node.  
  - **Version Requirements:** Function node syntax stable since early n8n versions.  
  - **Potential Failures:** Misconfigured JavaScript code could cause errors; currently static and simple so minimal risk.  
  - **Sub-workflows:** None.

#### 2.3 DNS Records Retrieval

- **Overview:**  
  This block performs the DNS records query by calling the uProc API with the specified domain. It retrieves multiple DNS record types and their respective values.

- **Nodes Involved:**  
  - Get DNS records

- **Node Details:**  
  - **Name:** Get DNS records  
  - **Type:** uProc Integration Node  
  - **Technical Role:** Connects to uProc's "Get Domain DNS records" tool to fetch DNS entries.  
  - **Configuration:**  
    - Tool: `getDomainRecords`  
    - Group: `internet`  
    - Domain: Expression referencing `{{$node["Create Domain Item"].json["domain"]}}` dynamically.  
    - Additional Options: None (default).  
  - **Key Expressions:** Domain fetched dynamically from the previous node output.  
  - **Input:** Receives domain item from "Create Domain Item".  
  - **Output:** Outputs multiple items with DNS record fields:  
    - `type`: DNS record type (e.g., A, CNAME, MX, etc.)  
    - `values`: Corresponding DNS record values array.  
  - **Credentials:** Requires valid uProc API credentials with email and API key configured under `uprocApi`.  
  - **Version Requirements:** Requires n8n version supporting uProc node (post-integration addition, typically v0.196+).  
  - **Potential Failures:**  
    - Authentication failures if API keys are missing or invalid.  
    - Network timeouts or API rate limits.  
    - Domain input empty or invalid leading to empty or error responses.  
  - **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                 | Input Node(s)        | Output Node(s)     | Sticky Note                                                                                                 |
|----------------------|---------------------|--------------------------------|----------------------|--------------------|-------------------------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger      | Starts workflow on user demand | None                 | Create Domain Item  |                                                                                                             |
| Create Domain Item    | Function Item       | Defines the domain to query     | On clicking 'execute' | Get DNS records     | You can replace "Create Domain Item" with any integration containing a domain, like Google Sheets, MySQL, or Zabbix server. |
| Get DNS records       | uProc Integration   | Retrieves DNS records from uProc| Create Domain Item    | None               | You need to add your credentials (Email and API Key - real -) located at [Integration section](https://app.uproc.io/#/settings/integration) to n8n. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**, name it "Get DNS entries".

2. **Add a Manual Trigger node**:  
   - Name: `On clicking 'execute'`  
   - No special configuration needed. This node starts the workflow manually.

3. **Add a Function Item node**:  
   - Name: `Create Domain Item`  
   - Set the function code to:  
     ```javascript
     item.domain = "n8n.io";
     return item;
     ```  
   - This node creates a JSON property `domain` with the domain value.  
   - Connect the output of `On clicking 'execute'` to the input of this node.

4. **Add a uProc node for DNS record retrieval**:  
   - Name: `Get DNS records`  
   - Under Parameters:  
     - Tool: Select `getDomainRecords`  
     - Group: `internet`  
     - Domain: Set as an expression referencing the domain from the previous node:  
       ```
       {{$node["Create Domain Item"].json["domain"]}}
       ```  
     - Leave Additional Options empty.  
   - Connect the output of `Create Domain Item` to the input of this node.

5. **Add uProc API Credentials** in n8n:  
   - Go to Credentials, create new credentials of type `uProc API`.  
   - Enter your **Email** and **API Key** from your uProc Integration settings page: https://app.uproc.io/#/settings/integration  
   - Assign these credentials to the `Get DNS records` node under Credentials > uprocApi.

6. **Activate or execute the workflow manually** by clicking "Execute".  
   - The workflow will emit multiple items containing DNS record types and their values.

7. **(Optional) Replace the domain source**:  
   - You may substitute the `Create Domain Item` function node with other data sources such as Google Sheets, MySQL, or Zabbix nodes that provide domain names dynamically.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| You need to add your credentials (Email and API Key - real -) located at Integration section to n8n.                 | https://app.uproc.io/#/settings/integration                                                        |
| This workflow uses the uProc "Get Domain DNS records" tool to check DNS records in real-time.                         | https://app.uproc.io/#/tools/processor/get/domain/records                                          |
| You can replace "Create Domain Item" node with any integration containing a domain, like Google Sheets, MySQL, or Zabbix server. | Workflow flexibility for domain input                                                              |

---

This document fully describes the "Get DNS entries" workflow, enabling users and AI agents to understand, reproduce, and extend it with confidence.