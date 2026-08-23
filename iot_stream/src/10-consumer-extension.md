# ThingWorx Consumer Extension Sample

This appendix describes the sample extension included under `project/10-consumer-extension`. The extension demonstrates how to implement a dedicated consumer for IoT Stream messages inside ThingWorx when external infrastructure is not permitted, or when platform administrators prefer to keep the entire integration in the ThingWorx execution environment.

## Extension Responsibilities

The sample extension focuses on three responsibilities:
- **Register a Queue Provider adapter** that understands the same queue names configured for IoT Streams
- **Create a background service** that consumes messages from the external broker and surfaces them as ThingWorx events, InfoTables, or custom services
- **Expose administration services** so application developers can monitor consumer status without leaving Composer

The implementation relies on the ThingWorx Extension SDK and mirrors the terminology established in Chapter 02.

## Key Services and Shapes

| Artifact | Purpose |
|----------|---------|
| `ConsumerManagerThing` | Central Thing created from the extension; coordinates connections to Azure Event Hub or Kafka |
| `StartConsumer` / `StopConsumer` | Runtime services for enabling or disabling a consumer group |
| `GetConsumerStatus` | Returns InfoTable with partition assignments, last offset, and lag metrics |
| `MessageReceived` Event | Fires when new IoT Stream messages arrive; downstream Things can subscribe and process the payload |

Payloads use the same JSON schema as native IoT Streams (`name`, `source`, `value`, `timestamp`, `quality`), ensuring parity with the external routing behavior covered in Chapters 05 through 08.

## Packaging and Deployment

1. Build the extension by running `gradlew build` in `project/10-consumer-extension`
2. Upload the generated `.zip` package through Composer → **Import/Export** → **Import**
3. After import, instantiate `ConsumerManagerThing` and configure:
   - **Queue Provider** – Reference to the provider entity used by the relevant IoT Stream
   - **Queue Name** – The Event Hub or topic the consumer should read
   - **Consumer Group** – Group identifier that the extension will manage internally
4. Invoke `StartConsumer` to begin processing. Messages can now be handled via subscriptions or custom mashups.

## Operational Guidance

- Use `GetConsumerStatus` to monitor lag and confirm partition assignments align with expectations from Chapter 09
- When testing new payload schemas, configure a dedicated consumer group to avoid disrupting production flows
- The extension is intended for light to medium throughput scenarios. For high-volume analytics pipelines, continue to favor external microservices as described in Chapters 07 and 08

This sample provides a starting point for teams that want to keep a portion of the consumption logic within ThingWorx while still leveraging the standardized IoT Stream payload and terminology introduced earlier in the book.
