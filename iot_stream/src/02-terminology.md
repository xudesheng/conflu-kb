# Thingworx Core Terminology and Concepts

This chapter standardizes the vocabulary that appears throughout the book. Each term follows the language used in official ThingWorx IoT Streams documentation so that the remaining chapters can reference the concepts without redefining them.

## ThingWorx IoT Streams Capability

**ThingWorx IoT Streams** is the platform capability introduced in ThingWorx 10.0 that provides a consistent way to publish platform data to external message systems while hardening critical internal queues. It covers two complementary scenarios:
- **External Routing** – Streaming property updates, events, and custom payloads to external consumers
- **Durable Queues** – Persisting internal queues (events, persistent properties, logged properties) to an external broker for restart protection

The rest of the book distinguishes clearly between these two use cases.

## IoT Stream Entity

An **IoT Stream** is a first-class ThingWorx entity that represents an outbound data channel. Configuration happens in Composer (or via the REST Entity Services) and is stored in the model like any other entity.

Key configuration fields:
- **Queue Provider** – Reference to the provider entity that supplies the connection to the external broker
- **Queue Name** – Logical queue or topic name within the provider (Event Hub name, Kafka topic, etc.)
- **Stream Type** – Currently fixed to JSON payload delivery
- **Subscriptions** – Optional event subscriptions defined directly on the IoT Stream for custom publishing logic

Runtime behavior:
- Property updates marked as *Externally Routed* automatically publish JSON messages to the configured queue
- Custom payloads can be sent through the `WriteJSONToQueue` service exposed by the IoT Stream
- Event subscriptions execute in the stream’s context and can publish payloads even if the source Thing is not set to external routing

## Queue Provider Entity

A **Queue Provider** abstracts the connection details for a specific messaging infrastructure. There are two common deployment patterns:

### Internal Queue Provider
- Defined in `platform-settings.json`
- Required for enabling Durable Queues (Chapter 04)
- Can simultaneously serve External Routing workloads
- Typically points at a managed Azure Event Hub namespace or a Kafka cluster that the platform team controls

### Composer-Managed Provider
- Created as an entity inside Composer
- Useful when only External Routing is required
- Supports multiple queue names (topics/hubs) that can be associated with different IoT Streams
- Still relies on platform services for authentication, TLS, and retry policies

Regardless of how it is provisioned, a queue provider exposes the services `AddQueue`, `DeleteQueue`, `TestConnection`, and `GetDeclaredQueues`, which are reused in later chapters.

## Queue Name

A **Queue Name** is the logical destination inside the ThingWorx provider entity. In Azure Event Hub it maps to the Event Hub name; in Kafka it is the topic name. Each IoT Stream references exactly one queue. Multiple IoT Streams can share a queue if they intentionally publish to the same topic. Note that the Queue Name can be different to the broker's topic name (eg: Queue Name "oeeData" can be mapped to the physical topic "topicOee") but they can use the same name (eg: developers can choose to name the Queue "topicOee") . Think of Queue Names as a friendly display name for the broker's topic.

Queue names defined on the provider also capture the **partition key strategy** and optional retention or consumer configuration. Chapter 07 demonstrates how these configurations affect message distribution and consumer balancing.

## Partition Key Strategy

The partition key strategy controls how ThingWorx calculates the key used by the provider to distribute messages across partitions. Common strategies include:
- **ROUND_ROBIN** – Evenly distributes messages without inspecting the payload
- **SOURCE** – Uses the source entity name as the partition key
- **SOURCE_AND_NAME** – Combines the source entity and property/event name (default for telemetry workloads)
- **CUSTOM** – Defers to custom Java extensions for advanced routing rules

Understanding partition behavior is essential when sizing Event Hub partitions or Kafka partitions. See Chapter 09 for architecture guidance.

## External Routing Configuration

The property editor in Composer exposes an **Externally Routed** checkbox. Enabling it causes successful property updates to emit a JSON message through the assigned IoT Stream. The JSON schema is consistent across the platform:
```json
{
  "name": "propertyName",
  "source": "ThingName",
  "value": <any>,
  "timestamp": <number>,
  "quality": "GOOD|BAD|UNKNOWN"
}
```

Related elements:
- **Default IoT Stream** – One stream can be designated as the default for properties that do not specify a stream explicitly
- **WriteJSONToQueue** – Service on the IoT Stream for programmatic publishing (used heavily in Chapters 05 and 08)
- **External Routing Rules** – Metadata mappings that determine which IoT Stream a property uses when multiple streams are defined

## Durable Queues

**Durable Queues** extend internal ThingWorx queues to an external broker. Three queues are enabled individually:
1. `ptc.unordered-events` – Event Processing Queue
2. `ptc.persistent-properties` – Persistent Property updates
3. `ptc.logged-properties` – Logged Property (Value Stream) buffer

When durability is enabled, ThingWorx persists queued items to the provider before acknowledging the write. Both producers and consumers remain internal to ThingWorx; customers should not consume these queues directly. Chapter 04 walks through enablement and inspection strategies.

## Message Consumers

External systems that read IoT Stream data are referred to as **consumers**. The terminology of the underlying broker still applies—Kafka consumer groups, Event Hub consumer groups, offsets, checkpoints, and so forth. The book uses the following conventions:
- **Consumer** – A single process or application reading from a queue name
- **Consumer Group** – A coordinated set of consumers that share work on a queue name
- **Checkpoint** – Offset (Kafka) or sequence number (Event Hub) recorded to resume processing

Chapters 07 and 09 illustrate how consumer group design affects scaling and resilience.

## Supporting Concepts

- **Value-Time-Quality (VTQ)** – Internal representation used by ThingWorx for property values; discussed in Chapter 06 when comparing DataChange, HistoricalDataLogged, and IoT Stream behavior
- **ThingWorx Extension SDK** – The platform SDK leveraged to implement custom queue providers or advanced partition strategies
- **Thing Shapes and Thing Templates** – Model constructs for applying External Routing policies consistently; revisited in architecture examples

With these definitions in place, the remaining chapters can focus on implementation details without re-explaining core terminology.
