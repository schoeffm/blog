+++
title =  "Response streaming between JAR-RS and Web-Components (Part 1)"
date = 2019-07-27T11:18:59+02:00
publishDate = 2019-07-25
tags = ["jax-rs", "jee", "jakartaee", "microprofile", "streaming"]
+++

When dealing with JAX-RS resources are normally assembled completely in memory before being put onto the wire for transmission. If the resulting response fits into one single chunk the transmission is accomplished in one go - if the content is bigger than 16kB it will be split into several chunks of that size. You can see the difference in the response headers where a one-go-transmission contains the `Content-Length`-header (where the server announces the size of the content to be transmitted) while a chunked transmission lacks the `Content-Length` header but contains a `Transfer-Encoding: chunked` header. 
<!--more-->
The crux with this approach is that you'll always have to __load all the data comprising the response before a single bit is transfered to the client__. That buffering can become a performance issue when dealing with large amounts of data or when loading of that data is sluggish.

To ship chunks as soon as they're filled and before even knowing the final bit of the response you can make use of JAX-RS' `StreamingOutput` interface. Put an implementation of that interface as the entity in a `Response` and JAX-RS will send each chunk as soon as it is filled. <br/>
Here's an example (simulating a large resource which also takes long to load):

{{< highlight java>}}
@GET
@Path("stream")
@Produces(MediaType.TEXT_PLAIN)
public Response streamMe() {
    StreamingOutput stream = output -> {
        Writer writer = new BufferedWriter(new OutputStreamWriter(output));
        for (int i = 0; i < 1000; i++) {
            writer.write(Json.createObjectBuilder()
                    .add("index", i)
                    .add("content", "Some random text which actually can be generated in order to be really random")
                    .build().toString() + "\n");
            pause(10L); // pause Thread for 10ms
        }
        writer.flush();
    };

    return Response.ok(stream).build();
}
{{< /highlight >}}

The `StreamingOutput`-implementation is set as the `Response` entity. Notice, that the returned content is announced as `text/plain` since it's just a _newline_ separated list of ASCII text (in part 2 when looking at a client implementation we'll see why this is necessary).


When hitting this endpoint in the browser it'll show the followin' waterfall diagram:


{{< figure src="/posts/streaming/waterfall_when_response_is_streamed.png" alt="Streamed Response">}}

Comparing this result to the behavior of a normal JAX-RS endpoint which doesn't use `StreamingOutput`

{{< figure src="/posts/streaming/waterfall_when_response_is_not_streamed.png" alt="Normal Response">}}

you can see that the overall load-time didn't change. __But the _time-to-first-byte (TTFB)_ was significantly reduced.__ In the streaming-version of the endpoint the client could start parsing/rendering the (first part of the) output after 2.75 seconds while without streaming it has to wait the full 11.40 seconds.

In both these examples the transmission was chunked - but only the first one leveraged the chunks to provide a better [perceived performance][perceived-performance].

In the 2nd part of this post I'll show a (simple) web-component which uses the [fetch-API][fetch] to load the streamed content - the complete code for these examples can be found in [this repository][repo].


[chunked-transfere]:https://en.wikipedia.org/wiki/Chunked_transfer_encoding
[example]:http://solutionhacker.com/rest-use-streamingout-return-big-content-chunk/
[perceived-performance]:https://www.keycdn.com/blog/perceived-performance
[repo]:https://github.com/schoeffm/jax-rs-streamingoutput
[fetch]:https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
