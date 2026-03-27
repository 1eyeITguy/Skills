# Validation and Testing Guidance

A script that compiles is not a script that works.  Before declaring output done,
think through how it will be verified.  Provide validation guidance alongside the code.

---

## Before You Write — Stop and Think

Answer all of these before opening a code block.  If you cannot answer one, say so
and ask — do not guess.

**Execution context**
- What is actually running this script?
- What account context — SYSTEM, service account, logged-in user?
- Is this a one-shot interactive run or something that will run repeatedly / automated?

**Environment**
- Is the tenant hybrid, cloud-only, or on-prem-only?
- What modules are guaranteed to be present in this execution context?
- Are there proxy, firewall, or network restrictions affecting module downloads
  or Graph API calls?
- For WinPE: has PS7 been explicitly injected, or is this a vanilla 5.1 WinPE boot?

**Permissions and dependencies**
- What Graph API scopes does this script actually need?  Minimum set only.
- What AD permissions does the running account need?
- Are there licenses (P1, P2, etc.) required on the tenant?
- Does this script depend on another script, module, or scheduled task?

**Scope and blast radius**
- Is this targeting one object, a filtered set, or everything?  Be explicit.
- If something goes wrong mid-run, what is the blast radius?
- Does this need a rollback path or a `-WhatIf` / `-DryRun` mode?

---

## Dry-Run First

Any script with `SupportsShouldProcess` should always be run with `-WhatIf` first:

```powershell
.\Invoke-UserCleanup.ps1 -TenantId $tid -ClientId $cid -WhatIf
```

If the script is destructive but does not support `-WhatIf`, add a `-ReportOnly`
or `-DryRun` switch that logs what *would* happen without making changes:

```powershell
[Parameter()] [switch]$DryRun

if ($DryRun) {
    Write-Verbose "[DryRun] Would remove user: $($user.UserPrincipalName)"
    return
}
# Real action here
Remove-MgUser -UserId $user.Id
```

---

## Verbose Output Check

Run with `-Verbose` to confirm the operational path is what you expect:

```powershell
.\Set-DeviceCompliance.ps1 -Verbose
```

Every major decision point, loop iteration, and branch should emit a `Write-Verbose`
line.  If `-Verbose` output is sparse or silent, the script is under-instrumented.

Signs of good `-Verbose` coverage:
- Start and end of every function
- Each object being processed in a loop
- Each branch taken in an `if/else`
- Each external call (Graph, AD, CIM)
- Tenant ID confirmed after Graph connection

---

## Exit Code Spot-Check (Intune / OSD)

For Intune detection scripts, test both paths explicitly:

```powershell
# Simulate compliant state → run detection → confirm exit 0
.\Detect-Setting.ps1
$LASTEXITCODE   # must be 0

# Simulate non-compliant state → run detection → confirm exit 1
# (modify the condition, then run again)
$LASTEXITCODE   # must be 1
```

For remediation scripts, confirm idempotency — running it twice must produce
the same result and exit `0` on the second run.

---

## Intune Detection / Remediation Pair Logic

Before deploying a detection + remediation pair to Intune, verify consistency:

- The detection script's **non-compliant** exit path must detect exactly what
  the remediation script **fixes**.
- After the remediation succeeds, the detection script must return `0`.
- A failed remediation must return `1` — never swallow an error and return `0`.

```
Verification sequence:
1. Run detection on non-compliant machine   → exit 1  ✓
2. Run remediation                          → exit 0  ✓
3. Run detection again                      → exit 0  ✓  (logic is consistent)
4. Run remediation again                    → exit 0  ✓  (idempotency confirmed)
```

Common consistency failures to check:
- Detection checks registry key X; remediation sets registry key Y (different key)
- Detection checks a value with `-eq`; remediation sets it with different casing
- Detection script catches errors and returns `0` on failure (wrong)

---

## MSP / Multi-Tenant Script Validation

Before running against a real client tenant:

```powershell
# Step 1 — WhatIf run against lab tenant
.\Invoke-TenantTask.ps1 -TenantId $labTenantId -ClientId $clientId -WhatIf

# Step 2 — Confirm correct tenant is targeted
#   Check Verbose output for: "Connected to tenant: <guid>"
#   Confirm the GUID matches the expected tenant

# Step 3 — Scope-limited real run against lab
.\Invoke-TenantTask.ps1 -TenantId $labTenantId -ClientId $clientId -Limit 1

# Step 4 — Full lab run, review output objects
.\Invoke-TenantTask.ps1 -TenantId $labTenantId -ClientId $clientId

# Step 5 — Only now run against the real client tenant, still with -Limit 1 first
.\Invoke-TenantTask.ps1 -TenantId $clientTenantId -ClientId $clientId -Limit 1
```

---

## Module Version Verification

Before deploying a script that uses a rapidly-changing module:

```powershell
# Check what version is installed in the execution environment
Get-Module -ListAvailable Microsoft.Graph.Authentication | Select-Object Version
Get-Module -ListAvailable OSD                            | Select-Object Version

# For Intune/OSD scripts: verify the same module version is available
# in the WinPE or Intune execution context, not just your workstation
```

---

## Validation Steps Comment Block

Always append this to the bottom of every generated script so the engineer
running it knows how to verify it — especially important for handoff:

```powershell
<#
VALIDATION STEPS
----------------
1. Run with -WhatIf first — confirm the listed actions are correct.
2. Run with -Verbose — confirm each decision point logs as expected.
3. Run against a single test object before targeting full scope.
4. Confirm exit code:  $LASTEXITCODE  (should be 0 on success)
5. [Intune pairs]   Re-run detection after remediation — must exit 0.
6. [MSP scripts]    Verify tenant ID in Verbose output before full run.
7. [Destructive ops] Confirm rollback path or document that none exists.
#>
```

Tailor items 5–7 to the specific script context — include only the ones that apply.
