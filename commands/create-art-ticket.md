---
description: Create an ART Jira ticket to request a release build from the ART team
allowed-tools: Bash, Read, Write, AskUserQuestion
---

# Create ART Ticket for MTA Release

Create an ART (OpenShift Automated Release Tooling) Jira ticket requesting the ART team to prepare an MTA release build. **Always create the ticket by cloning the template ticket ART-14539** (`[template] Operator release request`) — do NOT create a fresh issue from scratch. Cloning is required so the new ticket inherits the correct components, labels, and custom-field configuration (notably **Activity Type**) that the ART team's tooling expects; a plain create silently omits these.

## Prerequisites

- `JIRA_EMAIL` and `JIRA_API_TOKEN` environment variables must be set
- The operator NVR (Name-Version-Release) for the build

## Step 1 — Gather Inputs

Parse `$ARGUMENTS` for any provided values. Arguments may contain some or all of:
- MTA version (e.g. "8.1.3")
- Operator NVR (e.g. "mta-operator-container-8.1.3-202607271608.p2.gf5b3f83.assembly.stream.el9")

For anything not provided, ask the user using `AskUserQuestion`. You need:

| Input | Example | Required |
|-------|---------|----------|
| **MTA Version** | `8.1.3` | Yes |
| **Operator NVR** | `mta-operator-container-8.1.3-202607271608.p2.gf5b3f83.assembly.stream.el9` | Yes |
| **Target release date** | `2026-08-04` | Yes — ART needs 5 working days to prepare |
| **OCP version range** | `v4.12` through `v4.21` | Yes (default: v4.12-v4.21) |
| **JIRA issue filter** | Bugs + CVEs only | Yes (default: Bugs + Vulnerabilities from fixVersion) |

### Deriving the GROUP

The GROUP is the ocp-build-data branch, derived from the version's major.minor:
- Version `8.1.3` → Group `mta-8.1`
- Version `8.2.0` → Group `mta-8.2`

### Deriving the ASSEMBLY

The ASSEMBLY is the full version string (e.g. `8.1.3`).

## Step 2 — Query JIRA Issues

Query JIRA for tickets under the matching fixVersion using the new search API:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  "https://redhat.atlassian.net/rest/api/3/search/jql" \
  -d '{"jql": "project = MTA AND fixVersion = \"MTA {VERSION}\" ORDER BY key ASC", "maxResults": 100, "fields": ["summary", "status", "issuetype"]}'
```

**Note:** The old `/rest/api/3/search` GET endpoint has been removed. Always use `POST /rest/api/3/search/jql`.

Filter the results based on user preference:
- **Bugs + CVEs only** (default): Include only issues with `issuetype.name` of `Bug` or `Vulnerability`
- **All tickets**: Include everything under the fixVersion
- **Custom**: Let the user specify which tickets

Display the filtered list to the user for confirmation before proceeding.

## Step 3 — Build FBC Pull Specs

Generate one pull spec per OCP version using this template:

```
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:{OCP_VERSION}__operator_nvr__{NVR}
```

For example, with OCP v4.12-v4.21 and NVR `mta-operator-container-8.1.3-...`:
```
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v4.12__operator_nvr__mta-operator-container-8.1.3-...
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v4.13__operator_nvr__mta-operator-container-8.1.3-...
...
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v4.21__operator_nvr__mta-operator-container-8.1.3-...
```

## Step 4 — Clone the Template Ticket (ART-14539)

**Do not build the ticket from scratch.** Clone ART-14539 so the release ticket inherits the template's configuration. Because the Jira Cloud REST API has no native "clone" endpoint, cloning is done in three parts: fetch the template, create a new issue copying the template's create-settable fields, then edit the new issue to apply the fields that can only be set after creation (including the **due date**).

### 4a. Fetch the template

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Accept: application/json" \
  "https://redhat.atlassian.net/rest/api/3/issue/ART-14539" > /tmp/art_template.json
```

### 4b. Generate the create + update payloads

Run a Python script that reads `/tmp/art_template.json` and the gathered inputs, and writes two files:

- `/tmp/art_payload.json` — the CREATE payload (fields settable at create time)
- `/tmp/art_update.json` — the EDIT payload (fields that are NOT on the create screen)

Fields **copied from the template** into CREATE: `project`, `issuetype`, `labels`, `components`, and `customfield_10464` (**Activity Type** — the field a plain create would miss).
Fields **overridden** in CREATE: `summary` and `description` (built below).

Fields in the EDIT payload — these are editable but NOT settable on create, so they require a follow-up PUT:
- `duedate` — **always set this** to the target release date (`YYYY-MM-DD`). The create screen has no due-date field, so it MUST be set here.
- Copied from the template: `customfield_10483` (Blocked Reason), `customfield_10484` (Ready), `customfield_10517` (Blocked), `customfield_10712` (Color Status).

Normalize select/option custom fields to `{"id": "..."}` when copying; copy rich-text (ADF) custom fields verbatim. Example generator:

```python
import json

tpl = json.load(open("/tmp/art_template.json"))["fields"]

VERSION = "8.1.3"                    # from inputs
SUMMARY_DATE = "2026-AUG-04"        # YYYY-MMM-DD, uppercase month
DUE_DATE = "2026-08-04"             # YYYY-MM-DD, the target release date — ALWAYS set

def opt(v):
    # select/option fields -> {"id": ...}; ADF/text -> verbatim
    if isinstance(v, dict) and "id" in v and v.get("type") != "doc":
        return {"id": v["id"]}
    return v

description = { ... }  # ADF built per "Description format" below

create = {
    "fields": {
        "project": {"key": tpl["project"]["key"]},
        "issuetype": {"id": tpl["issuetype"]["id"]},
        "labels": tpl.get("labels", []),
        "components": [{"id": c["id"]} for c in (tpl.get("components") or [])],
        "customfield_10464": opt(tpl.get("customfield_10464")),  # Activity Type
        "summary": f"Release MTA {VERSION} [{SUMMARY_DATE}]",
        "description": description,
    }
}
json.dump(create, open("/tmp/art_payload.json", "w"), indent=2)

update_fields = {"duedate": DUE_DATE}  # always set the due date
for cf in ("customfield_10483", "customfield_10484", "customfield_10517", "customfield_10712"):
    if tpl.get(cf) is not None:
        update_fields[cf] = opt(tpl[cf])
json.dump({"fields": update_fields}, open("/tmp/art_update.json", "w"), indent=2)
```

**Summary date format:** `YYYY-MMM-DD` in uppercase month abbreviation (e.g. `2026-AUG-04`).

### 4c. Create the ticket

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -X POST -d @/tmp/art_payload.json \
  "https://redhat.atlassian.net/rest/api/3/issue"
```

Capture the returned issue key (e.g. `ART-XXXXX`).

### 4d. Apply the due date + template-only fields (always)

Using the key from 4c, PUT the edit payload. This step is **not optional** — it is how the due date gets set.

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -X PUT -d @/tmp/art_update.json \
  "https://redhat.atlassian.net/rest/api/3/issue/{TICKET_KEY}"
```

A successful PUT returns HTTP 204 with no body. Verify by re-fetching the issue and confirming `fields.duedate` matches the target release date.

**Important:** Use `curl` (not Python's urllib) for the create/update calls due to potential SSL certificate issues with Python on macOS. Generate the JSON payloads with Python (above), then submit with curl.

### Description format (Atlassian Document Format)

The description follows this exact ADF structure:

1. **GROUP:** (bold) paragraph
   - Bullet: "Group is your branch in ocp-build-data."
   - Code block: the group string (e.g. `mta-8.1`)

2. **ASSEMBLY:** (bold) paragraph
   - Bullet: "Assembly is your release version for the group."
   - Code block: the version string (e.g. `8.1.3`)

3. Empty paragraph

4. **Due date :** (bold) + " ART needs 5 working days to prepare release"

5. **Build Information :** (bold) + " Details of the build which will be released:"
   - Bullet: "FBC build pull specs, for each target OCP version is required"
   - Code block: all FBC pull specs, one per line, trailing newline

6. Non-breaking space paragraph

7. **JIRA issues :** (bold) + " List of JIRA issues which will be shipped with the release"
   - Bullet: "Jira issues "
   - Code block: all issue keys, one per line, trailing newline

8. Non-breaking space paragraph

9. Disclaimer paragraph: "Please note that ART will **not** verify bug fixes. We will proceed to prep the release with the builds and bugs that are present on the ticket."

The description ADF is built in the generator script in Step 4b and submitted as part of the create payload.

## Step 5 — Report Results

On success:
1. Print the ticket URL: `https://redhat.atlassian.net/browse/{TICKET_KEY}`
2. Confirm the ticket was **cloned from ART-14539** and that the **due date** was set (re-fetch and show `fields.duedate`)
3. Summarize: group, assembly, number of FBC pull specs, number of JIRA issues included
4. Note that the ticket is unassigned — the ART team will pick it up

On failure:
1. Print the error response
2. If the CREATE (4c) failed, save the payload to `/tmp/art_payload.json` for debugging
3. If the CREATE succeeded but the EDIT (4d) failed, the ticket exists **without a due date** — report the key and re-run the PUT from `/tmp/art_update.json`

## Reference

- **Template ticket (always cloned): ART-14539** — `[template] Operator release request`. Carries labels (`art:release:schedule`), components (`daily-ops`, `Release work`), and custom fields — most importantly **Activity Type** (`customfield_10464` = "Incidents & Support") — that a from-scratch create would omit.
- Prior tickets: ART-18736 (MTA 8.1.2), ART-21798 (MTA 8.1.3)
- Jira project: ART ("OpenShift Automated Release Tooling")
- The ART team typically self-assigns from the `art:release:schedule` label
- ART needs 5 working days lead time to prepare a release
- **`duedate` is not on the ART create screen** — it must be set via a follow-up PUT (Step 4d). The skill always sets it to the target release date.
- Edit-only template fields copied during clone: `customfield_10483` (Blocked Reason), `customfield_10484` (Ready), `customfield_10517` (Blocked), `customfield_10712` (Color Status).
