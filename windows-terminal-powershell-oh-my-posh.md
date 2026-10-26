---
title: Windows Terminal, PowerShell 7 and Oh My Posh on Windows 11
eyebrow: Engineering Standards · Developer Runbook
tagline: >-
  A full inventory of one working terminal setup, and the steps to rebuild it
  **exactly** on a second Windows 11 machine. It covers **Windows Terminal** (profiles,
  key bindings, shell-integration marks), **PowerShell 7** (a startup-cached profile,
  PSReadLine, fzf, zoxide, eza), the **Oh My Posh** prompt and its Nerd Fonts, and the
  hand-off that sends interactive Windows PowerShell 5.1 sessions to PowerShell 7.
  Every step runs in **PowerShell**.
budget:
  value: "~25"
  unit: "minutes, once, per laptop"
version: "1.0"
date: 2026-10-26
lang: en
---

<!--
AUTHORING CONVENTIONS: same as sign-your-commits.md (Pandoc Markdown, pandoc ≥ 3.0).
Build:  pandoc windows-terminal-powershell-oh-my-posh.md -f markdown -t html5 -s --section-divs --template runbook.html -o index.html
Steps marked shell="powershell" run in PowerShell 7, except Step 1, which also works in the built-in Windows PowerShell 5.1.
The four files embedded in Steps 6–9 were copied byte for byte from the source machine. Re-copy them, don't retype them, when the source changes.
-->

::: {.req label="Source machine"}
**Inventoried 2026-10-26** on Windows 11 Pro 10.0.26200, Windows Terminal **1.25.2733.0**, PowerShell **7.6.9**, Oh My Posh **31.9.1**. Every value below was read from the live machine: `winget list`, `Get-AppxPackage`, `Get-InstalledPSResource`, both PowerShell profiles, the Oh My Posh theme file, Windows Terminal's `settings.json` and `state.json`, the per-user and machine font registry, `HKCU\Console\%%Startup`, and `$PSHOME\powershell.config.json`. The user folder is written as `<you>`.
:::

## Inventory {#inventory}

### Toolchain {.unnumbered}

| Tool | Version | winget id | Role |
|---|---|---|---|
| Windows Terminal | 1.25.2733.0 | `Microsoft.WindowsTerminal` | Terminal host and the Windows **default terminal application** |
| PowerShell 7 | 7.6.9.0 | `Microsoft.PowerShell` | The shell. Windows Terminal's default profile |
| Windows PowerShell | 5.1 (OS component) | n/a | Kept for scripts. Interactive sessions hand off to PowerShell 7 (Step 8) |
| PSReadLine | 2.4.5 | n/a (ships with PowerShell 7.6.9) | Line editing, history predictions, the fzf key handlers |
| Oh My Posh | 31.9.1 | `JanDeDobbeleer.OhMyPosh` | Prompt. Installs as the MSIX package `ohmyposh.cli`, exe alias in `%LOCALAPPDATA%\Microsoft\WindowsApps` |
| fzf | 0.74.5 | `junegunn.fzf` | Fuzzy picker behind `Ctrl+R`, `Ctrl+T`, `Alt+C` |
| fd | 10.5.0 | `sharkdp.fd` | File and directory source for fzf (respects `.gitignore`, skips `.git`) |
| zoxide | 0.10.0 | `ajeetdsouza.zoxide` | `z` / `zi` frecency jumps |
| eza | 0.23.5 | `eza-community.eza` | `ls`, `ll`, `la`, `lt` |
| bat | 0.26.1 | `sharkdp.bat` | *Optional.* The profile only sets `BAT_THEME` |
| Claude Code | 2.1.310 | n/a (native installer, `~\.local\bin`) | *Optional.* The *Claude Code* Terminal profile and the `cc` / `ccc` / `ccr` shortcuts |
| VS Code | n/a | n/a | `EDITOR` / `GIT_EDITOR` (set by the profile) |

### Fonts {.unnumbered}

Windows Terminal asks for `Cascadia Code NF, MesloLGM Nerd Font, Consolas`, in that order, and uses the first one installed.

| Font | Where it comes from | Role |
|---|---|---|
| Cascadia Code NF | Ships with Windows 11 (`C:\Windows\Fonts\CascadiaCodeNF.ttf`, dated with the OS image) | The face actually used. Contains the Nerd Font glyphs the prompt draws |
| MesloLGM Nerd Font | Per-user install of the full Meslo LG Nerd Font family (S/M/L, DZ, Mono, Propo) in `%LOCALAPPDATA%\Microsoft\Windows\Fonts` | Fallback for a Windows build without Cascadia Code NF. Installed by `oh-my-posh font install Meslo` |
| Consolas | OS | Last resort. Draws no prompt glyphs |

### PowerShell modules {.unnumbered}

All four live in `OneDrive\Documents\PowerShell\Modules` (installed from PSGallery with PSResourceGet). **None is imported at startup.**

| Module | Version | How it is used |
|---|---|---|
| PSFzf | 2.7.12 | Not imported. The profile's own fzf key handlers replace it (it cost ~440 ms). `Import-Module PSFzf` still works on demand |
| Terminal-Icons | 0.11.0 | Opt-in: `Enable-TerminalIcons` (~400 ms) |
| CompletionPredictor | 0.1.1 | Opt-in: `Enable-CompletionPredictor` switches predictions to `HistoryAndPlugin` |
| posh-git | 1.1.0 | Not referenced anywhere (see *Audit findings*) |

### Files that make up the setup {.unnumbered}

| File | What it holds | Written in |
|---|---|---|
| `~\.config\oh-my-posh\jandedobbeleer-custom.omp.json` | The prompt theme | Step 6 |
| `$PROFILE` = `OneDrive\Documents\PowerShell\Microsoft.PowerShell_profile.ps1` | PowerShell 7 profile | Step 7 |
| `OneDrive\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1` | Windows PowerShell 5.1 hand-off | Step 8 |
| `%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json` | Windows Terminal settings | Step 9 |
| `%LOCALAPPDATA%\pwsh-cache\oh-my-posh.ps1`, `zoxide.ps1` | Generated init scripts. Never edit or copy them | The profile, on first start |
| `HKCU\Console\%%Startup` | `DelegationConsole` / `DelegationTerminal` = Windows Terminal | Step 2 |

**Not configured (deliberately absent, so leave them absent for an exact copy):** no `AllUsers*` profiles and no `CurrentUserAllHosts` `profile.ps1`, no persisted `POSH_*`, `FZF_*`, `EDITOR` or `BAT_THEME` environment variables (the profile sets them per session), no Windows Terminal fragments, no custom colour schemes or themes (`Dark+` is built in), no winget pins. The PowerShell 7 execution policy is the default `RemoteSigned`, from `$PSHOME\powershell.config.json`.

## Rebuild {#setup}

::: {.note .ink}
**OneDrive shortcut.** On the source machine *Documents* is backed up to OneDrive, so both profiles and the `Modules` folder live there. If OneDrive *Documents* backup is on for the new machine, Steps 5, 7 and 8 arrive on their own. Check with `Test-Path $PROFILE` and `Get-InstalledPSResource`, and skip whatever is already there. The theme (Step 6) and Terminal settings (Step 9) are **not** in OneDrive.
:::

### Install PowerShell 7 and update Windows Terminal {.step minutes="3 min" shell="powershell"}

Windows 11 already has Windows Terminal and Windows PowerShell 5.1. Run this from either one:

```powershell
winget install --id Microsoft.PowerShell       --exact --version 7.6.9.0     --source winget
winget install --id Microsoft.WindowsTerminal  --exact --version 1.25.2733.0 --source winget
```

If Windows Terminal is already newer than the pinned build, winget says so and changes nothing. If it does update, it may close every open Terminal window, including the one running the command, which is why PowerShell comes first. Close every terminal window, then open **Windows Terminal** and start a **PowerShell 7** tab (`pwsh`) for every step from here on.

::: note
**Pinned versions.** Each `--version` gives an exact copy. If winget answers *No version found matching*, that build has left the catalogue. Drop `--version` to take the current release, and record the difference.
:::

### Make Windows Terminal the default terminal {.step minutes="1 min" shell="powershell"}

The source machine routes every console window, including `powershell.exe` and `cmd.exe` started from Explorer or Run, into Windows Terminal:

```powershell
$k = 'HKCU:\Console\%%Startup'
if (-not (Test-Path $k)) { New-Item -Path $k | Out-Null }
Set-ItemProperty -Path $k -Name DelegationConsole  -Value '{2EACA947-7F5F-4CFA-BA87-8F7FBEEFBE69}'
Set-ItemProperty -Path $k -Name DelegationTerminal -Value '{E12CFF52-A866-4C77-9A90-F570A7AA2C6B}'
```

The two GUIDs are the fixed IDs of the Store build of Windows Terminal. The same setting is in the UI: **Terminal → Settings → Startup → Default terminal application → Windows Terminal**.

### Install Oh My Posh and the Meslo Nerd Font {.step minutes="3 min" shell="powershell"}

```powershell
winget install --id JanDeDobbeleer.OhMyPosh --exact --version 31.9.1 --source winget
```

Open a **new** tab so the `oh-my-posh` alias is on `PATH`, then:

```powershell
oh-my-posh version              # 31.9.1
oh-my-posh font install Meslo
```

Run it from a normal (not elevated) shell. The fonts then go to the per-user font folder, as on the source machine. Cascadia Code NF, the face Terminal actually uses, already ships with Windows 11. Check that it does on this machine:

```powershell
Test-Path C:\Windows\Fonts\CascadiaCodeNF.ttf   # True
```

If it prints `False`, Windows Terminal falls back to MesloLGM Nerd Font, and the prompt still renders correctly.

### Install the CLI companions {.step minutes="2 min" shell="powershell"}

```powershell
winget install --id junegunn.fzf        --exact --version 0.74.5 --source winget
winget install --id sharkdp.fd          --exact --version 10.5.0 --source winget
winget install --id ajeetdsouza.zoxide  --exact --version 0.10.0 --source winget
winget install --id eza-community.eza   --exact --version 0.23.5 --source winget

# optional: the profile only sets its theme
winget install --id sharkdp.bat         --exact --version 0.26.1 --source winget
```

winget installs these as *portable* packages and adds each package folder to your **user** `PATH`. Open a new tab before you continue, then check:

```powershell
fzf --version; fd --version; zoxide --version; eza --version
```

Every block in the profile is wrapped in `if (Get-Command …)`, so a missing tool only removes its feature. It never breaks startup.

### Install the PowerShell modules {.step minutes="2 min" shell="powershell"}

Skip this step if OneDrive already brought the `Modules` folder.

```powershell
Install-PSResource -Name PSFzf               -Version 2.7.12 -Repository PSGallery -TrustRepository
Install-PSResource -Name Terminal-Icons      -Version 0.11.0 -Repository PSGallery -TrustRepository
Install-PSResource -Name CompletionPredictor -Version 0.1.1  -Repository PSGallery -TrustRepository
Install-PSResource -Name posh-git            -Version 1.1.0  -Repository PSGallery -TrustRepository
```

All four are listed for an exact copy, but the profile imports none of them at startup. posh-git is unused, and you can leave it out (see *Audit findings*).

### Write the Oh My Posh theme {.step minutes="1 min" shell="powershell"}

The theme is the upstream `jandedobbeleer` theme with two additions: `shell_integration` (Terminal prompt marks) and `pwd: osc99` (new panes open in the current folder). *The prompt, explained* describes every segment.

```powershell
$dir = Join-Path $HOME '.config\oh-my-posh'
New-Item -ItemType Directory -Path $dir -Force | Out-Null

@'
{
  "$schema": "https://raw.githubusercontent.com/JanDeDobbeleer/oh-my-posh/main/themes/schema.json",
  "upgrade": {
    "source": "cdn",
    "interval": "168h",
    "auto": false,
    "notice": false
  },
  "console_title_template": "{{ .Shell }} in {{ .Folder }}",
  "shell_integration": true,
  "pwd": "osc99",
  "blocks": [
    {
      "type": "prompt",
      "alignment": "left",
      "segments": [
        {
          "leading_diamond": "\ue0b6",
          "trailing_diamond": "\ue0b0",
          "template": " {{ .UserName }} ",
          "foreground": "#ffffff",
          "background": "#c386f1",
          "type": "session",
          "style": "diamond"
        },
        {
          "options": {
            "folder_separator_icon": " \ue0b1 ",
            "home_icon": "~",
            "style": "folder"
          },
          "template": " \uea83  {{ .Path }} ",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",
          "background": "#ff479c",
          "type": "path",
          "style": "powerline"
        },
        {
          "options": {
            "branch_template": "{{ trunc 25 .Branch }}",
            "fetch_status": true,
            "fetch_upstream_icon": true
          },
          "leading_diamond": "\ue0b6",
          "trailing_diamond": "\ue0b4",
          "template": " {{ .UpstreamIcon }}{{ .HEAD }}{{if .BranchStatus }} {{ .BranchStatus }}{{ end }}{{ if .Working.Changed }} \uf044 {{ .Working.String }}{{ end }}{{ if and (.Working.Changed) (.Staging.Changed) }} |{{ end }}{{ if .Staging.Changed }} \uf046 {{ .Staging.String }}{{ end }}{{ if gt .StashCount 0 }} \ueb4b {{ .StashCount }}{{ end }} ",
          "foreground": "#193549",
          "powerline_symbol": "\ue0b0",
          "background": "#fffb38",
          "type": "git",
          "style": "powerline",
          "background_templates": [
            "{{ if or (.Working.Changed) (.Staging.Changed) }}#FF9248{{ end }}",
            "{{ if and (gt .Ahead 0) (gt .Behind 0) }}#ff4500{{ end }}",
            "{{ if gt .Ahead 0 }}#B388FF{{ end }}",
            "{{ if gt .Behind 0 }}#B388FF{{ end }}"
          ]
        },
        {
          "options": {
            "fetch_version": true
          },
          "template": " \ue718 {{ if .PackageManagerIcon }}{{ .PackageManagerIcon }} {{ end }}{{ .Full }} ",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",
          "background": "#6CA35E",
          "type": "node",
          "style": "powerline"
        },
        {
          "options": {
            "fetch_version": true
          },
          "template": " \ue626 {{ if .Error }}{{ .Error }}{{ else }}{{ .Full }}{{ end }} ",
          "foreground": "#111111",
          "powerline_symbol": "\ue0b0",
          "background": "#8ED1F7",
          "type": "go",
          "style": "powerline"
        },
        {
          "options": {
            "fetch_version": true
          },
          "template": " \ue624 {{ if .Error }}{{ .Error }}{{ else }}{{ .Full }}{{ end }} ",
          "foreground": "#111111",
          "powerline_symbol": "\ue0b0",
          "background": "#4063D8",
          "type": "julia",
          "style": "powerline"
        },
        {
          "options": {
            "display_mode": "files",
            "fetch_virtual_env": false
          },
          "template": " \ue235 {{ if .Error }}{{ .Error }}{{ else }}{{ .Full }}{{ end }} ",
          "foreground": "#111111",
          "powerline_symbol": "\ue0b0",
          "background": "#FFDE57",
          "type": "python",
          "style": "powerline"
        },
        {
          "options": {
            "display_mode": "files",
            "fetch_version": true
          },
          "template": " \ue791 {{ if .Error }}{{ .Error }}{{ else }}{{ .Full }}{{ end }} ",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",
          "background": "#AE1401",
          "type": "ruby",
          "style": "powerline"
        },
        {
          "options": {
            "display_mode": "files",
            "fetch_version": false
          },
          "template": " \uf0e7{{ if .Error }}{{ .Error }}{{ else }}{{ .Full }}{{ end }} ",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",
          "background": "#FEAC19",
          "type": "azfunc",
          "style": "powerline"
        },
        {
          "options": {
            "display_default": false
          },
          "template": " \ue7ad {{ .Profile }}{{ if .Region }}@{{ .Region }}{{ end }} ",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",
          "type": "aws",
          "style": "powerline",
          "background_templates": [
            "{{if contains \"default\" .Profile}}#FFA400{{end}}",
            "{{if contains \"jan\" .Profile}}#f1184c{{end}}"
          ]
        },
        {
          "template": " \uf0ad ",
          "foreground": "#111111",
          "powerline_symbol": "\ue0b0",
          "background": "#ffff66",
          "type": "root",
          "style": "powerline"
        },
        {
          "options": {
            "always_enabled": true
          },
          "template": "<transparent>\ue0b0</> \ueba2 {{ .FormattedMs }}\u2800",
          "foreground": "#ffffff",
          "background": "#83769c",
          "type": "executiontime",
          "style": "plain"
        },
        {
          "options": {
            "always_enabled": true
          },
          "trailing_diamond": "\ue0b4",
          "template": "<parentBackground>\ue0b0</> \ue23a ",
          "foreground": "#ffffff",
          "background": "#00897b",
          "type": "status",
          "style": "diamond",
          "background_templates": [
            "{{ if gt .Code 0 }}#e91e63{{ end }}"
          ]
        }
      ]
    },
    {
      "type": "rprompt",
      "segments": [
        {
          "template": "<#0077c2,transparent>\ue0b6</> \uf489 {{ .Name }} <transparent,#0077c2>\ue0b2</>",
          "foreground": "#ffffff",
          "background": "#0077c2",
          "type": "shell",
          "style": "plain"
        },
        {
          "options": {
            "paused_icon": "\uf04c ",
            "playing_icon": "\uf04b "
          },
          "template": " \uf167 {{ .Icon }}{{ if ne .Status \"stopped\" }}{{ .Artist }} - {{ .Track }}{{ end }} ",
          "foreground": "#111111",
          "powerline_symbol": "\ue0b2",
          "background": "#1BD760",
          "type": "ytm",
          "style": "powerline",
          "invert_powerline": true
        },
        {
          "options": {
            "charged_icon": "\ue22f ",
            "charging_icon": "\ue234 ",
            "discharging_icon": "\ue231 "
          },
          "template": " {{ if not .Error }}{{ .Icon }}{{ .Percentage }}{{ end }}{{ .Error }}\uf295 ",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b2",
          "background": "#f36943",
          "type": "battery",
          "style": "powerline",
          "background_templates": [
            "{{if eq \"Charging\" .State.String}}#40c4ff{{end}}",
            "{{if eq \"Discharging\" .State.String}}#ff5722{{end}}",
            "{{if eq \"Full\" .State.String}}#4caf50{{end}}"
          ],
          "invert_powerline": true
        },
        {
          "leading_diamond": "\ue0b2",
          "trailing_diamond": "\ue0b4",
          "template": " {{ .CurrentDate | date .Format }} ",
          "foreground": "#111111",
          "background": "#2e9599",
          "type": "time",
          "style": "diamond",
          "invert_powerline": true
        }
      ]
    }
  ],
  "version": 4,
  "final_space": true
}
'@ | Set-Content -Path (Join-Path $dir 'jandedobbeleer-custom.omp.json') -Encoding utf8
```

::: note
**Keep the single-quoted here-string (`@'` … `'@`).** In a double-quoted one, PowerShell would expand `$schema`, and the closing `'@` must sit at the start of its line. The `\ue0b6`-style escapes are JSON escapes for Nerd Font glyphs. Leave them as they are.
:::

### Write the PowerShell 7 profile {.step minutes="2 min" shell="powershell"}

Skip this step if OneDrive already brought `$PROFILE`. Otherwise back up anything that is there, then write the profile:

```powershell
if (Test-Path $PROFILE) { Copy-Item $PROFILE "$PROFILE.bak" }
New-Item -ItemType Directory -Path (Split-Path $PROFILE) -Force | Out-Null

@'
#Requires -Version 7.0
# PowerShell 7 profile — coding / CLI navigation / Claude Code
#
# Startup budget matters here: every expensive step is either cached to disk or
# turned into an opt-in function. Measured costs on this machine (pwsh 7.6.5):
#   oh-my-posh init  ~890 ms   -> cached, see PROMPT
#   Terminal-Icons   ~400 ms   -> opt-in, Enable-TerminalIcons
#   PSFzf            ~440 ms   -> replaced by hand-rolled fzf key handlers
# Run  Measure-Command { pwsh -NoLogo -Command exit }  after editing this file.

$ShellCache = Join-Path $env:LOCALAPPDATA 'pwsh-cache'
if (-not (Test-Path -LiteralPath $ShellCache)) {
    New-Item -ItemType Directory -Path $ShellCache -Force | Out-Null
}

# Cache the output of a generator command and dot-source the cached script.
# Regenerated when the cache is missing, empty, or older than $MaxAgeDays.
function Use-CachedInit {
    param(
        [Parameter(Mandatory)][string]   $Name,
        [Parameter(Mandatory)][scriptblock] $Generator,
        [int] $MaxAgeDays = 14
    )
    $path = Join-Path $ShellCache "$Name.ps1"
    $item = Get-Item -LiteralPath $path -ErrorAction SilentlyContinue
    if (-not $item -or $item.Length -eq 0 -or
        $item.LastWriteTime -lt (Get-Date).AddDays(-$MaxAgeDays)) {
        try {
            $generated = & $Generator
            if ($generated) {
                Set-Content -LiteralPath $path -Value $generated -Encoding utf8
            }
        } catch {
            Write-Warning "Could not refresh $Name init cache: $($_.Exception.Message)"
        }
    }
    if (Test-Path -LiteralPath $path) { . $path }
}

# Force all cached init scripts to regenerate on the next shell.
function Reset-ShellCache {
    Get-ChildItem -LiteralPath $ShellCache -Filter '*.ps1' -ErrorAction SilentlyContinue |
        Remove-Item -Force
    Write-Host 'Shell init cache cleared — open a new shell to rebuild.' -ForegroundColor Yellow
}

# ── Prompt: oh-my-posh (jandedobbeleer, local copy) ──────────────────────────
# `--print` emits the whole init script. Without it oh-my-posh prints a stub that
# re-invokes the exe, so every shell paid two process spawns before the prompt.
# The local theme copy adds shell_integration (FTCS marks -> Windows Terminal
# prompt marks) and pwd:osc99 (new tabs/panes inherit the current directory).
# The config path is baked into the cached init: run Reset-ShellCache after
# editing the theme or moving it.
if (Get-Command oh-my-posh -CommandType Application -ErrorAction SilentlyContinue) {
    Use-CachedInit -Name 'oh-my-posh' -Generator {
        $config = Join-Path $HOME '.config\oh-my-posh\jandedobbeleer-custom.omp.json'
        if (-not (Test-Path -LiteralPath $config)) { $config = 'jandedobbeleer' }
        oh-my-posh init pwsh --config $config --print
    }
    # The cached script carries the session id from the shell that generated it;
    # give this shell its own so oh-my-posh's per-session state stays separate.
    $env:POSH_SESSION_ID = [guid]::NewGuid().ToString()
}

# ── PSReadLine ───────────────────────────────────────────────────────────────
Set-PSReadLineOption -EditMode Windows
# Predictive suggestions need a real VT-capable console; setting them when output
# is redirected (scripts, hooks, `pwsh -Command | ...`) is a hard error.
if ($Host.UI.SupportsVirtualTerminal -and -not [Console]::IsOutputRedirected) {
    Set-PSReadLineOption -PredictionSource History -PredictionViewStyle ListView
}
Set-PSReadLineOption -HistoryNoDuplicates -HistorySearchCursorMovesToEnd
Set-PSReadLineOption -MaximumHistoryCount 20000 -BellStyle None
Set-PSReadLineOption -Colors @{ InlinePrediction = "`e[38;5;240m" }

# Keep obvious secrets out of the on-disk history file.
Set-PSReadLineOption -AddToHistoryHandler {
    param([string] $Line)
    if ($Line -match '(?i)\b(password|passwd|secret|token|apikey|api[-_]key|credential|connectionstring)\b\s*[:=]') {
        return $false
    }
    $true
}

Set-PSReadLineKeyHandler -Key Tab            -Function MenuComplete
Set-PSReadLineKeyHandler -Key UpArrow        -Function HistorySearchBackward
Set-PSReadLineKeyHandler -Key DownArrow      -Function HistorySearchForward
Set-PSReadLineKeyHandler -Key F2             -Function SwitchPredictionView
Set-PSReadLineKeyHandler -Key 'Ctrl+w'       -Function BackwardDeleteWord
Set-PSReadLineKeyHandler -Key 'Alt+d'        -Function DeleteWord
Set-PSReadLineKeyHandler -Key 'Ctrl+LeftArrow'  -Function BackwardWord
Set-PSReadLineKeyHandler -Key 'Ctrl+RightArrow' -Function ForwardWord

# ── fzf: environment + key handlers ──────────────────────────────────────────
# Hand-rolled instead of Import-Module PSFzf, which costs ~440 ms at startup.
# `Import-Module PSFzf` still works on demand if you want its extra commands.
if (Get-Command fzf -CommandType Application -ErrorAction SilentlyContinue) {
    $env:FZF_DEFAULT_OPTS = '--height=45% --layout=reverse --border=rounded --info=inline --cycle --bind=ctrl-/:toggle-preview'
    if (Get-Command fd -CommandType Application -ErrorAction SilentlyContinue) {
        $env:FZF_DEFAULT_COMMAND = 'fd --type f --hidden --follow --exclude .git'
    }

    # Ctrl+R — fuzzy search of the persisted command history, newest first.
    Set-PSReadLineKeyHandler -Key 'Ctrl+r' `
        -BriefDescription 'FuzzyHistory' `
        -LongDescription 'Search command history with fzf' `
        -ScriptBlock {
            $path = (Get-PSReadLineOption).HistorySavePath
            if (-not (Test-Path -LiteralPath $path)) { return }
            $seen  = [System.Collections.Generic.HashSet[string]]::new()
            $items = [System.Collections.Generic.List[string]]::new()
            foreach ($line in [System.IO.File]::ReadLines($path)) {
                if ($line.Trim() -and $seen.Add($line)) { $items.Add($line) }
            }
            $items.Reverse()
            $choice = $items | fzf --no-sort --exact --prompt 'history> '
            if ($choice) {
                [Microsoft.PowerShell.PSConsoleReadLine]::RevertLine()
                [Microsoft.PowerShell.PSConsoleReadLine]::Insert($choice)
            }
        }

    # Ctrl+T — pick one or more files and insert their paths at the cursor.
    Set-PSReadLineKeyHandler -Key 'Ctrl+t' `
        -BriefDescription 'FuzzyFile' `
        -LongDescription 'Insert file paths chosen with fzf' `
        -ScriptBlock {
            $picked = if (Get-Command fd -CommandType Application -ErrorAction SilentlyContinue) {
                fd --type f --hidden --follow --exclude .git --strip-cwd-prefix | fzf --multi --prompt 'files> '
            } else {
                Get-ChildItem -Recurse -File -Name | fzf --multi --prompt 'files> '
            }
            if ($picked) {
                $quoted = $picked | ForEach-Object {
                    if ($_ -match '\s') { "'" + $_.Replace("'", "''") + "'" } else { $_ }
                }
                [Microsoft.PowerShell.PSConsoleReadLine]::Insert(($quoted -join ' '))
            }
        }

    # Alt+C — pick a subdirectory and cd into it.
    Set-PSReadLineKeyHandler -Key 'Alt+c' `
        -BriefDescription 'FuzzyChangeDirectory' `
        -LongDescription 'cd into a directory chosen with fzf' `
        -ScriptBlock {
            $dir = if (Get-Command fd -CommandType Application -ErrorAction SilentlyContinue) {
                fd --type d --hidden --follow --exclude .git --strip-cwd-prefix | fzf --prompt 'cd> '
            } else {
                Get-ChildItem -Recurse -Directory -Name | fzf --prompt 'cd> '
            }
            if ($dir) {
                Set-Location -LiteralPath $dir
                [Microsoft.PowerShell.PSConsoleReadLine]::InvokePrompt()
            }
        }
}

# ── zoxide: `z <partial>` jumps by frecency, `zi` picks interactively ────────
# `cd` is deliberately left as the real cd so scripts and muscle memory are safe.
if (Get-Command zoxide -CommandType Application -ErrorAction SilentlyContinue) {
    Use-CachedInit -Name 'zoxide' -Generator { zoxide init powershell }
}

# ── Listing: eza ─────────────────────────────────────────────────────────────
# The built-in `ls` alias would shadow a function of the same name, so drop it.
if (Get-Command eza -CommandType Application -ErrorAction SilentlyContinue) {
    if (Test-Path Alias:ls) { Remove-Item Alias:ls -Force }
    function ls { eza --icons=auto --group-directories-first @args }
    function ll { eza --icons=auto --group-directories-first --long --git --header @args }
    function la { eza --icons=auto --group-directories-first --long --all --git --header @args }
    function lt { eza --icons=auto --tree --level=2 --git-ignore @args }
}

# Icons for the native Get-ChildItem formatter — ~400 ms, so it is opt-in.
function Enable-TerminalIcons { Import-Module Terminal-Icons }

# Adds a completion-based predictor to the ListView. Slow to import; opt-in.
function Enable-CompletionPredictor {
    Import-Module CompletionPredictor
    Set-PSReadLineOption -PredictionSource HistoryAndPlugin
}

# ── Tool defaults ────────────────────────────────────────────────────────────
if (Get-Command bat -CommandType Application -ErrorAction SilentlyContinue) {
    $env:BAT_THEME = 'Visual Studio Dark+'
}
if (Get-Command code -ErrorAction SilentlyContinue) {
    $env:EDITOR     = 'code'
    $env:GIT_EDITOR = 'code --wait'
}
$PSStyle.Progress.View = 'Minimal'

# ── Navigation helpers ───────────────────────────────────────────────────────
function .. { Set-Location .. }
function ... { Set-Location ../.. }
function .... { Set-Location ../../.. }

function mkcd {
    param([Parameter(Mandatory)][string] $Path)
    New-Item -ItemType Directory -Path $Path -Force | Out-Null
    Set-Location -LiteralPath $Path
}

# Jump to the root of the current git repository.
function groot {
    $root = git rev-parse --show-toplevel 2>$null
    if ($root) { Set-Location -LiteralPath $root }
    else { Write-Error 'Not inside a git repository.' }
}

function which {
    param([Parameter(Mandatory)][string] $Name)
    $cmd = Get-Command $Name -ErrorAction SilentlyContinue
    if (-not $cmd) { Write-Error "'$Name' not found."; return }
    if ($cmd.Source) { $cmd.Source } else { $cmd.Name }
}

# ── Claude Code ──────────────────────────────────────────────────────────────
if (Get-Command claude -ErrorAction SilentlyContinue) {
    Set-Alias cc claude
    function ccc { claude --continue @args }   # resume the most recent session
    function ccr { claude --resume @args }     # pick a session to resume
}
'@ | Set-Content -Path $PROFILE -Encoding utf8
```

Open a new tab. The first start is slower, because it generates `%LOCALAPPDATA%\pwsh-cache\oh-my-posh.ps1` and `zoxide.ps1`. Later starts dot-source the cache.

::: note
**The theme path is baked into the cache.** The cached Oh My Posh init carries the theme path and the init code of the Oh My Posh version that generated it, and it regenerates only every 14 days. Run `Reset-ShellCache` and open a new tab after you edit or move the theme, and after every `winget upgrade JanDeDobbeleer.OhMyPosh`.
:::

### Write the Windows PowerShell 5.1 hand-off profile {.step minutes="1 min" shell="powershell"}

Windows PowerShell 5.1 can't be uninstalled. This profile makes an *interactive* 5.1 console start PowerShell 7 and exit when PowerShell 7 exits. Scripted callers (`-Command`, `-File`, `-EncodedCommand`), `-NoProfile` and the ISE still get real 5.1.

```powershell
$wps = Join-Path ([Environment]::GetFolderPath('MyDocuments')) 'WindowsPowerShell'
New-Item -ItemType Directory -Path $wps -Force | Out-Null
$wpsProfile = Join-Path $wps 'Microsoft.PowerShell_profile.ps1'
if (Test-Path $wpsProfile) { Copy-Item $wpsProfile "$wpsProfile.bak" }

@'
# Windows PowerShell 5.1 is an OS component and cannot be uninstalled, so this
# profile makes it unreachable instead: any *interactive* 5.1 session re-launches
# into pwsh 7 and exits. See the pwsh 7 profile for the real setup.
#
# Deliberately NOT redirected, so nothing that depends on powershell.exe breaks:
#   -Command / -File / -EncodedCommand   scripted callers (installers, MSI custom
#                                        actions, GPO) run 5.1 as they always did
#   -NoProfile                           skips this file entirely
#   ISE / any non-ConsoleHost host       $Host.Name check below
# The guard fails safe: anything ambiguous skips the handoff.

# ProgramW6432 first: under 32-bit powershell.exe $env:ProgramFiles resolves to
# "Program Files (x86)", where pwsh 7 does not exist.
$pf   = if ($env:ProgramW6432) { $env:ProgramW6432 } else { $env:ProgramFiles }
$pwsh = Join-Path $pf 'PowerShell\7\pwsh.exe'

$argv     = [Environment]::GetCommandLineArgs()
$scripted = [bool](@($argv) -match '^-(c|com|comm|command|f|fi|file|e|ec|enc|encodedcommand)$')

if (-not $scripted -and $Host.Name -eq 'ConsoleHost' -and
    [Environment]::UserInteractive -and (Test-Path -LiteralPath $pwsh)) {
    & $pwsh -NoLogo
    # `exit $LASTEXITCODE` does NOT propagate reliably from a dot-sourced profile
    # in Windows PowerShell -- it was observed returning 0 and 1 where pwsh exited 3.
    # [Environment]::Exit sets the process exit code directly. Safe here: pwsh has
    # already exited and this session has nothing to unwind.
    $code = if ($null -eq $LASTEXITCODE) { 0 } else { $LASTEXITCODE }
    [Environment]::Exit($code)
}

# Everything past here is the scripted / ISE path. It stays empty on purpose.
#
# The old profile imported Terminal-Icons and ran oh-my-posh here. Both are wrong
# for this path: neither a scripted caller nor ISE renders a themed prompt, and
# oh-my-posh init cost ~900 ms on every `powershell.exe -Command ...`.
#
# Terminal-Icons is worse than useless here. It lives only under the pwsh 7 module
# path, so a clean 5.1 session can't see it -- but a 5.1 process spawned FROM pwsh 7
# inherits that PSModulePath, finds the module, and then blows up loading it against
# the wrong module set (Import-PowerShellDataFile resolves to pwsh 7's Utility).
# Not importing it at all is the only guard that holds in both cases.
'@ | Set-Content -Path $wpsProfile -Encoding ascii
```

::: note
**ASCII on purpose.** Windows PowerShell 5.1 reads a BOM-less file as ANSI, not UTF-8. The file is pure ASCII, so `-Encoding ascii` keeps it readable by 5.1.
:::

### Write the Windows Terminal settings {.step minutes="2 min" shell="powershell"}

Windows Terminal creates `settings.json` the first time it starts, and it reloads the file as soon as it changes, so you can run this from inside Terminal.

```powershell
$wt = Join-Path $env:LOCALAPPDATA 'Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json'
Copy-Item $wt "$wt.bak-$(Get-Date -Format yyyyMMdd)"

@'
{
    "$help": "https://aka.ms/terminal-documentation",
    "$schema": "https://aka.ms/terminal-profiles-schema",
    "actions": 
    [
        {
            "command": 
            {
                "action": "sendInput",
                "input": "\u001b\r"
            },
            "id": "User.sendInput.shiftEnter"
        },
        {
            "command": 
            {
                "action": "scrollToMark",
                "direction": "next"
            },
            "id": "User.scrollToMark.next"
        },
        {
            "command": 
            {
                "action": "newTab",
                "profile": "Claude Code"
            },
            "id": "User.newTab.claudeCode"
        },
        {
            "command": 
            {
                "action": "copy",
                "singleLine": false
            },
            "id": "User.copy.644BA8F2"
        },
        {
            "command": "paste",
            "id": "User.paste"
        },
        {
            "command": "toggleFocusMode",
            "id": "User.toggleFocusMode"
        },
        {
            "command": 
            {
                "action": "splitPane",
                "split": "auto",
                "splitMode": "duplicate"
            },
            "id": "User.splitPane.A6751878"
        },
        {
            "command": "find",
            "id": "User.find"
        },
        {
            "command": "togglePaneZoom",
            "id": "User.togglePaneZoom"
        },
        {
            "command": 
            {
                "action": "clearBuffer",
                "clear": "all"
            },
            "id": "User.clearBuffer.all"
        },
        {
            "command": 
            {
                "action": "scrollToMark",
                "direction": "previous"
            },
            "id": "User.scrollToMark.previous"
        },
        {
            "command": 
            {
                "action": "splitPane",
                "profile": "Claude Code",
                "split": "auto"
            },
            "id": "User.splitPane.claudeCode"
        }
    ],
    "copyFormatting": "none",
    "copyOnSelect": false,
    "defaultProfile": "{5fb123f1-af88-5b5c-8953-d14a8def1978}",
    "keybindings": 
    [
        {
            "id": "User.sendInput.shiftEnter",
            "keys": "shift+enter"
        },
        {
            "id": "User.splitPane.A6751878",
            "keys": "alt+shift+d"
        },
        {
            "id": "User.clearBuffer.all",
            "keys": "ctrl+shift+k"
        },
        {
            "id": "User.copy.644BA8F2",
            "keys": "ctrl+c"
        },
        {
            "id": "User.toggleFocusMode",
            "keys": "alt+shift+f"
        },
        {
            "id": "User.togglePaneZoom",
            "keys": "ctrl+shift+z"
        },
        {
            "id": "User.find",
            "keys": "ctrl+shift+f"
        },
        {
            "id": "User.paste",
            "keys": "ctrl+v"
        },
        {
            "id": "User.newTab.claudeCode",
            "keys": "ctrl+alt+c"
        },
        {
            "id": "User.scrollToMark.previous",
            "keys": "ctrl+alt+up"
        },
        {
            "id": "User.scrollToMark.next",
            "keys": "ctrl+alt+down"
        },
        {
            "id": "User.splitPane.claudeCode",
            "keys": "alt+shift+c"
        }
    ],
    "newTabMenu": 
    [
        {
            "type": "remainingProfiles"
        }
    ],
    "profiles": 
    {
        "defaults": 
        {
            "autoMarkPrompts": true,
            "bellStyle": "none",
            "colorScheme": "Dark+",
            "font": 
            {
                "face": "Cascadia Code NF, MesloLGM Nerd Font, Consolas",
                "size": 11
            },
            "historySize": 50000,
            "padding": "8, 8, 8, 8",
            "showMarksOnScrollbar": true,
            "snapOnInput": true
        },
        "list": 
        [
            {
                "commandline": "%SystemRoot%\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
                "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
                "hidden": true,
                "name": "Windows PowerShell"
            },
            {
                "commandline": "%SystemRoot%\\System32\\cmd.exe",
                "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
                "hidden": false,
                "name": "Command Prompt"
            },
            {
                "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
                "hidden": false,
                "name": "Azure Cloud Shell",
                "source": "Windows.Terminal.Azure"
            },
            {
                "commandline": "pwsh.exe -NoLogo -NoExit -Command claude",
                "guid": "{d8fc3c3e-a92e-4aee-8f6e-66af28bb19fb}",
                "hidden": false,
                "historySize": 200000,
                "name": "Claude Code",
                "suppressApplicationTitle": true,
                "tabTitle": "Claude Code"
            },
            {
                "guid": "{2ece5bfe-50ed-5f3a-ab87-5cd4baafed2b}",
                "hidden": false,
                "name": "Git Bash",
                "source": "Git"
            },
            {
                "experimental.retroTerminalEffect": false,
                "guid": "{5fb123f1-af88-5b5c-8953-d14a8def1978}",
                "hidden": false,
                "name": "PowerShell 7",
                "opacity": 95,
                "source": "Windows.Terminal.PowershellCore"
            }
        ]
    },
    "schemes": [],
    "themes": [],
    "warning.largePaste": false,
    "warning.multiLinePaste": "never"
}
'@ | Set-Content -Path $wt -Encoding utf8
```

The profile GUIDs carry over between machines. `{5fb123f1-…}` (PowerShell 7), `{2ece5bfe-…}` (Git Bash) and `{b453ae62-…}` (Azure Cloud Shell) are generated by Terminal from fixed names, and `{d8fc3c3e-…}` (*Claude Code*) is defined in the file. *Git Bash* appears only once Git for Windows is installed (see [Git CLI Setup for Windows 11](git-cli-windows-setup.md)). *Claude Code* needs `claude` on `PATH`.

## How the pieces fit {#explained}

### What happens when a tab opens {.unnumbered}

| Order | Piece | What it does |
|---|---|---|
| 1 | Windows Terminal | Starts the default profile, *PowerShell 7* (`pwsh.exe`), at 95 % opacity, in `Cascadia Code NF` 11 pt, with the `Dark+` scheme |
| 2 | Profile: `Use-CachedInit` | Dot-sources `pwsh-cache\oh-my-posh.ps1`, which was generated with `oh-my-posh init pwsh --print`. This saves ~890 ms and one extra process per start. A fresh `POSH_SESSION_ID` keeps per-tab prompt state apart |
| 3 | Profile: PSReadLine | Windows edit mode. History predictions in **ListView** (only when the console supports VT and output isn't redirected). 20 000 history entries, no duplicates, no bell, grey inline prediction text. Lines that look like `password=` / `token:` / `apikey=` are kept out of the history file |
| 4 | Profile: fzf | Sets `FZF_DEFAULT_OPTS` and, when `fd` exists, `FZF_DEFAULT_COMMAND`. Binds `Ctrl+R`, `Ctrl+T`, `Alt+C` |
| 5 | Profile: zoxide | Dot-sources the cached `zoxide init powershell`. `cd` stays the real `cd` |
| 6 | Profile: eza, tools, helpers | Replaces the `ls` alias with eza functions. Sets `BAT_THEME`, `EDITOR`, `GIT_EDITOR`, minimal progress bars. Defines the navigation and Claude Code shortcuts |
| 7 | Oh My Posh | Draws the prompt and sends FTCS marks (`shell_integration`) and the working folder (`OSC 9;9`) to Terminal |

Startup on the source machine, measured with `Measure-Command { pwsh -NoLogo -Command exit }`: **~0.7 s** with the profile, **~0.26 s** with `-NoProfile`.

### Keys and commands {.unnumbered}

| Where | Key / command | Does |
|---|---|---|
| PSReadLine | `Tab` | Menu completion |
| PSReadLine | `↑` / `↓` | History search on what is already typed |
| PSReadLine | `F2` | Switch predictions between ListView and InlineView |
| PSReadLine | `Ctrl+W` / `Alt+D` | Delete word back / forward |
| fzf | `Ctrl+R` | Fuzzy search of the whole history file, newest first |
| fzf | `Ctrl+T` | Pick files and insert their paths (quoted when they contain spaces) |
| fzf | `Alt+C` | Pick a subfolder and `cd` into it |
| fzf | `Ctrl+/` (inside fzf) | Toggle the preview pane |
| zoxide | `z <part>` / `zi` | Jump by frecency / pick interactively |
| eza | `ls` · `ll` · `la` · `lt` | Icons, folders first · long with Git status · plus hidden · tree, 2 levels |
| Profile | `..` · `...` · `....` · `mkcd` · `groot` · `which` | Up 1–3 folders · make and enter · Git repo root · resolve a command |
| Profile | `cc` · `ccc` · `ccr` | `claude` · `claude --continue` · `claude --resume` |
| Profile | `Reset-ShellCache` · `Enable-TerminalIcons` · `Enable-CompletionPredictor` | Rebuild init caches · opt-in modules |
| Terminal | `Ctrl+Alt+↑` / `Ctrl+Alt+↓` | Jump to the previous / next prompt mark |
| Terminal | `Alt+Shift+D` | Split pane, duplicating the current profile **and folder** |
| Terminal | `Ctrl+Alt+C` / `Alt+Shift+C` | New *Claude Code* tab / split |
| Terminal | `Shift+Enter` | Sends `ESC` + `CR`, a newline inside Claude Code's input box |
| Terminal | `Ctrl+C` / `Ctrl+V` | Copy (only when text is selected, otherwise interrupt) / paste |
| Terminal | `Ctrl+Shift+F` · `Ctrl+Shift+K` · `Ctrl+Shift+Z` · `Alt+Shift+F` | Find · clear buffer · zoom pane · focus mode |

Terminal also turns on `autoMarkPrompts` and `showMarksOnScrollbar` for every profile, keeps 50 000 lines of scrollback (200 000 in *Claude Code*), copies plain text only (`copyFormatting: none`), and shows **no** large or multi-line paste warnings.

### The prompt, explained {#prompt .unnumbered}

Left, in order:

| Segment | Shows | When |
|---|---|---|
| `session` | User name | Always |
| `path` | Current folder, `~` for home | Always |
| `git` | Upstream icon, branch (25 chars max), ahead/behind, working and staged counts, stash count | In a repo. Yellow when clean, orange when dirty, purple when ahead or behind, red-orange when both |
| `node`, `go`, `julia`, `python`, `ruby`, `azfunc` | Tool version | Only in a folder with that language's files |
| `aws` | AWS profile and region | Only when a non-default profile is set |
| `root` | Wrench | In an elevated shell |
| `executiontime` | Last command's duration | Always |
| `status` | Green, or pink on a non-zero exit code | Always |

Right: shell name, the YouTube Music track (when the YouTube Music desktop app is playing), battery, and the time. The window title reads `pwsh in <folder>`. Oh My Posh never updates itself (`upgrade.auto: false`), so upgrade it with winget.

## Prove it matches {#verify}

In a **new PowerShell 7 tab**:

```powershell
$PSVersionTable.PSVersion                      # 7.6.9
oh-my-posh version                             # 31.9.1
$env:POSH_CONFIG                               # C:\Users\<you>\.config\oh-my-posh\jandedobbeleer-custom.omp.json
Get-ChildItem $env:LOCALAPPDATA\pwsh-cache     # oh-my-posh.ps1, zoxide.ps1

Get-PSReadLineOption | Select-Object EditMode, PredictionSource, PredictionViewStyle
                                               # Windows, History, ListView
Get-PSReadLineKeyHandler -Bound | Where-Object Key -in 'Ctrl+r','Ctrl+t','Alt+c' | Select-Object Key, Function
                                               # FuzzyHistory, FuzzyFile, FuzzyChangeDirectory

Get-ItemProperty 'HKCU:\Console\%%Startup' | Select-Object Delegation*
                                               # {2EACA947-…}, {E12CFF52-…}
Measure-Command { pwsh -NoLogo -Command exit } # ~0.7 s on the source machine
```

Then check by eye:

1. The prompt draws diamonds, powerline arrows and icons, with no boxes or `?`.
2. In a Git repo, the Git segment turns orange after you edit a file.
3. Run a few commands, then press `Ctrl+Alt+↑`. Terminal jumps from prompt to prompt.
4. `cd` somewhere deep, press `Alt+Shift+D`. The new pane opens in the same folder.
5. Run `powershell` from the Run dialog. A Windows Terminal tab opens, and `$PSVersionTable.PSEdition` there prints `Core`.
6. `powershell -NoProfile -Command '$PSVersionTable.PSVersion.Major'` prints `5`, so scripted 5.1 still works.

## Faults, and what they actually mean {#faults}

::: faults

#### Boxes or `?` in the prompt
Neither Cascadia Code NF nor MesloLGM Nerd Font is installed, or Terminal didn't pick up the font yet. Re-run Step 3, then restart Windows Terminal completely. Other hosts (the VS Code terminal, a plain console) need their own font set to a Nerd Font.

#### Edited the theme, but the prompt didn't change
The theme path and init code come from the cache. Run `Reset-ShellCache` and open a new tab.

#### Errors from Oh My Posh functions after an upgrade
The cache still holds the older version's init script. Run `Reset-ShellCache` and open a new tab.

#### `oh-my-posh`, `fzf`, `z` or `eza` not found
The tab was opened before winget changed `PATH`. Open a new tab. Apps that were already running need a restart for the same reason.

#### `… cannot be loaded because running scripts is disabled`
The profile was copied from a downloaded file and still carries the *Mark of the Web*. Run `Unblock-File $PROFILE`. Don't loosen the execution policy.

#### Typing `powershell` lands in PowerShell 7
That is the hand-off from Step 8, by design. For real 5.1, run `powershell -NoProfile`, or give it a `-Command` / `-File`.

#### Predictions don't appear
Output was redirected, or the host has no VT support. The profile skips predictions then, on purpose. In a normal Terminal tab, check `(Get-PSReadLineOption).PredictionSource`.

#### winget: *No version found matching*
That build is no longer in the catalogue. Drop `--version` and note the version you got.

:::

## Audit findings on the source machine {#audit}

These are observations only. The rebuild above reproduces the source machine as it is.

1. **posh-git is installed but never loaded.** Nothing imports it. The Oh My Posh `git` segment does its job. You can uninstall it (`Uninstall-PSResource posh-git`) and leave it out of Step 5.
2. **The init cache can outlive an Oh My Posh upgrade.** It regenerates only after 14 days, so after `winget upgrade` the shell can run the old init script against the new exe until you run `Reset-ShellCache`. Making that part of the upgrade routine closes the gap.
3. **The history filter only catches `key=` and `key:` forms.** `$token = '…'` and `password: …` stay out of history, but parameter forms such as `-Password …` or `-Token …` are still saved to the history file. Avoid secrets on the command line. The filter isn't a safety net for them.
4. **Paste warnings are off.** `warning.largePaste` is `false` and `warning.multiLinePaste` is `never`. In PowerShell 7 a multi-line paste lands in the edit buffer, but in *Command Prompt* or *Git Bash* each pasted line runs at once.
5. **The theme carries segments that this machine doesn't use.** `julia`, `ruby`, `azfunc`, `aws` and `ytm` come from the upstream theme. They cost nothing until their trigger appears, but the `fetch_version` segments start a process (`node --version`, `go version`, …) whenever you are in a matching folder.
6. **Old backups sit next to the live settings.** `LocalState` holds three `settings.json.bak-*` files from August 2026, and `WindowsPowerShell` holds `Microsoft.PowerShell_profile.ps1.bak-20260826` (the old 5.1 profile, which imported Terminal-Icons and ran Oh My Posh). They are harmless, and they aren't part of the rebuild.

::: footer
Inventory method: `winget list` and `winget show --versions`; `Get-AppxPackage`; `Get-InstalledPSResource` and `Get-Module -ListAvailable`; both PowerShell profiles; `~\.config\oh-my-posh\jandedobbeleer-custom.omp.json`, compared against `oh-my-posh config export --config jandedobbeleer`; Windows Terminal `settings.json` and `state.json`; `HKCU`/`HKLM` font registry; `HKCU\Console\%%Startup`; `$PSHOME\powershell.config.json`; persisted User environment; `Measure-Command` startup timings. No secrets were found, and none are recorded here.
:::
