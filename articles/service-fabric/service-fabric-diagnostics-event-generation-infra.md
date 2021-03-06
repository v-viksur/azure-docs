---
title: Azure Service Fabric Infrastructure Level Monitoring | Microsoft Docs
description: Learn about infrastructure level events and logs used to monitor and diagnose Azure Service Fabric clusters.
services: service-fabric
documentationcenter: .net
author: dkkapur
manager: timlt
editor: ''

ms.assetid:
ms.service: service-fabric
ms.devlang: dotnet
ms.topic: article
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 05/26/2017
ms.author: dekapur

---

# Infrastructure level event and log generation

## Monitoring the cluster

It is important to monitor at the infrastructural level to determine whether or not your hardware and cluster are behaving as expected. Though Service Fabric can keep applications running during a hardware failure, but you still need to diagnose whether an error is occurring in an application or in the underlying infrastructure. You also should monitor your clusters to  better plan for capacity, helping in decisions about adding or removing infrastructure.

Service Fabric provides five different log channels out-of-the-box that generate the following events:

* Operational channel: high-level operations performed by Service Fabric and the cluster, including events for a node coming up, a new application being deployed, or a SF upgrade rollback, etc.
* Customer information channel: health reports and load balancing decisions
* [Reliable Services events](service-fabric-reliable-services-diagnostics.md): programming model specific events
* [Reliable Actors events](service-fabric-reliable-actors-diagnostics.md): programming model specific events and performance counters
* Support logs: system logs generated by Service Fabric only to be used by us when providing support

These various channels cover most of the infrastructural level logging that is recommended. To improve infrastructural level logging, consider investing in better understanding the health model and adding custom health reports, and add custom **Performance Counters** to build a real-time understanding of the impact of your services and applications on the cluster.

### Azure Service Fabric health and load reporting

Service Fabric has its own health model, which is described in detail in these articles:
- [Introduction to Service Fabric health monitoring](service-fabric-health-introduction.md)
- [Report and check service health](service-fabric-diagnostics-how-to-report-and-check-service-health.md)
- [Add custom Service Fabric health reports](service-fabric-report-health.md)
- [View Service Fabric health reports](service-fabric-view-entities-aggregated-health.md)

Health monitoring is critical to multiple aspects of operating a service. Health monitoring is especially important when Service Fabric performs a named application upgrade. After each upgrade domain of the service is upgraded and is available to your customers, the upgrade domain must pass health checks before the deployment moves to the next upgrade domain. If good health status cannot be achieved, the deployment is rolled back, so that the application is in a known, good state. Although some customers might be affected before the services are rolled back, most customers won't experience an issue. Also, a resolution occurs relatively quickly, and without having to wait for action from a human operator. The more health checks that are incorporated into your code, the more resilient your service is to deployment issues.

Another aspect of service health is reporting metrics from the service. Metrics are important in Service Fabric because they are used to balance resource usage. Metrics also can be an indicator of system health. For example, you might have an application that has many services, and each instance reports a requests per second (RPS) metric. If one service is using more resources than another service, Service Fabric moves service instances around the cluster, to try to maintain even resource utilization. For a more detailed explanation of how resource utilization works, see [Manage resource consumption and load in Service Fabric with metrics](service-fabric-cluster-resource-manager-metrics.md).

Metrics also can help give you insight into how your service is performing. Over time, you can use metrics to check that the service is operating within expected parameters. For example, if trends show that at 9 AM on Monday morning the average RPS is 1,000, then you might set up a health report that alerts you if the RPS is below 500 or above 1,500. Everything might be perfectly fine, but it might be worth a look to be sure that your customers are having a great experience. Your service can define a set of metrics that can be reported for health check purposes, but that don't affect the resource balancing of the cluster. To do this, set the metric weight to zero. We recommend that you start all metrics with a weight of zero, and not increase the weight  until you are sure that you understand how weighting the metrics affects resource balancing for your cluster.

> [!TIP]
> Don't use too many weighted metrics. It can be difficult to understand why service instances are being moved around for balancing. A few metrics can go a long way!

Any information that can indicate the health and performance of your application is a candidate for metrics and health reports. A CPU performance counter can tell you how your node is utilized, but it doesn't tell you whether a particular service is healthy, because multiple services might be running on a single node. But, metrics like RPS, items processed, and request latency all can indicate the health of a specific service.

To report health, use code similar to this:

  ```csharp
    if (!result.HasValue)
    {
        HealthInformation healthInformation = new HealthInformation("ServiceCode", "StateDictionary", HealthState.Error);
        this.Partition.ReportInstanceHealth(healthInformation);
    }
  ```

To report a metric, use code similar to this:

  ```csharp
    this.ServicePartition.ReportLoad(new List<LoadMetric> { new LoadMetric("MemoryInMb", 1234), new LoadMetric("metric1", 42) });
  ```

### Service Fabric support logs

If you need to contact Microsoft support for help with your Azure Service Fabric cluster, support logs are almost always required. If your cluster is hosted in Azure, support logs are automatically configured and collected as part of creating a cluster. The logs are stored in a dedicated storage account in your cluster's resource group. The storage account doesn't have a fixed name, but in the account, you see blob containers and tables with names that start with *fabric*. For information about setting up log collections for a standalone cluster, see [Create and manage a standalone Azure Service Fabric cluster](service-fabric-cluster-creation-for-windows-server.md) and [Configuration settings for a standalone Windows cluster](service-fabric-cluster-manifest.md). For standalone Service Fabric instances, the logs should be sent to a local file share. You are **required** to have these logs for support, but they are not intended to be usable by anyone outside of the Microsoft customer support team.

## Enabling Diagnostics for a cluster

In order to take advantage of these logs, it is highly recommended that during cluster creation, "Diagnostics" is enabled. By turning on diagnostics, when the cluster is deployed, Windows Azure Diagnostics is able to acknowledge the Operational, Reliable Services, and Reliable actors channels, and store the data as explained further **here**.

As seen above, there is also an optional field to add an Application Insights (AppInsights) instrumentation key. If you choose to use AppInsights for any event analysis (read more about this **here**), include the AppInsights resource instrumentationKey (GUID) here.


If you are going to deploy containers to your cluster, enable WAD to pick up docker stats by adding this to your "WadCfg > DiagnosticMonitorConfiguration":

```json
"DockerSources": {
    "Stats": {
        "enabled": true,
        "sampleRate": "PT1M"
    }
},

```

## Measuring performance

Measure performance of your cluster will help you understand how it is able to handle load and drive decisions around scaling your cluster (see more about [scaling a cluster on Azure](service-fabric-cluster-scale-up-down.md), or [on-premise](service-fabric-cluster-windows-server-add-remove-nodes.md)). Performance data is also useful when compared to actions you or your applications and services may have taken, when analyzing logs in the future. Here are two common ways in which you can set up collecting performance data for your cluster:

* Using an agent: this is the preferred way of collecting performance data, since modifying the data an agent collects is a relatively easier process than changing diagnostics configurations of the cluster. Read [this](service-fabric-diagnostics-event-analysis-oms.md) and [this](../log-analytics/log-analytics-windows-agents.md) article to learn more about the OMS agent, which is one such monitoring agent that is able to pick up performance data for Service Fabric cluster VMs and any deployed containers.

* Configuring diagnostics to write performance counters to a table: for clusters on Azure, this means changing the Azure Diagnostics configuration to pick up the appropriate performance counters from the VMs in your cluster, and enabling it to pick up docker stats if you will be deploying any containers. [This article](../cloud-services/cloud-services-dotnet-diagnostics-performance-counters.md) goes over setting up performance counters, albeit for .NET applications; the process is very similar for Service Fabric clusters as well. A quick sample of code added to your "WadCfg > DiagnosticMonitorConfiguration" in a Resource Manager template looks like following. In this case, we are setting one performance counter, which is being sampled every 15 seconds (this can be changed and follows the format of "PT\<time>\<unit>", for example, PT3M would sample at three minute intervals), and transferred to the appropriate storage table every one minute.

    ```json
    "PerformanceCounters": {
        "scheduledTransferPeriod": "PT1M",
        "PerformanceCounterConfiguration": [
            {
                "counterSpecifier": "\\Processor(_Total)\\% Processor Time",
                "sampleRate": "PT15S",
                "unit": "Percent",
                "annotation": [
                ],
                "sinks": ""
            }
        ]
    }
    ```

## Next steps

Your logs and events need to be aggregated before they can be sent to any analysis platform. Read about [EventFlow](service-fabric-diagnostics-event-aggregation-eventflow.md) and [WAD](service-fabric-diagnostics-event-aggregation-wad.md) to better understand some of the recommended options.
