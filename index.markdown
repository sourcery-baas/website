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

![sourcery](/images/diagram.png){:class="img-responsive"}

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

  
### Processors

Processors take data from streams and allow you to do complex transformations to apply business logic. Most of your business rules will live here.

- Stateless:
  - Get streams as input and get streams as output.
- Stateful
  - Combine streams with "state" to create new streams.

### Views

Views are read representations of your data that you need to perceive. A view can be represented as:

- A HTTP endpoint that you can query to populate your UI. Get the data exactly shaped as you need, ready to be exposed.
- A suscription that will emit events just as the get into any stream.
  - real-time: get theme from the point the suscription starts
  - catch-up: listen to all events from certain point in history to rebuild your custom representation

A view could also be seen as a specital type of processor that creates state.

For those familiar with Redux, think of views like reducers.

## Example

As an example we'll consider a very simple online store that we'll help us illustrate how you can use all of Sourcery componetes to build a complete workflow.

After done we will be able to:

- Manage a simple product inventory.
- Take orders in from a HTTP endpoint.
- Validate orders according to inventory.
- Get status of order.

### Create an API endpoint to receive orders

We first want to have a way to get orders in

`POST /api/streams`

with body: 

```json
{
  "name": "orders",
  "schema": {
    "id": "string",
    "customerId": "string",
    "productId": "string",
    "qty": "number",
    "status": "string"
  }
}
```

This will expose this endpoint for you: `POST /api/streams/orders` where you can POST with the previously defined "schema". Your only concern when creating the schema is about what data you want to receive from your user, you don't have to think about how you're going to expose it or anything else.


### Create an API endpoint to receive product inventory

Let's say now that you want to keep an inventory, so you can later validate that you can only successfully accept an order if there's enough stock of the wanted item.

`POST /api/streams`

with body: 

```json
{
  "name": "products",
  "schema": {
    "id": "string",
    "qty": "number",
    "price": "number"
  }
}
```

Again, that's it, don't worry about possible future use cases, worry only about your current needs, we'll have space to evolve this later!

Similarly, you now have available for you an endpoint to `POST /api/streams/product-inventory`

### Create a view to check order status

Now that you have some data coming in, it will be useful to have a way to read. In particular we want to be able to check the status of an order. We can simply create a view that will expose the latest state of our orders:

`POST /api/views`

with body:

```json
{
  "name": "orders-view",
  "from": "orders",
}
```

That's it! Orders will be grouped by id by default, so that will be the param we use when querying through our new available endpoint to `GET /api/views/orders/{id}`. 

> Do I have to manually create a view like this every time !? No! You can also pass a `"viewable": true` flag when creating the stream so you get an automatic view for it, but we won't discuss that here, we want to show the full process so you understand all the flexibility you get :) 

Since views are populated asynchronously, requests to this endpoint will issue a "long poll" until the id becomes available. If you POST any new data to /api/streams/orders, latest state will be reflected here. But ... we don't want you to be responsible to change order state, it should be updated reactively after validating it! We're almost there, but first ...

### Create a view to store latest product inventory

Since we want to keep the state of the inventory, we need a view for it. But we just want this state to be handled internally, we don't need to expose it.

`POST /api/views`

with:

```json
{
  "name": "products-view",
  "from": "products",
  "exposed": false
}
```

That's it. By default it is not exposed, we just want to use this state internally to feed our next processor:

### Create a processor that joins orders data product data

Now we want to create some logic to validate orders in response to every newly submitted order. We want a processor that will join every order with stock data for the desired product. We need a `join`, so we get a new stream where we have together the order with the current avialable stock:

`POST /api/processors`

with:

```json
{
  "name": "order-validation",
  "from": "orders",
  "join": {
    "with": "products-view",
    "by": "productId"
  },
  "to": "order-with-stock"
}
```

This is instructing the processor to join orders with products-view, by matching orders `productId` field with the id field on `products-view`. 

The new stream will be named `order-with-stock`, where every entry will look similar to this:

```json
{
  "order": {
    "id": "df4843fc-88eb-4709-8790-5acc02dbd65b",
    "customerId": "4834e62f-8a57-4c27-bb3c-3c0756de1e26",
    "productId": "19b561e8-04a2-47ef-84bd-c6a6e55c1b83",
    "qty": 2,
    "status": "string"
  },
  "product": {
    "id": "19b561e8-04a2-47ef-84bd-c6a6e55c1b83",
    "qty"
  }
}
```

> The schema of it can be changed if desired with a `map`, but that's not discussed in this example. 

On the next episode we will do the actual validation now that we have the necessary data for it. 

> For didactical purposes we're separating every processing step as a separate processor, but we will show at the end an example of a processor that do everything.

### Create a processor that validates stock for orders

We can finally write validation. What we want is to check for the order and emit the order again with its updated status. For that we're going to a split processor.

`POST /api/processors`

with:

```json
{
  "name": "order-validation",
  "from": "order-with-stock",
  "split": {
    "valid-orders": { "filter": {"order.qty": { "$lte": "product.qty" } } },
    "invalid-orders": { "filter": {"order.qty": { "$gt": "product.qty" } } }
  }
}
```

This will emit to new streams `valid-orders` and `invalid.orders` with the same data, that you can now use to write some more logic reacting to them like sending a notification, or just use it in another orders processor to update the order, let's do that.

> At this point you would also need to write something to update stock for the product, but we won't do that here. That's your homework, sorry.

### Create a processor to update order status

Updating the status is just about emitting the same order again withe the updated value, for that we will create the following processor:

`POST /api/processors`

with:

```json
{
  "name": "order-status-update",
  "from": "valid-orders",
  "map": {
    "$set": { "status": "confirmed"}
  },
  "to": "orders"
}
```

> We can write a similar one to update the status as "failed".

I know this is verbose, can we do better ? Yes! Now that we showed how you could write different streams, views and processors, we'll se an example of an improved simplified processor

### Create a multi-step processor that does everything itself

`POST /api/processors`

with:

```json
{
  "name": "order-validation",
  "from": "orders",
  "multi": [
    {
      "join": {
        "with": "products-view",
        "by": "productId"
      },
    },
    {
      "split": [
        {
          "filter": { "order.qty": { "$lte": "product.qty" } },
          "to": {
            "map": { "$set": { "status": "confirmed"} },
            "to": "orders"
          }
        },
        {
          "filter": { "order.qty": { "$gt": "product.qty" } },
          "to": {
            "map": { "$set": { "status": "failed"} },
            "to": "orders"
          }
        }
      ]
    }
  ]
}
```

> And there we have it ! Not fond or using an API. Good news is soon you will be able to do this with the new magical IDE, with a drag and drop UI, and the option to do easy integration testing of your whole workflow ! 
