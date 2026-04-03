---
title: "Understanding OpenTelemetry Collector: A Source Code Walkthrough"
date: 2026-04-02
---

> Spent some time asking Claude to walk me through the Observe-Agent repo this weekend, which is extending the original OpenTelemetry Collector. I managed to figure out some core concepts of OpenTelemetry and wanted to share them here.

## Basic Concept
OpenTelemetry Collector is an agent that can run in multiple host environments (Kubernetes is a common one). It gathers telemetry data from the host system and forwards it to different ingestion backends (Jaeger, Prometheus, etc.) for observability. It gathers the following telemetry data (referred to as Signal types):
1. Metric(CPU Usage and so on)
2. Trace(API Call, Span)
3. Log  

## Pipeline
To populate this telemetry data with a more general interface, otel has defined the following stages for the whole pipeline:
1. Receiver: Host Metric, Application Data, etc will be sent to and consumed by the receiver, e.g. heartbeatreceiver, otel receiver, host metrics, etc...
2. Processor: processing steps for the data collected by receiver, e.g. batch, redaction, transform, all these could be stacked together
3. Exporter: steps that send the data to the ingestion backend, e.g. this data can be sent to kafka for later ingestion, or, if the agent is running as DaemonSet inside a Pod, it might want to send it to a Service inside the cluster first for purposes like tail sampling.

### Example Configuration
Example configuration can be as follows:
``` yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  filelog/app:
    include: [/var/log/app/*.log]

processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 75
    spike_limit_percentage: 15

  batch:
    send_batch_size: 1024
    timeout: 5s

  resource/prod:
    attributes:
      - key: deployment.environment
        value: prod
        action: upsert

  transform/redact:
    error_mode: ignore
    log_statements:
      - context: log
        statements:
          - replace_pattern(body, "password=[^&\\s]+", "password=***")

exporters:
  otlp/jaeger:
    endpoint: jaeger-otlp-grpc.observability.svc.cluster.local:4317
    tls:
      insecure: true

  otlp/vendor:
    endpoint: vendor-otel-gateway.example.com:4317
    headers:
      api-key: ${VENDOR_API_KEY}

  debug:
    verbosity: basic

  prometheus:
    endpoint: 0.0.0.0:8889

  loki:
    endpoint: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push

connectors:
  spanmetrics: {}

service:
  pipelines:
    traces/main:
      receivers: [otlp]
      processors: [memory_limiter, resource/prod, batch]
      exporters: [otlp/jaeger, otlp/vendor, spanmetrics]

    traces/debug:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]

    metrics/from-spans:
      receivers: [spanmetrics]
      processors: [batch]
      exporters: [prometheus]

    logs/app:
      receivers: [otlp, filelog/app]
      processors: [memory_limiter, transform/redact, batch]
      exporters: [loki, debug]
```
As you can see, these pipelines are really flexible:
1. pipelines define pipelines that consume the data and finally export it somewhere, its name following `<signal>/<name>`, sometime we can refer it as just `<signal>`
2. a pipeline can have multiple receivers, since a pipeline is defined as only processing 1 signal type, all input data are implementing the same interface
3. there can be multiple processors for a pipeline, and they are run in order, e.g. `memory_limiter` before `batch` means data is memory-limited first before being batched
4. there can be multiple exporter for a pipeline so that the data can be exported to different places
5. different types of pipelines can be connected, e.g. `traces/main` export to `spanmetrics` and `metrics/from-spans` treats that as input

## Source Code Implement
> NOTE: I read the code through the observe-agent repo, so the implementation may not reflect the absolute latest upstream OTel and there are some Observe-specific additions as well

### Config Population
> The config part was not my focus, so I'd just cover it briefly
1. The wrapper built by Observe is based on Cobra. Inside `root.go`, it used viper to read config files and set up env vars.
2. Then when the `start` command is triggered, the viper state is deserialized into the `AgentConfig` struct, which is marshalled into YAML and base64-encoded into env vars (`OBSERVE_AGENT_CONFIG`, `OBSERVE_AGENT_OTEL_CONFIG`) — these are later read by the heartbeat receiver to report config snapshots back to Observe.
3. The OTel config is populated by rendering Go templates (bundled inside the binary via `embed.FS`) into temporary files on disk. OTel's `confmap.Resolver` then merges all those config files together and hands the result to the collector.

## DAG Graph Building
### Factory Binding
There are a lot of components in OTel: Receiver, Processor, Exporter, etc. Inside `components.go` (a generated file), the map of component names to their Factory functions is defined, e.g.
``` go
	factories.Extensions, err = otelcol.MakeFactoryMap[extension.Factory](
		zpagesextension.NewFactory(),
		cgroupruntimeextension.NewFactory(),
		healthcheckextension.NewFactory(),
		filestorage.NewFactory(),
		pprofextension.NewFactory(),
	)
```
This is defined inside `components.go::components()`, and is referenced by `generateCollectorSettings` when building `CollectorSettings` inside `GetOtelCollector`.
Then inside `Run`, there's a `setupConfigurationComponents` that wraps different component's Configs and Factories into a Service:
``` go
	col.service, err = service.New(ctx, service.Settings{
		BuildInfo:     col.set.BuildInfo,
		CollectorConf: conf,

		ReceiversConfigs:    cfg.Receivers,
		ReceiversFactories:  factories.Receivers,
		ProcessorsConfigs:   cfg.Processors,
		ProcessorsFactories: factories.Processors,
		ExportersConfigs:    cfg.Exporters,
		ExportersFactories:  factories.Exporters,
		ConnectorsConfigs:   cfg.Connectors,
		ConnectorsFactories: factories.Connectors,
		ExtensionsConfigs:   cfg.Extensions,
		ExtensionsFactories: factories.Extensions,

		ModuleInfos: service.ModuleInfos{
			Receiver:  buildModuleInfo(factories.ReceiverModules),
			Processor: buildModuleInfo(factories.ProcessorModules),
			Exporter:  buildModuleInfo(factories.ExporterModules),
			Extension: buildModuleInfo(factories.ExtensionModules),
			Connector: buildModuleInfo(factories.ConnectorModules),
		},
		AsyncErrorChannel: col.asyncErrorChannel,
		BuildZapLogger:    buildZapLogger,
		TelemetryFactory:  factories.Telemetry,
	}, cfg.Service)
```

### DAG Building
The DAG building happens inside `service.New`. It calls `graph.Build`, which is the function responsible for constructing the full pipeline graph.
``` go
// Build builds a full pipeline graph.
// Build also validates the configuration of the pipelines and does the actual initialization of each Component in the Graph.
func Build(ctx context.Context, set Settings) (*Graph, error) {
	pipelines := &Graph{
		componentGraph: simple.NewDirectedGraph(),
		pipelines:      make(map[pipeline.ID]*pipelineNodes, len(set.PipelineConfigs)),
		instanceIDs:    make(map[int64]*componentstatus.InstanceID),
		telemetry:      set.Telemetry,
	}
	for pipelineID := range set.PipelineConfigs {
		pipelines.pipelines[pipelineID] = &pipelineNodes{
			receivers: make(map[int64]graph.Node),
			exporters: make(map[int64]graph.Node),
		}
	}
	if err := pipelines.createNodes(set); err != nil {
		return nil, err
	}
	pipelines.createEdges()
	err := pipelines.buildComponents(ctx, set)
	return pipelines, err
}
```
#### Init Pipeline Structure
Inside the code, `simple.NewDirectedGraph()` returns a data structure that maintains:
``` go
// DirectedGraph implements a generalized directed graph.
type DirectedGraph struct {
	nodes map[int64]graph.Node
	from  map[int64]map[int64]graph.Edge
	to    map[int64]map[int64]graph.Edge

	nodeIDs *uid.Set
}
```

This for loop initialize a receiver and exporter map for each pipeline:
``` go
	for pipelineID := range set.PipelineConfigs {
		pipelines.pipelines[pipelineID] = &pipelineNodes{
			receivers: make(map[int64]graph.Node),
			exporters: make(map[int64]graph.Node),
		}
	}
```

#### createNodes
``` go
	// Build a list of all connectors for easy reference.
	connectors := make(map[component.ID]struct{})

	// Keep track of connectors and where they are used. (map[connectorID][]pipelineID).
	connectorsAsExporter := make(map[component.ID][]pipeline.ID)
	connectorsAsReceiver := make(map[component.ID][]pipeline.ID)
```
3 local maps are created first to track connectors (components that connect one signal type's exporter to another signal type's receiver)

``` go

	// Build each pipelineNodes struct for each pipeline by parsing the pipelineCfg.
	// Also populates the connectors, connectorsAsExporter and connectorsAsReceiver maps.
	for pipelineID, pipelineCfg := range set.PipelineConfigs {
		pipe := g.pipelines[pipelineID]
		for _, recvID := range pipelineCfg.Receivers {
			// Checks if this receiver is a connector or a regular receiver.
			if set.ConnectorBuilder.IsConfigured(recvID) {
				connectors[recvID] = struct{}{}
				connectorsAsReceiver[recvID] = append(connectorsAsReceiver[recvID], pipelineID)
				continue
			}
			rcvrNode := g.createReceiver(pipelineID, recvID)
			pipe.receivers[rcvrNode.ID()] = rcvrNode
		}

		pipe.capabilitiesNode = newCapabilitiesNode(pipelineID)

		for _, procID := range pipelineCfg.Processors {
			procNode := g.createProcessor(pipelineID, procID)
			pipe.processors = append(pipe.processors, procNode)
		}

		pipe.fanOutNode = newFanOutNode(pipelineID)

		for _, exprID := range pipelineCfg.Exporters {
			if set.ConnectorBuilder.IsConfigured(exprID) {
				connectors[exprID] = struct{}{}
				connectorsAsExporter[exprID] = append(connectorsAsExporter[exprID], pipelineID)
				continue
			}
			expNode := g.createExporter(pipelineID, exprID)
			pipe.exporters[expNode.ID()] = expNode
		}
	}
```
Then a for loop is used to fill in the maps for components that map the component id to the node

``` go
	for connID := range connectors {
		factory := set.ConnectorBuilder.Factory(connID.Type())
		if factory == nil {
			return fmt.Errorf("connector factory not available for: %q", connID.Type())
		}
		connFactory := factory.(connector.Factory)

		expTypes := make(map[pipeline.Signal]bool)
		for _, pipelineID := range connectorsAsExporter[connID] {
			// The presence of each key indicates how the connector is used as an exporter.
			// The value is initially set to false. Later we will set the value to true *if* we
			// confirm that there is a supported corresponding use as a receiver.
			expTypes[pipelineID.Signal()] = false
		}
		recTypes := make(map[pipeline.Signal]bool)
		for _, pipelineID := range connectorsAsReceiver[connID] {
			// The presence of each key indicates how the connector is used as a receiver.
			// The value is initially set to false. Later we will set the value to true *if* we
			// confirm that there is a supported corresponding use as an exporter.
			recTypes[pipelineID.Signal()] = false
		}

		for expType := range expTypes {
			for recType := range recTypes {
				// Typechecks the connector's receiving and exporting datatypes.
				if connectorStability(connFactory, expType, recType) != component.StabilityLevelUndefined {
					expTypes[expType] = true
					recTypes[recType] = true
				}
			}
		}

		for expType, supportedUse := range expTypes {
			if supportedUse {
				continue
			}
			return fmt.Errorf("connector %q used as exporter in %v pipeline but not used in any supported receiver pipeline", connID, formatPipelineNamesWithSignal(connectorsAsExporter[connID], expType))
		}
		for recType, supportedUse := range recTypes {
			if supportedUse {
				continue
			}
			return fmt.Errorf("connector %q used as receiver in %v pipeline but not used in any supported exporter pipeline", connID, formatPipelineNamesWithSignal(connectorsAsReceiver[connID], recType))
		}

		for _, eID := range connectorsAsExporter[connID] {
			for _, rID := range connectorsAsReceiver[connID] {
				if connectorStability(connFactory, eID.Signal(), rID.Signal()) == component.StabilityLevelUndefined {
					// Connector is not supported for this combination, but we know it is used correctly elsewhere
					continue
				}
				connNode := g.createConnector(eID, rID, connID)

				g.pipelines[eID].exporters[connNode.ID()] = connNode
				g.pipelines[rID].receivers[connNode.ID()] = connNode
			}
		}
	}
	return nil
```
At the end, we do some validation on the connectors and create connector nodes

#### createEdges
``` go
func (g *Graph) createEdges() {
	for _, pg := range g.pipelines {
		// Draw edges from each receiver to the capability node.
		for _, receiver := range pg.receivers {
			g.componentGraph.SetEdge(g.componentGraph.NewEdge(receiver, pg.capabilitiesNode))
		}

		// Iterates through processors, chaining them together.  starts with the capabilities node.
		var from, to graph.Node
		from = pg.capabilitiesNode
		for _, processor := range pg.processors {
			to = processor
			g.componentGraph.SetEdge(g.componentGraph.NewEdge(from, to))
			from = processor
		}
		// Always inserts a fanout node before any exporters. If there is only one
		// exporter, the fanout node is still created and acts as a noop.
		to = pg.fanOutNode
		g.componentGraph.SetEdge(g.componentGraph.NewEdge(from, to))

		for _, exporter := range pg.exporters {
			g.componentGraph.SetEdge(g.componentGraph.NewEdge(pg.fanOutNode, exporter))
		}
	}
}
```
This is straightforward: edges are built as `receiver -> capabilitiesNode -> processor(s) -> fanOutNode -> exporter(s)`

#### Topological Sort
```
func (g *Graph) buildComponents(ctx context.Context, set Settings) error {
	nodes, err := topo.Sort(g.componentGraph)
	if err != nil {
		return cycleErr(err, topo.DirectedCyclesIn(g.componentGraph))
	}

	for i := len(nodes) - 1; i >= 0; i-- {
		node := nodes[i]

		switch n := node.(type) {
		case *receiverNode:
			err = n.buildComponent(ctx, set.Telemetry, set.BuildInfo, set.ReceiverBuilder, g.nextConsumers(n.ID()))
		case *processorNode:
			// nextConsumers is guaranteed to be length 1.  Either it is the next processor or it is the fanout node for the exporters.
			err = n.buildComponent(ctx, set.Telemetry, set.BuildInfo, set.ProcessorBuilder, g.nextConsumers(n.ID())[0])
		case *exporterNode:
			err = n.buildComponent(ctx, set.Telemetry, set.BuildInfo, set.ExporterBuilder)
		case *connectorNode:
			err = n.buildComponent(ctx, set.Telemetry, set.BuildInfo, set.ConnectorBuilder, g.nextConsumers(n.ID()))
		case *capabilitiesNode:
			capability := consumer.Capabilities{
				// The fanOutNode represents the aggregate capabilities of the exporters in the pipeline.
				MutatesData: g.pipelines[n.pipelineID].fanOutNode.getConsumer().Capabilities().MutatesData,
			}
			for _, proc := range g.pipelines[n.pipelineID].processors {
				capability.MutatesData = capability.MutatesData || proc.(*processorNode).getConsumer().Capabilities().MutatesData
			}
			next := g.nextConsumers(n.ID())[0]
			switch n.pipelineID.Signal() {
			case pipeline.SignalTraces:
				cc := capabilityconsumer.NewTraces(next.(consumer.Traces), capability)
				n.baseConsumer = cc
				n.ConsumeTracesFunc = cc.ConsumeTraces
			case pipeline.SignalMetrics:
				cc := capabilityconsumer.NewMetrics(next.(consumer.Metrics), capability)
				n.baseConsumer = cc
				n.ConsumeMetricsFunc = cc.ConsumeMetrics
			case pipeline.SignalLogs:
				cc := capabilityconsumer.NewLogs(next.(consumer.Logs), capability)
				n.baseConsumer = cc
				n.ConsumeLogsFunc = cc.ConsumeLogs
			case xpipeline.SignalProfiles:
				cc := capabilityconsumer.NewProfiles(next.(xconsumer.Profiles), capability)
				n.baseConsumer = cc
				n.ConsumeProfilesFunc = cc.ConsumeProfiles
			}
		case *fanOutNode:
			nexts := g.nextConsumers(n.ID())
			switch n.pipelineID.Signal() {
			case pipeline.SignalTraces:
				consumers := make([]consumer.Traces, 0, len(nexts))
				for _, next := range nexts {
					consumers = append(consumers, next.(consumer.Traces))
				}
				n.baseConsumer = fanoutconsumer.NewTraces(consumers)
			case pipeline.SignalMetrics:
				consumers := make([]consumer.Metrics, 0, len(nexts))
				for _, next := range nexts {
					consumers = append(consumers, next.(consumer.Metrics))
				}
				n.baseConsumer = fanoutconsumer.NewMetrics(consumers)
			case pipeline.SignalLogs:
				consumers := make([]consumer.Logs, 0, len(nexts))
				for _, next := range nexts {
					consumers = append(consumers, next.(consumer.Logs))
				}
				n.baseConsumer = fanoutconsumer.NewLogs(consumers)
			case xpipeline.SignalProfiles:
				consumers := make([]xconsumer.Profiles, 0, len(nexts))
				for _, next := range nexts {
					consumers = append(consumers, next.(xconsumer.Profiles))
				}
				n.baseConsumer = fanoutconsumer.NewProfiles(consumers)
			}
		}
		if err != nil {
			return err
		}
	}
	return nil
}
```
We do a topological sort and iterate the result in reverse (exporters first, receivers last), so that when each component is initialized, its downstream consumers are already built and can be passed in as `nextConsumers`.

The `nextConsumer` is wired in through the factory. For example, `graph.Build` calls `builder.CreateLogs(ctx, set, nextConsumer)`, which invokes the receiver factory's `createLog` function. That function calls `r.Unwrap().registerLogsConsumer(consumer)` to store the next consumer on the receiver instance. One subtlety here: since a single OTLP receiver serves traces, metrics, and logs, a `sharedcomponent` map ensures only one `otlpReceiver` instance is created per config, and `registerTraceConsumer`/`registerMetricsConsumer`/`registerLogsConsumer` are each called separately on the same shared instance.

Once wired, each component can push data downstream by calling:
```go
err := r.nextConsumer.ConsumeLogs(ctx, logs)
```