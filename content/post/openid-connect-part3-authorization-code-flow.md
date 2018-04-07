---
title: "OpenID Connect Interactive authentication with Authorization Code Flow (OIDC Part 3)"
date: 2018-04-04T06:26:55+02:00
draft: false
---

Creating interactive authentication with an authorization code client (OIDC part 3)


In [part 2]() we created a simple OIDC setup using hard coded client credentials for the client to obtain an access token, so it could invoke the resource API.
In this post, we are gonna enable interactive login on the identity server with hard coded test users using authorization flow. After the users has successfully logged in, the requested scopes will be provided to the client app using the callback url. This will enable the client app to display the users claims on an authorized MVC view.

To enable this, we will need to do some changes on the authorization server and client app:

## Authorization server - setup interactive login

The easiest way to get started with an interactive login is to download the quickstart controllers and view supporting this by running the below script (as administrator) in the command prompt inside the authorization server folder:

```
iex ((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/IdentityServer/IdentityServer4.Quickstart.UI/release/get.ps1'))
```

After this the authorization server project should have a folder structure like this:
```
├───Quickstart
│   ├───Account
│   ├───Consent
│   ├───Diagnostics
│   ├───Grants
│   └───Home
├───Views
│   ├───Account
│   ├───Consent
│   ├───Diagnostics
│   ├───Grants
│   ├───Home
│   └───Shared
└───wwwroot
    ├───css
    ├───js
    └───lib
        ├───bootstrap
        │   ├───css
        │   ├───fonts
        │   └───js
        └───jquery
```

This provides you with controllers and views for managing accounts, consent and gives you a standard MVC homepage (of course with cringe-worthy bootstrap styling!).

For the login we are going to create an authorization code grant client in the Config.cs file:

**Config.cs**
{{< highlight csharp >}}
// OpenID Connect authorization flow client (MVC)
new Client
{
    ClientId = "mvc",
    ClientName = "MVC Client",
    AllowedGrantTypes = GrantTypes.Code,
    ClientSecrets =
    {
        new Secret("secret".Sha256())
    },
    RedirectUris = { "http://localhost:5002/signin-oidc" },
    PostLogoutRedirectUris = { "http://localhost:5002/signout-callback-oidc" },
    AllowedScopes =
    {
        IdentityServerConstants.StandardScopes.OpenId,
        IdentityServerConstants.StandardScopes.Profile,
        "resourceApi"
    }
}
{{< /highlight >}}

Here we are making this use authorization code flow by setting the *AllowedGrantTypes* to *Code*. Because we use authorization code flow we also need to specify a client secret, to be shared with the client app. The redirect urls are just set to identity server defaults. The allowed scopes is containing the resourceApi as well as two **IdentityResources** resources that are added to the Config.cs file as:

{{< highlight csharp >}}
// scopes define the resources in your system
public static IEnumerable<IdentityResource> GetIdentityResources()
{
    return new List<IdentityResource>
    {
        new IdentityResources.OpenId(),
        new IdentityResources.Profile(),
    };
}
{{< /highlight >}}
Where an APIResource grants access to resources on an API, Identity Resources grants access to information about the user. These Identity Resources enables scopes for getting a unique id of the users, called subject id, as well as users basic profile information such as name, address etc.

We are setting these new configurations up in the Startup.cs file that should look like this:
{{< highlight csharp >}}
public class Startup
{
    // This method gets called by the runtime. Use this method to add services to the container.
    // For more information on how to configure your application, visit https://go.microsoft.com/fwlink/?LinkID=398940
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMvc();

        services.AddIdentityServer()
        .AddDeveloperSigningCredential()
        .AddInMemoryApiResources(Config.GetApiResources())
        .AddInMemoryIdentityResources(Config.GetIdentityResources())
        .AddInMemoryClients(Config.GetClients())
        .AddTestUsers(Config.GetTestUsers());
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseIdentityServer();

        app.UseStaticFiles();
        app.UseMvcWithDefaultRoute();
    }
}
{{< /highlight >}}
Since the last part, we have now added MVC and static files middleware  as well as AddInMemoryIdentityResources for the identity resources.

## Client app - Create authorized view with user information

For the client app we are gonna create an authorized view for displaying user information contained in the id token.

First we are setting the authentication middleware up for our new authorization code flow client:

**ClientApp/Startup.cs**
{{< highlight csharp >}}

JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();
services.AddAuthentication(options =>
    {
        options.DefaultScheme = "Cookies";
        options.DefaultChallengeScheme = "oidc";
    })
    .AddCookie("Cookies")
    .AddOpenIdConnect("oidc", options =>
    {
        options.SignInScheme = "Cookies";

        options.Authority = "http://localhost:5000";
        options.RequireHttpsMetadata = false;
        options.ResponseType = "code";
        options.ClientId = "mvc";
        options.ClientSecret = "secret";
        options.SaveTokens = true;
    });
{{< /highlight >}}

Here we are disabling the Jwt Claim type mapping before we call the middleware`` ``JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();``
Otherwise the claim names would have long URI names instead of how they are in the JWT payload.
For ease, we are using cookies here that will be send to the app client server on every request. It is specified that the client should use authorization flow with:
`` options.ResponseType = "code" ``
We are connecting to the identity server client by specifying the corresponding client id and client secret. *options.SaveTokens = true* is saving the tokens in the authentication cookie. That way the tokens can easily be access from the *HTTPContext*.

The *IdentityController* from last part is to be rewritten a little to return the claims from basic authentication to view by encapsulating the logic in a private method:
**ClientApp/Controllers/IdentityController**
{{< highlight csharp >}}
private async Task<HttpResponseMessage> GetUserClaimsFromApiWithClientCredentials()
{
    // discover endpoints from metadata
    var oidcDiscoveryResult = await DiscoveryClient.GetAsync("http://localhost:5000");
    if (oidcDiscoveryResult.IsError)
    {
        Console.WriteLine(oidcDiscoveryResult.Error);
        throw new HttpRequestException(oidcDiscoveryResult.Error);
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

    // call API
    var client = new HttpClient();
    client.SetBearerToken(tokenResponse.AccessToken);

    var response = await client.GetAsync("http://localhost:5001/api/identity");
    return response;
}
{{< /highlight >}}

The *Index* action is now an authorized endpoint only accessible if the authentication middleware accepts the request (valid authentication of user).

**AppClient/Controllers/IdentityController**
{{< highlight csharp >}}
[Authorize]
public async Task<IActionResult> Index()
{
    var userClaimsVM = new UserClaimsVM();
    var userClaimsWithClientCredentials = await GetUserClaimsFromApiWithClientCredentials();
    userClaimsVM.UserClaimsWithClientCredentials = userClaimsWithClientCredentials.IsSuccessStatusCode ? await userClaimsWithClientCredentials.Content.ReadAsStringAsync() : userClaimsWithClientCredentials.StatusCode.ToString();

    return View(userClaimsVM);
}

{{< /highlight >}}

This is simply getting the user claims from the resourceApi using access token obtained with clientCredential authentication. The data is passed to the view using a simple view model containing a property for user claims with client credentials.

Also, we create a logout action for logging out the user by clearing the authentication cookie and sign out of OpenID connect session:
**ClientApp/Controllers/IdentityController.cs**
{{< highlight csharp >}}
public async Task Logout()
{
    await HttpContext.SignOutAsync("Cookies");
    await HttpContext.SignOutAsync("oidc");
}
{{< /highlight >}}

Finally we create a simple view for showing the client credentials from the resource API alongside with client credentials obtained using the new authorization code client:

**AppClient/Views/Identity/Index.cshtml**
{{< highlight html >}}
@model ClientApp.Models.UserClaimsVM;
@{
    ViewData["Title"] = "Index";
}

<h2>Index</h2>

<dl>
    <dt>
        UserClaimsWithClientCredentials
    </dt>
    <dd>
        @Model.UserClaimsWithClientCredentials
    </dd>
</dl>

User claims:
<dl>
    @foreach (var claim in User.Claims)
    {
        <dt>@claim.Type</dt>
        <dd>@claim.Value</dd>
    }
</dl>

<form asp-controller="Identity" asp-action="Logout" method="post">
    <button type="submit">Logout</button>
</form>

{{< /highlight >}}

Notice that in the bottom of the view we also add a button for logging out the user with the logout action.

# Running it all

We can now run the whole application by right clicking the solution in the Solution Explore->Properties and select *Start* for the three projects as:

![Run all project Visual Studio](/images/openid-connect/oidc-run-multi.PNG)

We should now be able to go to “http://localhost:5002/identity” and be prompted for login, which we do with one of the test users. Hereafter we will be redirected to the identity view looking like:

![OpenId screenshot](/images/openid-connect/identity-view-authcode.PNG)

Great job following along this far!
Next time we will setup an hybrid flow client that can call the resource API.
