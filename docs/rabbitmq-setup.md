# RabbitMQ topology

This page explains how to set up RabbitMQ and send messages between CT and DC.

You can install the RabbitMQ module from the Mendix Marketplace here: https://marketplace.mendix.com/link/component/305907. I recommend using Maia (Mendix' AI) to help you set it up according to the brief guide below, which contains a brief explanation of how RabbbitMQ works as well.

## 1. Set up the exchange and queues

A publisher sends a message to an **exchange**. The exchange uses the message's **routing key** to choose a **queue**, where the message waits for a consumer to read it.

Use these settings:

| Setting              | Value                                      |
| -------------------- | ------------------------------------------ |
| Virtual host (vhost) | `/`                                        |
| Exchange name        | `bpil.v1`                                  |
| Exchange type        | Direct: the routing key must match exactly |
| Exchange options     | Durable, auto-delete off                   |
| Queue type           | Classic                                    |
| Queue options        | Durable, exclusive off, auto-delete off    |

**Durable** means the exchange or queue survives a RabbitMQ restart. It does not, by itself, mean the messages survive. Turning off **auto-delete** prevents automatic removal when consumers or bindings disappear. Turning off **exclusive** means a queue is not tied to a single connection.

Create the queues and their **bindings** (the routing rules below) before sending messages. Use one logical consumer per queue. Different actor instances must not share a queue.

## 2. Route messages to the right queue

| Routing key                 | Destination queue | Sent by | Message type             |
| --------------------------- | ----------------- | ------- | ------------------------ |
| `dc.1.arrival.update`       | `bpil.dc.1.in`    | CT      | `LoadingScheduleUpdated` |
| `dc.1.arrival.actual`       | `bpil.dc.1.in`    | CT      | `TruckArrived`           |
| `ct.1.schedule.update`      | `bpil.ct.1.in`    | DC      | `TruckETAUpdated`        |
| `ct.1.loading.confirmation` | `bpil.ct.1.in`    | DC      | `LoadingCompleted`       |

For example, CT sends a `LoadingScheduleUpdated` message to exchange `bpil.v1` with routing key `dc.1.arrival.update`. RabbitMQ places it in queue `bpil.dc.1.in`, which DC reads.

The [integration contract](integration-contract.md) also lists `TruckLoaded` (DC → CT) for when the truck has finished loading **and departed**. It is separate from `LoadingCompleted`, which is sent when loading finishes. The routing key and payload for `TruckLoaded` still need to be agreed.

## 3. Message format

Send messages as **UTF-8 JSON** so set the content type to `application/json`. Send the payload directly, without an extra wrapper object. See the examples.

## Further reading

- [RabbitMQ exchanges](https://www.rabbitmq.com/docs/exchanges)
- [RabbitMQ tutorial: acknowledgements and message durability](https://www.rabbitmq.com/tutorials/tutorial-two-java)

Or just ask your questions to an LLM :)