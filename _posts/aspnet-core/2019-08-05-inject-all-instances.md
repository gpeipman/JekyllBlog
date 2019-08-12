---
layout: post
title:  "Inject all instances of interface"
date:   2019-08-05 08:00:00
categories: ['ASP.NET Core']
permalink: /aspnet-core/inject-all-instances/
---

We all probably know how to inject instance of interface to class using dependency injection in ASP.NET Core. But how can we inject all instances of interface to some class, let’s say controller? The trick is simple and it’s shown in this blog post.

> NB! Code here is written on ASP.NET Core 3.0 Preview 7 but is also works with previous ASP.NET Core versions.

## Getting started

Let’s start with some sample classes to play with during this experiment. Suppose we have interface for high-level alerts and we have two implementations for it – one for sending SMS and other for sending e-mail. Also we have customer class.

``` csharp
public class Customer
{
    public string Name { get; set; }
    public string Email { get; set; }
    public string Mobile { get; set; }
}
 
public interface IAlertService
{
    void SendAlert(Customer c, string title, string body);
}
 
public class EmailAlertService : IAlertService
{
    public void SendAlert(Customer c, string title, string body)
    {
        if(string.IsNullOrEmpty(c.Email))
        {
            return;
        }
 
        Debug.WriteLine($"Sending e-mail to {c.Email} for {c.Name}");
    }
}
 
public class SmsAlertService : IAlertService
{
    public void SendAlert(Customer c, string title, string body)
    {
        if(string.IsNullOrEmpty(c.Mobile))
        {
            return;
        }
 
        Debug.WriteLine($"Sending SMS to {c.Mobile} for {c.Name}");
    }
}
```

These implementations are registered in ConfigureServices() method of Startup class.

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllersWithViews();
    services.AddRazorPages();
 
    services.AddScoped<IAlertService, EmailAlertService>();
    services.AddScoped<IAlertService, SmsAlertService>();
}
```

We have now everything prepared to start playing with dependency injection.

## How to inject single instance

We have two instances for one interface registered. What happens when we use the most popular single dependency injection?

``` csharp
public class HomeController : Controller
{
    private readonly IAlertService _alertService;
 
    public HomeController(IAlertService alertService)
    {
        _alertService = alertService;
    }
}
```

We get back exactly one instance and it’s the last one we registered with dependency injection.

![alt text]({{ site.baseurl }}/images/2019/08/aspnet-core-di-single-instance.png.webp "Logo Title Text 1") 

But what about all registered types?
How to inject all instances to HomeController

We have to modify the code above a little bit to make it work with multiple instances. The trick is to use IEnumerable<IAlertService> instead of IAlertService like in code below.

``` csharp
public class HomeController : Controller
{
    private readonly IEnumerable<IAlertService> _alertServices;
 
    public HomeController(IEnumerable<IAlertService> alertServices)
    {
        _alertServices = alertServices.ToList();
    }
 
    public IActionResult Index()
    {
        var customers = new[] {
            new Customer { Name = "John", Email = "john@example.com" },
            new Customer { Name = "Jane", Mobile = "00-12345" },
            new Customer { Name = "Klaus", Email = "klaus@example.com", Mobile = "00-54321"}
        };
 
        var title = "Service alert";
        var body = "The roof is on fire!";
 
        foreach(var customer in customers)
        foreach(var alertService in _alertServices)
        {
            alertService.SendAlert(customer, title, body);
        }
 
        return View();
    }
}
```

When we run the application we will see the following output in output window.

![alt text]({{ site.baseurl }}/images/2019/08/aspnet-core-di-multi-instance.png.webp "Logo Title Text 1") 

As we can see then both implementations of IAlertService were called and therefore injected to HomeController.

It also works with instances from different scopes. If we change ConfigureServices() method of Startup class like shown below we will get the same result as before.

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllersWithViews();
    services.AddRazorPages();
 
    services.AddSingleton<IAlertService, EmailAlertService>();
    services.AddScoped<IAlertService, SmsAlertService>();
}
```

The solution for injecting all instances of interface is so nice and clean that it should be the first one that comes to our minds without even searching from internet.

## Wrapping up

Another problem in ASP.NET Core and another brilliant solution by product team. When using IEnumerable<IInterface> instead of IInterface we get automatically all instances of IInterface registered with ASP.NET Core dependency injection and this way we can inject all istances of interface to different classes.