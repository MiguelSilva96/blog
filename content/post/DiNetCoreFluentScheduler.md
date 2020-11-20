---
title: Using ASP.NET Core built-in DI for FluentScheduler
date: 2020-11-15T19:37:46Z
lastmod: 2020-11-15T19:37:46Z
author: Miguel Silva
cover: /blog/tasks.jpg
categories: ["backend-dev"]
tags: ["software", "API", ".NET Core", "backend", "job scheduling"]
draft: true
---

How to use .NET Core built-in dependency injection container with FluentScheduler job regestries.

## What is FluentScheduler?

FluentScheduler is an automated job scheduler with fluent interface for the .NET platform. (refer to https://fluentscheduler.github.io/)
In this post we will focus on using this scheduler specifically in .NET Core.

With FluentScheduler you have to configure the jobs you want to run. For this purpose you usually have to options:
Create a registry that you provide on initialization or else add jobs after initializing. The initialization should be called once, usually when the program starts.

```csharp
    // ... under main for example

    // FIRST APPROACH
    JobManager.Initialize();
    // and then you can add jobs to it
    JobManager.AddJob(() => Console.WriteLine("job!"), (s) => s.ToRunEvery(5).Seconds());
    // -------------------------
    // OR
    // -------------------------
    // SECOND APPROACH
    JobManager.Initialize(registry);
    // But, what is this registry?
```

Now, the registry is a class provided by the FluentScheduler library that allows you to schedule jobs. When you call AddJob as in the above example, all you are doing is adding a Job to the existing registry that was created on Initialize(). You have the option of passing a registry instance as a parameter on initialize and that will be the registry used. You can use a registry with all the jobs you need already registered. You can create a registry and schedule jobs or else, you can create a class that inherits from Registry and schedules all your jobs:

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

As you can see, there are multiple ways you can schedule a job, these examples were retrieved from the FluentScheduler docs. Now that you have a registry, you can understand what is the registry in the second approach of the first example. That registry could be as follows:

```csharp
    var registry = new MyRegistry();
    JobManager.Initialize(registry);
    // All the jobs in the MyRegistry constructor are now scheduled
```


When scheduling a job you have three options, using actions, specifying which job to create or create the job in an action. 

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

Notice that we added a custom job to our DI container because we want it to be created by the container so that its dependencies (dbcontext, services, etc) are injected. Also, notice that this time in our MyRegistry class the constructor receives the Application Services as an argument, let's understand why that is needed.

```csharp
public class MyRegistry : Registry
{
    // Inject the application service provider using pure DI.
    // (see Startup.cs code example)
    public MyRegistry(IServiceProvider sp)
    {
        // Use the service provider to retrieve an instance of the job.
        // If the job depends on a DbContext instance, create a scope and
        // use its service provider, so that each job gets a unique DbContext.
        Schedule(() => sp.CreateScope().ServiceProvider
                         .GetRequiredService<MyJob>())
                         .ToRunNow().AndEvery(5).Minutes();
    }
}
```

Let's have a look at what the MyJob job looks like:

```csharp
public class MyJob : IJob
{
    private readonly string _id;

    public MyJob(IExampleService service)
    {
        _id = Utils.ShortId();
        Console.WriteLine("Hello from MyJob");
    }
    public void Execute()
    {
        Console.WriteLine("Hello from MyJob executing");
    }
}
```