# Replacing the workshop API key

How to point the Mac minis at a new Anthropic Console organisation or a new
API key, in a way that **survives every re-run of `workshop-setup.sh`**. You
do this once per Mac mini; later workshops need no login step at all.

> The [Authentication](../README.md#authentication) section of the README
> still describes the older per-machine browser login. This document replaces
> that flow with a single workshop API key. `workshop-setup.sh` does not
> automate these steps yet — they are manual.

## How it works

Every run of `workshop-setup.sh` empties `~/.claude/`, so a key stored there
would be lost on each reset. Two places are **not** touched by the script:

| Where | What goes there |
|---|---|
| macOS Keychain, item `claude-workshop-api-key` | The API key itself |
| `/Library/Application Support/ClaudeCode/managed-settings.json` | An `apiKeyHelper` that reads the key from the Keychain |

Claude Code runs the `apiKeyHelper` command to get its key. The key is never
written to a plain-text file — not on the machine, not on the USB stick.

## 1. In the Console (once per organisation)

1. Log in to the new organisation at <https://console.anthropic.com>.
2. Create a workspace for the workshops: **Settings → Workspaces**, e.g.
   `Workshops`.
3. Set a **spend limit** on that workspace, so a runaway session can't
   produce a surprise bill.
4. Create an API key in that workspace, e.g. `workshop-macminis`.
   - One shared key is simplest.
   - One key per Mac mini (`workshop-macmini-01`, …) gives per-machine usage
     and lets you revoke one machine without touching the others.
5. Copy the key — the Console shows it only once.

## 2. On each Mac mini (once)

Do this logged in as the workshop user, at the machine itself (not over SSH:
the Keychain may be locked there).

### 2a. Remove the old login

```bash
claude
```

Inside Claude Code, run `/logout`, then exit. This removes the old
organisation's credential from the Keychain.

Also make sure no old key is set in the shell — an `ANTHROPIC_API_KEY`
environment variable takes precedence over `apiKeyHelper`:

```bash
grep -n ANTHROPIC_API_KEY ~/.zshrc ~/.zprofile 2>/dev/null
```

Remove any line it finds, then open a new terminal.

### 2b. Store the key in the Keychain

```bash
security add-generic-password -U -a "$USER" -s claude-workshop-api-key -w
```

It prompts for the key (twice), so it never lands in your shell history.
`-U` overwrites an existing item, so the same command works for replacing
the key later.

### 2c. Tell Claude Code where to find it

```bash
sudo mkdir -p "/Library/Application Support/ClaudeCode"
sudo tee "/Library/Application Support/ClaudeCode/managed-settings.json" >/dev/null <<'EOF'
{
  "apiKeyHelper": "security find-generic-password -s claude-workshop-api-key -w"
}
EOF
```

If that file already exists with other settings, add the `apiKeyHelper` line
to it instead of overwriting it.

### 2d. Verify

```bash
security find-generic-password -s claude-workshop-api-key -w | cut -c1-12
cd ~/workshop
claude
```

The first command should print the start of the key (`sk-ant-api03-…`). Inside
Claude Code, run `/status`: the API key source should be `apiKeyHelper`.
Send one short prompt to confirm requests go through.

Then run `workshop-setup.sh` once more and check that `claude` still works
without logging in — that proves the setup survives a reset.

## 3. Replacing the key later

When a key is revoked or you switch organisation again:

1. Create the new key in the Console (section 1, steps 4–5).
2. On each Mac mini, run step **2b** again with the new key.
3. Revoke the old key in the Console.

The managed settings stay as they are. No logout or re-login is needed,
unless you switch organisation and the machine still has an old browser
login — then do step 2a as well.

## Security notes

- **Attendees can read the key.** Anyone at the Mac mini can run
  `security find-generic-password … -w`. On machines used only for the
  workshop that is usually acceptable, given the spend limit. If not, revoke
  the key after each workshop and store a fresh one with step 2b.
- **Never commit a key** to this repository. The repo is public.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `claude` opens a browser login | `apiKeyHelper` is not picked up | Check that the managed-settings file exists and is valid JSON (`plutil -lint "/Library/Application Support/ClaudeCode/managed-settings.json"`) |
| `Invalid API key` / 401 | Key revoked or mistyped | Repeat step 2b with a valid key |
| `/status` shows the old organisation | Old credential or `ANTHROPIC_API_KEY` still active | Repeat step 2a |
| `security: SecKeychainSearchCopyNext: The specified item could not be found` | Key not stored for this user | Repeat step 2b as the workshop user |
| Keychain password prompt when starting `claude` | Keychain locked (e.g. over SSH) | Run at the machine itself, or unlock with `security unlock-keychain` |
