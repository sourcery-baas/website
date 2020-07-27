---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

## What is Sourcery ?
Backend as a Service for event driven architectures. 

Sourcery enables the construction of easily extensible data-driven systems, by harnessing the power of events.

Centralize an immutable stream of facts.
Decentralize the freedom to act, adapt and change.

### Principles

- Event Sourcing
  - Events as the source of truth.
  - Immutable log.
- CQRS patterns.
  - Decouple writes from reads.

## How it works ?

1. Write data in, forming streams.
2. Transform data according to your rules.
3. Read data out for visualization or side-effects.

Sourcery apps enable the construction of systems from three components:

- Streams. To get data in.
- Processors. That transform data according to business logic, shaping it to your needs.
- Views. To get data out.

That's it. Focus on data not on infrastructure.

## Concepts

### Streams

In Sourcery you create a Stream for each type of data that you want to write in your app.

Streams can be created from several data sources like:
- Custom HTTP endpoints. All you need is a unique name and a schema to start.
- An external DB. Connect your DB and capture writes as they come.
- A message broker. Connect your favorite message broker to feed your Sourcery app.
- A Sourcery Processor. Combine and transform streams into new streams.

### Views

Views are read representations of your data that you need to perceive. A view can be represented as:

- A HTTP endpoint that you can query to populate your UI. Get the data exactly shaped as you need, ready to be exposed.
- A suscription that will emit events just as the get into any stream.

### Processors

Processors take data from streams and allow you to do complex transformations to apply business logic. Most of your business rules will leave here.

## Tutorial

As an example we'll consider a very simple online store that we'll help us illustrate how you can use all of Sourcery componetes to build a complete workflow.

After done we will be able to:

- Manage a simple product stock.
- Take orders in from a HTTP endpoint, validating available stock.
- Get a view of ready orders.
- Emit a webhook notification every time an order gets ready for shipment.

