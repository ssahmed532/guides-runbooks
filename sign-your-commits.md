---
title: Sign Your Commits
eyebrow: Engineering Standards · Developer Runbook
tagline: >-
  Git's author field is **plain text that anyone can set**. A signed commit proves
  the change came from you and hasn't been altered since. Five steps — and every
  command below has to be run in the **Git Bash** shell that Git for Windows
  installs, never in PowerShell or Command Prompt. If VS Code is where you work,
  Git Bash is a tab inside it and the setup never leaves the editor.
budget:
  value: "~6"
  unit: "minutes, once, per laptop"
version: "1.5"
date: 2026-10-26
lang: en
---

<!--
AUTHORING CONVENTIONS — Pandoc Markdown (pandoc ≥ 3.0)
Build:  pandoc sign-your-commits.md -f markdown -t html5 -s --section-divs --template runbook.html -o index.html

Front matter      → page header: eyebrow, h1 title, tagline, budget tile (value + unit), footer version/date.
## Heading        → section h2 (uppercase rule style).
### Heading {.step minutes="…" shell="git-bash"}
                  → numbered step card; `minutes` renders as the grey pill, `shell="git-bash"` as the accent "Git Bash" pill.
                    Numbering (01, 02 …) comes from a CSS counter, not the source.
::: {.req label="…"}   → accent banner; `label` is the mono caption on the left.
::: note / {.note .ink} → amber / indigo callout. Leading **bold sentence.** is the callout title.
::: why / vsc / faults  → card grids; each #### heading starts a card.
::: {.fault .amber}     → fault card (default red top border; .amber = amber). Nested ::: cause is the grey cause line.
::: path               → UI breadcrumb. [text]{.dim} = grey separator/prefix; **text** = boxed UI element.
::: leads / footer     → left-accent box / page footer.
[text]{.mins .gb}      → inline "Git Bash" pill.  [X]{.flag .g|.u|.e|.n} → status letter badge.  [text]{.sub} → small grey sub-line.

Terminal blocks — ```{.console chrome="wt|wt-mini|vscode" title="…" cwd="…" branch="…"}
  chrome="wt"      Windows Terminal tab strip (Git icon, tab title, +, ▾, window controls)
  chrome="wt-mini" tab only, no buttons/window controls
  chrome="vscode"  VS Code panel strip: PROBLEMS · OUTPUT · TERMINAL(active) · "bash" profile
  Every line starting with "$ " gets the Git Bash PS1 prompt line generated above it:
      you@WORKSTATION (green) MINGW64 (magenta) {cwd} (yellow) ({branch}) (cyan, only if set)
  Flags (-x / --xyz) → .f;  trailing "# …" → .c (dim);  quoted printf format → .c
  Output lines starting "Good " → .g (green).
  {.audit} blocks: 2nd whitespace field G → .g, N → .r.
Pipe-table rows whose last cell is empty merge the last two cells (colspan=2).
-->

::: {.req label="Terminal requirement"}
**Run every command in this runbook from Git Bash.** Git Bash is the terminal app that *Git for Windows* installs — `C:\Program Files\Git\git-bash.exe`, listed in the Start menu as **Git Bash**. Launch it directly, open the **Git Bash** profile inside Windows Terminal, or open it as a terminal tab inside **VS Code**; all three are the same shell, and the prompt must read `MINGW64` before you type anything.

**PowerShell and Command Prompt will not do.** There `printf` does not exist and PowerShell\ 5.1 writes UTF-16 files that OpenSSH cannot read — both break step 4 — and the `ssh-keygen` you type resolves to the older copy Windows ships under `System32\OpenSSH`, not the one Git signs with. Every terminal window pictured in this runbook is a Git Bash session, and each step that needs one is marked [Git Bash]{.mins .gb}.
:::

## Preflight — 30 seconds {#preflight}

:::: pre
::: col

```{.console chrome="wt" title="MINGW64:/c/Users/you" cwd="~"}
$ git --version
git version 2.56.0.windows.3

$ ssh -V
OpenSSH_10.6p1, OpenSSL 3.5.9 29 Sep 2026

$ which ssh-keygen
/usr/bin/ssh-keygen              # Git's own build — the one you want
```

::: {.note .ink}
**Opening Git Bash.** Three routes, one shell: launch it on its own from the Start menu › **Git Bash**; in Windows Terminal, use the chevron beside `+` and pick the **Git Bash** profile the Git for Windows installer registers; or in VS Code, **Terminal › New Terminal** and pick the same profile from the chevron beside `+` — see *If VS Code is your primary surface* below. The terminals pictured throughout are the Windows Terminal tab; a standalone Git Bash window and a VS Code terminal tab behave identically. Check the prompt reads `MINGW64` before running the three commands above. A PowerShell or Command Prompt window will not do: an `ssh-keygen` you type there resolves to a different, older copy under `System32\OpenSSH`. Git itself never runs that copy, but the commands in this runbook would.
:::

::: note
**Below Git 2.34?** Update in place from Git Bash, then reopen it — the command prompts before downloading anything:

`git update-git-for-windows`
:::

:::
::: {.col .vers}

| Tool | You need | Latest today |
|------|----------|--------------|
| **Git for Windows**<br>[provides Git Bash — the required terminal]{.sub} | [**2.34**]{.min}<br>[SSH signing added]{.sub} | [**2.56.0(3)**]{.lat}<br>[per winget, 26 Oct 2026]{.sub} |
| **OpenSSH**<br>[bundled with Git — does the signing]{.sub} | [**8.1**]{.min}<br>[signing added]{.sub} | [**10.6P1**]{.lat}<br>[ships with the above]{.sub} |
| **Windows Terminal**<br>[optional host, not the shell]{.sub} | [Optional — Git Bash also runs standalone. Any current version; ships with Windows 11, verified on **1.24**.]{.sub} | |
| **VS Code**<br>[optional surface, not the shell]{.sub} | [Optional. Any version whose built-in Git extension is enabled; nothing to install, nothing to switch on. Verified on **1.135.0**.]{.sub} | |
| **GnuPG / Gpg4win** | [Not required. SSH signing needs no keyring, no admin rights.]{.sub} | |
| **GitHub** | [An account with your commit email verified on it.]{.sub} | |

:::
::::

::: why

#### Nobody can commit as you

Today `git config user.name` takes any value. Signing replaces a typed claim with a key only you hold.

#### Your commits show Verified

On GitHub, on every pull request, and in any clone. Reviewers stop having to take authorship on trust.

#### It can't be applied later

History cannot be signed retroactively without rewriting it. Every unsigned commit stays unverifiable permanently.

:::

## Setup {#setup}

### Generate a signing key {.step minutes="2 min" shell="git-bash"}

Open **Git Bash** — prompt reading `MINGW64` — and run the command below there, not in PowerShell or Command Prompt. Press Enter to accept the default path. Already have an SSH key you like? Skip this step and use it — one key can do both authentication and signing.

```{.console chrome="wt" title="MINGW64:/c/Users/you" cwd="~"}
$ ssh-keygen -t ed25519 -C "you@example.com"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/c/Users/you/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Your identification has been saved in /c/Users/you/.ssh/id_ed25519
Your public key has been saved in /c/Users/you/.ssh/id_ed25519.pub
```

::: note
**About the passphrase.** A passphrase is more secure, but every commit then needs it: in Git Bash you load the key into an agent once per window (see *Faults* below), and in VS Code you get a prompt on every single commit instead (see *If VS Code is your primary surface*). Other Git GUIs may have nowhere to show that prompt at all and simply fail. On an encrypted laptop, leaving it empty is a reasonable trade.
:::

### Tell Git to use it {.step minutes="1 min" shell="git-bash"}

Five lines, once, globally, again from a **Git Bash** window. After this every commit and tag you make is signed automatically — no `-S` flag to remember.

```{.console chrome="wt" title="MINGW64:/c/Users/you" cwd="~"}
$ git config --global gpg.format ssh

$ git config --global user.signingkey ~/.ssh/id_ed25519.pub

$ git config --global commit.gpgsign true

$ git config --global tag.gpgSign true

$ git config --global gpg.ssh.program \
    "C:/Program Files/Git/usr/bin/ssh-keygen.exe"
```

Point `user.signingkey` at the **.pub** file. Keep the `~/.ssh/…` form: Git expands the `~`, so the config survives a different Windows username. An absolute path also works if written with forward slashes, `C:/Users/you/.ssh/…` — typed unquoted in Git Bash, backslashes are stripped before Git ever sees them.

::: {.note .ink}
**Why the fifth line.** Git signs by running `ssh-keygen`. Started through Git's own launcher, `C:\Program Files\Git\cmd\git.exe` — which is what `git` means in PowerShell, Command Prompt, Git Bash and VS Code — Git already finds its bundled copy, because the launcher puts Git's `usr\bin` first on the `PATH` of everything Git runs. The pin is insurance for any Git started another way, such as a GUI that ships its own Git: it names one absolute path, so every route signs with the same OpenSSH and never the older one in `System32\OpenSSH`. Use forward slashes; quote the path for the space in *Program Files*.
:::

### Register the key on GitHub {.step minutes="1 min" shell="git-bash"}

Print your public key in **Git Bash** and copy it, then add it to your account — making sure to choose the **signing** key type, not authentication. Only the copy is a browser step; the command stays in Git Bash.

```{.console chrome="wt" title="MINGW64:/c/Users/you" cwd="~"}
$ cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI… you@example.com
```

Select the line and copy it — **Ctrl+Shift+C** in a Windows Terminal tab; plain Ctrl+C sends an interrupt to the shell instead of copying. Or skip the mouse: in Git Bash, `clip < ~/.ssh/id_ed25519.pub` puts the key straight on the Windows clipboard.

::: path
[GitHub]{.dim} Settings [›]{.dim} **SSH and GPG keys** [›]{.dim} New SSH key [›]{.dim} Key type: **Signing key** [›]{.dim} paste [›]{.dim} Add
:::

::: note
**Reusing your existing auth key?** GitHub stores authentication and signing keys separately, so you must upload the same public key a second time with the type set to *Signing key*. It will not be used for signing otherwise.
:::

### Tell Git whose signatures to trust {.step minutes="1 min" shell="git-bash"}

This step is missing from most guides and it is the one that confuses everybody. Without it, Git signs your commits correctly but **cannot verify them** — `git log` reports *“No signature”* on a commit that is perfectly signed. Add yourself to an allowed-signers file:

```{.console chrome="wt" title="MINGW64:/c/Users/you" cwd="~"}
$ printf '%s %s\n' "you@example.com" "$(cat ~/.ssh/id_ed25519.pub)" > ~/.ssh/allowed_signers

$ git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

To verify a teammate's commits too, append one line per person: their email, a space, then their public key.

::: note
**These two lines must run in Git Bash — not PowerShell, not Command Prompt.** `printf` does not exist there, and PowerShell\ 5.1's `>` writes UTF-16 — which OpenSSH cannot read, so verification fails with no useful error.
:::

### Prove it works {.step minutes="1 min" shell="git-bash"}

In **Git Bash**, `cd` into any repo, make a commit, then ask Git to check its own work. You want the word **Good**.

```{.console chrome="wt" title="MINGW64:/e/repos/vdc" cwd="/e/repos/vdc" branch="feature/signing"}
$ git commit -m "chore: verify commit signing"
[feature/signing 518b8fd] chore: verify commit signing
 1 file changed, 1 insertion(+)

$ git log -1 --show-signature
commit 518b8fd902216e01582bfe6dd3b426e3b7f5f622
Good "git" signature for you@example.com with ED25519 key SHA256:tOO/rn…
```

Push it, and GitHub shows a green **Verified** badge next to the commit. You are done — every commit from this laptop is signed from now on.

## If VS Code is your primary surface {#vscode}

::: {.req label="Same five steps"}
**Nothing in the setup changes, and nothing inside VS Code has to be switched on.** When you commit from the **Source Control** panel, VS Code runs the same `git.exe` against the same global config you set in step\ 2 — there is no separate VS Code signing mechanism to configure. Finish the five steps and your panel commits are signed.

**Run those five steps in VS Code's own Git Bash tab** and you never leave the editor: **Terminal › New Terminal**, then the profile chevron beside `+` › **Git Bash**. VS Code auto-detects that profile whenever Git for Windows is present and starts it as a login shell, so the prompt reads `MINGW64` and every command in this runbook behaves exactly as pictured. To stop picking it each time, run **Terminal: Select Default Profile** from the Command Palette, or set `terminal.integrated.defaultProfile.windows` to `Git Bash`.
:::

::: vsc

#### Leave `git.enableCommitSigning` alone

It ships `false`, and false does **not** mean unsigned: on false VS Code passes no signing flag at all, so `commit.gpgsign true` from step 2 governs and the commit is signed. Setting it `true` only adds `-S` to the same command. It is not the switch that makes signing work — flip it without step 2 and nothing is signed.

#### A passphrase means a prompt on every commit

VS Code gives its Git child processes its own askpass helper and requires OpenSSH to use it, so an encrypted key does not fail silently here — you get a masked **Passphrase** box labelled with the key's path. It is not cached: expect it on every commit, in the panel and in the integrated terminal alike. That is the default (`git.useIntegratedAskPass`); turn it off and the commit fails instead.

#### The panel never shows signature state

The built-in Git extension has no verification UI — no *Verified* badge in Source Control, in the graph, or in Timeline, and no warning on an unsigned commit. Read signatures in the integrated Git Bash tab, as below, or on GitHub.

#### When a commit fails, open the Git log

VS Code reports it as a modal *Failed to execute git* dialog that says nothing about signing. The buttons on that dialog are the fix: **Open Git Log** shows the Git output channel, **Show Command Output** shows the failing command's `stderr` — which is where the real `ssh-keygen` error is.

#### Remote-SSH, WSL and Dev Containers don't inherit this

In those windows Git runs on the remote machine or in the container, with its own `~/.ssh` and its own global config. Your Windows setup does not travel: run all five steps again from a shell inside that environment, and register nothing new on GitHub — one signing key per machine is the norm, but the key file has to exist where Git runs.

#### Signing config cannot ship in `.vscode/settings.json`

Every part of this runbook lives in Git's global config and in `~/.ssh`, per machine and per person. No workspace setting can sign a teammate's commits for them, so don't try to distribute it with the repo — send them this runbook.

:::

Verifying without leaving the editor — the integrated **Git Bash** tab is a Git Bash shell, so step 5's check works unchanged:

```{.console chrome="vscode" profile="bash" cwd="/e/repos/vdc" branch="feature/signing"}
$ git log -1 --show-signature
commit 518b8fd902216e01582bfe6dd3b426e3b7f5f622
Good "git" signature for you@example.com with ED25519 key SHA256:tOO/rn…
```

::: {.note .ink}
**Commit from the panel, verify in the tab.** Stage and commit however you prefer — **Commit**, **Commit Staged** or **Commit (Amend)** all run the same `git commit`, so all three are signed. Then run the command above, or the `%G?` audit in the next section, to see it.
:::

## Reading the signature status of any commit — from Git Bash {#status}

```{.console .audit chrome="wt" title="MINGW64:/e/repos/vdc" cwd="/e/repos/vdc" branch="master"}
$ git log --pretty='%h %G? %an' -10    # audit the last ten commits
518b8fd G A. Developer
0a246bf G A. Developer
4d7da14 N B. Developer
10a7ffc N B. Developer
```

| Code | Means | What to do |
|------|-------|------------|
| [G]{.flag .g} | **Good signature** from a signer you trust. | Nothing. This is the target state. |
| [U]{.flag .u} | **Valid signature, unknown signer.** The commit is signed, but that key isn't in your allowed-signers file. | Append that person's email and public key to `~/.ssh/allowed_signers`. |
| [E]{.flag .e} | **Signature present, key unavailable.** Almost always a commit made in the GitHub web editor — signed by GitHub's key, not by a person. | Nothing you can fix. Worth knowing this is *not* a developer signature. |
| [N]{.flag .n} | **No signature.** Either the commit genuinely isn't signed, or you skipped step 4 and Git can't check. | Confirm step 4 is done, then re-run. If it still says `N`, that commit is unsigned. |

## Faults, and what they actually mean {#faults}

::::: faults

:::: fault

#### fatal: failed to write commit object

::: cause
Your key has a passphrase and no agent is running, so Git can't unlock it.
:::

Load the key into an agent, once per Git Bash window:

```{.console chrome="wt-mini" title="MINGW64:/c/Users/you" cwd="~"}
$ eval "$(ssh-agent -s)"
Agent pid 1284

$ ssh-add ~/.ssh/id_ed25519
```

That agent belongs to the window you started it in. Nothing outside inherits it — not another Git Bash window, and not VS Code's Git extension, which prompts for the passphrase itself instead (see *If VS Code is your primary surface*). Git Bash's `ssh-agent` is also its own thing: it is not the Windows *OpenSSH Authentication Agent* service, which stays disabled and needs admin rights to start.

::::

:::: {.fault .amber}

#### “No signature” — but GitHub says Verified

::: cause
Step 4 was skipped. Git has no allowed-signers file, so it declines to verify anything.
:::

You may also see: `error: gpg.ssh.allowedSignersFile needs to be configured and exist for ssh signature verification`. Your commits are fine; only local verification is off.

If the file exists and verification still fails, check how it was written: a copy created from PowerShell is UTF-16, which OpenSSH cannot read (see step 4). Re-run the `printf` line in Git Bash to overwrite it.

::::

:::: {.fault .amber}

#### Signing works in one shell, not another

::: cause
Usually a passphrase key that is loaded into an agent in one window only.
:::

Git started through its launcher signs with its own bundled `ssh-keygen` from every shell and from VS Code's Git extension alike, so the shell is rarely the cause. If your key has a passphrase, the window where signing works is the one where you ran `ssh-add`; see the first fault above. The exception is a Git GUI that ships its own Git and bypasses the launcher: step 2's `gpg.ssh.program` line covers it, and the next commit picks it up — Git re-reads its config on every invocation, so there is nothing to restart. Typing `ssh-keygen` yourself in PowerShell or Command Prompt still gets the older `System32\OpenSSH` copy, which is why the steps above specify Git Bash.

::::

:::: {.fault .amber}

#### GitHub shows Unverified

::: cause
Two usual causes, both on the GitHub side.
:::

Either the key was uploaded as an **authentication** key rather than a signing key (step 3), or the email in your commits isn't one of the verified addresses on your GitHub account. Check `git config user.email`.

::::

:::: {.fault .amber}

#### Commits show no badge at all

::: cause
Unsigned commits are invisible by default rather than flagged.
:::

Turn on **Vigilant mode** (GitHub Settings › SSH and GPG keys) and unsigned commits start showing as *Unverified* instead of showing nothing. Recommended once you're signing.

::::

:::::

::: leads

### If you lead a repo, read this before enforcing it {.unnumbered}

- Get everyone signing **first**, verify it for a week, then turn on enforcement. Enabling it first blocks your team mid-release.
- With **Require signed commits** on a branch, GitHub will not let anyone squash-and-merge a pull request into it unless they are the PR author — and rebase-and-merge is blocked outright, because GitHub cannot sign that commit. If you rely on squash merges or on a reviewer pressing Merge, enforce signing on your release branch only.
- Commits that GitHub itself creates — web-editor edits and merges — are signed with *GitHub's* key. They satisfy the rule without proving any developer signed anything. The value is in the signed commits on the topic branch underneath.
- Some of your developers never open a terminal — they stage and commit in VS Code's Source Control panel. Rolling this out as “open Git Bash and run these commands” is where those people stall. Point them at the VS Code section: the setup is the same five steps, run in a Git Bash tab inside their editor, and there is no VS Code setting for them to hunt for.
- Fix duplicate author identities with a `.mailmap` before you start. If one person commits under two names or two addresses, signature data is unreadable.

:::

::: footer
**Commit signing runbook** · SSH signing from Git Bash on Windows, for GitHub · covers Git Bash, Windows Terminal and VS Code as commit surfaces · v1.5, 26 October 2026

Every command verified end-to-end on **Git 2.55.0.windows.3 / OpenSSH_10.3p1 / Windows Terminal 1.24 / Windows 11** on 2026-08-31, including the failure modes above: a commit signed with `gpg.ssh.program` pinned to Git's bundled `ssh-keygen` returns `Good "git" signature` and `%G?` of `G`. Re-checked on 2026-10-26 on **Git 2.56.0.windows.3 / OpenSSH_10.6p1**: a commit signed by this setup returns `Good "git" signature` and `%G?` of `G`, and from PowerShell, where `ssh-keygen` resolves to `System32\OpenSSH`, a Git alias probe (`!command -v ssh-keygen`) prints `/usr/bin/ssh-keygen` and shows `/ucrt64/libexec/git-core`, `/ucrt64/bin`, `/usr/bin` leading the `PATH` Git hands its children. The full Setup sequence was not re-run on 2.56. Versions in Preflight are as of 2026-10-26: **Git for Windows 2.56.0(3)**, the newest in the winget catalogue that day, bundling **OpenSSH 10.6P1** and built on upstream Git **2.56.0**. Minimums per **GitHub Docs** (Git 2.34, SSH signature verification) and the **OpenSSH 8.1** release notes, 9 Oct 2019, which introduced signing. Key-validity windows in an allowed-signers file need **OpenSSH 8.8+**.

The prompt shown is Git Bash's default `PS1` from **/etc/profile.d/git-prompt.sh** — [user@host]{.ok}, [MINGW64]{.accent}, path, then branch — and the tab title is that file's `$TITLEPREFIX:$PWD`. The **Git Bash** profile reaches Windows Terminal as a fragment installed by Git for Windows at **%ProgramData%\\Microsoft\\Windows Terminal\\Fragments\\Git\\git-bash.json**; Git Bash itself is **C:\\Program Files\\Git\\git-bash.exe**, listed in the Start menu as **Git Bash**, and it is the same shell whether launched standalone or as that profile. Windows ships its own separate OpenSSH under **System32\\OpenSSH**, an older build that other Windows Terminal profiles put first on `PATH`. Git's launcher, **C:\\Program Files\\Git\\cmd\\git.exe**, puts Git's own **usr\\bin** ahead of it for everything Git runs, which is what keeps Git off it; step 2's pin covers any Git started without that launcher.

The VS Code section was written against the built-in **Git extension** shipped in **VS Code 1.135.0** (build 08d4889f9e), reading its own `package.json`, `package.nls.json` and `dist/main.js` under **resources\\app\\extensions\\git** rather than taking the behaviour on trust. From that source: `git.enableCommitSigning` defaults to **false**, and the extension appends `-S` only when it is true and `--no-gpg-sign` only when signing is explicitly declined, so the default passes no flag and leaves `commit.gpgsign` in charge; `git.useIntegratedAskPass` defaults to **true**, and on that default the extension merges `SSH_ASKPASS` plus `SSH_ASKPASS_REQUIRE=force` into the environment of the Git processes it spawns, its askpass handler answering any prompt matching *passphrase* with a masked input box captioned by the key path and caching nothing; a failed Git command raises a modal *Failed to execute git* dialog carrying **Open Git Log** and, when there is `stderr`, **Show Command Output**; and **Git Bash** is an auto-detected `terminal.integrated.profiles.windows` entry launched as a login shell. The extension carries no commit-signature verification UI at that version. **Scope of that check:** those five behaviours were verified by reading the shipped extension source on 2026-09-03, not by re-running a commit through the Source Control panel; the command sequence in Setup was executed end-to-end in Git Bash as described above.

Configuration keys per the **git-config** documentation; UI steps and enforcement behaviour per **GitHub Docs**. Produced with **Claude Code (CLI)**, model **claude-opus-5[1m]**; v1.5 revised with model **claude-opus-5-5**.
:::
