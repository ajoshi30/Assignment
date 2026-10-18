Lead automation (Python, standard library only)

Takes each website-form lead, cleans it, de-duplicates it, tags it Hot/Warm/Cold with a drafted reply, writes it to the leads "sheet", pings on Hot leads, and records every problem with a reason.

Run it (also see "Deliverables" below)
bash
python main.py --reset                 # replays data/incoming_leads.json, writes ./output
python -m unittest discover -s tests   # 13 tests
python server.py                       # optional: real webhook at POST :8080/lead

Python 3.9+, no installs. Re-running is safe: already-processed submission ids are skipped.

Choices (the brief said to tell you)
Thing	What I used	Why
Leads sheet	CSV (output/leads.csv)	No Google auth needed. Input files are never modified.
LLM	Rules-based stub by default - not an LLM. Real: --provider anthropic or openai	No API key needed to demo. The real prompt is leadflow/prompts.py. The real providers are written but untested against a live API (no key here).
Notifications	output/notifications.log, or Slack if SLACK_WEBHOOK_URL is set	Zero setup.
Webhook	main.py loops the file; server.py is a real endpoint	Same Pipeline.process() behind both.

Real LLM: export ANTHROPIC_API_KEY=...; python main.py --reset --provider anthropic (free/local option: LLM_PROVIDER=openai LLM_BASE_URL=http://localhost:11434/v1 LLM_MODEL=llama3.1).

Pipeline (per lead)
Clean: trim/collapse whitespace, lowercase email, fix name casing (JAMES o'CONNOR -> James O'Connor, accents kept), date -> YYYY-MM-DD, phone pulled out of the message.
Validate email: missing or malformed -> exceptions.csv (needs_review), with a suggested fix only when unambiguous (sara@@x.se). I never guess ben.carter@gmail -> .com.
Dedupe: key = lowercased email with any +tag removed. One index covers the existing sheet and leads added earlier in the batch. Name+company is a secondary signal that only adds a possible_duplicate_of flag (never auto-merges).
LLM: tag + spam flag + injection flag + reasoning + draft, strict JSON, validated, retried with backoff.
Write to leads.csv (original 7 columns kept; added phone, message, draft_reply, flags, notes, submission_id, notified).
Notify for Hot.
Duplicates are not just dropped

A duplicate is logged in duplicates.csv and its message is appended to the original lead's notes; the original tag is never overwritten. If a duplicate is itself Hot (S-15: existing Warm lead L-1008 now says "ready to go ahead, send a proposal") a notification still fires, suggesting an upgrade. Silently discarding that would be the exact "missed lead" failure this project exists to remove.

Safety
Prompt injection (S-10): lead text is wrapped as untrusted data in the prompt; a regex check runs regardless of provider; an injection can only force a lead to Cold, never raise it; the draft is replaced by a safe template; it's logged as flagged_added.
Draft guardrail: any draft containing a %, discount, price, guarantee etc. is replaced. Nothing is ever auto-sent; drafts are for a human.
LLM output is schema-validated; an invalid tag counts as a failure, not a guess.
Making problems visible

output/exceptions.csv has one row per problem, with outcome and reason: needs_review, rejected_spam, llm_error, flagged_added, notification_failed, unexpected_error. Spam is rejected visibly, not silently deleted. A test asserts every submission appears in exactly one of the sheet, duplicates or exceptions. run_summary.md is a per-lead table of the run; run.log has the detail. LLM or Slack outages never lose a lead.

Results on the 15 sample leads (stub)

Added 6 (Hot: S-01, S-09) | Duplicates 4 (S-02, S-03 existing; S-14 in-batch; S-15 existing + Hot re-engagement) | Needs review 4 (S-04/S-05 no email, S-06/S-07 bad email) | Spam 1 (S-11).

Assumptions
A lead with no valid email can't be replied to or reliably de-duplicated, so it is held for human review rather than added (S-04, S-05, S-06, S-07). The phone number in S-05 is extracted into the review row.
"Duplicate" means same email after lowercasing, trimming and removing a +tag. Same name + company with a different email is only flagged, never auto-merged.
Hot = ready to buy now (explicit go-ahead/proposal request, or budget plus near-term start). Warm = real need, no commitment. Cold = vague, exploratory, far-future or empty. Spam = unsolicited marketing.
created_at is date-only to match the existing sheet; source is website for all form leads.
Drafts are never sent automatically. A human reviews every reply, and drafts may not contain prices, discounts or commitments.
Today's date, the sheet columns and the Hot rubric are my choices; all are easy to change (cleaning.py, store.py, prompts.py).
What happens when something fails
Failure	Behaviour
Missing / malformed email	Not added. Row in exceptions.csv (needs_review) with the reason and a suggested fix if unambiguous.
LLM down, times out, or returns invalid JSON	3 attempts with backoff, then llm_error in exceptions.csv. Lead is not added and not lost; re-processing is possible.
Model output tries to promise a discount/price	Draft replaced with a safe template, flagged draft_replaced_by_guardrail.
Prompt injection in the message	Forced to Cold, safe draft, flagged and logged as flagged_added.
Slack/notification fails	Lead is still added; notified=FAILED, flag notification_failed, row in exceptions.csv.
Spam	Not added; logged as rejected_spam so it can be audited.
Any unexpected exception	Caught per lead, logged as unexpected_error; the rest of the batch continues.
Crash mid-write	Sheet is written atomically (temp file + rename), so it is never half-written.
How I used AI tools

I used Claude (Anthropic) as a coding assistant: I gave it the brief and the two data files, and it generated the initial code, tests and README draft. I then ran it, read through the code and tests, checked the outcome for each of the 15 sample leads, and made the design decisions listed above (for example holding no-email leads for review and notifying on Hot duplicates). The pipeline's runtime LLM is separate: by default a rules-based stub, optionally a real model via --provider.

Known limits / next steps
CSV is single-writer; server.py serialises with a lock. For real use: Google Sheets/Postgres with a unique constraint on the email key.
Exceptions live in a CSV file; in production they'd go to a Slack channel or a tab in the sheet.
The stub's keyword rules are a demo; the tagging rubric in the prompt is the real logic and should be tuned on labelled leads.
Leads with no email (S-04, S-05) wait for a human. A nice extension is a reply asking for contact details, or a phone callback task for S-05.
Non-English drafting depends on the real LLM; the stub only has English/German templates.
Deliverables
Final leads sheet: output/leads.csv (18 rows: 12 original + 6 new). Also duplicates.csv, exceptions.csv, notifications.log, run_summary.md.
Never commit secrets: ANTHROPIC_API_KEY, LLM_API_KEY and SLACK_WEBHOOK_URL are read from environment variables only.
