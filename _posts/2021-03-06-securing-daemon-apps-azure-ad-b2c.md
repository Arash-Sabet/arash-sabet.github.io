---
layout: post
title:  "Securing daemon applications by Azure Ad B2C"
description: Server-side application authentication and obtaining tokens for Azure AD B2C tenants without a UI
tags: security authentication
tag: azure ad b2c
categories: Security, Azure AD B2C
mermaid: true
---
If you have stumbled upon this post, you have been likely exploring a solid way to secure your backend long-running processes to access a secured resource like a web API or an Azure Function. These applications can authenticate the application's identity (not a user's delegated identity) and by the OAuth 2.0 client credential flow.

## An example application

Now, let's take a look at an example application that consists of the following three components:

* A daemon app (console app) that produces some data
* A secured Web API that provides some inputs to the daemon application
* A secured HTTP-triggered Azure Function App that receives data from the daemon application to process them

The daemon app acquires a token from the [Microsoft Identity Platform](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-overview) as an application without user interaction, invokes the Web API to obtain the necessary inputs to produce data and submits the produced data to the HTTP-triggered function for further processing. The Web API and the HTTP-triggered function need to ensure that the incoming calls are legit and from the designated daemon app. That has to take place by examining the token present in the web requests' headers.

I am not trying to walk you through the setup steps or share a project in this post because such details are already in the public domain, e.g. in [this GitHub repo](https://github.com/Azure-Samples/active-directory-dotnetcore-daemon-v2). The repo covers an Azure AD tenant's setup steps, configuring a daemon application (a console app) and a Web API where the tokens are verified.

### Validating JWT tokens in Azure HTTP-triggered functions

There is only one area that I'd like to cover which is related to validating daemon apps' tokens in Azure HTTP-triggered Functions. Normally, a daemon application would send the obtained token in POST or GET requests' `Authorization` headers. The following C# code example illustrates how the token is added to the header in a daemon app:

```csharp
_httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", authToken);
```

Therefore the HTTP-Triggered function should look at that particular header, remove `Bearer` keyword and start validating the JWT token.

```csharp
var authHeader = AuthenticationHeaderValue.Parse(request.Headers["authorization"]);
```

The following code snippet illustrates how a JWT token is validated. There are two important points we need to consider to be able to validate the daemon app's token successfully: 

1- We need to ensure that the token's audience is the right one. Please see `ValidAudience` entry in the following code to realize how it's structured.

2- The issuer's entry conforms to the pattern shown in the example, i.e. `ValidIssuer`.

```csharp
 if (authHeader != null && 
 authHeader.Scheme.ToLower() == "bearer" &&
 !string.IsNullOrEmpty(authHeader.Parameter))
 {
    if (_validationParameters == null)
    { 
        var configManager = new ConfigurationManager<OpenIdConnectConfiguration>(
        $"{_daemonAuthenticationSettings.Instance}/{_daemonAuthenticationSettings.TenantId}/v2.0/.well-known/openid-configuration",
        new OpenIdConnectConfigurationRetriever());
        var openIdConnectConfiguration = await configManager.GetConfigurationAsync();
        _validationParameters = new TokenValidationParameters
        {
            IssuerSigningKeys = openIdConnectConfiguration.SigningKeys,
            ValidateAudience = true,
            ValidAudience = $"https://_daemonAuthenticationSettings.TenantName.onmicrosoft.com/{_daemonAuthenticationSettings.ClientId}",
            ValidateIssuer = true,
            ValidIssuer = $"https://sts.windows.net/{_daemonAuthenticationSettings.TenantId}/",
            ValidateLifetime = true,
            RequireExpirationTime = true
        };
    }

    var tokenHandler = new JwtSecurityTokenHandler();
    try
    {
        var result = tokenHandler.ValidateToken(authHeader.Parameter, _validationParameters, out var jwtToken);
        // If ValidateToken did not throw an exception, token is valid.
        return true;
    }
    catch (Exception exception)
    {
        onException(exception); // do something!
    }
 } 
```

### Why not using Microsoft Graph API tokens in securing daemon applications?

I have run into some posts online by developers who tried to use Microsoft Graph API to obtain tokens for their applications and secure them. The reality is that validating Microsoft Graph API-issued tokens will always fail because of the mismatching signature. The reason is simple, a Microsoft Graph API-issued token is meant for Azure resources protected by Graph API, not other applications. [This GitHub post](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/issues/609#issuecomment-524434987) covers further details on this topic really worth reading.