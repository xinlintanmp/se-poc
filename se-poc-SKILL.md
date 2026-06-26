---
name: analyze-gong
description: Analyze Gong call transcripts, match to use case tracker in Google Sheets, and write next steps
---

# Gong Call Analyzer

Analyze Gong sales/POC call transcripts, automatically match discussed topics to your POC use case tracker in Google Sheets, and write a summary with progress updates to a new tab.

## Usage

```bash
/analyze-gong <transcript-path> <google-sheet-url>
```

**Example:**
```bash
/analyze-gong ~/Downloads/call-transcript--Nykaa.txt https://docs.google.com/spreadsheets/d/1Xc3pSoV1SZa0rwSJ8R_klQsBFe8i5VrrGOLZ-l8m0yE/edit
```

## What This Skill Does

### 1. Analyze the Gong Transcript
Extracts from the call transcript:
- **Call quality rating** (1-10) with rationale
- **POC health score** (1-10) with status (🟢 green / 🟡 yellow / 🔴 red)
- **Participant analysis** - identifies champions, supporters, neutrals, skeptics, detractors
- **Topics discussed** - what features/use cases were covered and their status
- **Next steps** - agreed actions, meetings scheduled, open questions
- **Blockers** - technical or business issues identified

### 2. Read Your Google Sheets Use Case Tracker
- Opens the Google Sheet you provide
- Finds the use case tracker tab (e.g., "POC Tracker", "Use Cases")
- Reads all use cases with their current status

### 3. Match Call Topics to Use Cases
For each use case in your tracker, determines:
- ✅ **Completed** - fully addressed in the call
- 🟡 **In Progress** - partially addressed or being worked on
- 💬 **Discussed** - mentioned but not started
- ❌ **Not Discussed** - no mention in the call

Provides evidence from the transcript for each status.

### 4. Write Results to New Google Sheet Tab
Creates a new tab called `Call Summary - {date}` with:
- Call ratings and summary
- Use case progress with status updates
- Next steps with owners and timelines
- Blockers identified

### 5. Return Summary
Returns a formatted markdown summary showing what was accomplished.

## Implementation

Use the following workflow:

### Step 1: Parse Arguments
```typescript
const parts = args.trim().split(/\s+/)
if (parts.length < 2) {
  throw new Error("Please provide both transcript path and Google Sheet URL")
}
const transcriptPath = parts[0]
const sheetUrl = parts.slice(1).join(' ')
```

### Step 2: Read and Analyze Transcript

Read the transcript file, then analyze with structured output:

**Analysis Schema:**
```json
{
  "type": "object",
  "required": ["call_rating", "poc_rating", "discussed_topics", "discussed_next_steps", "summary"],
  "properties": {
    "call_rating": {
      "type": "object",
      "required": ["score", "rationale"],
      "properties": {
        "score": { "type": "number", "description": "1-10" },
        "rationale": { "type": "string" }
      }
    },
    "poc_rating": {
      "type": "object",
      "required": ["health_score", "status", "rationale"],
      "properties": {
        "health_score": { "type": "number" },
        "status": { "type": "string", "enum": ["green", "yellow", "red"] },
        "rationale": { "type": "string" }
      }
    },
    "participants_analysis": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "role", "classification"],
        "properties": {
          "name": { "type": "string" },
          "role": { "type": "string" },
          "classification": { 
            "type": "string", 
            "enum": ["champion", "supporter", "neutral", "skeptic", "detractor"] 
          },
          "engagement_level": { "type": "string", "enum": ["high", "medium", "low"] },
          "rationale": { "type": "string" }
        }
      }
    },
    "discussed_topics": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["topic", "status", "details"],
        "properties": {
          "topic": { "type": "string" },
          "status": { 
            "type": "string", 
            "enum": ["completed", "in_progress", "discussed", "blocked", "not_started"] 
          },
          "details": { "type": "string" }
        }
      }
    },
    "discussed_next_steps": {
      "type": "object",
      "required": ["agreed_actions", "meetings_scheduled"],
      "properties": {
        "agreed_actions": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["action", "owner", "timeline"],
            "properties": {
              "action": { "type": "string" },
              "owner": { "type": "string" },
              "timeline": { "type": "string" },
              "quote": { "type": "string" }
            }
          }
        },
        "meetings_scheduled": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["purpose", "timing"],
            "properties": {
              "purpose": { "type": "string" },
              "attendees": { "type": "string" },
              "timing": { "type": "string" },
              "confirmed": { "type": "boolean" }
            }
          }
        },
        "open_questions": {
          "type": "array",
          "items": { "type": "string" }
        }
      }
    },
    "key_blockers": {
      "type": "array",
      "items": { "type": "string" }
    },
    "summary": { "type": "string" }
  }
}
```

**Analysis Prompt:**
```
Analyze this Gong sales/POC call transcript.

Focus on:
1. Overall call quality and POC health
2. What features, use cases, or capabilities were discussed (and their status)
3. Next steps that were explicitly agreed to
4. Any blockers or issues
5. Participant engagement (who are champions, supporters, etc.)

Transcript:
{transcript_content}
```

### Step 3: Read Google Sheet Use Cases

Use Google Sheets MCP tools to read the use case tracker.

**Prompt for reading sheet:**
```
You have access to Google Drive/Sheets tools via MCP.

Your task:
1. Open this Google Sheet: {sheetUrl}
2. Find and read the use case tracker tab (commonly named "Use Cases", "POC Tracker", or similar)
3. Return all use cases with their current status

Use the available Google Sheets MCP tools (get_spreadsheet_metadata, get_range, get_cells, etc.) to:
- List all tabs in the spreadsheet
- Identify which tab contains the use cases
- Read all the use case data

Return a structured list.
```

**Expected schema:**
```json
{
  "type": "object",
  "required": ["use_cases"],
  "properties": {
    "use_cases": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["use_case", "current_status"],
        "properties": {
          "use_case": { "type": "string" },
          "current_status": { "type": "string" },
          "notes": { "type": "string" }
        }
      }
    },
    "sheet_tab_name": { "type": "string" }
  }
}
```

### Step 4: Match Topics to Use Cases

**Matching prompt:**
```
You have:
1. A list of use cases from a Google Sheet
2. Topics discussed in a Gong call

Match the discussed topics to the use cases and determine which use cases were addressed.

USE CASES FROM SHEET:
{JSON.stringify(use_cases, null, 2)}

TOPICS DISCUSSED IN CALL:
{JSON.stringify(discussed_topics, null, 2)}

For each use case, determine if it was:
- ✅ Completed (fully addressed in the call)
- 🟡 In Progress (partially addressed or being worked on)  
- 💬 Discussed (mentioned but not started)
- ❌ Not Discussed (no mention in the call)

Return the matched results with evidence from the transcript.
```

**Matching schema:**
```json
{
  "type": "object",
  "required": ["matched_use_cases"],
  "properties": {
    "matched_use_cases": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["use_case", "call_status", "evidence"],
        "properties": {
          "use_case": { "type": "string" },
          "previous_status": { "type": "string" },
          "call_status": { 
            "type": "string", 
            "enum": ["completed", "in_progress", "discussed", "not_discussed"] 
          },
          "evidence": { "type": "string" }
        }
      }
    }
  }
}
```

### Step 5: Write Results to Google Sheet

Create a new tab with formatted results.

**Content format:**
```
CALL SUMMARY

Call Quality: {score}/10
POC Health: {score}/10 ({status})

SUMMARY
{executive_summary}

USE CASE PROGRESS
Use Case | Status in Call | Evidence
{use_case_1} | {status_emoji} {status} | {evidence}
{use_case_2} | {status_emoji} {status} | {evidence}
...

NEXT STEPS
1. {action} (Owner: {owner}, Timeline: {timeline})
2. {action} (Owner: {owner}, Timeline: {timeline})
...

BLOCKERS
- {blocker_1}
- {blocker_2}
```

**Prompt for writing:**
```
Using the Google Sheets MCP tools:

1. Open this sheet: {sheetUrl}
2. Create a new tab called "Call Summary - {YYYY-MM-DD}"
3. Write the following content to the new tab:

{formatted_content}

Use the appropriate MCP tools (add_sheet, update_cells, etc.) to create the tab and write the data.
```

### Step 6: Format Output

Return a formatted markdown summary:

```markdown
# 📊 Gong Call Analysis Complete

## Executive Summary
{summary}

## Ratings
- **Call Quality:** {score}/10
- **POC Health:** {score}/10 ({STATUS})

## Use Case Progress
✅ **{use_case}** - completed
   {evidence}

🟡 **{use_case}** - in_progress
   {evidence}

❌ **{use_case}** - not_discussed
   No mention in call

## Next Steps
1. **{action}**
   - Owner: {owner}
   - Timeline: {timeline}

## ⚠️ Blockers
- {blocker}

📄 **Full results written to:** [Call Summary - {date}]({sheetUrl})
```

## Google Sheets MCP Tools

The skill should use these Google Sheets MCP tools:

- **get_spreadsheet_metadata** - Get sheet tabs and structure
- **get_range** - Read cell values as plain text
- **get_cells** - Read detailed cell data with formulas
- **search_spreadsheet_rows** - Search for specific data
- **add_sheet** - Create new tab
- **update_cells** - Write data to cells
- **batch_update_spreadsheet** - Make multiple updates at once

## Example

**Input:**
```bash
/analyze-gong ~/Downloads/call-transcript--Nykaa.txt https://docs.google.com/spreadsheets/d/1Xc3pSoV1SZa0rwSJ8R_klQsBFe8i5VrrGOLZ-l8m0yE/edit
```

**Expected behavior:**
1. Reads Nykaa transcript
2. Extracts: Experimentation, Feature Flags, Cohort Creation, Event Tracking discussed
3. Reads use cases from Google Sheet "POC Tracker" tab
4. Matches topics to use cases with status updates
5. Creates "Call Summary - 2026-06-26" tab with results
6. Returns formatted markdown output

**Sample output:**
```markdown
# 📊 Gong Call Analysis Complete

## Executive Summary
Weekly sync covering iOS launch prep (next week target). Technical questions resolved but QA incomplete, key resource on leave, Android not started.

## Ratings
- **Call Quality:** 6/10
- **POC Health:** 5/10 (YELLOW)

## Use Case Progress
✅ **Mixpanel Experimentation** - completed
   iOS implementation done via SDK, auto-fires experiment_started events

🟡 **Cohort Creation** - in_progress
   PM already creating cohorts but encountered property confusion (beaker icon issue)

💬 **Event Tracking** - discussed
   Discussed migration from Firebase to Mixpanel auto-tracking

❌ **Dashboards** - not_discussed
   No mention in this call

## Next Steps
1. **Complete QA validation for iOS**
   - Owner: Abhishek
   - Timeline: Coming days

2. **Assign Android implementation owner**
   - Owner: TBD
   - Timeline: Not set

## ⚠️ Blockers
- Android platform has no owner or timeline
- Vamsi (analytics lead) on paternity leave during critical validation phase

📄 **Full results written to:** [Call Summary - 2026-06-26](https://docs.google.com/spreadsheets/d/1Xc3pSoV1SZa0rwSJ8R_klQsBFe8i5VrrGOLZ-l8m0yE/edit)
```

## Notes

- The skill requires Google Drive MCP connector to be available and authenticated
- Works best in Claude desktop app where MCP tools are fully supported
- Transcript files should be plain text (.txt) format from Gong
- Google Sheet must be accessible to the authenticated Google account
