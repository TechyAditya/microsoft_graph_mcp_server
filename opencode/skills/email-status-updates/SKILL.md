---
name: email-status-updates
description: Use ONLY when the user asks an agent to send email status updates, notifications, or long-running-task checkpoints to their inbox via the Microsoft Graph MCP. Triggers - "email me", "notify me by email", "send a status email", "let me know via email", "ping me when done", running tasks longer than ~5 minutes where the user has stepped away. ALSO trigger after any compaction if `/tmp/opencode-email-threads/` is non-empty or the running summary mentions an active email thread - load this skill before sending the next update so the recovery ladder is followed. Threads every update for one task into a single Outlook conversation by reusing one Friendly Title across all sends, and survives session compactions by persisting the title to /tmp/opencode-email-threads/<slug>.txt (one file per task, slug derived from the title) and by recovering it via subject search of the inbox.
---

# Email Status Updates

You send status emails on the user's behalf via the `microsoft-graph` MCP server (already authenticated as `contact@adityasethi.dev`).

> **Post-compaction entry point.** If you wake into a task after a compaction and any of these are true — (a) `/tmp/opencode-email-threads/` exists and is non-empty, (b) the running summary mentions an email thread / status emails / `[STARTED]` etc., (c) the user references "the email thread" — **load this skill first, then walk the Recovery ladder below before sending anything**. Do not send a fresh first email from in-context memory.
>
> **Compaction breadcrumb (keep in summary, max ~15 words).** When summarizing for compaction, preserve only: `email-status-updates skill active; thread file at /tmp/opencode-email-threads/<slug>.txt`. That single line is enough to re-trigger this skill on the next turn — do not bloat the summary with the protocol itself, it lives here.

The user's design goals are non-negotiable:

1. **Inbox stays clean.** Each task = exactly one Outlook conversation thread. Never two threads for the same task.
2. **Identity is visible.** Every email shows which agent sent it (Sisyphus, Oracle, etc.) without changing the From address (which is fixed to `contact@adityasethi.dev`).
3. **Plus-addressing for routing.** All agent mail is sent To `contact+mcp@adityasethi.dev` so the user can filter agent traffic into a folder via Outlook rules.
4. **Survives compaction.** Mid-task, the conversation context can be summarized away — the protocol must still find the right thread.

## Format

| Field | Value |
| --- | --- |
| `to` | `["contact+mcp@adityasethi.dev"]` (plain string only — Graph rejects `"Name <addr>"` syntax) |
| `subject` | The Friendly Title, with NO status prefix and NO other decoration. Identical across every send for the task. |
| `htmlbody` | First paragraph **must** start with the status token in brackets, e.g. `<p>[STARTED] …</p>`. Last paragraph is the agent signature: `<p>— [Agent name]</p>` (substitute the speaking agent's actual name, e.g. Sisyphus, Claude, Oracle). |

> **Why status lives in the body, not the subject.** Exchange Online re-derives `conversationId` from the normalized subject at `/send` time. Any subject mutation between sends — even just swapping `[STARTED]` for `[PROGRESS]` — forks the conversation. Putting the status in the body's first paragraph is the only way to keep one task in one Outlook conversation while still surfacing the current status in iPhone / Outlook mobile notification previews (they show the first ~100 chars of the body).

### Status tokens (use exactly these, all caps, as the literal first text of the body's first `<p>`)

- `[STARTED]` — first email of a task. Sent via `action=send_new`.
- `[PROGRESS]` — intermediate update.
- `[BLOCKED]` — needs user input/decision.
- `[DONE]` — task complete, summary of what shipped.
- `[FAILED]` — task abandoned with reason.
- `[NOTE]` — out-of-band info during the task.

All updates after the first one go via `action=reply` (subject omitted) to keep the conversation threaded.

### Friendly Title rules

- 4–8 words, sentence case, no trailing punctuation, no emojis, no `[STATUS]` prefix.
- Describes the task, not the status (status lives in the body's first paragraph).
- Stays **byte-identical** across every update for the same task. Never edit it. Never prefix or suffix it.
- Examples: `Resend integration in portfolio site`, `Migrate auth to Graph OAuth2`, `Inbox cleanup for last 90 days`.

## Persistence (compaction survival)

State lives under `/tmp/opencode-email-threads/` — one file per active task so concurrent opencode sessions never clobber each other.

> **Compaction reality check.** opencode periodically compacts the conversation. After a compaction your in-context memory of the slug, the exact filename, and even your last few tool calls may be GONE. The protocol below is designed so the filename is **always recoverable from disk**, not from memory: the slug is a pure function of the Friendly Title, every active task has its own file under one well-known directory, and the directory itself is the index. Treat the filesystem as the source of truth for "which thread am I on?" — never your own recollection.

**Slug rule** (pure function — the same Friendly Title always produces the same slug, on this turn or any future turn after compaction):

- Lowercase the Friendly Title.
- Replace every run of non-alphanumeric characters with a single `-`.
- Strip leading/trailing `-`.
- Truncate to 64 characters.

Example: Friendly Title `Resend integration in portfolio site` → slug `resend-integration-in-portfolio-site` → file `/tmp/opencode-email-threads/resend-integration-in-portfolio-site.txt`.

At the start of any task that will email updates:

1. Pick the Friendly Title once. Compute the slug.
2. `mkdir -p /tmp/opencode-email-threads` (idempotent).
3. Write the Friendly Title to `/tmp/opencode-email-threads/<slug>.txt`:
   - File contains exactly one line: the Friendly Title (no quotes, no trailing whitespace).
4. On every subsequent send, **list the directory and re-read the file** to recover the canonical title — do not trust in-context memory of the slug or filename. Because the slug is deterministic, even if compaction wiped your memory of it, re-running the slug rule on the title from the file (or on a best-guess title from your remaining context) reproduces the exact same filename.

### Recovery ladder (after compaction, when you don't remember the slug)

Walk these steps in order — stop at the first one that gives an unambiguous answer:

1. **List `/tmp/opencode-email-threads/`.**
   - Exactly one file → that's your task. Read it for the canonical title.
   - Multiple files → re-slug your best-guess title from current context and look for that exact filename. If found, use it. If not found, ask the user which thread to continue (show the list of titles).
   - Zero files (e.g. `/tmp` wiped on reboot, or task pre-dates the persistence rule) → continue to step 2.
2. **Subject-search recovery via the inbox** (the email thread itself is durable state):
   - `microsoft-graph_search_emails` with `search_type="subject"`, `query="<best-guess title>"`, `days=30`, `inference_classification="all"`.
   - Exactly one matching thread → the subject IS the canonical Friendly Title (no status prefix to strip — the skill never put one there). Recompute the slug, rewrite `/tmp/opencode-email-threads/<slug>.txt` so future sends skip step 2.
3. **Still ambiguous** → ask the user which thread to continue. Never start a new thread to "be safe" — that's exactly the inbox-splitting failure mode this skill exists to prevent.

## Send protocol

### First email of a task

```
microsoft-graph_send_email
  action: "send_new"
  to: ["contact+mcp@adityasethi.dev"]
  subject: "<Friendly Title>"
  htmlbody: "<p>[STARTED] &lt;one-sentence scope&gt;</p>
             <p>Plan: ...</p>
             <p>— [Agent name]</p>"
```

The status token `[STARTED]` is the literal first text of the first `<p>` so iPhone / Outlook mobile previews surface it. Subject is the bare Friendly Title — no `[STATUS]` decoration, ever.

Then write the Friendly Title to `/tmp/opencode-email-threads/<slug>.txt` (creating the directory if needed).

### Every subsequent email — reply to the same thread

```
1. Read /tmp/opencode-email-threads/<slug>.txt → TITLE
2. microsoft-graph_search_emails
     search_type: "subject"
     query: TITLE
     days: 30
     inference_classification: "all"
3. microsoft-graph_browse_email_cache page_number=1
   → take the most recent matching email's `number` as cache_number
4. microsoft-graph_send_email
     action: "reply"
     cache_number: <from step 3>
     htmlbody: "<p>[PROGRESS|BLOCKED|DONE|FAILED|NOTE] &lt;update body&gt;</p>
                <p>— [Agent name]</p>"
```

`subject` is omitted on reply — Graph carries it forward as `RE: <Friendly Title>`. The status token is the very first text of the body's first `<p>` so the new status is what shows in iPhone notification previews.

> **Hard rule: never pass a custom `subject` on a reply.** Exchange Online re-derives `conversationId` from the normalized subject at `/send` time, so a reply with any subject other than the auto `RE: <Friendly Title>` forks into a new conversation even though `createReply` set the original conversationId on the draft. `In-Reply-To` / `References` headers do not override this on Exchange. Verified empirically on this tenant.

## HTML body rules

- No `<br>` between `<p>` tags (creates double-spacing in Outlook).
- No newlines/whitespace between block elements — keep HTML compact.
- Use `<p>`, `<strong>`, `<em>`, `<code>`, `<ul><li>`. No inline styles, no `<div>` salad.
- End every body with `<p>— [Agent name]</p>` where `[Agent name]` is the speaking agent's identity (e.g. `Sisyphus`, `Claude`, `Oracle`, `Sisyphus-Junior`). Substitute literally — do not keep the brackets in the sent email.
- Keep bodies short — this is a status email, not a report. 1–3 short paragraphs.

## When to send

Send only when the user has explicitly asked for email updates in this task. Cadence:

- `[STARTED]` — once, at task kickoff.
- `[PROGRESS]` — only at meaningful milestones (a feature shipped, a long step finished). Never as a heartbeat.
- `[BLOCKED]` — immediately when blocked.
- `[DONE]` / `[FAILED]` — once, at task end.
- `[NOTE]` — sparingly, for out-of-band info the user should see.

Do not email for trivial actions (tool calls, intermediate file edits, normal logs).

## When NOT to use this skill

- The user did not ask for email updates → don't send any.
- The task is one-shot and finishes in seconds → in-chat reply is enough.
- The user wants Slack/Discord/SMS instead → wrong skill.
- Sending TO someone who isn't the user → wrong skill (this is for self-notification only).

## Anti-patterns

- Starting a new `send_new` for an update that belongs to an existing task — splits the inbox.
- Editing the Friendly Title between updates — Outlook will fork the conversation.
- Putting **anything** other than the bare Friendly Title in the subject — no `[STATUS]` prefix, no `(update N)` suffix, no emoji. Any mutation re-derives `conversationId` and forks the thread.
- Forgetting to put the status token as the literal first text of the first `<p>` — iPhone / Outlook mobile previews then show the wrong / no status.
- Putting agent identity in the From address — impossible (M365 rewrites From to the primary SMTP).
- Sending to `contact@adityasethi.dev` instead of `contact+mcp@adityasethi.dev` — bypasses the user's inbox rule for agent traffic.
- Verbose HTML, signatures, or marketing-style formatting — this is a status email.
- Sending heartbeat updates the user didn't ask for.
