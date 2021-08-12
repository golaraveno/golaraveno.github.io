---
title: "OWIN and ASP.NET CORE"
date: 2020-06-19T13:21:04+02:00
draft: true
---

If you have followed the last article regarding process of migrating to asp.net core, we had been discussing a lot about OWIN and its usage amid this goal, I just wanna quickly let you know how easily you can share all your previous codes into asp.net core as well, keep tuned ;)



### How is it possible to use OWIN pipeline in Asp.net core? is it?

The answer is pretty straight forward: yes!

### Let's do it
First, create a simple asp.net core project using command line
I'm using .net core 3.1 though!
```
dotnet new web
```

And then easily make sure everything is working fine run  below command
```
dotnet run
```

If everything is ok so far, then you need to obviously install a couple of more packages to be using OWIN stuff in your project
```
Microsoft.AspNetCore.Owin
Microsoft.Owin
```

Now let's play a bit more with OWIN and create new middleware somehow as well as the below one
```
using System.Threading.Tasks;
using Microsoft.Owin;

public class MyMiddleware : OwinMiddleware
{
    public MyMiddleware(OwinMiddleware next)
        : base(next)
    {
    }
    public async override Task Invoke(IOwinContext context)
    {

        var uri = context.Request.Uri;

        await Next.Invoke(context);

        await context.Response.WriteAsync($"URI: {uri}");
    }
}
```

this middleware is totally capable to be added into our pipeline  however we need to apply some tricks
as a result I have created an extension method, pay attention please :)
```
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Hosting;
using Microsoft.Owin.Builder;
using Microsoft.Owin.BuilderProperties;
using Owin;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

public static class OwinExtensions
    {
        public static IApplicationBuilder UseOwinApp(
            this IApplicationBuilder aspNetCoreApp,
            Action<IAppBuilder> configuration)
        {
            return aspNetCoreApp.UseOwin(setup => setup(next =>
            {
                AppBuilder owinAppBuilder = new AppBuilder();

                IHostApplicationLifetime aspNetCoreLifetime = (IHostApplicationLifetime)aspNetCoreApp.ApplicationServices.GetService(typeof(IHostApplicationLifetime));

                AppProperties owinAppProperties = new AppProperties(owinAppBuilder.Properties);

                owinAppProperties.OnAppDisposing = aspNetCoreLifetime?.ApplicationStopping ?? CancellationToken.None;

                owinAppProperties.DefaultApp = next;

                configuration(owinAppBuilder);

                return owinAppBuilder.Build<Func<IDictionary<string, object>, Task>>();
            }));
        }
    }

```

then change startup class and apply OWIN pipeline there:
```
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Owin;

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseOwinApp(owinApp =>
        {
            owinApp.Use<MyMiddleware>();
        });
    }
}

```

run the application, you should be able to see the result, it works well, but we are not done, also you are capable to add ASP.NET core middleware side by side, lets take a look at this one either
```
public class MyCoreMiddleware
    {
        private readonly RequestDelegate _next;

        public MyCoreMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task Invoke(HttpContext httpContext)
        {
            await _next(httpContext);
        }
    }
```

then apply your middleware into your pipeline as well as your OWIN middleware
```
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            app.UseOwinApp(owinApp =>
            {
                owinApp.Use<MyMiddleware>();
            });

            app.UseMiddleware<MyCoreMiddleware>();
            
            //etc
        }
```

Neat! ha?

## Conclusion
You are not meant to be using OWIN pipeline anymore if you are about to create a new project, all this is supposed to be used when you want to update your old fashioned asp.net app to core one!
I have read another article here for more information

https://alikhalili.me/blog/posts/migrating-to-asp-net-core/


and also I had written an article regarding OWIN about 4 years ago, for more information you can have look there as well
https://www.codeproject.com/Articles/1122162/Implement-Owin-Pipeline-using-Asp-net-Core

Be Safe :)