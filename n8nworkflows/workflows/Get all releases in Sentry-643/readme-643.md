Get all releases in Sentry

https://n8nworkflows.xyz/workflows/get-all-releases-in-sentry-643


# Get all releases in Sentry

### 1. Workflow Overview

This workflow automates interaction with the Sentry.io API to first create a new release and then retrieve all releases for a specified organization and project. It is designed for users who want to programmatically manage releases within Sentry, enabling integration into CI/CD pipelines or release management processes.

The workflow is logically divided into two main blocks:

- **1.1 Trigger and Release Creation:** Starts the workflow manually and creates a new release in Sentry.
- **1.2 Retrieve All Releases:** After creating the release, fetches all releases associated with the specified organization and project.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Release Creation

**Overview:**  
This block initiates the workflow manually and creates a new release in Sentry using the configured organization and project details.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Sentry.io (Release Creation)

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow manually.  
  - *Configuration:* No parameters; triggers the workflow on user action.  
  - *Input:* None  
  - *Output:* Triggers the next node (Sentry.io release creation).  
  - *Edge Cases:* None typical; user must manually trigger.  
  - *Version:* n8n standard manual trigger node.

- **Sentry.io (Release Creation)**  
  - *Type:* Sentry.io API node  
  - *Role:* Creates a new release in Sentry.  
  - *Configuration:*  
    - Resource: `release`  
    - Operation: `create`  
    - Organization Slug: (empty, must be set by user)  
    - Projects: (empty array, must be set by user)  
    - Additional Fields: none configured  
    - URL: empty (defaults to Sentry API endpoint)  
  - *Credentials:* Uses `sentryIoApi` credential named "sentry" (OAuth2 or API token).  
  - *Input:* Triggered by manual trigger node.  
  - *Output:* Passes created release data to next node.  
  - *Edge Cases:*  
    - Missing or incorrect organization slug or projects will cause API errors.  
    - Authentication failures if credentials are invalid.  
    - API rate limits or network timeouts.  
  - *Version:* Uses version 0.0.1 of the Sentry.io node.

#### 1.2 Retrieve All Releases

**Overview:**  
This block fetches all releases from Sentry after the new release is created, allowing the user to see the full list of releases.

**Nodes Involved:**  
- Sentry.io1 (Get All Releases)

**Node Details:**

- **Sentry.io1 (Get All Releases)**  
  - *Type:* Sentry.io API node  
  - *Role:* Retrieves all releases for the specified organization.  
  - *Configuration:*  
    - Resource: `release`  
    - Operation: `getAll`  
    - Return All: `true` (fetches all releases without pagination)  
    - Organization Slug: (empty, must be set by user)  
    - Additional Fields: none configured  
  - *Credentials:* Uses the same `sentryIoApi` credential named "sentry".  
  - *Input:* Receives output from the release creation node.  
  - *Output:* Outputs the list of all releases.  
  - *Edge Cases:*  
    - Missing organization slug causes API errors.  
    - Authentication or permission errors if credentials lack access.  
    - Large data sets might cause performance issues if not handled properly.  
  - *Version:* Uses version 0.0.1 of the Sentry.io node.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role               | Input Node(s)          | Output Node(s)       | Sticky Note                         |
|---------------------|---------------------|------------------------------|------------------------|----------------------|-----------------------------------|
| On clicking 'execute'| Manual Trigger      | Starts the workflow manually | None                   | Sentry.io            |                                   |
| Sentry.io           | Sentry.io API       | Creates a new release        | On clicking 'execute'  | Sentry.io1           |                                   |
| Sentry.io1          | Sentry.io API       | Retrieves all releases       | Sentry.io              | None                 |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `On clicking 'execute'`.  
   - No parameters needed. This node will start the workflow manually.

2. **Add Sentry.io Node for Release Creation**  
   - Add a **Sentry.io** node named `Sentry.io`.  
   - Set **Resource** to `release`.  
   - Set **Operation** to `create`.  
   - Configure the **Organization Slug** with your Sentry organization identifier.  
   - Set **Projects** to an array containing your project slug(s).  
   - Leave **Additional Fields** empty unless specific release metadata is needed.  
   - Leave **URL** empty to use the default Sentry API endpoint.  
   - Under **Credentials**, select or create a credential of type `sentryIoApi` with your Sentry API token or OAuth2 credentials.  
   - Connect the output of `On clicking 'execute'` to the input of this node.

3. **Add Sentry.io Node for Getting All Releases**  
   - Add another **Sentry.io** node named `Sentry.io1`.  
   - Set **Resource** to `release`.  
   - Set **Operation** to `getAll`.  
   - Enable **Return All** to fetch all releases without pagination.  
   - Set the **Organization Slug** to the same value used in the previous node.  
   - Leave **Additional Fields** empty.  
   - Use the same `sentryIoApi` credential as before.  
   - Connect the output of the `Sentry.io` node (release creation) to the input of this node.

4. **Save and Activate Workflow**  
   - Ensure all nodes are properly connected and configured.  
   - Save the workflow.  
   - Execute manually by clicking the trigger node to test.

---

This completes the detailed analysis and reproduction instructions for the "Get all releases in Sentry" workflow.