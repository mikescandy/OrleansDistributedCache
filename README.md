# MS Orleans as a distributed cache

- [MS Orleans as a distributed cache](#ms-orleans-as-a-distributed-cache)
  - [Introduction](#introduction)
  - [Getting Started](#getting-started)
  - [Performance assessment](#performance-assessment)

## Introduction

Microsoft Orleans has been deployed in our production environment for x months, but until now it was used for non time sensitive operations.

We decided to start using it as a distributed cache, and we had to make sure the performance profile would be adequate for the email risk score api, where we have strict SLA requirements around response time

In the current implementation, the cache backend is a traditional Redis cache, with a standalone service that periodically loads data from SqlServer and updates the cache. When loading data from Redis, the response time is typically between 2ms and 5ms. Therefore, in order to be a viable solution, our Orleans realtime cache should have similar or better performance.

## Getting Started

The first entity we decided to cache was the Client entity. The Client grain implements a simple interface

```csharp
    public interface IClientGrain : IGrainWithStringKey
    {
        Task<ClientModelV2> GetClientAsync(string clientId);
    }
```

The cache backend is Orleans itself, and it's implemented as a singleton service. It consists of a simple key-value store, and is updated periodically in the background.

```csharp
    public interface IClientService
    {
        Task Init(bool clientsFeatureEnabled);
        ClientModelV2 GetClient(string clientId);
    }
```

Note how the GetClient method is not async. This is because the implementation retrieves the client from an in memory dictionary, there's no I/O, and the code executes synchronously.

## Performance assessment

Orleans is a distributed system, and it's hard to measure the performance of a distributed system.

Our approach consisted in a production deployment, hidden behind a feature flag. The call to retrieve data from the Orleans silo would be executed only in specific conditions.
This allowed us to understand how the system performed in real world conditions.
Our initial findings were not promising. The system showed high latency and high jitter, which was unexpected and far from what we knew Orleans is capable of.

| ![Call duration from ERS to Orleans chart](/img/1.png) |
| :----------------------------------------------------: |
|    <b>Fig.1 - Call duration from ERS to Orleans</b>    |

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

This is a simple mechanism that allows us to retrieve the result from the fastest reply.
Quoting the "[Tail at scale](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext)" article from Google, "A simple way to curb latency variability is to issue the same request to multiple replicas and use the results from whichever replica responds first."

This proved to be effective in lowering latency and jitter, but wasn't still reliable enough to be used as a realtime dependency.

We widened our analysis to the Dotnet performance metrics of the Orleans containers. Using the data collected by Datadog, we noticed that the Contention Time metric, which measures the cumulated time spent by threads waiting on a lock, was very high.
This is often correlated with long Garbage Collection pauses, which indicated a possible memory issue.
The Orleans cluster was scaling out to more than 100 containers, which should have been more than enough to handle our traffic volume, but was still struggling.
Our approach was to scale up the containers (from 256mb to 1gb of ram) and scale in the deployment (down to a minimum of 4 containers). This removed memory pressure from the Orleans cluster which started to perform well, handling the same production traffic as before, but with way better metrics and overall less resources.
After this change, we went back to look at the performance of the realtime calls, which now showed a better profile: latency went down, and the hedged calls showed remarkable stability.
