# cspm-azure-sizer
CloudGuard Azure Sizing queries

These queries can be used in the Azure Resource Graph explorer to find resources across accounts.

## Compute
```kql
resources
| where type == "microsoft.compute/virtualmachines" or type =~ "microsoft.sql/servers"
| extend prop = parse_json(properties)

| where (isnotempty(prop.hardwareProfile.vmSize) and prop.hardwareProfile.vmSize !contains "A0" and prop.extended.instanceView.powerState.displayStatus =~ "VM running") or (isnotempty(prop.hardwareProfile.vmSize) and prop.hardwareProfile.vmSize !contains "A0" and prop.extended.instanceView.powerState.displayStatus =~ "VM running")
or isempty(prop.hardwareProfile.vmSize)
| count as BillableAssetsCompute
```

## Kubernetes Nodes

```kql
resources
| where type =~ 'microsoft.containerservice/managedclusters'
| extend agentPoolProfiles = properties['agentPoolProfiles']
| project subscriptionId, clusterName = name, agentPoolProfiles, organizationUnit = tostring(tags['OrganizationUnit']) // Convert OU info to string
| mv-expand agentPoolProfiles
| extend vmSize = agentPoolProfiles['vmSize'], nodeCount = toint(agentPoolProfiles['count']) * 3
| project subscriptionId, clusterName, organizationUnit, vmSize, nodeCount
| summarize TotalNodeCount = sum(nodeCount) by subscriptionId, clusterName, tostring(organizationUnit) // Ensure organizationUnit is cast to string here as well
| order by subscriptionId, clusterName, organizationUnit
```

## Azure Functions

```kql
resources
| where type == "microsoft.web/sites" and kind startswith "functionapp"
| summarize count_name = count(name) by subscriptionId
| project SubscriptionId = subscriptionId, BillableAssetsFunctions = ceiling(count_name / 60.0)
```
