<properties 
	pageTitle="Sampling, filtering and preprocessing in the Application Insights SDK" 
	description="Write plug-ins for the SDK to filter, sample or add properties to the data before the telemetry is sent to the Application Insights portal." 
	services="application-insights"
    documentationCenter="" 
	authors="alancameronwills" 
	manager="douge"/>
 
<tags 
	ms.service="application-insights" 
	ms.workload="tbd" 
	ms.tgt_pltfrm="ibiza" 
	ms.devlang="multiple" 
	ms.topic="article" 
	ms.date="11/04/2015" 
	ms.author="awills"/>

# Sampling, filtering and preprocessing telemetry in the Application Insights SDK

*Application Insights is in preview.*

You can write and configure plug-ins for the Application Insights SDK to customize how telemetry is captured and processed before it is sent to the Application Insights service. 

Currently these features are available for the ASP.NET SDK.

* [Sampling](#sampling) reduces the volume of telemetry without affecting your statistics. It keeps together related data points so that you can navigate between them when diagnosing a problem. In the portal, the total counts are multiplied to compensate for the sampling.
* [Filtering](#filtering) lets you select or modify telemetry in the SDK before it is sent to the server. For example, you could reduce the volume of telemetry by excluding requests from robots. This is a more basic approach to reducing traffic than sampling. It allows you more control over what is transmitted, but you have to be aware that it will affect your statistics - for example, if you filter out all successful requests.
* [Add properties](#add-properties) to any telemetry sent from your app, including telemetry from the standard modules. For example, you could add calculated values; or version numbers by which to filter the data in the portal.
* [The SDK API](app-insights-api-custom-events-metrics.md) is used to send custom events and metrics.

Before you start:

* Install the [Application Insights SDK](app-insights-start-monitoring-app-health-usage.md) in your app. Install the NuGet packages manually and select the latest *prerelease* version.
* Try the [Application Insights API](app-insights-api-custom-events-metrics.md). 


## Sampling

*This feature is in beta.*

The recommended way to reduce traffic while preserving accurate statistics. The filter selects items that are related so that you can navigate between items in diagnosis. Event counts are adjusted in metric explorer to compensate for the filtered items.

1. Update your project's NuGet packages to the latest *pre-release* version of Application Insights. Right-click the project in Solution Explorer, choose Manage NuGet Packages, check **Include prerelease** and search for Microsoft.ApplicationInsights.Web. 

2. Add this snippet to [ApplicationInsights.config](app-insights-configuration-with-applicationinsights-config.md):

```XML

    <TelemetryProcessors>
     <Add Type="Microsoft.ApplicationInsights.WindowsServer.TelemetryChannel.SamplingTelemetryProcessor, Microsoft.AI.ServerTelemetryChannel">

     <!-- Set a percentage close to 100/N where N is an integer. -->
     <!-- E.g. 50 (=100/2), 33.33 (=100/3), 25 (=100/4), 10, 1 (=100/100), 0.1 (=100/1000) ... -->
     <SamplingPercentage>10</SamplingPercentage>
     </Add>
   </TelemetryProcessors>

```


To get sampling on the data from web pages, put an extra line in the [Application Insights snippet](app-insights-javascript.md) that you inserted (typically in a master page such as _Layout.cshtml):

*JavaScript*

```JavaScript

	}({ 

	samplingPercentage: 10.0, 

	instrumentationKey:...
	}); 
```

* Set a percentage (10 in these examples) that is equal to 100/N where N is an integer - for example 50 (=100/2), 33.33 (=100/3), 25 (=100/4), or 10 (=100/10). 
* If you have a lot of data, you can use very low sampling rates, such as 0.1.
* If you set sampling in both web page and server, make sure to set the same sampling percentage in both sides.
* Client and server sides will coordinate to select related items.

[Learn more about sampling](app-insights-sampling.md).

## Filtering

This technique gives you more direct control over what is included or excluded from the telemetry stream. You can use it in conjunction with Sampling, or separately.

To filter telemetry, you write a telemetry processor and register it with the SDK. All telemetry goes through your processor, and you can choose to drop it from the stream, or add properties. This includes telemetry from the standard modules such as the HTTP request collector and the dependency collector, as well as telemetry you have written yourself. You can, for example, filter out telemetry about requests from robots, or successful dependency calls. 

> [AZURE.WARNING] Filtering the telemetry sent from the SDK using processors can skew the statistics that you see in the portal, and make it difficult to follow related items.
> 
> Instead, consider using [sampling](#sampling).

### Create a telemetry processor

1. Update the Application Insights SDK to the latest version (2.0.0-beta2 or later). Right-click your project in Visual Studio Solution Explorer and choose Manage NuGet Packages. In NuGet package manager, check **Include prerelease** and search for Microsoft.ApplicationInsights.Web.

1. To create a filter, implement ITelemetryProcessor. This is another extensibility point like telemetry module, telemetry initializer and telemetry channel. 

    Notice that Telemetry Processors construct a chain of processing. When you instantiate a telemetry processor, you pass a link to the next processor in the chain. When a telemetry data point is passed to the Process method, it does its work and then calls the next Telemetry Processor in the chain.

    ``` C#

    using Microsoft.ApplicationInsights.Channel;
    using Microsoft.ApplicationInsights.Extensibility;

    public class SuccessfulDependencyFilter : ITelemetryProcessor
      {
        private ITelemetryProcessor Next { get; set; }

        // You can pass values from .config
        public string MyParamFromConfigFile { get; set; }

        // Link processors to each other in a chain.
        public SuccessfulDependencyFilter(ITelemetryProcessor next)
        {
            this.Next = next;
        }
        public void Process(ITelemetry item)
        {
            // To filter out an item, just return 
            if (!OKtoSend(item)) { return; }
            // Modify the item if required 
            ModifyItem(item);

            this.Next.Process(item);
        }

        // Example: replace with your own criteria.
        private bool OKtoSend (ITelemetry item)
        {
            var dependency = item as DependencyTelemetry;
            if (dependency == null) return true;

            return dependency.Success != true;
        }

        // Example: replace with your own modifiers.
        private void ModifyItem (ITelemetry item)
        {
            item.Context.Properties.Add("app-version", "1." + MyParamFromConfigFile);
        }
    }
    

    ```
2. Insert this in ApplicationInsights.config: 

```XML

    <TelemetryProcessors>
      <Add Type="WebApplication9.SuccessfulDependencyFilter, WebApplication9">
         <!-- Set public property -->
         <MyParamFromConfigFile>2-beta</MyParamFromConfigFile>
      </Add>
    </TelemetryProcessors>

```

(Notice that this is the same section where you initialize a sampling filter.)

You can pass string values from the .config file by providing public named properties in your class. 

> [AZURE.WARNING] Take care to match the type name and any property names in the .config file to the class and property names in the code. If the .config file references a non-existent type or property, the SDK may silently fail to send any telemetry.

 
**Alternatively,** you can initialize the filter in code. In a suitable initialization class - for example AppStart in Global.asax.cs - insert your processor into the chain:

    ```C#

    var builder = TelemetryConfiguration.Active.GetTelemetryProcessorChainBuilder();
    builder.Use((next) => new SuccessfulDependencyFilter(next));

    // If you have more processors:
    builder.Use((next) => new AnotherProcessor(next));

    builder.Build();

    ```

TelemetryClients created after this point will use your processors.

### Example filters

#### Synthetic requests

Filter out bots and web tests. Although Metrics Explorer gives you the option to filter out synthetic sources, this option reduces traffic by filtering them at the SDK.

``` C#

    public void Process(ITelemetry item)
    {
      if (!string.IsNullOrEmpty(item.Context.Operation.SyntheticSource)) {return;}

      // Send everything else: 
      this.Next.Process(item);
    }

```

#### Failed authentication

Filter out requests with a "401" response. 

```C#

public void Process(ITelemetry item)
{
    var request = item as RequestTelemetry;

    if (request != null &&
    request.ResponseCode.Equals("401", StringComparison.OrdinalIgnoreCase))
    {
        // To filter out an item, just terminate the chain: 
        return;
    }
    // Send everything else: 
    this.Next.Process(item);
}

```

#### Filter out fast remote dependency calls

If you only want to diagnose calls that are slow, filter out the fast ones. 

> [AZURE.NOTE] This will skew the statistics you see on the portal. The dependency chart will look as if the dependency calls are all failures.

``` C#

public void Process(ITelemetry item)
{
    var request = item as DependencyTelemetry;
            
    if (request != null && request.Duration.Milliseconds < 100)
    {
        return;
    }
    this.Next.Process(item);
}

```


## Add properties

Use telemetry initializers to define global properties that are sent with all telemetry; and to override selected behavior of the standard telemetry modules. 

For example, the Application Insights for Web package collects telemetry about HTTP requests. By default, it flags as failed any request with a response code >= 400. But if you want to treat 400 as a success, you can provide a telemetry initializer that sets the Success property.

If you provide a telemetry initializer, it is called whenever any of the Track*() methods is called. This includes methods called by the standard telemetry modules. By convention, these modules do not set any property that has already been set by an initializer. 

**Define your initializer**

*C#*

```C#

    using System;
    using Microsoft.ApplicationInsights.Channel;
    using Microsoft.ApplicationInsights.DataContracts;
    using Microsoft.ApplicationInsights.Extensibility;

    namespace MvcWebRole.Telemetry
    {
      /*
       * Custom TelemetryInitializer that overrides the default SDK 
       * behavior of treating response codes >= 400 as failed requests
       * 
       */
      public class MyTelemetryInitializer : ITelemetryInitializer
      {
        public void Initialize(ITelemetry telemetry)
        {
            var requestTelemetry = telemetry as RequestTelemetry;
            // Is this a TrackRequest() ?
            if (requestTelemetry == null) return;
            int code;
            bool parsed = Int32.TryParse(requestTelemetry.ResponseCode, out code);
            if (!parsed) return;
            if (code >= 400 && code < 500)
            {
                // If we set the Success property, the SDK won't change it:
                requestTelemetry.Success = true;
                // Allow us to filter these requests in the portal:
                requestTelemetry.Context.Properties["Overridden400s"] = "true";
            }
            // else leave the SDK to set the Success property      
        }
      }
    }
```

**Load your initializer**

In ApplicationInsights.config:

    <ApplicationInsights>
      <TelemetryInitializers>
        <!-- Fully qualified type name, assembly name: -->
        <Add Type="MvcWebRole.Telemetry.MyTelemetryInitializer, MvcWebRole"/> 
        ...
      </TelemetryInitializers>
    </ApplicationInsights>

*Alternatively,* you can instantiate the initializer in code, for example in Global.aspx.cs:


```C#
    protected void Application_Start()
    {
        // ...
        TelemetryConfiguration.Active.TelemetryInitializers
        .Add(new MyTelemetryInitializer());
    }
```


[See more of this sample.](https://github.com/Microsoft/ApplicationInsights-Home/tree/master/Samples/AzureEmailService/MvcWebRole)

<a name="js-initializer"></a>
### JavaScript telemetry initializers

*JavaScript*

Insert a telemetry initializer immediately after the initialization code that you got from the portal: 

```JS

    <script type="text/javascript">
        // ... initialization code
        ...({
            instrumentationKey: "your instrumentation key"
        });
        window.appInsights = appInsights;


        // Adding telemetry initializer.
        // This is called whenever a new telemetry item
        // is created.

        appInsights.queue.push(function () {
            appInsights.context.addTelemetryInitializer(function (envelope) {
                var telemetryItem = envelope.data.baseData;

                // To check the telemetry item’s type - for example PageView:
                if (envelope.name == Microsoft.ApplicationInsights.Telemetry.PageView.envelopeType) {
                    // this statement removes url from all page view documents
                    telemetryItem.url = "URL CENSORED";
                }

                // To set custom properties:
                telemetryItem.properties = telemetryItem.properties || {};
                telemetryItem.properties["globalProperty"] = "boo";

                // To set custom metrics:
                telemetryItem.measurements = telemetryItem.measurements || {};
                telemetryItem.measurements["globalMetric"] = 100;
            });
        });

        // End of inserted code.

        appInsights.trackPageView();
    </script>
```

For a summary of the non-custom properties available on the telemetryItem, see the [data model](app-insights-export-data-model.md/#lttelemetrytypegt).

You can add as many initializers as you like. 


## Reference docs

* [API Overview](app-insights-api-custom-events-metrics.md)

* [ASP.NET reference](https://msdn.microsoft.com/library/dn817570.aspx)


## SDK Code

* [ASP.NET Core SDK](https://github.com/Microsoft/ApplicationInsights-dotnet)
* [ASP.NET 5](https://github.com/Microsoft/ApplicationInsights-aspnet5)
* [JavaScript SDK](https://github.com/Microsoft/ApplicationInsights-JS)


## <a name="next"></a>Next steps


[Search events and logs][diagnostic]

[Samples and walkthroughs](app-insights-code-samples.md)

[Troubleshooting][qna]


<!--Link references-->

[client]: app-insights-javascript.md
[config]: app-insights-configuration-with-applicationinsights-config.md
[create]: app-insights-create-new-resource.md
[data]: app-insights-data-retention-privacy.md
[diagnostic]: app-insights-diagnostic-search.md
[exceptions]: app-insights-asp-net-exceptions.md
[greenbrown]: app-insights-start-monitoring-app-health-usage.md
[java]: app-insights-java-get-started.md
[metrics]: app-insights-metrics-explorer.md
[qna]: app-insights-troubleshoot-faq.md
[trace]: app-insights-search-diagnostic-logs.md
[windows]: app-insights-windows-get-started.md

 