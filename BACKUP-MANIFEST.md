# ASV2 (Upscale AI Software) - Jira Project Backup

**Backup Date:** 2026-03-27  
**Source Site:** bugatti-asic.atlassian.net  
**Project Key:** ASV2  
**Project Name:** Upscale AI Software  
**Project ID:** 10302  
**Cloud ID:** 1a3482b9-d041-4787-bd1b-bf1358cdfb00  

---

## Backup Contents

| File | Description | Size |
|------|-------------|------|
| `ASV2-all-issues.json` | All issues consolidated (deduplicated, flattened JSON) | 3.3 MB |
| `ASV2-all-issues.csv` | All issues in CSV format for spreadsheet use | 763 KB |
| `project-config.json` | Project configuration (issue types, statuses, priorities) | 1.1 KB |
| `raw-batches/` | Raw API responses (29 pages + key-range batches) | ~30 MB |
| `combine-backup.py` | Script used to consolidate raw data | 9.3 KB |

**Total backup size:** ~34 MB

---

## Issue Statistics

| Metric | Count |
|--------|-------|
| **Total Unique Issues** | **2,901** |
| Key Range | ASV2-1 through ASV2-3367 |

### By Issue Type
| Type | Count |
|------|-------|
| Task | 1,361 |
| Subtask | 994 |
| Story | 271 |
| Bug | 226 |
| Epic | 49 |

### By Status
| Status | Count |
|--------|-------|
| Done | 1,703 |
| OPEN | 815 |
| In Progress | 161 |
| RESOLVED | 93 |
| Review Design/Code | 68 |
| VERIFIED | 32 |
| WONT FIX | 11 |
| REOPENED | 6 |
| ASSIGNED | 5 |
| DUPLICATE | 3 |
| JUNK | 2 |
| NEED INFO | 2 |

### Team Members (49 distinct assignees)
Top assignees: ND Ramesh (467), Subrata Banerjee, Ramana Mohan Rao Dasari, Sukumar Puvvala, Gobinath Krishnamoorthy, and 44 others.

---

## Fields Captured Per Issue

- `key` - Jira issue key (e.g., ASV2-123)
- `id` - Internal Jira issue ID
- `summary` - Issue title
- `description` - Issue description (truncated to 500 chars)
- `issuetype` - Type name (Epic/Story/Task/Subtask/Bug)
- `status` - Current status name
- `statusCategory` - Status category (To Do/In Progress/Done)
- `priority` - Priority name
- `assignee` / `assigneeEmail` - Assigned person
- `reporter` / `reporterEmail` - Issue reporter
- `created` / `updated` - Timestamps
- `labels` - Array of labels
- `components` - Array of component names
- `fixVersions` - Array of fix version names
- `parentKey` / `parentSummary` - Parent issue reference
- `subtasks` - Array of child subtask references
- `issuelinks` - Array of issue link references

---

## How to Re-run Consolidation

```bash
cd /Users/prravichandran/ASV2-backup
python3 combine-backup.py
```

This regenerates `ASV2-all-issues.json` and `ASV2-all-issues.csv` from the raw batch files.

---

## Notes

- Project was originally migrated to Jira Cloud on 2026-02-26 21:54 UTC
- Project type: Team-managed (simplified)
- Board: #104
- Key gap: Issue keys jump from low range (~ASV2-1 to ASV2-100+) to high range (ASV2-3000+) due to migration renumbering
- Full descriptions in ADF format are preserved in `raw-batches/` files
- Attachments and comments are NOT included in this backup (would require individual issue API calls)
