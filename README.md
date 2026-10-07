# Git Credential Manager on Windows: popup spam, dual-helper antipattern, and multi-account separation

If Git Credential Manager (GCM) keeps prompting you for a password, opens authentication popups on every `git fetch`, or asks which account to use even though you have only one, the most common root cause is not a stale credential — it is **two credential helpers running simultaneously**. This repo documents how to diagnose that, how to fix it, and how to wire GCM correctly for a Windows machine with multiple GitHub accounts.

All commands are PowerShell 7. The diagnostic steps work the same on Git for Windows 2.x and GCM 2.x.

---

## The symptom

You signed into GitHub through GCM once. You expect Git to remember the token. Instead:

- A popup asks for your password on every push, fetch, or clone — sometimes twice in a row.
- An "account picker" dialog appears even though you only added one GitHub account.
- Multiple `git-credential-manager.exe` windows stack up in Task Manager.
- `git push` hangs for 10+ seconds before any prompt appears.
- Switching between two GitHub accounts (work + personal) on the same machine causes commits to be attributed to the wrong identity.

Each of these has a different proximate cause but the same underlying configuration smell.

---

## First diagnostic: count your credential helpers

Run this:

```powershell
git config --get-all credential.helper
```

You want exactly **one** line back. If you see two:

```
manager
wincred
```

…or any other pair, you have the dual-helper antipattern. Git invokes **every** configured credential helper on **every** credential operation, in order. With two helpers wired up, each `git fetch` calls both — and each one may pop a UI of its own.

This happens because the system-level `gitconfig` (installed by Git for Windows) sets `credential.helper = manager`, and somewhere along the way a `--global` user-level config added `credential.helper = wincred` (or another helper). Both apply. Both run.

To see exactly where each helper is declared:

```powershell
git config --list --show-origin | sls "credential.helper"
```

Output looks like:

```
file:C:/Program Files/Git/etc/gitconfig    credential.helper=manager
file:C:/Users/<you>/.gitconfig             credential.helper=wincred
```

The first line is system. The second is user-global. Both are active.

---

## The fix

Remove the duplicate from user-global. Keep the system default (which is the version of GCM that Git for Windows installed alongside itself):

```powershell
git config --global --unset credential.helper
```

Verify only one remains:

```powershell
git config --get-all credential.helper
# manager
```

Re-run a `git fetch`. The popup spam stops.

If `git config --get-all credential.helper` returns nothing, you have removed the helper entirely and Git will prompt you for a password in the terminal on every operation. In that case, restore GCM as the helper:

```powershell
git config --global credential.helper manager
```

Note: `manager` is the modern name (Git Credential Manager Core, 2.x+). Older docs reference `manager-core` and `wincred`. On a current Git for Windows install, use `manager`.

---

## Secondary diagnostic: stale cached entries

GCM caches tokens in Windows Credential Manager. After fixing the helper config, cached entries from the wrong helper may still cause confusing prompts. List the cached GitHub entries:

```powershell
cmdkey /list | sls github
```

You may see entries like:

```
Target: git:https://github.com
Target: git:https://x-access-token@github.com
Target: LegacyGeneric:target=git:https://github.com
```

Multiple entries for the same host are usually fine — GCM keys them by URL path and token-user. But if you have a leftover from an earlier helper (`LegacyGeneric:` is a strong signal), remove it:

```powershell
cmdkey /delete:"LegacyGeneric:target=git:https://github.com"
```

GCM will recreate its own entry on the next operation.

---

## Zombie process check (skip this and your diagnostics will lie)

If popups have been spamming for a while, you can accumulate orphaned `git-credential-manager` processes that never exited. Symptoms include:

- `Get-Process` and WMI queries on the machine slow down noticeably.
- Even after fixing the helper config, a popup still appears once before everything works.
- `cmdkey /list` takes 30+ seconds to return.

Count them:

```powershell
(Get-Process git-credential-manager -ErrorAction SilentlyContinue).Count
```

Healthy: 0 (no operation in flight). Acceptable: 1–2 transient. **Suspect:** 5+. **Cleanup needed:** 10+.

To clear them:

```powershell
Get-Process git-credential-manager -ErrorAction SilentlyContinue | Stop-Process -Force
```

This is safe — GCM is fully stateless after writing to Credential Manager, so killing in-flight instances at worst loses one unsaved credential decision. Re-run the failed Git command afterwards.

**Run this BEFORE the diagnostic commands above**, not after. Zombie GCM processes make `cmdkey /list` and `git config --list` hang, and you will spend time debugging the wrong thing.

---

## Multi-account separation

If you use two GitHub accounts on the same machine (work + personal), the default GCM behavior keys credentials only by host: `github.com`. Both accounts share the same credential slot. When you push to a personal repo while the work token is cached, GCM either uses the wrong token (push 403) or shows an account picker on every operation.

The fix is one config flag:

```powershell
git config --global credential.https://github.com.useHttpPath true
```

This tells GCM to key cached credentials by **host + full repo path**, not just host. After setting this:

- `https://github.com/CompanyOrg/work-repo.git` gets its own credential slot.
- `https://github.com/personal/side-project.git` gets a separate slot.
- GCM stops asking which account to use.

You will need to re-authenticate each repo once after enabling the flag, because the previous credential entry no longer matches the new key shape. After that, the picker stops for repos that already have a slot.

The trade-off: "once per repo" applies to **every repo you add later**, too. A repo that has never been pushed from this machine has no credential slot yet, so its first push must be able to show a GCM login. In a normal terminal that is a one-time browser sign-in. In a shell where prompts are disabled (CI, scripts, AI coding agents — see below) it fails immediately, even though other repos on the same machine push fine. With the flag on, `cmdkey /list` shows one entry per repo (`git:https://github.com/<owner>/<repo>.git`) instead of a single `git:https://github.com` entry.

---

## PAT recovery without reading Credential Manager directly

If you need to recover a stored PAT for scripting (CI setup, env var population, migrating to a new machine), there are two bad options and one good one.

**Bad option 1:** Read the Credential Vault via `Get-StoredCredential` or `Windows.Security.Credentials` API. This requires admin in some configurations, prompts UAC, and exposes the entire vault contents — most of which you do not need.

**Bad option 2:** Echo the remote URL with the token baked in (`https://<TOKEN>@github.com/...`). This puts the secret in `git remote -v` output, shell history, and any future `git config --list` dump.

**Good option:** Ask GCM directly via its `git credential` protocol. GCM reads from Credential Manager on your behalf, no admin needed, and returns only the credential for the URL you asked about:

```powershell
"url=https://x-access-token@github.com" | git credential fill
```

Output:

```
protocol=https
host=github.com
username=x-access-token
password=ghp_<token-value-here>
path=
```

Pipe to `grep` / `Select-String` for just the password line if scripting. To register a recovered PAT into Credential Manager under a clean URL key (without the `x-access-token@` user portion in the URL), use `git credential approve`:

```powershell
@"
protocol=https
host=github.com
username=x-access-token
password=ghp_<token>
path=
"@ | git credential approve
```

This is the same protocol Git uses internally, so any helper (GCM, wincred, custom) handles it.

---

## Authorship vs authorization (the part that bites you after the fix)

Once popup spam is gone, the next confusion is **commits being attributed to the wrong account**. The mental model that fixes this:

- **Authorization** = which PAT Git uses to talk to the remote. Set by `credential.helper` + the cached token for that URL.
- **Authorship** = which name/email goes into the commit. Set by `user.name` and `user.email` in repo or global config.

These are independent. GitHub maps the commit to a user account **by email**, not by which PAT pushed it. So:

- Pushing with your work PAT but having `user.email = personal@gmail.com` configured in the repo → commits appear under your personal GitHub profile.
- Pushing with your personal PAT but having `user.email = work@company.com` → commits appear under your work profile.

To prevent this:

```powershell
# In your work repo:
git config user.name "Work Name"
git config user.email "work@company.com"

# In your personal repo:
git config user.name "Personal Name"
git config user.email "personal@gmail.com"
```

(These are repo-scoped, not `--global`.)

Verify before your first commit in a new clone:

```powershell
git config user.email
```

If you forgot to set it and committed under the wrong email, you can rewrite the affected commits with `git commit --amend --reset-author` (last commit only) or a `git filter-repo --email-callback` for history.

---

## Pushing from an AI agent's shell (updated 2026-10)

AI coding agents that run shell commands for you (and the "run this command" escape hatch some of them offer) usually start that shell non-interactive, with:

```
GIT_TERMINAL_PROMPT=0
GCM_INTERACTIVE=never
```

So Git cannot ask for a username in the terminal, and GCM is not allowed to open its sign-in UI. If a valid credential is already cached for the URL, `git push` works. If not, it fails fast with one of:

```
fatal: could not read Username for 'https://github.com': terminal prompts disabled
fatal: Cannot prompt because user interactivity has been disabled.
```

This is **not** an expired or broken credential. Combined with `useHttpPath true`, it means: repos you have pushed before work from the agent; a brand-new repo or fresh fork fails until it has been signed in once.

**Fix 1: sign in once interactively.** In your own terminal (not the agent's shell), push the new repo once so GCM can show its browser login:

```powershell
git -C <path-to-repo> push origin <branch>
```

After that, plain `git push` from the agent's shell works for that repo.

**Fix 2: push without GCM, using a token for one command.** If you keep a fine-grained or classic PAT in an environment variable (here called `$env:MY_GITHUB_PAT` as a placeholder), you can bypass the credential helper for a single push and send the token as an HTTP header:

```powershell
$b = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("x-access-token:$env:MY_GITHUB_PAT"))
git -c credential.helper= -c "http.extraHeader=Authorization: Basic $b" push origin <branch>
```

- `-c credential.helper=` disables every configured helper for this one command, so GCM is not invoked (and cannot crash or hang).
- The header is passed on the command line only; nothing is written to `.git/config`, and `git remote -v` stays clean (unlike Bad option 2 above).
- It is still a secret on a command line: don't echo `$b`, and don't paste the command with the expanded value into logs or issues.

Prefer Fix 1 when you can: it keeps the token out of the agent's hands entirely.

---

## Cheatsheet

Diagnostic order on a sick machine:

```powershell
# 0. Kill zombies first or diagnostics will lie
Get-Process git-credential-manager -ErrorAction SilentlyContinue | Stop-Process -Force

# 1. Helper count (want exactly 1)
git config --get-all credential.helper

# 2. Where each helper is declared
git config --list --show-origin | sls "credential.helper"

# 3. Cached entries
cmdkey /list | sls github

# 4. Multi-account separation (run once)
git config --global credential.https://github.com.useHttpPath true

# 5. Authorship per repo (run in each clone)
git config user.email "<correct-email-for-this-repo>"

# 6. Prompts disabled in this shell? (agent / CI)
$env:GIT_TERMINAL_PROMPT; $env:GCM_INTERACTIVE
```

Fix the dual-helper:

```powershell
git config --global --unset credential.helper
```

Recover a PAT:

```powershell
"url=https://x-access-token@github.com" | git credential fill
```

---

## TL;DR

1. Popup spam = two credential helpers running. Check with `git config --get-all credential.helper`. Want one line, not two.
2. Fix with `git config --global --unset credential.helper` to keep only the system-default GCM.
3. Kill zombie GCM processes before diagnosing further (`Get-Process git-credential-manager | Stop-Process -Force`).
4. Multi-account: `git config --global credential.https://github.com.useHttpPath true` keys credentials per-repo.
5. Wrong-author commits = `user.email` problem, not credential problem. GitHub maps commits by email, not by PAT.
6. PAT recovery: `git credential fill` instead of reading the Vault directly.
7. Push fails instantly from an AI agent's shell on a new repo? Prompts are disabled (`GIT_TERMINAL_PROMPT=0`, `GCM_INTERACTIVE=never`) and `useHttpPath` means that repo has no credential yet. Sign in once interactively, or push once with `-c credential.helper=` + a token in `http.extraHeader`.

---

## References

- [Git Credential Manager docs](https://github.com/git-ecosystem/git-credential-manager)
- [Git credential helper docs](https://git-scm.com/docs/gitcredentials)
- [`git credential` protocol reference](https://git-scm.com/docs/git-credential)
- [`useHttpPath` config](https://git-scm.com/docs/gitcredentials#Documentation/gitcredentials.txt-useHttpPath)

---

## License

MIT. See [LICENSE](./LICENSE).
