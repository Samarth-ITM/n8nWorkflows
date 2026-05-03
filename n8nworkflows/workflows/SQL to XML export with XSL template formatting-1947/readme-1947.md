SQL to XML export with XSL template formatting

https://n8nworkflows.xyz/workflows/sql-to-xml-export-with-xsl-template-formatting-1947


# SQL to XML export with XSL template formatting

### 1. Workflow Overview

This workflow demonstrates exporting data from a SQL database as XML formatted output, enhanced with an XSL template for presentation. It is designed to serve XML data over HTTP with proper styling, while respecting browser CORS policies.

The workflow is divided into two main logical blocks:

- **1.1 Data Export and XML Generation**  
  Starts with a webhook that triggers a SQL query to fetch random product records, transforms the data into XML, and wraps it with an XML declaration linking to an XSL stylesheet hosted on the same domain.

- **1.2 XSL Template Delivery for CORS Compliance**  
  A helper webhook fetches the XSL template from a GitHub Gist and serves it via n8n’s own domain to comply with CORS rules, ensuring the XML data and stylesheet share the same origin.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Export and XML Generation

- **Overview:**  
  This block fetches 16 random product records from a MySQL database, transforms the output into a customized XML structure, and adds an XML header linking to the stylesheet served by the workflow. It finally sends the XML response with correct MIME type and CORS headers.

- **Nodes Involved:**  
  - Webhook (Trigger)  
  - Show 16 random products (MySQL)  
  - Define file structure (Set)  
  - Concatenate Items (Item Lists)  
  - Convert to XML (XML)  
  - Create HTML (HTML)  
  - Move Binary Data (Move Binary Data)  
  - Respond to Webhook (Respond to Webhook)

- **Node Details:**

  1. **Webhook**  
     - *Type & Role:* HTTP trigger node, entry point for XML export requests.  
     - *Config:* Path set to a unique webhook ID, response mode is "responseNode" to let a downstream node handle response.  
     - *Inputs/Outputs:* No inputs, output to "Show 16 random products".  
     - *Failure points:* Webhook URL accessibility, request method compatibility.

  2. **Show 16 random products (MySQL)**  
     - *Type & Role:* Executes SQL query on MySQL database.  
     - *Config:* Query selects 16 random rows from `products` table. Uses stored MySQL credentials.  
     - *Expressions:* Static query, no dynamic inputs.  
     - *Inputs/Outputs:* Input from Webhook, output to "Define file structure".  
     - *Failure points:* Database connection/auth errors, SQL query syntax errors, empty result sets.

  3. **Define file structure (Set)**  
     - *Type & Role:* Transforms SQL JSON data into a structured format with specific XML node names.  
     - *Config:* Uses expressions to map SQL fields to XML element names like `Product.Code`, `Product.Name`, etc.  
     - *Expressions:* `={{ $json.productCode }}`, etc.  
     - *Inputs/Outputs:* Input from MySQL node, output to "Concatenate Items".  
     - *Failure points:* Missing fields in SQL results, expression evaluation errors.

  4. **Concatenate Items (Item Lists)**  
     - *Type & Role:* Aggregates all item data into a single JSON object under a `Products` key for XML conversion.  
     - *Config:* Operation is concatenate all items into one field named "Products".  
     - *Inputs/Outputs:* Input from Set node, output to "Convert to XML".  
     - *Failure points:* Handling of empty data sets.

  5. **Convert to XML (XML)**  
     - *Type & Role:* Converts JSON data into XML string.  
     - *Config:* Mode is JSON to XML, with "headless" option enabled (no XML declaration added here).  
     - *Inputs/Outputs:* Input from Concatenate Items, output to "Create HTML".  
     - *Failure points:* JSON structure compatibility, conversion errors.

  6. **Create HTML (HTML)**  
     - *Type & Role:* Constructs final XML with XML declaration and XSL stylesheet reference injected.  
     - *Config:*  
       - XML declaration with encoding ISO-8859-1.  
       - XML stylesheet processing instruction: href dynamically constructed using environment variable `{{$env.WEBHOOK_URL}}` pointing to the XSL template webhook path.  
       - Embeds the XML data from previous node via `{{ $json.data }}`.  
     - *Inputs/Outputs:* Input from Convert to XML, output to "Move Binary Data".  
     - *Failure points:* Incorrect environment variable, malformed XML or broken XSL link.

  7. **Move Binary Data (Move Binary Data)**  
     - *Type & Role:* Converts the XML string to binary data for proper HTTP response handling.  
     - *Config:* Converts JSON field `html` to binary with MIME type `text/xml`, discards source JSON.  
     - *Inputs/Outputs:* Input from Create HTML, output to "Respond to Webhook".  
     - *Failure points:* Conversion failures, MIME type issues.

  8. **Respond to Webhook (Respond to Webhook)**  
     - *Type & Role:* Sends back the XML binary response to the HTTP requestor.  
     - *Config:*  
       - HTTP status code 200.  
       - Response headers include `Content-Type: text/xml` and `Control-Allow-Origin: *` for CORS.  
       - Responds with binary data.  
     - *Inputs/Outputs:* Input from Move Binary Data, no outputs (end).  
     - *Failure points:* Network errors, header misconfiguration.

- **Sticky Notes:**  
  - The Set node ("Define file structure") note explains XML declaration and XSL link addition with environment variable usage.  
  - The Webhook node note reminds that the XML node has the headless toggle enabled.  
  - The binary data and response nodes note emphasizes the conversion and response sending process.

---

#### 2.2 XSL Template Delivery for CORS Compliance

- **Overview:**  
  This helper block handles requests for the XSL template file, fetching it from a GitHub Gist and serving it via n8n’s webhook on the same domain. This avoids CORS issues when browsers load the stylesheet linked in the XML.

- **Nodes Involved:**  
  - Request xsl template (Webhook)  
  - Get XSLT (HTTP Request)  
  - Respond to Webhook1 (Respond to Webhook)  
  - Sticky Note (explaining CORS necessity)

- **Node Details:**

  1. **Request xsl template (Webhook)**  
     - *Type & Role:* HTTP trigger node that listens on the path `/models.xsl`.  
     - *Config:* Raw body disabled, response mode "responseNode" for downstream response node.  
     - *Inputs/Outputs:* No inputs, output to "Get XSLT".  
     - *Failure points:* Webhook URL availability, path conflicts.

  2. **Get XSLT (HTTP Request)**  
     - *Type & Role:* Fetches the XSL template file from a public GitHub Gist URL.  
     - *Config:* Response format is set to file to handle raw file data.  
     - *Inputs/Outputs:* Input from webhook, output to "Respond to Webhook1".  
     - *Failure points:* GitHub Gist availability, network timeouts.

  3. **Respond to Webhook1 (Respond to Webhook)**  
     - *Type & Role:* Sends the fetched XSL file back to the requester.  
     - *Config:*  
       - Response code 200.  
       - Content-Type header set to `text/xsl`.  
       - Responds with binary data.  
     - *Inputs/Outputs:* Input from HTTP Request node, no outputs (end).  
     - *Failure points:* Header misconfiguration, binary data issues.

- **Sticky Note:**  
  Explains that this webhook is necessary for CORS compliance, serving the XSL template from n8n’s domain rather than directly from GitHub.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                         | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                                   |
|-------------------------|---------------------|---------------------------------------|----------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook                 | Webhook             | Entry trigger for XML data export     | —                          | Show 16 random products          | XML node has headless toggle enabled                                                                         |
| Show 16 random products | MySQL               | Fetch 16 random product records       | Webhook                    | Define file structure            |                                                                                                              |
| Define file structure    | Set                 | Map SQL fields to XML node structure  | Show 16 random products    | Concatenate Items               | XML declaration and XSL link added here; uses {{$env.WEBHOOK_URL}} to auto-select n8n instance URL            |
| Concatenate Items       | Item Lists          | Aggregate all products into one field | Define file structure       | Convert to XML                  |                                                                                                              |
| Convert to XML           | XML                 | Convert JSON to XML string             | Concatenate Items          | Create HTML                    |                                                                                                              |
| Create HTML              | HTML                | Add XML declaration and stylesheet link | Convert to XML             | Move Binary Data               |                                                                                                              |
| Move Binary Data         | Move Binary Data    | Convert XML string to binary for response | Create HTML               | Respond to Webhook             | Convert to binary data and send webhook response                                                             |
| Respond to Webhook       | Respond to Webhook  | Return XML response over HTTP          | Move Binary Data           | —                               |                                                                                                              |
| Request xsl template     | Webhook             | Trigger for XSL template delivery     | —                          | Get XSLT                       | This webhook is needed to comply with CORS policy; fetches XSL template from GitHub Gist and serves via n8n   |
| Get XSLT                 | HTTP Request        | Fetch XSL template from GitHub Gist   | Request xsl template       | Respond to Webhook1             |                                                                                                              |
| Respond to Webhook1      | Respond to Webhook  | Respond with XSL template file         | Get XSLT                   | —                               |                                                                                                              |
| Sticky Note              | Sticky Note         | CORS policy explanation                | —                          | —                               | This webhook is needed to comply with the CORS policy of modern browsers. It requests XSL template stored...  |
| Sticky Note1             | Sticky Note         | Note on SQL data to XML conversion     | —                          | —                               | Get data from SQL and convert it to XML. Note XML node has headless toggle                                    |
| Sticky Note2             | Sticky Note         | Note on XML declaration and XSL link   | —                          | —                               | Set node can be used as well. XML declaration and a link to the XSL template are added here                   |
| Sticky Note3             | Sticky Note         | Note on binary conversion and response | —                          | —                               | Convert to binary data and send the webhook response                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Data Export Trigger):**  
   - Type: Webhook  
   - Path: Unique identifier (e.g., `81115579-ca32-496f-baa7-f14fb3baec6f`)  
   - Response Mode: `responseNode`  
   - No credentials needed.

2. **Add MySQL Node (Fetch Products):**  
   - Type: MySQL  
   - Operation: `executeQuery`  
   - Query:  
     ```sql
     SELECT * FROM products
     ORDER BY RAND()
     LIMIT 16;
     ```  
   - Credentials: Configure with your MySQL credentials (e.g., db4free MySQL).  
   - Connect Webhook output to this node.

3. **Add Set Node (Define File Structure):**  
   - Type: Set  
   - Keep only set: Enabled  
   - Add string fields mapping SQL fields to XML element names:  
     - `Product.Code` = `={{ $json.productCode }}`  
     - `Product.Name` = `={{ $json.productName }}`  
     - `Product.Line` = `={{ $json.productLine }}`  
     - `Product.Scale` = `={{ $json.productScale }}`  
     - `Product.Price` = `={{ $json.MSRP }}`  
   - Connect MySQL output to this node.

4. **Add Item Lists Node (Concatenate Items):**  
   - Type: Item Lists  
   - Operation: `concatenateItems`  
   - Aggregate: `aggregateAllItemData`  
   - Destination field name: `Products`  
   - Connect Set output to this node.

5. **Add XML Node (Convert to XML):**  
   - Type: XML  
   - Mode: `jsonToxml`  
   - Options: Headless toggle enabled (no XML declaration)  
   - Connect Item Lists output to this node.

6. **Add HTML Node (Create XML with Declaration and Stylesheet Link):**  
   - Type: HTML  
   - HTML content:  
     ```xml
     <?xml version="1.0" encoding="ISO-8859-1"?>
     <?xml-stylesheet type="text/xsl" href="{{ $env.WEBHOOK_URL }}webhook/044ccd2d-d1a7-485b-bb22-218a6848fd1c/models.xsl"?>

     {{ $json.data }}
     ```  
   - Connect XML output to this node.

7. **Add Move Binary Data Node:**  
   - Type: Move Binary Data  
   - Mode: `jsonToBinary`  
   - Source Key: `html`  
   - MIME type: `text/xml`  
   - Keep source: Disabled  
   - Connect HTML output to this node.

8. **Add Respond to Webhook Node:**  
   - Type: Respond to Webhook  
   - Response code: 200  
   - Response headers:  
     - Content-Type: `text/xml`  
     - Control-Allow-Origin: `*`  
   - Respond with: `binary`  
   - Connect Move Binary Data output to this node.

---

9. **Create Webhook Node (XSL Template Request):**  
   - Type: Webhook  
   - Path: `044ccd2d-d1a7-485b-bb22-218a6848fd1c/models.xsl`  
   - Raw body: Disabled  
   - Response Mode: `responseNode`  
   - No credentials needed.

10. **Add HTTP Request Node (Get XSLT):**  
    - Type: HTTP Request  
    - Request URL:  
      `https://gist.githubusercontent.com/teds-tech-talks/02c4bb157627e5e9031df996ffa7d2d2/raw/9a058ba2866c9853c52cdd136c74276b883af04a/models.xsl`  
    - Response Format: File (to get raw file data)  
    - Connect XSL Webhook output to this node.

11. **Add Respond to Webhook Node (Serve XSL):**  
    - Type: Respond to Webhook  
    - Response Code: 200  
    - Response Headers:  
      - Content-Type: `text/xsl`  
    - Respond with: `binary`  
    - Connect HTTP Request output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This webhook is needed to comply with the CORS policy of modern browsers. It requests XSL template stored on GitHub gist and serves it using your n8n URL. | See Sticky Note near XSL Template Delivery block                                                        |
| XML node has an activated Headless toggle to avoid multiple XML declarations in output.                        | See Sticky Note near Data Export and XML Generation block                                                |
| The environment variable `{{$env.WEBHOOK_URL}}` dynamically inserts the n8n instance URL to build correct stylesheet links. | Important for deployment flexibility                                                                    |
| Project demonstrates advanced XML handling and stylesheet embedding in n8n workflows.                         | Useful reference for integrating SQL data export with XML/XSL                                            |
| GitHub gist link for XSL template: https://gist.githubusercontent.com/teds-tech-talks/02c4bb157627e5e9031df996ffa7d2d2/raw/9a058ba2866c9853c52cdd136c74276b883af04a/models.xsl | Used by HTTP Request node to fetch XSL stylesheet                                                        |

---

This documentation fully describes the workflow’s purpose, structure, nodes, configuration, and reproduction steps, ensuring clarity for both human developers and automation agents.