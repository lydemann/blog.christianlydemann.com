---
title: "Setting up an OpenID connect system with Angular 5 and Identity server 4 (part 1)"
date: 2018-03-27T10:13:20+02:00
tags: ["OpenID Connect", "Identity Server", "Angular"]
draft: false
---

OpenID connect authentication with dotnet core and Angular will demonstrate how to setup an app that support authentication and access control of certain resources in the system. This guide is based on the Identity Server [docs](http://docs.identityserver.io/en/release/quickstarts/6_aspnet_identity.html) which seems to favour a setup with a client, an Identity server and an API being with authorized resources. This setup implements the OpenID connect standard which enables single sign on and distributed access control.

OpenID connect is a standard adding authentication (verifying the user’s identity) on top of OAUTH2, which is only for authorization (access control). OpenID connect adds authentication by introducing the notion of an ID token, which is is an JWT, providing a signed proof of authentication of the user.

## Why use OpenID Connect?

1. OpenID connect is very useful for centralizing authentication and authorization in an infrastructure with many microservices, enabling them to easily hook into the auth setup.
2. OpenID connect is a common standard, making it easy for team members to collaborate with a widely used and well documented standard. It also make it very easy for new employees to understand the architecture if it's build around a common standard.
3. Lots of good [tools](http://openid.net/developers/certified/) are making the implementation of this standard easy, like IdentityServer.

## The three flows of OpenID connect

In OpenID connect there are three flows, all based on the value of the response_type in the login request:

* Authorization code with response_type: 'code' (auhtorization code that can be exchanged for tokens and refresh token in another round trip)
* Implicit flow with response_type: 'id_token' and 'id_token token' (get the ID token or the ID token and access token)
* Hybrid flow with response_type: 'code id_token', 'code token' and 'code id_token token' (always get the authorization code as well as either ID token and/or access token in first response)

### Authorization code flow

This flow is the most commonly used flow used by traditional web apps with a server backend and contains two round trips:

1. Getting authorization code.
2. Exchange authorization code for tokens, including refresh tokens.

The flow is initiated with the response_type parameter set to code and a client secret shared between the client and the auth server in the login request. After the user has been logged in, the authorization endpoint on the authorization server sends the authorization code (using query params in a redirect), which can be exchanged for an id_token, access token and/or a refresh token. The client server then gets this authorization code and exchanges it for token(s) by sending the authorization code to the token endpoint. The client now gets the tokens and authenticates the user by validating the ID token.

### Implicit flow
This flow is used for browser based apps that don’t have a back end. This flow is called implicit flow because the authentication is implicit from a redirect when the user has successfully logged in. After the user has logged in the authorization server returns a redirect to the client containing the access token and/or id_token. Be aware that this flow should not use refresh tokens as these are can be stolen from the browser with huge security consequences.

### Hybrid flow
This flow contains a mix of the two above by requesting both a authorization code and tokens on first round trip. This flow enables the back end and front end to retrieve their own scoped tokens, such as a scope with refresh token for the back end and access tokens for the front end, but is not used very often. The flow is initiated by requesting the authorization endpoint with the following response_type parameter values:
Code id_token (returns authorization code and ID token)
Code token (returns authorization code and access token)
Code id_token token (returns authorization code, id_token and access token)


## The setup

We are gonna implement implicit flow in our app because we are using it with an Angular app, which should not keep secrets as it runs in the browser.

The basic flow is:


![Implicit flow](/images/openid-connect-part1/oic-implicit.png)

1. The user is trying to navigate to a authorized resource
2. The app is requesting the authorized resource without a valid access token
3. The Resource server returns an error 401
4. The 401 response makes the Angular app navigate to the Authorization endpoint on the Authorization server. This contains a response_code: id_token token for requesting both an id token and an access token.
5. The authorization server redirects to the login page
6. The user logs in and gives consent to accessing the authorized resources
7. Authorization endpoint returns redirect with tokens
8. The Angular app fetches tokens from the query params in the url and validates the id_token and access token according to: http://openid.net/specs/openid-connect-implicit-1_0.html#IDTokenValidation 
9. Angular app requests authorized resource with newly obtained access token.
10. The resource server is getting the json web key set (JWKS) from the authentication server (at {authUrl}/.well-known/openid-configuration/jwks) for verifying the access token signature with the public key, matching the private key that signed the JWT. Future request will just use an in memory cache of this JWKS. The claims of the access token are hereafter validated according to [this](https://connect2id.com/blog/how-to-validate-an-openid-connect-id-token).
11. The resource server now returns authorized resource to the client


In the following parts we are gonna create an OpenID connect implicit flow implementation with:

- Client server, hosting an Angular app
- Authorization server implemented as an ASP.NET core Identity server
- Resource server implemented as an ASP.NET core web api

Next part we will look into how to setup the authorization server with identity server 4.
