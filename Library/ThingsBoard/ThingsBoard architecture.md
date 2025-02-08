# ThingsBoard services
ThingsBoard is designed to be:
- **scalable**: horizontally scalable platform, build using leading open-source technologies.
- **fault-tolerant**: no single-point-of-failure, every node in the cluster is identical.
- **robust and efficient**: single server node can handle tens or even hundreds thousands of devices depending on use-case. ThingsBoard cluster can handle millions of devices.
- **durable**: never lose your data. ThingsBoard supports various queue implementations to provide extremely high message durability.
- **customizable**: adding new functionality is easy with customizable widgets and rule engine nodes.

The diagram below shows key system components and interfaces they provide. Let’s walk through them.
![[Pasted image 20250114161748.png]]

## ThingsBoard Transports
ThingsBoard provides [MQTT](https://thingsboard.io/docs/reference/mqtt-api/), [HTTP](https://thingsboard.io/docs/reference/http-api/), [CoAP](https://thingsboard.io/docs/reference/coap-api/) and [LwM2M](https://thingsboard.io/docs/reference/lwm2m-api/) based APIs that are available for your device applications/firmware. Each of the protocol APIs are provided by a separate server component and is part of ThingsBoard “Transport Layer”. MQTT Transport also provides [Gateway APIs](https://thingsboard.io/docs/reference/gateway-mqtt-api/) to be used by gateways that represent multiple connected devices and/or sensors.

Once the Transport receives the message from device, it is parsed and pushed to durable [Message Queue](https://thingsboard.io/docs/reference/#message-queues-are-awesome). The message delivery is acknowledged to device only after corresponding message is acknowledged by the message queue.

## ThingsBoard Core
ThingsBoard Core is responsible for handling [REST API](https://thingsboard.io/docs/reference/rest-api/) calls and WebSocket [subscriptions](https://thingsboard.io/docs/user-guide/telemetry/#websocket-api). It is also responsible for storing up to date information about active device sessions and monitoring device [connectivity state](https://thingsboard.io/docs/user-guide/device-connectivity-status/). ThingsBoard Core uses Actor System under the hood to implement actors for main entities: tenants and devices. Platform nodes can join the cluster, where each node is responsible for certain partitions of the incoming messages.


# ThingsBoard message queues
ThingsBoard supports two types of message queues: **Kafka** and **In-Memory**.
- **Kafka** is a widely used, distributed, and durable message queue system designed to handle large volumes of data. It is well-suited for production environments where high throughput, fault tolerance, and scalability are critical.
- **In-Memory** queue is a lightweight, fast, and simple message queue implementation designed for testing, smaller-scale or development environments. It stores messages in memory rather than on disk, prioritizing speed over persistence.

