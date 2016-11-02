---
layout: post
title:  "Greasy Testing"
date:   2013-12-09 09:58:21
categories: testing
disqus_id: 612fe097-a8e3-4ebb-aa31-b54ae4501783
---

When I accepted the job to work for Dovetail Software, one of the things that
I was most interested to learn about was the mythical testing infrastructure
that they had built up. Over the last two years, I have studied the code and
this is my attempt to extract the pattern from that team and share it with the
greater community.

## Specs

Project Size: Larger projects, or projects with critical testing needs

Investment Time: Extensive

## The Pieces

__Testing Framework__: In our case we use NUnit, what you need is a standard
way to run a set of tests. I am personally most familiar w/ xUnit style frameworks
but anything like that should be acceptable.

__AppContext__: this is what loads up the application context. For us, a lot of
this in the IoC Container - I have no idea what this is going to look like
in non-IoC languages. However the point of it is that you should be able to
bootstrap your application from something as simple as newing up an object. All
of the parameters come from configuration or are overriden at test start.

### Usage
{% highlight csharp %}
[Test]
public void TestSomething()
{
  AppContext.Bootstrap();
}
{% endhighlight %}

{% highlight csharp %}
public class AppBootstrapper
{
  public static void Bootstrap()
  {
    //initialize the two members below
  }

  public Application Application {get;set;}
  public IContainer Container {get;set;}
}
{% endhighlight %}

__UI Drivers__: these are focused pieces of code that interact directly with the
raw API (this could be WatiR, selenium, the rest api, or the code api). The
raw API can change quickly and suddenly (especially when you own the api) so
we need a layer to protect us from that. The example of the driver is a
small piece of code that can enter 'a value' into an Html form element. That
class will be very small, but the datepicker might be a bit tricker and will
depend highly on which datepicker you choose to use. This is the second level
of abstraction (the first being a tool like Selenium).

{% highlight csharp %}
public class TextBoxDriver
{
  IWebDriver _driver;
  string _selector;

  public TextBoxDriver(IWebDriver driver, string selector)
  {
    _driver = driver;
    _selector = selector;
  }

  public void Value(string input)
  {
      _driver.GetElement(_selector).Text(input);
  }
}
{% endhighlight %}

A more complex one might look like:


{% highlight csharp %}
public class DatePickerDriver
{
  IWebDriver _driver;
  string _selector;

  public NavigationDriver(IWebDriver driver, string selector)
  {
    _driver = driver;
  }

  public void Value(string input)
  {
      //where the input field is hidden in something else
      _driver.GetElement(_selector)
        .Find("input[type='hidden']")
        .Value(input);
  }
}
{% endhighlight %}

So now, if you change your date time picker code, you only have to go to one
place to update everything. Nice and dry.

__The Navigation Driver__: While this guy is not a UI type driver, it is a
critical piece to keeping your application testing sane. All this driver does
is work to get you on to the screen that you want. A key wrinkle in this driver
is if you say you want to go to a given url and you are not logged in, it will
take care of the login for you and then get you on that screen. This helps
remove needless noise in your test code.


{% highlight csharp %}
public class NavigationDriver
{
  IWebDriver _driver;
  LoginScreen _login;

  public NavigationDriver(IWebDriver driver, LoginScreen _login)
  {
    _driver = driver;
    _login = login;
  }

  public StatusCode NavigateToUrl<TScreen>()
  {
      var url = _urls.GetUrlFor<TScreen>();
      _driver.NavigateTo(url);
      //here it could check to see if on the logon screen
      //and it could deal with that.
      if(_login.IsPresent())
      {
        _login.LoginAsDefaultUser();
      }

      //verify that you were redirected
      _driver.EnsureAtUrl(url);
  }
}
{% endhighlight %}

__Converters__: Entity converters make testing life much easier as well by
taking in simple text phrases and converting them into test objects.

{% highlight csharp %}
public interface IEntityConverter<TEntity>
{
  TEntity Convert(string identifier);
}

public class CustomerConverter : IEntityConverter<Customer>
{
  IRepository _repository;

  public CustomerConverter(IRepository repository)
  {
    _repository = repository;
  }

  public Customer Convert(string indentifier)
  {
    return _repository.Find<Customer>(x=>x.Identifier == identifier);
  }
}
{% endhighlight %}

This guys are a powerful part of the system, but they require a lot of baking
to really become useful. You need to be able to express your testing commands
as strings so that under the covers, these guys can get involved to build out
the correct objects.

__Screens__: I dislike this name because not everything is a screen, but until
I can come up with a better word its what I will use. A screen simply orchestrates
a series of steps to acheive a higher level goal. For example, we have a
logon screen. This screen orchestrates entering a username and password and
submitting the form. The screen can also check to see that login was successful
or not and report back on errors visible on the screen.


{% highlight csharp %}

//LoginRequest is an 'input model' in FubuMVC speak
public class LoginScreen : Screen<LoginRequest>
{
  public LoginScreen(){}

  //notice its action based - rather than UI element based
  public void Login(string username, string password)
  {
    GetDriverFor<TextboxDriver>(x=>x.Username).Value(username);
    GetDriverFor<PasswordDriver>(x => x.Password).Value(password);
  }
}

{% endhighlight %}

There is a lot going on in the above example. Input models, our abstraction
on top of input models for DOM querying, and .Net generics (if you are new to
those).

## Putting it all together

Testing Framework (like NUnit)

- external shell of app context and screens

    - inside the shell with navigation and ui drivers

        - inside the drivers with the browser abstraction


## Benefits

The team has put a lot of time and effort investing into this pattern /
framework for our testing infrastructure. I can say with comfort that this
level of investment lets them make drastic changes to their entire system with
little effect to the testing code. Examples of this include replacing
view engines, changing out UI widgets, renaming low level constructs in the
application, etc. The bottom line is they are willing to make about any change
they deem fit, even after 4 years of active development on the product.
