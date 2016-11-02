---
layout: post
title:  "Green Pipes"
date:   2016-10-30 05:58:21
categories: greenpipes
published: true
---

Earlier this month, my colleague, [Jimmy Bogard](https://lostechies.com/jimmybogard/) posted a fantastically concise [article](https://lostechies.com/jimmybogard/2016/10/13/mediatr-pipeline-examples/) about how he composes an application pipeline for processing an application's various requests using his [MediatR](https://github.com/jbogard/MediatR) framework. He walks through, step by step, how he brings in each separate concern with code examples.

Today, I will reproduce his examples using the recently extracted pipeline from [MassTransit](https://github.com/masstransit/masstransit) called [GreenPipes](https://github.com/phatboyg/greenpipes). Both MediatR and GreenPipes attempt to solve the same problem, building business pipelines _without_ using the underlying host framework.

> This post assumes that you have read Jimmy's [post](https://lostechies.com/jimmybogard/2016/10/13/mediatr-pipeline-examples/).

## The Simplest Pipeline

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 BlankPipeline.cs %}

In this example what we are seeing is the creation of a brand new pipeline, now since GreenPipes is a different approach, the code will look different. The pipe isn't interesting _at all yet_, and we have this new thing called a _[Context](http://blog.phatboyg.com/GreenPipes/texts/contexts/)_. The context in this example, `BusinessContext`, is a silly name for our demo but what it represents is the context of the request, follow the link to read more about contexts in depth via the budding GreenPipes documentation. But for now we can just think about it as our custom `HttpContext` like object.

## Contextual Logging and Metrics

Ok, so step one is to add Serilog integration.

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 SerilogFilter.cs %}

Here we can a big divergence from MediatR. In GreenPipes you compose your pipeline by building up a [set of filters for your pipeline](http://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html). In this simple case I am using the `InlineFilter` extension method which is great for prototyping out new filters. I will build the rest of the examples out using this method, but at the end I will share the suggested approach for building a reusable and sharable filter.

On line #5 above we are opening up this new `InlineFilter` which allows us to pass a lambda that takes in a `Context` mentioned above and it also passes the `next` pipe segment. This allows us to use the classic .Net `using` pattern. Once inside of the `using` block we call `next.Send` passing down the context. This should look a lot like `OWIN` or any other similar framework. We purposefully kept this pattern; as its both powerful and well understood by the community.

How about some metrics?

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 MetricsPipeline.cs %}

> Note, I am not sure what library Mr. Bogard is using, so this is just some fake code at this point.

## Validation

Alright, lets get to something meaty! Let's follow Jimmy's lead and bake in some Fluent Validation support.

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 ValidationPipeline.cs %}

Just like Jimmy's example we have a way to apply validation logic in consistent manner with just a single location at play. One thing that is not directly obvious in my example is how we end up handling the _contravariant_ aspect of validation that Jimmy points out. For that, we will have to dive into how GreenPipes does dynamic message dispatch in another blog post. For now I'll have to tempt you with a [unit test](https://github.com/phatboyg/GreenPipes/blob/develop/src/GreenPipes.Tests/Dispatch_Specs.cs) from the code base. You can see some of how it all works at the end in the filter creation at the end.

An important item to note at this point is that when we throw the ValidationException it won't be caught by the sending code **UNLESS** it was `await`ed as GreenPipes heavily leverages the TPL and its patterns.

If the context passes validation, we then call `next.Send(cxt)` other wise we throw our exception and stop processing.

## Authorization

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 AuthorizingPipeline.cs %}

Authorization is similar to Fluent Validation above, so not much to explain here.

## Pre/Post Processing

By now I hope you can see where this is going. Jimmy lays out a great approach that is easy to follow as well. I'll leave this bit as an exercise up to the reader.

## The Overview at the End

In the end you might have something like this that you can hand off to the rest of the team. Not the prettiest thing out there, and not nearly as pretty as what Jimmy has laid out.

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 Final.cs %}

## Finally, The Deep Dive

I've eluded to a more complex pattern that can be used to set up a sharable GreenPipe filter. Well, buckle in, here we go.

From its origins GreenPipe is the infrastructure code that was used to build the composable pipelines for MassTransit. Because MT is a _framework_ not a _library_ we needed to allow for arbitrarily complex middleware. Everyone and their Managers each have a unique business problem that they are trying to solve and the need for flexibility in MT is high. This approach has allowed MT to add Sagas, State Machines, Quartz.net based timeout support, Retry mechanisms, multiple transport support, and a host of other features without having to muddy up the core code of MassTransit which continues to be all about handling serialization, transport and middleware concerns of a message based pipeline.

Because of these requirements our filters require a bit more set up than what we see with MediatR. This is an extra cost, but its one that we believe provides an immense amount of value in our ability to "scale with complexity."

### The Fluent Validation Filter Part Duex

And here is how we could build a complete filter for Fluent Validation with an error pipeline for alternate processing.

#### The Context

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 0_ValidationFailureContext.cs %}

The context for any GreenPipes pipeline is the heavy lifter of the data. In MassTransit we work a lot with the `ReceiveContext<TMessage>` which has, as one of its properties, the venerable `Message`. Now, in our scenario, we wrap the existing context and then attach the validation failures. We do this so that the downstream `ValidationFailurePipe` can decide what to do with them. That's right, its another pipe. We will get to that in more detail a bit further down.

#### The Extension Method

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 0_FluentValidationExtensionMethods.cs %}

Like all things in MassTransit, the extensions to the core framework show up as extension methods. In this case GreenPipes exposes some very low level functions that are pretty chatty and abstract. The recommended approach is to provide an extension method that papers over this complexity for your users. You can see this pattern in TopShelf, MassTransit and now GreenPipes.

> This patten has been very helpful as a way for extension authors to ship new and interesting behavior without the core product having to make any changes.

#### The Specification

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 1_FluentValidationSpecification.cs %}

This is one of my favorite pieces of the puzzle. GreenPipes forces you to make a specification class. This object holds the data from the extension method and allows you to populate your filter with that data. This by itself was pretty annoying for me, it felt like SRP for the sake of SRP. Until I wrote my first complex filter and came to _REALLY_ appreciate the `Validate()` method.

The `Validate()` method gives you a chance to validate the user's input in building out the filter. This is called at bus start up (_Fail Early_) Additionally, it gives you a means to report errors back to the user in a structured fashion along with every other validation failure.

#### The Filter

I know it's taken a while to get here, but we have finally arrived at the actual filter. At first blush this should seem very familiar as its almost a verbatim copy of what Mr. Bogard wrote up. I'd rather instead focus on the two further differences in the GreenPipe filter.

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 2_FluentValidationFilter.cs %}

First, we have the `Probe` method. This method provides a mechanism for dynamically inspecting the pipe at run time. This is the heart of the diagnostics in MassTransit and it was pulled out so that any one building on top of MassTransit can use it. Here's some example output from a unit test.

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 9_probe.json %}

Above, we can see the results of our Serilog filter, our FluentValidation filter and the inline filter that I'm using for the security concern. By taking the extra time to build our filters the GreenPipes' way we can expose a dynamically rich amount of data in a structured format. Currently, this uses JSON.Net to serialize data and you could easily use JSON.Net to then deserialize this data back into C# objects for use in your project's dashboard.

> The purpose of the probe is to allow filter authors to provide a diagnostic payload that is composable with the entire pipeline. Using this capability it's possible to see - in a running application - how the entire pipeline is constructed. Think of it as a debug view that you can access right off the pipeline using `pipe.GetProbeResult()`.

#### The Complete Picture

If you were to follow this process to its logical conclusion, your final pipe could look as minimal as this on the surface of the pipe.

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 clean.cs %}

## A parting note

The code for Fluent Validation looks just like MediatR but what is this other pipe?? Well, one final difference between GreenPipes and MediatR is that MediatR returns. The MediatR signature is `TResponse Handle(TRequest message)` this is gloriously simple and easy to grok, another benefit of the MediatR approach. We can also see that MediatR allows you to throw exceptions so that you can catch them when invoking MediatR. GreenPipes allows for a more complex model, again out of needs born in MassTransit. But, if you are willing to pay for this extra complexity you will get the ability to _jump the tracks_. What I mean is you can exit any given pipe and start down another pipe (any one want [Railway Programming](http://fsharpforfunandprofit.com/rop/) in C#?).

We can see that there is this other pipe for Validation Failures. If we fail validation, we immediately stop processing (by not calling `next.Send`) and divert down the `_validationFailurePipe`. Now this is a completely different pipe that can do all manner of things. I've used it to still save my users data, but then send out an email to that user alerting them to issues (please note, I rarely write UI oriented code). MassTransit has used this to take messages that have become poisonous and divert them to a poison message queue. All of this and more and you can control just how far and deep you want to go at each level. This is the power you get in exchange for the complexity of not having a direct return value.

> That said, you may be looking at me ಠ_ಠ like, "Dru, I _NEED_ return values." I completely understand and I've felt that pain. Chris and I are still working on how to best make this a reusable pattern - but the whole picture looks a bit [like this](https://gist.github.com/phatboyg/0e3ef1029d29195997ca8d046d0d384f). The basic pattern is that your context has a way to "set" a return value. If that property is of type `Task<T>` you can then `await` on that property of your context. Then if you wrap that in an extension method your send can now be a `Request`.


## In closing

The goal again was to show case a different approach to building out a pipeline for your business processes. Each framework takes its own approach and brings different values and levers for you to accomplish your goal.

### MediatR

- Synchronous Model by Default (pretty sure nothing is preventing you from doing TPL in `IRequestHandler` though)
- Simple to build and get an overview of the pipeline.
- No extra infrastructure code required
- Composed via static code

### GreenPipes

- Async Model by Default
- Ability to build dynamic and complex pipelines to suit business needs
- Requires you to build some infrastructure code to get dynamic probe tracing and filter validation working
- Composed via filter objects
