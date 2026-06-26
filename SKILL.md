SEs often juggle multiple deals, and a lot of time is spent recapping the status of the POC and how to move the deal forward. The skill is meant to support the SE in managing the critical elements of a POC - implementation status, use case progress and customer sentiment. 

the skill is meant to take the transcript of the previous call, and tell me about
    1. implementation status and blockers
    2. what use cases have been completed (through a google drive MCP integration)
    3. next steps for all parties 
    4. customer sentiment - champion, influencer, neutral, detractor

possible extensions in the future 
    1. integration with Gong MCP to programatically pull transcripts
    2. develop a database to store a memory of what happened in the earlier call, and report on progress
    3. integration with mixpanel headless to help with report and dashboard creation, or assist customer in data governance 
    
    ---
name: se-poc
description: >
  Analyze Gong call transcripts for POC/sales calls and surface a full analysis directly in Claude —
  including call quality ratings, MEDDPICC assessment, customer sentiment, implementation ownership
  check, next steps, and risks. Optionally cross-references a Google Sheets POC use case tracker.
  Trigger on "analyze this gong call", "analyze this transcript", "POC call analysis", "call recap",
  "analyze my call with [account]", "run through this transcript", or any time a Gong transcript
  file is provided and the user wants structured analysis. Do NOT trigger for pipeline queries or
  deal reviews without a transcript.
---

# Gong Call Analyzer

Analyze Gong POC/sales call transcripts and present a full structured analysis directly in Claude. No writing to external systems — everything surfaces in chat.

## Usage

Provide a Gong transcript file (plain text `.txt`) and optionally a Google Sheet URL for cross-referencing POC use cases:

```
/analyze-gong <transcript-path> [google-sheet-url]
```

**Examples:**
```
/analyze-gong ~/Downloads/call-transcript--Nykaa.txt
/analyze-gong ~/Downloads/call-transcript--Nykaa.txt https://docs.google.com/spreadsheets/d/...
```

---

## Workflow

### Step 0 (if Google Sheet provided): Find and Confirm Use Case Tab

If a Google Sheet URL was given:
1. Use `get_spreadsheet_metadata` to list **all** tabs in the sheet.
2. Display the full list of tab names to the user.
3. Identify any tabs whose names contain (case-insensitive): `use case`, `use cases`, `test case`, `test cases`, `success criteria`, `success`, `POC`.
4. If one or more candidate tabs are found, present them clearly and ask the user to confirm which one contains the use cases:
   *"I found these tabs that could be use cases: [tab A], [tab B]. Which one should I use?"*
5. If no candidate tabs match, show all tab names and ask the user to pick one.
6. **Wait for the user's confirmation before reading any tab data.**
7. Once confirmed, read all rows from that tab using `get_range` and display the raw contents as a markdown table — **copy cell values exactly, do not rephrase or reformat.**
8. Ask: *"Does this look correct? Reply yes to continue."*
9. **Wait for final confirmation before proceeding to Step 1.**

If no Google Sheet was provided, skip this step entirely and proceed to Step 1.

---

### Step 1: Read and Analyze the Transcript

Read the transcript file, then extract all of the following in a single structured pass:

#### 1a. Call Overview
- **Call quality rating** (1–10) with rationale
- **POC health score** (1–10) with status: 🟢 Green / 🟡 Yellow / 🔴 Red
- **Executive summary** (3–5 sentences covering the call's purpose, key outcomes, and current POC status)
- **MEDDPICC snapshot** — score each dimension inline as part of the overview:
  - Metrics, Economic Buyer, Decision Criteria, Decision Process, Identify Pain, Champion, Competition
  - Score: 🟢 Strong / 🟡 Partial / 🔴 Missing
  - One-line evidence and one-line gap for each dimension

#### 1b. Participant Analysis
For each participant:
- Name and role/title
- Classification: Champion / Supporter / Neutral / Skeptic / Detractor
- Engagement level: High / Medium / Low
- Brief rationale

#### 1c. Topics Discussed
For each topic or feature covered:
- Topic name
- Status: Completed / In Progress / Discussed / Not Started / Blocked
- Key details and context

#### 1d. Customer Sentiment
- Overall sentiment: Positive / Mixed / Negative
- Sentiment indicators — quotes or moments that reveal enthusiasm, hesitation, or frustration
- Engagement quality: were they asking good questions, pushing back, going quiet?
- Any shifts in tone during the call (e.g. started skeptical, warmed up)

#### 1e. Implementation Ownership Check
For each workstream or deliverable:
- Task/area
- Named owner (or flag as Unassigned if missing)
- Timeline (or flag as not set if missing)
- Notes on blockers or dependencies

#### 1f. Next Steps
For each agreed action:
- Action description
- Owner
- Timeline
- Supporting quote from transcript (if available)

Also list:
- Meetings scheduled (purpose, attendees, timing, confirmed?)
- Open questions that need resolution

#### 1g. Risks Identified
For each risk:
- Risk description
- Severity: High / Medium / Low
- Mitigation suggestion

---

### Step 2 (if Google Sheet provided): Match Topics to Use Cases

Cross-reference the discussed topics from Step 1c against the use cases read from the sheet.

For each use case, determine:
- Completed — fully addressed in this call
- In Progress — partially addressed or actively being worked on
- Discussed — mentioned but not yet started
- Not Discussed — no mention in the call

Provide a brief evidence note for each.

---

### Step 3: Present Full Analysis in Claude

Output the complete analysis as a structured markdown report directly in chat. Use this format:

```
# POC Call Analysis — {Account Name} ({Date})

## Call Overview
Call Quality: {n}/10 — {rationale}
POC Health: {n}/10 — 🟢/🟡/🔴 {status} — {rationale}

Executive Summary:
{3–5 sentence summary}

MEDDPICC Snapshot:
Table: Dimension | Score (🟢/🟡/🔴) | Evidence | Gap
  Metrics
  Economic Buyer
  Decision Criteria
  Decision Process
  Identify Pain
  Champion
  Competition

## Participant Analysis
Table: Name | Role | Classification | Engagement | Notes

## Topics Discussed
Table: Topic | Status | Details

## Customer Sentiment
Overall: {Positive / Mixed / Negative}
{2–4 sentences on tone, key moments, sentiment shifts}
Key indicators with quotes or paraphrases

## Implementation Ownership
Table: Task | Owner (flag ⚠️ if unassigned) | Timeline (flag ⚠️ if missing) | Notes

## Next Steps
Table: # | Action | Owner | Timeline
Meetings scheduled
Open questions

## Use Case Progress  [only if Google Sheet provided]
Table: Use Case | Previous Status | This Call | Evidence

## Risks Identified
Table: Risk | Severity (🔴 High / 🟡 Medium / 🟢 Low) | Mitigation
```

Use emoji to make statuses scannable:
- POC health: 🟢 Green, 🟡 Yellow, 🔴 Red
- MEDDPICC: 🟢 Strong, 🟡 Partial, 🔴 Missing
- Ownership gaps: ⚠️ Unassigned, ⚠️ No timeline
- Risk severity: 🔴 High, 🟡 Medium, 🟢 Low

---

## Notes

- No data is written to external systems. All output appears in Claude.
- The Google Sheet is read-only (used for use case cross-referencing only).
- Google Sheets MCP tools needed (read-only): `get_spreadsheet_metadata`, `get_range`
- Transcript files should be plain text (`.txt`) exported from Gong.
- If no Google Sheet is provided, the "Use Case Progress" section is omitted.
