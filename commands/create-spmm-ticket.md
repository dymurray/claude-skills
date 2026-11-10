---
description: Create an SPMM Jira ticket to declare MTA binaries for GA promotion
allowed-tools: Bash, Read, Write, AskUserQuestion
---

# Create SPMM Ticket for MTA GA Release

Create an SPMM Jira ticket declaring MTA binaries for GA promotion, following the established format from prior releases (e.g. SPMM-22733, SPMM-23184).

## Prerequisites

- `JIRA_EMAIL` and `JIRA_API_TOKEN` environment variables must be set
- The upload directory must contain the binary `.zip` files and `.MD5SUM` files
- Jira API access to `redhat.atlassian.net`

## Step 1 — Gather Inputs

Parse `$ARGUMENTS` for any provided values. The arguments may contain some or all of:
- MTA version (e.g. "8.2.0")
- Upload directory path (where the binaries live)
- Base download URL

For anything not provided in arguments, ask the user using `AskUserQuestion`. You need all of:

| Input | Example | Required |
|-------|---------|----------|
| **MTA Version** | `8.3.0` | Yes |
| **Upload directory** | `~/upload` | Yes |
| **Base download URL** | `https://download.devel.redhat.com/devel/candidates/middleware/migrationtoolkit/MTA-8.3.0.GA/` | Yes |
| **QE Signoff doc URL** | Google Docs link | Yes |
| **Security advisories included** | No | Yes (default: No) |
| **Due date** | `2026-08-15` | Yes |

## Step 2 — Scan Upload Directory

List the upload directory and categorize all `.zip` files (ignore `metadata.json`, `.DS_Store`, and `.MD5SUM` files).

### Auto-detect binary components

Group files by component. Known components and their description patterns:

| Component | Filename pattern | Description pattern (platform) | Description pattern (source) |
|-----------|-----------------|-------------------------------|------------------------------|
| `cli` | `mta-{VERSION}-cli-{os}-{arch}.zip` | `Migration Toolkit for Applications {VERSION} CLI ({OS} {arch})` | `Migration Toolkit for Applications CLI {VERSION} Source Code` |
| `analyzer-rpc` | `mta-{VERSION}-analyzer-rpc-{os}-{arch}.zip` | `Migration Toolkit for Applications {VERSION} Analyzer RPC ({OS} {arch})` | `Migration Toolkit for Applications Analyzer RPC {VERSION} Source Code` |
| `ops` | `mta-ops-{VERSION}-{os}-{arch}.zip` | `Migration Toolkit for Applications Ops {VERSION} - ({OS} {arch})` | `Migration Toolkit for Applications Ops {VERSION} Source Code` |

**OS name mapping for descriptions:**
- `linux` → `Linux`
- `darwin` → `macOS`
- `windows` → `Windows`

**If a new/unknown component is found** (filename doesn't match any known pattern), ask the user for its short description format using `AskUserQuestion`.

Read the `.MD5SUM` files for reference but note that MD5 checksums are NOT included in the SPMM ticket description — they are separate artifacts.

## Step 3 — Present Summary for Approval

Before creating the ticket, show the user a complete summary:

1. Ticket fields (summary, project, labels, priority, due date)
2. Header fields (TARGET DATE, PRODUCT, VERSION, QE SIGNOFF doc, SECURITY ADVISORIES, PUSH TO STAGING, CUSTOMER PORTAL NEW VERSION)
3. Complete file list with numbering, filenames, short descriptions, and locations
4. Total file count

Ask the user to confirm before proceeding.

## Step 4 — Build and Submit the Ticket

### File ordering convention

Order files by component group, and within each group: linux-amd64, darwin-amd64, windows-amd64, linux-arm64, darwin-arm64, windows-arm64, then src.

### Location field

- All platform binaries: `Developer Portal, CSP`
- The very last file in the ticket (typically the last source zip): `Developer Portal, CSP, FTP`

### Ticket fields

```json
{
  "project": {"key": "SPMM"},
  "summary": "Migration Toolkit for Applications (MTA) {VERSION} GA Release",
  "issuetype": {"name": "Task"},
  "priority": {"name": "Major"},
  "labels": ["content-gateway", "csp", "ftp", "mta-8"]
}
```

**Note:** The `duedate` field CANNOT be set via the Jira REST API for SPMM tasks. It must be set manually in the Jira UI after creation.

### Description format (Atlassian Document Format)

The description is structured as ADF JSON with these sections:

**Header paragraphs** (each is a paragraph node with field label + hardBreak + value):
- TARGET DATE: {due_date}
- PRODUCT(S): Migration Toolkit for Applications
- VERSION: {version}
- QE SIGNOFF doc: (rendered as an inlineCard with the Google Docs URL)
- SECURITY ADVISORIES INCLUDED: No/Yes
- PUSH TO STAGING: Yes
- CUSTOMER PORTAL NEW VERSION (FOR CASES): {version}

**File entries** (repeated for each file):
```
FILE {N}

ORIG LINK:
{base_url}/{filename}    ← base_url and filename are both rendered as linked text

FINAL FILE NAME:
{filename}

SHORT DESCRIPTION:
{description}

LOCATION:
{location}
```

Each of these fields is a separate ADF paragraph node. The ORIG LINK paragraph contains:
1. Text "ORIG LINK:"
2. hardBreak
3. Linked text of the base URL (href = base URL)
4. Text "/"
5. Linked text of the filename (href = full URL to the file)

### API call

Use `curl` to POST to `https://redhat.atlassian.net/rest/api/3/issue` with basic auth from `$JIRA_EMAIL:$JIRA_API_TOKEN`.

**Important:** Use `curl` (not Python's urllib) because the system may have SSL certificate issues with Python.

Generate the JSON payload with a Python script that builds the ADF structure, writes it to a temp file, then use curl to submit it:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -X POST \
  -d @/tmp/spmm_payload.json \
  "https://redhat.atlassian.net/rest/api/3/issue"
```

## Step 5 — Report Results

On success:
1. Print the ticket URL: `https://redhat.atlassian.net/browse/{TICKET_KEY}`
2. Remind the user to manually set the due date in the Jira UI (since the API doesn't support it for SPMM)
3. Print total number of binary files declared

On failure:
1. Print the error response
2. Save the generated payload to `/tmp/spmm_payload.json` for debugging

## Reference

- Prior tickets: SPMM-22733 (MTA 8.1.2), SPMM-23184 (MTA 8.2.0)
- Jira project: SPMM ("SP Middleware and Management")
- Assignee is typically left unassigned at creation; the SPMM team (Christopher O'Brien / cobrien@redhat.com) picks it up
