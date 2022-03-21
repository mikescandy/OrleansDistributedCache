# MS Orleans as a distributed cache

- [MS Orleans as a distributed cache](#ms-orleans-as-a-distributed-cache)
  - [Introduction](#introduction)
  - [Getting Started](#getting-started)
  - [Performance assessment](#performance-assessment)
  - [Lessons learned](#lessons-learned)

## Introduction

Microsoft Orleans has been deployed in our production environment for x months, but until now it was used for non time sensitive operations.

We decided to start using it as a distributed cache, and we had to make sure the performance profile would be adequate for the email risk score api, where we have strict SLA requirements around response time.

In the current implementation, the cache backend is a traditional Redis cache, with a standalone service that periodically loads data from SqlServer and updates the cache. When loading data from Redis, the response time is typically between 2ms and 5ms. Therefore, in order to be a viable solution, our Orleans realtime cache should have similar or better performance.

## Getting Started

The first entity we decided to cache was the Client entity. The Client grain implements a simple interface

```csharp
    public interface IClientGrain : IGrainWithStringKey
    {
        Task<ClientModelV2> GetClientAsync(string clientId);
    }
```

The cache backend run in Orleans itself, and it's implemented as a singleton service. It consists of a simple key-value store, and is updated periodically in the background.

```csharp
    public interface IClientService
    {
        Task Init(bool clientsFeatureEnabled);
        ClientModelV2 GetClient(string clientId);
    }
```

Note how the GetClient method is not async. This is because the implementation retrieves the client from an in memory dictionary, there's no I/O, and the code executes synchronously.

The IClientService implementation has a timer that updates the cache periodically. The timer is configured to run every 5 minutes.

The overall implementation is simple and straightforward. It's the kind of code that is easy to understand and trivial to debug.

## Performance assessment

Orleans is a distributed system, and it's hard to measure the performance of a distributed system.

Our approach consisted in a production deployment, hidden behind a feature flag. The call to retrieve data from the Orleans silo would be executed only in specific conditions.
This allowed us to understand how the system performed in real world conditions without affecting our customers.

The call duration is measured using a [GrainFilter](https://dotnet.github.io/orleans/docs/grains/interceptors.html), which is a filter that is applied to the grain calls. It works in the same way as a middleware in an ASP.NET pipeline, and can be executed on the client or on the server. We focused more on the client side, because it's the one that is most likely to affect the performance.

The initial findings were not promising. The system showed high latency and high jitter, which was unexpected and far from what we knew Orleans is capable of.

| ![Call duration from ERS to Orleans chart](/img/1.png) |
| :----------------------------------------------------: |
| <b>Img.1 - Call duration from ERS to Orleans in ms</b> |

To mitigate the issue, we introduced a simple hedging mechanism: we would make three calls to Orleans and retrieve the result from the fastest reply.

```csharp
var tasks = new List<Task>();
for (var grainIndex = 0; grainIndex < 3; grainIndex++)
{
    var internalGrainIndex = grainIndex;
    var grainPoolIndex = randomGrainIndex + internalGrainIndex;
    var grainRef = _clusterClient.GetGrain<IClientGrain>($"{clientId}_{grainPoolIndex}");
    var grainTask = Task.Run(async () =>
    {
        _client_ = await grainRef.GetClientAsync(clientId, grainCancellationTokenSource.Token);
    });

    tasks.Add(grainTask);
}

// Wait for the first task to complete
completedTask = await Task.WhenAny(tasks);
```

Quoting the "[Tail at scale](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext)" article from Google, "A simple way to curb latency variability is to issue the same request to multiple replicas and use the results from whichever replica responds first."

This proved to be effective in lowering latency and jitter, but wasn't still reliable enough to be used as a realtime dependency.

We widened our analysis to the dotnet performance metrics of the Orleans containers. Using the data collected by Datadog, we noticed that the Contention Time metric, which measures the cumulated time spent by threads waiting on a lock, was very high.

|    ![Contention chart](/img/3.png)    |
| :-----------------------------------: |
| <b>Img.2 - dotnet contention time</b> |

This is often correlated with long Garbage Collection pauses, which indicated a possible memory issue.

The Orleans cluster was scaling out to more than 100 containers, which should have been more than enough to handle our traffic volume, but was still struggling.

Our approach was to scale up the containers (from 256mb to 1gb of ram) and scale in the deployment (down to a minimum of 4 containers). This removed memory pressure from the Orleans cluster which started to perform well, handling the same production traffic as before, but with way better metrics and overall less resources.
After this change, we went back to look at the performance of the realtime calls, which now showed a better profile: latency went down, and the hedged calls showed remarkable stability.
If we look back at image 1 and 2, we can see when we deployed in production the new container configuration (Feb 25th).

## Lessons learned

Having great monitoring tools in place was key to understanding the performance of the Orleans cluster. The combination of metrics, logs and traces allowed us to understand the behavior of the system, and tools like Datadog and Splunk were able to provide insights into the system. Especially usuful was the ability to visualize and correlate different metrics at the same time.

This will be the base of our future work: understaing the key metrics of the Orleans cluster will allow us to build a reliable alerting system that will be able to react to performance issues and possibly anticipate them.

While investigating the memory issue, we learned about modern dotnet performance analysis tools. Commands like dotnet-counter, dotnet-dump and dotnet-trace are great tools to understand the behavior of a dotnet application, although the learning curve is high. [This MSDN guide](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/debug-memory-leak) is a good starting point.

|          ![Dotnet counters](/img/4.png)           |
| :-----------------------------------------------: |
| <b>Img.3 - A sample output of dotnet-counters</b> |

For more in depth analysis, tools like dotMemory are great, but they are only available on Windows.
