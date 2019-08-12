---
layout: post
title:  "Lightweight custom authentication with ASP.NET Core"
date:   2019-05-06 08:00:00
categories: ['ASP.NET Core']
permalink: /aspnet-core/custom-authentication-aspnet-core/
---

ASP.NET Core Identity is popular choice when web application needs authentication. It supports local accounts with username and password but also social ID-s like Facebook, Twitter, Microsoft Account etc. But what if ASP.NET Core Identity is too much for us and we need something smaller? What if requirements make it impossible to use it? Here’s my lightweight solution for custom authentication in ASP.NET Core.

We don’t have to use ASP.NET Core Identity always when we need authentication. I have specially interesting case I’m working on right now.

> I’m building a site where users authenticate using Estonian ID-card and it’s mobile counterpart mobile-ID. In both cases users is identified by official person code. Users can also use authentication services by local banks. Protocol is different but users are again identified by official person code. There will be no username-password or social media authentication.

In my case I don’t need ASP.NET Core Identity as it’s too much and probably there are some security requirements that wipe classic username and password authentication off from table.

## Configuring authentication

After some research it turned out that it’s actually very easy to go with cookie authentication and custom page where I implement support for those exotic authentication mechanisms.

First we have to tell ASP.NET Core that we need authentication. I’m going with cookie authentication as there’s no ping-pong between my site and external authentication services later. Let’s head to ConfigureServices() method of Startup class and enable authentication.

``` csharp
public void ConfigureServices(IServiceCollection services)
{ 
    // Enable cookie authentication
    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
            .AddCookie();
 
    services.AddHttpContextAccessor();
    services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
}
```

In Configure() method of Startup class we need to add authentication to request processing pipeline. The line after comment does the job.

``` csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    // ...
 
    // Add authentication to request pipeline
    app.UseAuthentication();
 
    app.UseStaticFiles();            
 
    app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });
}
```

No we have done smallest part of work – ASP.NET Core is configured to use cookie authentication.

## Implementing AccountController

Where is user redirected when authentication is needed? We didn’t say anything about it Startup class. If we don’t specify anything then ASP.NET Core expects AccountController with AccessDenied and Login actions. It’s bare minimum for my case. As users must be able to log out I added also Logout() action.

Here’s my account controller. Login() action is called with SSN parameter only by JavaScript that performs actual authentication. There’s always SSN when this method is called (of course, I will add more sanity checks later). Notice how I build up claims identity claim by claim.

``` csharp
public class AccountController : BaseController
{
    private readonly IUserService _userService;
 
    public AccountController(IUserService userService)
    {
        _userService = userService;
    }
 
    [HttpGet]
    public IActionResult Login()
    {
        return View();
    }
 
    [HttpPost]
    public async Task<IActionResult> Login(string ssn)
    {
        var user = await _userService.GetAllowedUser(ssn);
        if (user == null)
        {
            ModelState.AddModelError("", "User not found");
            return View();
        }
 
        var identity = new ClaimsIdentity(CookieAuthenticationDefaults.AuthenticationScheme);
        identity.AddClaim(new Claim(ClaimTypes.Name, user.Ssn));
        identity.AddClaim(new Claim(ClaimTypes.GivenName, user.FirstName));
        identity.AddClaim(new Claim(ClaimTypes.Surname, user.LastName));
 
        foreach (var role in user.Roles)
        {
            identity.AddClaim(new Claim(ClaimTypes.Role, role.Role));
        }
 
        var principal = new ClaimsPrincipal(identity);
        await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal);
 
        return RedirectToAction("Index","Home");
    }
 
    public async Task<IActionResult> Logout()
    {
        await HttpContext.SignOutAsync();
 
        return RedirectToAction(nameof(Login));
    }
 
    public IActionResult AccessDenied()
    {
        return View();
    }
}
```

And this is it. I have now cookie-based custom authentication in my web application.

To try things out I run my web application, log in and check what’s inside claims collection of current user. All claims I expected are there.

![alt text]({{ site.baseurl }}/images/2019/05/aspnet-core-custom-cookie-authentication.png.webp "Logo Title Text 1") 

## Wrapping up

ASP.NET Core is great on providing the base for basic, simple and lightweight solutions that doesn’t grow monsters over night. For authentication we can go with ASP.NET Core Identity but if it’s too much or not legally possible then it’s so-so easy to build our own custom cookie-based authentication. All we did was writing few lines of code to Startup class. On controllers side we needed just a simple AccountController where we implemented few actions for logging in, logging out and displaying access denied message.