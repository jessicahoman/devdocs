---
layout: default
title: CSRF Protection
github_link: developers-guide/csrf-protection/index.md
shopware_version: 5.2.0
indexed: true
tags:
  - security
  - csrf
---

This article will focus on some security features of Shopware. First, a short introduction to the problem:

> Cross-Site Request Forgery (CSRF) is an attack that forces an end user to execute unwanted actions on a web application in which they're currently authenticated. CSRF attacks specifically target state-changing requests, not theft of data, since the attacker has no way to see the response to the forged request. With a little help of social engineering (such as sending a link via email or chat), an attacker may trick the users of a web application into executing actions of the attacker's choosing. If the victim is a normal user, a successful CSRF attack can force the user to perform state changing requests like transferring funds, changing their email address, and so forth. If the victim is an administrative account, CSRF can compromise the entire web application.

*Source: [Open Web Application Security Project](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))*

<div class="toc-list"></div>


## Receiving a token

When you open the backend, the first request made will be a `generate` request. This request will return a new token, bound to your session. Since this token is required for every other request, all requests have been queued until the `generate` request returns with a response. From that point on, all future requests will automatically be modified to make use of the `X-CSRF-Token` header. If you have, for some reason, decided to use your own request library, make sure to set the `X-CSRF-Token` header in your request. Otherwise every request will result in an exception.

 Once the token has been returned, you can get it by using the `Ext.CSRFService` service. The `Ext.CSRFService.getToken()` method will return the current token.

## Validating the token

We have introduced a new component called `CSRFTokenValidator`. This component subscribes to the `Enlight_Controller_Action_PreDispatch_Backend` event and therefore catches every single backend request, regardless of whether it's a `GET`, `POST` or `DELETE` request. The validator now checks if there is a `X-CSRF-Token` header set, which indicates that the application is aware of this protection mechanism. The token now gets validated against the token saved in your session.

If there is no token set or the token is invalid, you'll get an exception with the following error message:

> The provided X-CSRF-Token header is invalid. If you're sure that the request should be valid, the called controller action needs to be whitelisted using the CSRFWhitelistAware interface.

That means that you either have to modify your plugin code or whitelist some of your actions.

## Whitelist particular actions

Simple GET requests made with a browser don't send custom headers. Therefore we cannot validate those requests. For this case, plugin developers have to implement a new interface called `CSRFWhitelistAware`. This interface contains one method, which should return a list with names of the whitelisted actions.

For example, given you have a action called `downloadAsCsvAction()`, you then have to add `downloadAsCsv` to list of whitelisted actions right inside of the new `getWhitelistedCSRFActions()` method.

```php
<?php

use Shopware\Components\CSRFWhitelistAware;

class Shopware_Controllers_Backend_MyPlugin extends Shopware_Controllers_Backend_ExtJs implements CSRFWhitelistAware
{
    public function getWhitelistedCSRFActions()
    {
        return [
            'downloadAsCsv'
        ];
    }
}
```