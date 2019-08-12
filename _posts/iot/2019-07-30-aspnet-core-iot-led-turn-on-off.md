---
layout: post
title:  "Turning LED on and off with ASP.NET Core 3.0 on RaspberryPi"
date:   2019-07-30 08:00:00
categories: ['IoT','ASP.NET Core']
permalink: /iot/aspnet-core-iot-led-turn-on-off/
---

After getting .NET Core SDK and ASP.NET Core 3.0 work on my RaspberryPi and Windows 10 IoT Core I wanted to try out if I can communicate with some electronics right from web application. It is possible and here is how to do it.

> Sample solution with code shown and discussed here is available at my Github repository gpeipman/AspNetCore3LedOnOff. It has a little bit more code and LED can be controlled from web application home page.

## Getting started with LED blinking

I started with same default web application I created in my blog post ASP.NET Core 3.0 on RaspberryPi and Windows 10 IoT Core. To communicate with GPIO we need .NET Core IoT libraries. It’s in alpha, it’s experimental but it works well. At least for me.

I added my RaspberryPi disk as network drive and then opened my web application from Visual Studio and added NuGet reference to System.Device.Gpio package. Don’t forget to check Include prereleases checkbox as this package has no stable version yet.

Next I took LED blink example from .NET Core IoT libraries sample repository and made sure that sample works on my board. Here’s how I wired things together.

![alt text]({{ site.baseurl }}/images/2019/07/rpi-led_bb_thumb.png "Logo Title Text 1") 

All the blinking work in sample project is done in Main() method of Program.cs file.

``` csharp
public static void Main(string[] args)
{
    var pin = 17;
    var lightTimeInMilliseconds = 1000;
    var dimTimeInMilliseconds = 200;
 
    using (GpioController controller = new GpioController())
    {
        controller.OpenPin(pin, PinMode.Output);
        Console.WriteLine($"GPIO pin enabled for use: {pin}");
        Console.CancelKeyPress += (object sender, ConsoleCancelEventArgs eventArgs) =>
        {
            controller.Dispose();
        };
 
        while (true)
        {
            Console.WriteLine($"Light for {lightTimeInMilliseconds}ms");
            controller.Write(pin, PinValue.High);
            Thread.Sleep(lightTimeInMilliseconds);
            Console.WriteLine($"Dim for {dimTimeInMilliseconds}ms");
            controller.Write(pin, PinValue.Low);
            Thread.Sleep(dimTimeInMilliseconds);
        }
    }
}
```

> Quick hint. I added content of Main () method above to Main() method of my web application and commented out call to CreateHostBuilder() method. It doesn’t matter if it’s web application or not – it’s still .NET Core application.

If everything works and led starts blinking then we are ready for moving to ASP.NET Core 3.0.

## Moving to ASP.NET Core

As I like to go with small steps when playing with new stuff I defined two new controller actions – LedOn() and LedOff(). Sometimes I like to be noobie like my students and these were my controller actions at first run.

``` csharp
public IActionResult LedOn()
{
    using (GpioController controller = new GpioController())
    {
        controller.OpenPin(Pin, PinMode.Output);
        controller.Write(Pin, PinValue.High);
        Thread.Sleep(2000);
    }
 
    return Content("Led on");
}
 
public IActionResult LedOff()
{
    using (GpioController controller = new GpioController())
    {
        controller.OpenPin(Pin, PinMode.Output);
        controller.Write(Pin, PinValue.Low);
    }
 
    return Content("Led off");
}
```

Guess what? It doesn’t work. Okay, it works because there are no errors but LED does nothing. The reason is simple – controller actions dispose GPIO controller class instance and it turns off all the pins automatically.

We need an instance of GPIO controller and we cannot dispose it in controller actions. To make controller actions work I added these two lines to Program class.

``` csharp
public const int LedPin = 17;
public static GpioController Controller = new GpioController();
```

We have now static instance of GPIO controller and constant for pin where LED is waiting.

When application starts we need to open LED pin and make sure that LED is turned off. When application closes we have to dispose GPIO controller. This is my Configure() method of Startup() class after these changes.

``` csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, IHostApplicationLifetime applicationLifetime)
{
    Program.Controller.OpenPin(Program.LedPin, PinMode.Output);
    Program.Controller.Write(Program.LedPin, PinValue.Low);
 
    applicationLifetime.ApplicationStopped.Register(() =>
    {
        Program.Controller.ClosePin(Program.LedPin);
        Program.Controller.Dispose();
    });
 
    // Default code follows
}
```

Controller actions that use static instance of GPIO controller are here.

``` csharp
public IActionResult LedOn()
{
    Program.Controller.Write(Program.LedPin, PinValue.High);
 
    return Content("Led on");
}
 
public IActionResult LedOff()
{
    Program.Controller.Write(Program.LedPin, PinValue.Low);
 
    return Content("Led off");
}
```

Those who want to try out if web application turns LED on and off can use these URL-s:

* On – /Home/LedOn
* Off – /Home/LedOff

Now it works but it’s not ASP.NET Core-ish enough.

## Introducing LedClient class

We all probably know how much bad mess and code smell there will be if raw static instances are used from other layers of application. Not to mention all kind of internal details that are exposed to other parts of application.

As we can now turn LED on and off from controller actions it’s time to make our code clean and follow best practices of our industry.

* Static instance – we don’t have to use static instances from classes on .NET Core as we have dependency injection. We can register some types as static and then inject these to controllers. By the way, .NET Core dependency injection supports also disposing.
* GPIO logic is visible – we can take MVC controller actions as client code that consumes GPIO controller on specific way. Currently MVC knows dirty secrets of GPIO controller world and it means that hardware and web layer are bound together too tight. It’s like screaming for troubles.

I know it’s more code but I want GPIO and MVC world have as less contact as possible. Those who write web side of the application should use some client class without messing directly with GPIO controller and pins.

So I wrote LedClient class that wraps GPIO controller related code.

``` csharp
public class LedClient : IDisposable
{
    private const int LedPin = 17;
 
    private GpioController _controller = new GpioController();
    private bool disposedValue = false;
 
    public LedClient()
    {
        _controller.OpenPin(LedPin, PinMode.Output);
        _controller.Write(LedPin, PinValue.Low);
    }
 
    public void LedOn()
    {
        _controller.Write(LedPin, PinValue.High);
    }
 
    public void LedOff()
    {
        _controller.Write(LedPin, PinValue.Low);
    }
        
    protected virtual void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
                _controller.Dispose();
            }
 
            disposedValue = true;
        }
    }
 
    public void Dispose()
    {
        Dispose(true);
    }
}
```

When we use something through .NET Core dependency injection we need to register it. Here’s ConfigureServices() method of my Startup class.

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllersWithViews();
    services.AddRazorPages();
 
    services.AddSingleton<LedClient>();
}
```

Now we can use constructor injection with HomeController to inject instance of LedClient. Here’s my HomeController with LedClient class.

``` csharp
public class HomeController : Controller
{
    private readonly LedClient _ledClient;
 
    public HomeController(LedClient ledClient)
    {
        _ledClient = ledClient;
    }
 
    public IActionResult Index()
    {
        return View();
    }
 
    public IActionResult LedOn()
    {
        _ledClient.LedOn();
 
        return Content("Led on");
    }
 
    public IActionResult LedOff()
    {
        _ledClient.LedOff();
 
        return Content("Led off");
    }
 
    [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
    public IActionResult Error()
    {
        return View(new ErrorViewModel
        {
            RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier
        });
    }
}
```

This is it. We have now ASP.NET Core 3.0 web application that runs on RaspberryPi and is able to communicate with LED using GPIO.

## Supporting multiple users

Our LedClient class is okay for one-man scenarios until this one man is not too fast pressing F5 in browser window where some of controller actions is open that controls LED. LedClient is also not safe for multi-user scenarios as users may try to turn LED on or off at same time.

We have to guarantee that different requests cannot access LED at same time. The easiest way is to use lock statement to keep requests on the row when they are about to change LED state.

``` csharp
public class LedClient : IDisposable
{
    private const int LedPin = 17;
 
    private GpioController _controller = new GpioController();
    private bool disposedValue = false;
    private object _locker = new object();
 
    public LedClient()
    {
        _controller.OpenPin(LedPin, PinMode.Output);
        _controller.Write(LedPin, PinValue.Low);
    }
 
    public void LedOn()
    {
        lock (_locker)
        {
            _controller.Write(LedPin, PinValue.High);
        }
    }
 
    public void LedOff()
    {
        lock (_locker)
        {
            _controller.Write(LedPin, PinValue.Low);
        }
    }
        
    protected virtual void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
                _controller.Dispose();
            }
 
            disposedValue = true;
        }
    }
 
    public void Dispose()
    {
        Dispose(true);
    }
}
```

Not sure how perfect this solution is but it works and parallel requests cannot write to pin at same time anymore.

## Wrapping up

Running ASP.NET Core 3.0 on RaspberryPi and Windows 10 IoT is supported scenario and using preprelease version of .NET Core IoT Library enables us to write web applications that communicate with GPIO connected devices. For web applications we have to consider also multi-user scenarios and make sure that parallel requests can’t use hardware at same time. Here we went with simple locking but actual locking strategy depends on devices and use cases that application provides.