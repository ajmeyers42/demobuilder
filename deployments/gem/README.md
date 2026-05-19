# Gemini Gem — Discovery

The Discovery Gem gives SDRs and AEs a no-setup tool for parsing discovery notes
and qualifying opportunities. They paste notes or attach a file; the Gem returns
a qualification recommendation, customer confirmation document, open questions, and
SA handoff brief in one response.

---

## Creating the Gem (First Time)

1. Go to [gemini.google.com](https://gemini.google.com) and sign in with your Google Workspace account
2. In the left sidebar, click **Gems** → **New Gem**
3. **Name:** `Discovery`
4. **Description:** paste from `gem-config.json` → `description`
5. **Instructions:** paste the full contents of `system-prompt.md` (everything in that file)
6. Click **Save**
7. Share the Gem link with your SDR/AE team

> Gems are private by default. To share with your team, use the share icon in the Gem
> header and share with specific people or a Google Group.

---

## Updating the Gem

**How to tell if an update is needed:** check the sync dates in
`skills/warp-discovery/components.md`. If either upstream source has been updated
more recently than the last time you refreshed the Gem, the Gem needs to be updated.

**To apply an update:**

1. Open the Gem at [gemini.google.com](https://gemini.google.com) → **Gems** → **Discovery**
2. Click **Edit** (pencil icon)
3. Select all text in the **Instructions** field and delete it
4. Paste the full contents of `deployments/gem/system-prompt.md` (the entire file)
5. Click **Save**

No code to deploy. No tokens to rotate. Takes under two minutes.

---

## How SDRs/AEs Use It

**Option A — Paste notes:**

1. Open the Discovery Gem
2. Type or paste: `Here are my notes from today's call with [Company Name]:` followed by the notes
3. The Gem asks to confirm the opportunity name, then returns all four outputs

**Option B — Attach a file:**

1. Open the Discovery Gem
2. Click the attachment icon and upload a `.txt`, `.pdf`, or `.docx` from a transcription service
3. Send the message — the Gem reads the file, confirms the opportunity name, then processes

**Copying the customer confirmation document to Word or Google Docs:**

The Gem wraps the confirmation document between `--- START CUSTOMER DOCUMENT ---` and
`--- END CUSTOMER DOCUMENT ---` markers. To get it into a shareable file:

1. Select everything between (not including) those two marker lines
2. Copy and paste into a new Word document or Google Doc
3. The headers, bullets, and table will carry over — light formatting cleanup may be needed
4. Delete or re-style the marker lines if they accidentally come through

**Follow-up calls:**
If there's a second call with new information, paste the new notes in the same Gem conversation
and say "Here's what we learned on the follow-up call." The Gem will update the qualification
and flag what changed.

---

## What the Gem Produces

| Output | Who reads it | What to do with it |
| --- | --- | --- |
| **Opportunity Qualification** (MEDDPIC score + PROCEED / CONTINUE DISCOVERY / NOT QUALIFIED) | AE + SDR | Review together; escalate or park the deal |
| **SA Build Readiness** (demo direction + sizing estimate sub-scores) | Solutions Architect | Tells the SA what they can start on immediately vs. what is still needed |
| Customer confirmation document | Customer | Copy, review, send via email |
| Open questions | AE + SDR (internal) | Prioritized follow-up agenda for next call |
| SA handoff brief | Solutions Architect | Forward to SA before ideation conversation |

---

## Troubleshooting

**Gem isn't using the right format** — verify the full contents of `system-prompt.md`
were pasted (including the output templates at the bottom). The output format instructions
are at the end of the file.

**Gem is making up content not in the notes** — this is a hallucination. Remind it:
"Only use what was in the notes I provided. If something wasn't stated, mark it as not
captured." The skill instructions already say this, but the reminder helps.

**File attachment not working** — Gemini supports `.txt`, `.pdf`, and `.docx`. Other formats
(.doc, .odt, .pages) should be converted before uploading. Google Docs can export to any of
the supported formats.
