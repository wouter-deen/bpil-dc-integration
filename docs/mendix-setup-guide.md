# Mendix setup guide

This guide is for both DC and CT teams. Use [REST](integration-contract.md) when one app needs an immediate response. Use [RabbitMQ](rabbitmq-setup.md) for updates the other app can read later.

## 1. Connect the apps with REST

The DC exposes `POST /orders`. The CT calls it with `staging_duration` and `loading_duration` already included in the order, as integer seconds.

For both REST and RabbitMQ, map event times to Mendix `Date and time` values internally. Check that the wire format is a UTC string with whole seconds and a trailing `Z` (for example, `2026-11-28T10:02:00Z`), not a local date/time or a number. Use synchronized system clocks.

1. Create an `Integration` module in Mendix. For each JSON request or response your app uses, create a **JSON Structure** and the import or export mapping you need. Prefer non-persistent entities for these temporary messages.
2. Add a **Published REST Service** at `rest/integration/v1`. Add the `POST /orders` operation in the DC app, with an import mapping for the request and an export mapping for the response.
3. In a microflow, validate the imported fields before saving anything to your app's domain model. Return the agreed HTTP status and error body when a request fails.
4. In the CT app, use **Call REST Service** in a microflow. Store the DC app's base URL in an environment-specific constant.
5. Test `POST /orders` in Postman before connecting the apps. The API specification tells you what to build, but it does not implement the service for you.

Mendix docs: [Publish REST](https://docs.mendix.com/refguide/publish-a-rest-service/) · [Consume REST](https://docs.mendix.com/howto/integration/consume-a-rest-service/)

## 2. Connect to the shared RabbitMQ broker

Use the [RabbitMQ topology guide](rabbitmq-setup.md#2-route-messages-to-the-right-queue) to find your inbound queue name and the routing keys for messages you send.

The broker operator (group 7) provisions the exchange, queues, and bindings before teams connect. The Mendix connector does not create them. Set `RabbitMQ.Host` to `rabbitmq.unsea.nl` and `RabbitMQ.PortNumber` to `5672`. See the [actor registry](actor-registry.md) for the virtual host and transport security notes. Use the AMQP endpoint.

Contact **Wouter Deen** to request your team's credentials and inbound queue. Include your Canvas group number and whether you're a CT or DC team. Don't wait until the last minute. Allow one to two weekdays for a response.

## 3. Install the Mendix connector

1. In Mendix, import [RabbitMQ by RapidData Technologies](https://marketplace.mendix.com/link/component/305907/RapidData-Technologies/RabbitMQ) from the Marketplace. This guide uses module version `1.0.0` (listed minimum Mendix `11.6`, built with `11.12.2`). Check compatibility before importing. If you use a different connector, follow its own action names and instructions.
2. Add the required JARs to your project’s `userlib`: [`amqp-client` 5.21.0 or newer](https://central.sonatype.com/artifact/com.rabbitmq/amqp-client) and [`slf4j-api` 2.0.13 or newer](https://central.sonatype.com/artifact/org.slf4j/slf4j-api). Check what is already there to avoid duplicate versions. Then synchronise the project directory and rebuild.
3. Set `RabbitMQ.Host`, `RabbitMQ.PortNumber`, `RabbitMQ.UserNameRabbitMQ`, and `RabbitMQ.PasswordRabbitMQ` for your environment. Do not commit passwords.

## 4. Send and receive messages

**Send a message**

1. Save the business change and keep a persistent copy of the outgoing message so you can retry it.
2. Export the message to a JSON string and call `PublishToRabbitMQ` with the exchange, routing key, and body. Use the parameter names shown in your installed action.
3. Mark the record as sent only if the action returns successfully. A successful return does not necessarily prove RabbitMQ received the message or the other app processed it.

**Receive a message**

1. In a microflow, call `ConsumeFromRabbitMQ` to read one message from your team's queue. Stop if there is no message. Check how the installed action reports empty results and errors.
2. Save the raw JSON, identify the message type, and import it with the matching mapping. Validate it before applying the update.
3. Run the microflow from a scheduled event or a bounded polling loop. Avoid overlapping runs. Choose a frequency that keeps updates responsive, and use the event timestamp in the payload when present rather than the time the message was polled.

## 5. Test before connecting the teams

- Send one message and check that it reaches the right queue and updates the receiving app.
- Try an empty queue, invalid JSON, a duplicate message, and messages arriving out of order.
- Check when your connector acknowledges received messages and how it handles failures. A broker acknowledgement does not confirm that the other app saved the update.

The connector instructions were checked against the module documentation on 25 November 2026. Installation and runtime behaviour were not tested in Mendix here, so run these checks before relying on the connector.

Optional reading: [Installing Marketplace content](https://docs.mendix.com/appstore/use-content/) · [RabbitMQ tutorials](https://www.rabbitmq.com/tutorials) · [AsyncAPI document structure](https://www.asyncapi.com/docs/concepts/asyncapi-document/structure) · [AsyncAPI AMQP bindings](https://www.asyncapi.com/docs/reference/bindings/amqp)

> AI Disclaimer: this guide was partially generated using GPT-6.2-Sol. The instructions here might contain errors, for which the authors take no responsibility.
