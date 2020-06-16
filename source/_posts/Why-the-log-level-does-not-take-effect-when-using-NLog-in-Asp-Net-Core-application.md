---
title: Asp.Net Core中使用NLog日志时NLog路由不生效只输出Info级别日志的问题
tags:
  - Asp.Net Core
  - NLog
categories: 正常的文章
date: 2020-06-16 21:44:44
---

## 问题由来

在一次将`Asp.net Core`默认日志换成`NLog`时，发现`NLog`配置文件中的设置不生效？具体的来说就是在`NLog`文件中设置的路由以及对应的日志级别只有在`Info`或者以上时才生效，而`Debug`、`Trace`级别则不会有日志输出。比如我的`NLog`配置：

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.nlog-project.org/schemas/NLog.xsd NLog.xsd"
      autoReload="true" throwExceptions="false" throwConfigExceptions="true"
      internalLogLevel="Warn" internalLogFile="${basedir}/logs/internal.log">

  <variable name="logDirectory" value="${basedir}/logs"/>

  <extensions>
    <add assembly="NLog.Web.AspNetCore"/>
  </extensions>

  <targets>
    <default-wrapper xsi:type="BufferingWrapper" bufferSize="100"/>
    <target xsi:type="File" name="file" fileName="${logDirectory}/${shortdate}-${level}.log" encoding="utf-8"
            layout="${longdate}|${uppercase:${level}}|${logger}|${message} ${exception:format=tostring}" />
  </targets>
  <targets>
    <default-wrapper xsi:type="AsyncWrapper">
      <wrapper-target xsi:type="RetryingWrapper"/>
    </default-wrapper>
    <target xsi:type="ColoredConsole" name="console" detectConsoleAvailable="true"
            layout="${longdate}|${uppercase:${level}}|${logger}|${message}"/>
  </targets>

  <rules>
    <logger name="Microsoft.*" minlevel="Debug" writeTo="console"/>
    <logger name="WebApplication.*" minlevel="Trace" writeTo="console"/>
    <logger name="*" minlevel="Info" writeTo="file" final="true"/>
  </rules>

</nlog>
```

按照`rules`中的设置，`Microsoft`命名空间下`Debug`级别的日志应该都可以输出在控制台中的，但是启动后控制台的输出只有：

```log
2020-06-16 20:41:34.9142|INFO|Microsoft.Hosting.Lifetime|Now listening on: https://localhost:5001
2020-06-16 20:41:34.9353|INFO|Microsoft.Hosting.Lifetime|Now listening on: http://localhost:5000
2020-06-16 20:41:34.9353|INFO|Microsoft.Hosting.Lifetime|Application started. Press Ctrl+C to shut down.
2020-06-16 20:41:34.9353|INFO|Microsoft.Hosting.Lifetime|Hosting environment: Development
2020-06-16 20:41:34.9353|INFO|Microsoft.Hosting.Lifetime|Content root path: D:\workspace\csharp\WebApplication\WebApplication
```

注意这里的格式已经应用了`NLog`中设置的布局，这说明配置文件被加载了并没有问题，但为什么日志级别的设置不生效呢？

## 原因所在

其实是因为`appsettings.{env}.json`中日志级别的设置覆盖了`Nlog`中的设置。在我们创建一个`Asp.Net Core`项目时，一般会帮我们创建好两个`appsettings.json`文件：

```json appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

```json appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  }
}
```

> 注意这里使用.net core3.1创建的模板，2.1创建的项目命名空间以及对应日志级别会有所不同。

从这两个文件很容易发现默认的日志级别为`Information`并且`Microsoft`命名空间下的日志级别被设置成了`Warning`，当然也就覆盖了`NLog`中的设置，为什么会覆盖呢？可以思考一下。并且这里的设置能够影响到`NLog`中的设置还有一个前提：将`NLog`添加到容器，并且是以`DI`的方式来获取`ILogger`：

```csharp
private static IHostBuilder CreateHostBuilder(string[] args)
{
    return Host.CreateDefaultBuilder(args)
        .ConfigureLogging(logBuilder =>
        {
            logBuilder.ClearProviders()
            .AddNLog("NLog.config");
        })
        .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>());
}
```

如果你是这样添加`NLog`，然后使用`DI`在需要记录日志的类中注入`ILogger`的话，那很可能就中招了。

```csharp
private static readonly NLog.Logger _logger = NLog.LogManager.GetCurrentClassLogger();
```

如果你不使用`DI`来注入而是使用上面这个方式来获取`logger`，那么就不存在这个问题。但是我们使用容器主要的目的不就是容器可以更方便的管理对象之间的依赖吗？更具体地说那就是`DI`。那这个问题该怎样解决呢？

## 解决方法

解决的方法其实很简单：

首先删除`appsettings.{env}.json`中日志相关的设置，然后在注册`NLog`时设置最低日志级别为你所用到的最低级别（因为就算去掉了这些设置，默认的日志级别也是`Information`），这样一来，就相当于把日志级别控制和输出都交给`NLog`来管理。比如我上面的文件中给自己项目的命名空间设置了`Trace`并且已是最低级别，那么在删除`appsettings.{env}.json`中日志相关的设置后可以这样来添加`NLog`：

```csharp
private static IHostBuilder CreateHostBuilder(string[] args)
{
    return Host.CreateDefaultBuilder(args)
        .ConfigureLogging(logBuilder =>
        {
            logBuilder.ClearProviders()
            .SetMinimumLevel(LogLevel.Trace) // 最低为Trace
            .AddNLog(_configFileRelativePath);
        })
        .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>());
}
```

或者你也可以使用`AddFilter`来细分使得日志输出可以“交到”`NLog`手中。当然，如果你不想删除原有的设置并且不想通过`SetMinimumLevel(LogLevel)`方法来设置最低级别，你也可以在`appsettings.{env}.json`文件中设置命名空间和相应的日志级别来保证`NLog`中的设置不会被覆盖（因为这个文件在启动时会自动被加载）。总之，只需要保证`NLog`中日志级别的设置不会因为该文件或者默认的`Information`级别所覆盖即可。如果不使用`DI`来获取的话那就当我没说。（逃
