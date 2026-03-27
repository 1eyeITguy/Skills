# MSP and Consultant Guidelines

Apply these rules in addition to `code-standards.md` whenever the context is
multi-tenant, client-facing, or a reusable consultant deliverable.

---

## The Core Rule

**Never assume you are the only person, or the only tenant.**

A script that works perfectly in your lab against your tenant is worthless — or worse,
dangerous — if it silently targets the wrong client when handed off.  Every MSP and
consultant script must be safe to run by someone who has never seen the codebase before,
against a tenant they just connected to five minutes ago.

---

## Parameterise Everything Tenant-Specific

No tenant values may appear as literals anywhere in the script body.

```powershell
# Every one of these must be a parameter, never a hardcoded value:
$TenantId       # Entra tenant GUID
$TenantDomain   # e.g., contoso.com
$ClientId       # App registration client ID
$OUPath         # Distinguished name of any OU
$GroupName      # Any group referenced by name
$LicenseSku     # Any license SKU ID or friendly name
```

---

## Graceful Degradation for Missing Features

A client tenant may not have the same feature set as your dev tenant.
Scripts must check and degrade gracefully:

```powershell
# Check for Entra P2 before using P2-only features
$skus = Get-MgSubscribedSku
$hasP2 = $skus | Where-Object { $_.SkuPartNumber -like '*AAD_PREMIUM_P2*' }
if (-not $hasP2) {
    Write-Warning "Tenant $TenantId does not have Entra ID P2 — skipping PIM configuration."
    return
}

# Check for hybrid join before querying on-prem attributes
$domain = (Get-MgOrganization).VerifiedDomains |
          Where-Object { $_.IsDefault } |
          Select-Object -ExpandProperty Name
if (-not $domain) {
    Write-Warning 'Could not determine tenant domain — verify organisation settings.'
    return
}
```

---

## Explicit Authentication Per Tenant

Never assume a Graph session is open or that it targets the right tenant.
Always connect, always log the tenant ID, always disconnect in `finally`.

```powershell
function Connect-Tenant {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string]$TenantId,
        [Parameter(Mandatory)] [string]$ClientId
    )
    $connectParams = @{
        TenantId  = $TenantId
        ClientId  = $ClientId
        Scopes    = @('User.Read.All')   # Minimum required; adjust per task
        NoWelcome = $true
    }
    Connect-MgGraph @connectParams

    # Always confirm the connected tenant — log it visibly
    $context = Get-MgContext
    Write-Verbose "Connected to tenant: $($context.TenantId)"

    if ($context.TenantId -ne $TenantId) {
        throw "Connected tenant $($context.TenantId) does not match expected $TenantId"
    }
}
```

Minimum Graph scope principle: request only the scopes the script actually uses.
Document them in `.NOTES` under `Permissions:`.

---

## Least-Privilege App Registrations

- Use a dedicated app registration per client, or per script category (read-only vs
  write), not a single God-mode app registration for everything.
- Never store a client secret in a script — use certificate authentication or a
  secrets manager (Keeper, Azure Key Vault).
- Document required permissions in `.NOTES` so another engineer can recreate the
  app registration without reading the script logic.

---

## Client-Facing Log Quality

Log output may be read by the client, a different MSP engineer, or your future self
eighteen months from now.  Write accordingly:

```powershell
# WRONG — internal jargon, meaningless to a client
Write-Verbose "MgContext TenantId mismatch — throwing"

# RIGHT — human-readable, actionable
Write-Verbose "Tenant verification failed: connected to $($ctx.TenantId) but expected $TenantId — check your ClientId and consent."
Write-Warning "No changes were made to tenant $TenantId."
```

Rules:
- No abbreviations in log strings unless universally understood (UPN, GUID are fine;
  `ctx`, `obj`, `cfg` are not).
- Every `Write-Warning` should tell the reader what happened AND what to do next.
- Every destructive action should log what was changed, on what object, and when.

---

## Scope Limiting and Dry Run

For any script that touches more than one object, build in a scope-limit parameter
and always document it:

```powershell
[Parameter()]
[ValidateRange(1, 10000)]
[int]$Limit = 0   # 0 = no limit; set a low number for first test run

# In the main logic:
$targets = Get-MgUser -All
if ($Limit -gt 0) {
    Write-Verbose "Scope limited to first $Limit objects for this run."
    $targets = $targets | Select-Object -First $Limit
}
```

Recommended first-run pattern for client work:
1. `-WhatIf` run to see what *would* happen
2. `-Limit 1` run against a single test object
3. `-Limit 10` run against a small representative set
4. Full run after confirming results

---

## Reusability and Portability

For consultant deliverables intended to be handed off:

- Include a `README.md` alongside the script or module explaining:
  - What the script does in plain English
  - Required permissions and how to configure them
  - Required parameters and example invocations
  - Known limitations and tested environments
- Version the script in `.NOTES` and in the file name for major deliverables
  (`Invoke-TenantAudit_v1.0.ps1`)
- Avoid module dependencies that require special installation steps unless you
  document them explicitly — or bundle them
- Test in a clean PS7 session (no pre-loaded modules) to catch missing `#Requires`
  statements

---

## Multi-Tenant Batch Pattern

When running the same script across multiple tenants:

```powershell
# Drive from a CSV: TenantId, ClientId, TenantName
$tenants = Import-Csv -Path $TenantListPath

foreach ($tenant in $tenants) {
    Write-Verbose "Processing tenant: $($tenant.TenantName) ($($tenant.TenantId))"
    try {
        Connect-Tenant -TenantId $tenant.TenantId -ClientId $tenant.ClientId
        Invoke-TenantTask -TenantId $tenant.TenantId
    }
    catch {
        # Log the failure but continue to the next tenant
        Write-Warning "Failed on tenant $($tenant.TenantName): $_"
        [PSCustomObject]@{
            TenantName = $tenant.TenantName
            TenantId   = $tenant.TenantId
            Status     = 'Failed'
            Error      = $_.Exception.Message
        }
    }
    finally {
        Disconnect-MgGraph -ErrorAction SilentlyContinue
    }
}
```

Key rules for batch runs:
- A failure on one tenant must never abort the remaining tenants — catch and continue.
- Emit a result object per tenant so the caller gets a complete success/failure report.
- Log which tenant is being processed at the start of each iteration.
- Always disconnect in `finally` — a lingering session from tenant A affecting
  tenant B is a critical MSP failure mode.
