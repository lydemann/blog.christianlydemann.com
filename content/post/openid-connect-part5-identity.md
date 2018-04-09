---
title: "OpenID Connect with IdentityServer and ASP.NET Core Identity (OIDC Part 5)"
date: 2018-04-09T18:22:12+02:00
tags: ["OpenID Connect", "Identity Server", "Angular"]
categories:
- OpenID connect
- IdentityServer4
draft: false
---

Great that you made it this far! Now we are getting closer to what would be a “normal” scenario. Until now we have played around with authenticating with client credentials, authorization code flow, and hybrid flow - all with hardcoded test users. Of course, this would not work in a production setup, so we will in this post enable users to register and authenticate (log in) using ASP.NET Core Identity.

ASP.NET Core Identity is a Package for ASP.NET applications that bootstraps the app with support for managing users and easily save them in a database with Entity Framework and Identity middleware.

In this post the client app will still use the hybrid flow client from last part, so we are only doing changes in the authorization server for this part. This part is implemented with inspiration from the official IdentityServer4 documentation [here](https://identityserver4.readthedocs.io/en/release/quickstarts/6_aspnet_identity.html).

# The OpenID connect with IdentityServer4 and Angular series

This series is learning you OpenID connect with Angular with these parts:

- **[Part 1: Creating an OpenID connect system with Angular 5 and IdentityServer4]({{< relref "openid-connect-part1-openid-connect-overview.md" >}})**
- **[Part 2: Creating identity server setup with client credential authentication]({{< relref "openid-connect-part2-client-credentials.md" >}})**
- **[Part 3: Creating interactive authentication with an authorization code client]({{< relref "openid-connect-part3-authorization-code-flow.md" >}})**
- **[Part 4: OpenID Connect Hybrid Flow for calling resource API]({{< relref "openid-connect-part4-hybrid-flow.md" >}})**
- **[Part 5: OpenID Connect with ASP.NET Identity]({{< relref "openid-connect-part5-identity.md" >}})** (this)
- **Part 6: OpenID Connect with Entity Framework for IdentityServer configuration (coming soon)**
- **Part 7: OpenID Connect with Angular client (coming soon)**


# Authorization Server

The easiest way to implement the authorization server for this part is to create a new project in Visual Studio 2017 with ASP.NET Core Identity already setup. To do this we first open Visual Studio 2017 and click New->project. We should now select an ASP.NET Core Web Application like this:

![select-aspnetcoreweb](/images/openid-connect/select-aspnetcoreweb.png)

Then we select "Web Application (Model-View-Controller):

![select-aspnetcoreweb](/images/openid-connect/select-mvc.png)

We want to have it bootstrapped with Core Identity so we click "choose Authentication" and choose "Individual User Accounts":

![select-authentication](/images/openid-connect/select-authentication.png)

Now we click next and this should set up a new MVC application with ASP.NET Identity. Remember to configure this to run on port 5000 like the old AuthorizationServer.

## Add Identity packages

The first we need to do is installing Identity by going on NuGet and find the package called:\
``IdentityServer4.AspNetIdentity``

This contains the *IdentityServer4* package, so we can run the IdentityServer middleware.


## The IdentityServer client

We are gonna use the same IdentityServer client with hybrid flow as we did in the last part, so feel free to copy the AuthorizationServer/Config.cs file to the new project.

## Setting up IdentityServer middleware

Identity middleware is setup with:
**AuthorizationServer/Startup.cs**
{{< highlight csharp >}}
services.AddIdentity<ApplicationUser, IdentityRole>()
              .AddEntityFrameworkStores<ApplicationDbContext>()
              .AddDefaultTokenProviders();
{{< /highlight >}}
This sets up all the routes and controllers that enables user registration, login, logout etc.

Now that Identity server middleware is setup we only need to hook Identity into the IdentityServer middleware with AddAspNetIdentity.
The IdentityServer middleware chain should now look like this:

**AuthorizationServer/Startup.cs**
{{< highlight csharp >}}
services.AddIdentityServer()
              .AddDeveloperSigningCredential()
              .AddInMemoryIdentityResources(Config.GetIdentityResources())
              .AddInMemoryApiResources(Config.GetApiResources())
              .AddInMemoryClients(Config.GetClients())
              .AddAspNetIdentity<ApplicationUser>();

{{< /highlight >}}
The hardcoded test users have now been substituted by user data from the database.

We are also gonna copy the Quickstart folder from the last part to the new project here as this gives us our registration, login and consent pages + controllers.

## Setting up the database for Identity

For setting up persistence of users we are gonna add a new DbContext with Identity’s ApplicationDbContext and make it use SQL server:

**AuthorizationServer/Startup.cs**
{{< highlight csharp >}}
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
{{< /highlight >}}

The AddDbContext<ApplicationDbContext> creates a new DB, configured with tables for user management, giving us a really good foundation for managing our users.

The whole startup file should look like [this](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%204%20-%20OIDC%20with%20Identity/AuthorizationServer/Startup.cs).

To actually create the Identity tables we need to update the database with:\
``dotnet ef database update``

This adds a new migration in the migration in the data folder of the AuthorizationServer project at **AuthorizationServer/Data/**:
![Identity Migrations](/images/openid-connect/identity-migrations.PNG)

We can now see that the database is set up with Identity tables:

![Identity Database](/images/openid-connect/identity-database-tables.PNG)

# Run it

Business as usual here. We select the three project and make them all run together like before. We should now be able to go to [http://localhost:5000/register](http://localhost:5000/register) and register users that can be authenticated by logging in with their credentials.

![OIDC register](/images/openid-connect/register.PNG)

When we go to [localhost:5002/identity](localhost:5002/identity) we should now be redirected to the login page as before. If we click on register we can create a new user and hereafter login and give consent to that user's resources.

And as before we should now be redirected to the Identity View showing our identity information:

![ASP.NET Identity view](/images/openid-connect/oidc-5-aspnet-identity-view.PNG)

# Conclusion

In this part we made the app more realistic by enabling dynamic registration of users instead of using hardcoded test users, using ASP.NET core Identity. This only required a few changes in the authorization server, so this shows how powerful the ASP.NET Core Identity package is.
Next part we are gonna move the hardcoded configuration data into the Database and use this for dynamic configuration of the identity server.
