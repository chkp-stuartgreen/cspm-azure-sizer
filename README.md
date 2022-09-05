# cspm-azure-sizer
CloudGuard Azure Sizing queries

These queries can be used in the Azure Resource Graph explorer to find resources across accounts.

## Compute
```kql
resources
| where type == "microsoft.compute/virtualmachines" or type =~ "microsoft.sql/servers" |
| ------------------------------------------------------------------------------------ |

| extend prop = parse_json(properties)

| where (isnotempty(prop.hardwareProfile.vmSize) and prop.hardwareProfile.vmSize !contains "A0" and prop.extended.instanceView.powerState.displayStatus =~ "VM running") or (isnotempty(prop.hardwareProfile.vmSize) and prop.hardwareProfile.vmSize !contains "A0" and prop.extended.instanceView.powerState.displayStatus =~ "VM running")
or isempty(prop.hardwareProfile.vmSize)
| count as BillableAssetsCompute
```

## Kubernetes Nodes

```kql
resources
| where type == "microsoft.containerservice/managedclusters"
| extend properties.agentPoolProfiles
| project subscriptionId, name, pool = (properties.agentPoolProfiles)
| mv-expand pool
| project subscription = subscriptionId, cluster = name, size = pool.vmSize, nodecount = toint(pool.['count'] * 3)
| summarize sum(nodecount)
| project BillableAssetsNodes = sum_nodecount
```

## Azure Functions

```kql
resources
| where type == "microsoft.web/sites" and kind startswith "functionapp"
| summarize count(name)
| project BillableAssetsFunctions = ceiling(count_name / 60.0)
```
