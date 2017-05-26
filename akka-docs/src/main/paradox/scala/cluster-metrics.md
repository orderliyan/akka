# Cluster Metrics Extension

## Introduction

The member nodes of the cluster can collect system health metrics and publish that to other cluster nodes
and to the registered subscribers on the system event bus with the help of Cluster Metrics Extension.

Cluster metrics information is primarily used for load-balancing routers,
and can also be used to implement advanced metrics-based node life cycles,
such as "Node Let-it-crash" when CPU steal time becomes excessive.

Cluster Metrics Extension is a separate Akka module delivered in `akka-cluster-metrics` jar.

To enable usage of the extension you need to add the following dependency to your project:
:

@@@vars
```
"com.typesafe.akka" % "akka-cluster-metrics_$scala.binary_version$" % "$akka.version$"
```
@@@

and add the following configuration stanza to your `application.conf`
:

```
akka.extensions = [ "akka.cluster.metrics.ClusterMetricsExtension" ]
```

Cluster members with status @ref:[WeaklyUp](cluster-usage.md#weakly-up), if that feature is enabled,
will participate in Cluster Metrics collection and dissemination.

## Metrics Collector

Metrics collection is delegated to an implementation of `akka.cluster.metrics.MetricsCollector`.

Different collector implementations provide different subsets of metrics published to the cluster.
Certain message routing and let-it-crash functions may not work when Sigar is not provisioned.

Cluster metrics extension comes with two built-in collector implementations:

 1. `akka.cluster.metrics.SigarMetricsCollector`, which requires Sigar provisioning, and is more rich/precise
 2. `akka.cluster.metrics.JmxMetricsCollector`, which is used as fall back, and is less rich/precise

You can also plug-in your own metrics collector implementation.

By default, metrics extension will use collector provider fall back and will try to load them in this order:

 1. configured user-provided collector
 2. built-in `akka.cluster.metrics.SigarMetricsCollector`
 3. and finally `akka.cluster.metrics.JmxMetricsCollector`

## Metrics Events

Metrics extension periodically publishes current snapshot of the cluster metrics to the node system event bus.

The publication interval is controlled by the `akka.cluster.metrics.collector.sample-interval` setting.

The payload of the `akka.cluster.metrics.ClusterMetricsChanged` event will contain
latest metrics of the node as well as other cluster member nodes metrics gossip
which was received during the collector sample interval.

You can subscribe your metrics listener actors to these events in order to implement custom node lifecycle
:

```
ClusterMetricsExtension(system).subscribe(metricsListenerActor)
```

## Hyperic Sigar Provisioning

Both user-provided and built-in metrics collectors can optionally use [Hyperic Sigar](http://www.hyperic.com/products/sigar)
for a wider and more accurate range of metrics compared to what can be retrieved from ordinary JMX MBeans.

Sigar is using a native o/s library, and requires library provisioning, i.e.
deployment, extraction and loading of the o/s native library into JVM at runtime.

User can provision Sigar classes and native library in one of the following ways:

 1. Use [Kamon sigar-loader](https://github.com/kamon-io/sigar-loader) as a project dependency for the user project.
Metrics extension will extract and load sigar library on demand with help of Kamon sigar provisioner.
 2. Use [Kamon sigar-loader](https://github.com/kamon-io/sigar-loader) as java agent: `java -javaagent:/path/to/sigar-loader.jar`.
Kamon sigar loader agent will extract and load sigar library during JVM start.
 3. Place `sigar.jar` on the `classpath` and Sigar native library for the o/s on the `java.library.path`.
User is required to manage both project dependency and library deployment manually.

@@@ warning

When using [Kamon sigar-loader](https://github.com/kamon-io/sigar-loader) and running multiple
instances of the same application on the same host, you have to make sure that sigar library is extracted to a
unique per instance directory. You can control the extract directory with the
`akka.cluster.metrics.native-library-extract-folder` configuration setting.

@@@

To enable usage of Sigar you can add the following dependency to the user project
:

@@@vars
```
"io.kamon" % "sigar-loader" % "$sigar_loader.version$"
```
@@@

You can download Kamon sigar-loader from [Maven Central](http://search.maven.org/#search%7Cga%7C1%7Csigar-loader)

## Adaptive Load Balancing

The `AdaptiveLoadBalancingPool` / `AdaptiveLoadBalancingGroup` performs load balancing of messages to cluster nodes based on the cluster metrics data.
It uses random selection of routees with probabilities derived from the remaining capacity of the corresponding node.
It can be configured to use a specific MetricsSelector to produce the probabilities, a.k.a. weights:

 * `heap` / `HeapMetricsSelector` - Used and max JVM heap memory. Weights based on remaining heap capacity; (max - used) / max
 * `load` / `SystemLoadAverageMetricsSelector` - System load average for the past 1 minute, corresponding value can be found in `top` of Linux systems. The system is possibly nearing a bottleneck if the system load average is nearing number of cpus/cores. Weights based on remaining load capacity; 1 - (load / processors)
 * `cpu` / `CpuMetricsSelector` - CPU utilization in percentage, sum of User + Sys + Nice + Wait. Weights based on remaining cpu capacity; 1 - utilization
 * `mix` / `MixMetricsSelector` - Combines heap, cpu and load. Weights based on mean of remaining capacity of the combined selectors.
 * Any custom implementation of `akka.cluster.metrics.MetricsSelector`

The collected metrics values are smoothed with [exponential weighted moving average](http://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average). In the @ref:[Cluster configuration](cluster-usage.md#cluster-configuration) you can adjust how quickly past data is decayed compared to new data.

Let's take a look at this router in action. What can be more demanding than calculating factorials?

The backend worker that performs the factorial calculation:

@@snip [FactorialBackend.scala]($code$/scala/docs/cluster/FactorialBackend.scala) { #backend }

The frontend that receives user jobs and delegates to the backends via the router:

@@snip [FactorialFrontend.scala]($code$/scala/docs/cluster/FactorialFrontend.scala) { #frontend }

As you can see, the router is defined in the same way as other routers, and in this case it is configured as follows:

```
akka.actor.deployment {
  /factorialFrontend/factorialBackendRouter = {
    # Router type provided by metrics extension.
    router = cluster-metrics-adaptive-group
    # Router parameter specific for metrics extension.
    # metrics-selector = heap
    # metrics-selector = load
    # metrics-selector = cpu
    metrics-selector = mix
    #
    routees.paths = ["/user/factorialBackend"]
    cluster {
      enabled = on
      use-role = backend
      allow-local-routees = off
    }
  }
}
```

It is only `router` type and the `metrics-selector` parameter that is specific to this router,
other things work in the same way as other routers.

The same type of router could also have been defined in code:

@@snip [FactorialFrontend.scala]($code$/scala/docs/cluster/FactorialFrontend.scala) { #router-lookup-in-code }

@@snip [FactorialFrontend.scala]($code$/scala/docs/cluster/FactorialFrontend.scala) { #router-deploy-in-code }

The easiest way to run **Adaptive Load Balancing** example yourself is to download the ready to run
@extref[Akka Cluster Sample with Scala](ecs:akka-samples-cluster-scala)
together with the tutorial. It contains instructions on how to run the **Adaptive Load Balancing** sample.
The source code of this sample can be found in the @extref[Akka Samples Repository](samples:akka-sample-cluster-scala).

## Subscribe to Metrics Events

It is possible to subscribe to the metrics events directly to implement other functionality.

@@snip [MetricsListener.scala]($code$/scala/docs/cluster/MetricsListener.scala) { #metrics-listener }

## Custom Metrics Collector

Metrics collection is delegated to the implementation of `akka.cluster.metrics.MetricsCollector`

You can plug-in your own metrics collector instead of built-in
`akka.cluster.metrics.SigarMetricsCollector` or `akka.cluster.metrics.JmxMetricsCollector`.

Look at those two implementations for inspiration.

Custom metrics collector implementation class must be specified in the
`akka.cluster.metrics.collector.provider` configuration property.

## Configuration

The Cluster metrics extension can be configured with the following properties:

@@snip [reference.conf]($akka$/akka-cluster-metrics/src/main/resources/reference.conf)