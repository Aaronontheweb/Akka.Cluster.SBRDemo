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
 
 
