Perform Multi-type DNS Lookups with Google's Free Public DNS Service

https://n8nworkflows.xyz/workflows/perform-multi-type-dns-lookups-with-google-s-free-public-dns-service-10200


# Perform Multi-type DNS Lookups with Google's Free Public DNS Service

### 1. Workflow Overview

This workflow performs multi-type DNS lookups using Google's free public DNS over HTTPS API (https://dns.google/resolve). It is designed to accept a domain name and optionally a set of DNS record types to query (e.g., A, AAAA, MX). If the DNS types are not provided, it defaults to a standard set of common record types. The workflow then queries each DNS type individually, aggregates the results, and outputs a human-readable consolidated list of DNS records for the domain.

The logic is grouped into the following blocks:

- **1.1 Input Reception and Validation**: Receives domain and optional DNS types from a form input, determines whether to use all or selected DNS types.
- **1.2 DNS Type Splitting and Query Execution**: Splits the DNS types into individual queries, performs the DNS lookup via HTTP requests to Google's DNS API.
- **1.3 Data Formatting and Aggregation**: Adds human-readable DNS type labels to the results, aggregates all individual DNS responses into a single structured output.
- **1.4 Output Delivery**: Outputs the aggregated DNS lookup results in a structured JSON format ready for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block receives user input (domain and DNS types) via a form trigger, checks if DNS types were provided, and sets default DNS types if none are specified.

**Nodes Involved:**  
- Form input  
- If no DNS type in input  
- Default to all DNS types  
- Use selected DNS types  

**Node Details:**  

- **Form input**  
  - *Type:* Form Trigger  
  - *Role:* Entry point, collects domain and optional DNS types from user input form.  
  - *Configuration:*  
    - Form fields:  
      - Domain (required text)  
      - Types (checkbox, optional, multiple values: A, CNAME, AAAA, MX, TXT, NS)  
    - Response mode set to last node (returns final output).  
  - *Inputs:* Webhook trigger on form submission.  
  - *Outputs:* JSON with `Domain` string and optionally `Types (leave empty to use all)` array.  
  - *Edge Cases:* Empty or invalid domain input will fail downstream; no validation beyond required field.  
  - *Notes:* Supports dynamic DNS type selection; empty array triggers default set.

- **If no DNS type in input**  
  - *Type:* If  
  - *Role:* Checks if the DNS types input is empty (array empty).  
  - *Configuration:* Condition tests if `Types (leave empty to use all)` is empty.  
  - *Inputs:* Connected from Form input.  
  - *Outputs:*  
    - True: No DNS types specified.  
    - False: DNS types specified.  
  - *Edge Cases:* Strict validation to detect empty array; fails if input is malformed or missing key.  

- **Default to all DNS types**  
  - *Type:* Set  
  - *Role:* Sets default DNS types array if none specified.  
  - *Configuration:* Sets JSON fields:  
    - Domain: copied from input  
    - Types: ["A", "CNAME", "AAAA", "MX", "TXT", "NS"]  
  - *Inputs:* True branch of If node.  
  - *Outputs:* JSON with domain and default DNS types.  
  - *Edge Cases:* Hardcoded default list must be updated manually if additional types are needed.

- **Use selected DNS types**  
  - *Type:* Set  
  - *Role:* Passes along user-selected DNS types if provided.  
  - *Configuration:* Sets JSON fields:  
    - Domain: copied from input  
    - Types: copied from `Types (leave empty to use all)` field  
  - *Inputs:* False branch of If node.  
  - *Outputs:* JSON with domain and user-selected DNS types.  
  - *Edge Cases:* Assumes valid DNS types; no validation beyond presence.

---

#### 2.2 DNS Type Splitting and Query Execution

**Overview:**  
This block splits the array of DNS types into individual items, performs a DNS lookup for each type via Google's DNS-over-HTTPS API, and attaches the human-readable DNS type to each result.

**Nodes Involved:**  
- Split Out  
- DNS Lookup  
- Set human readable type in output  
- For each DNS type  

**Node Details:**  

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the `Types` array into individual items for processing.  
  - *Configuration:* Splits on field `Types`, includes all other fields in each split item.  
  - *Inputs:* Output of Default to all DNS types or Use selected DNS types.  
  - *Outputs:* One item per DNS type with domain and that type.  
  - *Edge Cases:* If `Types` is empty or missing, no output items generated.

- **DNS Lookup**  
  - *Type:* HTTP Request  
  - *Role:* Performs the DNS query for a single DNS type using Google's DNS API.  
  - *Configuration:*  
    - URL: `https://dns.google/resolve`  
    - Query parameters:  
      - `name`: domain from JSON (`{{$json.Domain}}`)  
      - `type`: current DNS type from JSON (`{{$json.Types}}`)  
    - Method: GET  
    - Query parameters sent as URL params.  
  - *Inputs:* From For each DNS type node (see below).  
  - *Outputs:* JSON response from the DNS API with fields like `Answer`, `Question`, etc.  
  - *Edge Cases:*  
    - Network or API errors (timeouts, rate limits).  
    - DNS type unsupported by API returns empty answers.  
    - Malformed domain or invalid DNS type may cause errors or empty response.

- **Set human readable type in output**  
  - *Type:* Code (JavaScript)  
  - *Role:* Adds a `type_hr` field to the JSON output matching the current DNS type for clarity.  
  - *Configuration:* Loops over all input items, sets `type_hr` to the value from `Types` of the corresponding "For each DNS type" node.  
  - *Inputs:* From DNS Lookup node.  
  - *Outputs:* Enriched JSON with `type_hr`.  
  - *Edge Cases:* May fail if input structure changes or expected fields missing.

- **For each DNS type**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each DNS type item, feeding them sequentially to DNS Lookup and aggregation.  
  - *Configuration:* Default batching with no explicit batch size limit.  
  - *Inputs:* Output from Set human readable type in output (loop back) and Split Out.  
  - *Outputs:* Feeds DNS Lookup and Aggregate results nodes in parallel paths.  
  - *Edge Cases:* Large numbers of DNS types may increase runtime or cause rate limiting.

---

#### 2.3 Data Formatting and Aggregation

**Overview:**  
Aggregates all individual DNS query results into a single structured JSON object listing the domain and all DNS records grouped by type, handling cases where no records are returned.

**Nodes Involved:**  
- Aggregate results  

**Node Details:**  

- **Aggregate results**  
  - *Type:* Code (JavaScript)  
  - *Role:* Iterates over all DNS query results, extracts domain and DNS answers, and consolidates them into an array of records with type and data fields.  
  - *Configuration:*  
    - Extracts domain from the first item's Question array, removing trailing dots.  
    - Loops over each input item:  
      - Uses `type_hr` for human-readable DNS type or "UNKNOWN" fallback.  
      - Pushes each answer record's data under its type.  
      - If no answers, adds a record with data "NO ANSWER".  
    - Returns a single output item with domain and records array.  
  - *Inputs:* From For each DNS type node.  
  - *Outputs:* Single aggregated JSON output.  
  - *Edge Cases:*  
    - Missing or malformed Question or Answer arrays may cause errors or incomplete data.  
    - Ensures output even if no DNS answers found.

---

#### 2.4 Output Delivery

**Overview:**  
The final aggregated DNS lookup results are output from the Aggregate results node, ready to be used in downstream automation or display.

**Nodes Involved:**  
- (No explicit output node; the last node is Aggregate results which outputs the final data)  

**Node Details:**  

- Output is a JSON object with:  
  - `domain`: The queried domain name (string)  
  - `records`: Array of objects with fields:  
    - `type`: DNS record type (e.g., "A", "MX")  
    - `data`: Record data or "NO ANSWER" if none found  

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                             | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                               |
|----------------------------|----------------------|---------------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| Form input                 | Form Trigger         | Receives domain and DNS types input         | (Webhook trigger)             | If no DNS type in input      | ## Loops over each type and executes a DNS lookup                                                      |
| If no DNS type in input    | If                   | Checks if DNS types input is empty           | Form input                   | Default to all DNS types; Use selected DNS types | ## Loops over each type and executes a DNS lookup                                                      |
| Default to all DNS types   | Set                  | Sets default DNS types if none provided      | If no DNS type in input       | Split Out                   | ## Loops over each type and executes a DNS lookup                                                      |
| Use selected DNS types     | Set                  | Passes user-selected DNS types               | If no DNS type in input       | Split Out                   | ## Loops over each type and executes a DNS lookup                                                      |
| Split Out                 | Split Out            | Splits DNS types array into individual items | Default to all DNS types; Use selected DNS types | For each DNS type           | ## Loops over each type and executes a DNS lookup                                                      |
| For each DNS type          | Split In Batches     | Iterates over each DNS type and controls flow | Split Out; Set human readable type in output | Aggregate results; DNS Lookup | ## Loops over each type and executes a DNS lookup                                                      |
| DNS Lookup                 | HTTP Request        | Performs DNS lookup via Google DNS API       | For each DNS type             | Set human readable type in output | ## Loops over each type and executes a DNS lookup                                                      |
| Set human readable type in output | Code (JavaScript) | Adds human-readable DNS type label            | DNS Lookup                   | For each DNS type            | ## Loops over each type and executes a DNS lookup                                                      |
| Aggregate results          | Code (JavaScript)    | Aggregates all DNS lookup results             | For each DNS type             | (Final output)              | ## Outputs an aggregated list of the DNS lookup results                                               |
| Sticky Note1               | Sticky Note          | Comment on output aggregation                  |                              |                             | ## Outputs an aggregated list of the DNS lookup results                                               |
| Sticky Note2               | Sticky Note          | Comment on looping over each DNS type          |                              |                             | ## Loops over each type and executes a DNS lookup                                                      |
| Sticky Note4               | Sticky Note          | General info, instructions, and modification notes |                              |                             | ## Information Template created by Smultron Studio (https://smultronstudio.com/en) - feel free to reach out at hello@smultronstudio.com ... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `Form input`  
   - Configure webhook with a form titled "DNS lookup".  
   - Form fields:  
     - Text input `Domain` (required, placeholder: "domain.tld")  
     - Checkbox `Types (leave empty to use all)` with options: A, CNAME, AAAA, MX, TXT, NS (optional)  
   - Set response mode to "last node".

2. **Add an If Node**  
   - Name: `If no DNS type in input`  
   - Condition: Check if `Types (leave empty to use all)` field is an empty array.  
   - True branch: Proceed with default DNS types.  
   - False branch: Use selected DNS types from input.

3. **Add a Set Node for Default DNS Types**  
   - Name: `Default to all DNS types`  
   - Set JSON with:  
     ```json
     {
       "Domain": "{{ $json.Domain }}",
       "Types": ["A","CNAME","AAAA","MX","TXT","NS"]
     }
     ```
   - Connect True output of If node here.

4. **Add a Set Node for Selected DNS Types**  
   - Name: `Use selected DNS types`  
   - Set JSON with:  
     ```json
     {
       "Domain": "{{ $json.Domain }}",
       "Types": {{ $json["Types (leave empty to use all)"] }}
     }
     ```
   - Connect False output of If node here.

5. **Add a Split Out Node**  
   - Name: `Split Out`  
   - Configure to split on field `Types`, including all other fields.  
   - Connect both Set nodes (`Default to all DNS types` and `Use selected DNS types`) to this node.

6. **Add a Split In Batches Node**  
   - Name: `For each DNS type`  
   - Use default batch size (or adjust as needed).  
   - Connect output of `Split Out` here.

7. **Add an HTTP Request Node**  
   - Name: `DNS Lookup`  
   - Configure request:  
     - Method: GET  
     - URL: `https://dns.google/resolve`  
     - Query parameters:  
       - `name`: `{{$json.Domain}}`  
       - `type`: `{{$json.Types}}`  
   - Connect output of `For each DNS type` here.

8. **Add a Code Node to Set Human-readable Type**  
   - Name: `Set human readable type in output`  
   - Code:  
     ```javascript
     for (const item of $input.all()) {
       item.json.type_hr = $('For each DNS type').first().json.Types;
     }
     return $input.all();
     ```
   - Connect output of `DNS Lookup` here.

9. **Connect output of Code node back to `For each DNS type`**  
   - This creates a loop ensuring each DNS type is processed and enriched with human-readable type.

10. **Add a Code Node to Aggregate Results**  
    - Name: `Aggregate results`  
    - Code:  
      ```javascript
      let domain = null;
      const records = [];

      for (const item of $input.all()) {
        const itemJson = item.json;
        if (domain === null && itemJson.Question && itemJson.Question.length > 0) {
          domain = itemJson.Question[0].name.replace(/\.$/, '');
        }
        const recordType = itemJson.type_hr || 'UNKNOWN';
        const answerRecords = itemJson.Answer;

        if (answerRecords && answerRecords.length > 0) {
          for (const record of answerRecords) {
            records.push({
              type: recordType,
              data: record.data
            });
          }
        } else {
          records.push({
            type: recordType,
            data: "NO ANSWER"
          });
        }
      }

      const outputItem = {
        json: {
          domain: domain,
          records: records
        }
      };

      return [outputItem];
      ```
    - Connect a main output from `For each DNS type` node (parallel branch) to this node.

11. **Activate the workflow and test via form input with a domain and optional DNS types.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| Template created by Smultron Studio (https://smultronstudio.com/en). For support reach out at hello@smultronstudio.com.                                                                                                       | General info and template credits                                                                |
| To add additional DNS types to the default "ALL" list, modify the "Default to all DNS types" Set node and the Form input options accordingly.                                                                               | Instructions on modifying DNS types                                                              |
| Complete list of DNS record type IDs is available here: https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml                                                                                                | Reference for DNS types                                                                           |
| Sticky Note: "## Loops over each type and executes a DNS lookup" applies to nodes from Form input through For each DNS type, DNS Lookup, and Set human readable type in output node.                                          | Clarification on workflow looping mechanism                                                     |
| Sticky Note: "## Outputs an aggregated list of the DNS lookup results" applies to the Aggregate results node.                                                                                                                | Clarifies final result output                                                                    |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.