# Cloud Shell PowerShell — Set Application Insights ingestion sampling to 1%

### In Cloud Shell, use the Editor button to paste and save the script bellow as set-ai-sampling.ps1, then run:

``` powershell
.\set-ai-sampling.ps1
```


``` powershell
# Cloud Shell PowerShell — Set Application Insights ingestion sampling to 1%
# Paste everything at once, then press Enter.

# ===== CONFIG =====
$Percent = 1   # ingestion sampling percentage

# ===== QUIET WARNINGS (optional) =====
$env:SuppressAzurePowerShellBreakingChangeWarnings = '1'
$WarningPreference = 'SilentlyContinue'

function Update-AiSamplingForSubscription {
  param(
    [Parameter(Mandatory)][string]$SubscriptionId,
    [Parameter(Mandatory)][int]$Percent
  )

  Set-AzContext -Subscription $SubscriptionId | Out-Null

  $apps = Get-AzApplicationInsights -ErrorAction SilentlyContinue |
          Where-Object { $_.Name -and $_.ResourceGroupName -and $_.ResourceGroupName.Trim() -ne '' }

  if (-not $apps) {
    Write-Host "  (no Application Insights resources in this subscription)"
    return
  }

  Write-Host "-- Before:"
  $apps | Select-Object Name, ResourceGroupName, SamplingPercentage | Format-Table -AutoSize

  foreach ($app in $apps) {
    try {
      Update-AzApplicationInsights `
        -ResourceGroupName $app.ResourceGroupName `
        -Name $app.Name `
        -SamplingPercentage $Percent `
        -WarningAction SilentlyContinue | Out-Null
      Write-Host ("Updated: {0} in {1} -> {2}%" -f $app.Name, $app.ResourceGroupName, $Percent)
    } catch {
      Write-Warning ("Failed: {0} in {1} - {2}" -f $app.Name, $app.ResourceGroupName, $_.Exception.Message)
    }
  }

  Write-Host "-- After:"
  Get-AzApplicationInsights -ErrorAction SilentlyContinue |
    Where-Object { $_.Name -and $_.ResourceGroupName -and $_.ResourceGroupName.Trim() -ne '' } |
    Select-Object Name, ResourceGroupName, SamplingPercentage |
    Format-Table -AutoSize
}

# ===== BUILD SUBSCRIPTION MENU (across all tenants) =====
$allSubs = foreach ($t in (Get-AzTenant)) {
  Get-AzSubscription -TenantId $t.Id -ErrorAction SilentlyContinue |
    Select-Object @{n='TenantDomain';e={$t.Domain}}, Name, Id, TenantId
}

if (-not $allSubs) { Write-Error "No subscriptions available."; return }

$menu = @{}
$idx = 1
$allSubs | Sort-Object TenantDomain, Name | ForEach-Object {
  $menu[$idx] = $_
  Write-Host ("[{0}] {1}  ({2})  Tenant: {3}" -f $idx, $_.Name, $_.Id, $_.TenantDomain)
  $idx++
}

# ===== PICK & RUN =====
$sel = Read-Host "Enter number(s) to target (e.g. 1,3) or * for ALL"
if ($sel -eq '*') {
  $targets = $allSubs
} else {
  $nums = $sel -split '[, ]+' | ForEach-Object { $_.Trim() } | Where-Object { $_ -match '^\d+$' }
  if (-not $nums) { Write-Error "No valid selection."; return }
  $targets = foreach ($n in $nums) {
    $n = [int]$n
    if ($menu.ContainsKey($n)) { $menu[$n] }
  }
}

if (-not $targets) { Write-Error "Nothing selected."; return }

foreach ($s in $targets) {
  Set-AzContext -SubscriptionId $s.Id -Tenant $s.TenantId | Out-Null
  Write-Host "`n== Subscription: $($s.Name) (Tenant: $($s.TenantDomain)) =="
  Update-AiSamplingForSubscription -SubscriptionId $s.Id -Percent $Percent
}

```
