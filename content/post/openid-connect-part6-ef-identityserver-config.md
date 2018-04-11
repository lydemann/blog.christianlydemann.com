---
title: "Configure IdentityServer with Entity Framework (Part 6)"
date: 2018-04-11T19:00:15+02:00
draft: false
---

In this post, we are going to build upon our IdentityServer setup with ASP.NET Core Identity for user management by moving the previously hardcoded IdentityServer configuration data to the database. This enables dynamic change of how IdentityServer is configured instead of needed a rebuild of the server for every configuration change. For this, we are gonna use Entity Framework and are going to write a seed script that takes the Config.cs configuration data and populates the database with it.
As in the last post, this post only requires changes to the authorization server - the client app and the resource API stay the same.

# The OpenID connect with IdentityServer4 and Angular series

This series is learning you OpenID connect with Angular in these parts:

- [Part 1: Creating an OpenID connect system with Angular 5 and IdentityServer4]({{< relref "openid-connect-part1-openid-connect-overview.md" >}})
- [Part 2: Creating identity server setup with client credential authentication]({{< relref "openid-connect-part2-client-credentials.md" >}})
- [Part 3: Creating interactive authentication with an authorization code client]({{< relref "openid-connect-part3-authorization-code-flow.md" >}})
- [Part 4: OpenID Connect Hybrid Flow for calling resource API]({{< relref "openid-connect-part4-hybrid-flow.md" >}})
- [Part 5: OpenID Connect with ASP.NET Identity]({{< relref "openid-connect-part5-identity.md" >}})
- [Part 6: OpenID Connect with Entity Framework for IdentityServer configuration]({{< relref "openid-connect-part6-ef-identityserver-config.md" >}}) (this)
- **Part 7: OpenID Connect with Angular client (coming soon)**

## AuthorizationServer - Setup IdentityServer configuration management with Entity Framework

We are so fortunate that IdentityServer has a package on Nuget that gives us DbContexts, that we are using for creating the database for saving IdentityServer configuration data. We install the package on Nuget called:

``IdentityServer4.EntityFramework``

The database is updated with:

``dotnet ef migrations add InitConfigration -c ConfigurationDbContext -o Data/Migrations/IdentityServer/Configuration``
 
``dotnet ef migrations add InitPersistedGrant -c PersistedGrantDbContext -o Data/Migrations/IdentityServer/PersistedGrant``

If the look at the SQL server explorer we can now see that we have created the following tables:

![OIDC db tables](/images/openid-connect/oidc6-db-tables.PNG)

This is where we persist OpenID connect configuration data such as ApiResources, claims, Clients and Grants etc.

We are not gonna hardcode the configuration data anymore, so we will now delete the following middleware from the Startup.cs file in the AuthorizationServer project:

AddInMemoryPersistedGrants
AddInMemoryIdentityResources
AddInMemoryApiResources
AddInMemoryClients

Using the AddConfigurationStore middleware we are going to setup the Startup.cs as:

**AuthorizationServer/Startup.cs**
```csharp
           var migrationsAssembly = typeof(Startup).GetTypeInfo().Assembly.GetName().Name;
           services.AddIdentityServer()
               .AddDeveloperSigningCredential()
               .AddAspNetIdentity<ApplicationUser>()
               .AddConfigurationStore(options =>
               {
                   options.ConfigureDbContext = builder =>
                       builder.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"),
                           db => db.MigrationsAssembly(migrationsAssembly));
               })
               .AddOperationalStore(options =>
               {
                   options.ConfigureDbContext = builder =>
                       builder.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"),
                           db => db.MigrationsAssembly(migrationsAssembly));
               });
```
Here we are setting up an Configuration store with AddConfigurationStore and an OpertionalStore with AddOperationalStore. This works by only providing it with the SqlServer connection string and the migrationsassembly.

Notice that we need to provide the assemblyName for it to locate the migrations assembly. With this, we setup IdentityServer to lookup configuration data in the database.

The IdentityServer config data is now loaded from the Config.cs file into the database with a seed script:

**Data/SeedData.cs**
```csharp
   public class SeedData
   {
       public static void EnsureSeedData(IServiceProvider serviceProvider)
       {
           Console.WriteLine("Seeding database...");
           PerformMigrations(serviceProvider);

           EnsureSeedData(serviceProvider.GetRequiredService<ConfigurationDbContext>());
           Console.WriteLine("Done seeding database.");
       }

       private static void PerformMigrations(IServiceProvider serviceProvider)
       {
           serviceProvider.GetRequiredService<ApplicationDbContext>().Database.Migrate();
           serviceProvider.GetRequiredService<ConfigurationDbContext>().Database.Migrate();
           serviceProvider.GetRequiredService<PersistedGrantDbContext>().Database.Migrate();
       }

       private static void EnsureSeedData(ConfigurationDbContext context)
       {
           if (!context.Clients.Any())
           {
               Console.WriteLine("Clients being populated");
               foreach (var client in Config.GetClients().ToList())
               {
                   context.Clients.Add(client.ToEntity());
               }
               context.SaveChanges();
           }
           else
           {
               Console.WriteLine("Clients already populated");
           }

           if (!context.IdentityResources.Any())
           {
               Console.WriteLine("IdentityResources being populated");
               foreach (var resource in Config.GetIdentityResources().ToList())
               {
                   context.IdentityResources.Add(resource.ToEntity());
               }
               context.SaveChanges();
           }
           else
           {
               Console.WriteLine("IdentityResources already populated");
           }

           if (!context.ApiResources.Any())
           {
               Console.WriteLine("ApiResources being populated");
               foreach (var resource in Config.GetApiResources().ToList())
               {
                   context.ApiResources.Add(resource.ToEntity());
               }
               context.SaveChanges();
           }
           else
           {
               Console.WriteLine("ApiResources already populated");
           }
       }
   }
```

This seed script is called from program.cs with:


and adds or updates the configuration data in the database when the AuthorizationServer is run.

**[AuthorizationServer/Program.cs](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%205%20-%20OIDC%20with%20EF%20for%20configuration/AuthorizationServer/Program.cs)**
{{< highlight csharp >}}
...
using (var scope = host.Services.CreateScope())
{
    var services = scope.ServiceProvider;

    try
    {
        SeedData.EnsureSeedData(services);

    }
    catch (Exception ex)
    {
        var logger = services.GetRequiredService<ILogger<Program>>();
        logger.LogError(ex, "An error occurred while migrating and seeding the database.");
    }
}
...
{{< /highlight>}}

# Running it all

As usual, we can select the AuthorizationServer, AppClient, and ResourceAPI and run them in Visual Studio. The app should run like before, except we can change configuration data dynamically now by changing the database tables containing the IdentityServer configuration data. On the first run we run our seed methods that run our migrations and seed our tables with the IdentityServer configuration data.

For example we can see our client now in the Clients table:

![OpenID Connect database clients](/images/openid-connect/oidc6-db-clients.PNG)

# Wrapping up

In this post, we made the IdentityServer configuration dynamic by using the IdentityServer Entity Framework library to store OpenID connect configuration data in the database. This created new tables for storing the IdentityServer configuration data and we seeded these with a seed script, running upon start.

In the next post, we are going to implement the whole setup with Angular application in AppClient, using an implicit flow client on the Authorization server and being able to access authorized data from ResourceApi.
