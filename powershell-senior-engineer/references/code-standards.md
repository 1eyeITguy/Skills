# Code Standards

## Style and Structure

- Use approved PowerShell verbs (`Get-`, `Set-`, `New-`, `Remove-`, `Invoke-`, etc.).
- Every function and script must have `[CmdletBinding()]` and a `param()` block.
- Use PascalCase for function and parameter names; camelCase for internal variables.
- Opening braces on the same line; closing braces on a new line.
- 4-space indentation throughout.
- Prefer splatting over long parameter lines:

  ```powershell
  $params = @{
      Path        = $targetPath
      Recurse     = $true
      ErrorAction = 'Stop'
  }
  Get-ChildItem @params
  ```

- Prefix internal/helper functions with a project noun (`IRIS-`, `SS-`, `MSP-`, etc.)
  to avoid namespace collisions.
- **Use ASCII-only characters in all .ps1 files.** Never use Unicode punctuation
  (em dashes, smart quotes, ellipsis, non-breaking spaces, etc.) — they cause parse
  failures on PS 5.1 when files are copied or re-encoded.  Use `-` not `—`, `'` not
  `'`, `"` not `"`.  Save files as **UTF-8 without BOM**.

---

## Comment-Based Help

Required on every public function:

```powershell
<#
.SYNOPSIS     One-line description.
.DESCRIPTION  Full description.  For MSP/consultant scripts, include assumptions
              about the target environment.
.PARAMETER    ParameterName  What it does.  Note if required vs optional.
.OUTPUTS      [PSCustomObject] What the function returns.
.EXAMPLE      Show a realistic usage example, not just the function name.
.NOTES
    Author:      <name>
    Date:        YYYY-MM-DD
    Version:     1.0.0
    PSVersion:   7.0 / 5.1
    Context:     SYSTEM / interactive / WinPE / MSP automation
    ExitCodes:   0 = success, 1 = failure  (add more if applicable)
    Assumptions: List any environment assumptions (hybrid join, P2 license, etc.)
#>
```

---

## Parameter Design

- `[Parameter(Mandatory)]` — not `[Parameter(Mandatory = $true)]`.
- `[switch]` over `[bool]` for flags — switch defaults to `$false` when omitted;
  never require the caller to pass `$true` or `$false`.
- Apply validation attributes on every parameter:

  ```powershell
  [ValidateNotNullOrEmpty()]
  [ValidateSet('Dev', 'Test', 'Prod')]
  [ValidateRange(1, 100)]
  [ValidatePattern('^[A-Z]{2,5}$')]
  ```

- `ValueFromPipeline` for direct object input;
  `ValueFromPipelineByPropertyName` for property-name binding.
- `-PassThru` switch on action cmdlets that don't normally produce output:

  ```powershell
  [Parameter()] [switch]$PassThru
  if ($PassThru) { Write-Output $modifiedObject }
  ```

- Pair `SupportsShouldProcess` with a `-Force` switch so automation pipelines
  can suppress prompts.

---

## Pipeline Structure

Always implement `begin`, `process`, and `end` blocks in pipeline-aware functions.
Stream one object at a time through `process` — never bulk-collect in `begin`
and process in `end`:

```powershell
begin   { Write-Verbose "Starting $($MyInvocation.MyCommand.Name)" }
process { <# one object at a time #> }
end     { Write-Verbose "Completed $($MyInvocation.MyCommand.Name)" }
```

---

## ShouldProcess and Destructive Operations

Any function that modifies, deletes, or overwrites state must implement
`SupportsShouldProcess`.  Especially critical in MSP/consultant scripts.

```powershell
[CmdletBinding(SupportsShouldProcess, ConfirmImpact = 'High')]

if ($Force -or $PSCmdlet.ShouldProcess($targetName, 'Remove user account')) {
    Remove-ADUser -Identity $targetName -ErrorAction Stop
}
```

| ConfirmImpact | When to use |
|---|---|
| `'Low'`    | Informational or easily reversible |
| `'Medium'` | Reversible modifications |
| `'High'`   | Deletions, destructive, or irreversible |

- Use `$PSCmdlet.ShouldContinue()` for a second gate on bulk destructive operations.

---

## Error Handling

- `try/catch/finally` with specific exception types where possible.
- In advanced functions, use `$PSCmdlet.WriteError()` (non-terminating) and
  `$PSCmdlet.ThrowTerminatingError()` (terminating) — not bare `Write-Error` / `throw`.
  Both produce properly attributed error records that point to the caller.
- Construct `[ErrorRecord]` objects explicitly:

  ```powershell
  catch [System.IO.IOException] {
      $record = [System.Management.Automation.ErrorRecord]::new(
          $_.Exception,
          'FileAccessError',
          [System.Management.Automation.ErrorCategory]::ReadError,
          $targetPath
      )
      $PSCmdlet.ThrowTerminatingError($record)
  }
  catch {
      $PSCmdlet.ThrowTerminatingError($_)   # rethrow with context preserved
  }
  ```

- `Write-Warning` for non-fatal conditions; `Write-Verbose` for operational detail.
- Never `Write-Host` in non-interactive, Intune, or OSD scripts.
- Avoid `$ErrorActionPreference = 'Stop'` as a global; scope it to a `try` block
  or use `-ErrorAction Stop` per call.
- Intune / OSD exit codes: `0` = success/compliant, `1` = failure/non-compliant.
  Document additional codes in `.NOTES`.

---

## Active Directory / Entra ID

- `Microsoft.Graph` SDK for all Entra/Graph work; never `AzureAD` or `MSOnline`.
- For on-prem AD, always filter at the server (`-Filter`, `-LDAPFilter`) — never
  pull everything and filter client-side with `Where-Object`.
- OU paths are always variables/parameters — never hardcoded distinguished names.

---

## Intune / OSDCloud / OSDeploy / OSD.Workspace

- OSDCloud / WinPE: no profile, no drive letters, no proxy — handle all explicitly.
- PSADT: follow `AppDeployToolkit.ps1` structure; toolkit logging functions only —
  never write a parallel logging layer.
- Intune remediations: idempotent, exit `0` or `1`, no `Write-Host`.
- Prefer OSD module functions (`Get-OSDCloudVolume`, etc.) over raw CIM calls
  when an OSD equivalent exists.

---

## Module Authoring

- `.psm1` + `.psd1` manifest; export only public functions with `Export-ModuleMember`.
- Semantic versioning (`Major.Minor.Patch`).
- `CompatiblePSEditions = @('Desktop', 'Core')` unless a hard dependency prevents it.
- For MSP/consultant modules: include a `README.md` with required permissions,
  dependencies, and a quickstart example.

---

## Security

- No plaintext credentials — `SecureString`, DPAPI, Keeper, or Azure Key Vault.
- Validate all external input; no `Invoke-Expression` on untrusted strings.
- Sign scripts intended for production deployment.
- For MSP: use app registrations with least-privilege Graph scopes per tenant;
  never reuse credentials across tenants.
