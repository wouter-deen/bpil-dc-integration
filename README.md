# Communication with DCs

How control towers (CTs) can communicate with distribution centers (DCs) and vice versa is described in the [integration contract](docs/integration-contract.md). We use a combination of a REST API for synchronous communication and RabbitMQ for asynchronous communication. See the [setup guide](docs/mendix-setup-guide.md) to connect your Mendix app.

<img src="diagrams/supply-chain-collaboration.svg" alt="Supply-chain collaboration between the Transport Company, Control Tower, and Distribution Center" width="560">

# BPIL Integration Contracts

- [REST API docs (Swagger UI)](https://wouter-deen.github.io/bpil-dc-integration/): browse the DC OpenAPI contract. Try it out is disabled because the server URL is an illustrative localhost endpoint.
- [RabbitMQ docs (AsyncAPI)](https://wouter-deen.github.io/bpil-dc-integration/asyncapi/): browse the DC and CT channels, operations, and messages.

Pushes to `main` redeploy both pages together.
