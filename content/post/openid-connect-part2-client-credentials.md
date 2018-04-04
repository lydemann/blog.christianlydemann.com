---
title: "Creating identity server setup with client credential authentication (OIDC part 2)"
date: 2018-03-29T16:07:57+02:00
draft: false
---


In this post we are gonna take [part 1]({{< relref "openid-connect-part1-openid-connect-overview.md" >}}) into action by creating a OpenID connect setup with a three server system using client credentials for authentication The three servers are:

1. AuthorizationServer, implemented with IdentityServer4.
2. ResourceApi, implemented with ASP.NET core and IdentityServer4.AccessTokenValidation Nuget package for access token validation.
3. ClientApp, implemented as an ASP.NET MVC application with Angular using IdentityModel for getting access token.

The basic idea is that we register an in memory client and api resource on the AuthorizationServer, hardcode the client credentials in the ClientApp and exchanging these for an access token, which will grant the user access to an authorized endpoint on the ResourceApi.

**## Authorization server**

Setup the authorization server by creating a new ASP.NET core project (empty) with .NET core 2.0. Install IdentityServer4 by opening the Nuget console and write:

`Install-Package IdentityServer4`

or find the package on Nuget and click install.

In this part we are simply setting up a hardcoded api resource and test clients by creating a new file called Config.cs and specify these as:

```csharp
public class Config
{
   public static IEnumerable<ApiResource> GetApiResources()
   {
       return new List<ApiResource>
       {
           new ApiResource("resourceApi", "API Application")
       };
   }

   public static IEnumerable<Client> GetClients()
   {
       return new List<Client>
       {
           new Client
           {
               ClientId = "clientApp",

               // no interactive user, use the clientid/secret for authentication
               AllowedGrantTypes = GrantTypes.ClientCredentials,

               // secret for authentication
               ClientSecrets =
               {
                   new Secret("secret".Sha256())
               },

               AllowedScopes = { "resourceApi" }
           }
       };
   }
}
```

Above is the Config.cs file which is specifying an ApiResource with the name "resourceApi" and a client that has this api resource in its AllowedScopes. Because we are using basic authentication in this part we are setting the AllowedGrantTypes to ClientCredentials.

The client app and secret are also set here.

These are set as in memory resources in the Startup.cs with AddInMemoryApiResources and AddInMemoryClients:

```csharp
public void ConfigureServices(IServiceCollection services)
{
   services.AddIdentityServer()
   .AddDeveloperSigningCredential()
   .AddInMemoryApiResources(Config.GetApiResources())
   .AddInMemoryClients(Config.GetClients());
}
```

Note: AddDeveloperSigningCredential is for generating a random RSA256 pair on startup, which we will use for this demo. In a production setup, AddSigningCredential should be used to reference the real RSA256 keys.

Finally the IdentityServer middleware are setup with:

```
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
   if (env.IsDevelopment())
   {
       app.UseDeveloperExceptionPage();
   }
   app.UseIdentityServer();
}

```

## ResourceApi

The ResourceApi will contain an authorized endpoint for getting the claims of the access token. We create a new ASP.NET core web api and call it ResourceApi.

To enable validation of access token we install the Nuget package: ```IdentityServer4.AccessTokenValidation```

For this, we will first need to setup bearer authentication middleware in Startup.cs:

*ResourceApi/Startup.cs*
```
public void ConfigureServices(IServiceCollection services)
{
   services.AddMvcCore()
       .AddAuthorization()
       .AddJsonFormatters();
   services.AddAuthentication("Bearer")
       .AddIdentityServerAuthentication(options =>
       {
           options.Authority = "http://localhost:5000";
           options.RequireHttpsMetadata = false;
           options.ApiName = "resourceApi";
       });
}
```

This is setting up authorization, with the “AddAuthorization” middleware, for enabling making endpoint authorized with the "Authorized" attribute, as well as setting up middleware for bearer authentication. We specify that we use identityServer with AddIdentityServerAuthentication. The Autority is the baseUrl of the AuthorizationServer and this is used for getting the public key when the Resource server is verifying and validating the access token for an authorized request. Future requests will use an in memory cached public key for verifying the access token.\\
ApiName should match the name of the ApiResource we created for the Authorizationserver.
Now the authentication and authorization is setup for ResourceApi, so we create a new controller called IdentityController.cs and gives it an endpoint with the [Authorize] attribute, enforcing that the requester is authorized to access this:

```csharp
[Route("[controller]")]
[Authorize]
public class IdentityController : ControllerBase
{
   [HttpGet]
   public IActionResult Get()
   {
       return new JsonResult(from c in User.Claims select new { c.Type, c.Value });
   }
}
```

This is returning the Users claims as json if the requester is authorized.

## Client App

In this part, we are not using Angular, but because we are using it later, we will scaffold a new ASP.NET core MVC app with Angular.

We are here going to create a new controller called IdentityController.cs for authenticating using client credentials and thus obtaining an access token used for invoking the authorized endpoint at the ResourceApi.

For getting the access token from the AuthorizationServer, we are using the Nuget package:

``IdentityModel``

We are going to create an IdentityController endpoint called index:

*ClientApp/Controllers/IdentityController.cs*
```csharp
public async Task<IActionResult> Index()
{
   // discover endpoints from metadata
   var oidcDiscoveryResult = await DiscoveryClient.GetAsync("http://localhost:5000");
   if (oidcDiscoveryResult.IsError)
   {
       Console.WriteLine(oidcDiscoveryResult.Error);
       return Json(oidcDiscoveryResult.Error);
   }
   // request token
   var tokenClient = new TokenClient(oidcDiscoveryResult.TokenEndpoint, "clientApp", "secret");
   var tokenResponse = await tokenClient.RequestClientCredentialsAsync("resourceApi");
   if (tokenResponse.IsError)
   {
       Console.WriteLine(tokenResponse.Error);
       throw new HttpRequestException(tokenResponse.Error);
   }
   Console.WriteLine(tokenResponse.Json);
   Console.WriteLine("\n\n");
   // call api
   var client = new HttpClient();
   client.SetBearerToken(tokenResponse.AccessToken);
   var response = await client.GetAsync("http://localhost:5001/identity");
   if (!response.IsSuccessStatusCode)
   {
       Console.WriteLine(response.StatusCode);
       throw new HttpRequestException(response.StatusCode.ToString());
   }
   else
   {
       var content = await response.Content.ReadAsStringAsync();
       Console.WriteLine(JArray.Parse(content));
       return Json(content);
   }
}
```

This method is getting the User claims from the authorized endpoint at the resource server by performing three steps:

1. Use DiscoveryClient to get the TokenEndpoint.
2. Use the TokenEndpoint + client credentials and resource scope to request an access token with RequestClientCredentialsAsync.
3. Set the obtained access token with SetBearerToken and invoke the authorized ResourceApi endpoint.

# Running it all together

If we right click on our solution and click properties we can select "Multiple startup projects" and make our three projects run simultaneously. Hereafter we can click F5 and run all the projects. If we in the browser navigate to “[http://localhost:5002/identity](http://localhost:5002/identity)” we should be able to be authentication and authorized to access the authorized resource using the access token, obtained from the client credentials:
![OpenId screenshot](/images/openid-connect-part2/oidc-screenshot.png)

The whole solution for this part can be found on my Github [here]( https://github.com/lydemann/oidc-angular-identityserver/tree/master/Solution%201%20-%20Setup%20OIDC%20system%20with%20client%20credentials).

Next part we are gonna make the IdentityServer interactive by creating and sign up/login feature.
