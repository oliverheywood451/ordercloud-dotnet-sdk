# OrderCloud.io SDK for .NET

[![OrderCloud.SDK](https://img.shields.io/nuget/v/OrderCloud.SDK.svg?maxAge=3600)](https://www.nuget.org/packages/OrderCloud.SDK/)

The OrderClould.io SDK for .NET is a client library for building solutions targeting the [OrderCloud.io](https://developer.ordercloud.io/documentation/) eCommerce platform using C# or other .NET language. Compared to accessing OrderCloud.io's REST API via direct HTTP calls, the SDK aims to greatly improve developer productivity and reduce errors by providing discoverable, strongly typed wrappers for all public endpoints and request/response models.

- [Example](#example)
- [Notable Features](#notable-features)
    - [Strongly Typed xp](#strongly-typed-xp)
    - [Strongly Typed Partials](#strongly-typed-partials)
- [FAQ](#faq)
- [Supported Platforms](#supported-platforms)
- [Getting Help](#getting-help)

## Example
```c#
using OrderClould.SDK;

var client = new OrderCloudClient(new OrderCloudClientConfig {
    ClientId = "my-client-id",
    
    // client credentials grant flow:
    ClientSecret = "my-client-secret"
    
    // OR password grant flow:
    Username = u,
    Password = p,
    
    Roles = new[] { ApiRole.OrderAdmin }
});

var orders = await client.Orders.ListAsync(OrderDirection.Incoming, filters: new { Status = OrderStatus.Open });

Console.WriteLine($"{orders.Meta.TotalCount} open orders found.");
Console.WriteLine($"Fetched page {orders.Meta.Page} of {orders.Meta.TotalPages}.");
foreach (var order in orders.Items) {
    Console.WriteLine($"ID: {order.ID}, Total: {order.Total:C}");
}

```

## Notable Features
Here are a few niceties that the SDK provides. Features of the OrderCloud.io _platform_ are documented [here](https://developer.ordercloud.io/documentation).

### Strongly Typed xp
Extended Properties, or `xp`, is a platform feature that allows you to extend the OrderCloud.io data model. This is modeled in the SDK using (by default) a C# [dynamic](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/dynamic):

```c#
var user = new User();
user.xp.Gender = "male";
```

Even though `Gender` does not exist the native model, the code above will compile and work just fine with the API. But with dynamics you don't get compile-time type checking. If this is desired, the SDK gives you an alternative that works like this:

```c#
public class MyUserXp
{
    public string Gender { get; set; }
}

var user = new User<MyUserXp>()
user.xp.Gender = "male"; // strongly typed!
```

This time `Gender` is strongly typed, so you'll get the compile-time checking, Intellisense, autocomplete, etc. that you get with first-class properties. This is also available on calls that GET an object (or list):

```c#
var user = await client.Users.GetAsync<MyUserXp>(buyerId, userId);
Console.WriteLine(user.xp.Gender); // strongly typed!
```

### Strongly Typed Partials

OrderCloud.io supports PATCH endpoints for partially updating a resource. The idea is that if you only send the fields you want to change, you not only reduce the payload, you might also avoid excessive GETs that could be needed if the entire model was required for an update. For example, here's how you might activate an inactive user:

```c#
await client.Users.PatchAsync(buyerId, userId, new PartialUser { Active = true });
```

`PartialUser` is type provided out of the box by the SDK (and similarly for all models with a corresponding PATCH endpoint). A couple things to note about `PartialUser`:

1. It has all the same strongly-typed properties as `User`; in fact, it inherits from `User`.

2. What makes it different from `User` is how it gets serialized to JSON when the the API call is made. Instead of serializing _all_ properties, it only serializes those that are _explicitly set_. (In this example, `{"Active":true}` is sent in the request body.) This behavior is important to understand - once you set a property, you cannot "remove" it, even by setting it to `null`, for example.

## FAQ

#### How do I authenticate?
Although `OrderClouldClient` does provide an `AuthenticateAsync` method, you normally don't need to call it explicitly. As long as you've [configured](#example) `OrderClouldClient` with the correct credentials, authorization will be done implicitly upon your first API call, and the access token will be cached, reused, and refreshed as needed.

#### Do I have to use `async`/`await` when calling endpoints?
Yes. (Isn't it refreshing when the answer isn't always "it depends"?) The SDK uses [Flurl.Http](https://github.com/tmenier/Flurl), which uses `HttpClient`, which does not support synchronous calls (and for good reason). Do _not_ use `.Result` or `.Wait()` on calls made with this SDK, ever. These will block threads and potentially cause deadlocks. In short, [don't block on async code](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html). Use `await`.

#### HttpClient under the hood, eh? So I should use OrderClouldClient as a singleton?
It depends. ;)

The ideal scope is _one instance per set of authorization credentials_. Since the access token is cached and reused by `OrderCloudClient`, creating new instances with the same credentials will result in excessive authorization calls. If you're only using a single set of credentials for the lifetime of your application, using `OrderCloudClient` as a singleton is fully thread-safe. In any case, you do _not_ need to worry about [this infamous problem](https://aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/), because the SDK uses a static, lazily-instantiated singleton instance of `HttpClient`, regardless of how many `OrderCloudClient`s are created.

#### Is it IoC-friendly? Testable?
Yes and yes. All service-y classes implement interfaces (`IOrderCloudClient`, `IUserResource`, `IProductResource`, etc.), making them easily compatible with your favorite IoC container and mocking framework.

## Supported Platforms
The SDK targets .NET Framework 4.5 and [.NET Standard](https://docs.microsoft.com/en-us/dotnet/standard/net-standard)  1.3 and 2.0, meaning it'll run just about everywhere .NET runs, including .NET Core 1.0 and 2.0, Mono, Xamarin (iOS and Android), and UWP 10.

## Getting Help
If you're new to OrderCloud.io, exploring the [documentation](https://developer.ordercloud.io/documentation) is recommended, especially the [Intro to OrderCloud.io](https://developer.ordercloud.io/documentation/platform-guides/getting-started/introduction-to-ordercloud) and [Quick Start Guide](https://developer.ordercloud.io/documentation/platform-guides/getting-started/quick-start-guide). When you're ready to dive deeper, check out the [platform guides](https://developer.ordercloud.io/documentation/platform-guides) and [API reference](https://developer.ordercloud.io/documentation/api-reference).

For programming questions, please [ask](https://stackoverflow.com/questions/ask?tags=ordercloud) on Stack Overflow.

To report a bug or request a feature specific to the SDK, please open an [issue](https://github.com/ordercloud-api/ordercloud-dotnet-sdk/issues/new). 
