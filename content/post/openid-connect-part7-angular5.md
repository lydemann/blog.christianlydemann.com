---
title: "OpenID Connect with Angular 5 (OIDC Part 7)"
date: 2018-04-12T21:54:58+02:00
draft: true
---

Creating an Angular client with implicit flow OIDC client, that can call an authorized endpoint on the resource API.

# AuthorizationServer

## Setup implicit grant client for Angular app

**AuthorizationServer/Config.cs**
{{< highlight csharp >}}
                new Client
                {
                    ClientId = "spaClient",
                    ClientName = "SPA Client",
                    AllowedGrantTypes = GrantTypes.Implicit,
                    AllowAccessTokensViaBrowser = true,

                    RedirectUris = { "http://localhost:5002/callback" },
                    PostLogoutRedirectUris = { "http://localhost:5002/home" },
                    AllowedCorsOrigins = { "http://localhost:5002" },
                    AllowedScopes =
                    {
                        IdentityServerConstants.StandardScopes.OpenId,
                        IdentityServerConstants.StandardScopes.Profile, 
                        "resourceApi"
                    }
                }
{{< /highlight >}}

# ClientApp

## Setup Angular app with Angular 5

Remove the node_modules folder and replace your npm dependencies with the following:
[**ClientApp/package.json**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/package.json)
{{< highlight javascript>}}
  "devDependencies": {
    "@ngtools/webpack": "^6.0.0-beta.7",
    "@types/chai": "4.0.1",
    "@types/jasmine": "2.5.53",
    "@types/webpack-env": "1.13.0",
    "angular2-router-loader": "0.3.5",
    "angular2-template-loader": "0.6.2",
    "aspnet-prerendering": "^3.0.1",
    "aspnet-webpack": "^2.0.1",
    "awesome-typescript-loader": "3.2.1",
    "bootstrap": "3.3.7",
    "chai": "4.0.2",
    "css": "2.2.1",
    "css-loader": "0.28.4",
    "es6-shim": "0.35.3",
    "event-source-polyfill": "0.0.9",
    "expose-loader": "0.7.3",
    "extract-text-webpack-plugin": "2.1.2",
    "file-loader": "0.11.2",
    "html-loader": "0.4.5",
    "isomorphic-fetch": "2.2.1",
    "jasmine-core": "2.6.4",
    "jquery": "3.2.1",
    "json-loader": "0.5.4",
    "karma": "1.7.0",
    "karma-chai": "0.1.0",
    "karma-chrome-launcher": "2.2.0",
    "karma-cli": "1.0.1",
    "karma-jasmine": "1.1.0",
    "karma-webpack": "2.0.3",
    "preboot": "4.5.2",
    "raw-loader": "0.5.1",
    "reflect-metadata": "0.1.10",
    "rimraf": "^2.6.2",
    "style-loader": "0.18.2",
    "to-string-loader": "1.1.5",
    "typescript": "^2.8.1",
    "url-loader": "0.5.9",
    "webpack": "2.5.1",
    "webpack-hot-middleware": "2.18.2",
    "webpack-merge": "4.1.0",
    "zone.js": "0.8.12"
  },
  "dependencies": {
    "@angular/animations": "^5.2.9",
    "@angular/common": "^5.2.9",
    "@angular/compiler": "^5.2.9",
    "@angular/compiler-cli": "^5.2.9",
    "@angular/cli": "^1.7.3",
    "@angular/core": "^5.2.9",
    "@angular/forms": "^5.2.9",
    "@angular/http": "^5.2.9",
    "@angular/platform-browser": "^5.2.9",
    "@angular/platform-browser-dynamic": "^5.2.9",
    "@angular/platform-server": "^5.2.9",
    "@angular/router": "^5.2.9",
    "angular-auth-oidc-client": "^4.1.0",
    "rxjs": "^5.5.2"
  }
{{< /highlight >}}

After this do a:\
``npm i``


## Update webpack

In Angular 5 ahead of time compilation is default, so the Webpack config file should use AngularCompilerPlugin instead of AotPlugin:

[**ClientApp/webpack.config.js**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/webpack.config.js)
{{< highlight javascript>}}
...
const AotPlugin = require('@ngtools/webpack').AngularCompilerPlugin;
...
{{< /highlight >}}

## Inject AppSettings into the Angular app:

[**ClientApp/appsettings.json**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/appsettings.json)
{{< highlight javascript>}}
{
    "Logging": {
        "LogLevel": {
            "Default": "Warning"
        }
    },
    "ApiUrl": "http://localhost:5001",
    "AuthUrl": "http://localhost:5000"
}

{{< /highlight >}}

[ClientApp/Views/Home/Index.cshtml](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/Views/Home/Index.cshtml)
{{< highlight html>}}
<app asp-prerender-module="ClientApp/dist/main-server"
     asp-prerender-data='new {
        apiUrl = Model.ApiUrl,
        authUrl = Model.AuthUrl
      }'>Loading...</app>
{{< /highlight >}}



## Setup the Angular app for OIDC

### Setup Auth service

[**ClientApp/ClientAPp/app/components/core/auth.service.ts**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/ClientApp/app/components/core/auth.service.ts)
{{< highlight javascript>}}
const openIdImplicitFlowConfiguration = new OpenIDImplicitFlowConfiguration();
openIdImplicitFlowConfiguration.stsServer = authUrl,
openIdImplicitFlowConfiguration.redirect_url = originUrl + 'callback',
openIdImplicitFlowConfiguration.client_id = 'spaClient';
openIdImplicitFlowConfiguration.response_type = 'id_token token';
openIdImplicitFlowConfiguration.scope = 'openid profile resourceApi';
openIdImplicitFlowConfiguration.post_logout_redirect_uri = originUrl + 'home';
openIdImplicitFlowConfiguration.forbidden_route = '/forbidden';
openIdImplicitFlowConfiguration.unauthorized_route = '/unauthorized';
openIdImplicitFlowConfiguration.auto_userinfo = true;
openIdImplicitFlowConfiguration.log_console_warning_active = true;
openIdImplicitFlowConfiguration.log_console_debug_active = true;
openIdImplicitFlowConfiguration.max_id_token_iat_offset_allowed_in_seconds = 10;

const authWellKnownEndpoints = new AuthWellKnownEndpoints();
authWellKnownEndpoints.issuer = authUrl;
    
authWellKnownEndpoints.jwks_uri = authUrl + '/.well-known/openid-configuration/jwks';
authWellKnownEndpoints.authorization_endpoint = authUrl + '/connect/authorize';
authWellKnownEndpoints.token_endpoint = authUrl + '/connect/token';
authWellKnownEndpoints.userinfo_endpoint = authUrl + '/connect/userinfo';
authWellKnownEndpoints.end_session_endpoint = authUrl + '/connect/endsession';
authWellKnownEndpoints.check_session_iframe = authUrl + '/connect/checksession';
authWellKnownEndpoints.revocation_endpoint = authUrl + '/connect/revocation';
authWellKnownEndpoints.introspection_endpoint = authUrl + '/connect/introspect';
authWellKnownEndpoints.introspection_endpoint = authUrl + '/connect/introspect';

this.oidcSecurityService.setupModule(openIdImplicitFlowConfiguration, authWellKnownEndpoints);
{{< /highlight >}}



Install the angular-auth-oidc-client package and set up the URLs. Create methods for doing HTTP calls that set the Authorization header with the bearer token.

## Resource API
Setup an authorized controller with a method providing weather data.

[**ResourceApi/Controllers/SampleDataController.cs**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ResourceApi/Controllers/SampleDataController.cs)

{{< highlight csharp>}}
[Route("api/[controller]")]
[Authorize]
public class SampleDataController : Controller
{
    private static string[] Summaries = new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };

    [HttpGet("WeatherForecasts")]
    public IEnumerable<WeatherForecast> WeatherForecasts()
    {
        var rng = new Random();
        return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            DateFormatted = DateTime.Now.AddDays(index).ToString("d"),
            TemperatureC = rng.Next(-20, 55),
            Summary = Summaries[rng.Next(Summaries.Length)]
        });
    }

    public class WeatherForecast
    {
        public string DateFormatted { get; set; }

        public int TemperatureC { get; set; }

        public string Summary { get; set; }

        public int TemperatureF
        {
            get
            {
                return 32 + (int)(TemperatureC / 0.5556);
            }
        }
    }
}
{{< /highlight >}}

## Configure Startup.cs for a browser client

Enable CORS for the ClientApp in Startup.cs with CORS middleware:

[**ResourceApi/Startup.cs**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ResourceApi/Startup.cs)
{{< highlight csharp>}}
            services.AddCors(options =>
            {
                options.AddPolicy("default", policy =>
                {
                    policy.WithOrigins(Configuration["clientUrl"])
                        .AllowAnyHeader()
                        .AllowAnyMethod();
                });
            });

{{< /highlight >}}

# Running the app

![OpenID connect with Angular recording](/images/openid-connect/oidc-with-angular.gif)

# Conclusion



