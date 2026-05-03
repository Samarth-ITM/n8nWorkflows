Validate and Score Email Addresses with ZeroBounce AI

https://n8nworkflows.xyz/workflows/validate-and-score-email-addresses-with-zerobounce-ai-11498


# Validate and Score Email Addresses with ZeroBounce AI

### 1. Workflow Overview

This workflow is designed to validate and score email addresses using the ZeroBounce API, aiming to assess email deliverability and quality for marketing, CRM, or communication tools. It processes a batch of emails, validates each for legitimacy, checks account credits for API usage, optionally scores valid emails with AI-powered confidence scores, and categorizes them into confidence tiers for downstream action.

The workflow is logically divided into these blocks:

- **1.1 Input Preparation:** Loads test email addresses and splits them into individual items.
- **1.2 ZeroBounce Credits Check:** Optionally verifies available API credits for validation and scoring to avoid execution errors.
- **1.3 Email Validation:** Validates each email address using ZeroBounce’s validation API.
- **1.4 Validation Result Filtering:** Routes emails based on validation status (valid or not).
- **1.5 Conditional AI Scoring:** Checks for sufficient scoring credits and either proceeds to score valid emails or skips scoring.
- **1.6 Scoring and Merge:** Scores emails via ZeroBounce AI scoring API and merges scores with validation data.
- **1.7 Filtering by Score:** Categorizes scored emails into high, medium, or low confidence tiers.
- **1.8 Output Branches:** Provides filtered outputs for further use, e.g., marketing campaigns or suppression lists.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Preparation

- **Overview:** Creates a sample dataset of test emails (sandbox mode) and splits the email list into individual items for processing.
- **Nodes Involved:** `When clicking ‘Execute workflow’`, `Sandbox emails`, `Split into items`, `Sticky Note3`
- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Initiates workflow manually.
    - Inputs/Outputs: No inputs; outputs trigger `Sandbox emails`.
    - Edge Cases: None. Manual start only.

  - **Sandbox emails**
    - Type: Set node
    - Role: Provides a hardcoded list of 30 test emails for sandbox testing.
    - Configuration: JSON array of email strings under field `emails`.
    - Inputs: Trigger from manual.
    - Outputs: Single item with `emails` array.
    - Edge Cases: None; sandbox emails do not consume credits.

  - **Split into items**
    - Type: SplitOut
    - Role: Converts the single array of emails into separate items, each with an individual email.
    - Configuration: Splits on field `emails`; outputs individual email items.
    - Inputs: From `Sandbox emails`.
    - Outputs: Multiple items, each with `email` field.
    - Edge Cases: If input is missing or empty array, outputs no items.

  - **Sticky Note3**
    - Type: Sticky note for documentation.
    - Content: Explains the sandbox email data source and splitting logic.

#### 2.2 ZeroBounce Credits Check

- **Overview:** Checks if the ZeroBounce account has enough credits before performing validation or scoring to avoid runtime failures.
- **Nodes Involved:** `Check credits for validation`, `Check credits for scoring`, `Wait for validation credits check`, `Wait for scoring credits check`, `Sticky Note2`, `Sticky Note4`
- **Node Details:**

  - **Check credits for validation**
    - Type: ZeroBounce node (account resource)
    - Role: Checks if sufficient credits exist for validation of all input emails.
    - Parameters: `creditsRequired` set to total number of incoming items.
    - Credentials: Uses ZeroBounce API key.
    - Inputs: From `Split into items`.
    - Outputs: Single item with credit info.
    - Execute Once: Enabled.
    - Edge Cases: API errors, insufficient credits throw errors or halt workflow.

  - **Check credits for scoring**
    - Type: ZeroBounce node (account resource)
    - Role: Checks credits available for scoring emails.
    - Parameters: `creditsRequired` empty (optional check).
    - Credentials: ZeroBounce API key.
    - Inputs: From `Wait` node.
    - Outputs: Single item credit info.
    - Execute Once: Enabled.
    - Edge Cases: API errors, insufficient credits handled downstream.

  - **Wait for validation credits check**
    - Type: Merge node (combine all)
    - Role: Waits for credit check to complete before proceeding to validation.
    - Inputs: From `Check credits for validation` and `Split into items`.
    - Outputs: Proceeds to `Validate email`.
    - Edge Cases: Mismatch in input lengths.

  - **Wait for scoring credits check**
    - Type: Merge node (combine all)
    - Role: Waits for scoring credits check and validation results before scoring.
    - Inputs: From `Check credits for scoring` and `Is valid?`.
    - Outputs: Proceeds to `Skip scoring?`.
    - Edge Cases: Timing delays may affect credit accuracy.

  - **Sticky Notes 2 & 4**
    - Explain optional credit checks for validation and scoring, including API documentation links and best practices (e.g., preventing midway failures).

#### 2.3 Email Validation

- **Overview:** Validates each email address with ZeroBounce API, returning status and metadata.
- **Nodes Involved:** `Validate email`, `Is valid?`, `Not valid`, `Sticky Note`
- **Node Details:**

  - **Validate email**
    - Type: ZeroBounce node (validation resource)
    - Role: Validates individual email addresses.
    - Parameters: Uses `email` field, timeout 10 seconds.
    - Credentials: ZeroBounce API key.
    - Inputs: From `Wait for validation credits check`.
    - Outputs: Email status, e.g., 'valid', 'invalid', 'spamtrap'.
    - Edge Cases: Timeout, API errors, invalid input format.

  - **Is valid?**
    - Type: Switch node
    - Role: Routes emails based on validation status.
    - Parameters: Checks if `status == 'valid'`; outputs to "valid" or "other".
    - Inputs: From `Validate email`.
    - Outputs: "valid" branch to scoring flow; "other" branch to `Not valid`.
    - Edge Cases: Unexpected status strings.

  - **Not valid**
    - Type: NoOp (no operation)
    - Role: Endpoint for invalid or suspicious emails.
    - Inputs: From `Is valid?` "other" branch.
    - Outputs: None.
    - Edge Cases: None.

  - **Sticky Note**
    - Details the validation step, API docs link, and returned info.

#### 2.4 Conditional AI Scoring

- **Overview:** Determines if scoring should proceed based on credit availability, skipping scoring if insufficient.
- **Nodes Involved:** `Skip scoring?`, `Score email`, `Not scored`, `Sticky Note8`
- **Node Details:**

  - **Skip scoring?**
    - Type: Switch node
    - Role: Compares available credits with needed items to decide if scoring can proceed.
    - Parameters: Checks length of input array versus credits.
    - Inputs: From `Wait for scoring credits check`.
    - Outputs: "score" branch to `Score email`; "skip" branch to `Not scored`.
    - Edge Cases: Credit count mismatch, empty inputs.

  - **Score email**
    - Type: ZeroBounce node (scoring resource)
    - Role: Scores email addresses using ZeroBounce AI.
    - Parameters: Uses `address` field from merged validation results.
    - Credentials: ZeroBounce API key.
    - Inputs: From `Skip scoring?` "score" branch.
    - Outputs: Scoring result.
    - Edge Cases: API errors, timeout, invalid address.

  - **Not scored**
    - Type: NoOp
    - Role: Endpoint for emails skipped from scoring due to credit shortage.
    - Inputs: From `Skip scoring?` "skip" branch.
    - Outputs: None.
    - Edge Cases: None.

  - **Sticky Note8**
    - Describes logic for scoring or skipping based on credits.

#### 2.5 Scoring and Merge

- **Overview:** Merges scoring results with validation results to enrich email info.
- **Nodes Involved:** `Merge validation and scoring results`
- **Node Details:**

  - **Merge validation and scoring results**
    - Type: Merge node (enrichInput1)
    - Role: Combines validation data and scoring data by matching `address` and `email` fields.
    - Parameters: Join mode "enrichInput1", merging on email/address fields.
    - Inputs: Validation data from `Skip scoring?` and scoring data from `Score email`.
    - Outputs: Combined dataset for filtering.
    - Edge Cases: Missing match keys, partial merges.

#### 2.6 Filtering by Score

- **Overview:** Categorizes emails into three confidence tiers based on AI score.
- **Nodes Involved:** `Filter by score`, `High score`, `Medium score`, `Low score`, `Sticky Note6`, `Sticky Note7`, `Sticky Note5`
- **Node Details:**

  - **Filter by score**
    - Type: Switch node
    - Role: Routes emails based on `score` field into high (≥9), medium (≥3), or low (<3) categories.
    - Inputs: From `Merge validation and scoring results`.
    - Outputs: To respective NoOp nodes per score tier.
    - Edge Cases: Missing or non-numeric scores.

  - **High score**, **Medium score**, **Low score**
    - Type: NoOp
    - Role: Endpoints for categorized email groups.
    - Inputs: From `Filter by score`.
    - Outputs: None.
    - Edge Cases: None.

  - **Sticky Notes 5, 6, 7**
    - Provide descriptions for scoring, categorization, and output usage.

---

### 3. Summary Table

| Node Name                         | Node Type                        | Functional Role                            | Input Node(s)                  | Output Node(s)                        | Sticky Note                                                                                                      |
|----------------------------------|---------------------------------|--------------------------------------------|-------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Starts workflow manually                    | None                          | Sandbox emails                      |                                                                                                                 |
| Sandbox emails                   | Set                             | Provides sandbox email list                  | When clicking ‘Execute workflow’ | Split into items                   | ## 1. Input data ... sandbox emails for testing, split into individual items                                    |
| Split into items                 | SplitOut                        | Splits email array into individual items    | Sandbox emails                | Check credits for validation, Wait for validation credits check |                                                                                                                 |
| Check credits for validation     | ZeroBounce (account)             | Checks validation API credits                | Split into items              | Wait for validation credits check   | ## 2. Check ZeroBounce account credits balance (optional)... prevent issues mid-way                              |
| Wait for validation credits check| Merge                          | Synchronizes credit check and item flow     | Check credits for validation, Split into items | Validate email                  |                                                                                                                 |
| Validate email                  | ZeroBounce (validation)          | Validates each email                         | Wait for validation credits check | Is valid?                       | ## 3. ZeroBounce Validation ... see docs https://www.zerobounce.net/docs/email-validation-api-quickstart/v2-validate-emails |
| Is valid?                       | Switch                         | Routes emails by validation status           | Validate email               | Wait for scoring credits check, Wait |                                                                                                                 |
| Wait                           | Wait                           | Adds delay before scoring                    | Is valid? (valid branch)     | Check credits for scoring           |                                                                                                                 |
| Check credits for scoring        | ZeroBounce (account)             | Checks scoring API credits                    | Wait                         | Wait for scoring credits check      | ## 4. Check ZeroBounce account credits balance (optional)                                                       |
| Wait for scoring credits check  | Merge                          | Synchronizes credit check and validation     | Check credits for scoring, Is valid? | Skip scoring?                    |                                                                                                                 |
| Skip scoring?                   | Switch                         | Decides whether to score emails               | Wait for scoring credits check | Score email, Not scored             | ## 5. Score emails or finish ... allows validation use if scoring unavailable                                    |
| Score email                    | ZeroBounce (scoring)             | Scores email addresses                        | Skip scoring? (score branch) | Merge validation and scoring results | ## 6. ZeroBounce A.I. Scoring ... see https://www.zerobounce.net/docs/ai-scoring-api/single-email-scoring         |
| Not scored                    | NoOp                           | Endpoint for skipped scoring                  | Skip scoring? (skip branch)  | None                              |                                                                                                                 |
| Merge validation and scoring results | Merge                      | Combines validation and scoring data         | Score email, Skip scoring?   | Filter by score                   |                                                                                                                 |
| Filter by score                | Switch                         | Categorizes emails by AI score                | Merge validation and scoring results | High score, Medium score, Low score | ## 7. Split results by score ... refined list for confident use                                                |
| High score                    | NoOp                           | Output for high confidence emails             | Filter by score              | None                              |                                                                                                                 |
| Medium score                  | NoOp                           | Output for medium confidence emails           | Filter by score              | None                              |                                                                                                                 |
| Low score                     | NoOp                           | Output for low confidence emails              | Filter by score              | None                              |                                                                                                                 |
| Not valid                    | NoOp                           | Endpoint for invalid or suspicious emails     | Is valid? (other branch)     | None                              |                                                                                                                 |
| Sticky Note                   | Sticky Note                    | Documentation for ZeroBounce validation       | None                        | None                              | ## 3. ZeroBounce Validation ... https://www.zerobounce.net/docs/email-validation-api-quickstart/v2-validate-emails |
| Sticky Note2                  | Sticky Note                    | Documentation for credit checks                | None                        | None                              | ## 2. Check ZeroBounce account credits balance (optional) ... https://www.zerobounce.net/docs/email-validation-api-quickstart#get_balance__v2__ |
| Sticky Note3                  | Sticky Note                    | Documentation for input data                    | None                        | None                              | ## 1. Input data about sandbox emails and splitting                                                             |
| Sticky Note4                  | Sticky Note                    | Documentation for scoring credits               | None                        | None                              | ## 4. Check ZeroBounce account credits balance (optional)                                                       |
| Sticky Note5                  | Sticky Note                    | Documentation for filtered output branches      | None                        | None                              | ## Filtered outputs ... further follow-up options                                                              |
| Sticky Note6                  | Sticky Note                    | Documentation for score filtering                | None                        | None                              | ## 7. Split results by score                                                                                     |
| Sticky Note7                  | Sticky Note                    | Documentation for AI scoring                      | None                        | None                              | ## 6. ZeroBounce A.I. Scoring ... https://www.zerobounce.net/docs/ai-scoring-api/single-email-scoring            |
| Sticky Note8                  | Sticky Note                    | Documentation for scoring decision logic         | None                        | None                              | ## 5. Score emails or finish ... scoring conditional on credits                                                  |
| Sticky Note9                  | Sticky Note                    | Overview and intro to workflow                     | None                        | None                              | ## ZeroBounce Email Validation and Scoring overview and instructions, API key link                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Execute workflow’` to start the workflow manually.

2. **Add a Set node** named `Sandbox emails`:  
   - Set mode to "raw".  
   - Create a JSON output with an array field `emails` containing 30 sandbox test email addresses (as per the example in the workflow).

3. **Add a SplitOut node** named `Split into items`:  
   - Configure to split the `emails` array into individual items.  
   - Set to include all other fields, destination field name `email`.  
   - Connect `Sandbox emails` → `Split into items`.

4. **Add a ZeroBounce node** named `Check credits for validation`:  
   - Set resource to `account`.  
   - Set `creditsRequired` to `={{ $input.all().length }}` to require one credit per email.  
   - Connect `Split into items` → `Check credits for validation`.  
   - Enable "Execute Once" to run just once.  
   - Configure ZeroBounce credentials with your API key.

5. **Add a Merge node** named `Wait for validation credits check`:  
   - Set mode to `combine` by `combineAll`.  
   - Connect `Check credits for validation` and `Split into items` to inputs 1 and 2 respectively.  
   - This node waits for credit check before validating emails.

6. **Add a ZeroBounce node** named `Validate email`:  
   - Set to validate an email address.  
   - Use expression for `email`: `={{ $json.email }}`.  
   - Set timeout to 10 seconds.  
   - Connect `Wait for validation credits check` → `Validate email`.

7. **Add a Switch node** named `Is valid?`:  
   - Add rule: If `status` equals `"valid"`, output `valid`; else `other`.  
   - Connect `Validate email` → `Is valid?`.

8. **Add a NoOp node** named `Not valid` for invalid emails.  
   - Connect `Is valid?` "other" output → `Not valid`.

9. **Add a Wait node** named `Wait` with default delay (to allow credits update).  
   - Connect `Is valid?` "valid" output → `Wait`.

10. **Add a ZeroBounce node** named `Check credits for scoring`:  
    - Set resource to `account`.  
    - Leave `creditsRequired` empty to check balance without enforcing.  
    - Connect `Wait` → `Check credits for scoring`.  
    - Enable "Execute Once".  
    - Use ZeroBounce API credentials.

11. **Add a Merge node** named `Wait for scoring credits check`:  
    - Mode: `combine` by `combineAll`.  
    - Connect `Check credits for scoring` and `Is valid?` "valid" output to inputs 1 and 2 respectively.

12. **Add a Switch node** named `Skip scoring?`:  
    - Add two outputs:  
      - `score` if number of input items ≤ available credits (`$input.all().length <= $json.Credits`).  
      - `skip` if number of input items > available credits.  
    - Connect `Wait for scoring credits check` → `Skip scoring?`.

13. **Add a ZeroBounce node** named `Score email`:  
    - Set resource to `scoring`.  
    - Use expression for `email`: `={{ $json.address }}` (from merged data).  
    - Connect `Skip scoring?` "score" output → `Score email`.

14. **Add a NoOp node** named `Not scored` for skipped scoring.  
    - Connect `Skip scoring?` "skip" output → `Not scored`.

15. **Add a Merge node** named `Merge validation and scoring results`:  
    - Mode: `enrichInput1`.  
    - Merge by fields: `address` from scoring and `email` from validation.  
    - Connect `Score email` → input 2, and from `Skip scoring?` "score" output (or direct validation output if scoring skipped) → input 1.

16. **Add a Switch node** named `Filter by score`:  
    - Add rules to split by `score`:  
      - `high` if score ≥ 9.  
      - `medium` if score ≥ 3 and < 9.  
      - `low` if score < 3.  
    - Connect `Merge validation and scoring results` → `Filter by score`.

17. **Add three NoOp nodes** named `High score`, `Medium score`, and `Low score` for each category output.  
    - Connect `Filter by score` outputs accordingly.

18. **Add Sticky Note nodes** at relevant positions to document each block and steps as per the original workflow for clarity.

19. **Credentials Setup:**  
    - Configure ZeroBounce API credentials once and assign to all ZeroBounce nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Use [ZeroBounce API key](https://www.zerobounce.net/members/API) to enable API calls for validation and scoring.                                                                                                                                                                                                            | ZeroBounce API documentation and credentials setup                                                                |
| Sandbox emails are provided for testing without consuming credits; replace these with real data sources in production.                                                                                                                                                                                                    | https://www.zerobounce.net/docs/email-validation-api-quickstart/v2-sandbox-mode                                   |
| Validation API docs: [Single Email Validator](https://www.zerobounce.net/docs/email-validation-api-quickstart/v2-validate-emails)                                                                                                                                                                                        | ZeroBounce Email Validation API                                                                                   |
| Scoring API docs: [Single Email Scoring](https://www.zerobounce.net/docs/ai-scoring-api/single-email-scoring)                                                                                                                                                                                                             | ZeroBounce AI Scoring API                                                                                          |
| Credit balance check recommended before running bulk validations or scoring to avoid mid-process failures.                                                                                                                                                                                                                | https://www.zerobounce.net/docs/email-validation-api-quickstart#get_balance__v2__                                 |
| Filtered outputs can feed into further ZeroBounce features like [Email Finder](https://www.zerobounce.net/docs/email-finder-api) or [Domain Search](https://www.zerobounce.net/docs/domain-search-api) using n8n integrations.                                                                                                | ZeroBounce extended API use cases                                                                                  |
| This workflow uses best practices like "Execute Once" nodes for credit checks and wait/merge nodes to synchronize asynchronous API calls and input item streams.                                                                                                                                                          | n8n best practices                                                                                                |
| The scoring step is optional and conditioned on sufficient credits; workflow allows graceful fallback to use only validation results if scoring is skipped.                                                                                                                                                              | Workflow logic detail                                                                                              |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.