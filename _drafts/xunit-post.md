---
layout: post
title:  "Dependency Injection in xUnit test framework"
---

There are certainly cases when developers want to run integration tests against their code that are wired up heavily by dependency injection. [xUnit](https://xunit.net) is a mature, loved-by-developers, and robust test framework for testing .net code. A slight downside of xUnit is that it lacks an out-of-the-box solution to inject dependencies. Fortunately, xUnit has been equipped with "Fixture" feature which is meant to share contexts among test classes. So, this feature can be leveraged to retain dependency injection containers among test classes in an xUnit-powered test project.

There are several steps involved in utilizing dependency injection in an xUnit test project.

## Defining a class to retain dependency injection provider's instance

We need to define a class to instantiate Microsoft's dependency injection service provider and add services and configurations to it. We'd call this class `TestBedFixture` and have it implement `IDisposable` and optionally `IAsyncDisposable` interface(s). We require to add two key methods to this class to facilitate accessing the wired up services' instances. They are per the following code snippets:

```csharp
public T GetScopedService<T>(ITestOutputHelper testOutputHelper)
{
    var serviceProvider = GetServiceProvider(testOutputHelper);
    using var scope = serviceProvider.CreateScope();
    return scope.ServiceProvider.GetService<T>();
}

public T GetService<T>(ITestOutputHelper testOutputHelper)
    => GetServiceProvider(testOutputHelper).GetService<T>();
```

[This source code](https://github.com/Umplify/xunit-dependency-injection/blob/main/src/Abstracts/TestBedFixture.cs) illustrates a full-fledge implementation of a functional fixture class whose instance can be used by other test classes.

## Defining a fixture class

The next step is to decorate the test class with `IClassFixture` interface to leverage the dependency injection's scontainer to acquire services' instances by invoking `GetService` or `GetScopedService` methods. [This code snippet](https://github.com/Umplify/xunit-dependency-injection/blob/main/src/Abstracts/TestBed.cs) illustrates this approach.

[This open source C# project](https://github.com/Umplify/xunit-dependency-injection) facilitates using Microsoft dependency injection in xUnit test projects.

To use this project in actual test projects, simply install the NuGet package of [`Xunit.Microsoft.DependencyInjection`](https://www.nuget.org/packages/Xunit.Microsoft.DependencyInjection/) from https://NuGet.org.

```
Install-Package Xunit.Microsoft.DependencyInjection
```