# Monitor AKS using KQL
Some baseline KQL queries to allow you monitor your AKS clusters.

You can use Azure montoring to create tickets for these alerts.

In KQL you use // to comment lines.

## Monitor AKS Pods running at the CPU utilization threshold you defined for 15 straight or longer:

```
let startDateTime = ago(15m);
let _ResourceLimitCounterName = 'cpuLimitNanoCores';
let _ResourceUsageCounterName = 'cpuUsageNanoCores';
let _BinSize = 1m;
KubePodInventory
| where Namespace != 'default' and Namespace != 'kube-system' // add or remove namespaces that you know can cause extra noise for your alerts
| where ClusterName in //add your cluster name in this format ("cluster name", "cluster name")
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| extend InstanceName = strcat(ClusterId, '/', ContainerName),
ContainerName = strcat(ControllerName, '/', tostring(split(ContainerName, '/')[1]))
| distinct Computer, InstanceName, ClusterName
| join kind=inner hint.strategy=shuffle (
Perf
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| where ObjectName == 'K8SContainer' and CounterName == _ResourceLimitCounterName
| summarize MaxLimitValue = max(CounterValue) by Computer, InstanceName, bin(TimeGenerated, _BinSize)
| project
Computer,
InstanceName,
MaxLimitValue
)
on Computer, InstanceName
| join kind=inner hint.strategy=shuffle (
Perf
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| where ObjectName == 'K8SContainer' and CounterName == _ResourceUsageCounterName
| project Computer, InstanceName, UsageValue = CounterValue, TimeGenerated
)
on Computer, InstanceName
| join kind=leftouter (
KubePodInventory
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| where Namespace != 'default' and Namespace != 'kube-system' // add or remove namespaces that you know can cause extra noise for your alerts
| where ClusterName in //add your cluster name in this format ("cluster name", "cluster name")
| distinct Computer, Name, ClusterName, PodIp
)
on Computer
| project
Name,
InstanceName,
ClusterName,
Computer,
TimeGenerated,
PodIp,
CPUValue = UsageValue * 100.0 / MaxLimitValue
| summarize MaxValue = max(CPUValue) by bin(TimeGenerated, _BinSize), Computer, ClusterName, Name, PodIp
| where MaxValue between (95 .. 99) and (now() - TimeGenerated) >= 15m // use the operator for what you are trying to get for example <= or >= and set the threshold that works for you 
| where (now() - TimeGenerated) >= 15m

```

## Monitor AKS Pods running at the Memory utilization threshold you defined for 15 straight or longer:

```
let endDateTime = now();
let startDateTime = ago(15m);
let _ResourceLimitCounterName = 'memoryLimitBytes';
let _ResourceUsageCounterName = 'memoryRssBytes';
let _BinSize = 1m;
KubePodInventory
| where Namespace != 'default' and Namespace != 'kube-system' // add or remove namespaces that you know can cause extra noise for your alerts
| where ClusterName in //add your cluster name in this format ("cluster name", "cluster name")
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| extend InstanceName = strcat(ClusterId, '/', ContainerName),
ContainerName = strcat(ControllerName, '/', tostring(split(ContainerName, '/')[1]))
| distinct Computer, InstanceName, ClusterName
| join kind=inner hint.strategy=shuffle (
Perf
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| where ObjectName == 'K8SContainer' and CounterName == _ResourceLimitCounterName
| summarize MaxLimitValue = max(CounterValue) by Computer, InstanceName, bin(TimeGenerated, _BinSize)
| project
Computer,
InstanceName,
MaxLimitValue
)
on Computer, InstanceName
| join kind=inner hint.strategy=shuffle (
Perf
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| where ObjectName == 'K8SContainer' and CounterName == _ResourceUsageCounterName
| project Computer, InstanceName, UsageValue = CounterValue, TimeGenerated
)
on Computer, InstanceName
| join kind=leftouter (
KubePodInventory
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| where Namespace != 'default' and Namespace != 'kube-system' // add or remove namespaces that you know can cause extra noise for your alerts
| where ClusterName in //add your cluster name in this format ("cluster name", "cluster name")
| distinct Computer, Name, ClusterName, PodIp
)
on Computer
| project
Name,
InstanceName,
ClusterName,
Computer,
TimeGenerated,
PodIp,
MemoryValue = UsageValue * 100.0 / MaxLimitValue
| summarize MaxValue = max(MemoryValue) by bin(TimeGenerated, _BinSize), Computer, ClusterName, Name, PodIp
| where MaxValue >= 95 and (now() - TimeGenerated) >= 15m // use the operator for what you are trying to get for example <= or >= and set the threshold that works for you 
| where (now() - TimeGenerated) >= 15m
```

## Node disk utilization 

```
InsightsMetrics
| where Namespace == 'container.azm.ms/disk' and Name == 'used_percent' 
| where Val between  (85 .. 95 ) // use the operator for what you are trying to get for example <= or >= and set the threshold that works for you 
| where parse_json(Tags).["container.azm.ms/clusterName"] == "Your Cluster" or parse_json(Tags).["container.azm.ms/clusterName"] == "Your Cluster"
| summarize Val = arg_max(TimeGenerated, Val) by Computer, _ResourceId
```

## Catch Kubernetes Failed operations 

For this alert I would suggest to first query the KubeEvents table on its own to then filter down what matters to you using the query example.
```

let period = 1h;
let clusters = dynamic//add your cluster name in this format (["cluster name", "cluster name"]);
KubeEvents
| where KubeEventType == "Error" or KubeEventType == "Warning"
| where Namespace != 'default' and Namespace != 'kube-system' // add or remove namespaces that you know can cause extra noise for your alerts
| where ClusterName in (clusters) and Count >= 10 // set a count number good for you
| where ObjectKind <> "HorizontalPodAutoscaler" and Reason <> "FreezeScheduled" and Reason <> "FailedToUpdateEndpoint" // capture the ObjectKind of your intestest
| where TimeGenerated >= ago(period)
| project TimeGenerated, Name, Reason, Message, ObjectKind, Count, ClusterName, Namespace

```
## AKS node is not ready 

```
let period = 1h;
let clusters = dynamic//add your cluster name in this format (["cluster name", "cluster name"]);
KubeNodeInventory
| where ClusterName in (clusters)
| where Status !contains "Ready" and Status !contains "VMEventScheduled"
| where isnotempty(Status)
| where TimeGenerated >= ago(period)
| project ClusterName, Computer, Status, TimeGenerated

```

## Node could not be added due to a vCPU quota problem

```
let period = 1h;
let clusters = dynamic//add your cluster name in this format (["cluster name", "cluster name"]);
AzureActivity
| where TimeGenerated >= ago(period)
| where OperationNameValue == "Microsoft.ContainerService/managedClusters/write"
| where OperationName has "Create" or OperationName has "Scale"
| where parse_json(tostring(parse_json(Properties).statusMessage)).code == "QuotaExceeded"
| project Resource
```
