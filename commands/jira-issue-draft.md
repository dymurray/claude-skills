---
description: Draft and create Jira issues following HPBU hierarchy standards for the MTA project
allowed-tools: Read, Bash, AskUserQuestion, mcp__atlassian__searchJiraIssuesUsingJql, mcp__atlassian__getJiraIssue, mcp__atlassian__createJiraIssue, mcp__atlassian__getAccessibleAtlassianResources
---

# Jira Issue Draft

Draft and create Jira issues in the **MTA** project that follow the HPBU hierarchy standards — right type, right template, right parent.

## When to use

- "I need to create a Jira issue for ..."
- "Draft a Story for ..."
- "I have this piece of work, help me write it up in Jira"
- "What issue type should this be?"
- Translating plain-language work descriptions into standards-compliant Jira issues

## Prerequisites

- Atlassian MCP server configured and authenticated
- `JIRA_EMAIL` and `JIRA_API_TOKEN` environment variables (fallback if MCP unavailable)

## Project Context

All issues are created in the **MTA** project (`project = "Migration Toolkit for Applications"`).
The Jira instance is `redhat.atlassian.net`.

Available issue types in MTA: **Epic**, **Story**, **Spike**, **Bug**, **Task**, **Feature Request**, **Sub-task**.

## Workflow

### Step 1: Gather Input

Ask the user to describe the work in plain language. Accept any of:

- Free-form description of the work
- An existing Jira key to use as context (e.g., a parent Epic to create Stories under)
- A specific issue type if the user already knows what they want

If the user provides only a description, proceed to Step 2. If they provide a Jira key, fetch it with `getJiraIssue` to use as context.

### Step 2: Recommend Issue Type

Based on the work description and the hierarchy standards below, recommend an issue type. Present your recommendation with reasoning:

> "Based on your description, this sounds like a **Story** — it delivers a specific capability within a single sprint. Does that sound right, or would you prefer a different type?"

Key decision factors to surface:

- **Scope:** Single sprint → Story/Task. Multi-sprint → Epic.
- **Is it research?** → Spike.
- **Is it a defect?** → Bug.
- **Is it a user-facing enhancement request?** → Feature Request.
- **Is it a finite piece of work / action item?** → Task.

If the work is very large (e.g., describes an Epic), note that you can also suggest potential child Stories/Tasks after the primary issue is drafted.

### Step 3: Identify Parent

Based on the issue type, suggest a parent if appropriate:

1. **Search for candidate parents** using JQL in the MTA project. For example, if drafting a Story, search for unresolved Epics:
   ```
   mcp__atlassian__searchJiraIssuesUsingJql:
     cloudId: "redhat.atlassian.net"
     jql: 'project = "MTA" AND issuetype = Epic AND resolution is EMPTY ORDER BY updated DESC'
     maxResults: 20
     fields: ["summary", "status", "assignee"]
   ```
2. **Present candidates** and ask the user to confirm which parent to use, or confirm that no parent is needed.
3. **Validate** that the selected parent type is allowed per the hierarchy rules (e.g., a Story's parent must be an Epic, never another Story).

Remember: items are not required to have parents. If the user says no parent is needed, that is valid.

### Step 4: Draft Description

Using the confirmed issue type, draft the description following the **exact Jira template** from the Description Templates section below.

**Rules for drafting:**

- **Use the template structure verbatim** — keep all section headings and guidance text from the template.
- **Fill in sections** based on the user's input. Where information is missing, leave the template placeholder text intact so the user knows what to fill in later.
- **For Stories:** Use the "As a / I want / so that" format. Include both functional and non-functional acceptance criteria.
- **For Bugs:** Structure the reproduction steps clearly. Ask for version and reproducibility details if not provided.
- **For Epics:** Keep it concise — goal, importance, scenarios, dependencies.
- **For free-form types** (Spike, Task, Feature Request): Draft a clear, well-structured description appropriate to the type.

**Draft a Summary (title)** as well — concise, specific, and following the conventions visible in the MTA project's existing issues.

### Step 5: Present Draft for Review

Present the complete draft to the user:

```
## Draft Issue

**Type:** [Issue Type]
**Project:** MTA
**Summary:** [Title]
**Parent:** [Parent Key — or "None"]

### Description

[Full description using the Jira template]
```

Ask the user to review and request any changes. Iterate on the draft until the user is satisfied.

### Step 6: Suggest Child Items (when applicable)

If the issue is an Epic, analyze the description for potential child items:

- **Epic** → suggest potential Stories, Spikes, or Tasks

Present suggestions as a bulleted list with recommended type and one-line summary. Do not create these automatically — just surface them for the user to consider.

If the user wants to draft child items, repeat Steps 2–6 for each, using the parent issue as context.

### Step 7: Create in Jira

**Only after explicit user approval**, create the issue.

If MCP tools are available, use `mcp__atlassian__createJiraIssue`. Otherwise, fall back to the Jira REST API via curl:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -X POST \
  -d @/tmp/jira_payload.json \
  "https://redhat.atlassian.net/rest/api/3/issue"
```

**Important:** Use `curl` (not Python's urllib) due to potential SSL certificate issues with Python on macOS.

**Note:** The old `/rest/api/3/search` GET endpoint has been removed. For JQL queries via REST, always use `POST /rest/api/3/search/jql`.

Confirm the creation by showing:

- Issue key and URL
- Summary, type, project, and parent

If the user approved child items in Step 6, offer to draft and create those next.

## Error Handling

- **MCP not configured:** Fall back to Jira REST API with `JIRA_EMAIL` / `JIRA_API_TOKEN` env vars.
- **Parent key not found:** Search for alternatives and present candidates.
- **Issue type not available in project:** Report which types the project supports and suggest an alternative.
- **Create fails:** Show the error, suggest corrections, and offer to retry.

---

# HPBU Jira Hierarchy Standards

## Hierarchy Overview

```
Level 6  Strategic Goal     (HATSTRAT)           — Company Strategy
Level 5  Outcome            (HPSTRAT)            — Org Strategy
Level 4  Feature/Initiative (BU Strategy projects)— BU Strategy
Level 3  Epic               (Team projects)       — Execution
Level 2  Story/Spike/Bug/Task (Team projects)     — Execution
Level 1  Sub-task           (attached to L2)      — Execution
```

Work items link upward via the **Parent** field. Sideways relationships use **link types** (blocks / is-blocked-by, depends on, etc.).

**Items are not required to have parents.** Only create hierarchy as high as needed.

## Allowed Parent-Child Relationships

| Child Type                               | Allowed Parent Type(s)                                         |
| ---------------------------------------- | -------------------------------------------------------------- |
| Sub-task                                 | Story, Spike, Bug, Task, Vulnerability, Weakness, Risk, Ticket |
| Story / Spike / Bug / Task               | Epic                                                           |
| Vulnerability / Weakness / Risk / Ticket | Epic                                                           |
| Epic                                     | Feature or Initiative                                          |
| Feature                                  | Outcome                                                        |
| Initiative                               | Outcome                                                        |
| Outcome                                  | Strategic Goal                                                 |
| Strategic Goal                           | _(none — top of hierarchy)_                                    |

## Issue Type Definitions

### Epic (Level 3)

- **Purpose:** Extra-large work item that won't fit in a single sprint. Includes description of functionality with acceptance criteria, from the user's perspective.
- **Scope:** Larger than a single sprint; smaller than a release. Often scoped to a single team but may be shared. T-shirt sized.

### Story (Level 2)

- **Purpose:** Short, simple description of a capability from the perspective of the person who desires it.
- **Scope:** Fits within a single sprint. Scoped to a single team. Sized in points.

### Spike (Level 2)

- **Purpose:** Time-boxed research activity to help teams make better decisions.
- **Scope:** Single sprint, single team. Sized in points.
- **Delivery means:** Time-box met AND Stories resulting from research/decisions written.

### Bug (Level 2)

- **Purpose:** Error, flaw, or fault in software causing incorrect or unexpected results.
- **Scope:** Single sprint, single team. Sized in points.

### Task (Level 2)

- **Purpose:** Finite piece of work. Great for post-meeting follow-ups and action items.
- **Scope:** Single sprint, single team. Sized in points.

### Feature vs Initiative (Level 4)

| Signal                        | → Feature                          | → Initiative                                    |
| ----------------------------- | ---------------------------------- | ----------------------------------------------- |
| Who benefits?                 | Customers / end users              | Red Hat associates / internal teams             |
| Appears in release notes?     | Yes                                | No (quarterly accomplishments instead)          |
| Product roadmap item?         | Yes                                | No                                              |
| Keywords suggesting this type | customer, user-facing, API, GA     | internal, tech debt, refactor, CI/CD, tooling   |

## Description Templates

### Epic

```
Epic Goal

Why is this important?

Scenarios

Dependencies (internal and external)
```

### Story

```
User Story:
Foundation for product enhancement.

As a <type of user>
I want <some goal>
so that <value/some reason>

Acceptance Criteria: (Definition of Done)
Defines the scope, what to satisfy before the story is proclaimed as completed.
Defines pass/fail criteria.

Verify that…

Functional Acceptance Criteria

Non - Functional Acceptance Criteria
```

### Bug

```
Description of problem:

Version-Release number of selected component (if applicable):

How reproducible:

Steps to Reproduce:
1.
2.
3.

Actual results:

Expected results:

Additional info:
```

### Spike

_(No standard template — free-form description of questions to answer and decisions to make within a time-box.)_

### Task

_(No standard template — free-form description of the action to be completed.)_

### Feature

```
Feature Overview (aka. Goal Summary)

An elevator pitch (value statement) that describes the Feature in a clear, concise way.

Goals (aka. expected user outcomes)

The observable functionality that the user now has as a result of receiving this feature.

Requirements (aka. Acceptance Criteria):

A list of specific needs or objectives that a feature must deliver in order to be considered complete.

Out of Scope

High-level list of items that are out of scope.

Background

Provide any additional context needed to frame the feature.

Documentation Considerations

Provide information that needs to be considered and planned so that documentation will meet customer needs.
```

### Initiative

```
Goal

What is our purpose in implementing this? What are we enabling by doing this work?
Time-box goals to 4-6 months.

Benefit Hypothesis:

What are the benefits (to Red Hat, eventually to customers, to the community, etc.)?

We believe that the result of doing this work will be ...

Resources

Add any resources (docs, slides, etc.) pertinent to the definition of the work.

Responsibilities

Indicate which roles and/or teams will be responsible for contributing.

Success Criteria

Provide some examples of how we will know if we have achieved the goal.
```
