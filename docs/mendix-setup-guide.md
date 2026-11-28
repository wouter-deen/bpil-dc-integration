# Mendix setup guide

This guide is for both DC and CT teams. Use [REST](integration-contract.md) when one app needs an immediate response. Use [RabbitMQ](rabbitmq-setup.md) for updates the other app can read later.

## 1. Connect the apps with REST

The DC exposes `POST /orders`. The CT calls it with `staging_duration` and `loading_duration` already included in the order.

1. Create an `Integration` module in Mendix. For each JSON request or response your app uses, create a **JSON Structure** and the import or export mapping you need. Prefer non-persistent entities for these temporary messages.
2. Add a **Published REST Service** at `rest/integration/v1`. Add the `POST /orders` operation in the DC app, with an import mapping for the request and an export mapping for the response.
3. In a microflow, validate the imported fields before saving anything to your app's domain model. Return the agreed HTTP status and error body when a request fails.
4. In the CT app, use **Call REST Service** in a microflow. Store the DC app's base URL in an environment-specific constant.
5. Test `POST /orders` in Postman before connecting the apps. The API specification tells you what to build; it does not implement the service for you.

Mendix docs: [Publish REST](https://docs.mendix.com/refguide/publish-a-rest-service/) · [Consume REST](https://docs.mendix.com/howto/integration/consume-a-rest-service/)

## 2. Set up RabbitMQ

RabbitMQ holds messages until the receiving app reads them. For example, DC sends a `TruckETAUpdated` message to exchange `bpil.v1` with routing key `ct.1.schedule.update`. The exchange uses that key to put the message in `bpil.ct.1.in`, which CT reads. See the [RabbitMQ setup](rabbitmq-setup.md) for the full list of queues and routing rules.

One group hosts the shared broker. To try RabbitMQ **on your own machine** first, run:

```sh
docker run --name bpil-rabbit \
  -p 127.0.0.1:5672:5672 -p 127.0.0.1:15672:15672 \
  -e RABBITMQ_DEFAULT_USER=bpil \
  -e RABBITMQ_DEFAULT_PASS=local-dev-only \
  -d rabbitmq:3-management
```

1. Open [the local RabbitMQ dashboard](http://localhost:15672) and sign in with `bpil` / `local-dev-only`.
2. Create the exchange, queues, and bindings from the [RabbitMQ setup](rabbitmq-setup.md) **before** running the Mendix app. The connector does not create them for you.
3. For the shared broker, use a host both apps can reach, separate credentials, restricted network access, and TLS where supported. The Docker command only allows local connections. Do not reuse the demo credentials on the shared broker.

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
3. Run the microflow from a scheduled event or a loop with a fixed limit when the simulation advances. Avoid overlapping runs. Choose a frequency that keeps the UI responsive and does not distort simulation time. Do not assume a scheduled event can run every tick.

## 5. Test before connecting the teams

- Send one message and check that it reaches the right queue and updates the receiving app.
- Try an empty queue, invalid JSON, a duplicate message, and messages arriving out of order.
- Read the [RabbitMQ setup](rabbitmq-setup.md) for acknowledgement timing and what to do if a message is lost. A broker acknowledgement is not confirmation that the other app saved the update.

The connector instructions were checked against the module documentation on 25 November 2026. Installation and runtime behaviour were not tested in Mendix here, so run these checks before relying on the connector.

Optional reading: [Installing Marketplace content](https://docs.mendix.com/appstore/use-content/) · [RabbitMQ tutorials](https://www.rabbitmq.com/tutorials) · [AsyncAPI document structure](https://www.asyncapi.com/docs/concepts/asyncapi-document/structure) · [AsyncAPI AMQP bindings](https://www.asyncapi.com/docs/reference/bindings/amqp)

> AI Disclaimer: this guide was partially generated using GPT-6.2-Sol. The instructions here might contain errors, for which the authors take no responsibility.
