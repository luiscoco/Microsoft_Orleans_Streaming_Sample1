# Orleans Streaming with Azure EventHub (.NET8 Client-Server)

## 1. Create an EventHub in Azure

We log in to Azure Portal and we search for **EventHub** service and we create a new one

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/d5a752cf-160c-4709-969e-c7b3218edde0)

We press the Create **Event hub namespace** button

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/b4445b14-473f-4abb-a107-16e1c9948a46)

We input the required data: Subscription, ResourceGroup, EventHubNameSpace, Location and Pricing Tier (we select Standart to enable the option for creating a new **ConsumerGroup**)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/a0474e36-4a2c-40fd-9303-2a5b147cfa04)

We press the **Review + Create** button and then we press the **Create** button

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/fc3f970d-e8f0-47c5-9c95-1d0ec970cc57)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/95972a83-b79e-4b48-a2c1-0292a1b9f8a9)

We press the **Go to resource** button

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/5ffec69a-07bb-4f02-b0e6-b848f28b9bf1)

We can see the new **EventHubNameSpace** in the EventHubs list

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/956dd138-7451-4456-a2e5-53d133fe9f3c)

We can **create an EventHub** inside the EventHubNameSapce

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/17c7043e-913c-45bb-ac9c-a5ad1f9e5a92)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/6ca52243-498d-4fdd-9b92-dedd81c230b1)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/2c3d7437-f325-41ea-bbe6-eaf5c3eecf62)

We can also create and see the **new Consumer Group**

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/34fe89c6-27bb-490c-b7b0-3e93859fca3d)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/d24cb038-8050-477f-ac9e-bb3fed6effc0)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/1516df7a-481b-4c7b-93b6-fabd3b78408a)

## 2. Create a Table and Storage Account in Azure

We create a resource and select Storage and then Storage Account

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/8a6b4948-3b26-4a33-be6c-2df2b1e3e218)

We input the required data: 

We can see the overview storage account

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/86438bb1-52ca-48cb-8e2f-7c4193d8ffff)

We copy the 

## 3. Create a blank solution in Visual Studio 2022 Community Edition

We run Visual Studio 2022 and we create a new project

We select the project template **blank solution**

We input the solution name and location

We create five projects inside the solution: **Client** (Console project), **Common** (Class library project), **GrainInterfaces** (Class library project), **Grains** (Class library project), **SiloHost** (Console project)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/022d2ede-7e6f-4348-be5f-47141f59a7a8)

## 4. Create the Server(Silo) console project

We load the project dependencies

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/057102e5-36b7-4dda-895c-f34402ca6de1)

We input the code

**Program.cs**

```csharp
using Common;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

try
{
    var host = new HostBuilder()
        .UseOrleans(ConfigureSilo)
        .ConfigureLogging(logging => logging.AddConsole())
        .Build();

    await host.RunAsync();

    return 0;
}
catch (Exception ex)
{
    Console.Error.WriteLine(ex);
    return 1;
}

static void ConfigureSilo(HostBuilderContext context, ISiloBuilder siloBuilder)
{
    var secrets = Secrets.LoadFromFile()!;
    siloBuilder
        .UseLocalhostClustering(serviceId: Constants.ServiceId, clusterId: Constants.ClusterId)
        .AddAzureTableGrainStorage(
            "PubSubStore",
            options => options.ConfigureTableServiceClient(secrets.DataConnectionString))
        .AddEventHubStreams(Constants.StreamProvider, (ISiloEventHubStreamConfigurator configurator) =>
        {
            configurator.ConfigureEventHub(builder => builder.Configure(options =>
            {
                options.ConfigureEventHubConnection(
                    secrets.EventHubConnectionString,
                    Constants.EHPath,
                    Constants.EHConsumerGroup);
            }));
            configurator.UseAzureTableCheckpointer(
                builder => builder.Configure(options =>
            {
                options.ConfigureTableServiceClient(secrets.DataConnectionString);
                options.PersistInterval = TimeSpan.FromSeconds(10);
            }));
        });
}
```

## 5. Create the Client application console project

We load the project dependencies

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/0b40de04-dae8-4b28-bffd-06bdf0dadd97)

We input the code

**Program.cs**

```csharp
using GrainInterfaces;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Hosting;
using Orleans.Streams;
using Common;
using Microsoft.Extensions.DependencyInjection;
using Orleans.Runtime;

internal class Program
{
    private static async Task<int> Main(string[] args)
    {
        const int maxAttempts = 5;
        var attempt = 0;

        try
        {
            var secrets = Secrets.LoadFromFile()!;
            var host = new HostBuilder()
                .ConfigureLogging(logging => logging.AddConsole())
                .UseOrleansClient((context, client) =>
                {
                    client
                        .UseLocalhostClustering(serviceId: Constants.ServiceId, clusterId: Constants.ClusterId)
                        .UseConnectionRetryFilter(RetryFilter)
                        .AddEventHubStreams(
                            Constants.StreamProvider,
                            (configurator) => configurator.ConfigureEventHub(
                                builder => builder.Configure(options =>
                                {
                                    options.ConfigureEventHubConnection(
                                        secrets.EventHubConnectionString,
                                        Constants.EHPath,
                                        Constants.EHConsumerGroup);
                                })));
                })
                .Build();

            await host.StartAsync();
            Console.WriteLine("Client successfully connect to silo host");

            var client = host.Services.GetRequiredService<IClusterClient>();

            // Use the connected client to ask a grain to start producing events
            var key = Guid.NewGuid();
            var producer = client.GetGrain<IProducerGrain>("my-producer");
            await producer.StartProducing(Constants.StreamNamespace, key);

            // Now you should see that a consumer grain was activated on the silo, and is logging when it is receiving event

            // Client can also subscribe to streams
            var streamId = StreamId.Create(Constants.StreamNamespace, key);
            var stream = client
                .GetStreamProvider(Constants.StreamProvider)
                .GetStream<int>(streamId);
            await stream.SubscribeAsync(OnNextAsync);
            
            // Now the client will also log received events

            await Task.Delay(TimeSpan.FromSeconds(15));

            // Stop producing
            await producer.StopProducing();

            Console.ReadKey();
            return 0;
        }
        catch (Exception e)
        {
            Console.WriteLine(e);
            Console.ReadKey();
            return 1;
        }

        static Task OnNextAsync(int item, StreamSequenceToken? token = null)
        {
            Console.WriteLine("OnNextAsync: item: {0}, token = {1}", item, token);
            return Task.CompletedTask;
        }

        async Task<bool> RetryFilter(Exception exception, CancellationToken cancellationToken)
        {
            attempt++;
            Console.WriteLine($"Cluster client attempt {attempt} of {maxAttempts} failed to connect to cluster.  Exception: {exception}");
            if (attempt > maxAttempts)
            {
                return false;
            }
            await Task.Delay(TimeSpan.FromSeconds(4), cancellationToken);
            return true;
        }
    }
}
```

## 6. Create the GrainIntefaces 

This is the project files and dependencies structure

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/c6ff87cd-c1ec-4a86-abb8-62db86df7140)

We input the code

**IConsumerGrain.cs**

```csharp
namespace GrainInterfaces;

public interface IConsumerGrain : IGrainWithGuidKey
{
}
```

**IProducerGrain.cs**

```csharp
namespace GrainInterfaces;

public interface IProducerGrain : IGrainWithStringKey
{
    Task StartProducing(string ns, Guid key);

    Task StopProducing();
}
```

## 7. Create the Grains

This is the Grains files and dependencies structure

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/930179a6-de19-42fd-9067-bc861914ed68)

We input the code in the files

**ConsumerGrain.cs**

```csharp
using Common;
using GrainInterfaces;
using Microsoft.Extensions.Logging;
using Orleans.Streams;
using Orleans.Streams.Core;

namespace Grains;

// ImplicitStreamSubscription attribute here is to subscribe implicitely to all stream within
// a given namespace: whenever some data is pushed to the streams of namespace Constants.StreamNamespace,
// a grain of type ConsumerGrain with the same guid of the stream will receive the message.
// Even if no activations of the grain currently exist, the runtime will automatically
// create a new one and send the message to it.
[ImplicitStreamSubscription(Constants.StreamNamespace)]
public class ConsumerGrain : Grain, IConsumerGrain, IStreamSubscriptionObserver
{
    private readonly ILogger<IConsumerGrain> _logger;
    private readonly LoggerObserver _observer;

    /// <summary>
    /// Class that will log streaming events
    /// </summary>
    private class LoggerObserver : IAsyncObserver<int>
    {
        private readonly ILogger<IConsumerGrain> _logger;

        public LoggerObserver(ILogger<IConsumerGrain> logger)
        {
            _logger = logger;
        }

        public Task OnCompletedAsync()
        {
            _logger.LogInformation("OnCompletedAsync");
            return Task.CompletedTask;
        }

        public Task OnErrorAsync(Exception ex)
        {
            _logger.LogInformation("OnErrorAsync: {Exception}", ex);
            return Task.CompletedTask;
        }

        public Task OnNextAsync(int item, StreamSequenceToken? token = null)
        {
            _logger.LogInformation("OnNextAsync: item: {Item}, token = {Token}", item, token);
            return Task.CompletedTask;
        }
    }

    public ConsumerGrain(ILogger<IConsumerGrain> logger)
    {
        _logger = logger;
        _observer = new LoggerObserver(_logger);
    }

    // Called when a subscription is added
    public async Task OnSubscribed(IStreamSubscriptionHandleFactory handleFactory)
    {
        // Plug our LoggerObserver to the stream
        var handle = handleFactory.Create<int>();
        await handle.ResumeAsync(_observer);
    }


    public override Task OnActivateAsync(CancellationToken token)
    {
        _logger.LogInformation("OnActivateAsync");
        return Task.CompletedTask;
    }
}
```

**ProducerGrain.cs**

```csharp
using Common;
using GrainInterfaces;
using Microsoft.Extensions.Logging;
using Orleans.Runtime;
using Orleans.Streams;

namespace Grains;

public class ProducerGrain : Grain, IProducerGrain
{
    private readonly ILogger<IProducerGrain> _logger;

    private IAsyncStream<int>? _stream;
    private IDisposable? _timer;

    private int _counter = 0;

    public ProducerGrain(ILogger<IProducerGrain> logger)
    {
        _logger = logger;
    }

    public Task StartProducing(string ns, Guid key)
    {
        if (_timer is not null)
            throw new Exception("This grain is already producing events");

        // Get the stream
        var streamId = StreamId.Create(ns, key);
        _stream = this.GetStreamProvider(Constants.StreamProvider)
            .GetStream<int>(streamId);

        // Register a timer that produce an event every second
        var period = TimeSpan.FromSeconds(1);
        _timer = RegisterTimer(TimerTick, null, period, period);

        _logger.LogInformation("I will produce a new event every {Period}", period);

        return Task.CompletedTask;
    }

    private async Task TimerTick(object _)
    {
        var value = _counter++;
        _logger.LogInformation("Sending event {EventNumber}", value);
        if (_stream is not null)
        {
            await _stream.OnNextAsync(value);
        }
    }

    public Task StopProducing()
    {
        if (_timer is not null)
        {
            _timer.Dispose();
            _timer = null;
        }

        if (_stream is not null)
        {
            _stream = null;
        }

        return Task.CompletedTask;
    }
}
```

## 8. Create the Common project for storing Constants and Secrets

This is the project files and dependencies

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/24e16dda-15ce-4809-81bd-0bf3eaea8623)


**Constants.cs**

```csharp
namespace Common;

public static class Constants
{
    //Orleans data (local cluster) for connecting Silo and Client

    public const string ServiceId = "streaming-sample"; // Unique identifier for your Orleans service

    public const string ClusterId = "dev"; // Identifier for the cluster of silos in your Orleans application

    public const string StreamProvider = "my-stream-provider"; // Name of the stream provider configured in your Orleans application



    public const string StreamNamespace = "myOrleansStreamEventHubNameSpace"; // EventHub Namespace name

    public const string EHConsumerGroup = "myorleansconsumergroup"; // EventHub consumer group

    public const string EHPath = "myOrleansStreamEventHub"; // EventHub name
}
```

**Secrets.cs**

```csharp
using System.Text.Json;

namespace Common;

public class Secrets
{
    public string DataConnectionString { get; set; } = null!;

    public string EventHubConnectionString { get; set; } = null!;

    internal Secrets()
    {
    }

    public Secrets(string dataConnectionString, string eventHubConnectionString)
    {
        DataConnectionString = dataConnectionString
            ?? throw new ArgumentException(
                "Must provide a dataConnectionString", nameof(dataConnectionString));
        EventHubConnectionString = eventHubConnectionString
            ?? throw new ArgumentException(
                "Must provide an eventHubConnectionString", nameof(eventHubConnectionString));
    }

    public static Secrets? LoadFromFile(string filename = "Secrets.json")
    {
        var currentDir = new DirectoryInfo(Directory.GetCurrentDirectory());
        while (currentDir != null && currentDir.Exists)
        {
            var filePath = Path.Combine(currentDir.FullName, filename);
            if (File.Exists(filePath))
            {
                return JsonSerializer.Deserialize<Secrets>(File.ReadAllText(filePath));
            }

            currentDir = currentDir.Parent;
        }
        throw new FileNotFoundException($"Cannot find file {filename}");
    }
}
```

## 9. Create the Secrets.json file

```json
{
  "EventHubConnectionString": "",
  "DataConnectionString": ""
}
```

We place the **Secrets.json** file in SiloHost and Client projects root. See the following picture

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/faa81924-fd07-462e-a3ce-169405875b2a)

We copy the **StorageAccount connection string** and paste in the Secrest.json file in the DataConnectionString field

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/b58fd1ae-0f48-4a00-8ce4-81b723cdf802)

We also copy the **EventHub connection string** and paste in the Secrest.json file in the EventHubConnectionString

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/4c112638-e803-4631-8b9e-49e33c158e69)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/88314520-ccb8-4339-a926-2841da9c32fa)

## 10. Run and Test the solution

We right click on the solution name and we select the menu option "**Set the startup projects**" 

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/d0985adc-d76b-4a84-8cee-f9b8a44b7852)

We set the **SiloHost** and **Client** as **startup projects**

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/09deaae0-0097-42e3-87e8-ae76b2b2a8ed)

We build and run the solution. Two console windows are automatically are opened and the data is interchanged between the Server and the Client

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/8afa8591-3e8d-4d68-91f5-573bca5c382d)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/299e7c4d-904a-40a3-bf87-a174fbbdcc5f)

