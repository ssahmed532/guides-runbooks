---
title: uv-Only Python Playbook
eyebrow: Playbook · Windows 11 · local developer laptop
version: "1.3"
last_updated: 2026-09-10
status: executed and verified, two open items
owner: Salman Ahmed
machine_scope: single laptop, user profile
source_artifact: https://claude.ai/code/artifact/7f460654-de14-4745-b702-1b36ea09d514
source_version: 1789059199-1be5
dialect: Pandoc Markdown (fenced divs, bracketed spans, heading attributes, pipe tables, raw HTML)
---

<!--
  RECREATION NOTES
  - Content below is verbatim from the published artifact.
  - Layout/visual annotations use Pandoc Markdown syntax:
      ::: {.class}  -> <div class="class">          [text]{.class} -> <span class="class">
      ## Heading {#id} -> section anchor              - [x] / - [ ] -> li.done / li.open
  - Appendix A holds the exact stylesheet and fonts; Appendix B holds the exact HTML
    each annotation maps to. Together they reproduce the original page 1:1.
-->

:::::: {.wrap}

::::: header

[Playbook · Windows 11 · local developer laptop]{.eyebrow}

# uv-Only Python Playbook {style="margin-top:10px"}

::: lede
One interpreter source, one package manager, no global site-packages. Every Python app, tool, and script on this laptop runs through uv against the current CPython 3.14.x release.
:::

::: meta
[Version **1.3**]{} [Last updated **2026-09-10**]{} [Status **executed and verified, two open items**]{} [Owner **Salman Ahmed**]{} [Machine scope **single laptop, user profile**]{}
:::

:::::

::::: {.page style="margin-top:40px"}

:::: {.toc aria-label="Contents"}

[Contents]{.eyebrow}

1. [Current vs target state](#state)
2. [Decisions and trade-offs](#decisions)
3. [1 · Update uv](#p1)
4. [2 · Install managed 3.14.7](#p2)
5. [3 · Purge the user-site leak](#p3)
6. [4 · Remove the other interpreters](#p4)
7. [5 · Clean PATH](#p5)
8. [6 · Guardrails](#p6)
9. [7 · Verify end state](#p7)
10. [8 · Working conventions](#p8)
11. [9 · Migrate the 39 packages](#p9)
12. [10 · Editor](#p10)
13. [11 · Maintenance cadence](#p11)
14. [Definition of done](#done)
15. [Execution record](#record)

::: sub
Commands were checked against uv 0.9.29 and 0.12.11 help output, the Python 3.14 binaries on this machine, winget, Chocolatey 2.7.0, and the uv documentation, then executed on this laptop on 2026-09-09 and 2026-09-10. Items marked *preview* are flagged as such by Astral.
:::

::::

:::: main

<!-- ============================ STATE ============================ -->

::: {#state}

[Inventory taken 2026-09-09]{.eyebrow}

:::: state

::: {.panel .now}

### What was on the laptop

- **Three CPython 3.14 installs:** python.org MSI 3.14.7 in Program Files, Python Install Manager 3.14.2 under AppData\\Local\\Python, and a uv-managed 3.14.2 under AppData\\Roaming\\uv.
- **Two `py` launchers:** the legacy one in C:\\Windows and the new Install Manager one in WindowsApps.
- **A dead Chocolatey Python:** packages `python`, `python3`, `python314` at 3.14.5 whose target `C:\python314` no longer exists, leaving a broken `python3.14.exe` shim in `C:\ProgramData\chocolatey\bin`, which is on the machine PATH.
- **39 third-party packages** in the per-user site directory `%APPDATA%\Python\Python314\site-packages`. That directory is keyed by 3.14 only, so it is imported by *all three* interpreters, including uv's.
- **uv 0.9.29** from 2026-02-03. Its download catalog stops at CPython 3.14.2. Latest uv is 0.12.11 (2026-09-08).
- **PATH** carries five Python-related entries across user and machine scope.
- No uv.toml, no pip.ini, no PYTHON\* variables. No hooks, scheduled tasks, or shell profiles call Python. VS Code has the Python, Python Environments, Pylance, debugpy, and Ruff extensions.

:::

::: {.panel .target}

### Where it ended, verified 2026-09-10

- **One interpreter:** uv-managed CPython 3.14.7, upgraded in place by uv as 3.14.8 and later ship.
- **`python`, `uv`, and the uv tools resolve from one directory:** `%USERPROFILE%\.local\bin`. Nothing else Python-related on PATH. The only remnant is an empty `C:\Program Files\Python314` shell awaiting an elevated delete.
- **No user-site, no global packages.** Every dependency is declared in a pyproject.toml, an inline script block, or a uv tool environment.
- **Guardrails** make the old failure modes fail loudly: pip refuses to run outside a virtual environment, Python never reads a user-site directory, uv never falls back to a stray system Python.
- **A written convention** for apps, scripts, and CLI tools, plus a monthly upkeep routine.

:::

::::

:::

<!-- ========================== DECISIONS ========================== -->

## Decisions and trade-offs {#decisions}

Three calls were open at the start. The first option in each pair is what was chosen and is now in place; the alternatives remain valid if a constraint changes.

:::: decision

::: panel

### Interpreter source [taken]{.pill .tool}

**uv-managed builds only.** uv installs Astral's python-build-standalone distributions, not python.org binaries. They track upstream within days and are what uv is designed around. The cost: they are not the official installer, so if a corporate control ever requires python.org signed binaries, use the alternative.

[**Alternative:** keep the python.org MSI 3.14.7 as the one system interpreter and set `python-preference = "only-system"`. You then own patch upgrades manually through winget.]{.why .p}

:::

::: panel

### Python Install Manager [taken]{.pill .tool}

**Remove it.** It is Microsoft Store tooling that puts its own `python` and `py` on PATH and downloads runtimes outside uv's control. Keeping it means two managers competing for the same names.

[**Alternative:** keep it and set its runtime aside. You will need to remove its bin directory from PATH and accept a second `py` command.]{.why .p}

:::

::: panel

### `python` shim on PATH [taken]{.pill .tool} [preview]{.pill .dev}

**Install it.** The `--default` flag places unversioned `python` and `python3` executables in uv's bin directory. Astral marks this experimental. It is a convenience for one-liners; nothing in this playbook depends on it, because projects and scripts go through `uv run`.

[**Alternative:** skip `--default` and type `uv run python` for ad-hoc use. Zero preview surface.]{.why .p}

:::

::: panel

### Network [clear]{.pill .proj}

uv downloads interpreters from GitHub releases and packages from PyPI. Phase 1 will show immediately if a corporate proxy or TLS inspection blocks either. On this laptop both worked first time: the uv self-update, the 3.14.7 download, and PyPI fetches all completed without a proxy or certificate error.

:::

::::

<!-- =========================== PHASE 1 =========================== -->

:::: {#p1 .phase}

::: phase-head
[01]{.phase-num}

## Update uv
:::

::: why
The installed uv is seven months old and does not know CPython 3.14.3 through 3.14.7 exist. Everything downstream depends on this step. uv was installed with the standalone installer, so it can update itself.
:::

[PowerShell 7]{.shell}

```powershell
uv self update
uv --version                                  # expect 0.12.x
uv python list --only-downloads | Select-String "cpython-3.14"
```

::: verify
[Verify]{.eyebrow}

The download list now includes `cpython-3.14.7-windows-x86_64-none`. If it still stops at 3.14.2, the update did not take; check the output of the first command for a proxy or certificate error.
:::

::::

<!-- =========================== PHASE 2 =========================== -->

:::: {#p2 .phase}

::: phase-head
[02]{.phase-num}

## Install the managed CPython 3.14.7
:::

::: why
Do this before removing anything, so the laptop always has a working interpreter. Requesting `3.14` gives the newest patch uv knows. The `--default` flag also writes `python` and `python3` shims into uv's bin directory (preview, see Decisions). The stale managed 3.14.2 is removed so only one version remains.
:::

[PowerShell 7]{.shell}

```powershell
uv python install 3.14 --default --preview-features python-install-default
uv python update-shell                         # ensures %USERPROFILE%\.local\bin is on PATH
uv python uninstall 3.14.2                     # may say "No existing installations found": uv 0.12 already retired it
uv python list --only-installed
```

::: callout
**Expected warning on this laptop.** The install may end with `Failed to install executable … Failed to update Windows PE resources … Access is denied`. Trend Micro Behavior Monitoring blocks uv's final step of stamping version resources into a freshly written shim in `%TEMP%`. This is upstream uv issue #20955 (open, no fix yet). The shim itself is already written and works; the only loss is its version metadata. Confirm by rerunning the same install command: it should report `Python 3.14 is already installed` with all three shims recognised.
:::

::: verify
[Verify]{.eyebrow}

Exactly one managed entry appears: `cpython-3.14.7-windows-x86_64-none` under `%APPDATA%\uv\python`, plus a `cpython-3.14-windows-x86_64-none` junction pointing at it, which is how uv 0.12 delivers transparent patch upgrades. The system installs still show in the list until Phase 4; that is expected. So is a new `Astral\CPython3.14.7` row in the Install Manager's `py list`: uv 0.12 registers its interpreter with Windows under `HKCU\Software\Python\Astral`.
:::

::::

<!-- =========================== PHASE 3 =========================== -->

:::: {#p3 .phase}

::: phase-head
[03]{.phase-num}

## Purge the user-site leak
:::

::: why
The 39 packages live in one directory that pip owns entirely. Deleting the directory is cleaner and faster than 39 uninstalls. Freeze a manifest first; Phase 9 uses it. The directory also holds the `Scripts` folder with speedtest, pytest, and pygmentize shims, which go with it.
:::

[PowerShell 7]{.shell}

```powershell
& "C:\Program Files\Python314\python.exe" -m pip freeze --user |
  Out-File "$env:USERPROFILE\pip-user-site-manifest-2026-09-09.txt"
Remove-Item -Recurse -Force "$env:APPDATA\Python\Python314"
```

::: callout
**Expected breakage.** Any script that was silently relying on these packages now raises `ModuleNotFoundError`. That is the signal you want. Fix the script by declaring its dependencies (Phase 8), not by reinstalling globally.
:::

::: verify
[Verify]{.eyebrow}

The manifest file has 40 lines, the 39 packages plus pip itself, and `%APPDATA%\Python` no longer contains a `Python314` folder. `python -c "import anthropic"` now fails from every interpreter, uv's and the system ones alike.
:::

::::

<!-- =========================== PHASE 4 =========================== -->

:::: {#p4 .phase}

::: phase-head
[04]{.phase-num}

## Remove the other interpreters and launchers
:::

::: why
Five things go: the Install Manager's 3.14.2 runtime, the Install Manager itself, the python.org 3.14.7 install, the legacy Python Launcher, and the dead Chocolatey packages. The Install Manager runtime must be removed through its own `py` command first; winget cannot see it. Order matters: uninstall the runtime before the manager that owns it.
:::

[PowerShell 7 · elevated for the winget and choco steps]{.shell}

```powershell
# 4a. Install Manager runtime (it will prompt to confirm)
py uninstall 3.14

# 4b. The manager, the python.org install, and the legacy launcher
winget uninstall --id Python.PythonInstallManager --exact
winget uninstall --id Python.Python.3.14 --exact
winget uninstall --id Python.Launcher --exact

# 4c. Chocolatey metadata for a Python that is no longer on disk.
#     The three packages chain as dependencies; remove them together and skip
#     the auto-uninstaller because there is no installer left to run.
choco uninstall python --remove-dependencies --skip-autouninstaller -y

# 4d. Fallbacks, both needed on this laptop's first pass
winget uninstall --id Python.Python.3.14 --exact --version 3.14.7    # if winget reports more than one match
Get-AppxPackage -Name PythonSoftwareFoundation.PythonManager | Remove-AppxPackage   # if the Install Manager survives winget; runs unelevated
```

::: callout
**The Store alias will shadow uv's shim.** The Install Manager's own `py`, `pyw`, and `pythonw` aliases disappear with its package. The `python.exe` and `python3.exe` aliases in `%LOCALAPPDATA%\Microsoft\WindowsApps` belong to Microsoft's App Installer and stay. Because WindowsApps sits ahead of `.local\bin` on the user PATH, a fresh shell's `python --version` prints `Python was not found; run without arguments to install from the Microsoft Store…` with exit code 9009. Open Settings → Apps → Advanced app settings → App execution aliases and switch off both entries under App Installer. No command does this reliably; it is the one manual step in the playbook.
:::

::: verify
[Verify]{.eyebrow}

`winget list --name Python` returns nothing and no Python entries remain in the Windows uninstall registry. `choco list` shows no python packages and `C:\ProgramData\chocolatey\bin\python3.14.exe` is gone. `Get-Command py -ErrorAction SilentlyContinue` returns nothing. After the alias toggle, a fresh shell's `python --version` prints 3.14.7. Two husks remain for Phase 5: `%LOCALAPPDATA%\Python` with an empty `bin` and a cache file, and `C:\Program Files\Python314` as three empty directories.
:::

::::

<!-- =========================== PHASE 5 =========================== -->

:::: {#p5 .phase}

::: phase-head
[05]{.phase-num}

## Clean PATH
:::

::: why
Uninstallers usually remove their PATH entries but not always. Two user-scope entries and two machine-scope entries need to be gone, plus the empty folders the uninstallers left behind. Keep `%USERPROFILE%\.local\bin`; it is where uv, uvx, the Python shims, and uv tools live. Keep `C:\ProgramData\chocolatey\bin` too, since other Chocolatey packages use it; Phase 4c removes the Python shim from inside it.
:::

[PowerShell 7 · user scope]{.shell}

```powershell
$keep = [Environment]::GetEnvironmentVariable('Path','User') -split ';' |
  Where-Object { $_ -and $_ -notmatch 'AppData\\Local\\Python\\bin|Roaming\\Python\\Python314' }
[Environment]::SetEnvironmentVariable('Path', ($keep -join ';'), 'User')
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\Python", "$env:APPDATA\Python"
```

[PowerShell 7 · elevated, machine scope]{.shell}

```powershell
# On this laptop the python.org uninstaller removed both machine PATH entries itself; keep this for when it does not.
$keep = [Environment]::GetEnvironmentVariable('Path','Machine') -split ';' |
  Where-Object { $_ -and $_ -notmatch 'Program Files\\Python314' }
[Environment]::SetEnvironmentVariable('Path', ($keep -join ';'), 'Machine')
Remove-Item -Recurse -Force "C:\Program Files\Python314"        # empty shell left by the uninstaller
```

Close every terminal and VS Code window before continuing. PATH changes apply to new processes only.

::: verify
[Verify]{.eyebrow}

In a fresh shell, `$env:Path -split ';' | Select-String -Pattern 'Python|uv|\.local'` shows only `%USERPROFILE%\.local\bin`.
:::

::::

<!-- =========================== PHASE 6 =========================== -->

:::: {#p6 .phase}

::: phase-head
[06]{.phase-num}

## Guardrails
:::

::: why
Removing the mess is not the same as preventing it. Three settings make each old failure mode impossible or loud. They cost nothing day to day because `uv run`, `uv add`, and `uv tool` never trip them.
:::

### 6a. uv never falls back to a system Python

User-level uv configuration lives at `%APPDATA%\uv\uv.toml` and applies to every project unless a project overrides it. `only-managed` means a future stray install (an IDE bundling Python, a Store alias reappearing) is ignored rather than adopted.

[PowerShell 7]{.shell}

```powershell
New-Item -ItemType Directory -Force "$env:APPDATA\uv" | Out-Null
@'
python-preference = "only-managed"
python-downloads = "automatic"
'@ | Set-Content "$env:APPDATA\uv\uv.toml"
```

### 6b. Python never reads a user-site directory

`PYTHONNOUSERSITE` is a documented interpreter variable equivalent to the `-s` flag. With it set, even a future accidental `pip install --user` cannot leak into other environments.

### 6c. pip refuses to run outside a virtual environment

pip reads every long option from an environment variable of the form `PIP_<OPTION>`. This matters because uv's managed interpreter ships with pip 26.2.1, so `python -m pip` still exists. Inside a uv-created `.venv` the condition is satisfied, so project work is unaffected; a bare `pip install` on the managed interpreter fails with `ERROR: Could not find an activated virtualenv (required).`

[PowerShell 7]{.shell}

```powershell
[Environment]::SetEnvironmentVariable('PYTHONNOUSERSITE', '1', 'User')
[Environment]::SetEnvironmentVariable('PIP_REQUIRE_VIRTUALENV', 'true', 'User')
```

::: verify
[Verify]{.eyebrow}

Open a new terminal window from Windows Terminal or Explorer, not a child of an existing shell: user-scope variables reach only processes started fresh after the change. `Get-Content "$env:APPDATA\uv\uv.toml"` shows both keys. `$env:PYTHONNOUSERSITE` prints `1` and `$env:PIP_REQUIRE_VIRTUALENV` prints `true`. `python -c "import site; print(site.ENABLE_USER_SITE)"` prints `False`.
:::

::::

<!-- =========================== PHASE 7 =========================== -->

:::: {#p7 .phase}

::: phase-head
[07]{.phase-num}

## Verify the end state
:::

::: why
Run all of this in a fresh PowerShell 7 window. Every line has one expected outcome; anything else means an earlier phase is incomplete.
:::

[PowerShell 7 · fresh window]{.shell}

```powershell
Get-Command python, python3, uv, uvx | Select-Object Name, Source
# all four resolve under %USERPROFILE%\.local\bin

python --version
# Python 3.14.7

python -c "import sys; print(sys.executable)"
# path under %APPDATA%\uv\python\cpython-3.14.7-...

python -c "import anthropic"
# ModuleNotFoundError  (the leak is gone)

python -m pip install requests
# ERROR: Could not find an activated virtualenv (required).

uv run --with rich python -c "import rich; print('ok')"
# Installed N packages … ok   (an ephemeral dependency resolves without touching anything global)

uv python list --only-installed
# every line reads cpython-3.14.7-...: the install dir, the 3.14 junction, and the three shims. No other version.

uv python find 3.14
# same managed path as above

Get-Command py -ErrorAction SilentlyContinue
# nothing
```

::::

<!-- =========================== PHASE 8 =========================== -->

:::: {#p8 .phase}

::: phase-head
[08]{.phase-num}

## Working conventions
:::

::: why
Three shapes of Python work, three uv idioms. The rule that makes the whole setup hold: a dependency is never installed, it is *declared*, and uv resolves it at run time.
:::

### Apps and libraries: a project with pyproject.toml

Each repo owns a `pyproject.toml`, a `uv.lock`, a `.python-version`, and a `.venv` that uv creates and refreshes on its own. Commit the first three; ignore `.venv`.

[PowerShell 7]{.shell}

```powershell
uv init --app my-service --python 3.14      # or --lib for a package
cd my-service
uv add anthropic azure-identity pydantic
uv add --dev pytest ruff
uv run pytest                                 # syncs .venv, then runs
uv run python -m my_service
uv lock --upgrade                             # deliberate dependency refresh
uv sync                                       # fresh clone: recreate .venv from the lock
```

Never activate the venv by hand as a habit. `uv run` is the entry point; it keeps the environment in step with the lock file on every call.

### Scripts: one file, inline metadata

A standalone script declares its own requirements in a PEP 723 block at the top. uv builds a cached environment for it on first run. No project, no venv folder, nothing global.

[PowerShell 7]{.shell}

```powershell
uv init --script tag-audit.py --python 3.14
uv add --script tag-audit.py azure-identity httpx
uv run tag-audit.py
uv lock --script tag-audit.py                 # optional: pins to tag-audit.py.lock
```

The resulting header, which you can also write by hand:

```python
# /// script
# requires-python = ">=3.14"
# dependencies = [
#   "azure-identity",
#   "httpx",
# ]
# ///
```

For a throwaway one-liner: `uv run --with pandas python -c "..."`. Nothing persists.

### CLI tools: isolated, on PATH, upgradable

Anything you invoke as a command rather than import gets its own environment under `%APPDATA%\uv\tools` and a shim in `.local\bin`. For one-off use, `uvx` runs a tool without installing it. Both tools below were installed on 2026-09-10: ruff 0.16.6 and speedtest-cli 2.1.3.

[PowerShell 7]{.shell}

```powershell
uv tool install ruff
uv tool install speedtest-cli
uvx pygmentize --version                      # run once, keep nothing
uv tool list
uv tool upgrade --all
```

::::

<!-- =========================== PHASE 9 =========================== -->

:::: {#p9 .phase}

::: phase-head
[09]{.phase-num}

## Migrate the 39 packages
:::

::: why
The manifest from Phase 3 sorts into four buckets. Only fourteen packages were ever asked for; the rest arrived as dependencies and need no action.
:::

::: tablewrap

| Bucket | Packages | What to do |
|---|---|---|
| [tool]{.pill .tool} | speedtest-cli | `uv tool install speedtest-cli`. Used for the Wi-Fi diagnosis work. Done 2026-09-10: `speedtest` and `speedtest-cli` resolve from `.local\bin`. |
| [project dep]{.pill .proj} | anthropic, azure-identity, azure-core, msal, pydantic, httpx, requests, lxml, openpyxl, python-docx, PyYAML, Markdown, cryptography, PyJWT | `uv add` in whichever project or script actually imports it, as each repo is next touched. Do not reinstall the full list anywhere; add only what each file uses. |
| [dev dep]{.pill .dev} | pytest | `uv add --dev pytest` per project. Never a tool: it must see the project's own environment. |
| [transitive]{.pill .skip} | annotated-types, anyio, certifi, cffi, charset-normalizer, colorama, distro, docstring_parser, et_xmlfile, h11, httpcore, idna, iniconfig, jiter, msal-extensions, packaging, pluggy, pycparser, pydantic_core, Pygments, sniffio, typing_extensions, typing-inspection, urllib3 | Nothing. uv resolves these from the packages above. |

:::

::::

<!-- =========================== PHASE 10 ========================== -->

:::: {#p10 .phase}

::: phase-head
[10]{.phase-num}

## Editor
:::

::: why
VS Code already has the Python, Python Environments, Pylance, debugpy, and Ruff extensions. The Python Environments extension detects uv and the project `.venv`; nothing needs installing.
:::

- Open each repo at its root so the `.venv` that `uv run` creates is discovered. If the status bar shows a different interpreter, use <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd> → *Python: Select Interpreter* and pick `.venv`.
- Leave the Ruff extension on its bundled binary. The `uv tool install ruff` from Phase 8 serves the terminal and CI parity, not the editor.
- Use the integrated terminal's `uv run` for tests and scripts rather than the run button, so the editor and the shell resolve the same environment.

::::

<!-- =========================== PHASE 11 ========================== -->

:::: {#p11 .phase}

::: phase-head
[11]{.phase-num}

## Maintenance cadence
:::

::: why
Monthly, five commands. Patch upgrades of the interpreter are in preview per Astral, so read the output of the second command before trusting it.
:::

[PowerShell 7 · monthly]{.shell}

```powershell
uv self update
uv python upgrade                             # preview: 3.14.7 → 3.14.8 when released
uv tool upgrade --all
uv cache prune
uv python list --only-installed               # still a single version
```

Per project, upgrade dependencies deliberately with `uv lock --upgrade` and commit the lock change on its own, so a dependency bump is never mixed into a feature change.

::: callout
**Do not** install Python from python.org, the Microsoft Store, winget, or an IDE prompt again. If a tool insists on a Python it can find, point it at `uv python find 3.14`.
:::

::::

<!-- ======================= DEFINITION OF DONE ===================== -->

## Definition of done {#done}

::: legend
[verified 2026-09-10]{.l-done} [open]{.l-open}
:::

::: check

- [x] uv reports 0.12.x or later and its catalog lists CPython 3.14.7.
- [x] `uv python list --only-installed` shows a single version, 3.14.7, across its install directory, junction, and shims. No system interpreter appears.
- [x] `python`, `python3`, `uv`, `uvx` all resolve under `%USERPROFILE%\.local\bin`. `py` and `pip` do not resolve.
- [x] winget, the Windows uninstall registry, and Chocolatey list no Python; the Install Manager package is gone; the App Installer aliases for python and python3 are switched off.
- [ ] `%APPDATA%\Python` and `%LOCALAPPDATA%\Python` are gone. `C:\Program Files\Python314` is an empty shell and still needs an elevated `Remove-Item`.
- [x] PATH contains no Python entry other than `.local\bin`, in both user and machine scope.
- [x] `%APPDATA%\uv\uv.toml` sets `python-preference = "only-managed"`. `PYTHONNOUSERSITE` and `PIP_REQUIRE_VIRTUALENV` are set at user scope and their effect is confirmed.
- [x] `python -c "import anthropic"` fails and `python -m pip install requests` refuses.
- [x] speedtest-cli and ruff are uv tools. The thirteen project libraries and pytest get declared per repo as each is next touched; the twenty-four transitive packages need nothing.
- [ ] The monthly maintenance block is in your calendar.

:::

<!-- ======================== EXECUTION RECORD ====================== -->

## Execution record {#record}

What actually happened when the phases ran, and where reality differed from the plan above. Deviations are already folded into the phase text; this table is the audit trail.

::: tablewrap

| Phase | Date | Outcome | Deviation from plan |
|---|---|---|---|
| 1 | 2026-09-09 | uv 0.9.29 → 0.12.11; catalog now lists 3.14.7. | None. |
| 2 | 2026-09-09 | Managed 3.14.7 installed with python, python3, python3.14 shims and the 3.14 junction. | Trend Micro denied the PE-resource stamp on one shim (harmless, uv issue #20955). The managed 3.14.2 had already been retired by uv 0.12, so its uninstall found nothing. |
| 3 | 2026-09-09 | User-site directory deleted; 40-line manifest saved. | None. |
| 4 | 2026-09-10 | Install Manager runtime, Install Manager, python.org 3.14.7, legacy launcher, and Chocolatey packages all removed. | Chocolatey was added to scope after a dead shim surfaced. The two winget uninstalls left their targets in place on the first pass and succeeded on the second. The App Installer python aliases shadowed the shim until switched off in Settings. |
| 5 | 2026-09-10 | Two user PATH entries and the empty %APPDATA% and %LOCALAPPDATA% Python folders removed. | Machine PATH was already clean. The empty C:\\Program Files\\Python314 shell remains pending an elevated delete. |
| 6 | 2026-09-10 | uv.toml written; both environment variables set; user-site disabled and pip refusal confirmed. | None. |
| 7 | 2026-09-10 | All checks pass, including an ephemeral `uv run --with rich`. | uv 0.12 lists shims and junction alongside the install directory; the expected output was reworded to match. |
| 8, 10, 11 | — | Conventions, editor, and cadence. No machine change. | Calendar entry for the monthly block still to be created. |
| 9 | 2026-09-10 | speedtest-cli 2.1.3 and ruff 0.16.6 installed as uv tools. | Project and dev dependencies deferred to each repo by design. |

:::

<footer>

Produced with Claude Code (CLI, Windows) using Claude Fable 5.1, on 2026-09-09 and 2026-09-10, from a live inventory of this laptop and the uv documentation at docs.astral.sh. Commands were verified against uv 0.9.29 and 0.12.11 help output, the installed CPython 3.14 binaries, winget, Chocolatey 2.7.0, and the Python Install Manager 26.3.

</footer>

::::

:::::

::::::

---

# Appendix A — Visual specification (exact)

Everything in this appendix is copied unchanged from the published artifact. Apply it to the HTML produced from the content above.

## A.1 Page shell

- `<title>`: **uv-Only Python Playbook**
- `<meta charset="utf-8">`, `<meta name="viewport" content="width=device-width,initial-scale=1">`
- Fonts (Google Fonts):

```html
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans+Condensed:wght@500;600&family=IBM+Plex+Sans:ital,wght@0,400;0,500;0,600;1,400&family=IBM+Plex+Mono:wght@400;500&display=swap">
```

## A.2 Design tokens

| Token | Light | Dark |
|---|---|---|
| `--bg` | `#F6F6F3` | `#14171B` |
| `--surface` | `#FFFFFF` | `#1B1F24` |
| `--surface-2` | `#EEEFEA` | `#222730` |
| `--line` | `#D8DAD3` | `#2E343C` |
| `--line-strong` | `#B9BCB3` | `#434B55` |
| `--ink` | `#1B1F24` | `#E8EAE6` |
| `--ink-2` | `#4A5058` | `#B8BDC4` |
| `--muted` | `#6F767F` | `#8A9199` |
| `--accent` | `#0B6E69` | `#4FB8AF` |
| `--accent-ink` | `#0B6E69` | `#7CD0C8` |
| `--accent-soft` | `#DCEFEC` | `#183531` |
| `--warn` / `--warn-soft` | `#9A5B00` / `#F7ECD6` | `#E0A64A` / `#3A2D14` |
| `--bad` / `--bad-soft` | `#A33A2C` / `#F6E1DD` | `#E07B6C` / `#3E1F1A` |
| `--good` / `--good-soft` | `#2E6B3A` / `#DFEEDF` | `#7CC48A` / `#1B3322` |
| `--code-bg` | `#1E2329` | `#0F1216` |
| `--code-ink` | `#E6E8E3` | `#DDE1DC` |
| `--code-cmt` | `#8A939C` | `#7F8891` |
| `--code-str` | `#9ED3C6` | `#9ED3C6` |

Typography: body IBM Plex Sans 16px/1.6; headings IBM Plex Sans Condensed 600 (h1 2.5rem, h2 1.5rem, h3 1.1rem); code/labels IBM Plex Mono. Layout: 1120px max width, 220px sticky TOC column + fluid main, 48px gap, collapses to one column below 860px.

## A.3 Full stylesheet

```css
:root{
  --bg:#F6F6F3; --surface:#FFFFFF; --surface-2:#EEEFEA; --line:#D8DAD3; --line-strong:#B9BCB3;
  --ink:#1B1F24; --ink-2:#4A5058; --muted:#6F767F;
  --accent:#0B6E69; --accent-ink:#0B6E69; --accent-soft:#DCEFEC;
  --warn:#9A5B00; --warn-soft:#F7ECD6; --bad:#A33A2C; --bad-soft:#F6E1DD; --good:#2E6B3A; --good-soft:#DFEEDF;
  --code-bg:#1E2329; --code-ink:#E6E8E3; --code-cmt:#8A939C; --code-str:#9ED3C6;
  --shadow:0 1px 2px rgba(20,24,28,.06), 0 8px 24px -16px rgba(20,24,28,.25);
}
@media (prefers-color-scheme: dark){
  :root:not([data-theme="light"]){
    --bg:#14171B; --surface:#1B1F24; --surface-2:#222730; --line:#2E343C; --line-strong:#434B55;
    --ink:#E8EAE6; --ink-2:#B8BDC4; --muted:#8A9199;
    --accent:#4FB8AF; --accent-ink:#7CD0C8; --accent-soft:#183531;
    --warn:#E0A64A; --warn-soft:#3A2D14; --bad:#E07B6C; --bad-soft:#3E1F1A; --good:#7CC48A; --good-soft:#1B3322;
    --code-bg:#0F1216; --code-ink:#DDE1DC; --code-cmt:#7F8891; --code-str:#9ED3C6;
    --shadow:0 1px 2px rgba(0,0,0,.4), 0 8px 24px -16px rgba(0,0,0,.6);
  }
}
:root[data-theme="dark"]{
  --bg:#14171B; --surface:#1B1F24; --surface-2:#222730; --line:#2E343C; --line-strong:#434B55;
  --ink:#E8EAE6; --ink-2:#B8BDC4; --muted:#8A9199;
  --accent:#4FB8AF; --accent-ink:#7CD0C8; --accent-soft:#183531;
  --warn:#E0A64A; --warn-soft:#3A2D14; --bad:#E07B6C; --bad-soft:#3E1F1A; --good:#7CC48A; --good-soft:#1B3322;
  --code-bg:#0F1216; --code-ink:#DDE1DC; --code-cmt:#7F8891; --code-str:#9ED3C6;
  --shadow:0 1px 2px rgba(0,0,0,.4), 0 8px 24px -16px rgba(0,0,0,.6);
}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--ink);font:16px/1.6 "IBM Plex Sans",system-ui,Segoe UI,Arial,sans-serif;-webkit-font-smoothing:antialiased}
a{color:var(--accent-ink)}
.wrap{max-width:1120px;margin:0 auto;padding:40px 24px 64px}
.page{display:grid;grid-template-columns:220px minmax(0,1fr);gap:48px;align-items:start}
@media (max-width:860px){.page{grid-template-columns:1fr}.toc{position:static}}
h1,h2,h3{font-family:"IBM Plex Sans Condensed","IBM Plex Sans",system-ui,sans-serif;text-wrap:balance;margin:0;line-height:1.15}
h1{font-size:2.5rem;font-weight:600;letter-spacing:-.01em}
h2{font-size:1.5rem;font-weight:600;margin-top:56px;padding-top:20px;border-top:1px solid var(--line)}
h3{font-size:1.1rem;font-weight:600;margin-top:28px}
p{margin:12px 0;max-width:68ch}
ul,ol{max-width:70ch;padding-left:22px}
li{margin:6px 0}
.eyebrow{font-family:"IBM Plex Mono",Consolas,monospace;font-size:.75rem;letter-spacing:.08em;text-transform:uppercase;color:var(--muted)}
.meta{display:flex;flex-wrap:wrap;gap:8px 22px;margin:14px 0 0;font-size:.85rem;color:var(--ink-2)}
.meta b{color:var(--ink);font-weight:500}
.lede{font-size:1.1rem;color:var(--ink-2);max-width:62ch;margin-top:18px}
.toc{position:sticky;top:24px;font-size:.9rem}
.toc ol{list-style:none;padding:0;margin:8px 0 0;border-left:2px solid var(--line)}
.toc li{margin:0}
.toc a{display:block;padding:5px 12px;color:var(--ink-2);text-decoration:none;border-left:2px solid transparent;margin-left:-2px}
.toc a:hover,.toc a:focus-visible{color:var(--accent-ink);border-left-color:var(--accent)}
.toc .sub{font-size:.8rem;color:var(--muted);padding-left:12px;margin-top:14px}
pre{margin:14px 0;background:var(--code-bg);color:var(--code-ink);border-radius:6px;padding:14px 16px;overflow-x:auto;font:.86rem/1.55 "IBM Plex Mono",Consolas,monospace}
pre .c{color:var(--code-cmt)}
pre .s{color:var(--code-str)}
code{font-family:"IBM Plex Mono",Consolas,monospace;font-size:.9em;background:var(--surface-2);padding:1px 5px;border-radius:4px}
pre code{background:none;padding:0;font-size:inherit}
.shell{display:inline-block;font:.7rem/1 "IBM Plex Mono",monospace;letter-spacing:.06em;text-transform:uppercase;color:var(--muted);margin-top:14px}
.shell + pre{margin-top:6px}
.state{display:grid;grid-template-columns:1fr 1fr;gap:16px;margin-top:28px}
@media (max-width:700px){.state{grid-template-columns:1fr}}
.panel{background:var(--surface);border:1px solid var(--line);border-radius:8px;padding:18px 20px}
.panel h3{margin-top:0;font-size:1rem}
.panel ul{padding-left:18px;margin:10px 0 0;font-size:.93rem}
.panel.now{border-top:3px solid var(--bad)}
.panel.target{border-top:3px solid var(--accent)}
.callout{border-left:3px solid var(--warn);background:var(--warn-soft);padding:12px 16px;border-radius:0 6px 6px 0;margin:18px 0;max-width:72ch}
.callout.info{border-left-color:var(--accent);background:var(--accent-soft)}
.callout.bad{border-left-color:var(--bad);background:var(--bad-soft)}
.callout p{margin:6px 0}
.phase{margin-top:44px;padding-top:24px;border-top:1px solid var(--line)}
.phase-head{display:flex;align-items:baseline;gap:14px}
.phase-num{font-family:"IBM Plex Mono",monospace;font-size:.8rem;color:var(--accent-ink);letter-spacing:.06em;min-width:3ch}
.phase h2{border:0;margin:0;padding:0}
.why{color:var(--ink-2);font-size:.95rem}
.verify{margin-top:16px;border:1px solid var(--line);border-radius:6px;background:var(--surface);padding:12px 16px;max-width:72ch}
.verify .eyebrow{color:var(--good)}
.verify p{margin:6px 0;font-size:.93rem}
table{border-collapse:collapse;width:100%;font-size:.9rem;margin:14px 0}
.tablewrap{overflow-x:auto;border:1px solid var(--line);border-radius:6px;background:var(--surface)}
th,td{text-align:left;padding:9px 12px;border-bottom:1px solid var(--line);vertical-align:top}
th{font-weight:500;color:var(--ink-2);font-size:.8rem;letter-spacing:.04em;text-transform:uppercase;background:var(--surface-2)}
tr:last-child td{border-bottom:0}
td code{white-space:nowrap}
.pill{display:inline-block;font:.72rem/1 "IBM Plex Mono",monospace;padding:4px 8px;border-radius:999px;letter-spacing:.03em}
.pill.tool{background:var(--accent-soft);color:var(--accent-ink)}
.pill.proj{background:var(--good-soft);color:var(--good)}
.pill.dev{background:var(--warn-soft);color:var(--warn)}
.pill.skip{background:var(--surface-2);color:var(--muted)}
.check{list-style:none;padding:0;max-width:72ch}
.check li{display:grid;grid-template-columns:22px 1fr;gap:10px;padding:8px 0;border-bottom:1px dashed var(--line)}
.check li::before{content:"";width:16px;height:16px;border:1.5px solid var(--line-strong);border-radius:3px;margin-top:5px;font:600 11px/13px "IBM Plex Mono",monospace;text-align:center;color:var(--surface)}
.check li.done::before{content:"\2713";background:var(--accent);border-color:var(--accent)}
.check li.open::before{border-color:var(--warn);border-style:dashed}
.legend{display:flex;gap:18px;font-size:.8rem;color:var(--muted);margin:10px 0 0}
.legend span::before{content:"";display:inline-block;width:10px;height:10px;border-radius:2px;margin-right:6px;vertical-align:-1px;border:1.5px solid var(--line-strong)}
.legend .l-done::before{background:var(--accent);border-color:var(--accent)}
.legend .l-open::before{border-color:var(--warn);border-style:dashed}
.decision{display:grid;grid-template-columns:1fr 1fr;gap:16px}
@media (max-width:700px){.decision{grid-template-columns:1fr}}
.decision .panel h3{display:flex;justify-content:space-between;gap:8px}
footer{margin-top:64px;padding-top:18px;border-top:1px solid var(--line);font-size:.82rem;color:var(--muted);max-width:none}
footer p{max-width:none}
kbd{font-family:"IBM Plex Mono",monospace;font-size:.85em;border:1px solid var(--line-strong);border-bottom-width:2px;border-radius:4px;padding:0 5px;background:var(--surface)}
:focus-visible{outline:2px solid var(--accent);outline-offset:2px}
```

---

# Appendix B — Annotation → HTML mapping (exact markup)

| Markdown annotation | Original HTML |
|---|---|
| `:::::: {.wrap}` | `<div class="wrap">` |
| `::::: header` | `<header>` |
| `[text]{.eyebrow}` | `<div class="eyebrow">text</div>` |
| `# Title {style="margin-top:10px"}` | `<h1 style="margin-top:10px">` |
| `::: lede` | `<p class="lede">` |
| `::: meta` + `[Label **Value**]{}` | `<div class="meta"><span>Label <b>Value</b></span>…</div>` |
| `::::: {.page style="margin-top:40px"}` | `<div class="page" style="margin-top:40px">` |
| `:::: {.toc aria-label="Contents"}` + numbered list | `<nav class="toc" aria-label="Contents"><div class="eyebrow">Contents</div><ol><li><a href="#…">…</a></li>…</ol><div class="sub">…</div></nav>` (numbers hidden by `list-style:none`) |
| `:::: main` | `<main>` |
| `::: {#state}` | `<section id="state">` |
| `:::: state` / `::: {.panel .now}` / `::: {.panel .target}` | `<div class="state">` / `<div class="panel now">` / `<div class="panel target">` |
| `## Heading {#decisions}` (also `#done`, `#record`) | `<section id="decisions"><h2>Heading</h2>…</section>` |
| `:::: decision` / `::: panel` | `<div class="decision">` / `<div class="panel">` |
| `### Title [taken]{.pill .tool}` | `<h3>Title <span class="pill tool">taken</span></h3>` |
| `[**Alternative:** …]{.why .p}` | `<p class="why"><b>Alternative:</b> …</p>` |
| `:::: {#p1 .phase}` | `<section class="phase" id="p1">` |
| `::: phase-head` + `[01]{.phase-num}` + `## Title` | `<div class="phase-head"><span class="phase-num">01</span><h2>Title</h2></div>` |
| `::: why` (inside a phase) | `<p class="why">…</p>` |
| `[PowerShell 7]{.shell}` directly before a code fence | `<span class="shell">PowerShell 7</span>` followed by `<pre>` |
| ` ```powershell ` / ` ```python ` fences | `<pre><code>…</code></pre>`; every `#` comment wrapped in `<span class="c">…</span>`; the whole PEP 723 block is one `.c` span; no other highlighting |
| `::: callout` | `<div class="callout"><p>…</p></div>` (warn variant; `.info` and `.bad` exist in CSS but are unused) |
| `::: verify` + `[Verify]{.eyebrow}` | `<div class="verify"><div class="eyebrow">Verify</div><p>…</p></div>` |
| `::: tablewrap` + pipe table | `<div class="tablewrap"><table><thead>…</thead><tbody>…</tbody></table></div>` |
| `[tool]{.pill .tool}` · `[project dep]{.pill .proj}` · `[dev dep]{.pill .dev}` · `[transitive]{.pill .skip}` | `<span class="pill tool|proj|dev|skip">…</span>` |
| `::: legend` + `[…]{.l-done}` `[…]{.l-open}` | `<div class="legend"><span class="l-done">…</span><span class="l-open">…</span></div>` |
| `::: check` + `- [x]` / `- [ ]` | `<ul class="check"><li class="done">…</li><li class="open">…</li></ul>` (no `<input>` checkboxes; the box is drawn by `li::before`) |
| `<kbd>` | `<kbd>` (unchanged) |
| `<footer>` | `<footer><p>…</p></footer>`, last child of `<main>` |
