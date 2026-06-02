# email-status-updates skill (opencode)

A user-level [opencode](https://opencode.ai) skill that lets agents (Sisyphus, Oracle, Sisyphus-Junior, etc.) send threaded status emails to your inbox via this MCP server.

Designed around four constraints:

1. **One Outlook conversation per task.** Replies thread correctly (uses Graph's `createReply` flow, which this MCP fork implements — upstream's `reply_to_message` does not thread).
2. **Plus-addressing for routing.** All agent mail goes to `<you>+mcp@<your-domain>` so an Outlook rule can shunt it into a folder.
3. **Per-task state under `/tmp/opencode-email-threads/<slug>.txt`.** No conflicts between concurrent opencode sessions.
4. **Survives compaction.** Skill description re-fires post-compaction; a 15-word breadcrumb is enough to re-bootstrap.

## Install

> Adding the MCP to opencode does NOT auto-install this skill. The two are independent. You need both:
>
> 1. The MCP registered in `opencode.jsonc`
> 2. This skill copied to `~/.config/opencode/skill/email-status-updates/SKILL.md`

```bash
mkdir -p ~/.config/opencode/skill/email-status-updates
cp opencode/skills/email-status-updates/SKILL.md \
   ~/.config/opencode/skill/email-status-updates/SKILL.md
```

Then edit [`SKILL.md`](./SKILL.md) and replace `contact@adityasethi.dev` / `contact+mcp@adityasethi.dev` with your own addresses.

Restart opencode so it picks up the new skill.

## How it works

- First send: `action=send_new`, subject `[STARTED] <Friendly Title>`. Title written to `/tmp/opencode-email-threads/<slug>.txt`.
- Subsequent sends: `action=reply` against the most recent matching message. Subject prefix changes (`[PROGRESS]`, `[BLOCKED]`, `[DONE]`, `[FAILED]`, `[NOTE]`) live in the body; Graph's `createReply` carries thread headers so Outlook groups them.
- Post-compaction: skill description re-triggers on the breadcrumb `email-status-updates skill active; thread file at /tmp/opencode-email-threads/<slug>.txt`. Recovery ladder walks dir → subject search → ask user.

See [`SKILL.md`](./SKILL.md) for the full protocol.
