# Orleans Streaming with Azure EventHub (.NET8 Client-Server)

## 1. Create an EventHub in Azure

## 2. Create a Table and Storage Account in Azure

We can see the overview storage account

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/86438bb1-52ca-48cb-8e2f-7c4193d8ffff)

We copy the 

## 3. Create a blank solution in Visual Studio 2022 Community Edition


## 4. Create the Server(Silo) console project

We load the project dependencies

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/0a5a3e27-74b4-4eb7-a715-c2c6141ba573)

We input the code


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


## 7. Create the Grains


## 8. Create the Secrets.json file

```json
{
  "EventHubConnectionString": "",
  "DataConnectionString": ""
}
```

We place the **Secrets.json** file in SiloHost and Client projects root. See the following picture

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/faa81924-fd07-462e-a3ce-169405875b2a)

We copy the StorageAccount connection string and paste in the Secrest.json file in the DataConnectionString field

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/b58fd1ae-0f48-4a00-8ce4-81b723cdf802)

We also copy the EventHub connection string and paste in the Secrest.json file in the EventHubConnectionString

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/4c112638-e803-4631-8b9e-49e33c158e69)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/88314520-ccb8-4339-a926-2841da9c32fa)

## 9. Run and Test the solution

We right click on the solution name and we select the menu option "**Set the startup projects**" 

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/d0985adc-d76b-4a84-8cee-f9b8a44b7852)

We set the SiloHost and Client as startup projects

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/09deaae0-0097-42e3-87e8-ae76b2b2a8ed)

We build and run the solution. Two console windows are automatically are opened and the data is interchanged between the Server and the Client

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/8afa8591-3e8d-4d68-91f5-573bca5c382d)

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/299e7c4d-904a-40a3-bf87-a174fbbdcc5f)

