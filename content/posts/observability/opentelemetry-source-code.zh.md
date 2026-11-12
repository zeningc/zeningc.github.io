---
title: "理解 OpenTelemetry Collector：源码导读（第一部分）"
date: '2026-04-02T00:00:00-07:00'
draft: false
---

> 这个周末花了些时间让 Claude 带我读 Observe-Agent 仓库，它是在原版 OpenTelemetry Collector 基础上扩展而来的。我弄清楚了 OpenTelemetry 的一些核心概念，想在这里分享一下。

## 基本概念
OpenTelemetry Collector 是一个可以运行在多种宿主环境中的代理（Kubernetes 是常见的一种）。它从宿主系统收集遥测数据，并转发给不同的数据接入后端（Jaeger、Prometheus 等），用于可观测性。它收集以下几类遥测数据（称为 Signal 类型）：
1. Metric（CPU 使用率等）
2. Trace（API 调用、Span）
3. Log  

## 流水线（Pipeline）
为了用更通用的接口来处理这些遥测数据，OTel 为整条流水线定义了以下几个阶段：
1. Receiver（接收器）：主机指标、应用数据等会被发送给接收器并由它消费，例如 heartbeatreceiver、otel receiver、host metrics 等……
2. Processor（处理器）：对接收器收集到的数据进行处理的步骤，例如 batch、redaction、transform，这些都可以叠加组合
3. Exporter（导出器）：把数据发送到接入后端的步骤，例如可以把数据发到 Kafka 供后续接入；又或者，如果代理以 DaemonSet 的形式运行在 Pod 里，可能会先把数据发给集群内的某个 Service，用于尾部采样（tail sampling）等目的。

### 配置示例
一个配置示例如下：
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
可以看到，这些流水线非常灵活：
1. pipelines 定义了消费数据并最终导出到某处的各条流水线，名字遵循 `<signal>/<name>` 的格式，有时也可以只写 `<signal>`
2. 一条流水线可以有多个接收器：由于一条流水线只处理一种 signal 类型，所有输入数据都实现同一个接口
3. 一条流水线可以有多个处理器，并按顺序执行，例如 `memory_limiter` 在 `batch` 之前，意味着数据先做内存限制，再做批处理
4. 一条流水线可以有多个导出器，从而把数据导出到不同的地方
5. 不同类型的流水线可以相互连接，例如 `traces/main` 导出到 `spanmetrics`，而 `metrics/from-spans` 把它作为输入

## 源码实现
> 注意：我是通过 observe-agent 仓库阅读代码的，所以这里的实现不一定是上游 OTel 的最新版本，其中也有一些 Observe 特有的扩展

### 配置加载
> 配置部分不是我关注的重点，这里只简单介绍
1. Observe 构建的外层封装基于 Cobra。在 `root.go` 中，它用 viper 读取配置文件并设置环境变量。
2. 当 `start` 命令被触发时，viper 的状态会被反序列化成 `AgentConfig` 结构体，再序列化为 YAML 并做 base64 编码，写入环境变量（`OBSERVE_AGENT_CONFIG`、`OBSERVE_AGENT_OTEL_CONFIG`）。之后 heartbeat receiver 会读取它们，把配置快照上报给 Observe。
3. OTel 配置的生成方式是：把 Go 模板（通过 `embed.FS` 打包进二进制文件）渲染成磁盘上的临时文件。随后 OTel 的 `confmap.Resolver` 把这些配置文件合并起来，交给 collector。

## 构建 DAG 图
### 工厂绑定
OTel 中有很多组件：Receiver、Processor、Exporter 等。在 `components.go`（一个自动生成的文件）中，定义了组件名到对应 Factory 函数的映射，例如：
``` go
	factories.Extensions, err = otelcol.MakeFactoryMap[extension.Factory](
		zpagesextension.NewFactory(),
		cgroupruntimeextension.NewFactory(),
		healthcheckextension.NewFactory(),
		filestorage.NewFactory(),
		pprofextension.NewFactory(),
	)
```
它定义在 `components.go::components()` 中，`GetOtelCollector` 构建 `CollectorSettings` 时，会通过 `generateCollectorSettings` 引用它。
接着在 `Run` 中，`setupConfigurationComponents` 把各个组件的 Config 和 Factory 打包成一个 Service：
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

### 构建 DAG
DAG 的构建发生在 `service.New` 中。它调用 `graph.Build`，这个函数负责构建完整的流水线图。
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
#### 初始化流水线结构
代码中，`simple.NewDirectedGraph()` 返回一个维护以下内容的数据结构：
``` go
// DirectedGraph implements a generalized directed graph.
type DirectedGraph struct {
	nodes map[int64]graph.Node
	from  map[int64]map[int64]graph.Edge
	to    map[int64]map[int64]graph.Edge

	nodeIDs *uid.Set
}
```

这个 for 循环为每条流水线初始化接收器和导出器的映射表：
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
首先创建 3 个局部映射表，用来追踪连接器（connector，即把一种 signal 类型的导出器连接到另一种 signal 类型的接收器的组件）

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
然后用一个 for 循环填充各组件的映射表，把组件 id 映射到图中的节点

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
最后，对连接器做一些校验，并创建连接器节点

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
这部分很直接：边按照 `receiver -> capabilitiesNode -> processor(s) -> fanOutNode -> exporter(s)` 的顺序建立

#### 拓扑排序
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
先做拓扑排序，再逆序遍历结果（先导出器，最后接收器）。这样每个组件初始化时，它下游的消费者都已经构建好，可以作为 `nextConsumers` 传进去。

`nextConsumer` 是通过工厂接上的。例如，`graph.Build` 调用 `builder.CreateLogs(ctx, set, nextConsumer)`，它会调用接收器工厂的 `createLog` 函数。这个函数再调用 `r.Unwrap().registerLogsConsumer(consumer)`，把下一个消费者保存在接收器实例上。这里有一个细节：由于同一个 OTLP 接收器要同时服务 traces、metrics 和 logs，`sharedcomponent` 映射表保证每份配置只创建一个 `otlpReceiver` 实例，`registerTraceConsumer`/`registerMetricsConsumer`/`registerLogsConsumer` 会分别在这个共享实例上调用。

接好之后，每个组件都可以通过如下调用把数据推给下游：
```go
err := r.nextConsumer.ConsumeLogs(ctx, logs)
```