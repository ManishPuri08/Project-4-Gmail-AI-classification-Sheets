# n8n Gmail AI Classification Assistant

> An AI agent that reads every new unread email, classifies it by intent, logs it to Google Sheets, and labels it back in Gmail — so inbox triage happens automatically instead of one email at a time.

## Problem & Goal

Every unread email needs a human to read it, decide what kind of email it is, and route it accordingly — a Sales enquiry, a Support request, a Complaint, a Billing issue, or something general. That first pass is repetitive and easy to fall behind on, and by the time someone gets to sorting the inbox, urgent items have already been sitting unnoticed. This workflow removes that lag: every new unread email is classified the moment it arrives, logged to a spreadsheet for a running record, and labeled in Gmail — so nothing needs manual sorting before a human can act on it.

## Architecture

**Gmail Trigger → Fetch → Agent (classify) → Log → Label**

1. **Gmail Trigger** — polls the inbox every minute for new unread mail.
2. **Get a Message** — retrieves the full message content (headers, sender, subject, body) for the triggering email.
3. **AI Agent (LangChain)** — given the sender and email content, classifies the email into one of five categories — `Sales`, `Support`, `Complaint`, `Billing`, `General` — and assigns a `Priority`.
4. **Structured Output Parser** — forces the agent's output into a consistent JSON shape (`category`, `priority`), so downstream nodes always receive predictable fields.
5. **OpenRouter Chat Model (GPT-4.1-nano)** — powers the agent's classification.
6. **Append Row in Sheet** — logs date, sender, subject, category, priority, and body to a Google Sheet.
7. **Get Many Labels → Add Label to Message** — looks up the matching Gmail label ID at runtime and applies it back onto the original message.

See `Project-4 Gmail → AI classification → Sheets.json` for the exact node configuration — import it into n8n to view the visual canvas.

## Tools & Integrations Used

- **n8n** (Gmail Trigger, Gmail, LangChain Agent, Structured Output Parser nodes)
- **OpenRouter API** — GPT-4.1-nano for classification
- **Gmail API** — OAuth2-connected for reading mail and managing labels
- **Google Sheets API** — OAuth2-connected order/classification log

## Setup Instructions

1. Import `Project-4 Gmail → AI classification → Sheets.json` into your n8n instance (Workflows → Import from File).
2. Set up credentials for:
   - **OpenRouter API** (your own API key)
   - **Gmail OAuth2** (connect your own Gmail account)
   - **Google Sheets OAuth2** (connect your own Google account)
3. In your Gmail account, create the five labels used by this flow: `Sales`, `Support`, `Complaint`, `Billing`, `General` (or adjust the agent's prompt to match your own label set).
4. Update the **Google Sheets** node's spreadsheet and sheet reference to point to your own log sheet.
5. Activate the workflow. New unread emails will be checked every minute automatically.

## Product Decisions

- **Log and label, don't auto-reply:** The workflow classifies, logs, and labels — it never drafts or sends anything. Keeps the blast radius of a misclassification low; a human still decides how to respond.
- **Structured output parser over free-text parsing:** Forcing the AI's response into a strict JSON schema (`category`, `priority`) makes the downstream Sheets and label actions reliable, instead of parsing loosely-formatted text.
- **Label classification via live lookup, not hardcoded IDs:** The workflow calls the Gmail "get labels" tool at runtime to fetch actual label IDs, rather than hardcoding them — making the flow portable across different Gmail accounts/label setups.
- **Fixed taxonomy with explicit definitions:** Each of the five categories is defined in the prompt (what counts as Sales vs. Support vs. Complaint, etc.), narrowing the model's decision space instead of leaving classification open-ended.

## How I Checked Accuracy

- **Sample emails across categories:** Sent test emails clearly representative of each category (a pricing enquiry, a how-to question, a complaint, an invoice issue, a miscellaneous message) to confirm the agent lands on the intended label rather than defaulting to `General`.
- **Execution log inspection:** Used n8n's execution history to step through each node's input/output after a test run — confirming the agent's structured output actually matched the schema (`category`, `priority`) instead of returning malformed or partial JSON.
- **End-to-end confirmation:** Checked the Google Sheet row and the Gmail label after each test rather than trusting a "successful execution" status alone — a run can report success while still logging the wrong category or failing to find a matching label.
- **Boundary/edge cases:** Tested ambiguous emails that could plausibly fit more than one category, to see how the agent breaks ties and where the category definitions in the prompt needed tightening.

## AI Evaluation

- Used an AI assistant (Claude) to review the workflow's node logic and connection structure — confirming the classification step sits upstream of both the Sheets write and the label lookup, and that no node referenced a stale or renamed node.
- Cross-checked the agent's prompt against the structured output schema and found a gap: `category` has explicit definitions, but `priority` doesn't — the model infers urgency without any defined rubric. Flagged as a fix for v2 rather than left unnoticed.
- Used AI-assisted review to sanity-check this README against the actual JSON — making sure every claimed behavior (categories, node order, credentials used) matches what's really configured in the workflow file, rather than describing an idealized version of it.

## Limitations & Next Steps

- **Undefined priority rubric:** `priority` is part of the output schema but has no defined levels or criteria in the prompt, unlike `category`. A v2 should define priority the same explicit way categories are defined.
- **Single message per run:** Only processes one new message at a time (`maxResults: 1`) — could be extended to batch-process multiple unread emails per trigger.
- **No fallback if a label doesn't exist:** If the Gmail account doesn't already have a label matching the AI's category, the label step silently finds nothing to apply — no error or fallback notification yet.
- **No confidence scoring or human review step:** Ambiguous or borderline emails are classified with the same confidence as clear-cut ones; a future version could add a threshold that routes uncertain cases to manual review.
- **No handling for attachments or threads:** Each email is treated as a standalone message, with no awareness of prior messages in the same thread.
