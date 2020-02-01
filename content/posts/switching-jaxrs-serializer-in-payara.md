+++
title = "Switching JaxRS serializer in payara"
date = 2020-01-16T09:00:36+01:00
tags = ["payara", "jax-rs"]
+++

When working with [Payara][payara] I stumble every once in a while over JSON-serialization issues. Payara v4 uses [MOXy][moxy] as default provider while Payara v5 comes with [JsonB][jsonb]-support which apparently makes use of [Yasson][yasson].

Although, after moving from v4 to v5 I'm fine with the [JsonB][jsonb]-default it may still be useful to use [Jackson][jackson] as JSON provider instead of the default one (to be honest, many things just worked out-of-the-box with [Jackson][jackson] like reflective access to props (no extra getter necessary) or support for `Map`s etc.).

So, there are many useful hints out there where [this answer from Ondro Mihályi himself][stack-payara] is probably the most useful one. He mentions the `getProperties`-method which I perfer to use in my projects instead of the more explicitly shown descriptor-based approaches.

{{< highlight java>}}
@ApplicationPath("api")
public class BDQVApplication extends javax.ws.rs.core.Application {
    @Override
    public Map<String, Object> getProperties() {
        Map<String, Object> props = new HashMap<>();
        props.put("jersey.config.jsonFeature", "JacksonFeature");
        return props;
    }
    ...
}
{{< /highlight >}}

the equivalent can be achieved declaratively i.e. via your `web.xml`:
{{< highlight xml>}}
<context-param>
    <param-name>jersey.config.jsonFeature</param-name>
    <param-value>JacksonFeature</param-value>
</context-param>
{{< /highlight >}}

This solution works seemlessly in both verisons of Payara - v4 and v5.

[payara]:https://www.payara.fish
[stack-payara]:https://stackoverflow.com/questions/49793289/how-to-use-jackson-2-in-payara-5
[moxy]:https://www.eclipse.org/eclipselink/#moxy
[yasson]:https://eclipse-ee4j.github.io/yasson/
[jsonb]:http://json-b.net/
[jackson]:https://github.com/FasterXML/jackson
