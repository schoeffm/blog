+++
title = "API versioning using vender specific media types in JAX-RS"
date = 2019-06-10T10:40:12+02:00
publishDate = 2019-06-10
tags = ["jax-rs", "jee", "jakartaee", "microprofile", "rest"]
+++

Content Negotiation in JAX-RS allows you to leverage the information in the client requests `Accept` header to map that request to a specific handler-method within your application. In combination with vendor-specific media types this approach can be used for [versioning of an API][idea].
<!--more-->

So, the idea is to introduce our own, custom media type - a so called [vendor-specific media types][spec] and use that to dispatch clients requests to the proper method. 
A vendor-specific media type starts with a `vnd.`-prefix for its subtype-name - the rest of the subtype is up to you. So the following are valid vendor-types:

{{< highlight bash>}}
application/vnd.my.comp+xml
application/vnd.schoeffm+json
{{< /highlight >}}

By appending the `+xml` or `+json`-suffix we give the caller a hint what representation it can actually expect in the respective response.

Another feature of media types is that they can bear parameters - normally used to express precedence - which [could be used for requesting a specific version][version-as-param]:

{{< highlight bash>}}
Accept: application/vnd.schoeffm+json;version=1.0
{{< /highlight >}}

Although this looks perfect for versioning it doesn't work in practice since the parameter is optional. So when used in declarative method-mapping via `@Produces` the media-type is still `application/vnd.schoeffm+json`(without version parameter). Thus, if you have several, parallel implementations for an endpoint just differing in the version-attribute of the `javax.ws.rs.Produces` annotation the server will throw an exception at startup since it cannot distinguish 'em.<br/>_<small>It'd still be possible to do this using i.e. `javax.ws.rs.core.Variant`, but this approach seemed just too cumbersome to be used.</small>_.

Therefore we've modelled the version-information into the actual subtype-name (see [this Github-repo][repo])
{{< highlight java>}}
@GET
@Produces("application/vnd.schoeffm-v1+json")
public Response customersOld() { return Response.ok("Version 1.0").build(); }

@GET
@Produces("application/vnd.schoeffm-v2+json")
public Response customersNew() { return Response.ok("Version 2.0").build(); }
{{< /highlight >}}

Which allows us to:

- have several versions of an API exposed at the same time 
- several media-types map to one method (allowing i.e. fallbacks to always server `latest`)
- stable URIs for resources (so the resource doesn't change with every new version)



[spec]:https://tools.ietf.org/html/rfc4288#section-3.2
[idea]:http://blog.steveklabnik.com/posts/2011-07-03-nobody-understands-rest-or-http
[version-as-param]:http://www.informit.com/articles/article.aspx?p=1566460
[repo]:https://github.com/schoeffm/jax-rs-conneg
