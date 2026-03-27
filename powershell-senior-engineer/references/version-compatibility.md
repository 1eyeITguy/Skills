# PowerShell Version Compatibility

## The Rule

**Default to PS7.  Degrade gracefully to 5.1 where required.**

---

## Target PS7 Exclusively When

- Interactive admin tooling, module development, scheduled tasks on managed servers,
  MSP/consultant automation, anything invoked through VS Code or the PS7 shell.
- Use `#Requires -Version 7.0` and take full advantage of PS7 features.

## Maintain 5.1 Compatibility When

- Intune detection / remediation scripts (run under Windows PowerShell 5.1).
- PSADT wrappers (AppDeployToolkit runs under 5.1).
- OSDCloud / WinPE tasks (WinPE ships 5.1; PS7 must be explicitly injected).
- Any script deployed via GPO or SCCM to endpoints that may not have PS7.
- When in doubt for endpoint-facing scripts, target 5.1 and note it in `.NOTES`.

---

## PS7 Features — Use Freely in PS7-Only Contexts

```powershell
$x    = $condition ? 'yes' : 'no'               # Ternary
$val  = $input ?? 'default'                     # Null coalescing
$prop = $obj?.Property                          # Null conditional
ForEach-Object -Parallel { } -ThrottleLimit 5   # Always set -ThrottleLimit explicitly
cmd1 && cmd2 || fallback                        # Pipeline chain operators
Get-Error                                       # Rich exception detail
ConvertTo-Json -Depth 99                        # Always set -Depth (default is 2 in 5.1)
Join-Path / $PSScriptRoot                       # Cross-platform paths; no hardcoded backslashes
```

---

## Compatibility Patterns for Dual-Version Scripts

Declare minimum version in `#Requires`; for dual-version use `#Requires -Version 5.1`
and guard PS7-only blocks:

```powershell
if ($PSVersionTable.PSVersion.Major -ge 7) { <# PS7 path #> }
else                                        { <# 5.1 fallback #> }
```

| Topic | Rule |
|---|---|
| Aliases | Avoid `%`, `?`, `foreach` inline — behaviour differs or absent across versions |
| WMI | `Get-CimInstance` works in both; `Get-WmiObject` is removed in PS7 — never use it |
| Parallel | `ForEach-Object -Parallel` does not exist in 5.1 — provide a sequential fallback |
| OS variables | `$IsWindows` / `$IsLinux` / `$IsMacOS` do not exist in 5.1 — gate with version check |
| SecureString | DPAPI fine for Windows-only; note in `.NOTES` if cross-platform is possible |
| Regex | Test non-trivial patterns in both — PS7 uses .NET 7+ regex, 5.1 uses .NET 4.x |
| JSON depth | Always set `-Depth` explicitly — default is 2 in 5.1, 99 recommended |

---

## Module Compatibility

- Prefer `Microsoft.Graph` SDK over deprecated `AzureAD` / `MSOnline`
  (those only run on Desktop / 5.1).
- Set `CompatiblePSEditions` in `.psd1` for modules that must load in both:

  ```powershell
  CompatiblePSEditions = @('Desktop', 'Core')
  ```

- `Import-WinModule` (WindowsCompatibility) to shim 5.1-only modules into PS7
  is a last resort — document why it is needed.
