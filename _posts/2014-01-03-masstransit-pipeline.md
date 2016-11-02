---
layout: post
title:  "Building a Pipeline in MassTransit"
date:   2014-01-03 05:58:21
categories: masstransit
published: false
disqus_id: b90bcc77-b92d-4c7e-b378-5dbbeafee31f
---

At my new [employer](http://amplifier.com) we are rebuilding our order processing
pipeline to provide greater insight into the various stages of an order. This is
not a unique problem, so I thought I would share how I built it using the
MassTransit family of tools:

_Context: C# and .Net_

- [MassTransit](http://masstransit-project.com) (message routing)
- [Automatonymous](https://github.com/MassTransit/Automatonymous) (state machines)
- [MassTransit.Courier](https://github.com/MassTransit/MassTransit-Courier) (routing slips)

Context Reading

- [Message Interfaces]({% post_url 2014-01-01-masstransit-interfaces %})
- [Message Context]({% post_url 2014-01-02-masstransit-context %})

## The pipeline

- Receive partner command
    - Convert to standard model
- Store order data in primary system
- Initiate Order Flow w/ Courier Routing Slip
  - allocate order
  - approve order
  - create shipment
  - enter into Warehouse Management System
- monitor order (and update for customers to see) for
    - in wave (picking)
    - packing order
    - ship order
- close order out
    - close shipment
    - close order

## Accepting commands at the edge

At the edge is our company HTTP API, here we accept incoming commands from our
integration partners and convert them into our internal model.

{% highlight csharp %}
public class SalesOrderEndpoint
{
  MassTransit.IServiceBus _bus;

  public HttpResponse Post(HttpRequest request)
  {
    var command = ConvertToRegisterSalesOrderCommand(request);
    var endpointAddress = GetPriorityBasedEndpoint(command);
    _bus.GetEndpoint(endpointAddress)
      .Send(command);
  }
}
{% endhighlight %}

Nothing too surprising here I hope. We simply convert it from the integration
partner model into our internal model. Then we get a priority appropriate input
queue, this lets us maintain a high throughput for all customers, and avoids letting
any one customer take up the whole queue, starving other customers.

### Tracking IDs

Much like when I order something from Amazon, I want to be able to track my
order through my pipeline. In order to do so I need to identify and create a few
numbers. The first is the ID that I am going to use for idempotency - in this
case I will use RegisterSalesOrder.ReferenceId - this maps to the external
systems unique id. I additionally want to be able to track the progress of the
unique order through the pipeline each time (if it fails for instance) so I also
give it an instance id, which will be a Guid so I can create it in the edge
code.

### Idempotency check

I often get asked how I do idempotency in my messages. Well here at the
beginning of the pipeline is the best way i think.

{% highlight csharp %}
public class RegisterSalesOrderConsumer :
  Consumes<RegisterSalesOrder>.Context
{
  public void Consume(IConsumeContext<RegisterSalesOrder> context)
  {
    if(AlreadyProcessed(context))
    {
      _log.Info("Ignoring Order, already processed");
      return;
    }

    ProcessOrder(context);
  }
}
{% endhighlight %}

## Getting everyone to play nice

If you haven't seen MassTransit.Courier yet, you should take a second to read
Chris' [blog](http://blog.phatboyg.com/2013/03/27/implementing-routing-slip-with-masstransit/)
on it. We are going to use Courier to orchestrate the various steps in our pipeline.

Here we define some sample activities similar to what we might see in real code.

{% highlight csharp %}
public class RegisterSalesOrderActivity :
  Activity<RegisterSalesOrder, UndoSalesOrder>
{
  public ExecutionResult Execute(Execution<RegisterSalesOrder> execution)
  {
    var order = execution.Arguments;

    var id = StoreOrder(order);
    return execution.Completed(new UndoSalesOrder{ OrderId = id });
  }

  public CompensationResult Compensate(Compensation<UndoSalesOrder> compensation)
  {
    var log = compensation.Log;
    CancelOrder(log.OrderId);

  }
}

public class AllocateOrderActivity :
  Activity<AllocateOrder, DeallocateOrder>
{
  public ExecutionResult Execute(Execution<AllocateOrder> execution)
  {
    var order = execution.Arguments;

    var allocationId = AllocateOrder(order);

    return execution.Completed(new DeallocateOrder{ AllocationId = allocationId });
  }

  public CompensationResult Compensate(Compensation<DeallocateOrder> compensation)
  {
    var log = compensation.Log;
    DeallocateOrder(log.AllocationId);

  }
}
{% endhighlight %}

Now that we have the Activities that we want to orchestrate, we can build a
routing slip that will be executed on the bus.

{% highlight csharp %}
public class BuildSlipAndExecute
{
  IServiceBus _bus;

  //init in ctor

  public void Execute()
  {
    var routingSlip = //build it up
    _bus.Execute(routingSlip);
  }
}
{% endhighlight %}

So, we use routing slips and activities from corourier to automate the various
steps in our process and ensure that they get completed.

- what if there is a problem
    - compensatable
    - non-compensatable
- what if it stops half way into the routing slip?

## State Machines

{% highlight csharp %}
public class OrderState : StateMachineInstance
{
  public State CurrentState { get; set; }

  public DateTime ReceivedOn { get; set; }
  public DateTime? ShippedOn { get; set; }
  public string TrackingNumber { get; set; }
}

public class OrderWorkflow :
 AutomatonymousStateMachine<OrderState>
{
  public OrderWorkflow()
  {
    Event(() => Received);
    Event(() => Allocated);
    Event(() => Picked);
    Event(() => Shipped);

    State(() => AllocatingOrder);
    State(() => PickingOrder);
    State(() => PackingOrder);
    State(() => PendingShipment);
    State(() => Complete);

    Initially(
      When(Received).TransitionTo(AllocatingOrder),
      When(Allocated).TransitionTo(PickingOrder),
      When(Picked).TransitionTo(PackingOrder),
      When(Packed).TransitionTo(PendingShipment),
      When(Shipped)
        .Then((instance,data) =>
        {
          instance.TrackingNumber = data.TrackingNumber;
          instance.ShippedOn = DateTime.Now;
        })
        .TransitionTo(Shipped)
      );
  }


  public State AllocatingOrder { get; set; }
  public State PickingOrder { get; set; }
  public State PackingOrder { get; set; }
  public State PendingShipment { get; set; }
  public State Complete { get; set; }



  public Event<SalesOrderReceived> Received { get; set; }
  public Event<SalesOrderAllocated> Allocated { get; set; }
  public Event<SalesOrderPicked> Picked { get; set; }
  public Event<SalesOrderShipped> Shipped { get; set; }
}
{% endhighlight %}

Because we can persist Automatonymous state machines to a database with
their state, we can easily query these guys back out to see where our 'orders'
are.

- external saga for the customer
- that is 'fed?' by internal sagas

__how do statemachines get subscribed__

## Human Response Monitors?

automated steps are orchestrated by 'routing slips' (routing slip entry/creation points) then something
has to watch the humans, and start the next routing slip

the saga subscribes to the public events
- automatonymous


internal events / routing slip completed

- routing slip correlation table
    - watches for the routing slip completed events
        - faulted / failed / completed


3 different routing slip places

- routing slip completed is a global event
    - trying to decide if it matters or not
