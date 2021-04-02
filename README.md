# Akka.Cluster.SBRDemo
 Akka.Cluster.SBR vs. Akka.Cluster SplitBrainResolver demo - from [Akka.NET v1.4.17](https://github.com/akkadotnet/akka.net/releases/tag/1.4.17) and later
 
 This repository demonstrates the lack of robustness of the old [Akka.NET cluster split brain resolvers](https://getakka.net/articles/clustering/split-brain-resolver.html) compared to the new ones introduced into Akka.NET in v1.4.17.
 
 ## Running with Classic SBRs
 So the classic SBRs, which have been a part of Akka.NET for years - let's run the classic "keep majority" SBR strategy, which we can configure in [Lighthouse](https://github.com/petabridge/lighthouse) via [Akka.Bootstrap.Docker's environment variable substitution](https://github.com/petabridge/akkadotnet-bootstrap/tree/dev/src/Akka.Bootstrap.Docker#using-environment-variables-to-configure-akkanet):
 
 ```yml
 spec:
  serviceName: lighthouse
  replicas: 2
  selector:
    matchLabels:
      app: lighthouse
  template:
    metadata:
      labels:
        app: lighthouse
    spec:
      terminationGracePeriodSeconds: 35
      containers:
      - name: lighthouse
        image: petabridge/lighthouse:1.5.1
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "pbm 127.0.0.1:9110 cluster leave"]
        env:
        - name: ACTORSYSTEM
          value: AkkaTrader
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: CLUSTER_IP
          value: "$(POD_NAME).lighthouse"
        - name: CLUSTER_SEEDS
          value: akka.tcp://$(ACTORSYSTEM)@lighthouse-0.lighthouse:4053,akka.tcp://$(ACTORSYSTEM)@lighthouse-1.lighthouse:4053,akka.tcp://$(ACTORSYSTEM)@lighthouse-2.lighthouse:4053
        - name: AKKA__CLUSTER__DOWNING_PROVIDER_CLASS
          value: "Akka.Cluster.SplitBrainResolver, Akka.Cluster"
        - name: AKKA__CLUSTER__SPLIT_BRAIN_RESOLVER__ACTIVE_STRATEGY
          value: "keep-majority"
        livenessProbe:
          tcpSocket:
            port: 4053
        ports:
        - containerPort: 4053
          protocol: TCP
 ```
 
 Let's see what type of Akka.NET cluster forms when we run the following command:
 
 ```
 kubectl apply -f ./lighthouse-oldsbr.yml
 ```
 
 Using some [`Petabridge.Cmd`](https://cmd.petabridge.com/) CLI commands to inspect the state of the cluster, here is what we see immediately after initial formation on node 1:
 
 ```
PS>  kubectl -n lightouse-sbr1 exec lighthouse-0 -- pbm cluster show
akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053 | [lighthouse] | up |  | 1.4.0
Count: 1 nodes
 ```

And then on node 2:

```
PS>  kubectl -n lightouse-sbr1 exec lighthouse-1 -- pbm cluster show
akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053 | [lighthouse] | up |  | 1.4.0
Count: 1 nodes
```

So what we have here is a split brain. How did that happen?

Logs from node 1:

```
[Docker-Bootstrap] IP=lighthouse-0.lighthouse

[Docker-Bootstrap] PORT=4053

[Docker-Bootstrap] SEEDS=["akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053","akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053","akka.tcp://AkkaTrader@lighthouse-2.lighthouse:4053"]

[Lighthouse] ActorSystem: AkkaTrader; IP: lighthouse-0.lighthouse; PORT: 4053

[Lighthouse] Performing pre-boot sanity check. Should be able to parse address [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053]

[Lighthouse] Parse successful.

[INFO][04/02/2021 17:25:27][Thread 0001][remoting (akka://AkkaTrader)] Starting remoting

[INFO][04/02/2021 17:25:27][Thread 0001][remoting (akka://AkkaTrader)] Remoting started; listening on addresses : [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053]

[INFO][04/02/2021 17:25:27][Thread 0001][remoting (akka://AkkaTrader)] Remoting now listens on addresses: [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053]

[INFO][04/02/2021 17:25:27][Thread 0001][Cluster (akka://AkkaTrader)] Cluster Node [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053] - Starting up...

[INFO][04/02/2021 17:25:27][Thread 0001][Cluster (akka://AkkaTrader)] Cluster Node [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053] - Started up successfully

Press Control + C to terminate.

[INFO][04/02/2021 17:25:27][Thread 0019][akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053/user/petabridge.cmd] petabridge.cmd host bound to [0.0.0.0:9110]

[WARNING][04/02/2021 17:25:27][Thread 0013][akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053/system/endpointManager/reliableEndpointWriter-akka.tcp%3A%2F%2FAkkaTrader%40lighthouse-2.lighthouse%3A4053-2/endpointWriter] AssociationError [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053] -> akka.tcp://AkkaTrader@lighthouse-2.lighthouse:4053: Error [Association failed with akka.tcp://AkkaTrader@lighthouse-2.lighthouse:4053] []

[WARNING][04/02/2021 17:25:27][Thread 0012][akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053/system/endpointManager/reliableEndpointWriter-akka.tcp%3A%2F%2FAkkaTrader%40lighthouse-1.lighthouse%3A4053-1/endpointWriter] AssociationError [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053] -> akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053: Error [Association failed with akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053] []

[WARNING][04/02/2021 17:25:27][Thread 0012][remoting (akka://AkkaTrader)] Tried to associate with unreachable remote address [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053]. Address is now gated for 5000 ms, all messages to this address will be delivered to dead letters. Reason: [Association failed with akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053] Caused by: [System.AggregateException: One or more errors occurred. (Name or service not known)

---> System.Net.Internals.SocketExceptionFactory+ExtendedSocketException (00000005, 0xFFFDFFFF): Name or service not known

at System.Net.Dns.InternalGetHostByName(String hostName)

at System.Net.Dns.ResolveCallback(Object context)

--- End of stack trace from previous location where exception was thrown ---

at System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw(Exception source)

at System.Net.Dns.HostResolutionEndHelper(IAsyncResult asyncResult)

at System.Net.Dns.EndGetHostEntry(IAsyncResult asyncResult)

at System.Net.Dns.<>c.<GetHostEntryAsync>b__27_1(IAsyncResult asyncResult)

at System.Threading.Tasks.TaskFactory`1.FromAsyncCoreLogic(IAsyncResult iar, Func`2 endFunction, Action`1 endAction, Task`1 promise, Boolean requiresSynchronization)

--- End of stack trace from previous location where exception was thrown ---

at Akka.Remote.Transport.DotNetty.DotNettyTransport.ResolveNameAsync(DnsEndPoint address, AddressFamily addressFamily)

at Akka.Remote.Transport.DotNetty.DotNettyTransport.DnsToIPEndpoint(DnsEndPoint dns)

at Akka.Remote.Transport.DotNetty.TcpTransport.MapEndpointAsync(EndPoint socketAddress)

at Akka.Remote.Transport.DotNetty.TcpTransport.AssociateInternal(Address remoteAddress)

at Akka.Remote.Transport.DotNetty.DotNettyTransport.Associate(Address remoteAddress)

--- End of inner exception stack trace ---

at System.Threading.Tasks.Task`1.GetResultCore(Boolean waitCompletionNotification)

at System.Threading.Tasks.Task`1.get_Result()

at Akka.Remote.Transport.ProtocolStateActor.<>c.<InitializeFSM>b__12_18(Task`1 result)

at System.Threading.Tasks.ContinuationResultTaskFromResultTask`2.InnerInvoke()

at System.Threading.Tasks.Task.<>c.<.cctor>b__274_0(Object obj)

at System.Threading.ExecutionContext.RunInternal(ExecutionContext executionContext, ContextCallback callback, Object state)

--- End of stack trace from previous location where exception was thrown ---

at System.Threading.ExecutionContext.RunInternal(ExecutionContext executionContext, ContextCallback callback, Object state)

at System.Threading.Tasks.Task.ExecuteWithThreadLocal(Task& currentTaskSlot, Thread threadPoolThread)]

[WARNING][04/02/2021 17:25:27][Thread 0012][remoting (akka://AkkaTrader)] Tried to associate with unreachable remote address [akka.tcp://AkkaTrader@lighthouse-2.lighthouse:4053]. Address is now gated for 5000 ms, all messages to this address will be delivered to dead letters. Reason: [Association failed with akka.tcp://AkkaTrader@lighthouse-2.lighthouse:4053] Caused by: [System.AggregateException: One or more errors occurred. (Name or service not known)

---> System.Net.Internals.SocketExceptionFactory+ExtendedSocketException (00000005, 0xFFFDFFFF): Name or service not known

at System.Net.Dns.InternalGetHostByName(String hostName)

at System.Net.Dns.ResolveCallback(Object context)

--- End of stack trace from previous location where exception was thrown ---

at System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw(Exception source)

at System.Net.Dns.HostResolutionEndHelper(IAsyncResult asyncResult)

at System.Net.Dns.EndGetHostEntry(IAsyncResult asyncResult)

at System.Net.Dns.<>c.<GetHostEntryAsync>b__27_1(IAsyncResult asyncResult)

at System.Threading.Tasks.TaskFactory`1.FromAsyncCoreLogic(IAsyncResult iar, Func`2 endFunction, Action`1 endAction, Task`1 promise, Boolean requiresSynchronization)

--- End of stack trace from previous location where exception was thrown ---

at Akka.Remote.Transport.DotNetty.DotNettyTransport.ResolveNameAsync(DnsEndPoint address, AddressFamily addressFamily)

at Akka.Remote.Transport.DotNetty.DotNettyTransport.DnsToIPEndpoint(DnsEndPoint dns)

at Akka.Remote.Transport.DotNetty.TcpTransport.MapEndpointAsync(EndPoint socketAddress)

at Akka.Remote.Transport.DotNetty.TcpTransport.AssociateInternal(Address remoteAddress)

at Akka.Remote.Transport.DotNetty.DotNettyTransport.Associate(Address remoteAddress)

--- End of inner exception stack trace ---

at System.Threading.Tasks.Task`1.GetResultCore(Boolean waitCompletionNotification)

at System.Threading.Tasks.Task`1.get_Result()

at Akka.Remote.Transport.ProtocolStateActor.<>c.<InitializeFSM>b__12_18(Task`1 result)

at System.Threading.Tasks.ContinuationResultTaskFromResultTask`2.InnerInvoke()

at System.Threading.Tasks.Task.<>c.<.cctor>b__274_0(Object obj)

at System.Threading.ExecutionContext.RunInternal(ExecutionContext executionContext, ContextCallback callback, Object state)

--- End of stack trace from previous location where exception was thrown ---

at System.Threading.ExecutionContext.RunInternal(ExecutionContext executionContext, ContextCallback callback, Object state)

at System.Threading.Tasks.Task.ExecuteWithThreadLocal(Task& currentTaskSlot, Thread threadPoolThread)]

[INFO][04/02/2021 17:25:27][Thread 0004][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [1] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:27][Thread 0004][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [2] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:27][Thread 0011][akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053/system/endpointManager/reliableEndpointWriter-akka.tcp%3A%2F%2FAkkaTrader%40lighthouse-2.lighthouse%3A4053-2] Removing receive buffers for [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053]->[akka.tcp://AkkaTrader@lighthouse-2.lighthouse:4053]

[INFO][04/02/2021 17:25:27][Thread 0013][akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053/system/endpointManager/reliableEndpointWriter-akka.tcp%3A%2F%2FAkkaTrader%40lighthouse-1.lighthouse%3A4053-1] Removing receive buffers for [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053]->[akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053]

[INFO][04/02/2021 17:25:28][Thread 0004][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [3] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:28][Thread 0004][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [4] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:29][Thread 0009][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [5] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:29][Thread 0009][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [6] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:30][Thread 0009][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [7] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:30][Thread 0009][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [8] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:31][Thread 0009][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [9] dead letters encountered. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:31][Thread 0009][akka://AkkaTrader/deadLetters] Message [InitJoin] from akka://AkkaTrader/system/cluster/core/daemon/firstSeedNodeProcess-1 to akka://AkkaTrader/deadLetters was not delivered. [10] dead letters encountered, no more dead letters will be logged in next [00:05:00]. If this is not an expected behavior then akka://AkkaTrader/deadLetters may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

[INFO][04/02/2021 17:25:32][Thread 0004][Cluster (akka://AkkaTrader)] Cluster Node [1.4.0] - Node [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053] is JOINING itself (with roles [lighthouse], version [1.4.0]) and forming a new cluster

[INFO][04/02/2021 17:25:32][Thread 0004][Cluster (akka://AkkaTrader)] Cluster Node [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053] - is the new leader among reachable nodes (more leaders may exist)

[INFO][04/02/2021 17:25:32][Thread 0004][Cluster (akka://AkkaTrader)] Cluster Node [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053] - Leader is moving node [akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053] to [Up]

[INFO][04/02/2021 17:26:02][Thread 0022][akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053/user/petabridge.cmd] Received petabridge.cmd connection from client [127.0.0.1:34872]

[INFO][04/02/2021 17:26:02][Thread 0009][akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053/user/petabridge.cmd/127.0.0.1%3A34872] petabridge.cmd client disconnected. Shutting down...

[INFO][04/02/2021 17:26:02][Thread 0022][akka.tcp://AkkaTrader@lighthouse-0.lighthouse:4053/user/petabridge.cmd/127.0.0.1%3A34872] Terminating petabridge.cmd connection to client.
```

`lighthouse-2` was never started under this configuration, but it looks like we failed to connect to node 1 and never bothered after that - instead forming a new cluster. As the first node in the `seed-nodes` list, this is the intended behavior - it's supposed to form a new cluster with itself.

Now what does node 2's logs look like?

```
[Docker-Bootstrap] IP=lighthouse-1.lighthouse

[Docker-Bootstrap] PORT=4053

[Docker-Bootstrap] SEEDS=[]

[Lighthouse] ActorSystem: AkkaTrader; IP: lighthouse-1.lighthouse; PORT: 4053

[Lighthouse] Performing pre-boot sanity check. Should be able to parse address [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053]

[Lighthouse] Parse successful.

[INFO][04/02/2021 17:34:21][Thread 0001][remoting (akka://AkkaTrader)] Starting remoting

[INFO][04/02/2021 17:34:22][Thread 0001][remoting (akka://AkkaTrader)] Remoting started; listening on addresses : [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053]

[INFO][04/02/2021 17:34:22][Thread 0001][remoting (akka://AkkaTrader)] Remoting now listens on addresses: [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053]

[INFO][04/02/2021 17:34:22][Thread 0001][Cluster (akka://AkkaTrader)] Cluster Node [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053] - Starting up...

[INFO][04/02/2021 17:34:22][Thread 0001][Cluster (akka://AkkaTrader)] Cluster Node [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053] - Started up successfully

Press Control + C to terminate.

[INFO][04/02/2021 17:34:22][Thread 0009][akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053/system/cluster/core/daemon/downingProvider] SBR started. Config: strategy [KeepMajority], stable-after [00:00:20], down-all-when-unstable [00:00:15], selfUniqueAddress [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053].

[INFO][04/02/2021 17:34:22][Thread 0004][akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053/user/petabridge.cmd] petabridge.cmd host bound to [0.0.0.0:9110]

[INFO][04/02/2021 17:34:22][Thread 0014][Cluster (akka://AkkaTrader)] Cluster Node [1.4.0] - Node [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053] is JOINING itself (with roles [lighthouse], version [1.4.0]) and forming a new cluster

[INFO][04/02/2021 17:34:22][Thread 0014][Cluster (akka://AkkaTrader)] Cluster Node [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053] - is the new leader among reachable nodes (more leaders may exist)

[INFO][04/02/2021 17:34:22][Thread 0014][Cluster (akka://AkkaTrader)] Cluster Node [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053] - Leader is moving node [akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053] to [Up]

[INFO][04/02/2021 17:34:22][Thread 0014][akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053/system/cluster/core/daemon/downingProvider] This node is now the leader responsible for taking SBR decisions among the reachable nodes (more leaders may exist).

[INFO][04/02/2021 17:34:51][Thread 0004][akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053/user/petabridge.cmd] Received petabridge.cmd connection from client [127.0.0.1:39200]

[INFO][04/02/2021 17:34:51][Thread 0024][akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053/user/petabridge.cmd/127.0.0.1%3A39200] petabridge.cmd client disconnected. Shutting down...

[INFO][04/02/2021 17:34:51][Thread 0024][akka.tcp://AkkaTrader@lighthouse-1.lighthouse:4053/user/petabridge.cmd/127.0.0.1%3A39200] Terminating petabridge.cmd connection to client.
```

Notice that the `SEEDS` environment variable is empty - that's a problem.
