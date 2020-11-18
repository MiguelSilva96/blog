---
title: Using ASP.NET Core built-in DI for FluentScheduler
date: 2020-11-15T19:37:46Z
lastmod: 2020-11-15T19:37:46Z
author: Miguel Silva
cover: /blog/tasks.jpg
categories: ["backend dev"]
tags: ["software", "API", ".NET Core", "backend", "job scheduling"]
draft: true
---

How to use .NET Core built-in dependency injection container with FluentScheduler job regestries.

## What is FluentScheduler?

FluentScheduler is an automated job scheduler with fluent interface for the .NET platform. (refer to https://fluentscheduler.github.io/)
In this post we will focus on using this scheduler specifically in .NET Core.

With FluentScheduler you have to configure the jobs you want to run. For this purpose you usually have to options:
Create a registry that you provide on initialization or else add jobs after initializing.

```csharp
using FluentScheduler;

public class MyRegistry : Registry
{
    public MyRegistry()
    {
        // Schedule an IJob to run at an interval
        Schedule<MyJob>().ToRunNow().AndEvery(2).Seconds();

        // Schedule an IJob to run once, delayed by a specific time interval
        Schedule<MyJob>().ToRunOnceIn(5).Seconds();

        // Schedule a simple job to run at a specific time
        Schedule(() => Console.WriteLine("It's 9:15 PM now.")).ToRunEvery(1).Days().At(21, 15);

        // Schedule a more complex action to run immediately and on an monthly interval
        Schedule<MyComplexJob>().ToRunNow().AndEvery(1).Months().OnTheFirst(DayOfWeek.Monday).At(3, 0);

        // Schedule a job using a factory method and pass parameters to the constructor.
        Schedule(() => new MyComplexJob("Foo", DateTime.Now)).ToRunNow().AndEvery(2).Seconds();

        // Schedule multiple jobs to be run in a single schedule
        Schedule<MyJob>().AndThen<MyOtherJob>().ToRunNow().AndEvery(5).Minutes();
    }
}
```


For the first option you need to can a Registry class that inherits from Registry in the library and schedule all your intended tasks in the constructor.
The second option is adding after initializing, by simply calling AddJob.

```csharp
JobManager.AddJob(() => Console.WriteLine("Late job!"), (s) => s.ToRunEvery(5).Seconds());
```

When scheduling a job you have three options, using actions, specifying which job to create or create the job in an action. 
To better understand, here are some examples:
...

## Dependency injection in .NET Core

Now this is all fun and games until you need to integrate this in an ASP.NET Core application, using the built-in Dependency Injection to access your 
services or database context. Of course FluentScheduler is a general purpose scheduler and is not responsible for integrating with all dependency injection 
containers available in the .NET platform. So, we need to find a way to make it work, and the tools provided by FluentScheduler are enough for that.

In .NET Core we can initialize the job scheduler in the Startup file.


```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    // This method gets called by the runtime. Use this method to add services to the container.
    public void ConfigureServices(IServiceCollection services)
    {
        //... configure other services...

        // example with sql lite
        services.AddDbContext<MyDbContext>(options =>
            options.UseSqlite(
                Configuration.GetConnectionString("ConnectionString")));
        // ....
        // Register jobs as transient so that a fresh job instance is created
        // for each job execution.
        services.AddTransient<MyJob>();
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        //....

        app.UseHttpsRedirection();

        //...

        // Start jobs by initializing a manually instantiated registry.
        // Pass in application service provider, which the registry will use to
        // get the job instances.
        JobManager.Initialize(new MyRegistry(app.ApplicationServices));
    }
}
```

