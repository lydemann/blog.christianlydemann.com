---
title: "OpenID Connect Hybrid Flow for calling client api (OIDC Part 4)"
date: 2018-04-07T18:23:09+02:00
draft: false
---

Last post we created an authorization code client, enabling the client to get
the user claims from the id token, exchanged for the post login authorization
code. That way we were able to display the user roles on an authorized MVC view.
This time, instead of getting the user roles from the userInfo endpoint directly
with an access token from AppClient, we are going to get it from the resource
API, using an access token.

Note: This could also have been implemented with authentication code client,
since we are only using an MVC client in this part. Using the hybrid client here
is just for learning purpose, enabling you later to use the hybrid client both
from a browser and a MVC app simultaneously.

## AuthorizationServer - Setting up the hybrid client

Compared to part 3 of the OIDC series, the only thing we need to do to the
AuthorizationServer is to add a hybrid client:

**AuthorizationServer/Config.cs** 
{{< highlight csharp >}} 
// OpenID Connect hybrid flow client (MVC) new Client { ClientId = "mvc", ClientName = "MVC
Client", AllowedGrantTypes = GrantTypes.Hybrid,
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
},
AllowOfflineAccess = true
}
{{< /highlight >}}

The only difference between this and the previous authorization code client is
that browsers are here allowed to get the id token on first roundtrip. We could also get the access token on the first roundtrip if we enabled that here. With authorization code flow, they needed to exchange their authorization code for an id token.

## ClientApp - Calling the resource api with access token

In part 3 we already set the AppClient up for using hybrid flow by adding the ClientSecret in the Startup authentication middleware. The only difference is
here we are requesting the id_token as well as the authorization code on first
request:

**ClientApp/Startup.cs** 
``options.ResponseType = "code id_token";``

After the user has been authenticated, the *HTTPContext* is going to contain an
access token as well as the id token. We use this in a method for calling the resource api’s authenticated identity endpoint:

**ClientApp/Controllers/IdentityController** 
{{< highlight csharp >}} private
async Task CallApiUsingUserAccessToken() { 
    var accessToken = await HttpContext.GetTokenAsync("access_token");

var client = new HttpClient();
client.SetBearerToken(accessToken);

return await client.GetAsync("http://localhost:5001/api/identity");
} {{< /highlight >}}

This gets the access_token from the HttpContext, which is provided by the
authentication cookie that got set on successful user authentication. After
that, the access token is set and send as a bearer token. This method is used by
the IdentityController.Index action:

**ClientApp/Controllers/IdentityController**
{{< highlight csharp >}} [Authorize] public async Task Index() { var
userClaimsVM = new UserClaimsVM(); var userClaimsWithClientCredentials = await
GetUserClaimsFromApiWithClientCredentials();
userClaimsVM.UserClaimsWithClientCredentials =
userClaimsWithClientCredentials.IsSuccessStatusCode ? await
userClaimsWithClientCredentials.Content.ReadAsStringAsync() :
userClaimsWithClientCredentials.StatusCode.ToString();

var userClaimsWithAccessToken = await CallApiUsingUserAccessToken();
userClaimsVM.UserClaimsWithAccessToken = userClaimsWithAccessToken.IsSuccessStatusCode ? await userClaimsWithAccessToken.Content.ReadAsStringAsync() : userClaimsWithAccessToken.StatusCode.ToString();

return View(userClaimsVM);
} 
{{< /highlight >}}

The user claims from the resource are stored in the *userClaimsVM* and the
Identity/Index.cshtml should now look like this:

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
    <dt>
        UserClaimsWithAccessToken
    </dt>
    <dd>
        @Model.UserClaimsWithAccessToken
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

{{< /highlight>}}

Now the view is also showing user claims from the resource API using access token.

No change is needed for the resource api, so we are now ready to run the
servers together.

## Running the app

When running the three servers all together we should be able to get an access token using the: `HttpContext.GetTokenAsync("access_token")`.

By going to <https://jwt.io/>, this JWT can easily be decoded.

![OpenId access token](/images/openid-connect-part1/access-token.PNG)

We see that the scope is containing resourceApi as well as the audience
property, allowing the user to access the ResourceApi.

When going to http://localhost:5002/identity we should now be prompted for login
and consent for which we are gonna use one of the hard coded test users. Here
after we are gonna be redirected to our identity view, now containing the user
claims, fetched from the ResourceAPI:

![OpenId identity view screenshot](/images/openid-connect/oidc-part4-identity-view.PNG)

# Conclusion

That sums up part 4!
We accomplished to setup an hybrid flow client in the Authorization Server that can return both a authorization code as well as id token and access token, if requested.
Using this we obtained an access token containing the resource api as scope, which enabled our ClientApp to fetch and display authorized identity information by requesting the resource API.

Next time we are gonna make the setup more realistic by enabling user registration using ASP.NET identity.\
The code for this part can be found on my [Github](https://github.com/lydemann/oidc-angular-identityserver/tree/master/Solution%203%20-%20OIDC%20with%20Hybrid%20flow%20and%20call%20api).