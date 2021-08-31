---
versionFrom: 9.0.0
meta.Title: "Umbraco WebApi"
meta.Description: "A guide to implenting WebApi in Umbraco projects"
state: complete
verified-against: beta-4
update-links: true
---

# Umbraco Api

_This section will describe how to work with Web Api in Umbraco to create REST services_

Related links:

* [Umbraco api routes and Urls](routing-v9.md)
* [Umbraco api authorization](authorization-v9.md)

## What is Web API?

The Microsoft Web API reference can be found on the [official ASP.NET Web API website](https://www.asp.net/web-api). *"ASP.NET makes it easy to build services that reach a broad range of clients, including browsers and mobile devices. With ASP.NET you use the same framework and patterns to build both web pages and services, side-by-side in the same project."*

Essentially it's a great platform for building REST services.

A great resource for getting started with creating web API's using .Net Core is the [official Microsoft documentation](https://docs.microsoft.com/en-gb/aspnet/core/web-api/?view=aspnetcore-5.0).

## Web Api in Umbraco

We've created a base api controller for developers to inherit from which will ensure that the api controller gets routed. Unlike V8, this does not expose any specific umbraco related services or objects, but does inherit from the .Net Core controller base, meaning you will have access to the same things you would from a regular .Net Core controller. Dependency injection is also available to controllers, so any Umbraco specific services or objects you might need can be injected in the constructor.

The class to inherit from is: `Umbraco.Cms.Web.Common.Controllers.UmbracoApiController`

## Creating a Web Api controller

There are 2 types of Umbraco Api controllers:

1. A locally declared controller - is **not** routed via an Area.
2. A plugin based controller - is routed via an Area.

When working on your own projects you will normally be creating a locally declared controller which requires no additional steps. However, if you are creating an Umbraco package, to be distributed, you will want to create a plugin based controller so it gets routed via its own area. This ensures that the route will not overlap with someone's locally declared controller if they are both named the same thing.

### Naming conventions

It is very important that you name your controllers according to these guidelines or else they will not get routed:

All controller class names must be suffixed with "**Controller**" and inherit from **UmbracoApiController**. Some examples:

```csharp
public class ProductsController : UmbracoApiController
public class CustomersController : UmbracoApiController
public class ScoresController : UmbracoApiController
```

### Locally declared controller

This is the most common way to create an Umbraco Api controller, you inherit from the class `Umbraco.Cms.Web.Common.Controllers.UmbracoApiController` and that is all. You will need to follow the guidelines specified by Microsoft for creating a Web Api controller, documentation can be found on the [official Microsoft documentation website](https://docs.microsoft.com/en-gb/aspnet/core/web-api/?view=aspnetcore-5.0).

Example:

```csharp
public class ProductsController : UmbracoApiController
{
    public IEnumerable<string> GetAllProducts()
    {
        return new[] {"Table", "Chair", "Desk", "Computer"};
    }
}
```

All locally declared Umbraco api controllers will be routed under the url path of:

*~/Umbraco/Api/[YourControllerName]*

E.g. *~/Umbraco/Api/Products/GetAllProducts*

Note that the "Controller" part of your controller name gets stripped away.

### Plugin based controller

If you are creating an Umbraco Api controller to be shipped in an Umbraco package you will need to add the `Umbraco.Cms.Web.Common.Attributes.PluginController` attribute to your controller to ensure that it is routed via an area. The area name is up to you to specify in the attribute.

Example:

```csharp
[PluginController("AwesomeProducts")]
public class ProductsController : UmbracoApiController
{
    public IEnumerable<string> GetAllProducts()
    {
        return new[] {"Table", "Chair", "Desk", "Computer"};
    }
}
```

Now this controller will be routed via the area called "AwesomeProducts". All plugin based Umbraco api controllers will be routed under the url path of:

*~/Umbraco/[YourAreaName]/[YourControllerName]*

E.g. *~/Umbraco/AwesomeProducts/Products/GetAllProducts*

For more information about areas, Urls and routing see the [routing section](routing-v9.md)

## Backoffice controllers

If you are creating a controller to work within the Umbraco backoffice then you will need to ensure that it is secured properly by inheriting from: `UmbracoAuthorizedApiController` or `UmbracoAuthorizedJsonController`. This controller type will auto-route your controller like the above examples except that it will add another segment to the path: 'backoffice'.


*~/Umbraco/backoffice/Api/[YourControllerName]*

*~/Umbraco/backoffice/[YourAreaName]/[YourControllerName]*


E.g.
*~/Umbraco/backoffice/Api/Products/GetAllProducts* or

*~/Umbraco/backoffice/AwesomeProducts/Products/GetAllProducts* for PluginController

### More Information

* [Authenticating & Authorizing controllers](../Authorized/index-v9.md)

## Using MVC Attribute Routing in Umbraco Web API Controllers

*Attribute routing* uses attributes to define routes. *Attribute routing* gives you more control over the URIs in your web application. 

To use attribute routing, add the `Microsoft.AspNetCore.Mvc.Route` attribute to the controller or controller action you want to route. If you want to attribute route an entire controller you have to add the `[action]` token in order to route to an action, for instance: 

```C#
[Route("products/[action]")]
public class ProductsController : UmbracoApiController
{
    public IEnumerable<string> GetAllProducts()
    {
        return new[] {"Table", "Chair", "Desk", "Computer"};
    }

    public string GetProduct()
    {
        return "Monitor";
    }
}
```

This route the controllers actions like so:

`~/products/GetAllProducts` and `~/products/GetProduct`

If you use the route attribute for a specific action the `[action]` token is not nececary, but you can request parameters from the path in a similar manner, using the `{parameterName}` syntax, for instance: 

```C#
public class ProductsController : UmbracoApiController
{
    public IEnumerable<string> GetAllProducts()
    {
        return new[] {"Table", "Chair", "Desk", "Computer"};
    }

    [Route("product/{id?}")]
    public string GetProduct(int? id)
    {
        if (id is not null)
        {
            return $"Monitor model {id}";
        }
        return "Base model Monitor";
    }
}
```

Here the `GetAllProducts` endpoint will be routed normally, but the `GetProduct` will be routed as `~/product` where you can optionally access it as `~/product/4`, or any other number, if a number is included as the last segment of the path, let's say 4, the action will return "Monitor model 4", otherwise it will just return "Base model Monitor".

This is not anything Umbraco specific, so to read more about attribute routing, see the [routing article on the official Microsoft documentation](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/routing?view=aspnetcore-5.0#attribute-routing-for-rest-apis).