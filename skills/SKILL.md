---
name: trigger-art-release
description: Trigger an MTA downstream release by running the ART self-service OpenShift Pipelines (Tekton) in the art-mta-tenant namespace — the release-from-fbc pipeline (stage) then trigger-prod-release (prod). Derives group/assembly and the per-OCP FBC pull specs from the operator NVR, and gathers the release's Jira bugs, to fill the pipeline parameters. Replaces the old ART Jira ticket (clone ART-14539) flow.
allowed-tools: Bash, Read, AskUserQuestion
---

# Trigger an MTA Release via the ART Tekton Pipelines

Trigger an MTA downstream release by running the **ART self-service OpenShift Pipelines (Tekton)** in
the team namespace `art-mta-tenant`. This is **step 9** of the release workflow — see `release_tickets`
and `software_lifecycle` for where it fits.

The release runs in two pipelines:

1. **`release-from-fbc`** (stage) — creates a Shipment MR on GitLab
   `hybrid-platforms/art/ocp-shipment-data` and auto-triggers the **stage** release.
2. **`trigger-prod-release`** (prod) — performs the **production** release and merges the Shipment MR.
   Its `dry-run` param defaults to `true`; you review the dry run, then re-run with `dry-run=false`.

> **The Shipment MR often finishes the release on its own.** Once the stage pipeline opens the MR,
> the MR's own GitLab CI (art-bot-jenkins) may auto-drive stage → prod → **merge** without you
> running `trigger-prod-release` at all. **Always check the MR state before running the prod step**
> (see Step 5) — if it is already `merged` with a `prod-release-success` label, the release is done
> and running `trigger-prod-release` will fail (its 404-on-a-shipment-file symptom just means the
> source branch was deleted on merge).

> This replaces the old process of filing (cloning) an ART Jira ticket (`ART-14539`) and waiting for
> the ART team to run a manual build. You now run the release yourself in the MTA namespace.

## Prerequisites

- **Logged in to the ART cluster** `artc2023` with access to the `art-mta-tenant` namespace. Verify:
  ```bash
  oc whoami                       # confirms you are logged in
  oc project art-mta-tenant       # switch to the MTA release namespace
  ```
  > **Cluster-context gotcha.** The release pipelines run on **artc2023**, but the default `oc`
  > context is usually the **Konflux** cluster (used for build/ITS checks). Select the ART context
  > before every `tkn ... -n art-mta-tenant` call, since the shell may reset between commands:
  > `oc config use-context "art-mta-tenant/api-artc2023-pc3z-p1-openshiftapps-com:6443/<user>"`.
  > If `oc whoami` returns `Unauthorized` (the token expires mid-session) or `tkn` reports the
  > pipeline "does not exist" / asks for credentials, re-login:
  > `oc login https://api.artc2023.pc3z.p1.openshiftapps.com:6443 --web`.
- **`tkn` (Tekton CLI) installed.** Verify: `tkn version`.
- **`glab` (GitLab CLI) authenticated** to `gitlab.cee.redhat.com` — for checking the Shipment MR
  state (Step 5a) and pulling advisory URLs (Step 6).
- `JIRA_EMAIL` and `JIRA_API_TOKEN` environment variables set (for the Jira bug query in Step 2).
- The **operator NVR** for the build — comes from the operator bundle metadata build in ART/Konflux
  (see `software_lifecycle` step 4).

## Step 1 — Gather Inputs

Parse `$ARGUMENTS` for any provided values, then ask the user (via `AskUserQuestion`) for anything
missing. You need:

| Input | Example | Required |
|-------|---------|----------|
| **Operator NVR** | `mta-operator-container-8.3.0-202607271608.p2.gf5b3f83.assembly.stream.el9` | Yes |
| **OCP versions to release** | `v4.18` … `v5.0` (for 8.3) | Yes (default depends on the MTA minor — see the table below) |
| **Target release date** | `2026-08-04` | Yes (`YYYY-MM-DD`) |
| **Extra image NVRs** | _(none)_ | No (default: empty) |

### Derive the version, GROUP, and ASSEMBLY (confirm with the user)

The operator NVR embeds the release version. Extract the `x.y.z` that follows
`mta-operator-container-` (e.g. NVR `mta-operator-container-8.3.0-...` → version `8.3.0`), then:

- **VERSION** = the full `<major.minor.patch>` (e.g. `8.3.0`) — used for the Jira query and `assembly`.
- **`group`** = `mta-<major.minor>` (the ocp-build-data branch) — e.g. version `8.3.0` → `mta-8.3`.
- **`assembly`** = the full version string — e.g. `8.3.0`.

Show the derived VERSION / `group` / `assembly` and let the user correct them before proceeding.

### Build the FBC pull specs (`fbc-pullspecs`)

The OCP versions shipped **depend on the MTA minor version** (see `release_schedule`):

| MTA version | Default OCP versions |
|-------------|----------------------|
| 8.1 | v4.12, v4.13, …, v4.21 |
| 8.2 | v4.12, v4.13, …, v4.22 |
| 8.3 | v4.18, v4.19, v4.20, v4.21, v4.22, v5.0 |

Pick the default set from the derived MTA minor, then **confirm the full list with the user** (they
may add/remove versions). Note the succession: OCP increments by minor within 4.x, and **v5.0 is the
successor to v4.22** (there is no v4.23) — so a range ending at v5.0 expands to `…, v4.21, v4.22, v5.0`.
For an MTA minor not in the table, ask the user for the range explicitly.

Generate one pull spec per OCP version using this pattern:

```
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:{OCP_VERSION}__operator_nvr__{NVR}
```

For example, MTA 8.3 (OCP v4.18–v5.0) with NVR `mta-operator-container-8.3.0-...`:

```
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v4.18__operator_nvr__mta-operator-container-8.3.0-...
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v4.19__operator_nvr__mta-operator-container-8.3.0-...
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v4.20__operator_nvr__mta-operator-container-8.3.0-...
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v4.21__operator_nvr__mta-operator-container-8.3.0-...
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v4.22__operator_nvr__mta-operator-container-8.3.0-...
quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:v5.0__operator_nvr__mta-operator-container-8.3.0-...
```

Join the pull specs into a single **comma-separated** string for the `fbc-pullspecs` param.

## Step 2 — Gather the Jira Bugs (`jira-bugs`)

Query Jira for the tickets under the matching fixVersion using the search API:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  "https://redhat.atlassian.net/rest/api/3/search/jql" \
  -d '{"jql": "project = MTA AND fixVersion = \"MTA {VERSION}\" ORDER BY key ASC", "maxResults": 100, "fields": ["summary", "status", "issuetype"]}'
```

**Note:** The old `/rest/api/3/search` GET endpoint has been removed. Always use
`POST /rest/api/3/search/jql`.

Filter the results (default matches the previous ART ticket behavior):

- **Bugs + CVEs only** (default): include only issues whose `issuetype.name` is `Bug` or `Vulnerability`.
- **All tickets**: include everything under the fixVersion.
- **Custom**: let the user specify which tickets.

Display the filtered list for confirmation, then join the issue keys into a single **comma-separated**
string for the `jira-bugs` param (e.g. `MTA-1234,MTA-1240,MTA-1255`).

> **Delimiter matters — use commas, no spaces.** Elliott splits `jira-bugs` on commas. A
> space-separated value is treated as a *single* issue ID and the run fails at the bug-processing
> step with `JiraError HTTP 404 .../rest/api/2/issue/MTA-1234%20MTA-1240%20...` (before any MR is
> created, so nothing is shipped — just re-run with commas).

## Step 3 — Preflight

Confirm access and that the pipelines exist in the namespace, then echo the assembled parameters for a
final review before launching:

```bash
oc project art-mta-tenant
tkn pipeline list -n art-mta-tenant     # expect release-from-fbc and trigger-prod-release
```

Print the values that will be passed: `group`, `assembly`, `fbc-pullspecs` (count + list),
`jira-bugs` (count + list), `target-release-date`, `extra-image-nvrs`.

## Step 4 — Stage Release (`release-from-fbc`)

Start the stage pipeline. All parameters come from Steps 1–2:

```bash
tkn pipeline start release-from-fbc --use-param-defaults \
  -p group="mta-8.3" \
  -p assembly="8.3.0" \
  -p fbc-pullspecs="<comma-separated FBC pull specs>" \
  -p jira-bugs="<comma-separated MTA-#### keys>" \
  -p target-release-date="2026-08-04" \
  -p extra-image-nvrs="" \
  -n art-mta-tenant
```

Follow the run to completion:

```bash
tkn pipelinerun logs -f -n art-mta-tenant      # tails the latest run; pick the run if prompted
tkn pipelinerun list -n art-mta-tenant         # to see status / the run name
```

On success this pipeline:

- Creates a **Shipment MR** on GitLab `hybrid-platforms/art/ocp-shipment-data`.
- **Auto-triggers the stage release** from that MR.

Report the PipelineRun name and the Shipment MR URL, and confirm the stage release started.

## Step 5 — Production Release (`trigger-prod-release`) — guided & gated

The prod pipeline is **idempotent** (safe to re-run) and its `dry-run` param **defaults to `true`**.
It takes a required **`mr-url`** param (the full Shipment MR URL from Step 4). Run it in two passes
with a confirmation gate between them.

### 5a. Check whether prod already shipped

The MR's own CI may have already driven stage → prod → merge (see the note at the top). Check first:

```bash
glab api "projects/hybrid-platforms%2Fart%2Focp-shipment-data/merge_requests/<MR#>" \
  | jq -r '{state, labels}'
```

If it is `merged` with a `prod-release-success` label, the release is **done** — skip to Step 6 and
report; do **not** run `trigger-prod-release`.

### 5b. Dry run (default)

`mr-url` has no default, so pass it explicitly (`--use-param-defaults` still supplies `dry-run=true`):

```bash
tkn pipeline start trigger-prod-release --use-param-defaults \
  -p mr-url="<Shipment MR URL from Step 4>" \
  -n art-mta-tenant
tkn pipelinerun logs -f -n art-mta-tenant
```

Surface the dry-run output to the user (what it *would* release). **Do not proceed automatically.**

### 5c. Confirmation gate

Use `AskUserQuestion` to confirm the real production release. Only continue on an explicit **yes**.

### 5d. Real production release

```bash
tkn pipeline start trigger-prod-release --use-param-defaults \
  -p mr-url="<Shipment MR URL from Step 4>" \
  -p dry-run=false \
  -n art-mta-tenant
tkn pipelinerun logs -f -n art-mta-tenant
```

On success this performs the production release and **merges the Shipment MR**. If a run fails
partway, it is safe to re-run (idempotent) — it resumes/retries and, on completion, merges the MR.

## Step 6 — Report Results

Summarize:

1. The `release-from-fbc` PipelineRun name and status, and the **Shipment MR URL**.
2. How prod completed — either the `trigger-prod-release` dry-run/real-run PipelineRun names and
   statuses, **or** (the common case) that the MR's own CI auto-drove it: report the MR `state`
   (`merged`) and `prod-release-success` label from the Step 5a check.
3. Console links for the runs (see the URL pattern in Reference).
4. The **advisory URLs** (stage + prod RHSA/RHBA), pulled from the image shipment file on the merged
   branch:
   ```bash
   glab api "projects/hybrid-platforms%2Fart%2Focp-shipment-data/repository/files/\
   shipment%2Fmta%2Fmta-<M.m>%2Fmta-<M-m>%2Fprod%2F<version>.image.<ts>.yaml/raw?ref=main" \
     | grep -iE "releasePlan|advisory|url:"
   ```
5. A one-line summary: `group`, `assembly`, number of FBC pull specs, number of Jira bugs, target date.

On failure, print the failing PipelineRun name and the relevant `tkn pipelinerun logs` output, and note
that `trigger-prod-release` can be safely re-run. Remember a 404 fetching a shipment file at
`ref=prepare-shipment-...` usually means the MR already merged (branch deleted) — check the MR state
before retrying.

## Reference

- **Cluster:** `artc2023` — **Namespace:** `art-mta-tenant`.
- **Pipelines:** `release-from-fbc` (stage), `trigger-prod-release` (prod).
- **CLI form:** `tkn pipeline start <pipeline> --use-param-defaults -p <key>=<value> -n art-mta-tenant`.
- **Console pipeline URL:**
  `https://console-openshift-console.apps.artc2023.pc3z.p1.openshiftapps.com/k8s/ns/art-mta-tenant/tekton.dev~v1~Pipeline/<pipeline>`
- **Shipment data repo:** GitLab `hybrid-platforms/art/ocp-shipment-data` (the stage pipeline opens the MR there).

### `release-from-fbc` parameters

| Param | Source | Format |
|-------|--------|--------|
| `group` | `mta-<major.minor>` derived from the operator NVR | e.g. `mta-8.3` |
| `assembly` | full `<major.minor.patch>` from the NVR | e.g. `8.3.0` |
| `fbc-pullspecs` | one FBC pull spec per released OCP version | **comma-separated** |
| `jira-bugs` | Bugs + Vulnerabilities under the release fixVersion | **comma-separated** keys (no spaces) |
| `target-release-date` | user input | `YYYY-MM-DD` |
| `extra-image-nvrs` | optional extra (non-FBC) image NVRs | default empty |

### `trigger-prod-release` parameters

| Param | Notes |
|-------|-------|
| `mr-url` | **Required, no default.** Full GitLab Shipment MR URL from Step 4. |
| `dry-run` | defaults `true`; set `-p dry-run=false` for the real release. Idempotent; merges the Shipment MR on success. (Hyphen, not `dry_run`.) |

- **GROUP** is your branch in ocp-build-data; **ASSEMBLY** is the release version for the group.
- OCP range is **per MTA minor** — 8.1: v4.12–v4.21, 8.2: v4.12–v4.22, 8.3: v4.18–v5.0 (see `release_schedule`).
- Prior ART tickets (historical, pre-Tekton): ART-18736 (MTA 8.1.2), ART-21798 (MTA 8.1.3),
  ART-22698 (MTA 8.2.1) — see `release_schedule`.
