---
name: "jotform-intake"
description: "Process Leadership Now Project (LNP) member onboarding JotForm survey responses and create or update contacts in DonorDock via the MCP. Use this skill whenever a user pastes, shares, or uploads a JotForm submission — whether as a CSV row, pasted text, or a single member's answers. Also triggers when someone says \"process this new member\", \"add this JotForm response\", \"onboard this member\", or \"create a contact from this survey\". ALSO runs in FETCH MODE on a schedule (e.g. twice daily) or when asked to \"pull new submissions\", \"check JotForm for new members\", or \"run the intake\": it pulls new submissions directly from JotForm via list_submissions and processes each one. Always use this skill rather than manually calling individual MCP tools for JotForm intake."
---

# JotForm → DonorDock Intake

You are processing Leadership Now Project member onboarding survey submissions into DonorDock.
Work through the steps below in order. Be methodical — each step depends on the previous one.

The core job of this skill: read every JotForm answer, map it to the correct DonorDock
destination (core contact field, custom field, or badge), and write it accurately — including
derived logic (Chapter from ZIP, Cohort from submission date) — so DonorDock always reflects
what the member actually entered on the form.

---

## Scope — where this skill ends

This skill owns the **inbound** direction: a JotForm submission becomes DonorDock data. It is also
the authority on what a form answer *means* — which question belongs in which DonorDock field,
badge, or derived value.

The **outbound** direction — generating a personalized prefill link so an existing member receives
the form already populated from their DonorDock record — belongs to the `jotform-prefill` skill.
That skill owns the URL parameter layer: JotForm unique names, value formatting, and dropdown
option matching. It defers to this skill on field semantics.

The two skills describe the same relationships in opposite directions, so if a form field is added,
renamed, or reordered, both mapping tables need updating. Checking the other one is cheap
insurance against silent drift.

---

## How This Skill Runs — Two Modes

**Fetch mode (default).** This is the normal operation — on a schedule or whenever the skill
is invoked without a submission in hand. Start at Step 0: pull recent submissions yourself,
run Steps 1–7d for each *new* one, and finish with the batch digest in Step 9.

**Single-submission mode (override).** Only when a human pastes or uploads one specific
submission. Skip Step 0 and start at Step 1, processing that one record through Steps 1–8.

Routing rule: if — and only if — a submission is provided in the message, use single-submission
mode. In every other case (scheduled run, "run the intake", "check JotForm", or any ambiguous
invocation), default to fetch mode.

**Which window when not told:** if the run isn't labeled morning or afternoon, decide by the
current time in America/Chicago — before 12:00 CT use the morning window, otherwise the
afternoon window (see Step 0).

---

## Step 0 — Fetch Mode (Scheduled Runs)

### Configuration
- `ONBOARDING_FORM_ID`: `260216535465052` — new-member onboarding survey.
- `EXISTING_MEMBER_FORM_ID`: `262105169919159` — **V2 2026 Existing Member Survey**, used for
  re-surveying existing members via prefilled links. It's a clone of the onboarding form and its
  submissions map identically (same question set, same Step 1 field table) — the one difference
  is that everyone who submits here is already a DonorDock contact, which matters for Member
  Since handling below.
- Run cadence: twice daily (see "Scheduling" section at the end). Run times 07:00 and
  16:00 America/Chicago (CT).

If either form ID is blank, resolve it once with the JotForm `search` tool by form title
(e.g. "Member Onboarding" / "Existing Member Survey") and record the numeric form ID in the
config above so future runs are deterministic.

### Pull recent submissions
Call `list_submissions` with:
- `form_ids`: `[ONBOARDING_FORM_ID, EXISTING_MEMBER_FORM_ID]`
- `filter.date_filter`:
  - **Afternoon run:** `[today, today]`
  - **Morning run:** `[yesterday, today]` — picks up anything submitted after yesterday's
    afternoon run.
- `filter.is_filtered_by_rules`: `false`
- `limit`: `50`

`date_filter` is day-granularity only, so the morning and afternoon runs will overlap on the
same day's records. That is expected — the dedup check below makes overlapping runs safe.

Record each submission's originating form_id alongside its parsed fields — Step 5 branches on
it (new-member vs. existing-member survey) when writing Member Since.

### Dedup — the "Onboarding Survey" custom field is the processed marker
For each submission returned, before processing:
1. Search DonorDock by the submission's email (`search_contacts`).
2. If a contact exists, call `get_contact_custom_fields` and check whether **Onboarding Survey**
   is already `true`. If so, this member's data was already merged in an earlier run —
   **skip entirely.** Do not re-write fields, do not re-run Steps 5–7d.
3. Otherwise, treat it as new or incomplete and run Steps 1–7d for that submission.

**Note:** the **Onboarding Flow** badge is applied by a separate DonorDock automation, not by
this skill, and may lag behind this run. Don't use it for dedup — use the Onboarding Survey
custom field, which this skill controls directly and sets only once a submission is fully
processed (see Step 7d).

### Field source in fetch mode
Submissions from `list_submissions` arrive as structured answers (question label → answer),
not CSV columns. The field mapping in Step 1 still applies — just read each field from the
submission's answers by its question label instead of a CSV column. Multi-select answers come
back as a list rather than repeated columns.

Process each new submission through Steps 1–7d, collecting per-member results, then produce the
batch digest in Step 9.

---

## Step 1 — Parse the Input

The input may arrive as:
- A pasted CSV row (with or without headers)
- Free-form text summarizing a member's answers
- A single row extracted from the JotForm CSV export
- A structured submission from `list_submissions` (fetch mode)

Extract these fields (leave blank if not present):

| Field | JotForm Column | DonorDock / Klaviyo Target |
|---|---|---|
| First Name | Split from "Full Name" on first space | contact.FirstName |
| Last Name | Everything after the first space in "Full Name" | contact.LastName |
| Email | "Primary Email" | contact.Email |
| Phone | "Phone Number" | contact.MainPhone |
| Mobile Phone | "Mobile Phone" / "Cell Phone" | contact.MobilePhone |
| Date of Birth | "Date of Birth" | Contact Attributes → Date of Birth (contact.DateOfBirth) |
| Address | "Address" — parse into street, city, state, zip | contact address fields |
| Employer | "Who is your current employer?" | contact.Employer + contact.JobTitle |
| Job Title | "What is your job title?" | contact.JobTitle |
| Graduate Academic Affiliation | "What is your Graduate Academic Affiliation?" | FieldId 3 (Text) |
| Graduate Year | "Year Graduated" (first occurrence) | FieldId 4 (Number) |
| Undergraduate Academic Affiliation | "What is your Undergraduate Academic Affiliation?" | FieldId 5 (Text) |
| Undergraduate Year | "Year Graduated" (second occurrence / "Year Graduated.1" in CSV) | FieldId 6 (Number) |
| Industry | "What industry do you work in?" | FieldId 29 (Select) |
| Sector | "What sector?" or similar | FieldId 30 (Select) |
| Member Tier | Member tier field on form | FieldId 31 (Select) |
| Political Affiliation | "What is your current political party affiliation, if any?" | FieldId 11 (Select) |
| Pay to Play | "Do you have Pay to Play Restrictions?" (Yes/No) | FieldId 20 (Boolean) — "Yes" → checkbox checked (true), "No" → unchecked (false) |
| Archetype Potential | "Archetype Potential" field on form | DonorDock custom field "Archetype Potential" **and** Klaviyo profile property `archetype` |
| Influence Style Signal | "Influence Style Signal" field on form | DonorDock custom field "Influence Style Signal" |
| Network Strength Signal | "Network Strength Signal" field on form | DonorDock custom field "Network Strength Signal" |
| LinkedIn URL | "What is your LinkedIn URL?" | Attempted write to contact LinkedIn URL field (best-effort — see Step 4) |
| Copy Assistant? | "Should we copy your Assistant/Scheduler on all communications?" | DonorDock custom field "Copy Assistant?" (Boolean) |
| Assistant Name | "What is your Assistant/Scheduler's name?" | DonorDock custom field "Assistant Name" (Text) |
| Assistant Email | "What is your Assistant/Scheduler's email address?" | FieldId 12 (Text) |
| Priority Interests | "Which of these Leadership Now 2025-26 key priorities..." (multi-select) | Badges **and** Klaviyo profile property `priority_focus_areas` |
| Contribution Modes | "How are you best positioned to support this work right now?" (multi-select) | Badges |
| Policy Expertise | "In which areas do you have policy expertise..." | Badges |
| Member Notes | "Anything else you'd like us to know..." | contact.Description |
| Submission Date | "Submission Date" | Used for Cohort derivation |

Only write Assistant Name / Assistant Email / Copy Assistant? if the member actually entered an
Assistant/Scheduler on the form — leave these blank otherwise.

**Address parsing:** The address arrives as one string like "555 California St San Francisco, CA, 94104".
Parse it as: everything before the last comma-separated city block is Address1, then City, State abbreviation, Zip. Country defaults to "United States".

**Multi-select columns:** JotForm spreads multi-select answers across repeated columns with the same header. Collect all non-empty values from all "How are you best positioned..." columns as the Contribution Modes list. (In fetch mode these arrive as a single list — collect all values either way.)

---

## Step 2 — Check for Duplicate

Before creating, search DonorDock for the member by email or name using search_contacts. If they already exist, skip create_contact and use update_contact instead with their contactId.

**Fetch mode:** if the existing contact's **Onboarding Survey** custom field is already `true`,
this submission's data was already merged in a prior run — skip it entirely (see Step 0 dedup).
Only fall through to update_contact for a contact that exists but hasn't had survey data merged yet.

Be aware that DonorDock carries substantial duplication, and duplicates often split one person's
data across records — one has the current email, another has the employer. If several records
plausibly match, prefer the one with the most complete custom fields and real giving history, and
flag the duplication in the Step 8 summary so someone can merge them at the source.

---

## Step 3 — Create (or Update) the Contact

Call `create_contact` with:
- firstName, lastName
- email
- mainPhone
- mobilePhone
- dateOfBirth
- address1, city, stateOrProvince, postalCode, country

If the contact already exists, call `update_contact` with only the fields that are changing.

After creation, note the returned contactId — you will need it for all subsequent steps.

**Owner:** no action needed here — OwnerId now defaults to Lauren Barra Rourke automatically on
the DonorDock side, so this skill does not need to set or flag it.

---

## Step 4 — Write Employer, Job Title, Description, and LinkedIn

Make these `update_contact` calls if the values are non-empty:

**Employer + job title:**
```
update_contact({ contactId, employer: "...", jobTitle: "..." })
```
(`update_employment` also works now as a direct alternative if preferred, but `update_contact`
covers employer/jobTitle in one call and needs no extra step.)

**Description** (if "Anything else you'd like us to know" has content):
```
update_contact({ contactId, description: "..." })
```

**LinkedIn URL (best-effort):**
```
update_contact({ contactId, linkedInUrl: "..." })
```
Attempt this write whenever a LinkedIn URL was submitted. Social-field writes have been flaky
via the API historically, so if the call errors or the value doesn't persist on read-back, add
LinkedIn to the "Needs manual entry" list in Step 8 rather than blocking the rest of the run.

---

## Step 5 — Write Custom Fields

Call `set_contact_custom_fields` with all mappable custom fields below. Select fields accept
label strings directly; pass values by field name — no FieldId lookup needed unless a write
comes back `confirmed: false`.

### Chapter — derive from state and ZIP

| State(s) | Chapter |
|---|---|
| NY, NJ, CT | Greater New York |
| DC, MD, VA | Washington D.C. |
| MA, NH, RI, VT | Boston |
| FL | Florida |
| GA | Georgia |
| TX | Texas |
| WI | Wisconsin |
| CA — ZIP > 93999 AND ZIP < 96200 | Bay Area |
| CA — ZIP > 89999 AND ZIP < 94000 | Los Angeles |
| ME | Maine |
| OH | Ohio |
| NC | North Carolina |
| MI | Michigan |
| PA | Pennsylvania |
| AZ | Arizona |
| Non-US address | International |
| All others | Leave blank — manual assignment |

For California addresses, derive Chapter strictly from the parsed ZIP code using the two rules
above (not city name). The ranges are contiguous and non-overlapping: 90000–93999 → Los Angeles,
94000–96199 → Bay Area. If the ZIP falls outside both ranges, leave Chapter blank for manual
assignment.

Watch for ZIPs that arrive without their leading zero — DonorDock stores Northeast ZIPs unpadded
(`7450` for a Ridgewood NJ address). Pad to five digits before applying any ZIP range logic.

### Cohort — derive from submission date

| Year | Date Range | Cohort |
|---|---|---|
| 2026 | Jan 1 – Apr 30 | Q1 |
| 2026 | May 1 – Jun 30 | Q2 |
| 2026 | Jul 1 – Sep 30 | Q3 |
| 2026 | Oct 1 – Dec 31 | Q4 |
| 2027+ | Jan 1 – Mar 31 | Q1 |
| 2027+ | Apr 1 – Jun 30 | Q2 |
| 2027+ | Jul 1 – Sep 30 | Q3 |
| 2027+ | Oct 1 – Dec 31 | Q4 |

### Political Affiliation — valid option strings

Pass the survey answer as-is. Valid DonorDock values: `Democratic`, `Republican`,
`Independent / Unaffiliated`, `Prefer not to say`, `Other`.

### Pay to Play — Boolean

Map the JotForm Yes/No answer directly: "Yes" → `true` (checkbox checked), "No" → `false`
(checkbox unchecked).

### Membership Status — set on every processed submission

Whenever this skill merges survey data for a member (new contact or update), set the
**Membership Status** custom field to `"Active - Current"`. This reflects that the person has
an active, current onboarding record — not a payment or dues status.

### Member Since — from submission date (ONBOARDING_FORM_ID only)

Write the JotForm **Submission Date** as Member Since (FieldId 17), but only for submissions
from `ONBOARDING_FORM_ID`. This marks membership as starting the day the onboarding survey was
completed, not the date of any gift — write it on every processed new-member submission, gift
history or not.

**Skip this write entirely for `EXISTING_MEMBER_FORM_ID` submissions.** Everyone submitting the
V2 existing-member survey is already a DonorDock contact with a real, earlier Member Since —
overwriting it with today's resubmission date would erase their actual join date (e.g. a member
who joined in 2018 would incorrectly show as joining the day they filled out a check-in survey).
Every other field in this step (Chapter, Cohort, Political Affiliation, Pay to Play, Archetype,
Membership Status, etc.) still writes normally regardless of which form the submission came
from — this exception applies to Member Since only.

### Full set_contact_custom_fields call

```
set_contact_custom_fields({
  contactId,
  fields: {
    "Graduate Academic Affiliation": "...",     // FieldId 3, Text
    "Graduate Year": 2005,                      // FieldId 4, Number
    "Undergraduate Academic Affiliation": "...", // FieldId 5, Text
    "Undergraduate Year": 2001,                 // FieldId 6, Number
    "Political Affiliation": "Democratic",      // FieldId 11, Select
    "Assistant/Scheduler Email Address": "...", // FieldId 12, Text — only if provided
    "Chapter": "Bay Area",                      // FieldId 16, Select — derived from ZIP
    "Member Since": "2026-01-18",               // FieldId 17, Date — ONBOARDING_FORM_ID only, omit for EXISTING_MEMBER_FORM_ID
    "Pay to Play Restrictions?": true,          // FieldId 20, Boolean — from Yes/No answer
    "Industry": "...",                          // FieldId 29, Select
    "Sector": "...",                            // FieldId 30, Select
    "Member Tier": "...",                       // FieldId 31, Select
    "Cohort": "Q2",                              // FieldId 32, Select — derived
    "Archetype Potential": "...",
    "Influence Style Signal": "...",
    "Network Strength Signal": "...",
    "Assistant Name": "...",                    // only if provided
    "Copy Assistant?": true,                    // only if provided, Boolean
    "Membership Status": "Active - Current",    // set on every processed submission
  }
})
```

Omit any field with a blank/null value (except Membership Status, which always gets set).
Verify the response shows `confirmed: true` for each field. Any field with `confirmed: false`
goes to the manual entry list in Step 8.

Do **not** include `"Onboarding Survey"` in this call — it is set separately, and last, in
Step 7d, once everything else below has succeeded.

---

## Step 6 — Add Badges

Badges are DonorDock's multi-value tagging system. Call `add_badge` once per badge. Always use
the contactId from Step 3.

### Priority Interests → "Priority: [X]" badges

| JotForm Answer (contains...) | Badge |
|---|---|
| "Policy:" | Priority: Policy |
| "Risk:" | Priority: Risk |
| "Restore the Balance of Power" | Priority: Balance of Power |
| "Talent:" | Priority: Talent |
| "Expand our proven model" | Priority: Expand |

### Contribution Modes → "Contribution: [X]" badges

| JotForm Answer (contains...) | Badge |
|---|---|
| "financial resources" or "Invest or donate" | Contribution: Financial |
| "time and energy" or "volunteer" or "nominating" | Contribution: Time/Energy |

### Policy Expertise → "Policy: [X]" badges

For each policy area listed, add a badge: "Policy: Immigration", "Policy: Workforce/AI",
"Policy: Housing", etc.

**Do not add an "Onboarding Flow" badge here.** That badge is applied automatically by a
separate DonorDock automation once its own conditions are met on the contact record — this
skill does not add it and should not depend on it being present at the time this run finishes.

---

## Step 7 — Create Klaviyo Profile and Enroll in List

After DonorDock is complete, create or update the member's Klaviyo profile and enroll them
in the Leadership Now Members list.

### 7a — Subscribe and enroll

Call `subscribe_profile_to_marketing` with:
- `email` — from survey
- `subscriptions.email.marketing.consent` = "SUBSCRIBED"
- List relationship: **Leadership Now Members** (list ID: `YqM4pm`)

This call creates the profile if it doesn't exist, updates it if it does, and enrolls them
in the list — all in one shot. It also returns the Klaviyo profile ID needed for 7b.

### 7b — Write profile fields

Call `update_profile` on the returned profile ID with:

| Klaviyo Field | Source |
|---|---|
| `first_name` | From survey |
| `last_name` | From survey |
| `location.address1` | From survey |
| `location.city` | From survey |
| `location.region` | State abbreviation from survey |
| `location.zip` | From survey |
| `location.country` | "United States" (default) |
| `properties.archetype` | "Archetype Potential" answer from survey |
| `properties.priority_focus_areas` | Priority Interests list from survey (array of the selected priority labels) |

Writing Archetype and Priority Interests as custom `properties` is the simplest way to make both
filterable in Klaviyo — they can be used directly in segment conditions on
`properties.archetype` / `properties.priority_focus_areas`.

### 7c — Enrollment notes

- `subscribe_profile_to_marketing` must always be called **before** `update_profile` — it
  returns the profile ID the latter requires.
- Klaviyo matches profiles by email — if the profile already exists it will update in place,
  not create a duplicate.
- Do **not** enroll in Day 1 flow here — that is triggered by the separate Enrollment Bridge
  Zap (dues paid in DonorDock → Klaviyo Day 1 flow), which Marie owns.

### 7d — Mark the submission complete

Only after Steps 5, 6, and 7 have all succeeded, make one final call:

```
set_contact_custom_fields({
  contactId,
  fields: { "Onboarding Survey": true }
})
```

This is the completion marker: it confirms the survey data has been fully merged into DonorDock,
and it's what Step 0/Step 2 check on the next run to avoid reprocessing this member. Set it last
and only on full success — if any earlier step failed outright, leave it unset so the next run
retries the missing pieces.

---

## Step 8 — Output a Summary (per member)

Present a clean summary with:

**Contact created/updated** — name, contactId, and direct DonorDock link:
`https://leadershipnowproject1.donordock.com/donors/detail/{contactId}`

**Written successfully — DonorDock** — list every field, custom field, and badge saved,
including Mobile Phone, Date of Birth, Membership Status ("Active - Current"), Onboarding
Survey (checked), Archetype Potential, Influence Style Signal, Network Strength Signal, and
Assistant details when provided.

**Written successfully — Klaviyo** — confirm:
- Profile fields written (name, location, archetype, priority focus areas)
- Enrolled in: Leadership Now Members (YqM4pm)

**Needs manual entry in DonorDock** — list every blocked field with the value so staff can paste it in:
- LinkedIn: {value} — only if the Step 4 write failed or didn't persist
- Any custom field that came back `confirmed: false`

The "Needs manual entry" section is just as important as the successes — staff rely on it to complete the intake.

In single-submission mode this is the final output. In fetch mode, collect each member's
result and roll them into the Step 9 digest.

---

## Step 9 — Batch Digest (Fetch Mode Only)

After processing all submissions in the run, output one digest:

- **Run:** {date} {morning|afternoon} run
- **Pulled:** N submissions in window (broken out by form: onboarding vs. existing-member)
- **New / processed:** N
- **Skipped (already onboarded):** N
- **Per new member:** name + source form (New Member Onboarding / Existing Member Update) +
  DonorDock link + a one-line "needs manual entry" flag if any
- **Anomalies to review:** any failed write, any submission that could not be parsed

If a Slack channel is configured for the routine, post this digest there via the Slack MCP so
the team has visibility without opening Claude. Keep it to the counts plus anything that needs
a human — not a wall of per-field detail.

---

## Known Limitations

- **LinkedIn** is attempted via `update_contact` (Step 4) but social-field writes have a history
  of failing silently or 404ing. Treat every LinkedIn write as best-effort and confirm on
  read-back; fall back to manual entry if it doesn't stick.
- Everything else previously blocked (`update_employment`, Archetype/Archetype Potential,
  Influence Style Signal, Network Strength Signal) now writes successfully via the MCP and is
  handled directly in Steps 4–5 — there is nothing else on this list to skip.

---

## Scheduling: Running 2× per Day

This skill does not schedule itself. Wire the cadence with a **Claude Code Routine** (cloud —
runs on Anthropic's infrastructure, so it does not depend on anyone's laptop being on; a Cowork
scheduled task would not be reliable here):

1. Put this skill in the routine's repository (skills committed into the repo are available to
   the run).
2. Create a routine at claude.ai/code/routines with a self-contained prompt telling Claude to
   run the jotform-intake skill in fetch mode, determining morning vs. afternoon from the
   current time in America/Chicago.
3. Attach the **JotForm, DonorDock, and Klaviyo** connectors (plus Slack if you want the digest
   posted).
4. Add **two scheduled triggers** on the same routine — 07:00 and 16:00 America/Chicago (CT).

Things to know before it goes live:
- Routines are in research preview, with daily run caps by plan tier (roughly 5 / 15 / 25 for
  Pro / Max / Team+Enterprise) — two runs/day fits comfortably.
- A routine runs under the identity of whoever created it, so connector auth is that person's.
  Create it under a shared/service account if you want it org-owned rather than tied to one
  person. Alternatively, have the run use a JotForm **API key** (an account-level credential)
  stored as an environment variable instead of the personal OAuth connector.
- Run it once manually ("Run now") and confirm a clean digest before trusting the schedule.
- Confirm the routine is actually created and enabled at claude.ai/code/routines — it does not
  exist until someone sets it up there; this skill file alone does not schedule anything.

