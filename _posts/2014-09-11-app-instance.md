---
layout: post
title:  "Application Instance"
date:   2014-09-11 05:58:21
categories: architecture
published: true
---

So, if [IoC is the Context](% post_url 2013-12-09-ioc-as-context %}) how do we
leverage this concept so we can package it up nicely in our code bases. The goal
is to use this context to unlock new aspects.

I have an application factory class that generates application instances.

The factory is responsible for bootstrapping the logging, doing all of the
assembly scanning, then passes a [TypePool](% post_url 2014-09-10-typepool %}) and the container into the application.

{% highlight csharp %}
public static class AppFactory
{
  public static Application Build<TApplication>() where TApplication : ApplicationMarker, new()
  {
    //boot strap logging

    var pool = new TypePool();
    //collect all assemblies for this host

    var container = new Container();
    //build up container

    return new ApplicationInstance(container, pool);
  }
}
{% endhighlight %}

The application instance looks like this

{% highlight csharp %}
public interface Application : IDisposable
{
  void Start();
  TComponent Resolve<TComponent>();
  void Scope(Action<ILifetimeScope> action);
  void Dispatch(Request request);
}
{% endhighlight %}

The Start method looks like

{% highlight csharp %}
public class ApplicationInstance : Application
{
  public void Start()
  {
    //run db migrations

    //run all bootup code
  }

  //other stuff
}
{% endhighlight %}

The ApplicationMarker looks like


{% highlight csharp %}
public interface ApplicationMarker
{
  void ConfigureContainer(TypePool pool, ContainerBuilder builder);
}
{% endhighlight %}

Benefits include super simple testing
a predictable and shared common architecture across command line apps
web apps / messaging apps (this is very nice for larger companies)

In integration tests I can say things like:

{% highlight csharp %}
public class SampleTest
{
  [Test]
  public void Test()
  {
    var app = AppFactory.Build<MyApplication>();
    var sut = app.Resolve<TheSystemToTest>();
    var result = sut.TheMethodToTest(some, parameters);
    result.ShouldNotBeNull();
  }
}
{% endhighlight %}

Now I know that my app instance is the same as how its going to be built in
in test as it is in the application host.
