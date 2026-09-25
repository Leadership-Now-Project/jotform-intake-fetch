---
name: "jotform-intake"
description: "Process Leadership Now Project (LNP) member onboarding JotForm survey responses (the 2026 Member Onboarding Survey, form 260216535465052 — the only form this skill uses) and create or update contacts in DonorDock via the MCP. Primarily for new members, but also usable for prospects who already have a partial DonorDock record (event RSVPs, briefing attendees) — enriches the existing record instead of requiring a from-scratch contact. Use whenever a user pastes, shares, or uploads a JotForm submission — CSV row, pasted text, or one member's answers — or says \"process this new member\", \"add this JotForm response\", \"onboard this member\", or \"create a contact from this survey\". Also runs in FETCH MODE on a schedule (e.g. twice daily, weekdays) or when asked to \"pull new submissions\", \"check JotForm for new members\", or \"run the intake\": pulls new submissions via list_submissions and processes each one. Use this skill rather than manually calling individual MCP tools for JotForm intake."
---

# JotForm → DonorDock Intake

You are processing Leadership Now Project member onboarding survey submissions into DonorDock.
Work through the steps below in order. Be methodical — each step depends on the previous one.

The core job of this skill: read every JotForm answer, map it to the correct DonorDock
destination (core contact field, custom field, or badge), and write it accurately — including
derived logic (Chapter from ZIP) — so DonorDock always reflects
what the member actually entered on the form.

This skill uses a single form: the **2026 Member Onboarding Survey** (`260216535465052`,
https://form.jotform.com/260216535465052). Its primary use is genuinely new members completing
it for the first time, but it's equally valid for a prospect who already has a partial DonorDock
record — someone who RSVP'd to an event or attended a briefing and has a sparse contact record
with just a name and email, say. In that case this skill enriches the existing record (Steps 2–3
already handle "contact exists, update instead of create") rather than needing a from-scratch
contact. No other form is in scope for this skill.

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

**Confirmed drift (2026-09-11):** the Onboarding form's Address and Secondary Address State
fields were converted from free-text boxes to constrained "American States" dropdowns.
Submissions now arrive with the **full state name** (e.g. "New Jersey") instead of the two-letter
abbreviation members used to type freehand. DonorDock stores state as a two-letter abbreviation,
and this skill's own Chapter-derivation table (Step 5) and Klaviyo `location.region` mapping
(Step 7b) both key off the abbreviation — see the normalization step in Step 1 below.

**Confirmed rebuild (2026-09-03, mapped 2026-09-10, applied to this skill 2026-09-14):** the
Onboarding form (`260216535465052`) was substantially rebuilt on Sept 3 — questions moved, four
scoring fields were deleted, and the old single "how are you positioned to support this work"
question was replaced by four separate per-priority questions. This rewrite of the skill is the
fix for that rebuild, based on the reconciliation Marie wrote up ("Survey → DonorDock Map v3,"
Sept 10) plus a DonorDock custom-field catalog audit done the same week. See each step below for
what specifically changed; the short version is in "What changed in this rewrite" just below.

---

## What changed in this rewrite (2026-09-14) — read this once, then skip to the steps

- **Graduate/Undergraduate Year were being written to each other's fields.** The college block now
  comes before the graduate block on the live form, so the old "first Year Graduated occurrence =
  grad, second = undergrad" position-based logic had them backwards. Fixed to read by question
  label instead (Step 1).
- **Contribution Modes is now four separate questions**, not one repeated-header question. Collect
  all four and dedupe into a `Currency:` badge set (Step 1, Step 6) — replacing the old
  `Contribution:` badges per the Sept 9 membership-ops decision.
- **This skill no longer writes Archetype at all** — not Archetype Potential (custom field 39), not
  Influence Style Signal (37), not Network Strength Signal (38). The questions that fed them were
  deleted in the Sept 3 rebuild, and per direct instruction this skill is out of the archetype
  business entirely going forward; DonorDock's Archetype fields are Marie's scoring model's
  territory now, not this skill's.
- **Member Tier and Mobile Phone / Date of Birth mappings are removed.** None of these exist on
  the live form; keeping them in the mapping table was dead code.
- **Nine question labels drifted** (same meaning, new wording) — see the label-drift table in
  Step 1.
- **Two new fields are added**: Who Referred You (→ custom fields 18 and 22) and the Networks/
  Affiliations question (→ auto-created `Affiliation: [Name]` badges).
- **Prefill Email is now the primary match key** in Step 2, and is also what gets written to
  `contact.Email` — not the visible Primary Email answer. They're the same value in the normal
  case (Klaviyo hands the member a link with their email locked into the hidden field); Prefill
  Email is just the authoritative source.
- **Two new DonorDock date fields** — custom field 40 (Onboarding survey completed on) and custom
  field 41 (Most recent member survey) — replace the Boolean "Onboarding Survey Complete" custom
  field (26) as the mechanism that decides whether a submission is new-or-already-processed. Field
  26 itself is untouched — still written exactly as it always was, just no longer read for that
  decision (see Step 0).
- **Fetch mode pulls only from the Onboarding form — no other form is in scope for this skill.**
  An earlier version of this rewrite also supported an on-demand mode targeting a "V2 2026
  Existing Member Survey" (`262105169919159`) — that form and mode have been dropped entirely per
  direct instruction. This skill now has exactly one input form, used both for genuinely new
  members and for enriching prospects who already have a partial DonorDock record (event RSVPs,
  briefing attendees).

---

## What changed in this update (2026-09-25)

- **DonorDock contact ID is now the primary match key (Step 2)**, ahead of Prefill Email. It
  arrives as a new hidden, locked field on the form — **"Prefill DonorDock ID"** (qid 118, unique
  name `prefillDonordock`) — populated the same way Prefill Email is: from the personalized link
  Klaviyo sends. When it's present, use it to fetch the contact directly
  (`get_contact_profile`/`get_contact_custom_fields` by contactId) instead of searching by email —
  it's an exact key, so there's no duplicate-matching ambiguity to resolve. Prefill Email remains
  the fallback for a submission with no contact ID (a cold submission with no personalized link),
  and Primary Email remains the fallback after that. See Step 1 and Step 2.
- **New "Willing to host" badge.** A "Yes" answer to the space-hosting question (qid 114, unique
  name `leadershipNow`) adds the `Willing to host` badge to the contact. See Step 1 and Step 6.
- **Affiliation badges now check for a near-duplicate before creating a new one** — e.g. an "Other"
  answer of "New Jersey Democrats" should attach to an existing `Affiliation: NJ Democrats` badge
  rather than spawning a second, near-identical one. See Step 6.
- **Two new real-time Slack alerts to #onboarding-survey-tasks**, distinct from the existing
  end-of-run digest (Step 9): one whenever "Any other academic affiliations?" is filled in (for
  manual entry — this skill still does not write it to DonorDock), and one whenever a submission
  hits an error or a case the skill isn't confident how to handle. See the new Step 6b and the
  "Slack alerts" subsection under Step 8/9.

---

## How This Skill Runs — Two Modes

**Fetch mode (default, scheduled).** The normal weekday operation. Pulls only from the **2026
Member Onboarding Survey** (`260216535465052`, https://form.jotform.com/260216535465052) — the
only form this skill ever reads from — start at Step 0, run Steps 1–7d for each *new-or-updated*
submission, finish with the batch digest in Step 9.

**Single-submission mode (override).** A human pastes or uploads one specific submission. Skip
Step 0 and start at Step 1, processing that one record through Steps 1–8. This is still always a
submission from the same Onboarding form's question set — if someone pastes answers from a
different survey entirely, say so rather than assuming the Step 1 mapping table applies.

Routing rule: a submission pasted or uploaded in the message → single-submission mode. Everything
else (scheduled run, "run the intake", "check JotForm", any ambiguous invocation with no pasted
submission) → fetch mode.

**Which window when not told (fetch mode):** if the run isn't labeled morning or afternoon, decide
by the current time in America/Chicago — before 12:00 CT use the morning window, otherwise the
afternoon window (see Step 0). The routine runs Monday–Friday only; Monday's morning run
additionally has to cover everything since Friday afternoon's run — both the full weekend (which
the routine never touches) and the tail end of Friday itself — see Step 0's Monday window for the
exact range.

---

## Step 0 — Fetch Mode (Scheduled Runs)

### Configuration
- `ONBOARDING_FORM_ID`: `260216535465052` — **2026 Member Onboarding Survey**
  (https://form.jotform.com/260216535465052). This is the only form this skill uses, in either
  mode — there is no second configured form.
- Run cadence (fetch mode only): twice daily, **Monday through Friday**, 07:00 and 16:00
  America/Chicago (CT). No weekend runs (see "Scheduling" section at the end).
- `KNOWN_BLOCKED_CONTACTS`: contacts confirmed stuck on DonorDock's archived-custom-field write
  block (see "Known Limitations" below). Self-maintained by Step 3 going forward: add a contactId
  here the first time it hits this block, remove it once someone clears the archived field in the
  DonorDock UI and a retest succeeds.
- `PENDING_MANUAL_ENTRY`: contacts held back from a clean completion because a specific field wrote
  to a known-wrong value that this skill's tools couldn't correct (see Step 7d's "known-wrong
  value" case). Track as `{ contactId, field, expectedValue, flaggedOn }`. Self-maintained: Step 7d
  adds an entry here; Step 9's reconciliation pass checks and clears entries once a human confirms
  the fix.
- `SLACK_ALERT_CHANNEL`: `onboarding-survey-tasks` (confirmed live private Slack channel,
  created 2026-09-25). Destination for both new real-time alerts (Step 6b's academic-affiliations
  flag and Step 8/9's error/questionable-response flag) as well as, going forward, the Step 9 batch
  digest — post the digest here too rather than to a separately-configured channel, since this
  channel was set up specifically for this routine.

If `ONBOARDING_FORM_ID` is blank, resolve it once with the JotForm `search` tool by form title
("2026 Member Onboarding Survey") and record the numeric form ID in the config above so future
runs are deterministic.

### Pull recent submissions
Call `list_submissions` with:
- `form_ids`: `[ONBOARDING_FORM_ID]` — always, this is the only form.
- `filter.date_filter`:
  - **Afternoon run:** `[today, today]`
  - **Morning run, Tuesday–Friday:** `[yesterday, today]` — picks up anything submitted after
    yesterday's afternoon run.
  - **Morning run, Monday (catch-up window):** `[last Friday's date, today]`. The routine only
    runs weekdays, so a plain "yesterday" (Sunday) window would silently miss anything submitted
    Saturday or Sunday — and starting at Friday's date, not Saturday's, also re-covers whatever
    came in after Friday afternoon's own run cut off. This is the "weekend plus the rest of
    Friday afternoon" window.
  - **Any run following a gap longer than the above** (a run was missed, the routine was paused,
    or this is the first run after setup): widen the start date back to the last date you can
    confirm a run actually completed — don't assume a 1-day or 3-day gap if the actual gap is
    longer.
- `filter.is_filtered_by_rules`: `false`
- `limit`: `50`

`date_filter` is day-granularity only, so overlapping windows across runs — including the wider
Monday window — are expected and safe: the dedup check below (comparing against custom field 41)
skips anyone already fully processed for that submission, so a wider window costs a few extra
lookups, never a duplicate write.

### Dedup — compare the submission date against custom field 41, not a Boolean

DonorDock has two purpose-built date fields for this now: **custom field 40, "Onboarding survey
completed on"** (Date), and **custom field 41, "Most recent member survey"** (Date). Both exist in
the field catalog as of 2026-09-14 but neither has been observed holding a value on a live record
yet — confirm on the first real write-then-read this skill does against them, per the general
sparse-payload caveat in Known Limitations.

For each submission returned, before processing:
1. Search DonorDock by the submission's email — use **Prefill Email** first if present, falling
   back to the visible Primary Email answer (`search_contacts`). See Step 1 for why these are
   expected to be the same value.
2. If no contact exists, treat as new and process (Steps 1–7d).
3. If a contact exists, call `get_contact_custom_fields` and read the current value of **custom
   field 41 (Most recent member survey)**.
   - If field 41 is unset, or the submission's own date is strictly newer than field 41's stored
     value: this submission hasn't been merged yet — process it (Steps 1–7d).
   - If the submission's date is the same as or older than field 41's stored value: this exact
     submission (or a later one) was already merged in a prior run — **skip entirely.**

This replaces the old Boolean-based dedup, which checked "Onboarding Survey Complete" (custom
field 26) and skipped forever once it was `true` — which meant a member's future resubmission of
this same form would have been silently skipped too, since nothing ever unchecked that box.
Comparing submission dates against field 41 fixes that: a genuinely new, later submission always
has a newer date than what's stored, so it always gets processed, whether the member was
onboarded years ago or is resubmitting.

**Field 26 itself is unchanged.** It's still written exactly as before — see Step 7d — this rewrite
just stopped using it as the dedup trigger. Don't remove it or repurpose it further without a
separate, explicit decision to do so.

### Field source in fetch mode
Submissions from `list_submissions` arrive as structured answers (question label → answer),
not CSV columns. The field mapping in Step 1 still applies — just read each field from the
submission's answers by its question label instead of a CSV column. Multi-select answers come
back as a list rather than repeated columns. Since fetch mode only ever pulls from
`ONBOARDING_FORM_ID`, every submission it sees uses the exact Step 1 question set — no
branching on which form a submission came from is needed anywhere downstream.

Process each new submission through Steps 1–7d, collecting per-member results, then produce the
batch digest in Step 9.

---

## Step 1 — Parse the Input

The input may arrive as:
- A pasted CSV row (with or without headers)
- Free-form text summarizing a member's answers
- A single row extracted from the JotForm CSV export
- A structured submission from `list_submissions` (fetch mode)

Extract these fields (leave blank if not present). Where the live question label has drifted from
an older name this skill used to look for, both are listed — always match on the **current**
label.

| Field | JotForm Question (current label) | DonorDock / Klaviyo Target |
|---|---|---|
| DonorDock Contact ID | "Prefill DonorDock ID" (hidden, locked field, qid 118, unique name `prefillDonordock`) | **New primary match key (Step 2)** — when present, resolves the contact directly with no search needed |
| First Name | Split from "Full Name" on first space | contact.FirstName |
| Last Name | Everything after the first space in "Full Name" | contact.LastName |
| Primary Email | "Primary Email" | Used only as a fallback match key / fallback for contact.Email if Prefill Email is blank |
| Prefill Email | "Prefill Email" (hidden, locked field) | **Primary match key (Step 2)** and **the value actually written to contact.Email.** Expected to equal Primary Email in the normal flow — a member reaching the form through a Klaviyo prefill link always carries this; treat it as authoritative over whatever they typed in the visible Primary Email box |
| Phone | "Main Phone Number" *(was "Phone Number")* | contact.MainPhone |
| Address | "Address" — parse into street, city, state, zip (state needs normalizing — see below) | contact address fields (state as two-letter abbreviation) |
| Employer | "Who is your current employer?" | contact.Employer + contact.JobTitle |
| Job Title | "What is your job title?" | contact.JobTitle |
| Graduate Academic Affiliation | "Where did you attend graduate school?" *(was "What is your Graduate Academic Affiliation?")* | FieldId 3 (Text) |
| Graduate Year | The "Year Graduated" paired with the graduate-school question above — **match by the question it's attached to, not by column position** (see "Graduate/undergrad year fix" below) | FieldId 4 (Number) |
| Undergraduate Academic Affiliation | "Where did you attend college or university?" *(was "What is your Undergraduate Academic Affiliation?")* | FieldId 5 (Text) |
| Undergraduate Year | The "Year Graduated" paired with the college/university question above | FieldId 6 (Number) |
| Industry | "What industry do you work?" *(was "...work in?" — the live form is missing the "in", not a typo you should silently correct — match the string as it actually appears)* | FieldId 29 (Select) |
| Sector | "What sector do you work in?" *(was "What sector?")* | FieldId 30 (Select) |
| Political Affiliation | "What is your current political party affiliation, if any?" | FieldId 11 (Select) |
| Pay to Play | "Do you have Pay to Play Restrictions?" (Yes / No / I'm not sure) | FieldId 20 (Boolean) — "Yes" → checkbox checked (true); "No", "I'm not sure", or blank → omit the field entirely, leave DonorDock unset |
| LinkedIn URL | "What is your LinkedIn URL?" | contact.linkedInUsername — full URLs are auto-normalized to the bare handle (see Step 4) |
| Copy Assistant? | "Should we copy an executive assistant/scheduler on communications?" *(was "Should we copy your Assistant/Scheduler on all communications?")* | DonorDock custom field "Copy Assistant?" (Boolean) — FieldId unverified, see Known Limitations |
| Assistant Name | "Executive Assistant/Scheduler name:" *(was "What is your Assistant/Scheduler's name?" — note the trailing colon is part of the live label)* | DonorDock custom field "Assistant/Scheduler Name", FieldId 34 (Text) |
| Assistant Email | "Executive Assistant/Scheduler email: " *(was "What is your Assistant/Scheduler's email address?" — trailing colon and space are part of the live label)* | DonorDock custom field "Assistant/Scheduler Email Address", FieldId 12 (Text) — FieldId 35 is a likely-duplicate field of the same name; FieldId 12 is the one this skill writes |
| Who referred you to Leadership Now Project? | "Who referred you to Leadership Now Project?" *(new mapping — this question existed before but was never wired up)* | Custom field 18, "Referred By" (Text) — **and** custom field 22, "Member Referral?" (Boolean) set to `true` whenever this answer is non-blank. Both writes, always together |
| Priority Interests | "Which of these Leadership Now key priorities are you most eager to engage in?" *(was "...2025-26 key priorities...")* (multi-select) | Badges **and** Klaviyo profile property `priority_focus_areas` |
| Contribution / Currency questions | Four separate questions, one each for Expansion, Risk, Balance of Power, and Talent (all phrased "How are you best positioned to support this work right now?" with a per-priority suffix) — **this replaced the old single repeated-header question; collect all four keys, not one shared header** | Badges — see "Currency badges" under Step 6 |
| Policy Expertise | "Where does your policy expertise overlap with our Business Plan for America topic areas?" *(was "In which areas do you have policy expertise...")* | Badges |
| Are you an active member of any of the following networks? | "Are you an active member of any of the following networks?" (multi-select, including two "(please specify)" options and a free-text Other) | Badges — `Affiliation: [Name]`, see Step 6 |
| Member Notes | "Anything else you'd like us to know..." | contact.Description |
| Willing to host | "Leadership Now is often looking for space to host member convenings and events. Could we reach out to ask you about hosting at your home, office, or a social club?" (qid 114, unique name `leadershipNow`, Yes/No) | Badge — `Willing to host`, added only on "Yes" (see Step 6) |
| Submission Date | "Submission Date" | Used for custom fields 40/41 |

Only write Assistant/Scheduler Name / Assistant Email / Copy Assistant? if the member actually
entered an Assistant/Scheduler on the form — leave these blank otherwise. Same rule for Who
Referred You — only write fields 18/22 if there's an actual answer.

**Fields the live form has that this skill still does not map** (no established DonorDock
destination, or destination not yet decided — don't invent one):
- "What is your current employment status?" — no destination decided as of this writing.
- Any additional academic affiliations beyond the two questions above — no destination decided.

If a real DonorDock destination for either shows up later, add it here rather than guessing one.

**Graduate/undergrad year fix — read this carefully.** The Sept 3 rebuild moved the college block
ahead of the graduate-school block on the live form. If your extraction logic reads "the first
Year Graduated value on the form" as the graduate year and "the second" as the undergraduate year
— as this skill used to — the two years land in each other's fields, because undergrad's question
now comes first. Fix: always associate each "Year Graduated" answer with the school question it
sits directly beneath/after on the submission, never by ordinal position across the whole
submission. In fetch mode this is straightforward since `list_submissions` returns
answers keyed by question, not by column; in single-submission mode (a pasted CSV row), watch for
a CSV export that still uses generic repeated headers like "Year Graduated" / "Year Graduated.1"
and confirm which one is actually which before writing — don't assume the CSV column order matches
the old (now-wrong) assumption either.

**Prefill Email vs. Primary Email — there should be no conflict to resolve.** These are expected
to carry the same value: Klaviyo sends a prefill link with the member's known email locked into
the hidden Prefill Email field, and the visible Primary Email question is prefilled with (and
normally left as) that same value. Write contact.Email from Prefill Email when it's present; fall
back to Primary Email only for a submission that has no Prefill Email at all (e.g. someone who
filled out the form cold, not via a personalized link — a genuinely new prospect). If the two ever
do disagree on a real submission, that's worth a flag in Step 8/9's summary rather than silently
picking one — it likely means the member edited the visible field after clicking through, and
someone should confirm which value is actually current.

**Address parsing:** The address arrives as one string like "555 California St San Francisco, CA, 94104".
Parse it as: everything before the last comma-separated city block is Address1, then City, State, Zip. Country defaults to "United States".

**State must be normalized to a two-letter abbreviation.** As of 2026-09-11 the Address and
Secondary Address State fields on the Onboarding form are constrained dropdowns that submit the
**full state name** (e.g. "New Jersey"), not an abbreviation. DonorDock stores state as the
two-letter code, and this skill's Chapter table (Step 5) and Klaviyo `location.region` mapping
(Step 7b) both expect the abbreviation — so convert immediately after parsing, before the value
is used anywhere downstream. If a submission instead arrives with an abbreviation already (an
older submission predating this change), pass the value through unchanged; only convert when the
parsed value is a full state name.

| Full Name | Abbr | Full Name | Abbr | Full Name | Abbr |
|---|---|---|---|---|---|
| Alabama | AL | Kentucky | KY | North Dakota | ND |
| Alaska | AK | Louisiana | LA | Ohio | OH |
| Arizona | AZ | Maine | ME | Oklahoma | OK |
| Arkansas | AR | Maryland | MD | Oregon | OR |
| California | CA | Massachusetts | MA | Pennsylvania | PA |
| Colorado | CO | Michigan | MI | Rhode Island | RI |
| Connecticut | CT | Minnesota | MN | South Carolina | SC |
| Delaware | DE | Mississippi | MS | South Dakota | SD |
| District of Columbia | DC | Missouri | MO | Tennessee | TN |
| Florida | FL | Montana | MT | Texas | TX |
| Georgia | GA | Nebraska | NE | Utah | UT |
| Hawaii | HI | Nevada | NV | Vermont | VT |
| Idaho | ID | New Hampshire | NH | Virginia | VA |
| Illinois | IL | New Jersey | NJ | Washington | WA |
| Indiana | IN | New Mexico | NM | West Virginia | WV |
| Iowa | IA | New York | NY | Wisconsin | WI |
| Kansas | KS | North Carolina | NC | Wyoming | WY |

This is the exact inverse of the `expand_state()` mapping the `jotform-prefill` skill uses to go
the other direction (DonorDock abbreviation → full name, for prefill links). Keep both tables in
sync if a state name/abbreviation pairing ever needs correcting, and apply the same normalization
to the Secondary Address State answer if it's ever mapped to a DonorDock or Klaviyo field.

**Multi-select columns:** JotForm spreads multi-select answers across repeated columns with the same header. Collect all non-empty values from all repeated-header columns (in fetch mode these arrive as a single list — collect all values either way). The Currency questions are the one exception to watch for — they are four *distinct* questions, not one repeated header, so collecting "everything sharing this header" will silently miss three of the four.

---

## Step 2 — Check for Duplicate

Before creating, resolve the contact. **Match priority, as of 2026-09-25: DonorDock Contact ID
first (the new "Prefill DonorDock ID" field) — falling back to Prefill Email if the contact ID is
blank, and to Primary Email if both are blank.** Use name only as a secondary check — if the
matched contact's name has changed (see Step 3's name-change handling), that's a flag to review,
not a reason to reject the match; the contact ID (and, one step down, email via the Klaviyo-locked
Prefill Email field) is the primary key precisely because names change and these mostly don't.

**When the Contact ID is present, skip `search_contacts` entirely** — call `get_contact_profile`
(and `get_contact_custom_fields`) directly with that contactId. This is an exact key, not a
fuzzy match, so there's no duplicate-record ambiguity to resolve the way there is with an
email-only match. If the contactId comes back not-found (a stale or malformed value — the record
was deleted, or the link is old), fall back to the email-based match below and flag the mismatch
in Step 8 rather than silently failing.

**Email-based fallback (only when Contact ID is blank), unchanged from before:**

If they already exist, skip create_contact and use update_contact instead with their contactId.

**Fetch mode:** use the custom-field-41 comparison from Step 0's dedup section — don't duplicate
that logic here, just confirm you're holding the contactId Step 0 already resolved.

Be aware that DonorDock carries substantial duplication, and duplicates often split one person's
data across records — one has the current email, another has the employer. If several records
plausibly match, prefer the one with the most complete custom fields and real giving history, and
flag the duplication in the Step 8 summary so someone can merge them at the source.

**A match with sparse existing data is expected, not a red flag.** A contact search may return a
record that only has a name and email — a prior event RSVP or briefing attendee who was added to
DonorDock without going through this form. Treat that the same as any other existing-contact
match: update it (Step 3) rather than creating a duplicate, and let Steps 4–6 fill in whatever the
survey now provides that the sparse record was missing.

---

## Step 3 — Create (or Update) the Contact

If the resolved contactId is in `KNOWN_BLOCKED_CONTACTS` (Step 0 config), skip straight to Step 6
— DonorDock will reject every write to this contact's fields until someone clears its archived
custom field in the UI (see "Known Limitations" below). Badges (`add_badge`) and Klaviyo (Step 7) are unaffected by
this, so still run those; just note in Step 8 that this member's custom-field data is pending a
manual DonorDock fix and wasn't attempted.

If a Contact ID was resolved in Step 2, there is no `create_contact` call to make at all — the
contact already exists; go straight to `update_contact` with whatever fields are changing.

Otherwise (no Contact ID, and no email match either — a genuinely new person), call `create_contact`
with:
- firstName, lastName
- email — **from Prefill Email, falling back to Primary Email** (see Step 1)
- mainPhone
- address1, city, stateOrProvince (the two-letter abbreviation from the Step 1 normalization,
  never the raw full state name), postalCode, country

If the contact already exists, call `update_contact` with only the fields that are changing.

After creation, note the returned contactId — you will need it for all subsequent steps.

**If this call returns `blockedByArchivedField`:** DonorDock is rejecting every write to this
contact because of a stale value in an archived custom field. Stop attempting DonorDock field
writes for this contact — skip the rest of Step 3/4/5 and go straight to Step 6 (badges, which
are unaffected). Add the contactId to `KNOWN_BLOCKED_CONTACTS` in Step 0's config if not already
there, and flag it in Step 8's summary with the guidance DonorDock returned.

**Owner:** no action needed here — OwnerId defaults to Lauren Barra Rourke automatically on
the DonorDock side.

**Name changes — check on every submission for an existing contact**, not just first-time ones,
now that fetch mode's dedup no longer hinges on a one-time completion flag. Split the
survey's "Full Name" the same way as Step 1 (first space rule) and compare the result against the
existing contact's current `FirstName`/`LastName`. If either differs, include the new `firstName`/
`lastName` in the `update_contact` call — don't assume the DonorDock record is already current
just because the contact matched on email. Flag any detected name change explicitly in the Step 8
summary (old value → new value) rather than silently folding it into the general "fields updated"
list.

---

## Step 4 — Write Employer, Job Title, Description, and LinkedIn

Skip this step entirely for a contact already routed to Step 6 in Step 3 (archived-field block).

Make these `update_contact` calls if the values are non-empty:

**Employer + job title:**
```
update_employment({ contactId, employer: "...", jobTitle: "..." })
```
Use `update_employment`, not `update_contact`, for these two fields — it's the tool that targets
DonorDock's distinct "Employment" block correctly and fuzzy-matches/links the Organization record.

**Do not write Employer to a custom field.** FieldId 15 is a legacy custom field also labeled
"Employer" that predates this skill and holds stale data on some older contacts (confirmed
populated on 4 of 5 sampled contacts in the 2026-09 field audit — a bigger latent data-quality
issue than "a few stale older records," worth flagging to Marie/Lauren as a cleanup item, separate
from anything this skill needs to change). Never include `"Employer"` as a key in a
`set_contact_custom_fields` call.

**Description** (if "Anything else you'd like us to know" has content):
```
update_contact({ contactId, description: "..." })
```

**LinkedIn URL:**
```
update_contact({ contactId, linkedInUsername: "..." })
```
`linkedInUsername` is a real, writable top-level contact field — pass the full submitted URL and
it's auto-normalized to the bare handle.

---

## Step 5 — Write Custom Fields

Call `set_contact_custom_fields` with all mappable custom fields below. Select fields accept
label strings directly; pass values by field name — no FieldId lookup needed unless a write
comes back `confirmed: false`.

**Always re-derive Chapter fresh from the current submission — never carry forward an
existing DonorDock value as-is,** even when reconstructing a full field payload for
`mergeStrategy: 'replace'`. An existing record's derived fields may be stale, wrong, or from
before this mapping table was corrected; only trust a fresh derivation from the submission's own
data.

### Chapter — derive from state and ZIP

Use the normalized two-letter abbreviation from Step 1 here, not the raw submitted value.

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

**"National" is a real, live Chapter value on existing records** that this ZIP/state table has no
rule for producing — this is a known, still-open gap (flagged by Marie, not yet resolved as of
2026-09-14; see Known Limitations). Don't invent a trigger condition for it — leave Chapter blank
per the "all others" rule above rather than guessing when National should apply, and don't let a
re-derivation overwrite an existing National value with something else just because this table
can't reproduce it (the "always re-derive fresh" rule above has this one carve-out: if the
existing value is exactly `National` and the ZIP/state table would otherwise leave it blank,
leave the existing value alone instead of blanking it).

### Political Affiliation — valid option strings

**Do not pass the survey's raw "Independent / Unaffiliated" text as-is — it will be rejected.**
DonorDock validates Select values by exact string match, whitespace included:

- `Independent / Unaffiliated` (space before **and** after the slash — the raw survey text) →
  **rejected**.
- `Independent/ Unaffiliated` (no space before the slash, one space after) → **valid, confirmed**
  — this is DonorDock's real option, verified against a live record.
- `Independent` (bare word, no slash) → **also valid, confirmed** — a separate, distinct option.

**Map the survey's "Independent / Unaffiliated" answer to the exact string
`Independent/ Unaffiliated`** (no space before the slash). Confirmed valid DonorDock values:
`Democratic`, `Republican`, `Independent`, `Independent/ Unaffiliated`. `Prefer not to say` and
`Other` are what the form offers but have not been verified against DonorDock's actual picklist —
verify on read-back the first time either comes through.

This exact-whitespace sensitivity applies to every Select field this skill writes — DonorDock has
no API endpoint that returns a field's valid option list, so there's no way to validate a label
before sending it. If a Select write ever comes back with `"Invalid option selected for X: <value>"`,
check for a stray or missing space around a slash or other punctuation before assuming the option
doesn't exist at all.

### Pay to Play — Boolean

The JotForm question offers three options: "Yes", "No", "I'm not sure". Only "Yes" writes
anything — include `"Pay to Play Restrictions?": true` in the `set_contact_custom_fields` call.
For "No", "I'm not sure", or a blank answer, **omit the field entirely** from the call — do not
write `false`. DonorDock reporting/filtering treats "no value" and "false" differently, and only a
genuine "Yes" answer is a real, confirmed restriction worth recording.

### Membership Status — set on every processed submission

Whenever this skill merges survey data for a member (new contact or update), set the
**Membership Status** custom field to `"Active - Current"`.

### Member Since — from submission date, write once

Write the JotForm **Submission Date** as Member Since (FieldId 17) the first time a contact is
ever processed through this skill. Write it only when Member Since isn't already set — check the
current value via `get_contact_custom_fields` first. **Skip this write for a contact that already
has a Member Since date** — a resubmission (including from a prospect who already has a partial
DonorDock record with a Member Since date set some other way) should never overwrite an existing
join date with today's date.

### Onboarding survey completed on — custom field 40 (Date), write once

Set custom field 40 the first time a contact is ever fully processed through this skill — same
write-once timing as Member Since (a member's completion date should reflect when they actually
completed the survey, not when they resubmit it later). Write it only when it isn't already set —
check the current value via `get_contact_custom_fields` before writing, the same way you'd check
for any other "write once" field. If field 40 already holds a value, leave it alone.

### Most recent member survey — custom field 41 (Date), write every time

Set custom field 41 to the submission's own date on **every** processed submission, every time —
including resubmissions. This is the field Step 0's dedup logic reads on the next run, so it must
be current after every successful process, not just the first.

### Referred By and Member Referral? — custom fields 18 and 22

If "Who referred you to Leadership Now Project?" has a non-blank answer, write both:
```
"Referred By": "...",         // FieldId 18, Text
"Member Referral?": true,     // FieldId 22, Boolean
```
Always together — never set 22 without 18, and don't write either if the question was left blank.

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
    "Chapter": "Bay Area",                      // FieldId 16, Select — derived from ZIP; see the National carve-out above
    "Member Since": "2026-01-18",               // FieldId 17, Date — write once only, see above
    "Referred By": "...",                       // FieldId 18, Text — only if provided
    "Pay to Play Restrictions?": true,          // FieldId 20, Boolean — only if the answer was exactly "Yes"
    "Member Referral?": true,                   // FieldId 22, Boolean — only if Referred By was provided, always paired with it
    "Industry": "...",                          // FieldId 29, Select
    "Sector": "...",                            // FieldId 30, Select
    "Assistant/Scheduler Name": "...",          // FieldId 34, Text — only if provided
    "Copy Assistant?": true,                    // only if provided, Boolean — FieldId unverified, see Known Limitations
    "Onboarding survey completed on": "2026-09-14", // FieldId 40, Date — write once only, see above
    "Most recent member survey": "2026-09-14",  // FieldId 41, Date — write every processed submission
    "Membership Status": "Active - Current",    // set on every processed submission
  }
})
```

Note what's **not** in this payload anymore: Archetype Potential (39), Influence Style Signal (37),
Network Strength Signal (38), and Member Tier (no confirmed field). This skill does not write any
of them.

Omit any field with a blank/null value (except Membership Status, which always gets set, and field
41, which always gets set on every processed submission). Pay to Play Restrictions? is a special
case of this same rule — treat "No" and "I'm not sure" as equivalent to blank (see Pay to Play
section above), not just a literally-empty answer.

Verify the response shows `confirmed: true` for each field. Any field with `confirmed: false`
goes to the manual entry list in Step 8.

Do **not** include `"Onboarding Survey Complete"` (field 26) in this call — it is set separately,
and last, in Step 7d, once everything else below has succeeded, exactly as before.

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

Only Priority: Policy, Priority: Talent, and Priority: Expand existed in DonorDock as of the last
audit — Priority: Risk and Priority: Balance of Power get created on first write. That's expected
and fine; just watch for a typo forking the set on that first write.

### Willing to host — from the space-hosting question

If the answer to "Leadership Now is often looking for space to host member convenings and events.
Could we reach out to ask you about hosting at your home, office, or a social club?" (qid 114,
`leadershipNow`) is **"Yes"**, add the badge `Willing to host`. On "No" or a blank answer, add
nothing — don't add an inverse/negative badge, and don't remove an existing `Willing to host`
badge on a resubmission that now answers "No" (that's a change worth a human glance, not an
automatic badge removal — flag it in Step 8 if a member's answer flips from Yes to No on a later
submission).

### Currency badges (replaces "Contribution:") — from the four priority-support questions

The old single "How are you best positioned to support this work right now?" question was
replaced by four separate questions (Expansion, Risk, Balance of Power, Talent — see Step 1).
Collect all four answers, dedupe, and map into this badge set:

`Currency: Voice` · `Currency: Network` · `Currency: Financial` · `Currency: Time` ·
`Currency: Expertise`

**The exact answer-option text for these four new questions has not been read off the live form
yet** — the keyword matches below are carried forward from the old Contribution Modes question and
are a reasonable starting point, not a confirmed mapping:

| Answer contains... | Badge |
|---|---|
| "financial resources" or "Invest or donate" | Currency: Financial |
| "time and energy" or "volunteer" or "nominating" | Currency: Time |

Before trusting this for a real batch, load the live form and read the actual option text on all
four priority-support questions, then extend this table to cover Voice, Network, and Expertise —
none of which had a confirmed source string under the old single-question design. Flag any answer
that doesn't match a known keyword rather than silently dropping it or guessing a badge.

### Policy Expertise → "Policy: [X]" badges

For each policy area listed, add a badge: "Policy: Immigration", "Policy: Workforce/AI",
"Policy: Housing", etc.

### Affiliation badges — from "Are you an active member of any of the following networks?"

For every option the member selects on this question, add `Affiliation: [Name]` — auto-created,
no manual review queue. This includes the fixed-option answers, the two "(please specify)"
options (use the specify text as `[Name]`), and a free-text Other answer (title-case it before
badging). Examples:

- `YPO / WPO` → `Affiliation: YPO / WPO`
- `Aspen Institute / Aspen Global Leadership Network` → `Affiliation: Aspen Institute / Aspen Global Leadership Network`
- `Council on Foreign Relations` → `Affiliation: Council on Foreign Relations` (rename the one
  existing bare `Council on Foreign Relations` badge to this format if you encounter it)
- `Milken Institute` → `Affiliation: Milken Institute`
- `World Economic Forum` → `Affiliation: World Economic Forum`
- `Industry Association / Chamber of Commerce (please specify)` with specify text "US Chamber of
  Commerce" → `Affiliation: US Chamber of Commerce`
- `University Alumni Board or advisory group (please specify)` with specify text "MIT Alumni
  Association" → `Affiliation: MIT Alumni Association`
- Other, free text "Milken Young Leaders Circle" → `Affiliation: Milken Young Leaders Circle`

**Before creating a new `Affiliation: [Name]` badge, check for a near-duplicate that already
exists (2026-09-25).** DonorDock has no fuzzy-search on badge names, so pull the current badge list
(`list_badges`, or `get_contact_profile`'s badges if that's cheaper) and compare the candidate name
against every existing `Affiliation: *` badge using this normalization, in order:

1. **Case- and whitespace-insensitive exact match** on the candidate vs. an existing badge's name
   (after trimming and collapsing internal whitespace) — if equal, use the existing badge, don't
   create a new one.
2. **State-abbreviation expansion** — if the candidate or an existing badge contains a two-letter
   token that matches a US state abbreviation (use the same 50-state table as Step 1's Chapter/state
   normalization), expand it to the full state name before re-comparing. This is exactly the "NJ
   Democrats" vs. "New Jersey Democrats" case: both normalize to "New Jersey Democrats" and match.
3. **Token containment** — if every significant word (ignore "the", "of", "and", punctuation) in the
   shorter name appears, in any order, in the longer name, treat them as the same badge (e.g.
   "Aspen Global Leadership Network" vs. "Aspen Institute / Aspen Global Leadership Network").
4. If none of the above matches anything, create the new badge as usual.

When a match is found under 2 or 3, **attach the badge using its existing exact name** — don't
rename the existing badge to the new submission's spelling, and don't create a second badge just
because the wording differs. Flag the merge in Step 8 ("matched '<submitted text>' to existing
badge '<existing name>'") so a human can sanity-check it, since this is a heuristic, not an exact
match, and an overly aggressive merge (e.g. two genuinely different regional chapters of the same
national org) is a worse outcome than an occasional near-duplicate badge. If two existing badges
both plausibly match, don't guess — flag it in Step 8 and skip creating/attaching either until a
human resolves which one is correct.

**Do not add an "Onboarding Flow" badge here.** That badge is applied automatically by a
separate DonorDock automation once its own conditions are met on the contact record — this
skill does not add it and should not depend on it being present at the time this run finishes.

---

## Step 6b — Slack alert: "Any other academic affiliations?" (2026-09-25)

This skill still does not have a DonorDock destination for "Any other academic affiliations?" (qid
93) — see Step 1's "does not map" list. Rather than silently dropping it, **whenever this answer is
non-blank, post an alert to the `onboarding-survey-tasks` Slack channel** (`SLACK_ALERT_CHANNEL`,
Step 0) with enough detail for a human to enter it manually:

```
New academic affiliation to enter manually
Member: {firstName} {lastName} — {DonorDock link}
Answer: "{the raw text}"
Likely field: the new Academic Affiliations custom fields (FieldId 51 "Secondary Graduate Academic
Affiliation" / FieldId 52 "Secondary Undergraduate Academic Affiliation") — pick whichever applies,
or split across both if the answer names more than one affiliation.
```

**Do not write this to DonorDock automatically** — the whole point of this alert is that a human
reviews and enters it, since this skill still doesn't have a confirmed, confident mapping from this
free-text answer to one of the two custom fields. Post the alert whether the submission otherwise
succeeds or fails — an unmapped answer is worth flagging on every occurrence, not just when
something else also goes wrong. In single-submission mode, post it the same way (don't hold it for
a fetch-mode-only digest).

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

1. Call `get_profiles` with `filter: 'equals(email,"{email}")'` (use the same email this skill
   wrote to contact.Email — Prefill Email, falling back to Primary Email) and
   `additional_fields_profile: ["subscriptions"]`.
2. If a profile exists and `subscriptions.email.marketing.consent` is already `"SUBSCRIBED"`,
   treat 7a as done — record the returned profile ID and skip straight to 7b. (This does not by
   itself confirm list membership; if you need to be certain they're on the Leadership Now
   Members list specifically, cross-check with `get_lists` and add via `add_profiles_to_list` if
   missing — that call does not require interactive confirmation.)
3. Otherwise (no profile, or not subscribed), call `subscribe_profile_to_marketing` with:
   - `email` — from survey (Prefill Email / Primary Email, as above)
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
| `location.region` | Normalized state abbreviation (see the Step 1 State normalization table) |
| `location.zip` | From survey |
| `location.country` | "United States" (default) |
| `properties.priority_focus_areas` | Priority Interests list from survey (array of the selected priority labels) |
| `external_id` | The DonorDock contactId resolved in Step 2/3 — set this on every profile write so a future prefill link for this member can carry `prefillDonordock` (see the `jotform-prefill` skill's "Downstream" section). Not a `properties.*` key — `external_id` is a top-level Klaviyo profile field. |

**`properties.archetype` is no longer written** — there is no survey-sourced archetype value
anymore (see "What changed in this rewrite"). If Klaviyo segmentation still needs an archetype
property, it has to be sourced from DonorDock's own Archetype field by a separate process, not
from this skill's Klaviyo write.

### 7c — Enrollment notes

- `subscribe_profile_to_marketing` must always be called **before** `update_profile` — it
  returns the profile ID the latter requires.
- Klaviyo matches profiles by email — if the profile already exists it will update in place,
  not create a duplicate.
- Do **not** enroll in Day 1 flow here — that is triggered by the separate Enrollment Bridge
  Zap (dues paid in DonorDock → Klaviyo Day 1 flow), which Marie owns.

### 7d — Mark the submission complete

Only after Step 3/4 (core contact fields — name, address, phone, employer/job title, LinkedIn,
description) **and** Steps 5, 6, and 7 have all succeeded, make one final call:

```
set_contact_custom_fields({
  contactId,
  fields: { "Onboarding Survey Complete": true }   // FieldId 26, Boolean
})
```

This is unchanged from before — still written last, still only on full success, still the same
field. It is simply no longer what Step 0 reads to decide whether to reprocess a contact (see
Step 0's dedup section) — custom field 41 does that job now. Set field 26 last and only on full
success; if any earlier step failed outright, leave it unset so the next run retries the missing
pieces.

**Step 3/4 failures count too, not just 5/6/7.** A contact can have Steps 5–7 succeed fully while
a core field from Step 3/4 silently fails. Before this call, confirm on read-back that every Step
3/4 field you attempted to change actually persisted, not just that the write call didn't throw.
If any Step 3/4 field didn't persist, skip this call entirely and flag it in Step 8.

**A field that "succeeded" but holds a known-wrong value also blocks this marker.** Not every
problem shows up as a failed write — DonorDock's Boolean custom fields cannot be nulled via
`merge` (rejects null/empty string) and `replace` does not actually drop fields omitted from the
payload, so there is currently no way to correct a stuck Boolean through this skill's tools once
it happens. If a read-back shows a custom field holding a value that contradicts the submission's
actual answer, treat that the same as a failed write: do not set Onboarding Survey Complete, and
flag the specific field/discrepancy in Step 8 for manual correction in the DonorDock UI.

In fetch mode specifically, also add an entry to `PENDING_MANUAL_ENTRY` (Step 0 config)
— `{ contactId, field, expectedValue, flaggedOn: <date> }` — rather than only surfacing it in that
one run's digest and losing track of it afterward. Step 9's reconciliation pass checks this list
every run so the fix gets picked up whenever someone actually makes it.

---

## Step 8 — Output a Summary (per member)

Present a clean summary with:

**Contact created/updated** — name, contactId, and direct DonorDock link:
`https://leadershipnowproject1.donordock.com/donors/detail/{contactId}`

**Written successfully — DonorDock** — list every field, custom field, and badge saved,
including Membership Status ("Active - Current"), Onboarding Survey Complete (checked),
Most Recent Member Survey date, Onboarding Survey Completed On date (if this was the first
write), LinkedIn, Referred By/Member Referral (if provided), and Assistant details when provided.

**Written successfully — Klaviyo** — confirm:
- Profile fields written (name, location, priority focus areas)
- Enrolled in: Leadership Now Members (YqM4pm)

**Needs manual entry in DonorDock** — list every blocked field with the value so staff can paste it in:
- LinkedIn: {value} — only if the Step 4 write failed or didn't persist
- Any custom field that came back `confirmed: false`
- Any Prefill Email / Primary Email mismatch flagged in Step 1
- Any Contact ID that didn't resolve to a real record, requiring the email-based fallback (Step 2)
- Any "Any other academic affiliations?" answer — also Slack-alerted per Step 6b, list it here too
  so it's not solely dependent on someone having seen the Slack post
- Any Affiliation badge merge decision flagged in Step 6 (near-duplicate matched, or two plausible
  matches left unresolved)

The "Needs manual entry" section is just as important as the successes — staff rely on it to complete the intake.

In single-submission mode this is the final output. In fetch mode, collect each member's result
and roll them into the Step 9 digest.

---

## Step 9 — Batch Digest (Fetch Mode)

### 9a — Reconciliation pass: check PENDING_MANUAL_ENTRY before building the digest

Before pulling new submissions (or after — order doesn't matter, but do this every run, not just
when there's new activity), go through every entry in `PENDING_MANUAL_ENTRY` (Step 0 config):

1. Call `get_contact_custom_fields` for the contact and check the flagged `field`.
2. **If the current value now matches `expectedValue`:** don't clear the entry or set Onboarding
   Survey Complete automatically — a value matching by coincidence isn't the same as confirming a
   human actually went in and fixed it. Instead, add this contact to a "Pending confirmation"
   section in the digest (below), asking directly whether it was entered manually. Leave the
   `PENDING_MANUAL_ENTRY` entry in place until that confirmation comes back.
3. **If a later message in the conversation confirms a specific pending item**, then and only
   then: call `set_contact_custom_fields` with `{ "Onboarding Survey Complete": true }` for that
   contact, and remove the entry from `PENDING_MANUAL_ENTRY`.
4. **If the current value still doesn't match `expectedValue`:** leave the entry as-is, no digest
   mention needed unless it's been pending an unusually long time.

After processing all submissions in the run, output one digest:

- **Run:** {date} {morning|afternoon} run {— Monday catch-up window back to {date} | — catch-up
  window after a {N}-day gap, if applicable}
- **Pulled:** N submissions in window
- **New / processed:** N (broken out: brand-new contacts vs. resubmissions from an existing
  contact, since the field-41 dedup now legitimately processes both — and "existing contact" here
  includes prospects who already had a sparse DonorDock record before this submission)
- **Skipped (already current per field 41):** N
- **Per new/updated member:** name + DonorDock link + a one-line "needs manual entry" flag if any
- **Pending confirmation (manual entry check):** any contact from 9a whose flagged field now
  matches the expected value, awaiting a yes/no on whether it was entered manually
- **Anomalies to review:** any failed write, any submission that could not be parsed, any
  Prefill Email / Primary Email mismatch

Post this digest to `SLACK_ALERT_CHANNEL` (`onboarding-survey-tasks`, Step 0) via the Slack MCP
so the team has visibility without opening Claude. Keep it to the counts plus anything that needs
a human — not a wall of per-field detail.

### Real-time error / questionable-response alerts (2026-09-25) — separate from the digest above

In addition to the end-of-run digest, post an immediate Slack message to `SLACK_ALERT_CHANNEL` the
moment any of the following happens during a submission, in either mode — don't hold it for the
digest:

- A DonorDock write fails outright, or a `blockedByArchivedField` response is hit (Step 3).
- Any `set_contact_custom_fields` field comes back `confirmed: false`, or a read-back shows a
  value that contradicts the submission (Step 7d).
- A Select/dropdown value doesn't match any known option (a Political Affiliation, Industry,
  Sector, school, etc. that fails the exact-string match in Step 5).
- Prefill Email and Primary Email disagree (Step 1).
- Two existing Affiliation badges both plausibly match a candidate name, and the skill can't
  pick one (Step 6).
- Any other point where this skill genuinely isn't sure what the right action is — that
  uncertainty is itself worth a human's attention in real time, not just a mention buried in an
  end-of-run count.

Each alert should name the member (name + DonorDock link when a contactId is known), what
specifically is wrong or uncertain, and what this skill did as a result (skipped the field, left
Onboarding Survey Complete unset, proceeded with a fallback, etc.) — enough for someone to act on
the Slack message alone without re-opening the run.

---

## Known Limitations

- **Archetype is entirely out of scope for this skill as of 2026-09-14.** It does not write
  Archetype Potential (custom field 39), Influence Style Signal (37), or Network Strength Signal
  (38) — none of these have a source on the live form anymore, and DonorDock's Archetype fields
  are Marie's scoring model's territory now. If that changes, treat it as a new mapping decision,
  not a reversion to the old one.
- **Custom fields 40 and 41 are new and unconfirmed live.** Both exist in the field catalog
  ("Onboarding survey completed on" / "Most recent member survey", both Date) as of 2026-09-14,
  but neither has been observed holding a value on a real contact yet. Confirm on the first actual
  write-then-read this skill does against them. This also resolves an old contradiction: an
  earlier catalog audit had field 40 labeled "Last Updated Via MCP" with a note claiming it "was
  tested and confirmed not to exist" — both were wrong; 40 is a real, differently-purposed field.
- **"National" as a Chapter value has no derivation rule** — see Step 5. This is a known open item
  (Marie, Sept 9 doc), not yet resolved. Don't invent a ZIP/state trigger for it.
- **Currency badge keyword matching is unverified** for the four new priority-support questions —
  see Step 6. Read the live option text before trusting this for a real batch; Voice, Network, and
  Expertise currently have no confirmed source string at all.
- **FieldId 33+ was never a real platform limitation — it was a gap in this skill's own field
  catalog.** Full remap, confirmed by direct probing:

  | FieldId | Label | DataType |
  |---|---|---|
  | 14 | Past Engagements | Select |
  | 18 | Referred By | Text |
  | 22 | Member Referral? | Boolean |
  | 25 | Social Media Engagement | Select |
  | 26 | Onboarding Survey Complete | Boolean |
  | 27, 28 | (archived) | — rejected on write |
  | 31 | Archetype | Select — not written by this skill |
  | 33 | Concierge Comms Flag (Yes/No) | Boolean |
  | 34 | Assistant/Scheduler Name | Text |
  | 35 | Assistant/Scheduler Email — likely duplicate of FieldId 12 | Text |
  | 37 | Influence Style Signal | Select — not written by this skill |
  | 38 | Network Strength Signal | Select — not written by this skill |
  | 39 | Archetype Potential | Select — not written by this skill |
  | 40 | Onboarding survey completed on | Date — see above |
  | 41 | Most recent member survey | Date — see above |

- **The old Boolean-only dedup was a real bug, now fixed.** This skill used to check "Onboarding
  Survey Complete" (field 26) and skip forever once it was `true` — meaning a member's future
  resubmission of this form would have been silently skipped, since nothing ever unchecked that
  box. Field 41-based comparison (Step 0) fixes this. If duplicate or skipped processing matters
  historically, check run history from before 2026-09-14 for evidence of this failure mode.
- **A second form and an on-demand mode were dropped from this skill (2026-09-14).** An earlier
  version of this rewrite supported pointing a one-off run at a "V2 2026 Existing Member Survey"
  (`262105169919159`) as an on-demand alternative to the scheduled Onboarding-form pull. Both the
  form and the mode have been removed entirely per direct instruction — this skill now has exactly
  one input form and exactly two modes (fetch, single-submission). If a second survey needs to be
  supported again later, treat it as a fresh scoping decision, not a reversion.
- **FieldId 35 vs FieldId 12 (both "Assistant/Scheduler Email") — needs a canonical decision.**
  This skill writes FieldId 12, which was already known and in active use. FieldId 35 surfaced as
  a likely duplicate. Don't switch without confirming which one DonorDock treats as canonical.
- **"Copy Assistant?" FieldId is unverified.** Write it, then confirm on read-back before trusting
  the write, and flag to manual entry if the value doesn't persist.
- **Archived custom fields can block all writes to a contact — confirmed mechanism, actively
  detected by the MCP.** DonorDock rejects every write to a contact record while ANY archived
  custom field on that record still carries a non-empty value. Known archived FieldIds: 21
  (`Use_Chapter`), 27, 28. It can only be cleared by editing the record directly in the DonorDock
  UI. `update_contact` and `set_contact_custom_fields` both check for this proactively and return
  a structured `blockedByArchivedField` field instead of a generic error — see Step 3's handling.
- **No confirmed way to unset a Boolean custom field once it has any value.** `merge` rejects both
  `null` and `""` for a Boolean field, and `replace` does not actually drop omitted fields despite
  its own description. Once a Boolean custom field has been set incorrectly, only a manual edit in
  the DonorDock UI can clear it. Don't retry `replace` expecting different results — go straight to
  flagging it for manual correction (see Step 7d).
- **State needs normalization on intake — see Step 1.** The Onboarding form's State fields are a
  full-state-name dropdown; every downstream consumer of state expects the two-letter
  abbreviation. Convert at parse time, not later.
- **Employer disagreement between core contact field and legacy custom field 15 is bigger than
  previously documented** — populated on 4 of 5 sampled contacts as of the 2026-09 field audit,
  not just "a few older contacts." A data-cleanup item, not something this skill's code needs to
  change (it already only ever writes via `update_employment`, never to field 15).
- **Contact-ID matching (Step 2) is new and unconfirmed against a large batch as of 2026-09-25.**
  The "Prefill DonorDock ID" field exists on the live form and the mapping is in place, but this
  hasn't yet been exercised against a real submission carrying a populated value — confirm on the
  first live submission that actually arrives with a non-blank `prefillDonordock` before trusting
  it fully for a scheduled run.
- **Affiliation badge fuzzy-merge (Step 6) is a heuristic, not an exact-match system.** It will
  occasionally merge two badges that a human would have kept separate, or miss a genuine match
  worded in a way the three rules don't cover. Every merge is flagged in Step 8 specifically so
  this can be caught and corrected rather than silently accumulating.
- **The academic-affiliations and error/questionable-response Slack alerts (Step 6b, Step 9) are
  new as of 2026-09-25 and assume the `onboarding-survey-tasks` Slack channel and the Slack MCP
  connector are both available at run time.** If the Slack post itself fails, don't let that block
  the rest of the run — note the failure in that member's Step 8 summary instead so the underlying
  issue isn't lost even if the notification is.

---

## Scheduling: Running 2× per Day, Weekdays Only

This skill does not schedule itself. Wire the cadence with a **Claude Code Routine** (cloud —
runs on Anthropic's infrastructure, so it does not depend on anyone's laptop being on; a Cowork
scheduled task would not be reliable here):

1. Put this skill in the routine's repository (skills committed into the repo are available to
   the run).
2. Create a routine at claude.ai/code/routines with a self-contained prompt telling Claude to
   run the jotform-intake skill in **fetch mode** (the Onboarding form only — see "How This Skill
   Runs"), determining morning vs. afternoon from the current time in America/Chicago, and the
   Monday catch-up window per Step 0.
3. Attach the **JotForm, DonorDock, and Klaviyo** connectors (plus Slack if you want the digest
   posted).
4. Add **two scheduled triggers** on the same routine — 07:00 and 16:00 America/Chicago (CT),
   **Monday through Friday only.** No weekend firings.

Things to know before it goes live:
- Routines are in research preview, with daily run caps by plan tier (roughly 5 / 15 / 25 for
  Pro / Max / Team+Enterprise) — two runs/day, five days a week fits comfortably.
- A routine runs under the identity of whoever created it, so connector auth is that person's.
  Create it under a shared/service account if you want it org-owned rather than tied to one
  person. Alternatively, have the run use a JotForm **API key** (an account-level credential)
  stored as an environment variable instead of the personal OAuth connector.
- Run it once manually ("Run now") and confirm a clean digest before trusting the schedule.
- Confirm the routine is actually created and enabled at claude.ai/code/routines — it does not
  exist until someone sets it up there; this skill file alone does not schedule anything.
