Sign PDF Documents with X.509 Certificates using PAdES Standards

https://n8nworkflows.xyz/workflows/sign-pdf-documents-with-x-509-certificates-using-pades-standards-10606


# Sign PDF Documents with X.509 Certificates using PAdES Standards

### 1. Workflow Overview

This workflow provides a comprehensive solution for digitally signing PDF documents using X.509 certificates in compliance with PAdES standards. It targets use cases where secure and legally compliant digital signatures are required, such as contract signing, official document verification, and archival. The workflow supports multiple PAdES signature levels (B, T, LT, LTA) with options for visible or invisible signatures.

The workflow is logically divided into the following blocks:

- **1.1 Webhook Reception and File Handling:** Accepts multipart POST requests with PDF, PFX certificate, password, signature level, and visibility option; writes uploaded files to temporary storage.
- **1.2 Dependency and Environment Setup:** Checks for required Java runtime environment and installs dependencies if missing.
- **1.3 Certificate Processing:** Extracts certificate and private key components from the uploaded PFX file for signing.
- **1.4 Signature Parameter Determination:** Routes the flow based on signature visibility and level, setting appropriate parameters.
- **1.5 PDF Signing Execution:** Executes the signing shell script with constructed parameters to produce the signed PDF.
- **1.6 Response and Cleanup:** Returns the signed PDF to the client with proper headers and cleans up temporary files.
- **1.7 Landing Page Management:** Serves localized HTML landing pages for user interaction in English, German, and Hungarian.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Reception and File Handling

- **Overview:**  
  This block receives the signing request via a webhook. It accepts multipart form data including the PDF to sign, PFX certificate with password, signature level, and visibility flag. Uploaded files are saved as temporary files for further processing.

- **Nodes Involved:**  
  - Webhook  
  - Write Files : PDF  
  - Write Files : PFX  
  - Write Files : LOGO

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook (HTTP POST)  
    - Configuration: Path `/pdf-sign`, POST method, response mode set to wait for response node  
    - Inputs: HTTP request with multipart form data (pdfFile, pfxFile, pfxPassword, signLevel, isVisible)  
    - Outputs: JSON body and binary data containing uploaded files  
    - Potential Failures: Missing files, malformed requests, unsupported HTTP method  
  - **Write Files : PDF**  
    - Type: Read & Write File  
    - Configuration: Writes binary PDF data to `/tmp/testpdf.pdf`  
    - Input: PDF binary data from webhook  
    - Output: Passes JSON with file reference  
    - Edge Cases: Disk write failures, permission issues  
  - **Write Files : PFX**  
    - Type: Read & Write File  
    - Configuration: Writes binary PFX certificate to `/tmp/testcert.pfx`  
    - Input: PFX binary data from webhook  
    - Output: Passes JSON with file reference  
    - Edge Cases: Disk write failures, permission issues  
  - **Write Files : LOGO**  
    - Type: Read & Write File  
    - Configuration: Writes optional logo PNG to `/tmp/logo.png`  
    - Input: Optional binary logo data from webhook (if provided)  
    - Output: Passes JSON with file reference  
    - Edge Cases: File not provided (node configured to continue on error)  

---

#### 2.2 Dependency and Environment Setup

- **Overview:**  
  Ensures the environment has Java installed (required for the signing library). If Java is missing, downloads necessary scripts and JAR files, installs dependencies, and adds test certificate authority trust for testing.

- **Nodes Involved:**  
  - Check Java  
  - Java Missing? (IF)  
  - Get Install Script  
  - Write Install Script  
  - Install Dependencies  
  - Get JAR  
  - Write JAR  
  - Get Trust Script  
  - Write Trust Script  
  - Add Codegic Trust

- **Node Details:**  
  - **Check Java**  
    - Type: Execute Command  
    - Command: `which keytool` to check Java presence  
    - Output: Exit code and error string  
    - Edge Cases: Command not found, permission denied  
  - **Java Missing?**  
    - Type: IF node  
    - Conditions: Checks if exit code ≤ 0 and error message does not contain "keytool"  
    - Routes to install scripts if Java missing  
  - **Get Install Script, Write Install Script, Install Dependencies**  
    - Type: HTTP Request + File Write + Execute Command  
    - Downloads a shell script to install dependencies and executes it  
    - Edge Cases: Network failures, script execution errors  
  - **Get JAR, Write JAR**  
    - Type: HTTP Request + File Write  
    - Downloads the PDF signing JAR (open-pdf-sign.jar) and saves it  
  - **Get Trust Script, Write Trust Script, Add Codegic Trust**  
    - Type: HTTP Request + File Write + Execute Command  
    - Adds test CA certificates to Java truststore for testing  
    - Skippable in production environment  

---

#### 2.3 Certificate Processing

- **Overview:**  
  Extracts certificate chain and private key PEM files from the uploaded PFX file using a shell script. These components are required for the signing process.

- **Nodes Involved:**  
  - Extract Password  
  - Get Cert Script  
  - Write Cert Script  
  - Run Cert Script  
  - Read PFX  
  - Write PFX

- **Node Details:**  
  - **Extract Password**  
    - Type: Set  
    - Extracts `pfxPassword` from webhook input JSON and sets a timestamp for unique filenames  
  - **Get Cert Script, Write Cert Script**  
    - Downloads and writes the `pfx-split.sh` script used for splitting the PFX file  
  - **Run Cert Script**  
    - Executes `pfx-split.sh` with PFX file path, password, and timestamp parameters to generate PEM files  
    - Edge Cases: Incorrect password, corrupted PFX, script failures  
  - **Read PFX, Write PFX**  
    - Reads the PFX from `/tmp/testcert.pfx` and writes it with timestamped filename for processing  

---

#### 2.4 Signature Parameter Determination

- **Overview:**  
  Determines signature visibility and signature level. According to the `isVisible` flag and `signLevel` value, sets parameters to control the signing process, including visibility flags and PAdES profile.

- **Nodes Involved:**  
  - Without Visible? (IF)  
  - No Visible (Set)  
  - Show Visible (Set)  
  - Switch Sign Visible (Switch)  
  - B (Set)  
  - T (Set)  
  - LT (Set)  
  - LTA (Set)  
  - Components (Set)

- **Node Details:**  
  - **Without Visible?**  
    - IF node checking `isVisible` field presence and value (`false` or not present)  
    - Routes to invisible or visible signing branches  
  - **No Visible**  
    - Sets `visibleParams` as empty string (no visible signature parameters)  
  - **Show Visible**  
    - Sets `visibleParams` to `" -v -p 1"` (parameters enabling visible signature on page 1)  
  - **Switch Sign Visible**  
    - Switch node on `signLevel` input (B, T, LT, LTA)  
    - Routes to respective nodes setting signature levels  
  - **B, T, LT, LTA**  
    - Each sets the `signatureLevel` string parameter according to PAdES standards  
  - **Components**  
    - Constructs a command-line string with input PDF, output PDF, certificate PEM, and private key PEM paths using the timestamped filenames  

---

#### 2.5 PDF Signing Execution

- **Overview:**  
  Executes the signing shell script with all constructed parameters, generating the signed PDF file.

- **Nodes Involved:**  
  - Sign PDF

- **Node Details:**  
  - **Sign PDF**  
    - Type: Execute Command  
    - Command: Runs `/tmp/processpdf.sh` with components string, signature level, and visibility parameters  
    - Inputs: Parameters from previous nodes  
    - Outputs: Signed PDF saved as `/tmp/output_<timestamp>.pdf`  
    - Edge Cases: Script failures, parameter errors, Java runtime errors  

---

#### 2.6 Response and Cleanup

- **Overview:**  
  Reads the signed PDF file, sends it as a binary HTTP response with appropriate headers, then runs cleanup scripts to remove temporary files securely.

- **Nodes Involved:**  
  - Read Signed PDF  
  - Return Signed PDF  
  - Get Cleanup Script  
  - Write Cleanup Script  
  - Cleanup

- **Node Details:**  
  - **Read Signed PDF**  
    - Reads signed PDF from `/tmp/output_<timestamp>.pdf`  
  - **Return Signed PDF**  
    - Responds to webhook with headers:  
      - `Content-Type: application/pdf`  
      - `Content-Disposition: attachment; filename="signed.pdf"`  
    - Sends binary content of signed PDF  
  - **Get Cleanup Script, Write Cleanup Script**  
    - Downloads and writes cleanup shell script to `/tmp/cleanup.sh`  
  - **Cleanup**  
    - Executes cleanup script with timestamp to remove temporary files  
    - Ensures security and prevents file clutter  
    - Edge Cases: File permission errors, partial cleanup  

---

#### 2.7 Landing Page Management

- **Overview:**  
  Provides localized landing pages for user interaction and signature form submission in English, German, and Hungarian.

- **Nodes Involved:**  
  - Webhook - LandingPage-LangEN  
  - Get LandingPage-LangEN  
  - Respond to Webhook - LandingPage-LangEN  
  - Webhook - LandingPage-LangDE  
  - Get LandingPage-LangDE  
  - Respond to Webhook - LandingPage-LangDE  
  - Webhook - LandingPage-LangHU  
  - Get LandingPage-LangHU  
  - Respond to Webhook - LandingPage-LangHU  

- **Node Details:**  
  - **Webhook - LandingPage-LangXX**  
    - Webhook nodes exposing paths `/en/signpage`, `/de/signpage`, `/hu/signpage`  
  - **Get LandingPage-LangXX**  
    - HTTP Request nodes fetching HTML content from GitHub URLs for respective languages  
  - **Respond to Webhook - LandingPage-LangXX**  
    - Respond nodes returning HTML content with `Content-Type: text/html`  
  - Edge Cases: Network failures fetching HTML, missing pages, encoding issues  

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                         | Input Node(s)                     | Output Node(s)                                    | Sticky Note                                                                                                      |
|-------------------------------|-----------------------|--------------------------------------|---------------------------------|--------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook               | Entry point for PDF sign requests    |                                 | Write Files : PDF                                | ## Workflow Entry Point: Accepts multipart POST with PDF, PFX, password, signLevel, visibility /webhook/pdf-sign |
| Write Files : PDF             | ReadWriteFile         | Save uploaded PDF to temp file       | Webhook                         | Write Files : PFX                                | ## File Upload Process                                                                                           |
| Write Files : PFX             | ReadWriteFile         | Save uploaded PFX certificate file   | Write Files : PDF               | Write Files : LOGO                               | ## File Upload Process                                                                                           |
| Write Files : LOGO            | ReadWriteFile         | Save optional logo image              | Write Files : PFX               | Check Java                                       | ## File Upload Process                                                                                           |
| Check Java                   | ExecuteCommand        | Verify Java runtime presence          | Write Files : LOGO              | Java Missing?                                    | ## Dependency Management: checks 'keytool' availability                                                        |
| Java Missing?                | IF                    | Branch for Java missing or present    | Check Java                     | Get JAR (true), Get Install Script (false)      | ## Dependency Management                                                                                         |
| Get Install Script           | HTTP Request          | Download install dependencies script  | Java Missing?                  | Write Install Script                             | ## Dependency Management                                                                                         |
| Write Install Script         | ReadWriteFile         | Save install dependencies script      | Get Install Script             | Install Dependencies                             | ## Dependency Management                                                                                         |
| Install Dependencies         | ExecuteCommand        | Run installation script                | Write Install Script           | Get JAR                                         | ## Dependency Management                                                                                         |
| Get JAR                     | HTTP Request          | Download PDF signing JAR file          | Java Missing?, Install Dependencies | Write JAR                                     | ## Dependency Management                                                                                         |
| Write JAR                   | ReadWriteFile         | Save JAR file                         | Get JAR                       | Extract Password                                | ## Dependency Management                                                                                         |
| Extract Password            | Set                   | Extract PFX password, set timestamp   | Write JAR                     | Read PDF                                        | ## Certificate Processing                                                                                        |
| Read PDF                    | ReadWriteFile         | Read uploaded PDF for processing      | Extract Password              | Write PDF                                       | ## Certificate Processing                                                                                        |
| Write PDF                   | ReadWriteFile         | Write input PDF with timestamped name | Read PDF                      | Get Trust Script                                | ## Certificate Processing                                                                                        |
| Get Trust Script            | HTTP Request          | Download trust certificate script     | Write PDF                     | Write Trust Script                              | ## Dependency Management, Testing Help                                                                           |
| Write Trust Script          | ReadWriteFile         | Save trust adding script               | Get Trust Script              | Add Codegic Trust                               | ## Dependency Management, Testing Help                                                                           |
| Add Codegic Trust           | ExecuteCommand        | Add test CA certificates               | Write Trust Script            | Read PFX                                        | ## Dependency Management, Testing Help                                                                           |
| Read PFX                    | ReadWriteFile         | Read uploaded PFX file                 | Add Codegic Trust             | Write PFX                                       | ## Certificate Processing                                                                                        |
| Write PFX                   | ReadWriteFile         | Write PFX with timestamped filename   | Read PFX                     | Get Cert Script                                 | ## Certificate Processing                                                                                        |
| Get Cert Script             | HTTP Request          | Download PFX split script              | Write PDF                    | Write Cert Script                               | ## Certificate Processing                                                                                        |
| Write Cert Script           | ReadWriteFile         | Save PFX split script                  | Get Cert Script              | Run Cert Script                                 | ## Certificate Processing                                                                                        |
| Run Cert Script             | ExecuteCommand        | Extract PEM cert and key from PFX     | Write Cert Script            | Get Process Script                              | ## Certificate Processing                                                                                        |
| Get Process Script          | HTTP Request          | Download PDF processing script         | Run Cert Script              | Write Process Script                            | ## PDF Signing Execution                                                                                          |
| Write Process Script        | ReadWriteFile         | Save processing shell script           | Get Process Script           | Without Visible?                                | ## PDF Signing Execution                                                                                          |
| Without Visible?            | IF                    | Check if signature should be invisible | Write Process Script          | No Visible, Show Visible                         | ## Signature Parameter Determination                                                                             |
| No Visible                  | Set                   | Set parameters for invisible signature | Without Visible?              | Switch Sign Visible                             | ## Signature Parameter Determination                                                                             |
| Show Visible               | Set                   | Set parameters for visible signature   | Without Visible?              | Switch Sign Visible                             | ## Signature Parameter Determination                                                                             |
| Switch Sign Visible         | Switch                | Route based on signature level (B,T,LT,LTA) | No Visible, Show Visible    | B, T, LT, LTA                                   | ## Signature Parameter Determination                                                                             |
| B                          | Set                   | Set parameters for BASELINE-B level    | Switch Sign Visible           | Components                                      | ## Signature Parameter Determination                                                                             |
| T                          | Set                   | Set parameters for BASELINE-T level    | Switch Sign Visible           | Components                                      | ## Signature Parameter Determination                                                                             |
| LT                         | Set                   | Set parameters for BASELINE-LT level   | Switch Sign Visible           | Components                                      | ## Signature Parameter Determination                                                                             |
| LTA                        | Set                   | Set parameters for BASELINE-LTA level  | Switch Sign Visible           | Components                                      | ## Signature Parameter Determination                                                                             |
| Components                 | Set                   | Build command line parameters for signing | B, T, LT, LTA               | Sign PDF                                        | ## PDF Signing Execution                                                                                          |
| Sign PDF                   | ExecuteCommand        | Run signing script with parameters      | Components                   | Read Signed PDF                                 | ## PDF Signing Execution                                                                                          |
| Read Signed PDF            | ReadBinaryFile        | Read signed PDF from file                | Sign PDF                     | Return Signed PDF                               | ## Response Handling                                                                                              |
| Return Signed PDF          | RespondToWebhook      | Return signed PDF to client              | Read Signed PDF              | Get Cleanup Script                              | ## Response Handling                                                                                              |
| Get Cleanup Script         | HTTP Request          | Download cleanup shell script             | Return Signed PDF            | Write Cleanup Script                            | ## Response Handling                                                                                              |
| Write Cleanup Script       | ReadWriteFile         | Save cleanup script                       | Get Cleanup Script           | Cleanup                                         | ## Response Handling                                                                                              |
| Cleanup                   | ExecuteCommand        | Execute cleanup script                     | Write Cleanup Script         |                                                  | ## Response Handling                                                                                              |
| Webhook - LandingPage-LangEN | Webhook             | Serve English landing page                |                            | Get LandingPage-LangEN                          | ## Landing Pages for signature form                                                                              |
| Get LandingPage-LangEN     | HTTP Request          | Download English landing page HTML        | Webhook - LandingPage-LangEN | Respond to Webhook - LandingPage-LangEN        | ## Landing Pages for signature form                                                                              |
| Respond to Webhook - LandingPage-LangEN | RespondToWebhook | Respond with English landing page HTML | Get LandingPage-LangEN       |                                                  | ## Landing Pages for signature form                                                                              |
| Webhook - LandingPage-LangDE | Webhook             | Serve German landing page                  |                            | Get LandingPage-LangDE                          | ## Landing Pages for signature form                                                                              |
| Get LandingPage-LangDE     | HTTP Request          | Download German landing page HTML          | Webhook - LandingPage-LangDE | Respond to Webhook - LandingPage-LangDE        | ## Landing Pages for signature form                                                                              |
| Respond to Webhook - LandingPage-LangDE | RespondToWebhook | Respond with German landing page HTML    | Get LandingPage-LangDE       |                                                  | ## Landing Pages for signature form                                                                              |
| Webhook - LandingPage-LangHU | Webhook             | Serve Hungarian landing page               |                            | Get LandingPage-LangHU                          | ## Landing Pages for signature form                                                                              |
| Get LandingPage-LangHU     | HTTP Request          | Download Hungarian landing page HTML       | Webhook - LandingPage-LangHU | Respond to Webhook - LandingPage-LangHU        | ## Landing Pages for signature form                                                                              |
| Respond to Webhook - LandingPage-LangHU | RespondToWebhook | Respond with Hungarian landing page HTML | Get LandingPage-LangHU       |                                                  | ## Landing Pages for signature form                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Webhook`  
   - Path: `/pdf-sign`  
   - HTTP Method: POST  
   - Response Mode: `Response Node`  
   - Accept multipart form data fields: `pdfFile`, `pfxFile`, `pfxPassword`, `signLevel`, `isVisible`  

2. **Write Uploaded Files:**  
   - Create `Write Files : PDF` node (ReadWriteFile)  
     - Write binary `pdfFile` to `/tmp/testpdf.pdf`  
   - Connect `Webhook` → `Write Files : PDF`  
   - Create `Write Files : PFX` node (ReadWriteFile)  
     - Write binary `pfxFile` to `/tmp/testcert.pfx`  
   - Connect `Write Files : PDF` → `Write Files : PFX`  
   - Create `Write Files : LOGO` node (ReadWriteFile)  
     - Write binary logo image (optional) to `/tmp/logo.png`  
     - Set "Continue On Fail" to true (in case logo not provided)  
   - Connect `Write Files : PFX` → `Write Files : LOGO`  

3. **Check Java Installation:**  
   - Create `Check Java` node (Execute Command)  
     - Command: `which keytool`  
   - Connect `Write Files : LOGO` → `Check Java`  

4. **Java Missing? Conditional Branch:**  
   - Create `Java Missing?` node (IF)  
     - Condition:  
       - Number: `exitCode <= 0`  
       - String: `error` does not contain `"keytool"`  
   - Connect `Check Java` → `Java Missing?`  

5. **If Java Missing:**  
   - Create `Get Install Script` (HTTP Request)  
     - URL: `https://raw.githubusercontent.com/vighsandor/n8n-externals/main/landingpages-for-pdfsign-workflow/install-deps.sh`  
     - Response: File, save to property `installdeps`  
   - Connect `Java Missing? (true)` → `Get Install Script`  
   - Create `Write Install Script` (ReadWriteFile)  
     - Write `installdeps` to `/tmp/install-deps.sh`  
   - Connect `Get Install Script` → `Write Install Script`  
   - Create `Install Dependencies` (Execute Command)  
     - Command: `sh /tmp/install-deps.sh`  
   - Connect `Write Install Script` → `Install Dependencies`  
   - Create `Get JAR` (HTTP Request)  
     - URL: `https://github.com/open-pdf-sign/open-pdf-sign/releases/download/v0.3.0/open-pdf-sign.jar`  
     - Response: File, save to `open-pdf-sign`  
   - Connect `Install Dependencies` → `Get JAR`  

6. **If Java Present:**  
   - Connect `Java Missing? (false)` → `Get JAR`  

7. **Write JAR File:**  
   - Create `Write JAR` (ReadWriteFile)  
     - Write `open-pdf-sign` binary to `/tmp/open-pdf-sign.jar`  
   - Connect `Get JAR` → `Write JAR`  

8. **Extract Password and Timestamp:**  
   - Create `Extract Password` (Set)  
     - Set variable `pfxPassword` to `{{$json.body.pfxPassword}}` from webhook input  
     - Set variable `timestamp` to current timestamp (`{{Date.now()}}`)  
   - Connect `Write JAR` → `Extract Password`  

9. **Read and Write PDF with Timestamp:**  
   - Create `Read PDF` (ReadWriteFile)  
     - Read `/tmp/testpdf.pdf` into property `toProcess`  
   - Connect `Extract Password` → `Read PDF`  
   - Create `Write PDF` (ReadWriteFile)  
     - Write `toProcess` binary to `/tmp/input_{{$json.timestamp}}.pdf`  
   - Connect `Read PDF` → `Write PDF`  

10. **Download and Write Trust Script:**  
    - Create `Get Trust Script` (HTTP Request)  
      - URL: `https://raw.githubusercontent.com/vighsandor/n8n-externals/main/landingpages-for-pdfsign-workflow/addtestcatrust.sh`  
      - Response as file to `addtestcatrust`  
    - Connect `Write PDF` → `Get Trust Script`  
    - Create `Write Trust Script` (ReadWriteFile)  
      - Write to `/tmp/addtestcatrust.sh`  
    - Connect `Get Trust Script` → `Write Trust Script`  
    - Create `Add Codegic Trust` (Execute Command)  
      - Command: `sh /tmp/addtestcatrust.sh`  
    - Connect `Write Trust Script` → `Add Codegic Trust`  

11. **Read and Write PFX with Timestamp:**  
    - Create `Read PFX` (ReadWriteFile)  
      - Read `/tmp/testcert.pfx` into `withProcess`  
    - Connect `Add Codegic Trust` → `Read PFX`  
    - Create `Write PFX` (ReadWriteFile)  
      - Write `withProcess` to `/tmp/cert_{{$json.timestamp}}.pfx`  
    - Connect `Read PFX` → `Write PFX`  

12. **Download and Write Certificate Split Script:**  
    - Create `Get Cert Script` (HTTP Request)  
      - URL: `https://raw.githubusercontent.com/vighsandor/n8n-externals/main/landingpages-for-pdfsign-workflow/pfx-split.sh`  
      - Response as file to `pfx-split`  
    - Connect `Write PFX` → `Get Cert Script`  
    - Create `Write Cert Script` (ReadWriteFile)  
      - Write to `/tmp/pfx-split.sh`  
    - Connect `Get Cert Script` → `Write Cert Script`  

13. **Run Certificate Split Script:**  
    - Create `Run Cert Script` (Execute Command)  
      - Command: `sh /tmp/pfx-split.sh /tmp/cert_{{$json.timestamp}}.pfx {{$json.pfxPassword}} {{$json.timestamp}}`  
    - Connect `Write Cert Script` → `Run Cert Script`  

14. **Download and Write PDF Processing Script:**  
    - Create `Get Process Script` (HTTP Request)  
      - URL: `https://raw.githubusercontent.com/vighsandor/n8n-externals/main/landingpages-for-pdfsign-workflow/processpdf.sh`  
      - Response as file to `processpdf`  
    - Connect `Run Cert Script` → `Get Process Script`  
    - Create `Write Process Script` (ReadWriteFile)  
      - Write to `/tmp/processpdf.sh`  
    - Connect `Get Process Script` → `Write Process Script`  

15. **Check Signature Visibility:**  
    - Create `Without Visible?` (IF)  
      - Condition: If `isVisible` is `"false"` or missing  
    - Connect `Write Process Script` → `Without Visible?`  

16. **Set Visibility Parameters:**  
    - Create `No Visible` (Set)  
      - Set `visibleParams` to empty string `" "`  
    - Create `Show Visible` (Set)  
      - Set `visibleParams` to `" -v -p 1"`  
    - Connect `Without Visible?` outputs: true → `No Visible`, false → `Show Visible`  

17. **Switch Signature Level:**  
    - Create `Switch Sign Visible` (Switch)  
      - Switch on `signLevel` input with cases `B`, `T`, `LT`, `LTA`  
    - Connect `No Visible` → `Switch Sign Visible`  
    - Connect `Show Visible` → `Switch Sign Visible`  

18. **Set Signature Level Parameters:**  
    - Create `B` (Set) → set `signatureLevel` to `' basic'`  
    - Create `T` (Set) → set `signatureLevel` to `'timestamp'`  
    - Create `LT` (Set) → set `signatureLevel` to `'baseline-lt'`  
    - Create `LTA` (Set) → set `signatureLevel` to `'baseline-lta'`  
    - Connect `Switch Sign Visible` outputs accordingly  

19. **Build Signing Command Components:**  
    - Create `Components` (Set)  
      - Constructs string:  
        `-i /tmp/input_{{$json.timestamp}}.pdf -o /tmp/output_{{$json.timestamp}}.pdf -c /tmp/cert_{{$json.timestamp}}.pem -k /tmp/key_{{$json.timestamp}}.pem`  
    - Connect each of `B`, `T`, `LT`, `LTA` → `Components`  

20. **Execute PDF Signing:**  
    - Create `Sign PDF` (Execute Command)  
      - Command: `sh /tmp/processpdf.sh {{ $json.components }} -m {{ $json.signatureLevel }} {{ $json.visibleParams }}`  
    - Connect `Components` → `Sign PDF`  

21. **Read and Return Signed PDF:**  
    - Create `Read Signed PDF` (Read Binary File)  
      - Reads `/tmp/output_{{$json.timestamp}}.pdf`  
    - Connect `Sign PDF` → `Read Signed PDF`  
    - Create `Return Signed PDF` (RespondToWebhook)  
      - Respond with headers:  
        - `Content-Type: application/pdf`  
        - `Content-Disposition: attachment; filename="signed.pdf"`  
      - Respond with binary data  
    - Connect `Read Signed PDF` → `Return Signed PDF`  

22. **Cleanup Temporary Files:**  
    - Create `Get Cleanup Script` (HTTP Request)  
      - URL: `https://raw.githubusercontent.com/vighsandor/n8n-externals/main/landingpages-for-pdfsign-workflow/cleanup.sh`  
      - Response as file to `cleanup`  
    - Connect `Return Signed PDF` → `Get Cleanup Script`  
    - Create `Write Cleanup Script` (ReadWriteFile)  
      - Write to `/tmp/cleanup.sh`  
    - Connect `Get Cleanup Script` → `Write Cleanup Script`  
    - Create `Cleanup` (Execute Command)  
      - Command: `sh /tmp/cleanup.sh {{$json.timestamp}}`  
    - Connect `Write Cleanup Script` → `Cleanup`  

23. **Setup Landing Pages:**  
    For each language EN, DE, HU:  
    - Create `Webhook - LandingPage-LangXX` (Webhook) with path `/xx/signpage`  
    - Create `Get LandingPage-LangXX` (HTTP Request)  
      - URL: `https://raw.githubusercontent.com/vighsandor/n8n-externals/main/landingpages-for-pdfsign-workflow/xx.html`  
      - Response as file  
    - Create `Respond to Webhook - LandingPage-LangXX` (RespondToWebhook)  
      - Respond with content-type `text/html`  
    - Connect `Webhook - LandingPage-LangXX` → `Get LandingPage-LangXX` → `Respond to Webhook - LandingPage-LangXX`  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow requires Java 11 JRE to run the signing library. If missing, it installs automatically.                                                 | Dependency Management block                                                                                      |
| Testing certificates and trust additions are included for demo purposes; in production, these steps can be skipped.                                  | Sticky Notes on Testing Help                                                                                     |
| Visible signatures place a signature image at the top-left corner, customizable with extra parameters such as hint text and labels.                  | Implementation information sticky note                                                                           |
| The webhook endpoint is `/pdf-sign` accepting multipart POST with PDF, PFX, password, signLevel, and visibility.                                     | Workflow Entry Point sticky note                                                                                 |
| Temporary files are stored under `/tmp` with timestamp suffixes for uniqueness and security.                                                        | Configuration Notes sticky note                                                                                   |
| Landing pages for signature form interface are available in English, German, and Hungarian at `/en/signpage`, `/de/signpage`, `/hu/signpage`.        | Landing Pages sticky note                                                                                         |
| CURL examples for API usage are available at https://github.com/vighsandor/n8n-externals/blob/main/landingpages-for-pdfsign-workflow/curl-examples.md | See Sticky Note with CURL examples                                                                                |
| PDF signing uses the open-pdf-sign open-source library available at https://github.com/open-pdf-sign/open-pdf-sign                                   | PDF Signing information sticky note                                                                               |
| For signature levels beyond BASELINE-T, a full production certificate with revocation info is required.                                               | Testing Help sticky note                                                                                          |
| The workflow cleans up temporary files after each request to prevent security risks and disk clutter.                                                | Response Handling sticky note                                                                                      |

---

**Disclaimer:**  
The provided text and workflow derive exclusively from an automated workflow created with n8n, an integration and automation tool. The process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.