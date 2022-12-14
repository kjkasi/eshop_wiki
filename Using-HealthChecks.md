ASP.NET Core 2.2 HealthChecks package is used in all APIs and applications of eShopOnContainers.

All applications and APIs expose two endpoints (`/liveness` and `/hc`) to check the current application and all their dependencies. The `liveness` endpoint is intended to be used as a liveness probe in Kubernetes and the `hc` is intended to be used as a readiness probe in Kubernetes.

## Implementing health checks in ASP.NET Core services

Here is the documentation about how implement HealthChecks in ASP.NET Core 2.2:

https://docs.microsoft.com/dotnet/standard/microservices-architecture/implement-resilient-applications/monitor-app-health 

Also, there's a **nice blog post** on HealthChecks by @scottsauber
https://scottsauber.com/2017/05/22/using-the-microsoft-aspnetcore-healthchecks-package/ 

## Implementation in eShopOnContainers

The readiness endpoint (`/hc`) checks all the dependencies of the API. Let's take the MVC client as an example. This client depends on:

* Web purchasing BFF
* Web marketing BFF
* Identity API

So, following code is added to `ConfigureServices` in `Startup`:

```cs
services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddUrlGroup(new Uri(configuration["PurchaseUrlHC"]), name: "purchaseapigw-check", tags: new string[] { "purchaseapigw" })
    .AddUrlGroup(new Uri(configuration["MarketingUrlHC"]), name: "marketingapigw-check", tags: new string[] { "marketingapigw" })
    .AddUrlGroup(new Uri(configuration["IdentityUrlHC"]), name: "identityapi-check", tags: new string[] { "identityapi" });                
return services;
```

Four checkers are added: one named "self" that will return always OK, and three that will check the dependent services. Next step is to add the two endpoints (`/liveness` and `/hc`). Note that the `/liveness` must return always an HTTP 200 (if liveness endpoint can be reached that means that the MVC web is in healthy state (although it may not be usable if some dependent service is not healthy).

```cs
app.UseHealthChecks("/liveness", new HealthCheckOptions
{
    Predicate = r => r.Name.Contains("self")
});
```

The predicate defines which checkers are executed. In this case for the `/liveness` endpoint we only want to run the checker named "self" (the one that returns always OK).

Next step is to define the `/hc` endpoint:

```cs
 app.UseHealthChecks("/hc", new HealthCheckOptions()
{
    Predicate = _ => true,
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

In this case we want to run **all checkers defined** (so, the predicate will always return true to select all checkers).

## Configuring probes for Kubernetes using health checks

Helm charts already configure the needed probes in Kubernetes using the healthchecks, but you can override the configuration provided by **editing the file `/k8s/helm/<chart-folder>/values.yaml`**. You'll see a code like that:

```yaml
probes:
  liveness:
    path: /liveness
    initialDelaySeconds: 10
    periodSeconds: 15
    port: 80
  readiness:
    path: /hc
    timeoutSeconds: 5
    initialDelaySeconds: 90
    periodSeconds: 60
    port: 80
```

You can remove a probe if you want or update its configuration. Default configuration is the same for all charts:

* 10 seconds before k8s starts to test the liveness probe
* 1 sec of timeout for liveness probe (**not configurable**)
* 15 sec between liveness probes calls
* 90 seconds before k8s starts to test the readiness probe
* 5 sec of timeout for readiness probe
* 60 sec between readiness probes calls
