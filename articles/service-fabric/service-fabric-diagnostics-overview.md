---
title: Azure Service Fabric monitoring and diagnostics overview | Microsoft Docs
description: Learn how to monitor and diagnose Microsoft Azure Service Fabric applications that are hosted in Azure, development, or on-premises.
services: service-fabric
documentationcenter: .net
author: dkkapur
manager: mfussell
editor: ''

ms.assetid: edcc0631-ed2d-45a3-851d-2c4fa0f4a326
ms.service: service-fabric
ms.devlang: dotnet
ms.topic: article
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 05/10/2017
ms.author: dekapur

---
# Monitoring and Diagnostics in Azure Service Fabric

Monitoring and diagnostics are critical to developing, testing, and successfully deploying in production environments. Service Fabric solutions work best when you implement monitoring and diagnostics from the get go, helping to ensure that services work seamlessly in local development environments and in real-world production cluster setups.

The main goals of monitoring and diagnostics are to:
* Detect and diagnose hardware and infrastructure issues
* Detect software and app issues, reduce service downtime
* Understand resource consumption and help drive operations decisions
* Optimize application, service, and infrastructure performance
* Generate business insights and identify areas of improvement


Monitoring and diagnostics is important in guaranteeing that everything is working as expected, and enables you to respond to situations with minimum disruption of service. 

The overall workflow of monitoring and diagnostics consists of three steps: 
1. **Event generation**: this includes events (logs, traces, custom events) at both the infrastructure (cluster) and application / service levels 
2. **Event aggregation**: generated events need to be collected and aggregated before they can be displayed. This can be through storage tables as set up by Windows Azure Diagnostics or by creating an EventFlow pipeline
3. **Analysis**: events need to be visualized and accessible in some format, to allow for analysis and display as needed. You can choose to use tools such as ApplicationInsights, Operations Management Suite, Kibana, and several others

Although multiple products are available that cover all of these three areas, many customers choose different technologies for each of these aspects. It is important that pieces work together to deliver an end-to-end monitoring solution for the application.

Depending on your needs, you can further extend diagnostics and monitoring to include automated alerts and mitigation, which often come built into the analytics tools you may choose to use. 

## Event Generation

Events, logs, and traces can be generated from the infrastructure layer (anything from the cluster, the machines, or Service Fabric actions) or from the application layer (any instrumentation added to apps and services deployed to the cluster).

### Infrastructure: Monitoring the cluster

Service Fabric can help keep an application running during infrastructure failures, but you still need to understand whether an error is occurring in the application or in the underlying infrastructure. You also need to monitor the infrastructure for capacity planning, so you know when to add or remove infrastructure. It's important to monitor and troubleshoot both the infrastructure and the application that make up a Service Fabric deployment. Even if an application is available to customers, the infrastructure might still be a potential source of issues. To make sure that your cluster is behaving in the way that you expect, Service Fabric sets you up with five different logging channels out-of-the-box:

1. Operational channel: high-level operations performed by Service Fabric and the cluster, includes events for a node coming up, a new application being deployed, or a SF upgrade rollback, etc. 
2. Customer information channel: health reports and load balancing decisions
3. Reliable Services events: programming model specific events
4. Reliable Actors events: programming model specific events and performance counters
5. Support logs: system logs generated by Service Fabric only to be used by us when providing support

It is highly recommended that on cluster creation, "Diagnostics" are enabled. This can be done in the portal as seen below or via using an Azure Resource Manager template that includes diagnostics. 

![Azure portal cluster creation with diagnostics enabled](./media/service-fabric-diagnostics-overview/azure-enable-diagnostics.png)

As seen above, there is also an optional field to add an Application Insights (AppInsights) instrumentation key. If you choose to use AppInsights for any event analysis (AppInsights is one of the recommended solutions), include the AppInsights resource instrumentationKey (GUID) here.

#### Service Fabric support logs

If you need to contact Microsoft support for help with your Azure Service Fabric cluster, support logs are almost always required. If your cluster is hosted in Azure, support logs are automatically configured and collected as part of creating a cluster. The logs are stored in a dedicated storage account in your cluster's resource group. The storage account doesn't have a fixed name, but in the account, you see blob containers and tables with names that start with *fabric*. For information about setting up log collections for a standalone cluster, see [Create and manage a standalone Azure Service Fabric cluster](service-fabric-cluster-creation-for-windows-server.md) and [Configuration settings for a standalone Windows cluster](service-fabric-cluster-manifest.md). For standalone Service Fabric instances, the logs should be sent to a local file share. You are **required** to have these logs for support, but they are not intended to be usable by anyone outside of the Microsoft customer support team.

#### Azure Service Fabric health and load reporting

Service Fabric has its own health model, which is described in detail in these articles:
- [Introduction to Service Fabric health monitoring](service-fabric-health-introduction.md)
- [Report and check service health](service-fabric-diagnostics-how-to-report-and-check-service-health.md)
- [Add custom Service Fabric health reports](service-fabric-report-health.md)
- [View Service Fabric health reports](service-fabric-view-entities-aggregated-health.md)

Health monitoring is critical to multiple aspects of operating a service. Health monitoring is especially important when Service Fabric performs a named application upgrade. After each upgrade domain of the service is upgraded and is available to your customers, the upgrade domain must pass health checks before the deployment moves to the next upgrade domain. If good health status cannot be achieved, the deployment is rolled back, so that the application is in a known, good state. Although some customers might be affected before the services are rolled back, most customers won't experience an issue. Also, a resolution occurs relatively quickly, and without having to wait for action from a human operator. The more health checks that are incorporated into your code, the more resilient your service is to deployment issues.

Another aspect of service health is reporting metrics from the service. Metrics are important in Service Fabric because they are used to balance resource usage. Metrics also can be an indicator of system health. For example, you might have an application that has many services, and each instance reports a requests per second (RPS) metric. If one service is using more resources than another service, Service Fabric moves service instances around the cluster, to try to maintain even resource utilization. For a more detailed explanation of how resource utilization works, see [Manage resource consumption and load in Service Fabric with metrics](service-fabric-cluster-resource-manager-metrics.md).

Metrics also can help give you insight into how your service is performing. Over time, you can use metrics to check that the service is operating within expected parameters. For example, if trends show that at 9 AM on Monday morning the average RPS is 1,000, then you might set up a health report that alerts you if the RPS is below 500 or above 1,500. Everything might be perfectly fine, but it might be worth a look to be sure that your customers are having a great experience. Your service can define a set of metrics that can be reported for health check purposes, but that don't affect the resource balancing of the cluster. To do this, set the metric weight to zero. We recommend that you start all metrics with a weight of zero, and not increase the weight until you are sure that you understand how weighting the metrics affects resource balancing for your cluster.

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


### Application: instrumenting code for custom events

Instrumenting the code is the basis for most other aspects of monitoring your services. Instrumentation is the only way you can know that something is wrong, and to diagnose what needs to be fixed. Although technically it's possible to connect a debugger to a production service, it's not a common practice. So, having detailed instrumentation data is important. 

Some products automatically instrument your code. Although these solutions can work well, manual instrumentation is almost always required. In the end, you must have enough information to forensically debug the application. The next sections describe different approaches to instrumenting your code, and when to choose one approach over another.

#### EventSource

When you create a Service Fabric solution from a template in Visual Studio, an **EventSource**-derived class (**ServiceEventSource** or **ActorEventSource**) is generated. A template is created, in which you can add events for your application or service. The **EventSource** name **must** be unique, and should be renamed from the default template string MyCompany-&lt;solution&gt;-&lt;project&gt;. Having multiple **EventSource** definitions that use the same name causes an issue at run time. Each defined event must have a unique identifier. If an identifier is not unique, a runtime failure occurs. Some organizations preassign ranges of values for identifiers to avoid conflicts between separate development teams. For more information, see [Vance's blog](https://blogs.msdn.microsoft.com/vancem/2012/07/09/introduction-tutorial-logging-etw-events-in-c-system-diagnostics-tracing-eventsource/) or the [MSDN documentation](https://msdn.microsoft.com/library/dn774985(v=pandp.20).aspx).

##### Using structured EventSource events

Each of the events in the code examples in this section are defined for a specific case, for example, when a service type is registered. When you define messages by use case, data can be packaged with the text of the error, and you can more easily search and filter based on the names or values of the specified properties. Structuring the instrumentation output makes it easier to consume, but requires more thought and time to define a new event for each use case. Some event definitions can be shared across the entire application. For example, a method start or stop event would be reused across many services within an application. A domain-specific service, like an order system, might have a **CreateOrder** event, which has its own unique event. This approach might generate many events, and potentially require coordination of identifiers across project teams. 

```csharp
    [EventSource(Name = "MyCompany-VotingState-VotingStateService")]
    internal sealed class ServiceEventSource : EventSource
    {
        public static readonly ServiceEventSource Current = new ServiceEventSource();

        // The instance constructor is private to enforce singleton semantics.
        private ServiceEventSource() : base() { }

        ...

        // The ServiceTypeRegistered event contains a unique identifier, an event attribute that defined the event, and the code implementation of the event.
        private const int ServiceTypeRegisteredEventId = 3;
        [Event(ServiceTypeRegisteredEventId, Level = EventLevel.Informational, Message = "Service host process {0} registered service type {1}", Keywords = Keywords.ServiceInitialization)]
        public void ServiceTypeRegistered(int hostProcessId, string serviceType)
        {
            WriteEvent(ServiceTypeRegisteredEventId, hostProcessId, serviceType);
        }

        // The ServiceHostInitializationFailed event contains a unique identifier, an event attribute that defined the event, and the code implementation of the event.
        private const int ServiceHostInitializationFailedEventId = 4;
        [Event(ServiceHostInitializationFailedEventId, Level = EventLevel.Error, Message = "Service host initialization failed", Keywords = Keywords.ServiceInitialization)]
        public void ServiceHostInitializationFailed(string exception)
        {
            WriteEvent(ServiceHostInitializationFailedEventId, exception);
        }
```

##### Using EventSource generically

Because defining specific events can be difficult, many people define a few events with a common set of parameters that generally output their information as a string. Much of the structured aspect is lost, and it's more difficult to search and filter the results. In this approach, a few events that usually correspond to the logging levels are defined. The following snippet defines a debug and error message:

```csharp
    [EventSource(Name = "MyCompany-VotingState-VotingStateService")]
    internal sealed class ServiceEventSource : EventSource
    {
        public static readonly ServiceEventSource Current = new ServiceEventSource();

        // The Instance constructor is private, to enforce singleton semantics.
        private ServiceEventSource() : base() { }

        ...

        private const int DebugEventId = 10;
        [Event(DebugEventId, Level = EventLevel.Verbose, Message = "{0}")]
        public void Debug(string msg)
        {
            WriteEvent(DebugEventId, msg);
        }

        private const int ErrorEventId = 11;
        [Event(ErrorEventId, Level = EventLevel.Error, Message = "Error: {0} - {1}")]
        public void Error(string error, string msg)
        {
            WriteEvent(ErrorEventId, error, msg);
        }
```

Using a hybrid of structured and generic instrumentation also can work well. Structured instrumentation is used for reporting errors and metrics. Generic events can be used for the detailed logging that is consumed by engineers for troubleshooting.

#### ASP.NET Core logging

It's important to carefully plan how you will instrument your code. The right instrumentation plan can help you avoid potentially destabilizing your code base, and then needing to reinstrument the code. To reduce risk, you can choose an instrumentation library like [Microsoft.Extensions.Logging](https://www.nuget.org/packages/Microsoft.Extensions.Logging/), which is part of Microsoft ASP.NET Core. ASP.NET Core has an [ILogger](https://docs.microsoft.com/aspnet/core/api/microsoft.extensions.logging.ilogger) interface that you can use with the provider of your choice, while minimizing the effect on existing code. You can use the code in ASP.NET Core on Windows and Linux, and in the full .NET Framework, so your instrumentation code is standardized.

##### Using Microsoft.Extensions.Logging in Service Fabric

1. Add the Microsoft.Extensions.Logging NuGet package to the project you want to instrument. Also, add any provider packages (for a third-party package, see the following example). For more information, see [Logging in ASP.NET Core](https://docs.microsoft.com/aspnet/core/fundamentals/logging).
2. Add a **using** directive for Microsoft.Extensions.Logging to your service file.
3. Define a private variable within your service class.

  ```csharp
  private ILogger _logger = null;

  ```
4. In the constructor of your service class, add this code.

  ```csharp
  _logger = new LoggerFactory().CreateLogger<Stateless>();

  ```
5. Start instrumenting your code in your methods. Here are a few samples:

  ```csharp
  _logger.LogDebug("Debug-level event from Microsoft.Logging");
  _logger.LogInformation("Informational-level event from Microsoft.Logging");

  // In this variant, we're adding structured properties RequestName and Duration, which have values MyRequest and the duration of the request.
  // Later in the article, we discuss why this step is useful.
  _logger.LogInformation("{RequestName} {Duration}", "MyRequest", requestDuration);

  ```

#### Using other logging providers

Some third-party providers use the approach described in the preceding section, including [Serilog](https://serilog.net/), [NLog](http://nlog-project.org/), and [Loggr](https://github.com/imobile3/Loggr.Extensions.Logging). You can plug each of these into ASP.NET Core logging, or you can use them separately. Serilog has a feature that enriches all messages sent from a logger. This feature can be useful to output the service name, type, and partition information. To use this capability in the ASP.NET Core infrastructure, do these steps:

1. Add the Serilog, Serilog.Extensions.Logging, and Serilog.Sinks.Observable NuGet packages to the project. For the next example, also add Serilog.Sinks.Literate. A better approach is shown later in this article.
2. In Serilog, create a LoggerConfiguration and the logger instance.

  ```csharp
  Log.Logger = new LoggerConfiguration().WriteTo.LiterateConsole().CreateLogger();
  ```

3. Add a Serilog.ILogger argument to the service constructor, and pass the newly created logger.

  ```csharp
  ServiceRuntime.RegisterServiceAsync("StatelessType", context => new Stateless(context, Log.Logger)).GetAwaiter().GetResult();
  ```

4. In the service constructor, add the following code. The code creates the property enrichers for the **ServiceTypeName**, **ServiceName**, **PartitionId**, and **InstanceId** properties of the service. It also adds a property enricher to the ASP.NET Core logging factory, so you can use Microsoft.Extensions.Logging.ILogger in your code.

  ```csharp
  public Stateless(StatelessServiceContext context, Serilog.ILogger serilog)
      : base(context)
  {
      PropertyEnricher[] properties = new PropertyEnricher[]
      {
          new PropertyEnricher("ServiceTypeName", context.ServiceTypeName),
          new PropertyEnricher("ServiceName", context.ServiceName),
          new PropertyEnricher("PartitionId", context.PartitionId),
          new PropertyEnricher("InstanceId", context.ReplicaOrInstanceId),
      };

      serilog.ForContext(properties);

      _logger = new LoggerFactory().AddSerilog(serilog.ForContext(properties)).CreateLogger<Stateless>();
  }
  ```

5. Instrument the code the same as if you were using ASP.NET Core without Serilog.

  > [!NOTE]
  > We recommend that you don't use the static Log.Logger with the preceding example. Service Fabric can host multiple instances of the same service type within a single process. If you use the static Log.Logger, the last writer of the property enrichers will show values for all instances that are running. This is one reason why the _logger variable is a private member variable of the service class. Also, you must make the _logger available to common code, which might be used across services.

#### Choosing a logging provider

If your application relies on high performance, **EventSource** is the best approach to use. **EventSource** *generally* uses fewer resources and performs better than ASP.NET Core logging or any of the available third-party solutions.  This isn't an issue for many services, but if your service is performance-oriented, using **EventSource** might be a better choice. To get the same benefits of structured logging, **EventSource** requires a larger investment from your engineering team. To determine the approach to use for your project, do a quick prototype of what each option entails, and then choose the one that best meets your needs.

## Event Aggregation and Collection

### Azure Diagnostics

In addition to the information that Azure Monitor provides, Azure collects events from each of the services at a central location. For more information, learn how to configure event collection for [Windows](service-fabric-diagnostics-how-to-setup-wad.md) and [Linux](service-fabric-diagnostics-how-to-setup-lad.md). These articles show how you can collect the event data and send it to Azure storage. You can do this in the Azure portal or in your Azure Resource Manager template by enabling diagnostics. Azure Diagnostics collects a few event sources that Service Fabric automatically produces:

- **ETW** events emmitted as part of the Operational channel
- **EventSource** events and performance counters when you use the Reliable Actor programming model. The events are enumerated in [Diagnostics and performance monitoring for Reliable Actors](service-fabric-reliable-actors-diagnostics.md).
- **EventSource** events when you use the Reliable Services programming model. The events are enumerated in [Diagnostic functionality for stateful Reliable Services](service-fabric-reliable-services-diagnostics.md).


When configured, events appear in an Azure storage account that was created when you created the cluster, assuming that you enabled diagnostics. The tables are named WADServiceFabricReliableActorEventTable, WADServiceFabricReliableServiceEventTable, and WADServiceFabricSystemEventTable. Health events are not added by default; you must modify the Resource Manager template to add them. For more information, see [Collect logs by using Azure Diagnostics](service-fabric-diagnostics-how-to-setup-wad.md).

These articles listed in this section also can show you how to get custom events into Azure storage. Other Azure Diagnostics articles about configuring performance counters, or articles that have other monitoring information from virtual machines to Azure Diagnostics also apply to a Service Fabric cluster. For instance, if you don't want to use Azure Table storage as a destination, see [Streaming Azure Diagnostics data in the hot path by using Event Hubs](../event-hubs/event-hubs-streaming-azure-diags-data.md). When the events are in Azure Event Hubs, you can read them and send them to the location you choose. You can get more information about integrating [Azure diagnostic information with Application Insights](https://azure.microsoft.com/blog/azure-diagnostics-integration-with-application-insights/).

A disadvantage of using Azure Diagnostics is that you set it up by using a Resource Manager template. Diagnostics, then, occurs only at the virtual machine scale set level. A virtual machine scale set corresponds to a node type in Service Fabric. You configure each node type for all the applications and services that might run on a node of that type. This might be many **EventSource** events, depending on the number of applications and services you configure. You also must deploy Resource Manager any time an application configuration changes. Ideally, the monitoring configuration would travel with the service configuration.

Azure Diagnostics works only for Service Fabric clusters deployed to Azure. It works both for Windows and Linux clusters.

### EventFlow

[Microsoft Diagnostics EventFlow](https://github.com/Azure/diagnostics-eventflow) can route events from a node to one or more monitoring destinations. Because it is included as a NuGet package in your service project, EventFlow code and configuration travel with the service, eliminating the per-node configuration issue mentioned earlier about Azure Diagnostics. EventFlow runs within your service process, and connects directly to the configured outputs. Because of the direct connection, EventFlow works for Azure, container, and on-premises service deployments. Be careful if you run EventFlow in high-density scenarios, such as in a container, because each EventFlow pipeline makes an external connection. If you host a lot of processes, you get a lot of outbound connections! This isn't as much a concern for Service Fabric applications, because all replicas of a `ServiceType` run in the same process, and this limits the number of outbound connections. EventFlow also offers event filtering, so that only the events that match the specified filter are sent. For detailed information about how to use EventFlow with Service Fabric, see [Collect logs directly from an Azure Service Fabric service process](service-fabric-diagnostic-collect-logs-without-an-agent.md).

To use EventFlow:

1. Add the NuGet package to your service project.
2. In the service's **Main** function, create the EventFlow pipeline, and then configure the outputs. In the following example, we use Serilog as an output.

  ```csharp
  internal static class Program
  {
      /// <summary>
      /// This is the entry point of the service host process.
      /// </summary>
      private static void Main()
      {
          try
          {
              using (var pipeline = ServiceFabricDiagnosticPipelineFactory.CreatePipeline("MonitoringE2E-Stateless-Pipeline"))
              {
                  Log.Logger = new LoggerConfiguration().WriteTo.EventFlow(pipeline).CreateLogger();

                  // The ServiceManifest.xml file defines one or more service type names.
                  // Registering a service maps a service type name to a .NET type.
                  // When Service Fabric creates an instance of this service type,
                  // an instance of the class is created in this host process.

                  ServiceRuntime.RegisterServiceAsync("StatelessType", context => new Stateless(context, Log.Logger)).GetAwaiter().GetResult();
                  ServiceEventSource.Current.ServiceTypeRegistered(Process.GetCurrentProcess().Id, typeof(Stateless).Name);

                  // Prevents this host process from terminating, so services keep running.
                  Thread.Sleep(Timeout.Infinite);
              }
          }
          catch (Exception e)
          {
              ServiceEventSource.Current.ServiceHostInitializationFailed(e.ToString());
              throw;
          }
      }
  }
  ```

3. Create a file named eventFlowConfig.json in the service's \\PackageRoot\\Config folder. Inside the file, the configuration looks like this:

  ```json
      {
      "inputs": [
          {
          "type": "EventSource",
          "sources": [
              { "providerName": "Microsoft-ServiceFabric-Services" },
              { "providerName": "Microsoft-ServiceFabric-Actors" },
              { "providerName": "MyCompany-MonitoringE2E-Stateless" }
          ]
          },
          {
          "type": "Serilog"
          }
      ],
      "filters": [
          {
          "type": "drop",
          "include": "Level == Verbose"
          },
          {
          "type": "metadata",
          "metadata": "request",
          "requestNameProperty": "RequestName",
          "include":  "RequestName==MyRequest",
          "durationProperty": "Duration",
          "durationUnit": "milliseconds"
          }
      ],
      "outputs": [
          {
          "type": "StdOutput"
          },
          {
          "type": "ApplicationInsights",
          "instrumentationKey": "== instrumentation key here =="
          }
      ],
      "schemaVersion": "2016-08-11",
      "extensions": []
      }
  ```
    In the configuration, two inputs are defined: the two **EventSource**-based sources that Service Fabric creates, and the **EventSource** for the service. The system-level and health events that use ETW are not available to EventFlow. This is because a high-level privilege is required to listen to an ETW source, and services should never run with high privileges. The other input is Serilog. Serilog configuration occurred in the **Main** method.  Some filters are applied. The first filter tells EventFlow to drop all events that have an event level of verbose. Two outputs are configured: standard output, which writes to the output window in Visual Studio, and ApplicationInsights. Make sure you add your instrumentation key.

4. Instrument the code. In the next example, we instrument `RunAsync` a few different ways, to show examples. In the following code, we're still using Serilog. Some of the syntax we use is specific to Serilog. Note the specific capabilities for the logging solution you choose. Three events are generated: a debug-level event, and two informational events. The second informational event tracks the request duration. In the configuration of EventFlow described earlier, the debug-level event should not flow to the output.

  ```csharp
      Stopwatch sw = Stopwatch.StartNew();

      while (true)
      {
          cancellationToken.ThrowIfCancellationRequested();

          sw.Restart();

          // Delay a random interval, to provide a more interesting request duration.
          await Task.Delay(TimeSpan.FromMilliseconds(DateTime.Now.Millisecond), cancellationToken);

          ServiceEventSource.Current.ServiceMessage(this.Context, "Working-{0}", ++iterations);
          _logger.LogDebug("Debug level event from Microsoft.Logging");
          _logger.LogInformation("Informational level event from Microsoft.Logging");
          _logger.LogInformation("{RequestName} {Duration}", "MyRequest", sw.ElapsedMilliseconds);
      }
  ```

To view the events in Azure Application Insights, in the Azure portal, go to your Application Insights resource. To see the events, select the **Search** box.

![Application Insights Search view of events](./media/service-fabric-diagnostics-overview/ai-search-events.png)

You can see the traces at the bottom of the preceding screenshot. It shows only two events, and that the debug-level event was dropped by EventFlow. The request entry preceding the trace is the third `_logger` instrumentation line. The line shows that the event was translated into a request metric in Application Insights.

In the filter definition, the type is **metadata**. This declares that an event that has a property of `RequestName` with the value `MyRequest`, and another property, `Duration`, contain the duration of the request, in milliseconds. This is what you see in the request event in Application Insights. The same approach works with any of the supported EventFlow inputs, including **EventSource**.

If you have a standalone cluster that cannot be connected to a cloud-based solution for policy reasons, you can use Elasticsearch as an output. However, other outputs can be written, and pull requests are encouraged. Some third-party providers for ASP.NET Core logging also have solutions that support on-premises installations.

## Visualization, analysis, and alerts

The final part of monitoring is visualizing the event stream, reporting on service performance, and alerting when an issue is detected. You can use different solutions for this aspect of monitoring. AppInsights and Operations Management Suite (OMS) to alert based on the stream of events. You can use Microsoft Power BI or a third-party solution like [Kibana](https://www.elastic.co/products/kibana) or [Splunk](https://www.splunk.com/) to visualize the data.

### AppInsights

AppInsights is one of the recommended tools for monitoring applications and services. The updated AI.SDK does a great job with Service Fabric events, and in addition to providing good data visualizations and a querying tool (through AppInsights Analytics), it also is able to create an accurate AppMap that can help you trace dependencies between processes in an application or a cluster.

Set up an AppInsights resource by searching for "Application Insights" in the Azure Marketplace. After you create one, go to *Properties* to find the AI Instrumentation Key (in the form of a GUID). This is used for:
* Integrating AppInsights to receive infrastructure level events from a Service Fabric cluster directly via an Azure Resource Manager template or through Azure portal when creating a cluster, assuming Diagnostics has been enabled
* Configuring EventFlow (eventFlowConfig.json) to output data to Application Insights, as shown in the section above

### OMS

OMS is another recommended tool for diagnostics and monitoring of Service Fabric clusters. The Service Fabric solution can be added to any workspace and has a dashboard to show the different types of Service Fabric events. OMS workspaces also have a powerful log querying tool in Log Analytics.

To configure an OMS workspace, confirm that Diagnostics was enabled for your cluster. Add "Service Fabric Analytics" from Azure Marketplace to an existing OMS workspace or create a new one. Configure the workspace's data sources to connect to the Azure Storage tables that your cluster is writing events to. 

In order for OMS to pick up custom events, you also have to make sure that the instrumentation you add to your applications should also write to the same Storage tables, or any other ones that are also configured to be sources for the workspace. 

OMS is also currently the recommended path to take for data visualization and analysis if you need monitoring and diagnostics for containers, since you can add a Containers solution to your workspace that works well with Service Fabric orchestrated containers. To view a short guide on setting this up, go [here](service-fabric-diagnostics-containers-windowsserver.md).

### Azure Monitor

You can use [Azure Monitor](../monitoring-and-diagnostics/monitoring-overview.md) to monitor many of the Azure resources on which a Service Fabric cluster is built. A set of metrics for the [virtual machine scale set](../monitoring-and-diagnostics/monitoring-supported-metrics.md#microsoftcomputevirtualmachinescalesets) and individual [virtual machines](../monitoring-and-diagnostics/monitoring-supported-metrics.md#microsoftcomputevirtualmachinescalesetsvirtualmachines) is automatically collected and displayed in the Azure portal. To view the collected information, in the Azure portal, select the resource group that contains the Service Fabric cluster. Then, select the virtual machine scale set that you want to view. In the **Monitoring** section, select **Metrics** to view a graph of the values.

![Azure portal view of collected metric information](./media/service-fabric-diagnostics-overview/azure-monitoring-metrics.png)

To customize the charts, follow the instructions in [Metrics in Microsoft Azure](../monitoring-and-diagnostics/insights-how-to-customize-monitoring.md). You also can create alerts based on these metrics, as described in [Create alerts in Azure Monitor for Azure services](../monitoring-and-diagnostics/insights-alerts-portal.md). You can send alerts to a notification service by using web hooks, as described in [Configure a web hook on an Azure metric alert](../monitoring-and-diagnostics/insights-webhooks-alerts.md). Azure Monitor supports only one subscription. If you need to monitor multiple subscriptions, or if you need additional features, [Log Analytics](https://azure.microsoft.com/documentation/services/log-analytics/), part of Microsoft Operations Management Suite, provides a holistic IT management solution both for on-premises and cloud-based infrastructures. You can route data from Azure Monitor directly to Log Analytics, so you can see metrics and logs for your entire environment in a single place.

We recommend using Operations Management Suite to monitor your on-premises infrastructure, but you can use any existing solution that your organization uses for infrastructure monitoring.

## Additional steps

### Watchdogs

A watchdog is a separate service that can watch health and load across services, and report health for anything in the health model hierarchy. This can help prevent errors that would not be detected based on the view of a single service. Watchdogs also are a good place to host code that can perform remediation actions for known conditions without user interaction. You can find a sample watchdog service implementation [here](https://github.com/Azure-Samples/service-fabric-watchdog-service).


## Next steps

* [Collect logs with Azure Diagnostics](service-fabric-diagnostics-how-to-setup-wad.md)
* [Collect logs directly from an Azure Service Fabric service process](service-fabric-diagnostic-collect-logs-without-an-agent.md)
*  [Manage resource consumption and load in Service Fabric with metrics](service-fabric-cluster-resource-manager-metrics.md)
