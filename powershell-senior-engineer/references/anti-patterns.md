# Anti-Patterns — Never Do These

These are the specific shortcuts AI tools and junior engineers reach for.
Reject them in all generated and reviewed code.

---

## Character Encoding — ASCII-Safe Output for PS 5.1 Scripts

**This rule applies to any script that targets or must run on Windows PowerShell 5.1.**
PS 7 reads files as UTF-8 by default and handles Unicode characters correctly, so this
rule does NOT apply to PS 7-only scripts as long as the file is saved as UTF-8.

**Why it matters for PS 5.1:**
PS 5.1 defaults to the system code page (Windows-1252 / ANSI on English Windows) when
reading a file with no BOM.  AI tools generate UTF-8 text containing Unicode punctuation
(em dashes `—`, smart quotes `"` `"` `'` `'`, ellipsis `…`, etc.).  When that file is
copied between machines or editors without preserving encoding, those multi-byte sequences
are misread as Windows-1252 and produce garbage like `a EUR"` — which breaks parsing.

```powershell
# WRONG for PS 5.1 — Unicode punctuation breaks after copy/transfer in enterprise envs
Write-Output "Cleaning temp files — this may take a moment..."
if ($PSCmdlet.ShouldProcess("$label — $sizeDisplay", 'Clear')) { ... }

# RIGHT for PS 5.1 — plain ASCII only
Write-Output "Cleaning temp files - this may take a moment..."
if ($PSCmdlet.ShouldProcess("$label - $sizeDisplay", 'Clear')) { ... }
```

**Characters to avoid in PS 5.1 .ps1 files:**

| Character | Unicode | Replace with |
|---|---|---|
| Em dash `—` | U+2014 | Hyphen-minus `-` |
| En dash `–` | U+2013 | Hyphen-minus `-` |
| Smart quote `"` `"` | U+201C / U+201D | Straight double quote `"` |
| Smart apostrophe `'` `'` | U+2018 / U+2019 | Straight single quote `'` |
| Ellipsis `…` | U+2026 | Three dots `...` |
| Non-breaking space | U+00A0 | Regular space |

**Save PS 5.1 scripts as UTF-8 without BOM** (or ASCII).
UTF-8 with BOM causes issues on older PS 5.1 builds; ANSI mangles anything above U+007F.
PS 7 scripts: save as UTF-8 (with or without BOM — both work).

---

## Version and Compatibility

```powershell
# WRONG — Get-WmiObject is removed in PS7
Get-WmiObject -Class Win32_OperatingSystem

# RIGHT
Get-CimInstance -ClassName Win32_OperatingSystem
```

```powershell
# WRONG — $IsWindows is undefined in PS 5.1; causes silent failures
if ($IsWindows) { ... }

# RIGHT — gate with a version check first
if ($PSVersionTable.PSVersion.Major -ge 7 -and $IsWindows) { ... }
```

```powershell
# WRONG — aliases are ambiguous and behave differently across versions
$users | % { $_.DisplayName }
$items | ? { $_.Enabled }

# RIGHT — always use full cmdlet names in scripts
$users | ForEach-Object { $_.DisplayName }
$items | Where-Object { $_.Enabled }
```

```powershell
# WRONG — Set-StrictMode -Version Latest throws when Measure-Object receives no
#         pipeline input and returns $null; accessing .Sum on $null is a property
#         error under strict mode even though you check for $null on the next line.
Set-StrictMode -Version Latest
$sum = (Get-ChildItem $path -File -Recurse -ErrorAction SilentlyContinue |
    Measure-Object -Property Length -Sum).Sum    # THROWS if directory is empty
if ($null -eq $sum) { return 0 }

# RIGHT — assign to an intermediate variable, then inspect the object before
#         touching any of its properties.
$measured = Get-ChildItem $path -File -Recurse -ErrorAction SilentlyContinue |
    Measure-Object -Property Length -Sum
if ($null -eq $measured -or $null -eq $measured.Sum) { return [long]0 }
return [long]$measured.Sum
```

```powershell
# WRONG — [ordered]@{} creates System.Collections.Specialized.OrderedDictionary
#         which is a non-generic IDictionary.  It does NOT have .ContainsKey().
#         This throws on PS 5.1 at runtime even though it looks correct.
$map = [ordered]@{ Foo = 1; Bar = 2 }
if ($map.ContainsKey('Foo')) { ... }   # MethodNotFound exception on PS 5.1

# RIGHT — OrderedDictionary uses .Contains()
if ($map.Contains('Foo')) { ... }

# NOTE — plain @{} (Hashtable) DOES have .ContainsKey(); the rule only applies
#         to [ordered]@{} / OrderedDictionary.
$ht = @{ Foo = 1 }
if ($ht.ContainsKey('Foo')) { ... }    # Fine on both PS 5.1 and PS 7
```

---

## Error Handling

```powershell
# WRONG — silently swallows every error
try { Do-Something }
catch { }

# WRONG — global scope bleeds into all called functions and cmdlets
$ErrorActionPreference = 'Stop'

# WRONG — bare throw loses caller attribution in advanced functions
catch { throw $_ }

# RIGHT — rethrow with context preserved
catch { $PSCmdlet.ThrowTerminatingError($_) }
```

```powershell
# WRONG — catch-all with no specific handling hides root cause
catch {
    Write-Host "Something went wrong"
}

# RIGHT — specific exception type + meaningful log + rethrow or handle
catch [System.UnauthorizedAccessException] {
    Write-Warning "Access denied to $targetPath — check permissions."
    $PSCmdlet.ThrowTerminatingError($_)
}
```

---

## Hardcoding

```powershell
# WRONG — breaks in every other environment; deadly in MSP/consultant work
$ouPath   = 'OU=Users,OU=Corp,DC=contoso,DC=com'
$tenantId = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
$domain   = 'contoso.com'

# RIGHT — every environment-specific value is a parameter
param(
    [Parameter(Mandatory)] [string]$OUPath,
    [Parameter(Mandatory)] [string]$TenantId,
    [Parameter(Mandatory)] [string]$Domain
)
```

---

## Output and Logging

```powershell
# WRONG — not pipeline-friendly; breaks automated consumers
Write-Host "Found $($users.Count) users"

# WRONG — in Intune/OSD context, Write-Host pollutes stdout
#         which is evaluated for exit state
Write-Host 'Done'

# WRONG — returning formatted strings instead of objects
return "User: $($user.DisplayName), Status: $($user.AccountEnabled)"

# RIGHT
Write-Verbose "Found $($users.Count) users"    # operational detail
Write-Output $resultObject                      # pipeline output — emit objects
```

---

## Parameters

```powershell
# WRONG — positional parameters are ambiguous and fragile
function Get-UserData($Username, $Domain) { ... }

# WRONG — bool param forces caller to pass $true / $false explicitly
param( [bool]$Force )

# WRONG — long parameter lines are hard to read and diff
New-ADUser -Name $name -SamAccountName $sam -UserPrincipalName $upn -Path $ou -AccountPassword $pw -Enabled $true

# RIGHT
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string]$Username,

    [Parameter()]
    [string]$Domain,

    [Parameter()]
    [switch]$Force
)

# RIGHT — splatting for long calls
$adParams = @{
    Name              = $name
    SamAccountName    = $sam
    UserPrincipalName = $upn
    Path              = $ou
    AccountPassword   = $pw
    Enabled           = $true
}
New-ADUser @adParams
```

---

## MSP / Multi-Tenant

```powershell
# WRONG — assumes whatever Graph session is already open is the right tenant
#         In an MSP context this is a catastrophic assumption
Get-MgUser -All

# WRONG — reusing a credential object across tenants
$cred = Get-StoredCredential 'MSP-Admin'
Connect-MgGraph -Credential $cred   # which tenant does this hit?

# RIGHT — connect explicitly per tenant, confirm tenant ID, disconnect in finally
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId -NoWelcome
Write-Verbose "Connected to tenant: $((Get-MgContext).TenantId)"
try   { Get-MgUser -All }
finally { Disconnect-MgGraph -ErrorAction SilentlyContinue }
```

---

## Intune / OSD Specific

```powershell
# WRONG — detection script exits 0 even when an error occurs
try { ... }
catch { Write-Host "Error"; exit 0 }   # This tells Intune the device is compliant!

# RIGHT — always exit 1 on unhandled errors in detection/remediation
catch {
    Write-Warning "Unhandled error: $_"
    exit 1
}
```

```powershell
# WRONG — remediation script is not idempotent
# (fails or produces duplicate state if run twice)
Add-Content -Path $logFile -Value "Remediation ran"

# RIGHT — check state before acting; running twice must be safe
if (-not (Test-Path $logFile)) {
    Set-Content -Path $logFile -Value "Remediation ran"
}
```

```powershell
# WRONG — assuming drive letters in WinPE
$source = 'C:\Temp\file.txt'

# RIGHT — resolve paths dynamically; no assumptions about drive letters in WinPE
$source = Join-Path $PSScriptRoot 'file.txt'
```
