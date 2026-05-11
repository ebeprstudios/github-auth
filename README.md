# github-auth

A Claude skill that handles `gh auth login` end-to-end — runs the interactive device-code flow in the background, surfaces the one-time code, and reports success/failure when the browser flow finishes.

## Install

```bash
npx skills add ebeprstudios/github-auth
```

## What it does

Triggers when you (or Claude) run `gh auth login`, ask to "sign into GitHub", or hit a `gh` 401/not-authenticated error. The skill:

1. Launches `gh auth login` in the background with a long timeout.
2. Reads the device code and shows it with the authorize URL.
3. Waits for the task-completion notification.
4. Confirms login on exit 0, or diagnoses and offers a retry (including the `--with-token` path) on failure.

See [SKILL.md](./SKILL.md) for full behavior.
