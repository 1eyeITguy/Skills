# Script Region Layout Templates

Use `#region` / `#endregion` to organise scripts into consistent, navigable sections.
Pick the template that matches the execution context from the decision tree in `SKILL.md`.

---

## Template A — Intune Detection or Remediation Script

Context: Runs as SYSTEM under Windows PowerShell 5.1.  No user profile.
Exit 0 = compliant/success.  Exit 1 = non-compliant/failure.

```powershell
#Requires -Version 5.1
<#
.SYNOPSIS   Detects / remediates <condition>.
.NOTES
    Author:    <n>
    Date:      YYYY-MM-DD
    Version:   1.0.0
    Context:   Intune — SYSTEM, Windows PowerShell 5.1
    ExitCodes: 0 = compliant/success, 1 = non-compliant/failure
#>

#region Configuration
$setting = 'ExpectedValue'
$regPath = 'HKLM:\Software\...'
$regName = 'SettingName'
#endregion

#region Functions
function Test-Compliance {
    [CmdletBinding()]
    param()
    # Return $true if compliant, $false if not
}
#endregion

#region Main
try {
    if (Test-Compliance) {
        Write-Output 'Compliant'
        exit 0
    }
    else {
        # Detection script:  exit 1 here
        # Remediation script: fix the condition, then exit 0 if successful
        Write-Output 'Non-compliant'
        exit 1
    }
}
catch {
    Write-Warning "Unhandled error: $_"
    exit 1
}
#endregion

<#
VALIDATION STEPS
----------------
1. Simulate compliant state → run script → confirm $LASTEXITCODE = 0
2. Simulate non-compliant state → run script → confirm $LASTEXITCODE = 1
3. Run remediation → confirm $LASTEXITCODE = 0
4. Re-run detection → confirm $LASTEXITCODE = 0  (idempotency)
5. Re-run remediation → confirm $LASTEXITCODE = 0  (idempotency)
#>
```

---

## Template B — OSDCloud / WinPE Task Script

Context: WinPE environment.  No drive letters, no proxy, no user profile.
PS7 is not guaranteed unless explicitly injected.

```powershell
#Requires -Version 5.1
<#
.SYNOPSIS   OSDCloud task — <description>.
.NOTES
    Author:    <n>
    Date:      YYYY-MM-DD
    Version:   1.0.0
    Context:   WinPE / OSDCloud — no profile, no drive letters, no proxy
    ExitCodes: 0 = success, 1 = failure
#>

#region WinPE Safety Checks
if (-not (Get-Module -ListAvailable -Name OSD)) {
    Write-Warning 'OSD module not available — is this WinPE with OSD injected?'
    exit 1
}
#endregion

#region Configuration
$targetDisk = 0
$osLanguage = 'en-us'
#endregion

#region Functions
function Invoke-OSDTask {
    [CmdletBinding()]
    param()
    # Use OSD module functions; avoid raw CIM/WMI where an OSD equivalent exists
}
#endregion

#region Main
try {
    Invoke-OSDTask
    exit 0
}
catch {
    Write-Warning "OSDCloud task failed: $_"
    exit 1
}
#endregion

<#
VALIDATION STEPS
----------------
1. Run with -Verbose to confirm each stage logs as expected.
2. Verify OSD module version matches expected syntax.
3. Test in a WinPE lab environment before deploying to production imaging.
4. Confirm $LASTEXITCODE after execution.
#>
```

---

## Template C — Interactive / Scheduled Admin Script (PS7)

Context: Engineer-run or service-account scheduled task.
Full PS7 features available.  ShouldProcess required for destructive operations.

```powershell
#Requires -Version 7.0
<#
.SYNOPSIS     <Action> — short description.
.DESCRIPTION  Full description.
.PARAMETER    TargetOU   Distinguished name of the OU to process.
.PARAMETER    Force      Suppress ShouldProcess confirmation prompts.
.OUTPUTS      [PSCustomObject[]] One result object per processed item.
.EXAMPLE
    .\Invoke-AdminTask.ps1 -TargetOU 'OU=Users,DC=contoso,DC=com'
.NOTES
    Author:      <n>
    Date:        YYYY-MM-DD
    Version:     1.0.0
    PSVersion:   7.0
    Context:     Interactive or scheduled task — service account or engineer
    Assumptions: ActiveDirectory module available; appropriate AD permissions
#>

[CmdletBinding(SupportsShouldProcess, ConfirmImpact = 'Medium')]
param(
    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string]$TargetOU,

    [Parameter()]
    [switch]$Force
)

#region Bootstrap
Set-StrictMode -Version Latest
#endregion

#region Functions
function Invoke-AdminTask {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string]$TargetOU
    )
    begin   { Write-Verbose "Starting $($MyInvocation.MyCommand.Name)" }
    process {
        if ($Force -or $PSCmdlet.ShouldProcess($TargetOU, 'Apply change')) {
            # Do work here
        }
    }
    end     { Write-Verbose "Completed $($MyInvocation.MyCommand.Name)" }
}
#endregion

#region Main
try {
    Invoke-AdminTask -TargetOU $TargetOU
}
catch {
    $PSCmdlet.ThrowTerminatingError($_)
}
#endregion

<#
VALIDATION STEPS
----------------
1. Run with -WhatIf first — confirm listed actions are correct.
2. Run with -Verbose — confirm each decision point logs as expected.
3. Run against a test OU / single object before full scope.
4. Confirm $LASTEXITCODE after execution.
#>
```

---

## Template D — MSP / Consultant Multi-Tenant Script

Context: Targets an arbitrary tenant by parameter.
All tenant-specific values are mandatory parameters.
Always connect and disconnect explicitly; never assume an open session.

```powershell
#Requires -Version 7.0
#Requires -Modules Microsoft.Graph.Authentication
<#
.SYNOPSIS     <Action> for a target tenant.
.DESCRIPTION  Designed to run against any tenant — all tenant-specific values
              are parameterised.  Run against a lab tenant with -WhatIf first.
.PARAMETER    TenantId   Entra tenant ID (GUID) of the target tenant.
.PARAMETER    ClientId   App registration client ID for Graph authentication.
.PARAMETER    Force      Suppress ShouldProcess confirmation prompts.
.OUTPUTS      [PSCustomObject[]] One result object per processed item.
.EXAMPLE
    .\Invoke-TenantTask.ps1 -TenantId 'xxxxxxxx-...' -ClientId 'yyyyyyyy-...' -WhatIf
.NOTES
    Author:      <n>
    Date:        YYYY-MM-DD
    Version:     1.0.0
    PSVersion:   7.0
    Context:     Interactive / scheduled — not intended to run as SYSTEM
    Assumptions: App registration with <required Graph scopes> pre-consented
    Permissions: <list minimum required Graph API permissions here>
    ExitCodes:   0 = success, 1 = failure
#>

[CmdletBinding(SupportsShouldProcess, ConfirmImpact = 'Medium')]
param(
    [Parameter(Mandatory)]
    [ValidatePattern('^[0-9a-fA-F\-]{36}$')]
    [string]$TenantId,

    [Parameter(Mandatory)]
    [ValidatePattern('^[0-9a-fA-F\-]{36}$')]
    [string]$ClientId,

    [Parameter()]
    [switch]$Force
)

#region Bootstrap
Set-StrictMode -Version Latest
#endregion

#region Authentication
function Connect-Tenant {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string]$TenantId,
        [Parameter(Mandatory)] [string]$ClientId
    )
    $connectParams = @{
        TenantId  = $TenantId
        ClientId  = $ClientId
        Scopes    = @('User.Read.All')   # Adjust to minimum required
        NoWelcome = $true
    }
    Connect-MgGraph @connectParams
    Write-Verbose "Connected to tenant: $((Get-MgContext).TenantId)"
}
#endregion

#region Functions
function Invoke-TenantTask {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string]$TenantId
    )
    begin   { Write-Verbose "Starting $($MyInvocation.MyCommand.Name) for tenant $TenantId" }
    process {
        if ($Force -or $PSCmdlet.ShouldProcess($TenantId, 'Apply change')) {
            # Do work here
        }
    }
    end     { Write-Verbose "Completed $($MyInvocation.MyCommand.Name)" }
}
#endregion

#region Main
try {
    Connect-Tenant -TenantId $TenantId -ClientId $ClientId
    Invoke-TenantTask -TenantId $TenantId
}
catch {
    Write-Warning "Script failed for tenant ${TenantId}: $_"
    exit 1
}
finally {
    Disconnect-MgGraph -ErrorAction SilentlyContinue
}
#endregion

<#
VALIDATION STEPS
----------------
1. Run against a lab/dev tenant first:
   .\script.ps1 -TenantId $labId -ClientId $clientId -WhatIf
2. Verify the correct tenant is targeted (check Verbose output for tenant ID).
3. Scope-limit the first real run (single user, single device) before full scope.
4. Confirm $LASTEXITCODE after execution.
5. Confirm Disconnect-MgGraph ran (no lingering session).
#>
```
