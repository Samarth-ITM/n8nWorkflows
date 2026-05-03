One-way sync Stripe Invoice PDFs to a S3 Bucket

https://n8nworkflows.xyz/workflows/one-way-sync-stripe-invoice-pdfs-to-a-s3-bucket-2450


# One-way sync Stripe Invoice PDFs to a S3 Bucket

### 1. Workflow Overview

This workflow automates the monthly synchronization of Stripe Invoice PDFs into an AWS S3 bucket. It fetches invoices created after a configurable date (defaulting to the first day of the previous month), downloads the corresponding PDF files from Stripe, and uploads them to an S3 bucket arranged in a structured folder hierarchy based on year and month.

The workflow is logically divided into these blocks:

- **1.1 Trigger Block:** Initiates the workflow manually or on a monthly schedule.
- **1.2 Environment Setup Block:** Sets and cleans environment variables controlling the target sync period and S3 folder configuration.
- **1.3 Stripe Invoice Retrieval Block:** Queries Stripe API to retrieve invoices created after the specified date.
- **1.4 Invoice Filtering Block:** Filters objects to ensure only invoices are processed; stops workflow on unexpected data.
- **1.5 Invoice PDF Download Block:** Downloads the invoice PDF from Stripe.
- **1.6 S3 Path Construction Block:** Builds the target folder path and filename for S3 storage.
- **1.7 S3 Upload Block:** Uploads the downloaded PDF into the specified S3 bucket and path.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

**Overview:**  
Defines how and when the workflow runs. Supports manual triggering via UI and automated monthly runs.

**Nodes Involved:**  
- `When clicking ‘Test workflow’` (Manual Trigger)  
- `Every Month the First Day of the Month` (Schedule Trigger)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
  - Role: Enables manual execution for testing or on-demand sync  
  - Configuration: Default, no parameters  
  - Output: Triggers the ENV* node  
  - Edge Cases: None specific; user must manually trigger

- **Every Month the First Day of the Month**  
  - Type: Schedule Trigger (n8n-nodes-base.scheduleTrigger)  
  - Role: Automatically triggers workflow once per month  
  - Configuration: Interval set to every 1 month  
  - Output: Triggers the ENV* node  
  - Edge Cases: Time zone considerations may affect exact trigger time

---

#### 1.2 Environment Setup Block

**Overview:**  
Sets and sanitizes key environment variables including the year, month, S3 folder, and bucket name used throughout the workflow. Supports overrides for manual syncs.

**Nodes Involved:**  
- `ENV*` (Set)  
- `Clean and Escape ENV` (Set)

**Node Details:**

- **ENV***  
  - Type: Set (n8n-nodes-base.set)  
  - Role: Initializes variables: `year`, `month`, `subFolder`, and `bucketName`  
  - Configuration:  
    - `year` and `month`: Default to last month dynamically (`$now.minus(1,"month")`)  
    - `subFolder`: Default `"invoices"` (optional subfolder in S3)  
    - `bucketName`: Placeholder `"myBucket"` (user must replace with actual bucket)  
  - Output: Passes variables to `Clean and Escape ENV`  
  - Edge Cases: Manual overrides must respect formatting; invalid months or bucket names can cause failure

- **Clean and Escape ENV**  
  - Type: Set (n8n-nodes-base.set)  
  - Role: Cleans and formats environment variables to prevent errors due to whitespace or escape characters  
  - Configuration:  
    - Trims `bucketName` and `subFolder`, removes backslashes  
    - Pads month to two digits (e.g., "03")  
    - Ensures `year` is an integer  
  - Output: Passes sanitized variables to Stripe API request node  
  - Edge Cases: Unexpected input types could cause expression failures

---

#### 1.3 Stripe Invoice Retrieval Block

**Overview:**  
Fetches invoices from Stripe API created after the configured year-month date.

**Nodes Involved:**  
- `Get all Invoices*` (HTTP Request)  
- `List Invoices` (Split Out)

**Node Details:**

- **Get all Invoices***  
  - Type: HTTP Request (n8n-nodes-base.httpRequest)  
  - Role: Calls Stripe API’s `/v1/invoices` endpoint  
  - Configuration:  
    - Query parameters:  
      - `limit=100` (max invoices per call)  
      - `created[gte]` set to timestamp of first day of specified year-month  
    - Authentication: Stripe predefined credential type  
  - Output: JSON response with invoices array in `data` field  
  - Edge Cases: Stripe API rate limits, auth errors, network issues

- **List Invoices**  
  - Type: Split Out (n8n-nodes-base.splitOut)  
  - Role: Splits the array of invoices from Stripe into individual items for processing  
  - Configuration: Splits on `data` array  
  - Output: One invoice per item to next node  
  - Edge Cases: Empty array yields no output; malformed data causes failures

---

#### 1.4 Invoice Filtering Block

**Overview:**  
Ensures only Stripe Invoice objects are processed and stops workflow on unexpected data.

**Nodes Involved:**  
- `We do only Invoice Objects` (If)  
- `It shouldn't be something else` (Stop and Error)

**Node Details:**

- **We do only Invoice Objects**  
  - Type: If (n8n-nodes-base.if)  
  - Role: Checks if `$json.object === "invoice"`  
  - Configuration: Strict case-sensitive comparison  
  - Output:  
    - True: Continue with PDF download  
    - False: Route to error node  
  - Edge Cases: Unexpected object types or missing `object` field

- **It shouldn't be something else**  
  - Type: Stop and Error (n8n-nodes-base.stopAndError)  
  - Role: Stops workflow execution with message `"Unexpected or missing Invoice Obj"`  
  - Configuration: Custom error message  
  - Edge Cases: Stops workflow on invalid data, preventing downstream errors

---

#### 1.5 Invoice PDF Download Block

**Overview:**  
Downloads the invoice PDF file from Stripe using the invoice’s `invoice_pdf` URL.

**Nodes Involved:**  
- `Download Invoice PDF from Stripe` (HTTP Request)  
- `Inject s3 Subpath` (Set)  
- `Set-Subpath` (Set)

**Node Details:**

- **Download Invoice PDF from Stripe**  
  - Type: HTTP Request (n8n-nodes-base.httpRequest)  
  - Role: Downloads binary PDF file from URL in `$json.invoice_pdf`  
  - Configuration: URL set dynamically from invoice JSON  
  - Output: Binary data for the PDF file  
  - Edge Cases: URL missing or inaccessible, network errors, Stripe token expiration

- **Inject s3 Subpath**  
  - Type: Set (n8n-nodes-base.set)  
  - Role: Injects variables to construct S3 path segments: year, month, folder  
  - Configuration:  
    - `_s3_year`: Extracted from invoice creation timestamp formatted as yyyy  
    - `_s3_month`: Extracted as MM  
    - `_s3_folder`: From cleaned `subFolder` environment variable  
  - Output: Passes with original fields plus these variables  
  - Edge Cases: Malformed timestamps cause expression failures

- **Set-Subpath**  
  - Type: Set (n8n-nodes-base.set)  
  - Role: Concatenates folder path and filename for S3 key  
  - Configuration:  
    - `_s3_path` = `${_s3_folder}/_s3_year/_s3_month/filename` (handles optional folder)  
    - Filename taken from binary data metadata `fileName`  
  - Output: Includes full S3 key path for upload  
  - Edge Cases: Missing fileName breaks path; empty folder results in root-level upload

---

#### 1.6 S3 Upload Block

**Overview:**  
Uploads the downloaded invoice PDF to the specified S3 bucket under the constructed path.

**Nodes Involved:**  
- `Upload to S3 Bucket*` (AWS S3)

**Node Details:**

- **Upload to S3 Bucket***  
  - Type: AWS S3 (n8n-nodes-base.awsS3)  
  - Role: Uploads the binary PDF file to S3 using the constructed key path  
  - Configuration:  
    - Operation: `upload`  
    - Bucket Name: dynamically from cleaned environment variable  
    - File Name (key): from `_s3_path`  
    - Additional Fields: Storage Class set to `intelligentTiering` (cost optimization)  
  - Credentials: User must configure AWS credentials with write access  
  - Edge Cases: AWS permission errors, incorrect bucket name, network timeouts, invalid keys

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                      | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                               |
|-------------------------------|---------------------|------------------------------------|----------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger      | Manual workflow start              | —                                | ENV*                           |                                                                                                           |
| Every Month the First Day of the Month | Schedule Trigger  | Monthly automatic workflow start   | —                                | ENV*                           |                                                                                                           |
| ENV*                          | Set                 | Initialize environment variables    | When clicking ‘Test workflow’, Every Month the First Day of the Month | Clean and Escape ENV           | ## 👇  Configure here<br><br>`folderName` *(optional)* = Subfolder for your Invoices.<br>`bucketName` *(required)* = the S3 Bucket Name.<br>`year` and `month` default to last month but can be overridden. |
| Clean and Escape ENV           | Set                 | Sanitize environment variables      | ENV*                            | Get all Invoices*              |                                                                                                           |
| Get all Invoices*              | HTTP Request        | Retrieve invoices from Stripe API   | Clean and Escape ENV             | List Invoices                  | ### Use Stripe Predefined Credential                                                                       |
| List Invoices                 | Split Out           | Split invoice array into items      | Get all Invoices*                | We do only Invoice Objects     |                                                                                                           |
| We do only Invoice Objects     | If                  | Filter only invoice objects         | List Invoices                   | Download Invoice PDF from Stripe, It shouldn't be something else |                                                                                                           |
| It shouldn't be something else | Stop and Error      | Stop on unexpected data             | We do only Invoice Objects       | —                              |                                                                                                           |
| Download Invoice PDF from Stripe | HTTP Request       | Download invoice PDF binary         | We do only Invoice Objects       | Inject s3 Subpath              |                                                                                                           |
| Inject s3 Subpath              | Set                 | Extract year, month, folder for S3 | Download Invoice PDF from Stripe | Set-Subpath                   | ## Build Pathes<br><br>*yourFolder/invoiceYear/invoiceMonth/fileName*<br>e.g.: invoices/2024/12/invoice-number-123.pdf |
| Set-Subpath                   | Set                 | Construct full S3 key path          | Inject s3 Subpath                | Upload to S3 Bucket*           | ## Upload to Bucket<br>**⚠️ You might want to check Storage Class, ACL, etc.**                              |
| Upload to S3 Bucket*           | AWS S3               | Upload PDF to S3 bucket             | Set-Subpath                     | —                              |                                                                                                           |
| Sticky Note                   | Sticky Note          | Configuration instructions          | —                                | —                              | ## 👇  Configure here\n\n`folderName` *(optional)*\n`bucketName` *(required)*\n`year` and `month` defaults. |
| Sticky Note1                  | Sticky Note          | Explains folder path structure      | —                                | —                              | ## Build Pathes\n\n*yourFolder/invoiceYear/invoiceMonth/fileName*                                         |
| Sticky Note2                  | Sticky Note          | Reminder about S3 upload options    | —                                | —                              | ## Upload to Bucket\n\n**⚠️ You might want to check Storage Class, ACL, etc.**                             |
| Sticky Note3                  | Sticky Note          | General instructions and credits    | —                                | —                              | ## Instructions\n\nSync Stripe Invoice PDFs monthly to S3.\n\n[https://let-the-work-flow.com](https://let-the-work-flow.com) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`. No special configuration.  
   - Add a **Schedule Trigger** node named `Every Month the First Day of the Month`. Configure to trigger every 1 month.

2. **Create Environment Variable Setup**  
   - Add a **Set** node named `ENV*`.  
     - Add fields:  
       - `year` (number): Expression `{{$now.minus(1,"month").format("yyyy")}}`  
       - `month` (number): Expression `{{$now.minus(1,"month").format("MM")}}`  
       - `subFolder` (string): `"invoices"` (default, user can change)  
       - `bucketName` (string): `"myBucket"` (replace with your bucket name)  
   - Connect both trigger nodes to this node.

3. **Create Environment Variable Cleaning**  
   - Add a **Set** node named `Clean and Escape ENV`.  
   - Configure assignments:  
     - `bucketName`: Expression to trim and remove backslashes from previous `bucketName`  
     - `subFolder`: Expression to trim and remove backslashes from previous `subFolder`  
     - `month`: Expression to pad month to two digits (string)  
     - `year`: Expression to parse as integer  
   - Connect `ENV*` node to this node.

4. **Add Stripe API HTTP Request Node**  
   - Add an **HTTP Request** node named `Get all Invoices*`.  
   - Set Method: GET  
   - URL: `https://api.stripe.com/v1/invoices`  
   - Authentication: Use predefined Stripe API credential (must be configured in n8n)  
   - Query Parameters:  
     - `limit` = `100`  
     - `created[gte]` = Expression `{{ DateTime.fromISO($json.year+"-"+$json.month+"-01T00:00:00").toSeconds() }}`  
   - Connect `Clean and Escape ENV` node to this node.

5. **Split Invoice Array**  
   - Add a **Split Out** node named `List Invoices`.  
   - Field to split out: `data`  
   - Connect `Get all Invoices*` node to this node.

6. **Add Invoice Filter**  
   - Add an **If** node named `We do only Invoice Objects`.  
   - Condition: `$json.object` equals `"invoice"` (case sensitive, strict)  
   - Connect `List Invoices` to this node.

7. **Add Stop on Unexpected Objects**  
   - Add a **Stop and Error** node named `It shouldn't be something else`.  
   - Error message: `"Unexpected or missing Invoice Obj"`  
   - Connect the false output of `We do only Invoice Objects` to this node.

8. **Download Invoice PDF**  
   - Add an **HTTP Request** node named `Download Invoice PDF from Stripe`.  
   - Method: GET  
   - URL: Expression `{{$json.invoice_pdf}}`  
   - No authentication needed (public URL from Stripe)  
   - Connect true output of `We do only Invoice Objects` to this node.

9. **Inject S3 Path Variables**  
   - Add a **Set** node named `Inject s3 Subpath`.  
   - Assign:  
     - `_s3_year`: Expression `{{ DateTime.fromSeconds($json.created).format("yyyy") }}`  
     - `_s3_month`: Expression `{{ DateTime.fromSeconds($json.created).format("MM") }}`  
     - `_s3_folder`: Expression `{{ $("Clean and Escape ENV").first().json.subFolder }}`  
   - Include other fields to pass through.  
   - Connect `Download Invoice PDF from Stripe` to this node.

10. **Build Full S3 Key Path**  
    - Add a **Set** node named `Set-Subpath`.  
    - Assign:  
      - `_s3_path`: Expression `{{ ($json._s3_folder ? $json._s3_folder+"/" : "") + $json._s3_year + "/" + $json._s3_month + "/" + $binary.data.fileName }}`  
    - Include other fields to pass through.  
    - Connect `Inject s3 Subpath` to this node.

11. **Upload to S3 Bucket**  
    - Add an **AWS S3** node named `Upload to S3 Bucket*`.  
    - Operation: Upload  
    - Bucket Name: Expression `{{ $("Clean and Escape ENV").first().json.bucketName }}`  
    - File Name: Expression `{{ $json._s3_path }}`  
    - Additional Fields: Set Storage Class to `intelligentTiering`  
    - Credentials: Configure AWS credentials with write access to the target bucket  
    - Connect `Set-Subpath` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow syncs monthly your Invoice PDFs from Stripe to an AWS S3 Bucket with a folder structure: *yourFolder/invoiceYear/invoiceMonth/fileName*. You can override the `year` and `month` in the ENV* node for manual syncs. The workflow exports all invoices created after the specified first day of the month.                                                                                         | https://let-the-work-flow.com                 |
| `folderName` (optional) allows organizing invoices into a subfolder inside the bucket, default is `"invoices"`. The `bucketName` is required and must exist in AWS S3.                                                                                                                                                                                                                             | Configuration instructions in Sticky Note    |
| When uploading to S3, consider checking the Storage Class and Access Control List (ACL) settings to optimize cost and permissions. The workflow uses `intelligentTiering` storage class by default.                                                                                                                                                                                                | Sticky Note near Upload to S3 node            |
| Use Stripe predefined credentials in n8n for authentication with the Stripe API node. Ensure your API keys have the required permissions to read invoices.                                                                                                                                                                                                                                        | Sticky Note near Stripe API node              |
| The workflow includes error handling to stop if unexpected object types are encountered, preventing unnecessary API calls or storage attempts.                                                                                                                                                                                                                                                   | Business logic and error node                  |

---

This completes the full structured documentation for the "One-way sync Stripe Invoice PDFs to a S3 Bucket" workflow. It is designed to enable replication, modification, and troubleshooting by advanced users and automation agents.