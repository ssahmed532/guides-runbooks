---
title: Git CLI Setup for Windows 11
eyebrow: Engineering Standards · Developer Runbook
tagline: >-
  A full inventory of one working Git for Windows setup, and the steps to rebuild it
  **exactly** on a second Windows 11 machine. It covers the installer choices, the
  system and global config, the **delta**-based diff pipeline, the GitHub credential
  helper, SSH commit signing and the global ignore file. Install steps run in
  **PowerShell**. Config steps run in **Git Bash**.
budget:
  value: "~20"
  unit: "minutes, once, per laptop"
version: "1.1"
date: 2026-10-26
lang: en
---

<!--
AUTHORING CONVENTIONS: same as sign-your-commits.md (Pandoc Markdown, pandoc ≥ 3.0).
Build:  pandoc git-cli-windows-setup.md -f markdown -t html5 -s --section-divs --template runbook.html -o index.html
Steps marked shell="git-bash" run in Git Bash (prompt reads MINGW64); shell="powershell" steps run in PowerShell 7.
-->

::: {.req label="Source machine"}
**Inventoried 2026-10-26** on Windows 11 Pro 10.0.26200, Git for Windows **2.56.0.windows.3**. Every value below was read from the live machine: `git config --list --show-scope --show-origin`, the raw `~/.gitconfig` and `C:\Program Files\Git\etc\gitconfig`, the Git installer's registry record, `winget list`, and the PowerShell profile. Personal identity values are written as placeholders (`Your Name`, `you@example.com`). Put in your own.

**v1.1, 2026-10-26:** the diff pipeline was re-verified on the same machine and the *How delta pages* section was added, from `delta --show-config`, a `git` alias probe run in PowerShell, `git-config(1)` and the delta manual.
:::

## Inventory {#inventory}

### Toolchain {.unnumbered}

| Tool | Version | winget id | Role in the Git setup |
|---|---|---|---|
| Git for Windows | 2.56.0.3 | `Git.Git` | Git, Git Bash, bundled OpenSSH 10.6p1, Git LFS 3.8.1, Git Credential Manager 2.9.1, **less 710** (the pager delta runs) |
| delta | 0.20.1 | `dandavison.delta` | Renders every diff, log, show and `add -p`, then pages the result through Git's `less` |
| GitHub CLI | 2.103.2 | `GitHub.cli` | Credential helper for `github.com` and `gist.github.com` |
| Notepad++ | 8.9.8.1 | `Notepad++.Notepad++` | `core.editor` (Git Bash, cmd, GUIs) |
| VS Code | n/a | n/a | `GIT_EDITOR` inside PowerShell (set by the profile) |
| bat | 0.26.1 | `sharkdp.bat` | *Optional.* Not used by Git. Shares the `Visual Studio Dark+` theme with delta |
| lazygit | 0.66.1 | `JesseDuffield.lazygit` | *Optional.* Git TUI, no config file (defaults) |

### Effective configuration by scope {.unnumbered}

**System:** `C:\Program Files\Git\etc\gitconfig`. The installer writes this file from the choices in Step 2. Never edit it by hand.

| Key | Value | Why it matters |
|---|---|---|
| `core.autocrlf` | `true` | CRLF in the working tree, LF in the repo ("Checkout Windows-style, commit Unix-style") |
| `core.fscache` | `true` | Faster `git status` on NTFS |
| `core.symlinks` | `false` | Symlinks check out as plain files |
| `core.editor` | `"C:\Program Files\Notepad++\notepad++.exe" -multiInst -notabbar -nosession -noPlugin` | Commit and rebase editor (see the editor note below) |
| `pull.rebase` | `false` | `git pull` merges |
| `init.defaultBranch` | `master` | New repos start on `master` |
| `http.sslBackend` | `schannel` | TLS through the Windows certificate store |
| `credential.helper` | `manager` | Git Credential Manager for every host not overridden below |
| `credential.https://dev.azure.com.useHttpPath` | `true` | GCM keeps one credential per Azure DevOps org/repo path |
| `diff.astextplain.textconv` | `astextplain` | With the installer's `etc/gitattributes`, `.doc/.docx/.pdf/.rtf/.odt…` diff as text |
| `filter.lfs.*` | `clean`, `smudge`, `process`, `required=true` | Git LFS hooked in |

**Global:** `~/.gitconfig`. This file is yours. Step 5 writes it in full.

| Key | Value | Why it matters |
|---|---|---|
| `user.name` / `user.email` | *your identity* | Author and committer |
| `core.pager` | `delta` | **All** paged Git output goes through delta, which in turn runs `less -R` (see *How delta pages*) |
| `interactive.diffFilter` | `delta --color-only` | `git add -p` / `checkout -p` hunks get delta colouring and keep their 1:1 line mapping |
| `delta.navigate` | `true` | `n` / `N` jump between files in the pager |
| `delta.line-numbers` | `true` | Old/new line-number gutter |
| `delta.side-by-side` | `false` | Unified layout (switch it per command, see *The diff pipeline*) |
| `delta.syntax-theme` | `Visual Studio Dark+` | Syntax highlighting theme (a *dark* theme, also used for bat) |
| `diff.algorithm` | `histogram` | Cleaner hunks than the default Myers on reordered code |
| `diff.colorMoved` | `default` | Moved blocks are coloured apart from real adds/deletes |
| `merge.conflictstyle` | `zdiff3` | Conflict markers include the merge-base section, with common lines zipped out |
| `credential.https://github.com.helper` | *(empty)*, then `!'C:\\Program Files\\GitHub CLI\\gh.exe' auth git-credential` | The empty entry **resets** the inherited GCM helper. GitHub auth then comes from `gh` |
| `credential.https://gist.github.com.helper` | same pair | Same for gists |
| `gpg.format` | `ssh` | Sign with an SSH key, not GPG |
| `user.signingkey` | `~/.ssh/id_ed25519.pub` | Signing key (the **public** half) |
| `commit.gpgsign` / `tag.gpgSign` | `true` / `true` | Every commit and annotated tag is signed |
| `gpg.ssh.program` | `C:/Program Files/Git/usr/bin/ssh-keygen.exe` | Pins Git's OpenSSH 10.6, not `System32\OpenSSH` 9.5 |
| `gpg.ssh.allowedSignersFile` | `~/.ssh/allowed_signers` | Lets `git log --show-signature` verify your own signatures |

**XDG / other files**

| File | Contents |
|---|---|
| `~/.config/git/ignore` | `**/.claude/settings.local.json`. This is the default `core.excludesFile` location, so no config key is needed |
| `~/.ssh/allowed_signers` | One line: `<email> ssh-ed25519 <key>` |
| PowerShell profile | `EDITOR=code`, `GIT_EDITOR=code --wait`, `BAT_THEME=Visual Studio Dark+` (only if `code` / `bat` exist) |

**Not configured (deliberately absent, so leave them absent for an exact copy):** no `[alias]`, no `[include]`/`[includeIf]`, no `diff.tool`/`merge.tool`, no `~/.gitattributes`, no `~/.config/git/attributes`, no `GIT_PAGER`, `DELTA_PAGER`, `BAT_PAGER`, `PAGER`, `LESS` or `COLORTERM` variables anywhere (none persisted, none set by a profile), no other `GIT_*` / `DELTA_*` variables, `ssh-agent` Windows service disabled, Git auto-updater not installed.

## Rebuild {#setup}

### Install Notepad++ first {.step minutes="1 min" shell="powershell"}

The Git installer's *Notepad++* editor option only applies if Notepad++ is already on disk. Otherwise it silently falls back to Vim.

```powershell
winget install --id Notepad++.Notepad++ --exact --version 8.9.8.1 --source winget
```

::: note
**Pinned versions.** Each `--version` gives an exact copy. If winget answers *No version found matching*, that build has left the catalogue. Drop `--version` to take the current release, and record the difference.
:::

### Install Git for Windows with the same installer choices {.step minutes="3 min" shell="powershell"}

The installer's choices were read from its registry record (`HKLM\…\Uninstall\Git_is1`). Save them as an answer file and feed it to a silent install:

```powershell
$inf = "$env:TEMP\git-install.inf"
@'
[Setup]
Lang=default
Dir=C:\Program Files\Git
Group=Git
NoIcons=0
SetupType=default
Components=icons,icons\desktop,ext,ext\shellhere,ext\guihere,gitlfs,assoc,assoc_sh,windowsterminal,scalar
Tasks=
EditorOption=Notepad++
CustomEditorPath=
DefaultBranchOption=master
PathOption=Cmd
SSHOption=OpenSSH
TortoiseOption=false
CURLOption=WinSSL
CRLFOption=CRLFAlways
BashTerminalOption=MinTTY
GitPullBehaviorOption=Merge
UseCredentialManager=Enabled
PerformanceTweaksFSCache=Enabled
EnableSymlinks=Disabled
EnableFSMonitor=Disabled
'@ | Set-Content -Path $inf -Encoding ascii

winget install --id Git.Git --exact --version 2.56.0.3 --source winget `
  --override "/VERYSILENT /NORESTART /SUPPRESSMSGBOXES /LOADINF=$inf"
```

Leaving `autoupdate` out of `Components` matches the source machine: Git does not update itself, and you upgrade it with `winget upgrade Git.Git`.

Then confirm that the installer recorded the same answers. Close and reopen PowerShell first so the new `PATH` loads:

```powershell
Get-Content 'C:\Program Files\Git\etc\install-options.txt'
git config --system --list
```

`install-options.txt` must read `Editor Option: Notepad++`, `Path Option: Cmd`, `SSH Option: OpenSSH`, `CURL Option: WinSSL`, `CRLF Option: CRLFAlways`, `Bash Terminal Option: MinTTY`, `Git Pull Behavior Option: Merge`, `Use Credential Manager: Enabled`, `Performance Tweaks FSCache: Enabled`, `Enable Symlinks: Disabled`, `Enable FSMonitor: Disabled`. The system config must match the **System** table above. The only line that may differ is the `Default Branch Option` text: the source machine shows it blank and still produced `init.defaultBranch=master`.

::: {.note .ink}
**Prefer clicking through?** Run `winget install --id Git.Git --exact --interactive` and pick, page by page: components as listed in `Components=` above (no auto-updates) · *Use Notepad++* · *Let Git decide* · *Git from the command line and also from 3rd-party software* · *Use bundled OpenSSH* · *Use the native Windows Secure Channel library* · *Checkout Windows-style, commit Unix-style* · *Use MinTTY* · *Fast-forward or merge* · *Git Credential Manager* · *Enable file system caching* · symbolic links **off** · FSMonitor **off**. Then run the same two checks.
:::

### Install delta, GitHub CLI and the optional companions {.step minutes="2 min" shell="powershell"}

```powershell
winget install --id dandavison.delta --exact --version 0.20.1 --source winget
winget install --id GitHub.cli       --exact --version 2.103.2 --source winget

# optional, not used by Git itself
winget install --id sharkdp.bat            --exact --version 0.26.1 --source winget
winget install --id JesseDuffield.lazygit  --exact --version 0.66.1 --source winget
```

winget installs delta, bat and lazygit as *portable* packages and adds each package folder to your **user** `PATH`. Open a **new** terminal before you continue, then check:

```powershell
delta --version    # delta 0.20.1
gh --version       # gh version 2.103.2
```

### Restore the PowerShell profile lines {.step minutes="1 min" shell="powershell"}

On the source machine the profile lives in OneDrive (`OneDrive\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`). If OneDrive *Documents* backup is on for the new machine, the profile arrives on its own and you can skip this step. Otherwise add these lines to `$PROFILE`:

```powershell
if (Get-Command bat -CommandType Application -ErrorAction SilentlyContinue) {
    $env:BAT_THEME = 'Visual Studio Dark+'
}
if (Get-Command code -ErrorAction SilentlyContinue) {
    $env:EDITOR     = 'code'
    $env:GIT_EDITOR = 'code --wait'
}
```

::: note
**Two editors, by design.** `GIT_EDITOR` outranks `core.editor`. Commits made from **PowerShell** open in VS Code, and commits from **Git Bash**, cmd or a GUI open in Notepad++. Run `git var GIT_EDITOR` in each shell to see which one applies there.
:::

### Write the global config {.step minutes="2 min" shell="git-bash"}

Open **Git Bash** (prompt reads `MINGW64`). If `~/.gitconfig` already exists on the new machine, back it up first. The heredoc below **replaces** the file.

```{.console chrome="wt" title="MINGW64:/c/Users/you" cwd="~"}
$ [ -f ~/.gitconfig ] && cp ~/.gitconfig ~/.gitconfig.bak

$ cat > ~/.gitconfig <<'EOF'
[user]
	name = Your Name
	email = you@example.com
	signingkey = ~/.ssh/id_ed25519.pub
[core]
	pager = delta
[interactive]
	diffFilter = delta --color-only
[delta]
	navigate = true
	line-numbers = true
	side-by-side = false
	syntax-theme = Visual Studio Dark+
[merge]
	conflictstyle = zdiff3
[diff]
	colorMoved = default
	algorithm = histogram
[credential "https://github.com"]
	helper = 
	helper = !'C:\\Program Files\\GitHub CLI\\gh.exe' auth git-credential
[credential "https://gist.github.com"]
	helper = 
	helper = !'C:\\Program Files\\GitHub CLI\\gh.exe' auth git-credential
[gpg]
	format = ssh
[commit]
	gpgsign = true
[tag]
	gpgSign = true
[gpg "ssh"]
	program = C:/Program Files/Git/usr/bin/ssh-keygen.exe
	allowedSignersFile = ~/.ssh/allowed_signers
EOF
```

Replace the two `[user]` placeholders. This file is the source machine's `~/.gitconfig` line for line. Git expands the `~` in both `~/.ssh/…` paths, so the file works under any Windows username.

::: note
**Keep the quoted heredoc marker (`<<'EOF'`).** Without the quotes, Bash collapses the `\\` in the `gh.exe` path to `\`, and Git then fails to parse the helper. Also keep the `helper = ` line that is empty. It cancels the system-level `manager` helper for GitHub hosts, so `gh` is the only helper Git asks.
:::

### Sign in to GitHub through gh {.step minutes="2 min" shell="git-bash"}

```{.console chrome="wt" title="MINGW64:/c/Users/you" cwd="~"}
$ gh auth login --hostname github.com --git-protocol https --web
```

Answer **Yes** to *Authenticate Git with your GitHub credentials?*. `gh auth setup-git` replaces the helper entries rather than appending to them, so the config written in Step 5 stays the same. The token goes to Windows Credential Manager (`gh auth status` shows `(keyring)`).

Remotes on the source machine are all **HTTPS**. SSH is used only for signing.

### Create the signing key and allowed-signers file {.step minutes="5 min" shell="git-bash"}

Follow **[Sign Your Commits](sign-your-commits.md)**, but only these parts. Step 5 already set the config keys.

1. **Generate a signing key**: `ssh-keygen -t ed25519 -C "you@example.com"`, default path `~/.ssh/id_ed25519`.
2. **Register the key on GitHub** as a **Signing key**.
3. **Tell Git whose signatures to trust**: run only the `printf … > ~/.ssh/allowed_signers` line.

::: {.note .ink}
**Generate a new key. Don't copy the old private key.** Give each machine its own key, so that losing one laptop means revoking one key. To let *this* machine verify commits you signed on the *old* one, append the old machine's public key as a second line in `~/.ssh/allowed_signers`. Use the same email, then a space, then the contents of the old `id_ed25519.pub`.
:::

The Windows `ssh-agent` service is **Disabled** on the source machine and the setup doesn't need it. Git signs through its own `ssh-keygen.exe`. If you give the key a passphrase, read *About the passphrase* in the signing runbook.

### Restore the global ignore file {.step minutes="1 min" shell="git-bash"}

```{.console chrome="wt" title="MINGW64:/c/Users/you" cwd="~"}
$ mkdir -p ~/.config/git

$ printf '%s\n' '**/.claude/settings.local.json' > ~/.config/git/ignore
```

`~/.config/git/ignore` is Git's default global excludes file. Leave `core.excludesFile` unset, as it is on the source machine.

## The diff pipeline, explained {#diff}

Every layer that shapes what `git diff` shows, in the order Git applies them:

| Layer | Setting | Effect |
|---|---|---|
| 1. Text conversion | system `diff.astextplain.textconv` + installer `etc/gitattributes` | `.doc .docx .docm .dot .dotx .dotm .pdf .rtf .ods .odf .odt` (either case) run through `astextplain` and diff as plain text, not *Binary files differ* |
| 2. LFS | system `filter.lfs.*` | LFS pointers are smudged and cleaned. Diffs of LFS files show pointer changes |
| 3. Hunk algorithm | `diff.algorithm=histogram` | Picks the hunks. Applies to `diff`, `log -p`, `show`, `add -p` and `format-patch` |
| 4. Move detection | `diff.colorMoved=default` | Git colours moved blocks apart from adds and deletes. delta passes those colours through unchanged: its `inspect-raw-lines` switch (on by default) is how it supports `--color-moved`. No `map-styles` is set |
| 5. Rendering | `core.pager=delta` + `[delta]` | Syntax highlighting, line numbers, `n`/`N` navigation. delta then hands the result to `less -R` (see *How delta pages*) |
| 6. Interactive staging | `interactive.diffFilter=delta --color-only` | `add -p`, `checkout -p`, `reset -p` hunks get colour only, with no decorations, because Git needs a 1:1 line mapping |
| 7. Conflicts | `merge.conflictstyle=zdiff3` | Shows `<<<<<<< ours · \|\|\|\|\|\|\| base · ======= · >>>>>>> theirs`, with lines common to both sides pulled out of the conflict |

Day-to-day variations, with no config changes needed:

```{.console chrome="wt-mini" title="MINGW64" cwd="~/repo"}
$ git -c delta.side-by-side=true diff     # one-off side-by-side

$ git --no-pager diff                     # raw Git output, no delta

$ git diff --stat                         # delta passes stat output through unchanged

$ git diff | delta                        # force delta when Git itself would not page (see below)
```

### How delta pages {.unnumbered}

Nothing in the config names a pager for delta, and `less` is not on the PowerShell `PATH`, yet `git log` pages correctly from PowerShell, cmd and Git Bash alike. This is the chain, in order:

| Step | Who | What happens on this machine |
|---|---|---|
| 1. Terminal check | Git | Git starts a pager only when its stdout is a terminal. `git diff \| grep …` and `git diff > file` get raw, uncoloured Git output and never touch delta |
| 2. Environment | Git | Before running `core.pager`, Git sets `LESS=FRX` and `LV=-c` **if they are unset** ([git-config(1)](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corepager)). Nothing on this machine sets `LESS`, so `FRX` applies: `F` quit if it fits one screen, `R` pass colour codes through, `X` don't clear the screen on exit |
| 3. `PATH` for children | `git.exe` | `C:\Program Files\Git\cmd\git.exe` is a launcher. It prepends `ucrt64\libexec\git-core`, `ucrt64\bin` and `usr\bin` to `PATH` before running the real Git, and every child, delta included, inherits that `PATH`. That is why delta finds **Git's** `less` 710 from PowerShell, even though `Get-Command less` fails there |
| 4. Pager choice | delta | delta reads `DELTA_PAGER`, then `BAT_PAGER`, then `PAGER`. None is set, so it runs its default `less -R` ([delta manual](https://dandavison.github.io/delta/environment-variables.html)). `delta --show-config` confirms `pager = none` and `paging = auto`, meaning page only when delta's own stdout is a terminal |
| 5. Paging | `less` | `less -R` plus the `FRX` from step 2. `n` / `N` are delta's `navigate` feature: it emits markers that match `navigate-regex` `^(commit\|added:\|removed:\|renamed:\|Δ\|•)` and `less` searches for them |
| 6. Colour depth | Windows Terminal | Windows Terminal does **not** export `COLORTERM`, so delta's `true-color` auto-detection resolves to `false` and *Visual Studio Dark+* is rendered with the nearest 256-colour approximations. That is the current look. See the audit findings |
| 7. `add -p` | Git | Interactive staging uses no pager. Git pipes each hunk through `interactive.diffFilter`, that is `delta --color-only`, and reads the coloured hunk back |

Two consequences worth knowing:

- **Keep every pager variable unset.** `GIT_PAGER` must be unset or exactly `delta`. If you ever set `DELTA_PAGER`, `BAT_PAGER` or `PAGER`, include `-R` and point at a **recent** `less`: the delta manual calls a current `less` essential on Windows and recommends the one bundled with Git. An old `less.exe` elsewhere on `PATH` only becomes a problem if one of those variables names it, because Git's launcher puts `usr\bin` first.
- **delta run by hand is not Git.** When you run `delta` directly from PowerShell, or pipe into it with `git diff | delta`, delta's `PATH` is the plain PowerShell one, with no `less` on it. Do that from Git Bash, where `less` is on `PATH`, or add `C:\Program Files\Git\usr\bin` to the session `PATH` first.

```powershell
# Prove the chain from PowerShell
git -c alias.probe='!command -v less' probe           # /usr/bin/less  (Git's less 710)
delta --show-config | Select-String 'pager|paging'   # pager = none, paging = auto
git log -3                                           # pages in delta; n / N move between commits, q quits
```

::: note
**No external diff or merge tool is configured.** `git difftool` and `git mergetool` fall back to Git's built-in guesses. Leave them unset for an exact copy.
:::

## Prove it matches {#verify}

Run in **Git Bash** inside any cloned repo:

```{.console chrome="wt" title="MINGW64:/c/repos/any" cwd="/c/repos/any" branch="master"}
$ git --version                              # git version 2.56.0.windows.3

$ git config --list --show-scope --show-origin   # compare with the Inventory tables

$ git var GIT_PAGER                          # delta

$ git var GIT_EDITOR                         # "C:\Program Files\Notepad++\notepad++.exe" -multiInst …

$ git config --get-all credential.https://github.com.helper   # (blank line), then !'C:\Program Files\GitHub CLI\gh.exe' auth git-credential

$ git check-ignore -v --no-index .claude/settings.local.json # …/.config/git/ignore:1:**/.claude/settings.local.json

$ git ls-remote origin HEAD                  # no prompt: gh supplied the credential

$ git commit --allow-empty -m "chore: verify setup"

$ git log -1 --show-signature                # Good "git" signature for you@example.com …

$ git show                                   # delta: line numbers, syntax colour, n/N works
```

Then run `git add -p` on a real change and check that the hunks are coloured. Undo the test commit with `git reset --hard HEAD~1` if you don't want it.

In **PowerShell**:

```powershell
git var GIT_EDITOR                                   # code --wait
git -c alias.probe='!command -v less' probe           # /usr/bin/less
git log -3                                           # pages in delta with n / N working
git diff HEAD~1 | Select-Object -First 3             # raw Git output: piped, so no delta. Expected
```

## Faults, and what they actually mean {#faults}

::: faults

#### `delta: command not found` / `cannot run delta`
The terminal was opened before winget added delta's folder to the user `PATH`. Open a new terminal. GUI apps that were already running when you installed need a restart for the same reason.

#### `fatal: bad config line … in ~/.gitconfig`
The heredoc was unquoted, or an editor turned the tab indentation into something else. Re-run Step 5 exactly, with `<<'EOF'`.

#### GitHub keeps prompting, or GCM's browser window opens for github.com
The empty `helper =` line is missing, so GCM still handles GitHub. Check with `git config --show-origin --get-all credential.https://github.com.helper`.

#### `git log` reports “No signature” on your own commits
`~/.ssh/allowed_signers` is missing, has the wrong email, or was written by PowerShell (UTF-16). See *Faults* in [Sign Your Commits](sign-your-commits.md).

#### Commit editor is Vim
Notepad++ wasn't installed when Git was, so the installer fell back to Vim. Re-run Step 2. The installer rewrites `core.editor` in the system file.

#### `git diff` shows plain `+` / `-` lines with no colour
The output was piped or redirected, so Git never started a pager (step 1 of *How delta pages*). Run the command on its own, or pipe through `delta` explicitly from Git Bash.

#### Wrong colours, stray characters, or `n` / `N` do nothing
delta is running a different `less`. Something set `DELTA_PAGER`, `BAT_PAGER` or `PAGER`. Check `delta --show-config | grep pager` (must read `none`) and `env | grep -E 'PAGER|LESS'` (must be empty). Unset the variable; delta then falls back to `less -R`, and Git's launcher guarantees that is Git's `less` 710.

#### Colours look flat next to VS Code's *Dark+*
Expected on this setup. Windows Terminal does not set `COLORTERM`, so delta renders in 256 colours. Audit finding 7 has the one-line opt-in for 24-bit colour.

#### winget: *No version found matching*
That build is no longer in the catalogue. Drop `--version` and note the version you got.

:::

## Audit findings on the source machine {#audit}

These are observations only. The rebuild above reproduces the source machine as it is, apart from finding 1.

1. **Dead line in the global ignore file.** `~/.config/git/ignore` holds `**/.claude\settings.local.json` and also the correct `**/.claude/settings.local.json`. In gitignore syntax a backslash escapes the next character. It is not a path separator, so the first line matches nothing. Step 8 writes only the working line.
2. **Absolute home paths in `~/.gitconfig`. Fixed 2026-10-26.** `user.signingkey` and `gpg.ssh.allowedSignersFile` were written as `C:/Users/<you>/…`, which break under a different Windows username. Both were changed to `~/…` on the source machine, and a signed commit still verified as *Good* afterwards.
3. **Editor depends on the shell.** Notepad++ is the editor in Git Bash, cmd and GUIs, and VS Code in PowerShell (via `GIT_EDITOR`). This is deliberate (see Step 4), but people hit it when a rebase opens a different editor than they expect.
4. **No Git auto-update.** The `autoupdate` component is deselected. Keep Git current with `winget upgrade Git.Git`. A winget upgrade reuses the recorded installer choices.
5. **A single global identity.** A personal email is set globally and nothing overrides it per directory. If org repos are ever cloned on this machine, an `[includeIf "gitdir:…"]` with a work identity would stop commits going out under the personal address. Not applied here, to keep the copy exact.
6. **Signing works but nothing is tuned beyond it.** `fetch.prune`, `push.autoSetupRemote`, `rerere.enabled` and aliases are all unset. You could add them later. They are left out here on purpose.
7. **delta renders in 256 colours, not 24-bit.** Windows Terminal supports 24-bit colour but does not export `COLORTERM`, so delta's auto-detection stays off (`delta --show-config` reads `true-color = false`). `true-color = always` under `[delta]`, or `COLORTERM=truecolor` in the profiles, would switch *Visual Studio Dark+* to its real colours. Not applied, to keep the copy exact; apply it on both machines or neither.
8. **`less` exists only through Git's launcher.** No `less` is on the PowerShell `PATH`. Every Git command still pages correctly because `git.exe` prepends Git's `usr\bin` for its children (*How delta pages*, step 3). Keep it that way: adding another `less` to `PATH` changes nothing for Git, and setting a `*PAGER` variable to one does.

::: footer
Inventory method: `git config --list --show-scope --show-origin`; raw `~/.gitconfig` and `C:\Program Files\Git\etc\gitconfig`; `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Git_is1`; `C:\Program Files\Git\etc\install-options.txt`; `winget list`; persisted User/Machine environment; PowerShell profile; `delta --show-config`; `git -c alias.probe='!command -v less' probe` from PowerShell; `git-config(1)` *core.pager*; the delta manual (*Environment variables*, *--color-moved support*, *Using Delta on Windows*, *24 bit color*). Secrets were masked during collection, and none are recorded here.
:::
