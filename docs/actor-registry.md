# Actor registry

This will be updated when new queues are created. The table now contains temporary placeholders.

| Actor | Instance ID | REST base URL (example only)              | Queue        | Owner/contact |
| ----- | ----------: | ----------------------------------------- | ------------ | ------------- |
| DC    |           1 | http://localhost:8080/rest/integration/v1 | bpil.dc.1.in | TBD           |
| CT    |           1 | http://localhost:8081/rest/integration/v1 | bpil.ct.1.in | TBD           |

The shared RabbitMQ broker is available at `rabbitmq.unsea.nl` on AMQP port `5672`. Use virtual host `/` and exchange `bpil.v1`. AMQP TLS is not configured. The management dashboard is private and managed by group 7. See the [Mendix setup guide](mendix-setup-guide.md#2-connect-to-the-shared-rabbitmq-broker) to request access.

An instance ID is the Canvas group number that identifies a CT or DC team and is used for the inbound queue name, such as `bpil.ct.4.in` for CT group `4`. When publishing, put the receiving team's number in the routing key, such as `ct.4.schedule.update`. RabbitMQ matches the routing key exactly, so the wrong number can send the message to another team or leave it unrouted. The table and AsyncAPI specs use `1` as an example. See the [RabbitMQ topology guide](rabbitmq-setup.md#2-route-messages-to-the-right-queue) for how Canvas group numbers map to queue names and routing keys. REST URLs must be reachable from the other runtime (localhost only works on the same machine). TC endpoints and routing are owned by CT/TC groups, not DC.
