---
title: "Entra ID's memberOf Retirement (Nov 3) — Auditing What Just Went Dark"
date: 2026-11-05
draft: false
categories: ["post"]
tags: ["Microsoft Entra ID", "Dynamic Groups", "Identity", "IAM", "PowerShell", "Microsoft Graph", "Conditional Access"]
description: "Microsoft retired the memberOf rule operator in Entra ID on November 3, 2026. Dynamic groups, dynamic administrative units and entitlement auto-assignment policies built on it are now frozen. Here is how to find them, measure the drift, and fix them."
---

On November 3, 2026, Microsoft retired the memberOf rule operator in Microsoft Entra ID. The operator spent years in public preview with a clear "not for production" label, and plenty of tenants built access, licensing and Conditional Access on top of it anyway.

Two days later, the useful question is no longer what Microsoft announced. It is this: **what in my tenant is frozen right now, and who has the wrong access because of it?**

This post is the audit I would run. Inventory, blast radius, drift, then fix.

---

## What was retired

The operator let you build a dynamic rule from the membership of other groups instead of from user or device attributes:

```
user.memberof -any (group.objectId -in ['<group-id-1>', '<group-id-2>'])
```

Three kinds of objects can carry that rule, and all three are affected:

| Object | What happens after November 3 |
|---|---|
| Dynamic membership groups | Stop processing membership changes |
| Dynamic administrative units | Stop updating membership and administrative scope |
| Entitlement management auto-assignment policies | Stop updating assignments |

Nothing is deleted and nothing throws an error. These objects simply keep whatever membership they had on November 3.

---

## Why this one is dangerous: it fails quietly

A frozen group looks perfectly healthy. The consequences show up somewhere else:

- **Joiners** never enter the group, so they miss licenses, app assignments and Teams or SharePoint access.
- **Leavers and movers** never leave it, so they keep access they should have lost.
- **Conditional Access** policies that include or exclude these groups may stop receiving updated membership. Microsoft calls this out explicitly in its final reminders.
- **SCIM provisioning** from a frozen group goes quiet. The WorkOS write-up on this describes no new accounts for joiners and no role changes for movers, while disable and delete events still flow through.

One more detail from Microsoft's own documentation: memberOf groups never updated automatically when a child group was deleted or when members were removed from a child group. So some of your groups may have been drifting well before November 3.

---

## Why Microsoft pulled it

Microsoft's stated reason is scale and reliability. A single memberOf rule could slow dynamic membership processing for every dynamic group in the tenant, not just the one using it. The preview also came with hard limits that were never lifted:

- Maximum of 500 memberOf groups per tenant
- Could not be combined with any other condition in the rule
- Required Entra ID P1 or P2
- Public cloud only

It was never promoted to general availability, so it was retired instead.

---

## Timeline

| Date | What happened |
|---|---|
| August 5, 2026 | MC1448379 announces the retirement |
| October 5, 2026 | MC1488834 final reminder |
| October 28, 2026 | MC1529463 final reminder and migration guidance |
| November 3, 2026 | Retirement, worldwide |

---

## A note on the Graph memberOf endpoint

Some coverage says the Graph `memberOf` API call stops working too and breaks scripts. I could not confirm that. The Message Center text I read lists only the three services in the table above, and the same coverage says the migration path for scripts is the Graph API for direct and transitive membership. That only makes sense if `memberOf` and `transitiveMemberOf` keep working.

Step 4 below shows you how to check this in your own tenant instead of guessing.

---

## Step 1: Inventory every memberOf rule

This script is read-only. It finds dynamic groups, dynamic administrative units and entitlement assignment policies that reference memberOf, and exports each to CSV.

```powershell
Connect-MgGraph -Scopes "Group.Read.All","AdministrativeUnit.Read.All","EntitlementManagement.Read.All"

$out  = "C:\Temp\memberOf-audit"
New-Item -ItemType Directory -Path $out -Force | Out-Null
$guid = '[0-9a-fA-F]{8}-(?:[0-9a-fA-F]{4}-){3}[0-9a-fA-F]{12}'

function Get-GraphPaged([string]$Uri) {
    do {
        $page = Invoke-MgGraphRequest -Method GET -Uri $Uri
        $page.value
        $Uri  = $page.'@odata.nextLink'
    } while ($Uri)
}

# 1. Dynamic groups
$groups = Get-MgGroup -Filter "groupTypes/any(c:c eq 'DynamicMembership')" -All -Property "id,displayName,membershipRule,membershipRuleProcessingState,assignedLicenses"

$affectedGroups = $groups | Where-Object { $_.MembershipRule -match 'memberof' } | ForEach-Object {
    [PSCustomObject]@{
        GroupId         = $_.Id
        DisplayName     = $_.DisplayName
        ProcessingState = $_.MembershipRuleProcessingState
        Rule            = $_.MembershipRule
        SourceGroupIds  = (([regex]::Matches($_.MembershipRule, $guid).Value | Select-Object -Unique) -join ";")
        LicenseSkuIds   = ($_.AssignedLicenses.SkuId -join ";")
    }
}

# 2. Dynamic administrative units
$aus = Get-GraphPaged 'https://graph.microsoft.com/v1.0/administrativeUnits?$select=id,displayName,membershipType,membershipRule,membershipRuleProcessingState'

$affectedAUs = $aus | Where-Object { $_.membershipRule -match 'memberof' } | ForEach-Object {
    [PSCustomObject]@{
        AdminUnitId     = $_.id
        DisplayName     = $_.displayName
        ProcessingState = $_.membershipRuleProcessingState
        Rule            = $_.membershipRule
    }
}

# 3. Entitlement management assignment policies (auto-assignment rules live here)
$policies = Get-GraphPaged 'https://graph.microsoft.com/v1.0/identityGovernance/entitlementManagement/assignmentPolicies'

$affectedPolicies = $policies | Where-Object { ($_ | ConvertTo-Json -Depth 10) -match 'memberof' } | ForEach-Object {
    [PSCustomObject]@{
        PolicyId        = $_.id
        DisplayName     = $_.displayName
        AccessPackageId = $_.accessPackageId
    }
}

$affectedGroups   | Export-Csv "$out\dynamic-groups.csv"    -NoTypeInformation -Encoding UTF8
$affectedAUs      | Export-Csv "$out\dynamic-admin-units.csv" -NoTypeInformation -Encoding UTF8
$affectedPolicies | Export-Csv "$out\entitlement-policies.csv" -NoTypeInformation -Encoding UTF8

Write-Host "Dynamic groups using memberOf:   $(@($affectedGroups).Count)" -ForegroundColor Yellow
Write-Host "Dynamic admin units:             $(@($affectedAUs).Count)" -ForegroundColor Yellow
Write-Host "Entitlement policies:            $(@($affectedPolicies).Count)" -ForegroundColor Yellow
Write-Host "Reports saved to $out" -ForegroundColor Green
```

The `SourceGroupIds` column holds the groups each rule pointed to. You need it for Step 3.

---

## Step 2: Map the blast radius

A list of frozen groups is not useful until you know what each one controls. This script reads the CSV from Step 1 and, for every affected group, lists the Conditional Access policies that reference it, the enterprise apps it is assigned to, and the licenses it carries.

```powershell
Connect-MgGraph -Scopes "Group.Read.All","Policy.Read.All","Directory.Read.All"

$out            = "C:\Temp\memberOf-audit"
$affectedGroups = Import-Csv "$out\dynamic-groups.csv"
$caPolicies     = Get-MgIdentityConditionalAccessPolicy -All

$impact = foreach ($g in $affectedGroups) {
    $ca = $caPolicies | Where-Object {
        $_.Conditions.Users.IncludeGroups -contains $g.GroupId -or
        $_.Conditions.Users.ExcludeGroups -contains $g.GroupId
    }
    $apps = Get-MgGroupAppRoleAssignment -GroupId $g.GroupId -All

    [PSCustomObject]@{
        DisplayName        = $g.DisplayName
        GroupId            = $g.GroupId
        ConditionalAccess  = ($ca.DisplayName -join "; ")
        AssignedApps       = (($apps.ResourceDisplayName | Select-Object -Unique) -join "; ")
        GroupBasedLicenses = $g.LicenseSkuIds
    }
}

$impact | Export-Csv "$out\blast-radius.csv" -NoTypeInformation -Encoding UTF8
$impact | Format-Table DisplayName, ConditionalAccess, AssignedApps -AutoSize
```

License SKU IDs are GUIDs. Translate them with `Get-MgSubscribedSku | Select-Object SkuPartNumber, SkuId`.

---

## Step 3: Measure the drift

The memberOf operator built a group from the direct members of its source groups. That means you can calculate what the membership should be today and compare it with what the frozen group actually contains:

- **Missing** means the person should be in the group but is not. These are joiners who never got access.
- **Stale** means the person is in the group but should not be. These are leavers and movers who kept access.

```powershell
Connect-MgGraph -Scopes "Group.Read.All"

$out            = "C:\Temp\memberOf-audit"
$affectedGroups = Import-Csv "$out\dynamic-groups.csv"

function Get-DirectMemberIds([string]$GroupId) {
    Get-MgGroupMember -GroupId $GroupId -All |
        Where-Object { $_.AdditionalProperties['@odata.type'] -ne '#microsoft.graph.group' } |
        Select-Object -ExpandProperty Id
}

$summary = @()
$details = @()

foreach ($g in $affectedGroups) {

    # -notAny rules invert the logic: review those by hand
    if ($g.Rule -match '-notany') {
        $summary += [PSCustomObject]@{ DisplayName = $g.DisplayName; GroupId = $g.GroupId; Expected = ""; Actual = ""; Missing = "MANUAL REVIEW"; Stale = "" }
        continue
    }

    $expected = @($g.SourceGroupIds -split ";" | Where-Object { $_ } | ForEach-Object { Get-DirectMemberIds $_ } | Select-Object -Unique)
    $actual   = @(Get-DirectMemberIds $g.GroupId)
    $missing  = @($expected | Where-Object { $_ -notin $actual })
    $stale    = @($actual   | Where-Object { $_ -notin $expected })

    $summary += [PSCustomObject]@{
        DisplayName = $g.DisplayName
        GroupId     = $g.GroupId
        Expected    = $expected.Count
        Actual      = $actual.Count
        Missing     = $missing.Count
        Stale       = $stale.Count
    }

    foreach ($id in $missing) { $details += [PSCustomObject]@{ GroupId = $g.GroupId; ObjectId = $id; Issue = "Should be a member (not added)" } }
    foreach ($id in $stale)   { $details += [PSCustomObject]@{ GroupId = $g.GroupId; ObjectId = $id; Issue = "Should not be a member (still in group)" } }
}

$summary | Export-Csv "$out\drift-summary.csv" -NoTypeInformation -Encoding UTF8
$details | Export-Csv "$out\drift-details.csv" -NoTypeInformation -Encoding UTF8
$summary | Sort-Object Stale, Missing -Descending | Format-Table -AutoSize
```

Treat the numbers as a starting point, not a verdict. A group with 0 missing and 0 stale after two days may simply have had no joiners or leavers yet. It is still frozen.

---

## Step 4: Check who is calling the memberOf endpoint

If you send Microsoft Graph activity logs to a Log Analytics workspace, you can see which applications call the `memberOf` endpoint, and whether those calls started failing after November 3:

```kusto
MicrosoftGraphActivityLogs
| where TimeGenerated > ago(30d)
| where RequestUri has "memberOf"
| summarize Calls = count(),
            Failures = countif(ResponseStatusCode >= 400),
            LastCall = max(TimeGenerated)
    by AppId, RequestMethod
| order by Calls desc
```

A spike in `Failures` after the cutoff would settle the question from the note above. If the failure count stays flat, those integrations are fine. `has` matches whole terms, so `transitiveMemberOf` calls are not included.

---

## Step 5: Fix what you found

Work through the blast-radius CSV and decide per group. There are three options.

### Option A: Replace the rule with supported attributes

This is the cleanest fix when the source groups were themselves driven by attributes. Move that logic into the group's own rule. This is an illustrative example:

```
# Before
user.memberof -any (group.objectId -in ['<Sales-NL-id>', '<Sales-BE-id>'])

# After: the logic the source groups used, written directly
(user.department -eq "Sales") -and ((user.country -eq "NL") -or (user.country -eq "BE"))
```

Microsoft's own commentary and the community write-ups agree there is no one-to-one replacement for nested group logic. If the source groups were manually curated, this option does not apply.

### Option B: Convert to assigned membership

In the Entra admin center: **Groups > All groups > open the group > change Membership type from Dynamic to Assigned > Save.**

Converting does not fix drift. Whatever is wrong when you convert stays wrong until someone edits the group by hand. Use `drift-details.csv` to add the missing people and remove the stale ones as part of the conversion.

### Option C: Delete what nobody uses

If the blast-radius report shows no Conditional Access policies, no app assignments and no licenses, and the owner confirms the group is dead, remove it instead of migrating it.

Dynamic administrative units and entitlement policies follow the same three options. Record the owner, the decision and the reason in the ticket for each one.

---

## Where to start

| Priority | The group controls | Why it comes first |
|---|---|---|
| 1 | Conditional Access include/exclude | Stale membership is a policy gap |
| 1 | App assignment and SCIM provisioning | Leavers keep access, joiners get nothing |
| 2 | Dynamic administrative units | Delegated admin scope is out of date |
| 2 | Group-based licensing | Joiners unlicensed, leavers still consuming licenses |
| 3 | Teams, SharePoint, mail groups | Stale audiences, lower risk |

---

## After the fix

Re-run Step 1 until it returns zero rows in all three reports. For every group you converted or rewrote, re-run Step 3 against the new state and confirm the stale and missing counts are zero.

---

## Key takeaways

- The retirement does not delete or break anything visibly. It freezes membership, and the damage is stale access.
- Leavers keeping access is the real security exposure. Joiners missing access is the operational one.
- Always check what a group controls before deciding how to fix it. Conditional Access and app assignments come first.
- Converting to assigned membership locks in the current state, so reconcile the drift first.
- Confirm Graph endpoint behavior in your own logs instead of relying on secondary coverage.

---

## References

- [MC1448379: Replace MemberOf rules by November 3, 2026](https://mc.merill.net/message/MC1448379)
- [MC1488834: Final reminder to replace MemberOf rule operator configurations](https://mc.merill.net/message/MC1488834)
- [MC1529463: Final reminder and guidance](https://mc.merill.net/message/MC1529463)
- [BornCity: Entra ID memberOf shutdown stops group updates from November 3](https://borncity.com/news/microsoft-entra-id-memberof-abschaltung-stoppt-gruppenupdates-ab-3-november/)
- [LazyAdmin: Microsoft Entra ID is retiring the MemberOf rule for dynamic groups](https://lazyadmin.nl/office-365/microsoft-entra-id-is-retiring-the-memberof-rule-for-dynamic-groups/)
- [WorkOS: What the memberOf retirement means for SCIM provisioning](https://workos.com/blog/entra-memberof-retirement-scim-provisioning)
- [AdminDroid: Retirement of the memberOf rule operator](https://blog.admindroid.com/retirement-of-microsoft-entra-id-memberof-rule-operator/)
- [PDQ: Retirement of the MemberOf rule](https://www.pdq.com/blog/retirement-of-the-memberof-rule/)
- [Microsoft Learn: Dynamic membership rules for groups](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership)
