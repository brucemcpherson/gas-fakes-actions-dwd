# Automating Google Apps Script with `gas-fakes` and GitHub Actions

This project demonstrates how to run complex Google Apps Script (GAS) logic—like identifying duplicate files across a Google Drive—directly from a GitHub Actions runner. By using the `@mcpher/gas-fakes` library, we can execute GAS code in a Node.js environment without actually deploying to the Apps Script editor.

See [gas-fakes](https://github.com/brucemcpherson/gas-fakes) for more information about gas-fakes and getting started. Workload identity federation (WIF) relies on domain-wide delegation (DWD) to authenticate to Google workspace, so you should authenticate with gas-fakes and use the provided setup-wif.sh to initialize the WIF configuration for your project.

## The Architecture

### 1. The Core Logic (`example.js`)
The script uses standard GAS services (`DriveApp`, `Drive`, `ScriptApp`) to:
- **Map Folders:** Recursively build a map of all folders owned by the user.
- **Handle Edge Cases:** Gracefully handles special top-level entities like "My Mac" or other Computer sync folders that lack a traditional parent.
- **Find Duplicates:** Scans files using MD5 checksums to identify duplicate content.
- **Reporting:** Writes the results back to a Google Sheet using the `bmPreFiddler` library.

### 2. Authentication: WIF + DWD
To run securely in GitHub Actions without managing long-lived "Service Account Keys" (JSON files), we use **Workload Identity Federation (WIF)**:
- **Identity Pool:** GitHub is configured as a trusted OIDC provider for Google Cloud.
- **Service Account:** A Google Service Account is granted permission to be impersonated by the specific GitHub repository.
- **Domain-Wide Delegation (DWD):** The Service Account is authorized at the Workspace Admin level to "impersonate" a specific user (the `GOOGLE_WORKSPACE_SUBJECT`), allowing it to see that user's Drive files as if it *were* that user.

### 3. The GitHub Workflow (`gas-fakes.yml`)
The workflow is designed for **On-Demand** execution via the `workflow_dispatch` trigger.

#### Key Components:
- **Environment Parity:** The workflow maps `GF_` (gas-fakes) environment variables from GitHub Repository Variables. This ensures that the runner knows exactly which script ID and manifest (`appsscript.json`) to use, enabling it to load the correct Apps Script libraries (like `bmPreFiddler`) at runtime.
- **Modern Runtime:** The workflow uses the latest versions of GitHub Actions (`actions/checkout@v6`, `auth@v3`) to ensure compatibility with the **Node.js 24** internal runner requirements, while executing your actual logic on **Node.js 22**.
- **Dynamic Scopes:** Rather than hardcoding permissions in the YAML, the required OAuth scopes are dynamically injected from environment variables, keeping the configuration flexible and secure.

## How to Run

You can trigger the process manually using the GitHub CLI:

```bash
# Trigger the workflow
gh workflow run gas-fakes.yml

# Watch the execution in real-time
gh run watch
```

## Summary of Recent Fixes
- **CWD Resolution:** Updated the `npm run containerrun` script to change directory into `sources/`. This ensures the `.env` and `appsscript.json` files are correctly located by the `gas-fakes` worker.
- **Folder Parent Logic:** Fixed a crash when the script encountered "Computer" root folders on Drive that don't have parent IDs.
- **Library Resolution:** Ensured `GF_SCRIPT_ID` is passed to the runner so that external Apps Script libraries are correctly resolved and loaded.
