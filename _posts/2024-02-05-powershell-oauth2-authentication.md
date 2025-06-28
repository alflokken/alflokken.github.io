---
title: "PowerShell OAuth2 Authentication"
date: 2024-02-05T15:34:30-04:00
last_modified_at: 2025-06-28T15:34:30-04:00
categories: [PowerShell, Authentication]
tags:
  - Authentication
  - PowerShell
  - OAuth2
---
{% include analytics.html %}
This quick guide helps you get started with OAuth2 in PowerShell using the [PSAuthClient](https://github.com/alflokken/PSAuthClient) module. You can use any OAuth2 or OpenID Connect (OIDC) compliant provider, as long as it supports standard authorization and token endpoints— Spotify is just used here as an example.

This is a practical starting point for using OAuth2 in PowerShell, if you're looking to understand the underlying concepts in more detail, [auth0](https://auth0.com/intro-to-iam/what-is-oauth-2) offers some great resources.

> For a similar guide on using OpenID Connect (OIDC) with Microsoft Graph, see [this article](https://alflokken.github.io/posts/powershell-oidc-authentication/).
{: .prompt-tip }

![PSAuthClient in use]({{site.baseurl}}/assets/img/psauthclient.gif)
<sub>Interactive authentication, using PSAuthClient in PowerShell.</sub>

## Getting started
Either install the module from PSGallery `Install-Module PSAuthClient -Scope:CurrentUser` or [download](https://github.com/alflokken/PSAuthClient/releases) and unzip to ‘$home\Documents\WindowsPowerShell\Module’.

>The module is created and maintained by me, and the source code is available on [GitHub](https://github.com/alflokken/PSAuthClient). 
{: .prompt-info }

# Parameters
The parameters below are used (and modified) throughout the examples below, which use [Spotify](https://developer.spotify.com/documentation/web-api/tutorials/getting-started) as the OAuth 2.0 provider.

```powershell
$splat = @{  
  uri          = "https://accounts.spotify.com/authorize"  
  client_id    = "5eda97cf-2963–41e9-bea0-b6ba2bbf8f99"  
  scope        = "user-read-currently-playing user-modify-playback-state"  
  redirect_uri = "https://localhost"  
}
```
## **Authorization Code Flow with Proof Key for Code Exchange (PKCE)**

The Authorization Code grant type is used by confidential and public clients to exchange an authorization code for an access token, PKCE is used as an extension to prevent cross-site request forgery (CSRF) and authorization code injection attacks.

```powershell
# Request - Code Authorization (user authorization)
$code = Invoke-OAuth2AuthorizationEndpoint @splat  

# Response - Code
client_id     : 5eda97cf-2963–41e9-bea0-b6ba2bbf8f99  
code          : AQBrjtIWm7DHVD36lGZTf-5pnGc6jWs1q...  
code_verifier : 7Z2~0re4tiux4xb1twHSCW.wVkHukJKGx...  
redirect_uri  : https://localhost

# Request - Token Code Exchange (using the response from the previous step)
$token = Invoke-OAuth2TokenEndpoint -uri "https://accounts.spotify.com/api/token" @code  
  
# Response - Token Code Exchange (access token)
access_token    : BQDYX8DbcOOwIyKGQMW49dZzHCZQefO...  
token_type      : Bearer
expires_in      : 3600
refresh_token   : AQAwczh7CZ9LMjGNSxf3gLYgG6kmvis...  
scope           : user-modify-playback-state user...  
expiry_datetime : 05.02.2024 12:05:03
```

## **Refreshing tokens (Refresh Token Grant)**

A refresh_token is typically a long lived credential artifact that OAuth can be use to obtain a new access token without user interaction.

This makes it possible to exchange the refresh_token for a new access token when the access token has expired.

```powershell
# Request - Token Exchange
$splat.uri = "https://accounts.spotify.com/api/token" # update the existing uri value of our initial splat 
$token = Invoke-OAuth2TokenEndpoint @splat -refresh_token $token.refresh_token # use the refresh_token from the previous step
  
# Response - Token Exchange (new access token)
access_token    : BQCQjnqLaGXZ-sEchDq9A5kSVGMxlly1YH...  
token_type      : Bearer  
expires_in      : 3600  
refresh_token   : AQCWggiet87ydHb2FXnT7wFOeP1EoOQpBj...  
scope           : user-modify-playback-state user-re...  
expiry_datetime : 05.02.2024 12:24:22
```

## **Client Credentials Grant (secret)**

Often used when applications request access tokens to access their own resources (not on behalf of a user).

```powershell
# Request - Client Authentication
$splat.uri = "https://accounts.spotify.com/api/token" # update the existing uri value of our initial splat
$token = Invoke-OAuth2TokenEndpoint @splat -client_secret "very-secret123"  
  
# Response - Client Authentication
access_token    : BQDW7m7ebU5n2d8PU_HvrRr1t6mWWBttt-eX1k85YJv...  
token_type      : Bearer  
expires_in      : 3600  
scope           : user-modify-playback-state user-read-curren...  
expiry_datetime : 05.02.2024 12:30:57
```

## **Implicit Grant Grant (deprecated)**

Simplified OAuth Grant which returns the access_token without any extra authorization code exchange. This is not recommended due to the risk associated with returning tokens in a HTTP redirect (without any confirmation that it has been received by the client).

> You should only use this flow when other more secure flows are not viable, note that some servers prohibit this flow entirely.
{: .prompt-warning }

```powershell
# Request - Implicit Grant
$splat.uri = "https://accounts.spotify.com/authorize"  
$token = Invoke-OAuth2AuthorizationEndpoint @splat -response_type "token"  
  
# Response - Implicit Grant  
access_token    : BQBVEl2y3AymUtm69VWUF_OJxE7w8DZgtriS2O2WvyrnT…  
token_type      : Bearer  
expiry_datetime : 05.02.2024 12:28:43  
expires_in      : 3600
```