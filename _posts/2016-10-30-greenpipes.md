---
layout: post
title:  "Green Pipes"
date:   2016-10-30 05:58:21
categories: greenpipes
published: true
---

Earlier this month, my colleague, [Jimmy Bogard](https://lostechies.com/jimmybogard/) posted a fantastically concise [article](https://lostechies.com/jimmybogard/2016/10/13/mediatr-pipeline-examples/) about how he composes an application pipeline for processing an application's various requests using his [MediatR](https://github.com/jbogard/MediatR) framework. He walks through step by step how he brings in each separate concern with code examples.

Today, I want to try and reproduce his examples using the recently extracted pipeline from [MassTransit](https://github.com/masstransit/masstransit) called [GreenPipes](https://github.com/phatboyg/greenpipes).

This post assumes that you have read Jimmy's post.

## The Simplest pipeline

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 BlankPipeline.cs %}

Ok, in this example what we are seeing is the creation of a brand new pipeline, now since GreenPipes is a bit different approach, the code does look different. The pipe part isn't interesting _at all yet_ and we have this new thing called a _[Context](http://blog.phatboyg.com/GreenPipes/texts/contexts/)_. The `BusinessContext` is a silly name for our demo but it represents the context of the request, follow the link to read more about contexts in depth. But for now we can just think about it as our custom HttpContext like object.

I agree with Jimmy, nothing exciting yet so lets kick this thing up a notch and do something interesting.

## Contextual Logging and Metrics

Ok, so step one is to add Serilog integration.

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 SerilogFilter.cs %}

So, here we can the biggest divergence from MediatR. In GreenPipes you compose your pipeline by building up a set of filters for your pipeline. In this simple case I am using the `InlineFilter` filter which is great for prototyping out new filters. I will build the rest of the examples out using this method, but at the end I will share the suggested approach for building a reusable and sharable filter.

Now with metrics.

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 MetricsPipeline.cs %}

## Validation

Alright, lets get to something meaty! Let's follow Jimmy's lead and bake in some Fluent Validation support.

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 ValidationPipeline.cs %}

Just like Jimmy's example we have a way to apply validation logic in consistent manner with just a single location at play. One thing that is not directly obvious in my example is how we end up handling the _contravariant_ aspect of validation that Jimmy points out. For that, I'll have to break out into our more complex model of supporting dynamic message models and how to build more complex filters. Which again, will happen at the end.

## Authorization

And now the authorization step:

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 AuthorizingPipeline.cs %}


## Pre/Post Processing

Hopefully, by now you can see how this would be put together following Jimmy's initial lead but using GreenPipes infrastructure. So, now lets finally dig into that complex model I kept talking about.

## The Overview at the End

In the end, once I've written all of this up and I'm read to share it with my team, it might look something like this.

{% gist drusellers/cda975d202e263dc7f6ee31c1d906404 clean.cs %}

## Finally, The Deep Dive

Ok, so I've eluded to a more complex pattern that can be used to set up a sharable GreenPipe. Well here we go.

From its origins GreenPipe is the infrastructure code that was used to build the composable pipelines for MassTransit. Because MT is a _framework_ not a _library_ we needed to allow for arbitrarily complex middleware. This framework has allowed us to add Sagas, State Machines, Quartz.net based timeout support, Retry mechanisms, multiple transport support, and a host of other features without having to muddy up the core code of MassTransit which continues to be about handling serialization, transport and middleware concerns of a message based pipeline.

Because of these requirements our filters require a bit more set up than what we see with MediatR. This is an extra cost, but its one that we believe provides an immense amount of value in our ability to "scale with complexity."

### The Fluent Validation Filter

And here is how we could build a complete filter for Fluent Validation with an error pipeline for alternate processing.

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 %}

Let's break this down.

#### The Context

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 0_ValidationFailureContext.cs %}

The context for any GreenPipes pipeline is the heavy lifter of the data. In MassTransit we work a lot with the `ReceiveContext<TMessage>` which has as one of its properties the venerable `Message`. Now, in our scenario, we wrap the existing context and then attach the validation failures. We do this so that the downstream `ValidationFailurePipe` can decide what to do with them. That's right, its another pipe. We will get to that in more detail a bit down the road.

#### The Extension Method

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 0_FluentValidationExtensionMethods.cs %}

Like all things in MassTransit, the extensions to the core framework show up as extension methods. In this case GreenPipe's exposes some very low level functions that are pretty chatty and abstract. The recommended approach is to instead provide an extension method that papers over this complexity for your users. You can see this pattern in TopShelf, MassTransit and now GreenPipes used to a lot of benefit.

#### The Specification

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 1_FluentValidationSpecification.cs %}

This is one of my favorite pieces of the puzzle. GreenPipes forces you to make a specification class. This object holds the data from the extension method and allows you to populate your filter with that data. This by itself was pretty annoying for me. It felt like SRP for the sake of SRP. Until I wrote my first complex filter and came to _REALLY_ appreciate the `Validate()` method.

The `Validate()` method gives you chance to validate the users input in building out filter. Additionally, it gives you a means to report errors back to the user in a structured fashion and these will get checked at bus start.

#### The Filter

{% gist drusellers/27c1834368ebd7fa6425b21912da8358 2_FluentValidationFilter.cs %}

I know its taken a while to get here, but we have finally arrived at the actual filter. At first blush this should seem very familiar as its almost a verbatum copy of what Mr. Bogard wrote up. I'd rather instead focus on the two big differences in the GreenPipe filter.

First, we have the `Probe` method. This method provides a mechanism for dynamically inspecting the pipe at run time. This is the heart of the diagnostics in MassTransit and it was pulled out so that any one building on top of MassTransit can use it. Here's some example output from a unit test.

```json
{
  "resultId": "7e6480b3-3583-4b77-b26d-3c4bf85b3832",
  "probeId": "47845c20-5bc9-4733-9bfa-abd80ad168db",
  "startTimestamp": "2016-10-30T17:05:35.9366782Z",
  "duration": "00:00:00.0130561",
  "host": {
    "machineName": "DESKTOP-TI6NFQ2",
    "processName": "JetBrains.ReSharper.TaskRunner.CLR45.x64",
    "processId": 6392,
    "assembly": "GreenPipes",
    "assemblyVersion": "1.0.3.0",
    "frameworkVersion": "4.0.30319.42000",
    "greenPipesVersion": "1.0.3.0",
    "operatingSystemVersion": "Microsoft Windows NT 10.0.14393.0"
  },
  "results": {
    "serilog": {
      "greenPipeRequestType": "GreenPipes.Tests.JimmyPost.BusinessContext"
    },
    "fluentValidation": {
      "filters": {
        "filterType": "delegate"
      }
    },
    "filters": {
      "filterType": "inline"
    }
  }
}
```

Above, we can see the results of our serilog filter, our fluentValidation filter and the inline filter that I'm using for the security filter. By taking the extra time to build our filters the GreenPipes' way we can expose a dynamically rich amount of data in a structured format. Currently, this uses JSON.Net to serialize data and you could easily use JSON.Net to then deserialize this data back into C# objects for use in your project's dashboard.

Ok, so `Probe` is awesome. The code for Fluent Validation looks just like MediatR but what is this other pipe?? Well, one final difference between GreenPipes and MediatR is that MediatR returns. The MediatR signature is `TResponse Handle(TRequest message)` this is gloriously simple and easy to grok, another benefit of the MediatR approach. We can also see that MediatR allows you to throw exceptions so that you can catch them when invoking MediatR. GreenPipes allows for a more complex model, again out of needs born in MassTransit. But, if you are willing to pay for this extra complexity you will get the ability to _jump the tracks_. What I mean is you can exit any given pipe and start down another pipe (any one want [Railway Programming](http://fsharpforfunandprofit.com/rop/) in C#?).

We can see that there is this other pipe for Validation Failures. If we fail validation, we immediately stop processing (by not calling `next.Send`) and divert down the `_validationFailurePipe`. Now this is a completely different pipe that can do all manner of things. I've used it to still save my users data, but then send out an email to that user alerting them to issues (please note, I rarely write UI oriented code). MassTransit has used this to take messages that have become poisonous and divert them to a poison message queue. All of this and more and you can control just how far and deep you want to go at each level. This is the power you get in exchange for the complexity of not having a direct return value.

> That said, you may be looking at me ಠ_ಠ like, "Dru, I _NEED_ return values." I completely understand and I've felt that pain. I present to you my very simple pattern for handling this. Whatever you want for a return value, just stuff it in the `cxt` variable, then if you `await` the `Send` you can pull that data back out of the `PipeContext`. BOOM.

Ok, whew. That was a whole lot of content to just compare two libraries that are trying to make pipelines easier.

All of this said, my hat goes off to Mr. Bogard. He has created a very nice library in MediatR and I have personally enjoyed its benefits more than once.
