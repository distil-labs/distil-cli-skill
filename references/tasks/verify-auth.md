# Verify Distil CLI Authentication

A common source of confusion is that **`/login` (Claude Code) and `distil login` (Distil CLI) are different things**. They authenticate different sessions. The Distil CLI uses its own credentials independent of Claude.

If a `distil` command fails with "Credit balance is too low", a 401-style error, or simply returns no user from `distil whoami`, the CLI session is missing or expired. The fix is `distil login` — *not* `/login`.

## Verify

Check who the Distil CLI thinks you are:

```bash
distil whoami
```

If this returns a user, you're authenticated and can proceed. Move on.

## Fix (when not authenticated)

If `distil whoami` errors or returns no user, the user needs to authenticate the CLI in their own shell. From a Claude Code prompt, the `!` prefix runs the command in the user's shell rather than as a tool call:

```
! distil login
```

This opens a browser flow for them to log in. Once complete, re-run `distil whoami` to confirm.

## When to do this

- **Before starting any workflow** — first thing in Step 0.
- **Whenever a `distil` command unexpectedly fails with auth-like errors** — even mid-workflow. Auth tokens expire.

## Why this matters

Without this check, you'll spend several minutes diagnosing config files or platform issues for a problem that's actually a missing auth token. The bug bash transcripts showed users running `/login` thinking it would authenticate Distil, then getting confusing "Credit balance is too low" errors instead of a clear "you're not logged in" message.
