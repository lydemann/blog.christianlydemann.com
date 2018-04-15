---
title: "OpenID Connect with Angular 5 (OIDC Part 7)"
date: 2018-04-12T21:54:58+02:00
draft: true
---

Now we are getting somewhere, huh!? So far we have played around with the authorization code, hybrid and client credentials flow clients to get a grasp of when to use what. We have made the authorization server interactive and dynamic by saving user data and IdentityServer data in the database.

In this part, we are gonna implement the Angular app on the client app to use an implicit flow client for authenticating and calling an authorized endpoint on the resource API. As spoken about in the first part, we are using implicit flow here because we are using an Angular app, which runs in the browser and can’t keep a client secret that is needed if we wanted to use authorization code in exchange for a refresh token

In the first part, I wrote about the security concerns of Implicit flow and how to mitigate them, and Angular is beneficial for this purpose in the following ways:

- Build in XSS protection (if not doing manual dom manipulation, which is an Angular anti-pattern anyway).
- Official OpenID connect approved implementations of the OIDC standard for the client according to the specification. It is more error-prone to implement the OpenID connect standard ourselves, with stuff like token validation, implementing validation rules etc.


# AuthorizationServer

For this part, the authorization server needs an implicit flow client for the Angular application.

## Setup implicit grant client for Angular app

We go to the Config.cs file and add the following client to the AuthorizationServer’s Config.cs:

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

Here we are creating a client for single page applications (SPAs) like Angular. This is using implicit grant type and is allowed to reveal the access token to a browser. As usual, it also specifies the redirect URLs and the scopes contained in the access token.

Remember, in the previous part we set up dynamic configuration with Entity Framework, containing a seed method reading this Config.cs file for populating the database if not already populated. If your database is already populated from the previous part, remember to delete it to get it seeded with this new client.

# ClientApp

## Setup Angular app with Angular 5

ASP.NET Core has built-in support for Angular apps. In part 2 we scaffolded ClientApp as an ASP.NET application with Angular setting it up with Angular 4. Angular 4 is so last week, so we are going to upgrade that to Angular 5, but we need to change a few things:

- The node_modules should be removed and be reinstalled with all @angular packages being updated to Angular 5 and use at least rxjs version 5.5*
- Update Webpack to use the AngularCompilerPlugin instead of the now deprecated AotPlugin. After this update, the vendor and app bundles should be rebuild and replace the old ones.

### Update to Angular 5

First we remove the node_modules folder and replace your npm dependencies with the following:
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

What has changed here is we have installed the Angular 5 libraries and updated dependencies like *rxjs* accordingly.

### Update webpack to use AngularCompilerPlugin

In Angular 5 ahead of time compilation is default, so the Webpack config file should use AngularCompilerPlugin instead of AotPlugin:

[**ClientApp/webpack.config.js**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/webpack.config.js)
{{< highlight javascript>}}
...
const AotPlugin = require('@ngtools/webpack').AngularCompilerPlugin;
...
{{< /highlight >}}

After this we delete the "ClientApp/wwwroot/dist" folder and rebuild it by opening the terminal at the ClientApp project and run the following command:

``npm run build``

## Inject AppSettings into the Angular app:

Doing server-side rendering enables us to inject environment variables into the Angular app at runtime instead of needed a separate build for each environment (the Angular CLI way).

This way the server and client can also share the same configuration values without duplication.
We setup the environment variables in the AppSettings.json file like this:

[**ClientApp/AppSettings.json**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/appsettings.json)
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

I like type safety, so I use a little trick, where I map the configuration to a typed configuration class and register it in the ASP.NET Core IoC:

{{< highlight csharp >}}
            var appSettings = new AppSettings();
            Configuration.Bind(appSettings);
            services.AddSingleton(appSettings);
{{< /highlight >}}

Now the app settings can be injected and used as a view model in the Index view:

{{< highlight csharp >}}
        public AppSettings AppSettings { get; }

        public HomeController(AppSettings appSettings)
        {
            AppSettings = appSettings;
        }

        public IActionResult Index()
        {
            return View(AppSettings);
        }
{{ /csharp }}


The app settings get passed through the index view model into the view and injected into the Angular app by setting it as the prerender data:

[ClientApp/Views/Home/Index.cshtml](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/Views/Home/Index.cshtml)
{{< highlight html>}}
<app asp-prerender-module="ClientApp/dist/main-server"
    asp-prerender-data='new {
       apiUrl = Model.ApiUrl,
       authUrl = Model.AuthUrl
     }'>Loading...</app>
{{< /highlight >}}

This makes the data available inside the Angular applications server boot method. From here we can pass the app settings data as "globals", getting the settings set in the global window variable:

[**ClientApp/ClientApp/boot.server.ts**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/ClientApp/boot.server.ts)
{{< highlight javascript >}}
export default createServerRenderer(params => {
    const providers = [
        { provide: INITIAL_CONFIG, useValue: { document: '<app></app>', url: params.url } },
        { provide: APP_BASE_HREF, useValue: params.baseUrl },
        { provide: 'ORIGIN_URL', useValue: params.origin },
        { provide: 'BASE_URL', useValue: params.origin + params.baseUrl },
        { provide: 'API_URL', useValue: params.data.apiUrl },
        { provide: 'AUTH_URL', useValue: params.data.authUrl },
    ];

    return platformDynamicServer(providers).bootstrapModule(AppModule).then(moduleRef => {
        const appRef: ApplicationRef = moduleRef.injector.get(ApplicationRef);
        const state = moduleRef.injector.get(PlatformState);
        const zone = moduleRef.injector.get(NgZone);

        return new Promise<RenderResult>((resolve, reject) => {
            zone.onError.subscribe((errorInfo: any) => reject(errorInfo));
            appRef.isStable.first(isStable => isStable).subscribe(() => {
                // Because 'onStable' fires before 'onError', we have to delay slightly before
                // completing the request in case there's an error to report
                setImmediate(() => {
                    resolve({
                        html: state.renderToString(),
                        globals: { urlConfig: params.data }
                    });
                    moduleRef.destroy();
                });
            });
        });
    });
});
{{< /highlight >}}

Now, these settings are taken from window and set up as a provider:

[**AppClient/AppClient/app/app.browser.module.ts**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/ClientApp/app/app.browser.module.ts)
{{< highlight javascript >}}
@NgModule({
    bootstrap: [ AppComponent ],
    imports: [
        BrowserModule,
        AppModuleShared
    ],
    providers: [
        { provide: 'ORIGIN_URL', useFactory: getBaseUrl },
        { provide: 'API_URL', useFactory: apiUrlFactory },
        { provide: 'AUTH_URL', useFactory: authUrlFactory },
    ]
})
export class AppModule {
}

export function getBaseUrl() {
    return document.getElementsByTagName('base')[0].href;
}

export function apiUrlFactory() {
    return (window as any).urlConfig.apiUrl;
}

export function authUrlFactory() {
    return (window as any).urlConfig.authUrl;
}
{{< /highlight >}}

## Setup the Angular app for OIDC

We set up OpenID connect in the Angular with the specification approved library called **angular-auth-oidc-client**. This gives us an easy abstraction to use in our Angular application that implements the validation rules according to the OpenID connect specification.

We install this using:
``npm I angular-auth-oidc-client``

### Setup Auth service

We create a new service called Auth for wrapping the authentication and authorization logic, including the oidc client library. I'm following the Angular style guide which puts global singleton services into a folder named “Core”.

We create a new folder named core and create an auth.service.ts file.

#### Configure OIDC Client

We configure the OIDC client so it requests the right tokens, scopes, does the right redirects and knows how to locate the endpoints on the authorization server:

[**ClientApp/ClientApp/app/components/core/auth.service.ts**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/ClientApp/app/components/core/auth.service.ts)
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

#### Create HTTP helpers for authorized requests

We create methods for doing HTTP calls that set the Authorization header with the bearer token:

[**ClientApp/ClientApp/app/components/core/auth.service.ts**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/ClientApp/app/components/core/auth.service.ts)
{{ highlight javascript }}
    get(url: string): Observable<any> {
        return this.http.get(url, { headers: this.getHeaders() });
    }

    put(url: string, data: any): Observable<any> {
        const body = JSON.stringify(data);
        return this.http.put(url, body, { headers: this.getHeaders() });
    }

    delete(url: string): Observable<any> {
        return this.http.delete(url, { headers: this.getHeaders() });
    }

    post(url: string, data: any): Observable<any> {
        const body = JSON.stringify(data);
        return this.http.post(url, body, { headers: this.getHeaders() });
    }

    private getHeaders() {
        let headers = new HttpHeaders();
        headers = headers.set('Content-Type', 'application/json');
        return this.appendAuthHeader(headers);
    }

    private appendAuthHeader(headers: HttpHeaders) {
        const token = this.oidcSecurityService.getToken();

        if (token === '') return headers;

        const tokenValue = 'Bearer ' + token;
        return headers.set('Authorization', tokenValue);
    }
{{< /highlight >}}

We create a login and logout methods:

[**ClientApp/ClientApp/app/components/core/auth.service.ts**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/ClientApp/app/components/core/auth.service.ts)
{{< highlight javascript >}}
    login() {
        console.log('start login');
        this.oidcSecurityService.authorize();
    }

    logout() {
        console.log('start logoff');
        this.oidcSecurityService.logoff();
    }
{{< /highlight >}}

## Request authorized weather data

The weather data is fetched from the Resource API by calling its endpoint with the authService get method:

[**ClientApp/ClientApp/app/components/fetchdata/fetchdata.component.ts**](https://github.com/lydemann/oidc-angular-identityserver/blob/master/Solution%206%20-%20OIDC%20and%20Angular%20client/ClientApp/ClientApp/app/components/fetchdata/fetchdata.component.ts)
{{< highlight javascript >}}
    ngOnInit(): void {
        this.authService.get(this.apiUrl + '/api/SampleData/WeatherForecasts').subscribe(result => {
            this.forecasts = result as WeatherForecast[];
        }, (error) => {
            console.error(error);
        });
    }
{{ /highlight }}

# Resource API
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

In this part, we got our system setup with an Angular client using an implicit flow OpenID connect client. We used an Angular library approved by the OpenID connect standard for easily plugging the Angular app into the OpenID connect setup. This made the Angular app able to authenticate and be authorized to request an authorized resource on the resource API.

This is concluding the OpenID connect with Angular series. I hope you learned a lot and it gave you a grasp of the different variations of the OpenID connect implementation, including the different flow types, scopes and hands-on implementations of this. OpenID connect is extremely relevant today as it is the most common standard for auth, implemented in almost all big companies in one variation or another.
You really got an advantage knowing this stuff now on a practical level as many people talk about this in vague terms because they don’t really know what’s going on under the helmet. You do now; no more buzzword bingo!

