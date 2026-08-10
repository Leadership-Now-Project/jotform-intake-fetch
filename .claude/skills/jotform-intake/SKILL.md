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
afternoon window (see Step 0). If it's a Monday morning run, also apply the Monday catch-up
window described in Step 0 — the routine doesn't run on Saturday or Sunday, so a plain
"yesterday" lookback would silently skip weekend submissions.

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
  - **Morning run, Tuesday–Friday:** `[yesterday, today]` — picks up anything submitted after
    yesterday's afternoon run.
  - **Morning run, Monday (catch-up window):** `[last Friday's date, today]`. The routine only
    runs weekdays, so a plain "yesterday" (Sunday) window would silently miss anything submitted
    Saturday or Sunday. Widen the start date back to the prior Friday so the whole weekend is
    covered.
  - **Any run following a gap longer than the above** (a run was missed, the routine was paused,
    or this is the first run after setup): widen the start date back to the last date you can
    confirm a run actually completed — don't assume a 1-day or 3-day gap if the actual gap is
    longer.
- `filter.is_filtered_by_rules`: `false`
- `limit`: `50`

`date_filter` is day-granularity only, so overlapping windows across runs — including the wider
Monday and catch-up windows — are expected and safe: the dedup check below (the "Onboarding
Survey" custom field) skips anyone already fully processed, so a wider window costs a few extra
lookups, never a duplicate write.

Record each submission's originating form_id alongside its parsed fields — Step 5 branches on
it (new-member vs. existing-member survey) when writing Member Since.

### Dedup — the "Onboarding Survey Complete" custom field is the processed marker
For each submission returned, before processing:
1. Search DonorDock by the submission's email (`search_contacts`).
2. If a contact exists, call `get_contact_custom_fields` and check whether
   **Onboarding Survey Complete** (FieldId 26, Boolean) is already `true`. If so, this member's
   data was already merged in an earlier run — **skip entirely.** Do not re-write fields, do not
   re-run Steps 5–7d.
3. Otherwise, treat it as new or incomplete and run Steps 1–7d for that submission.

**Note:** the **Onboarding Flow** badge is applied by a separate DonorDock automation, not by
this skill, and may lag behind this run. Don't use it for dedup — use the Onboarding Survey
Complete custom field, which this skill controls directly and sets only once a submission is
fully processed (see Step 7d).

**Historical bug, now fixed:** this skill previously wrote and checked a field literally called
"Onboarding Survey" — that label never existed in DonorDock, so every dedup check silently found
nothing and every completion write silently failed. The real field is "Onboarding Survey
Complete" (FieldId 26). This means dedup has likely never actually worked in any past run — every
scheduled run may have been eligible to reprocess every previously-onboarded member. Worth
checking run history for evidence of repeat processing on the same contacts.

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
| Member Tier | Member tier field on form | DonorDock custom field "Member Tier" — **FieldId unverified, see Known Limitations** |
| Political Affiliation | "What is your current political party affiliation, if any?" | FieldId 11 (Select) |
| Pay to Play | "Do you have Pay to Play Restrictions?" (Yes/No) | FieldId 20 (Boolean) — "Yes" → checkbox checked (true), "No" → unchecked (false) |
| Archetype Potential | "Archetype Potential" field on form | DonorDock custom field "Archetype Potential", FieldId 39, Select (confirmed distinct from FieldId 31 "Archetype", which this skill never writes) **and** Klaviyo profile property `archetype` |
| Influence Style Signal | "Influence Style Signal" field on form | DonorDock custom field "Influence Style Signal", FieldId 37 (Select) |
| Network Strength Signal | "Network Strength Signal" field on form | DonorDock custom field "Network Strength Signal", FieldId 38 (Select) |
| LinkedIn URL | "What is your LinkedIn URL?" | contact.linkedInUsername — real, writable top-level field; full URLs are auto-normalized to the bare handle (see Step 4) |
| Copy Assistant? | "Should we copy your Assistant/Scheduler on all communications?" | DonorDock custom field "Copy Assistant?" (Boolean) — **FieldId unverified, see Known Limitations** |
| Assistant Name | "What is your Assistant/Scheduler's name?" | DonorDock custom field "Assistant/Scheduler Name", FieldId 34 (Text) |
| Assistant Email | "What is your Assistant/Scheduler's email address?" | DonorDock custom field "Assistant/Scheduler Email Address", FieldId 12 (Text) — FieldId 35 is a likely-duplicate field of the same name; FieldId 12 is the one this skill writes |
| Priority Interests | "Which of these Leadership Now 2025-26 key priorities..." (multi-select) | Badges **and** Klaviyo profile property `priority_focus_areas` |
| Contribution Modes | "How are you best positioned to support this work right now?" (multi-select) | Badges |
| Policy Expertise | "In which areas do you have policy expertise..." | Badges |
| Member Notes | "Anything else you'd like us to know..." | contact.Description |
| Submission Date | "Submission Date" | Used for Cohort derivation |

Only write Assistant/Scheduler Name / Assistant Email / Copy Assistant? if the member actually
entered an Assistant/Scheduler on the form — leave these blank otherwise.

**Address parsing:** The address arrives as one string like "555 California St San Francisco, CA, 94104".
Parse it as: everything before the last comma-separated city block is Address1, then City, State abbreviation, Zip. Country defaults to "United States".

**Multi-select columns:** JotForm spreads multi-select answers across repeated columns with the same header. Collect all non-empty values from all "How are you best positioned..." columns as the Contribution Modes list. (In fetch mode these arrive as a single list — collect all values either way.)

---

## Step 2 — Check for Duplicate

Before creating, search DonorDock for the member by email or name using search_contacts. If they already exist, skip create_contact and use update_contact instead with their contactId.

**Fetch mode:** if the existing contact's **Onboarding Survey Complete** custom field is already
`true`, this submission's data was already merged in a prior run — skip it entirely (see Step 0
dedup). Only fall through to update_contact for a contact that exists but hasn't had survey data
merged yet.

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

**LinkedIn URL:**
```
update_contact({ contactId, linkedInUsername: "..." })
```
`linkedInUsername` is a real, writable top-level contact field — pass the full submitted URL and
it's auto-normalized to the bare handle. This used to be flaky via the API; that's now fixed, so
write it whenever a LinkedIn URL was submitted. Still confirm on read-back once per run just in
case, but don't expect it to fail — if it ever does, add it to the "Needs manual entry" list in
Step 8 rather than blocking the rest of the run.

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

**Skip this write entirely for `EXISTING_MEMBER_FORM_ID` submissions**, same reasoning as Member
Since below: Cohort marks which intake class a member originally joined in. Re-deriving it from
a V2 check-in survey's submission date would reassign a member's cohort to whenever they happened
to resubmit the survey, not when they actually joined — e.g. a member onboarded in Q1 would
incorrectly show as Q3 just for filling out a mid-year check-in. Only write Cohort on
`ONBOARDING_FORM_ID` submissions.

### Political Affiliation — valid option strings

Do not pass the survey answer as-is when it's "Independent / Unaffiliated" — DonorDock's actual
option for this is `Independent` (confirmed by testing; `Independent / Unaffiliated` and
`Unaffiliated` are both rejected with `"Invalid option selected"`). Map the survey's
"Independent / Unaffiliated" answer to `Independent` before writing. Confirmed valid DonorDock
values: `Democratic`, `Republican`, `Independent`. `Prefer not to say` and `Other` are what the
form offers but have not been verified against DonorDock's actual picklist — verify on read-back
the first time either comes through, and update this list once confirmed.

Select-field writes generally resolve correctly server-side now (label strings are matched
reliably) — the Independent/Unaffiliated mismatch above was a one-off wrong label in this skill's
own docs, not a sign that Select fields are broadly unreliable. Still, any option string not
listed above (or in a field's known-valid list elsewhere in this doc) should be treated as
unverified until confirmed on read-back once.

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
Every other field in this step (Chapter, Political Affiliation, Pay to Play, Archetype,
Membership Status, etc.) still writes normally regardless of which form the submission came
from — **except Cohort**, which has the same `ONBOARDING_FORM_ID`-only exception, for the same
reason (see the Cohort section above).

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
    "Member Tier": "...",                       // FieldId unverified — see Known Limitations
    "Cohort": "Q2",                              // FieldId 32, Select — derived, ONBOARDING_FORM_ID only, omit for EXISTING_MEMBER_FORM_ID
    "Archetype Potential": "...",               // FieldId 39, Select
    "Influence Style Signal": "...",            // FieldId 37, Select
    "Network Strength Signal": "...",           // FieldId 38, Select
    "Assistant/Scheduler Name": "...",          // FieldId 34, Text — only if provided
    "Copy Assistant?": true,                    // only if provided, Boolean — FieldId unverified, see Known Limitations
    "Membership Status": "Active - Current",    // set on every processed submission
  }
})
```

Omit any field with a blank/null value (except Membership Status, which always gets set).
Verify the response shows `confirmed: true` for each field. Any field with `confirmed: false`
goes to the manual entry list in Step 8.

Do **not** include `"Onboarding Survey Complete"` in this call — it is set separately, and last,
in Step 7d, once everything else below has succeeded.

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

### 7a — Verify first, then subscribe and enroll only if needed

`subscribe_profile_to_marketing` requires interactive user confirmation on every call — it will
not go through unattended (e.g. during a scheduled fetch-mode run with no one watching). Members
are frequently already subscribed/enrolled by an earlier step in the onboarding process before
this skill ever runs, so don't call it blindly — check first and only call it when it's actually
needed:

1. Call `get_profiles` with `filter: 'equals(email,"{email}")'` and
   `additional_fields_profile: ["subscriptions"]`.
2. If a profile exists and `subscriptions.email.marketing.consent` is already `"SUBSCRIBED"`,
   treat 7a as done — record the returned profile ID and skip straight to 7b. (This does not by
   itself confirm list membership; if you need to be certain they're on the Leadership Now
   Members list specifically, cross-check with `get_lists` and add via `add_profiles_to_list` if
   missing — that call does not require interactive confirmation.)
3. Otherwise (no profile, or not subscribed), call `subscribe_profile_to_marketing` with:
   - `email` — from survey
   - `subscriptions.email.marketing.consent` = "SUBSCRIBED"
   - List relationship: **Leadership Now Members** (list ID: `YqM4pm`)

   This call creates the profile if it doesn't exist, updates it if it does, and enrolls them
   in the list — all in one shot. It also returns the Klaviyo profile ID needed for 7b. In an
   unattended run, if this call comes back asking for confirmation that can't be obtained, don't
   block the rest of the run on it — note it in the "Needs manual entry" list (Step 8) and
   continue; DonorDock writes and badges are independent of this step.

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
  fields: { "Onboarding Survey Complete": true }   // FieldId 26, Boolean
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
Survey Complete (checked), Archetype Potential, Influence Style Signal, Network Strength Signal,
LinkedIn, and Assistant details when provided.

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

- **Run:** {date} {morning|afternoon} run {— Monday catch-up window back to {date} | — catch-up window after a {N}-day gap, if applicable}
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

- **LinkedIn, employer/job title, Archetype Potential, Influence Style Signal, Network Strength
  Signal, and Select-field/new-field writes in general are all confirmed working** via the MCP as
  of the FieldId remap below. Nothing on this list needs a workaround anymore — write them
  directly per Steps 4–5.
- **FieldId 33+ was never a real platform limitation — it was a gap in this skill's own field
  catalog**, and every field name below was wrong or unmapped as a result. Full remap, confirmed
  by direct probing:

  | FieldId | Label | DataType |
  |---|---|---|
  | 14 | Past Engagements | Select |
  | 25 | Social Media Engagement | Select |
  | 26 | Onboarding Survey Complete | Boolean |
  | 27, 28 | (archived) | — rejected on write |
  | 31 | Archetype | Select |
  | 33 | Concierge Comms Flag (Yes/No) | Boolean |
  | 34 | Assistant/Scheduler Name | Text |
  | 35 | Assistant/Scheduler Email — likely duplicate of FieldId 12 | Text |
  | 37 | Influence Style Signal | Select |
  | 38 | Network Strength Signal | Select |
  | 39 | Archetype Potential — confirmed distinct from FieldId 31 | Select |

  The field set ends at 39 (FieldId 40 tested and confirmed not to exist).

- **The dedup marker was the wrong field name — now fixed, but treat past runs as unreliable.**
  This skill used to write/check a field called "Onboarding Survey", which never existed; the
  real field is "Onboarding Survey Complete" (FieldId 26), now used throughout Steps 0/2/5/7d/8.
  Every write to the old name was silently falling into `unresolved`, meaning dedup has likely
  never actually worked — every past scheduled run may have been eligible to reprocess every
  previously-onboarded member. If duplicate processing matters (e.g. duplicate badges, overwritten
  manual edits), check recent run history for repeat writes to the same contacts before trusting
  historical digests.
- **FieldId 35 vs FieldId 12 (both "Assistant/Scheduler Email") — needs a canonical decision.**
  This skill writes FieldId 12 ("Assistant/Scheduler Email Address"), which was already known and
  in active use before the remap. FieldId 35 surfaced as a likely duplicate. Don't switch to 35
  without confirming which one DonorDock actually treats as canonical — for now, keep using 12.
- **"Member Tier" and "Copy Assistant?" do not exist as writable fields — confirmed, not just
  unverified.** Tested directly against a clean contact: both labels come back `unresolved`
  (unknown field name), and probing the one remaining gap in the catalog, FieldId 36, also came
  back unresolved. The full field set (1–39, with 27/28 archived and 36 apparently never
  assigned) is now accounted for, and neither field is in it. Don't attempt these writes going
  forward — go straight to the "Needs manual entry" list in Step 8 with the value, and note that
  DonorDock/staff need to clarify what these should map to (possibly "Membership Type", FieldId 2,
  for Member Tier, and "Concierge Comms Flag (Yes/No)", FieldId 33, for Copy Assistant? — both
  unconfirmed guesses based on thematic overlap, not verified equivalences, so don't write to
  those FieldIds under these labels without confirming first).
- **Archived "Use_Chapter" (FieldId 21) blocker is CONFIRMED still broken — retested after the
  broader MCP fixes landed, no change.** Kol Chu Birke and Lauren Barra Rourke both still reject
  every real custom-field write with the same behavior as before: hard error under
  `mergeStrategy: 'merge'`, silent `confirmed: false` under `'replace'`. This includes the
  corrected "Onboarding Survey Complete" field itself — it could not be set on either contact,
  so both remain stuck without a completion marker regardless of the field-name fix. The broader
  "Select-field/new-field writes confirmed working" fix above does not cover this specific
  interaction. This is still an open DonorDock data issue on these two (and possibly other
  long-tenured) contacts; `add_badge` remains unaffected. If a contact hits this, flag it in the
  "Needs manual entry" list (Step 8) with the specific field(s) and value(s) that wouldn't save,
  and note that DonorDock needs to clear that contact's archived `Use_Chapter` value directly
  (not via the API) before automated writes — including the completion marker — will work on it
  again.

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

