+++
title = "Cors by Example"
date = 2019-08-30T15:33:13+02:00
draft = true
tags = ["cors", "api", "fetch", "web-components"]
+++

More often than not CORS mean little more to developers than just getting rid of the infamous browser error 
![CORS error](/posts/cors-error.png)
in order to continue cranking out features as soon as possible.

Hence, to shed some light upon the topic, I'd like to give some working examples of CORS setups and show why they work. The more detailed explanation I relinquish to the excellent [MDN documentation][cors] if you'd like to learn more about it.

## What is it

For security reasons a browser denies a webpage loaded from __origin A__ (i.e. `http://dogpics.com`) to embed resources which have to be [fetched][fetch] from a different __origin B__ (i.e. `http://catpics.com`). That is in essence what is normally refered to as [same-origin-policy][same-origin].

To bypass that restriction (but still prevent the inadvertent usage of resources) __origin A__ and __origin B__ can agree on a deliberate exchange of resources. They do this by defining a [Cross-Origin Resource Sharing][cors] strategy (or [CORS][cors] for short). When done properly the scenario described above becomes possible again and the webpage of `http://dogpics.com` can also embed resources from `http://catpics.com`.

## When do I need it

Well, whenever your frontend JavaScript tries to [fetch][fetch] resources from different [origins][origin] according to the [same-origin-policy][same-origin]. In other words, it applies if __origin B__ differs in either the _protocol_, the _domain_ or the _port_. So for our __origin A__ `http://dogpics.com` this means:

origin B | Same Origin | Why
--------|------|------
`https://dogpics.com` | nope | _protocol_ differs
`http://dogpics.com/cat.png` | yes | only the path is different - the origin is the same
`http://api.dogpics.com` | nope | _domain_ is different (yes, this includes sub-domains)
`http://dogpics.com:8443` | nope | _port_ is different

## CORS headers and their effect

Basically, it's the __origin B__ server which determines who is able to embed its resources by using a bunch of [HTTP-headers][headers]. Nevertheless, for some use-cases the requesting party (here the frontend JavaScript of __origin A__) also has to be adjusted. Also depending on the use-case is the choice of [CORS-headers][headers] to be applied.<br/>
Here's a short recap of the most important CORS-headers (again, details can be found [here][headers]) by which __origin B__ can control who else can fetch its resources:

- `Access-Control-Allow-Origin`: can only be set to one specific origin which is then allowed to fetch from this server. It can also be set to wildcard `'*'` but this value is __mutually exclusive__ to `"include"` credentials mode (which makes it useless when dealing with auth-cookies/headers and is most probably only relevant for local development).
- `Access-Control-Allow-Credentials`: _When a request's credentials mode (`Request.credentials`) is `"include"`, browsers will only expose the response to frontend JavaScript code if the `Access-Control-Allow-Credentials` value is `true` ([see here for more details][credentials])_
- `Access-Control-Allow-Methods`: all allowed methods which can be used in the actual fetch call 
- `Access-Control-Allow-Headers`: _Used in response to a preflight request to indicate which HTTP headers can be used when making the actual request._ Also allows wildcard `'*'` but again doesn't work with auth-cookies/header etc. 

## Examples of CORS requests

### Example 1 (without auth):

__Situation:__

We'd like to fetch public (no authentication is necessary) JSON-content using a GET-request. Since we request `Content-Type: application/json` this will cause a preflight request before the actual GET-request is issued (find [details here][preflight] about the difference between simple requests and preflight requests).

__Request:__

{{< highlight javascript>}}
const config = { "headers": {"Content-Type": "application/json"} };
fetch(url, config).then(response => response.json());
{{< /highlight >}}

__Header-Setup:__

{{< highlight bash>}}
Access-Control-Allow-Origin: '*'
Access-Control-Allow-Headers: 'content-type'
Access-Control-Allow-Methods: 'GET'
{{< /highlight >}}

__Result:__

- Since we don't have to deal with authentication whatsoever we can use the `'*'`-value for `Access-Control-Allow-Origin` (_Notice: the fact that it works doesn't mean that it's a good idea to allow access for the whole world_).<br/>
- Since we want to use HTTP-GET we have to allow that method using `Access-Control-Allow-Methods: 'GET'`
- And finally, since we want the server to return a JSON-representation of the resource we have to allow the `Content-Type` header to be used.

### Example 2 (with Cookies):

__Situation:__

For whatever reason the server of __origin B__ needs its cookies to be included in the request (i.e. session-cookie).

__Request:__

{{< highlight javascript>}}
const config = { 
    "credentials": "include", // makes sure that cookies will be sent 
    "headers": {"Content-Type": "application/json"} 
};
fetch(url, config).then(response => response.json());
{{< /highlight >}}

__Header-Setup:__

{{< highlight bash>}}
Access-Control-Allow-Origin: 'http://localhost:8081'
Access-Control-Allow-Credentials: 'true'
Access-Control-Allow-Headers: 'content-type'
Access-Control-Allow-Methods: 'GET'
{{< /highlight >}}

__Result:__

The usage of cookies has a bunch of implications for the setup to work:

- first, the client has to use `include` credentials mode to make sure the browser includes the __origin B__ cookies when issuing the GET-request.
- consequently, __origin B__ has to set the CORS-header `Access-Control-Allow-Credentials` to `true` - if that is not set (or set to `false`), it won't work (since the client explicitly requested it).
- when using `Access-Control-Allow-Credentials:true` the wildcard `'*'` for `Access-Control-Allow-Origin` is not applicable anymore - hence we have to be explicit about who actually is __origin A__.

### Example 3 (with Auth-Tokens):

__Situation:__

When your authentication setup doesn't use cookies but auth-headers you strangly don't have to use the `credentials: include` mode (although you can of course).

__Request:__

{{< highlight javascript>}}
const config = { 
    "headers": {
        "Content-Type": "application/json", 
        "Authorization": "Basic SXQncyBub3Qgd29yayB0aGUgZWZmb3J0IGR1ZGU=" 
    } 
};
fetch(url, config).then(response => response.json());
{{< /highlight >}}

__Header-Setup:__

{{< highlight bash>}}
Access-Control-Allow-Origin: '*'
Access-Control-Allow-Credentials: 'false'
Access-Control-Allow-Headers: 'content-type, authorization'
Access-Control-Allow-Methods: 'GET'
{{< /highlight >}}

__Result:__

This is basically the same setup as in __Example 1__ except that we explicitly allow the `Authorization` header as well<br/>
_Notice: In those cases (with or without credentials-mode) make sure that you use a TLS-connection._

## Test it out yourself

When you'd like to test this yourself you can over to [this repository][repo] in order to setup a simple 2-server constellation where one origin (`http://localhost:8081`) is including a resource of the other origin (`http://localhost:8080`). Using the UI you can tweak the server-headers and the config for the fetch-request.

![CORS UI](/posts/cors-ui.png)

In the server-logs you can inspect what headers and cookies are transmitted by the CORS-request (and how OPTIONS- a.k.a. preflight- and GET-request differ). 

[fetch]:https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch
[XHR]:https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
[cors]:https://developer.mozilla.org/en-US/docs/Glossary/CORS
[credentials]:https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials
[same-origin]:https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy
[origin]:https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#Definition_of_an_origin
[repo]:https://github.com/schoeffm/cors-with-fetch
[headers]:https://developer.mozilla.org/de/docs/Web/HTTP/CORS#The_HTTP_response_headers
[preflight]:https://developer.mozilla.org/de/docs/Web/HTTP/CORS#Preflighted_requests
