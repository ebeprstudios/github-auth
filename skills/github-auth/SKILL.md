---
name: github-auth
description: Authenticate the GitHub CLI (gh) on this machine via device-code browser flow. Use whenever the user runs `gh auth login`, asks to "sign into GitHub", reports `gh` permission errors (e.g. "gh: not authenticated", "HTTP 401"), or needs to refresh / switch GitHub accounts. Handles the long-running interactive prompt, surfaces the one-time code, and retries cleanly on timeout.
---

# GitHub CLI Authentication

## When to use

- User runs `gh auth login` (with or without flags like `-h github.com`, `--scopes`, `--web`).
- User asks to "log in to GitHub", "authenticate gh", "connect GitHub CLI".
- A `gh` command fails with `not authenticated` / `HTTP 401` / `bad credentials` and re-auth is the fix.
- User wants to add scopes, switch accounts, or refresh an expired token.

Do NOT trigger for: GitHub web UI logins, OAuth in apps the user is building, git credential helper config (that's `gh auth setup-git` *after* login), or PAT-management questions unrelated to `gh`.

## How to run it

`gh auth login` is **interactive and long-running** — it prints a device code, then blocks waiting for the user to authorize in a browser. Run it in the background so the conversation isn't blocked, then surface the code immediately.

### Step 1 — start the command in the background

```
Bash(command: "gh auth login -h github.com", run_in_background: true, timeout: 600000)
```

- Default host is `github.com`; pass the user's exact flags through verbatim if they gave any (`--scopes`, `--git-protocol`, `-p ssh`, etc.).
- 10-minute timeout (`600000` ms) — the device-code flow expires faster than that, but giving headroom avoids killing a slow user mid-authorize.
- Do **not** run it in the foreground. The default 2-minute Bash timeout will kill the prompt before the user finishes, and you can't see the code until output is read.

### Step 2 — read the device code and show it to the user

After kicking off the background command, read its output file once (no `sleep` loops — one read is enough; the code is printed within ~1s):

```
Bash(command: "cat <output-file>", description: "Read auth output")
```

The output looks like:

```
! First copy your one-time code: XXXX-XXXX
Open this URL to continue in your web browser: https://github.com/login/device
```

Surface to the user in a short, scannable format:

1. **One-time code:** `XXXX-XXXX`
2. **Open:** https://github.com/login/device
3. Paste the code and authorize.

Then say you'll wait for the browser flow to finish — the task notification will fire when the command exits.

### Step 3 — on completion, confirm or retry

When the background task notification arrives:

- **Exit 0** → read the output, confirm with the username from the `✓ Logged in as <user>` line. One short sentence is enough.
- **Exit non-zero** → read the output and diagnose. The common failure is `failed to authenticate via web browser: context deadline exceeded` — the device code expired because the user didn't authorize in time. Offer to re-run, or suggest the token path:

  ```
  gh auth login -h github.com --with-token < token-file
  ```

  Tokens are created at https://github.com/settings/tokens. For typical `gh` use, the `repo`, `read:org`, and `workflow` scopes cover most needs.

## Notes

- If `gh` isn't installed, `gh: command not found` will surface immediately on Step 1. Tell the user to install via `brew install gh` (this machine is macOS) before retrying.
- `gh auth status` is the read-only check — use it first if you're unsure whether the user is already logged in, before kicking off a fresh login flow.
- Never write the device code or token to memory, a file, or anywhere persistent. The code is single-use and short-lived; tokens are secrets.
