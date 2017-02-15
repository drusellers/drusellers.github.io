---
layout: post
title:  "Automatonymous and an Order"
date:   2016-11-05 05:58:21
categories: Architecture
published: false
disqus_id: 22ec0c92-bb3f-4d81-ad4f-b92820f08292
---

How do you integrate the state machine library Automatonymous with an existing object in your enterprise? Its surprisingly quite easy. When I first started down this path I was expecting something a bit difficult and was worried about the state machine goop mucking up my business objects and making them harder to understand. However, in the end I was quite pleased with the approach that had developed since I first started using it.

What follows is a journal of my integration of Automatonymous with our order tracking system at work.

## The Original Order Model

Below is a rough sketch of the order model.

```
Order
    Sales Order Number
    Purchase Order Number
    Address
    Promised Deliver On
    Errors
    Lines
        Line Number
        Sku
        Quantity Ordered
    Shipments
        Tracking Number
        Shipped On
        Lines
            Sku
            Quantity Shipped
        Invoice
            Invoice Number
            Freight Amount
            Lines
                Sku
                Quantity
                Cost
```

## The Model After Integrating Automatonymous

```
OrderTransaction
    SagaId
    State
    StartedOn
    EndedOn
    Order
        Sales Order Number
        Purchase Order Number
        ...
```

All we had to do was put this "on top" of the existing model.

Yeah, I didn't expect it to be that easy. I initially assumed that I would need to and want to put the state on the order object itself. But that means my UI is going to be competing on the same table as the Automatonymous table which could be receiving quite a bit of traffic. So pulling that out into a different table adds some nice performance value.

There is some additional value in pulling out the state of the "transaction/saga" from the order object itself. In my case, the order doesn't really change - what changes is the "state" of the order. Is it fulfilled? Is it Complete? Has it been cancelled and these are strictly on the order. I can treat my order class as immutable if I pull the state out of the object itself.

## Are the lines Sagas too?

In my model they are indeed sagas too. Each line can be complete or cancelled on its own. They are "value objects" in DDD terminology as the PK of an order line is its parents order identifier + the line number of the order. No one in the business talks about line 45678-78543, they say what's the status of line 22 on order 46. So, we can see that we need to track the status of each line as well. This is a good point of clarity that you don't have to be a DDD entity to have a saga tracking your lifecycle.

```
OrderLineTransaction
    SagaId
    OrderId
    LineNumber
    State
```

Tracking the Line State and the Order State in isolation seems pretty easy, but how do I marry these two things together? How do I make them play well?
