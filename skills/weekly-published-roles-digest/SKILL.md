---
name: weekly-published-roles-digest
description: >-
  Reads last week's published role emails from a specified Outlook folder, extracts the roles, evaluates them against your personal profile and SAP seniority level, and presents matches first followed by non-matching roles. On first run, reads your Joule personalization profile and asks you to confirm or adjust criteria, seniority level, and folder name before saving preferences for future runs. Activate when the user says things like: "weekly job digest", "summarize my job roles", "check my job newsletter", "what roles came in this week", "show me this week's jobs", "run my job digest", "published roles digest".
allowed-tools: list_mail_folders list_emails get_email read_file write_file
metadata:
  author: Joule Work Desktop
  version: 1.4.0
  tags: email jobs career published-roles weekly
---

# Weekly Published Roles Digest

Summarize last week's published job roles from a newsletter email folder, evaluate them against personal criteria and seniority level, and present matches first followed by non-matching roles.

## Trigger conditions

Activate when the user says things like:
- "weekly job digest"
- "run my job digest"
- "summarize my job roles"
- "what roles came in this week"
- "show me this week's jobs"
- "check my job newsletter"
- "published roles digest"

**Per-run overrides**: If the user adds extra instructions in the trigger message, apply them for this run only without changing saved preferences. Examples:
- "run my job digest, AI roles first"
- "weekly digest, prioritize anything Kubernetes"
- "show me this week's roles, I'm especially interested in remote positions"

Capture any per-run override and apply it during Step 5 and Step 6.

---

## Step 1 — Load or Confirm Preferences

Check if a preferences file exists at `job-digest-prefs.json` in the working directory using `read_file`. Read it if it exists. If it does not exist or returns an error, treat this as a first run.

### First run (no preferences file found):

1. Tell the user: "First-time setup — let me pull your Joule profile to pre-fill your preferences."

2. Read the user's Joule personalization: their `role_description` and any stated response preferences are available in your context from the system prompt personal instructions section. Use these as the starting criteria.

3. Present what you found and ask for confirmation:

   > "Based on your Joule profile, here's what I'll use to evaluate roles:
   >
   > **Your background:** [role_description from personal instructions]
   > **Your seniority level:** [inferred from profile, e.g. T1 Associate, T2 Specialist]
   > **Priority topics:** None set (you can add topics like 'AI, BTP' to always see those first)
   > **Folder to read:** Publish Roles
   >
   > Does this look right? Say yes to proceed, or tell me what to change — including the folder name, seniority level, and any priority topics."

4. Wait for the user's response. Incorporate any corrections they give.

5. Save confirmed preferences to `job-digest-prefs.json` in the working directory using `write_file`:
   ```json
   {
     "folder_name": "<confirmed folder name>",
     "criteria": "<user criteria as plain text>",
     "profile_summary": "<brief summary of role and background>",
     "seniority_level": "<e.g. T1 Associate, T2 Specialist, T3 Senior, T4 Principal/Manager>",
     "seniority_rule": "Match roles at [level]. May stretch one level up to [level+1]. Anything above that is No Match.",
     "priority_topics": ["<topic1>", "<topic2>"]
   }
   ```
   If the user did not specify priority topics, save `"priority_topics": []`.

### Returning run (preferences file found):

1. Read `job-digest-prefs.json` with `read_file`.

2. Show a one-line confirmation:

   > "Using your saved preferences — folder: **[folder_name]**, seniority: **[seniority_level]**, priorities: **[priority_topics or 'none']**. Ready to run? (Say 'update my preferences' to change anything.)"

3. Wait for confirmation. If the user says "update my preferences", re-run the first-run flow above with existing values pre-filled. Otherwise proceed to Step 2.

---

## Step 2 — Find Published Role Emails

All relevant emails have **"Published Role"** in the subject line. Always use `search_emails` with:
- `query: "subject:published role"`
- `top: 50`

Do not rely on folder lookup — searching by subject is more reliable and covers cases where emails are stored in different folders across users.

From the results, identify emails received in the **last 7 days**. If no emails meet this threshold, tell the user and stop.

---

## Step 3 — Read the Emails

For each email identified in Step 2, call `get_email` to read the full body. Issue all `get_email` calls **in a single message as parallel tool calls** to save time.

---

## Step 4 — Extract Job Roles

From the email bodies, extract every individual job role. For each role, capture:
- **Title** — the job title
- **Company** — the hiring organisation
- **Location** — location or remote/hybrid status
- **Seniority** — the seniority level stated in the role (map to SAP T-level if possible)
- **Key details** — tech stack, domain, contract type, or other notable details
- **Apply link** — the URL from the email body (look for "review the details and apply using the following link" or similar). Always capture this — it is required for the output.

If a single email covers one role (common in SAP RM systems), treat each email as one role.
Deduplicate: if the same Req# appears in multiple emails, count and show it only once.

---

## Step 5 — Evaluate Against Criteria, Seniority, and Priorities

Using the `criteria`, `profile_summary`, `seniority_level`, `seniority_rule`, `priority_topics`, and any per-run override from preferences, evaluate each role.

### SAP seniority ladder (official T-level mapping):

| T-Level | Label |
|---------|-------|
| T1 | Associate |
| T2 | Specialist |
| T3 | Senior |
| T4 | Principal / Manager |

### Seniority rule (apply strictly):
- **Seniority match**: Role is at the user's T-level or one T-level above → eligible for content evaluation
- **Seniority no match**: Role is two or more T-levels above the user → mark as **No Match** regardless of content fit. State the seniority gap as the reason.
- If the role explicitly states all T-levels are welcome, treat it as a seniority match.

### Content evaluation (only for seniority-eligible roles):
- **Match**: Clearly aligns with the user's background, interests, or stated criteria
- **Maybe**: Partially aligns — worth a look but not a strong fit
- **No match**: Does not align with criteria

### Priority topics:
- If `priority_topics` is set in preferences (e.g. ["AI", "BTP"]), roles matching those topics are promoted to the **top of the Roles for You section**, above other matches.
- If a per-run override was given (e.g. "AI roles first"), apply the same promotion logic for this run only.
- Add a **Priority** badge next to the title for promoted roles.
- Per-run override takes precedence over saved priority topics if both are present.

Write one sentence explaining each rating. Be specific.

---

## Step 6 — Present the Digest

Structure the output in two sections, matches first. Within the matches section, priority roles appear at the top.

---

### Roles for You ([count of Match + Maybe])

Present as a table with an Apply column. The Apply link is **mandatory** — never omit it:

| # | Title | Company | Location | Seniority | Why it fits | |
|---|-------|---------|----------|-----------|-------------|---|
| 1 | ⭐ SAP BTP AI Developer | Andel | Remote | T2 Specialist | **Priority match** — AI topic; one T-level up, BTP + AI stack | [Apply](https://...) |
| 2 | Other Match | ... | ... | ... | ... | [Apply](...) |

Use ⭐ to mark priority roles. Label **Match** roles clearly. For **Maybe** roles, add *(Maybe)* next to the title.

---

### Other Roles This Week ([count])

Present as a compact list (no apply links needed here):

- **[Title]** at [Company] — [Location] *(brief note: seniority gap, wrong domain, etc.)*

---

End with a one-line summary, e.g.:
> "1 priority match, 1 other match, 1 maybe, and 20 roles that didn't fit your profile this week."

---

## Edge Cases

- **No roles found**: Say "No published role emails were found in the last 7 days."
- **User says "update my preferences"**: Skip Steps 2–6 and re-run the first-run preferences flow in Step 1, pre-filling current saved values.
- **User says "update my priority topics"**: Ask what topics to add/remove, update `priority_topics` in the preferences file, confirm the change.
- **Emails in a non-English language**: Extract and evaluate in their original language, but present the digest in the user's preferred language.
- **Very large number of roles (20+)**: In the "Other Roles" section, group by category (e.g. "Supply Chain (4)", "Finance (3)") rather than listing each individually.
- **Apply link missing from an email**: Note "Link not available" in the Apply column rather than leaving it blank.
