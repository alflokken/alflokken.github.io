---
title: "PowerShell OpenID Connect (OIDC) Authentication"
date: 2024-02-05T15:34:30-04:00
last_modified_at: 2025-06-28T12:39:38-04:00
categories: [PowerShell, Authentication]
tags:
  - Authentication
  - PowerShell
  - OIDC
  - Microsoft Graph
---

This quick guide helps you get started with OpenID Connect (OIDC) in PowerShell using the [PSAuthClient](https://github.com/alflokken/PSAuthClient) module. You can use any OpenID Connect (OIDC) compliant provider, as long as it supports standard authorization and token endpoints— Microsoft Graph is just used here as an example.

> For a similar guide on using OAuth2.0 see [this article](https://alflokken.github.io/posts/powershell-oauth2-authentication/).
{: .prompt-info }

![PSAuthClient in use]({{site.baseurl}}/assets/img/psauthclient.gif)
<sub>Interactive authentication, using PSAuthClient in PowerShell.</sub>

## Getting started
Either install the module from PSGallery `Install-Module PSAuthClient -Scope:CurrentUser` or [download](https://github.com/alflokken/PSAuthClient/releases) and unzip to ‘$home\Documents\WindowsPowerShell\Modules’.

>The module is created and maintained by me, and the source code is available on [GitHub](https://github.com/alflokken/PSAuthClient). 
{: .prompt-info }

## What is OpenID Connect?
OpenID Connect is an identity layer built on top of OAuth 2.0 that enables clients to request and receive ID tokens for user authentication, using the openid scope and has extended the accepted response_type values to include id_token.

> This is a practical starting point for OIDC in PowerShell, if you're looking to understand the underlying concepts in more detail, [okta](https://developer.okta.com/docs/concepts/oauth-openid/) has some great resources.
{: .prompt-tip }

# Parameters
The parameters below are used (and modified) throughout the examples below, which use [Microsoft Graph](https://learn.microsoft.com/en-us/graph/auth/auth-concepts) as the OIDC/OAuth2.0 provider.

```powershell
$authorization_endpoint = "https://login.microsoftonline.com/example.org/oauth2/v2.0/authorize"  
$token_endpoint         = "https://login.microsoftonline.com/example.org/oauth2/v2.0/token"  
  
$splat = @{  
    client_id        = "5eda97cf-2963-41e9-bea0-b6ba2bbf8f99"  
    scope            = "user.read openid offline_access"  
    redirect_uri     = "https://login.microsoftonline.com/common/oauth2/nativeclient"  
    customParameters = @{
        # Indicates the type of user interaction required for the authorization to Microsoft Graph. 
        # ref. https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc  
        prompt = "none"
    }  
}
```

# Authorization Code Grants

The Authorization Code grant type is used by confidential and public clients to exchange an authorization code for an access token.

## Authorization Code Grant with Proof Key for Code Exchange (PKCE)

PKCE is used as an extension to prevent cross-site request forgery (CSRF) and authorization code injection attacks.


```powershell
# Request - Code Authorization (user authorization)
$code = Invoke-OAuth2AuthorizationEndpoint -uri $authorization_endpoint @splat

# Response - Code
nonce        : UYhqAG~GLvZqGj4hnlTkYFJY9LVcS9TWi...  
redirect_uri : https://login.microsoftonline.co/...  
client_id    : 5eda97cf-2963-41e9-bea0-b6ba2bbf8f99  
code         : 0.AUcAjvFfm8BTokWLwpwMj2CyxiGBP5h...  

# Request - Token Code Exchange (using the response from the previous step)
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @code  

# Response - Token Code Exchange (access token)
token_type      : Bearer  
scope           : User.Read profile openid email  
expires_in      : 3848  
ext_expires_in  : 3848  
access_token    : eyJ0eXAiOiJKV1QiLCJub62jZSI6Im...  
refresh_token   : 0.AUcAjvFfm8BTokWLwpwMkJCyxiGB...  
id_token        : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUz...  
expiry_datetime : 31.01.2024 14:05:18
```

## Authorization Code Grant (without PKCE)

Included just as an example, some services might not have PKCE implemented even though they should.


```powershell
# Request - Code Authorization  
$code = Invoke-OAuth2AuthorizationEndpoint -uri $authorization_endpoint @splat -usePkce:$false
  
# Response - Code 
nonce        : UYhqAG~GLvZqGj4hnlTkYFJY9LVcS9TrW...  
redirect_uri : https://login.microsoftonline.com...  
client_id    : 5eda97cf-2963-41e9-bea0-b6ba2bbf8f99  
code         : 0.AUcAjvFfm8BTokWLwpwMj2CyxiGBP5h...  

# Request - Token Code Exchange  
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @code  

# Response - Token Code Exchange  
token_type      : Bearer  
scope           : User.Read profile openid email  
expires_in      : 3848  
ext_expires_in  : 3848  
access_token    : eyJ0eXAiOiJKV1QiLCJub62jZSI6Ih...  
refresh_token   : 0.AUcAjvFfm8BTokWLwpwMkJCyxiGB...  
id_token        : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUz...  
expiry_datetime : 31.01.2024 14:05:18
```

## Authorization Code Grant with Client Authentication (secret)

By using client authentication in the code authorization grant, this further reduces the risk of an attacker intercepting the authorization code.


```powershell
# Request - Code Authorization
$splat.redirect_uri = "https://localhost/web" # update the existing redirect_uri value of our initial splat
$code = Invoke-OAuth2AuthorizationEndpoint -uri $authorization_endpoint @splat

client_id     : 5eda97cf-2963-41e9-bea0-b6ba2bbf8f99  
code_verifier : jWe-ecfnqZ.weAxbb-qHiZ3oe7LZ-tEyW...  
nonce         : HRBD6BuH9PQM2_Kmuqj6KTranVVcuL80f...  
redirect_uri  : https://localhost/web  
code          : 0.AUcAjvFfm8BTokWLwpwMj2CyxiGBP5h...  
  
# Request - Token Code Exchange (with client authentication)
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @code -client_secret "verysecret123"  
  
# Response - Token Code Exchange
token_type      : Bearer  
scope           : User.Read profile openid email  
expires_in      : 4069  
ext_expires_in  : 4069  
access_token    : eyJ0eXAiOiJKG1QqLCJub25jZSI5Il...  
refresh_token   : 0.AUcAjvFfmC9TokWLwpwMj2CyxiGB...  
id_token        : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUz...  
expiry_datetime : 31.01.2024 14:28:58
```

## Refresh Token Grant

A refresh_token is typically a long lived credential artifact that OAuth can be used to obtain a new access token without user interaction. This makes it possible to exchange the refresh_token for a new access token when the access token has expired.

```powershell
# Request - Token Exchange
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint -refresh_token $token.refresh_token -client_id $splat.client_id -scope $splat.scope -nonce $code.nonce  

# Response - Token Exchange
token_type      : Bearer  
scope           : User.Read profile openid email  
expires_in      : 3951  
ext_expires_in  : 3951  
access_token    : eyJ0eXAiOiJKR1QiLCJsf52jZSI6Ij...  
refresh_token   : 0.AUcAjvFfm1BTokWLkjrMj3CyxiGB...  
id_token        : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUz...  
expiry_datetime : 31.01.2024 14:16:56  
```

## Device Code Grant

Intended for browserless or input-constrained devices. Use previously obtained device code to exchange for an access token.


```powershell
# Request - Device Code
$deviceCode = Invoke-OAuth2DeviceAuthorizationEndpoint -uri "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/devicecode" -client_id $splat.client_id -scope $splat.scope  
  
user_code        : L8EFTXRY3  
device_code      : LAQABAAEAAAAmoFfGtYxvRrNriQdPKIZ...  
verification_uri : https://microsoft.com/devicelogin  
expires_in       : 900  
interval         : 5  
message          : To sign in, use a web browser and...  
  
# User interaction required
Invoke-WebView2 -uri "https://microsoft.com/devicelogin" -UrlCloseConditionRegex "//appverify$" -title "Device Code Flow" | Out-Null  
  
# After user-interaction has been completed.  
# Request - Device Code Token Exchange
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint -device_code $deviceCode.device_code -client_id $splat.client_id  

# Response - Device Code Token Exchange
token_type      : Bearer  
scope           : User.Read profile openid email  
expires_in      : 5320  
ext_expires_in  : 5320  
access_token    : eyJ0eXAiOiJKV1QiKH6Gb25jZSI5IjlzanppVWtNSlkR4WxfWjBRWFJRZUl4TEdyaDBad05TQ01sQ1...  
refresh_token   : 0.AUcAjvFfm8BlORWLwpwMj2CyxiGBP5hz2ZpErkU62chlhOUNAVw.AgABAAEAAAAmoFfGtYxvRrlK...  
id_token        : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6ImtXYmthYTZxczh3c1RuQndpaU5ZT2hIYm...  
expiry_datetime : 31.01.2024 15:07:19
```

## Client Credentials - Secret

Often used when applications request access tokens to access their own resources (not on behalf of a user). Based on the client authentication methods supported by the authorization server, we can do different kinds of client authentication.

**Client Credentials Grant (client_secret_basic)**

HTTP Basic Authentication (BASE64 Encoded clientId:secret in authorization header).

```powershell
# Request - Client Authentication
$splat.Remove("customParameters")
$splat.scope = ".default"  
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @splat -client_secret "secret" -client_auth_method client_secret_basic  

# Response - Client Authentication
token_type      : Bearer  
expires_in      : 3599  
ext_expires_in  : 3599  
access_token    : eyJ0eXAiOiJKV1DiLCJub25jZSI3IjUt...  
expiry_datetime : 31.01.2024 14:14:06
```

**Client Credentials Grant (client_secret_post)**

Send the client_id and client_secret as request parameters in the token request.

```powershell
# Request - Client Authentication
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @splat -client_secret (Invoke-Cache -keyName "PSC_Test-ClientSecret" -asSecureString)  
  
# Response - Client Authentication
token_type      : Bearer  
expires_in      : 3599  
ext_expires_in  : 3599  
access_token    : eyJ0eXAiOiJKV1QiGCJub25jZSI3I...  
expiry_datetime : 31.01.2024 14:16:10
```

**Client Credentials Grant (client_secret_jwt)**

An indirect way to prove that the client has the secret, the secret is not sent to the authorization server. Rather a JWT Assertion is crafted and a HMAC (hash-based message authentication code) is calculated using the client_secret and used as the assertion ‘signature’.

Microsoft Graph DOES NOT support client_secret_jwt. But if they did, this is how you would do it with PSAuthClient.

```powershell
# Request - Client Authentication
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @splat -client_secret $client_secret -client_auth_method "client_secret_jwt"  

# Response - Client Authentication
error             : invalid_client   
error_description : AADSTS5002723: Invalid JWT token. No certi...
```

## Client Credentials - Certificate

When using a certificate, the client can prove its identity by signing a JWT Assertion with the private key of the certificate. This is more secure than using a client_secret, as it does not require the client to store a secret in plaintext.

**Client Credentials Grant certificate (private_key_jwt)**

Uses a JWT Assertion (like client_secret_jwt), but the signature of the client assertion is created by using a client certificate. (asymmetric key).


```powershell
# Request - Client Authentication
$token = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @splat -client_certificate "Cert:\CurrentUser\My\8ade399dddc5973e04e34ac19fe8f8759ba059b8"  
  
# Response - Client Authentication
token_type      : Bearer  
expires_in      : 3599  
ext_expires_in  : 3599  
access_token    : eyJ0eXAiOiJKV1QiLCJub21jZSI2InpBUjQ6U...  
expiry_datetime : 31.01.2024 14:20:03
```

## Implicit Grant
Simplified Grant which returns the access_token without any extra authorization code exchange.

> Not recommended due to the risk associated with returning tokens in a HTTP redirect (without any confirmation that it has been received by the client). (Note: this does not apply if using form_post response mode).
{: .prompt-warning }

**Implicit Grant (OAuth2)**

```powershell
# Request - Implicit Grant
$splat.redirect_uri = "https://localhost/spa"  
$splat.scope = "User.Read"  
$token = Invoke-OAuth2AuthorizationEndpoint -uri $authorization_endpoint @splat -response_type "token" -usePkce:$false  
  
# Response - Implicit Grant
expires_in      : 4371  
expiry_datetime : 31.01.2024 14:39:19  
scope           : User.Read profile openid email  
session_state   : 5c044a56-543e-4bcc-a94f-d411ddec5a87  
access_token    : eyJ0eXAiOiJKV1QiLCJkj76jZSI6InlaZ...  
token_type      : Bearer
```

**Implicit Grant (OIDC)**

```powershell
# Request - Implicit Grant
$token = Invoke-OAuth2AuthorizationEndpoint -uri $authorization_endpoint @splat -response_type "token id_token" -usePkce:$false  

# Response - Implicit Grant
nonce           : NtKwrnSuV7xQQiya.jNXF940RQkS0OMlT...  
expires_in      : 4949  
id_token        : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1N...  
expiry_datetime : 31.01.2024 14:46:35  
scope           : User.Read profile openid email  
session_state   : 5c044a56-543e-4bcc-a94f-d411ddec5a87  
access_token    : eyJ0eXAiOiJKV1QiLCJub51jZSI6Ik2sa...  
token_type      : Bearer
```

**Hybrid Grant with Form_Post**

This grant uses POST as opposed to placing tokens in URL fragments which can expose tokens to browser history attacks, redirect headers etc.

```powershell
# Request - Implicit Grant
$splat.redirect_uri = "http://localhost:5001/"  
$token = Invoke-OAuth2AuthorizationEndpoint -uri $authorization_endpoint @splat -response_type "code id_token" -response_mode "form_post"  

# Response - Implicit Grant (Form_Post)
nonce           : iOJ6n7jBlYAL_TrYlFjfKwOsPklX1-4iR  
code_verifier   : j1v4ZEjF4AE.lMfsQ36UzF6OoBp.zwuJ7...  
id_token        : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1N...  
client_id       : 5eda97cf-2963-41e9-bea0-b6ba2bbf8f99  
session_state   : 5c044a56-543e-4bcc-a94f-d411ddec5a87  
redirect_uri    : http://localhost:5001/  
code            : 0.AUcAjvFfm8BTokWLwpwMj2CyxiGBP5Z...  
  
# Request - Code Exchange
$token.Remove("id_token"); $token.Remove("session_state")  
$tokens = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @token  

# Response - Code Exchange
token_type      : Bearer  
scope           : User.Read profile openid email  
expires_in      : 4840  
ext_expires_in  : 4840  
access_token    : eyJ0eXAiOiJKV1QiLCJub55jZSI6IlR...  
refresh_token   : 0.AUcAjvFfm8BTokSLwpwMj2CyxiGBP...  
id_token        : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI...  
expiry_datetime : 31.01.2024 14:54:54
```
**OpenID Connect Hybrid Grant**

Combines capabilities of the implicit grant and authorization code grant, to allow the client to receive both ID Token and Authorization Code from the authorization_endpoint.

```powershell
# Request - Hybrid Grant
$splat.scope = "user.read openid offline_access"
$splat.redirect_uri = "http://localhost"  
$splat.usePkce = $true  
$token = Invoke-OAuth2AuthorizationEndpoint -uri $authorization_endpoint @splat -response_type "code id_token"  
  
# Response - Hybrid Grant
nonce         : 7B61P-.ST87WdKZ9TPF~1a5sMkPs.atxj...  
code_verifier : w6Fvr5LTkex0k.aRJhL9rZeEDNSO5sdc8...  
id_token      : eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1N...  
client_id     : 5eda97cf-2963-41e9-bea0-b6ba2bbf8f99  
session_state : 5c044a56-543e-4bcc-a94f-d411ddec5a87  
redirect_uri  : http://localhost  
code          : 0.AUcAjvFfm8BTokWLwpwMj2CyxiGBP5h...  

# Request - Code Exchange
$token.Remove("id_token"); $token.Remove("session_state")  
$tokens = Invoke-OAuth2TokenEndpoint -uri $token_endpoint @token  

# Response - Code Exchange
nonce         : da1EE3-RRVJO.fFeCEw2TvG7hK46AWFWH...  
code_verifier : ~4fYq2QcXlSIZN_vZ7pnKsO5VZ0Pq39hs...  
client_id     : 5eda97cf-2963-41e9-bea0-b6ba2bbf8f99  
redirect_uri  : http://localhost  
code          : 0.AUcAjvFfm8BTokWLwpwMj2CyxiGBP5h...
```