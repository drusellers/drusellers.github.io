---
layout: post
title:  "The Container is the Context"
date:   2013-12-09 05:58:21
categories:
disqus_id: 886929f5-804a-47c9-90f3-37cc870711b3
---

The container is the app context. When I am building a piece of sufficiently
complex piece of software, I try very hard to make the container the holder
of all of the context.

_Context: C# and .Net_

## Example

### For Consoles

{% highlight csharp %}
public class Program
{
  public static int main()
  {
    //autofac example
    var builder = new ContainerBuilder();

    //build the context/container up
    var container = builder.Build();

    return container.GetInstance<ITheProgram>().Run();
  }
}
{% endhighlight %}

### For Web Apps

{% highlight csharp %}
public class Global
{
  static IContainer _container;

  public void App_Start()
  {
    //autofac example
    var builder = new ContainerBuilder();

    //build the context/container up
    _container = builder.Build();

    container.GetInstance<ITheProgram>().Initialize();
  }
}
{% endhighlight %}

By pushing the context/container to the start of the program, I can guarentee
that every part of my application can be found and serviced through the container.
This has allowed for some very powerful composition in my applications. One function
of containers, that I also really enjoy is the concept of child or nested containers.

Child containers are indeed powerful.

It amazes me that having used an IoC container for as long as I have I have
only scratech the surface of the power in child contexts. This post serves
as a place for me to note, what I have since learned.



My App Class
{% highlight csharp %}
public static class App
{
  public static int Run()
  {
    //autofac example
    var builder = new ContainerBuilder();

    //build the context up
    var container = builder.Build();

    //need a dispatcher / routing framework
    var dispatcher = container.GetInstance<IDispatcher>();

    var result = dispatcher.Dispatch(new CommandLineContext(Environment.CommandLine))

    return formatResultsForCommandLine(result);
  }

  static int formatResultsForCommandLine(DispatchResult result)
  {
    //stuff
  }
}
{% endhighlight %}

Ok, there is a lot going on in here. The one thing that I'm gonna guess is new
to most is the concept of a dispatcher. I am still working on this concept / name
but this an idea that was stolen / inspired by [connect.js][connect] in combination
with work [Chris Patterson][phatboyg] did in magnum to build a routing lib and
the web framework [FubuMVC][fubumvc].

## Routing

Why can't .Net have some kick ass middleware? I have seen a lot of fighting over
types and how those hinder things. Who wants to take a dependency on the framework's
types? Are we really so maddingly afraid of dependencies? Maybe, but I have seen
a way around it. Does it take a whole lot more code to make happen? Yes. Is it
worth it? I think so. So here is my basic idea.

### Request / Response

Every interaction with a program is a request / response.

- HTTP Requests and their responses
- Emails are often thought of as fire and forget, but down at the transport level
  its a request / response, if only to the first SMTP server.
- Messaging infrastructure is also thought of acheiving fire and forget, but
  again, down at the transport level, its request / response.
- Executing a command line program. Request / Response.

I find this is true because its all based on TCP/IP rather than UDP. But I
really don't know what I'm talking about and this is all an abstraction anyways.

### Ideas on how this could work

Each transport would have its own idea of a Context, but that each transport
would share the concept of request and response.

#### A command line example

{% highlight csharp %}
public static class App
{
  public static int Run()
  {
    //autofac example
    var builder = new ContainerBuilder();

    //build the context up
    var container = builder.Build();

    //need a dispatcher / routing framework
    var dispatcher = container.GetInstance<IDispatcher>();

    var request = new Request()
      .WithCommandLine()
      .WithEnvironmentVariables()
      .WithAppConfig();

    IResponse response = dispatcher.Dispatch(request);

    return formatResultsForCommandLine(response);
  }

  static int formatResultsForCommandLine(IResponse response)
  {
    //stuff
  }
}
{% endhighlight %}

A Container is key to this whole process. It holds your application context,
but after that its all about how we dispatch the request via our router.



[fubumvc]: http://fubuworld.com/fubumvc
[magnum]: http://github.com/phatboyg/magnum
[phatboyg]: http://phatboyg.com/
[connect]: http://www.senchalabs.org/connect/
