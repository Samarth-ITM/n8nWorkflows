Track Software Vulnerability Patents with ScrapeGraphAI, Matrix, and Intercom

https://n8nworkflows.xyz/workflows/track-software-vulnerability-patents-with-scrapegraphai--matrix--and-intercom-11404


# Track Software Vulnerability Patents with ScrapeGraphAI, Matrix, and Intercom

### 1. Workflow Overview

This workflow automates the tracking of newly published software vulnerability patents using ScrapeGraphAI, Matrix, and Intercom. It is designed for security teams who want to monitor patent publications weekly and receive immediate alerts for highly relevant patents, while archiving less critical information for future reference.

The workflow is structured into three logical blocks:

- **1.1 Data Collection**: Weekly trigger initiates the process, constructs a search URL targeting recent software vulnerability patents on Google Patents, and ScrapeGraphAI extracts structured patent data.

- **1.2 Relevance Filter**: A code node analyzes each patent’s title and abstract for specific vulnerability-related keywords, tagging them as relevant or not. An IF node routes the flow based on this flag.

- **1.3 Alert & Archive**: Relevant patents generate alert messages sent via Matrix to an engineering room. Non-relevant patents are mapped and stored as contact records with custom attributes in Intercom for later research.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Collection

**Overview:**  
This block triggers the workflow weekly, dynamically constructs a Google Patents search URL filtered to the last 7 days for the term "software vulnerability," and uses ScrapeGraphAI to extract structured data fields from the search results.

**Nodes Involved:**  
- Weekly Patent Check (Schedule Trigger)  
- Construct Patent Query (Code)  
- Scrape Patent Data (ScrapeGraphAI)

**Node Details:**  

- **Weekly Patent Check**  
  - Type: Schedule Trigger  
  - Configuration: Fires every week (interval set to weeks; defaults to Monday)  
  - Inputs: None (start node)  
  - Outputs: Connects to Construct Patent Query  
  - Potential Failures: Misconfiguration of schedule, time zone issues  

- **Construct Patent Query**  
  - Type: Code (JavaScript)  
  - Role: Builds a URL for Google Patents searching "software vulnerability" for patents published in last 7 days  
  - Key Expression:  
    ```js
    const query = encodeURIComponent("software vulnerability");
    const url = `https://patents.google.com/?q=${query}&before=priority:today&after=priority:7d`;
    return [{ json: { url } }];
    ```  
  - Input: Trigger data from schedule  
  - Output: JSON with `url` field  
  - Failures: URL encoding errors, unexpected input formats  

- **Scrape Patent Data**  
  - Type: ScrapeGraphAI node  
  - Role: Visits the constructed URL and extracts structured patent fields  
  - Configuration:  
    - `userPrompt` specifies extraction schema: title, publication_number, abstract, filing_date, assignee, url  
    - `websiteUrl` dynamically set from Construct Patent Query output  
  - Inputs: URL from Construct Patent Query  
  - Outputs: Array of patent entries, each with specified fields  
  - Failures: API key missing or invalid, scraping errors, website changes affecting data extraction, network timeouts  

---

#### 2.2 Relevance Filter

**Overview:**  
This block analyzes each patent listing for keywords related to software vulnerabilities. It flags each patent as relevant or not, then routes the flow accordingly.

**Nodes Involved:**  
- Analyze Relevance (Code)  
- Relevant? (IF)

**Node Details:**  

- **Analyze Relevance**  
  - Type: Code (JavaScript)  
  - Role: Checks patent title and abstract for keywords: 'vulnerability', 'exploit', 'buffer overflow', 'cve', 'cybersecurity'  
  - Key Expression:
    ```js
    const keywords = [
      'vulnerability',
      'exploit',
      'buffer overflow',
      'cve',
      'cybersecurity'
    ];
    return $input.all().map(item => {
      const data = item.json;
      const text = `${data.title || ''} ${data.abstract || ''}`.toLowerCase();
      const relevant = keywords.some(k => text.includes(k.toLowerCase()));
      return { json: { ...data, relevant } };
    });
    ```
  - Input: Patent data array from Scrape Patent Data  
  - Output: Patent data augmented with boolean `relevant` field  
  - Failures: Expression errors, null or undefined fields, case sensitivity issues  

- **Relevant?**  
  - Type: IF node  
  - Role: Routes patents based on `relevant` boolean  
  - Condition: `relevant == true` on main output 0; else main output 1  
  - Inputs: Patents with `relevant` flag from Analyze Relevance  
  - Outputs:  
    - True branch: to Matrix Alert node  
    - False branch: to Map Fields for Storage node  
  - Failures: Misconfigured conditions, missing `relevant` property  

---

#### 2.3 Alert & Archive

**Overview:**  
Relevant patents trigger an alert sent to a Matrix room with formatted patent details. Non-relevant patents are mapped into Intercom contact fields and saved for future research.

**Nodes Involved:**  
- Matrix Alert (Matrix node)  
- Map Fields for Storage (Set node)  
- Save to Intercom (Intercom node)

**Node Details:**  

- **Matrix Alert**  
  - Type: Matrix node  
  - Role: Sends message to configured Matrix room  
  - Configuration:  
    - Operation: Send Message  
    - Message includes patent title, publication number, filing date, assignee, and link (assumed formatted in the node, details inferred)  
  - Input: Patents flagged relevant from IF node  
  - Output: None (terminal node)  
  - Failures: Authentication errors (Matrix bot), network issues, invalid room ID or bot permissions  

- **Map Fields for Storage**  
  - Type: Set node  
  - Role: Maps patent fields into Intercom contact format with custom attributes  
  - Configuration: Maps title, publication_number, abstract, filing_date, assignee, url into corresponding contact fields/custom attributes (exact mapping fields not shown but implied)  
  - Input: Patents flagged non-relevant  
  - Output: Data prepared for Intercom contact creation  
  - Failures: Missing fields, mapping errors  

- **Save to Intercom**  
  - Type: Intercom node  
  - Role: Creates contacts in Intercom workspace using mapped patent data  
  - Configuration:  
    - Resource: Contact  
    - Operation: Create contact record  
  - Input: Mapped patent data from Set node  
  - Output: None (terminal node)  
  - Failures: Authentication errors, API limits, malformed data, existing contact conflicts  

---

### 3. Summary Table

| Node Name              | Node Type                    | Functional Role               | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                                                              |
|------------------------|------------------------------|------------------------------|-------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview      | Sticky Note                  | Overview and setup instructions | None                    | None                            | Explains weekly flow: Schedule Trigger → ScrapeGraphAI → relevance analysis → Matrix alert or Intercom archive; setup steps included  |
| Section – Collection   | Sticky Note                  | Describes Data Collection block | None                    | None                            | Schedule Trigger weekly, build URL, scrape patent data extraction                                                                        |
| Section – Filter       | Sticky Note                  | Describes Relevance Filter block | None                    | None                            | Keywords used to flag patents; routing logic                                                                                             |
| Section – Output       | Sticky Note                  | Describes Alert & Archive block | None                    | None                            | Matrix alert for relevant patents, Intercom archive for others                                                                           |
| Weekly Patent Check    | Schedule Trigger             | Weekly trigger to start workflow | None                    | Construct Patent Query          |                                                                                                                                          |
| Construct Patent Query | Code                        | Builds Google Patents search URL | Weekly Patent Check      | Scrape Patent Data              |                                                                                                                                          |
| Scrape Patent Data     | ScrapeGraphAI               | Extracts structured patent data | Construct Patent Query   | Analyze Relevance               |                                                                                                                                          |
| Analyze Relevance      | Code                        | Flags patents relevant by keywords | Scrape Patent Data       | Relevant?                      |                                                                                                                                          |
| Relevant?              | IF                          | Routes based on relevance flag | Analyze Relevance        | Matrix Alert (true), Map Fields for Storage (false) |                                                                                                                                          |
| Matrix Alert           | Matrix                      | Sends alert message for relevant patents | Relevant? (true)         | None                          |                                                                                                                                          |
| Map Fields for Storage | Set                         | Maps patent data for Intercom contact | Relevant? (false)        | Save to Intercom                |                                                                                                                                          |
| Save to Intercom       | Intercom                    | Stores patent data as contact in Intercom | Map Fields for Storage   | None                          |                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Weekly Patent Check`  
   - Type: Schedule Trigger  
   - Set the trigger interval to every 1 week (default Monday)  
   - No input connections needed  

2. **Add Code Node to Construct Patent Query**  
   - Name: `Construct Patent Query`  
   - Type: Code (JavaScript)  
   - Connect input from `Weekly Patent Check` output  
   - Paste this code:
     ```js
     const query = encodeURIComponent("software vulnerability");
     const url = `https://patents.google.com/?q=${query}&before=priority:today&after=priority:7d`;
     return [{ json: { url } }];
     ```
   - Output: JSON with `url` field  

3. **Add ScrapeGraphAI Node**  
   - Name: `Scrape Patent Data`  
   - Type: ScrapeGraphAI  
   - Connect input from `Construct Patent Query` output  
   - Configure with your ScrapeGraphAI credentials (add API key in n8n credentials)  
   - Set `websiteUrl` parameter to expression: `{{$json.url}}`  
   - Set `userPrompt` to:
     ```
     Extract the latest patent listings. Use this schema: {"title":"string","publication_number":"string","abstract":"string","filing_date":"string","assignee":"string","url":"string"}
     ```
   - Output: Array of patent JSON objects  

4. **Add Code Node to Analyze Relevance**  
   - Name: `Analyze Relevance`  
   - Type: Code (JavaScript)  
   - Connect input from `Scrape Patent Data` output  
   - Paste this code:
     ```js
     const keywords = [
       'vulnerability',
       'exploit',
       'buffer overflow',
       'cve',
       'cybersecurity'
     ];
     return $input.all().map(item => {
       const data = item.json;
       const text = `${data.title || ''} ${data.abstract || ''}`.toLowerCase();
       const relevant = keywords.some(k => text.includes(k.toLowerCase()));
       return { json: { ...data, relevant } };
     });
     ```
   - Output: Patent objects with `relevant` boolean  

5. **Add IF Node to Route Based on Relevance**  
   - Name: `Relevant?`  
   - Type: IF  
   - Connect input from `Analyze Relevance` output  
   - Set condition: Boolean, value1 = `{{$json.relevant}}` equals `true`  
   - True output: For relevant patents  
   - False output: For non-relevant patents  

6. **Add Matrix Node for Alerts**  
   - Name: `Matrix Alert`  
   - Type: Matrix  
   - Connect input from `Relevant?` true output  
   - Configure with Matrix bot credentials (create bot account, get room ID, set credentials in n8n)  
   - Operation: Send Message  
   - Configure message text to include patent details: title, publication number, filing date, assignee, and link (format message accordingly)  

7. **Add Set Node to Map Fields for Intercom**  
   - Name: `Map Fields for Storage`  
   - Type: Set  
   - Connect input from `Relevant?` false output  
   - Map patent fields to Intercom contact fields/custom attributes. For example:  
     - `name` or custom field: patent title  
     - `email` or similar: (if not applicable, leave blank)  
     - Custom attributes: publication_number, abstract, filing_date, assignee, url  

8. **Add Intercom Node to Save Data**  
   - Name: `Save to Intercom`  
   - Type: Intercom  
   - Connect input from `Map Fields for Storage` output  
   - Configure with Intercom workspace credentials  
   - Resource: Contact  
   - Operation: Create contact record with mapped fields  

9. **Activate the Workflow**  
   - Test manually before enabling the schedule trigger  
   - Ensure all credentials (ScrapeGraphAI, Matrix, Intercom) are correctly configured and authorized  

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow runs unattended every Monday and can be customized by editing the keyword list or search query in the respective code nodes. | Workflow Overview sticky note                                                                     |
| Setup requires API keys and credentials for ScrapeGraphAI, Matrix bot account and room, and Intercom workspace integration.     | Workflow Overview sticky note                                                                     |
| Matrix alerts provide immediate notifications to engineering teams, while Intercom archives enable long-term research.          | Section – Output sticky note                                                                      |
| Google Patents URL syntax uses `before=priority:today` and `after=priority:7d` to filter patents published in the last 7 days. | Section – Collection sticky note                                                                 |

---

This documentation fully covers the workflow’s structure, configuration, and logic to allow efficient reproduction, modification, and error anticipation by users or automation agents.