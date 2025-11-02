# SetIngestionSampling Automation

### Use the following code to create a runbook. It run on Powershell 5.1

``` Powershell
<#
.SYNOPSIS
  Azure Automation Runbook — Set Application Insights ingestion sampling to N% (default 1%)
  across ALL accessible tenants/subscriptions of the Automation Account’s Managed Identity.

.DESCRIPTION
  - Non-interactive (no Read-Host). Optionally filter with -IncludeSubscriptions / -ExcludeSubscriptions.
  - Honors -WhatIf to show intended changes without applying them.
  - Emits Before/After tables per subscription for easy job output viewing.

.REQUIREMENTS
  - Runbook type: PowerShell 7.2 (recommended) or 5.1
  - Managed Identity on the Automation Account with access to target scopes
    (e.g., Subscription Contributor or "Application Insights Component Contributor")
  - Imported Az modules (recent versions): Az.Accounts, Az.Resources, Az.ApplicationInsights
#>

<#
Fix for the runbook error:
  Variable reference is not valid. ':' was not followed by a valid variable name character.

Cause: a string contained "$SubscriptionId: $_" — the ":" right after an expanded $variable
       makes PowerShell think you're using a scoped variable. Use -f formatting or $().
Below is the corrected runbook (same logic as before), with the offending line fixed.
#>

#Requires -Modules Az.Accounts, Az.Resources, Az.ApplicationInsights

param(
  [ValidateRange(0,100)][int]$Percent = 1,
  [string[]]$IncludeSubscriptions,
  [string[]]$ExcludeSubscriptions,
  [switch]$WhatIf,
  [switch]$QuietWarnings
)

if ($QuietWarnings) {
  $env:SuppressAzurePowerShellBreakingChangeWarnings = '1'
  $WarningPreference = 'SilentlyContinue'
}
$InformationPreference = 'Continue'

try {
  Connect-AzAccount -Identity -ErrorAction Stop | Out-Null
} catch {
  Write-Error "Failed to authenticate with Managed Identity. $_"
  throw
}

function Test-SubMatch {
  param([Parameter(Mandatory)][object]$Sub, [string[]]$List)
  if (-not $List -or $List.Count -eq 0) { return $false }
  foreach ($it in $List) {
    if ($Sub.Id -ieq $it -or $Sub.Name -ieq $it) { return $true }
  }
  return $false
}

function Get-AiAppsSafe {
  Get-AzApplicationInsights -ErrorAction SilentlyContinue |
    Where-Object { $_.Name -and $_.ResourceGroupName -and $_.ResourceGroupName.Trim() -ne '' }
}

function Format-AiTable { param([array]$Apps)
  if ($Apps) { $Apps | Select-Object Name, ResourceGroupName, SamplingPercentage | Format-Table -AutoSize }
  else { Write-Information "  (no Application Insights resources in this subscription)" }
}

function Write-SubHeader { param([string]$Name, [string]$Tenant)
  Write-Output ""
  Write-Output ("== Subscription: {0}  (Tenant: {1}) ==" -f $Name, $Tenant)
}

function Get-AllSubsAcrossTenants {
  $out = @()
  $tenants = Get-AzTenant -ErrorAction SilentlyContinue
  foreach ($t in $tenants) {
    $tenantDomain = $t.DefaultDomain
    try {
      foreach ($s in (Get-AzSubscription -TenantId $t.Id -ErrorAction Stop)) {
        $out += [pscustomobject]@{
          Name         = $s.Name
          Id           = $s.Id
          TenantId     = $s.TenantId
          TenantDomain = $tenantDomain
        }
      }
    } catch {
      Write-Warning ("Failed to enumerate subscriptions for tenant {0} ({1}): {2}" -f $tenantDomain, $t.Id, $_)
    }
  }
  $out
}

function Update-AiSamplingForSubscription {
  [CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Low')]
  param(
    [Parameter(Mandatory)][string]$SubscriptionId,
    [Parameter(Mandatory)][int]$Percent
  )

  try {
    Set-AzContext -Subscription $SubscriptionId -ErrorAction Stop | Out-Null
  } catch {
    # *** FIXED LINE (use -f formatting to avoid "$SubscriptionId:" parsing) ***
    Write-Warning ("  Cannot set context to subscription {0}: {1}" -f $SubscriptionId, $_)
    return
  }

  $apps = Get-AiAppsSafe
  Write-Output "-- Before:"
  Format-AiTable -Apps $apps

  foreach ($app in $apps) {
    $target = "$($app.Name) in RG '$($app.ResourceGroupName)'"
    if ($PSCmdlet.ShouldProcess($target, "Set SamplingPercentage -> $Percent%")) {
      try {
        Update-AzApplicationInsights -ResourceGroupName $app.ResourceGroupName -Name $app.Name `
          -SamplingPercentage $Percent -WarningAction SilentlyContinue -ErrorAction Stop | Out-Null
        Write-Output ("  Updated: {0} -> {1}%" -f $target, $Percent)
      } catch {
        Write-Warning ("  Failed: {0} - {1}" -f $target, $_.Exception.Message)
      }
    } else {
      Write-Output ("  Would update: {0} -> {1}%" -f $target, $Percent)
    }
  }

  $after = Get-AiAppsSafe
  Write-Output "-- After:"
  Format-AiTable -Apps $after
}

$allSubs = Get-AllSubsAcrossTenants
if (-not $allSubs) { Write-Error "No subscriptions available to this identity."; return }

$targets = $allSubs
if ($IncludeSubscriptions) {
  $targets = $targets | Where-Object { Test-SubMatch -Sub $_ -List $IncludeSubscriptions }
  if (-not $targets) { Write-Warning "No subscriptions matched -IncludeSubscriptions."; return }
}
if ($ExcludeSubscriptions) {
  $targets = $targets | Where-Object { -not (Test-SubMatch -Sub $_ -List $ExcludeSubscriptions) }
  if (-not $targets) { Write-Warning "All targeted subscriptions were excluded by -ExcludeSubscriptions."; return }
}

Write-Output "Target sampling percentage: $Percent%"
if ($WhatIf) { Write-Output "Running in WHATIF mode (no changes will be made)." }

foreach ($s in ($targets | Sort-Object TenantDomain, Name)) {
  try { Set-AzContext -Tenant $s.TenantId -ErrorAction Stop | Out-Null } catch { }
  Write-SubHeader -Name $s.Name -Tenant $s.TenantDomain
  if ($WhatIf) { Update-AiSamplingForSubscription -SubscriptionId $s.Id -Percent $Percent -WhatIf }
  else { Update-AiSamplingForSubscription -SubscriptionId $s.Id -Percent $Percent }
}

Write-Output "`nDone."


```

### Use the following powershell script to test this runbook if necessary.

``` powershell
# Cloud Shell — Start your NEW PS5.1 runbook and stream output live (null-safe)

# ===== RUNBOOK CONFIG =====
$ResourceGroup     = 'Operation-Automation-Group-Dev'
$AutomationAccount = 'Operation-Automation-Dev'
$RunbookName       = 'SetApplicationInsightsIngestionSamplingPS51'  # <-- new PS5.1 runbook

# Optional parameters to pass to the runbook
$rbParams = @{
  Percent = 1
  # IncludeSubscriptions = @('1b28a41b-9fdf-4643-b48c-daf64f1c8119')  # names or IDs
  # ExcludeSubscriptions = @()
  # WhatIf = $true
  # QuietWarnings = $true
}

# ===== PREP =====
Import-Module Az.Accounts -ErrorAction SilentlyContinue
Import-Module Az.Automation -ErrorAction SilentlyContinue
try { Get-AzContext | Out-Null } catch { Connect-AzAccount | Out-Null }

$ProgressPreference = 'SilentlyContinue'
$InformationPreference = 'Continue'

# ===== LOGGING =====
$ts      = Get-Date -Format 'yyyyMMdd-HHmmss'
$logPath = Join-Path $HOME ("automation-{0}-{1}.log" -f $RunbookName,$ts)
$errLog  = [System.IO.Path]::ChangeExtension($logPath,'errors.log')
function _stamp { "[{0:HH:mm:ss}]" -f (Get-Date) }
function Write-Log { param([string]$Text,[switch]$Err)
  $line = "{0} {1}" -f (_stamp), $Text
  if ($Err) { Write-Host $line -ForegroundColor Red; Add-Content $errLog $line }
  else      { Write-Host $line;                     Add-Content $logPath $line }
}

function Format-OutputRecord {
  param($rec)
  $rt  = $rec.RecordType
  $msg = $rec.Summary
  if (-not $msg) { $msg = $rec.Value }
  if (-not $msg -and $rec.Exception) { $msg = $rec.Exception.Message }
  if (-not $msg) { $msg = '(no message payload)' }
  "[$rt] $msg"
}

# ===== START RUNBOOK =====
Write-Log ("Starting runbook '{0}' in '{1}' (RG: {2})" -f $RunbookName,$AutomationAccount,$ResourceGroup)
$job = Start-AzAutomationRunbook -ResourceGroupName $ResourceGroup `
        -AutomationAccountName $AutomationAccount -Name $RunbookName `
        -Parameters $rbParams -ErrorAction Stop
Write-Log ("JobId: {0}" -f $job.JobId)
Write-Log ("Streaming to: {0}" -f $logPath)

# ===== STREAM OUTPUT (null-safe, deduped) =====
$seen = New-Object 'System.Collections.Generic.HashSet[string]'
function Add-SeenKey { param([string]$Key) if ([string]::IsNullOrEmpty($Key)) { return $false } ; $seen.Add($Key) }

$done = @('Completed','Failed','Suspended','Stopped')
do {
  $streams = Get-AzAutomationJobOutput -ResourceGroupName $ResourceGroup `
              -AutomationAccountName $AutomationAccount -Id $job.JobId -Stream Any `
              -ErrorAction SilentlyContinue

  foreach ($s in $streams) {
    if (-not $s) { continue }
    $key = if ($s.Id) { [string]$s.Id } else {
      $typ = if ($s.StreamType) { $s.StreamType } else { $s.Type }
      $sum = if ($s.Summary)    { $s.Summary }    else { '' }
      "{0}|{1}|{2}" -f $typ,$sum,(Get-Date).Ticks
    }
    if (-not (Add-SeenKey $key)) { continue }

    if ($s.Id) {
      $rec = Get-AzAutomationJobOutputRecord -ResourceGroupName $ResourceGroup `
               -AutomationAccountName $AutomationAccount -JobId $job.JobId -Id $s.Id `
               -ErrorAction SilentlyContinue
      if ($rec) {
        $line = Format-OutputRecord $rec
        if ($rec.RecordType -match 'Error|Exception') { Write-Log $line -Err } else { Write-Log $line }
      }
    } else {
      $rt  = if ($s.StreamType) { $s.StreamType } else { $s.Type }
      $msg = if ($s.Summary)    { $s.Summary }    else { '(no message payload)' }
      if ($rt -match 'Error|Exception') { Write-Log ("[{0}] {1}" -f $rt,$msg) -Err }
      else                              { Write-Log ("[{0}] {1}" -f $rt,$msg) }
    }
  }

  Start-Sleep -Seconds 2
  $job = Get-AzAutomationJob -ResourceGroupName $ResourceGroup `
           -AutomationAccountName $AutomationAccount -Id $job.JobId
} until ($done -contains $job.Status)

Write-Log ("Job finished with status: {0}" -f $job.Status)
if ($job.Status -eq 'Failed') {
  Write-Log "Collecting ERROR stream..." -Err
  $err = Get-AzAutomationJobOutput -ResourceGroupName $ResourceGroup `
           -AutomationAccountName $AutomationAccount -Id $job.JobId -Stream Error `
           -ErrorAction SilentlyContinue
  foreach ($e in $err) {
    if ($e.Id) {
      $r = Get-AzAutomationJobOutputRecord -ResourceGroupName $ResourceGroup `
             -AutomationAccountName $AutomationAccount -JobId $job.JobId -Id $e.Id `
             -ErrorAction SilentlyContinue
      if ($r) { Write-Log (Format-OutputRecord $r) -Err }
    } else {
      $rt  = if ($e.StreamType) { $e.StreamType } else { $e.Type }
      $msg = if ($e.Summary)    { $e.Summary }    else { '(no message payload)' }
      Write-Log ("[{0}] {1}" -f $rt,$msg) -Err
    }
  }
}

Write-Host ""
Write-Host ("Full log:  {0}" -f $logPath)
Write-Host ("Error log: {0}" -f $errLog)

```
