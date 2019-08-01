+++
title =  "Response streaming between JAR-RS and Web-Components (Part 2)"
date = 2019-07-28T12:18:59+02:00
tags = ["web-components", "fetch", "chunk", "streaming"]
+++

In [part one]({{< relref "response-streaming-between-jaxrs-and-webcomponents-part1.md" >}}) we had a look at a JAX-RS endpoint that streams its content to the requesting client. Now I'd like to show how the `fetch`-API can be used to consume that streamed content in a web component.
<!--more-->
When talking about _streamed content_ here we're actually refering to [chunked transfere encoding][chunked] where the complete content of a response is spread accross several transmissions. <br/>
In order to consume our streaming endpoint we'll use the [fetch-API][fetch] - a more flexible and powerful approach to retrieve resources over the network than the former used [XMLHttpRequest][XMLHttpRequest].

__Notice:__ In order to use `fetch` in our web-component we have to rely on a feature called _Streaming response body_ which is marked as [experimental at the time of this writing][experimental] (although supported by all major browsers).

Let's start with a simple `index.html` that loads our `app.js`-module and uses the `sfm-stream-output`-component defined by it.

{{< highlight html>}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <sfm-stream-output></sfm-stream-output>
    <script type="module" src="app.js"></script>
</body>
</html>
{{< /highlight >}}

For the actual component let's start with the followin' skeleton:

{{< highlight javascript>}}
class SfmStreamOutput extends HTMLElement {

    constructor() {
        super();
        this.root = this.attachShadow({mode: 'open'});
        this.elements = [];
    }

    connectedCallback() {
        customElements
            .whenDefined('sfm-stream-output')
            .then(_ => this.render());
    }

    render() {
        this.root.innerHTML = this.elements
            .map(e => `<div>${e.index}. -- ${e.content}</div>`)
            .join('')
    }
}    
{{< /highlight >}}

Nothing fancy so far - the purpose of the component is to render a `<div>`-tag for each entry in the components `elements`-array. Right now this array is empty - so let's fill it with the response-data from our streaming endpoint using `fetch`.

A call to `fetch` will return a `Promise` which resolves to the `Response` of that request. Now, it's actually the [Body mixin][body] which provides a bunch of methods to read the responses content. Most of the time you'll see usages of methods like `response.text()` or `response.json()` which actually consume the [underlying stream][stream] for you and read it to completion. But you can also access the stream and consume it yourself by using `response.body`:

{{< highlight javascript>}}
fetch('/streamoutput/api/resource/stream')  // call the endpoint
    .then(response => response.body)        // get the stream 
    .then(body => { 
        const reader = body.getReader();    // get a reader 
        reader.read()
            .then(({done, value}) => console.log(done, value));
    });                 
{{< /highlight >}}

When executing the above code you'll notice several things:

1. your `console.log` will be called only once ... so apparently you're not consuming the complete response
2. the value is of type [Uint8Array][Uint8Array] ... so a binary representation you'll have to decode before it can be used

Let's fix the decoding first:

{{< highlight javascript>}}
fetch('/streamoutput/api/resource/stream')  // call the endpoint
    .then(response => response.body)        // get the stream 
    .then(body => { 
        const reader = body.getReader();    // get a reader 
        reader.read().then(({done, value}) => 
            console.log(done, new TextDecoder('utf-8').decode(value)));
    });                 
{{< /highlight >}}

We still got only one call to `console.log` - but this time we can actually see the content of our first chunk

{{< highlight javascript>}}
...
{"index":36,"content":"Some random text which actually can be generated in order to be really random"}
{"index":37,"content":"Some random text which a
{{< /highlight >}}

which reveals that it get's tricky to parse that chunk since it was split arbitrarily. 

But now we know at least everything we need in order to process the response.

1. make sure we call `read()` several times to consume all chunks of the stream (done in line <tt>17</tt> and <tt>28</tt>).
2. split each junk at our newline-delimiter to get parsable portions (so a valid JSON-object - done via the regex in line <tt>8</tt>)
3. buffer incomplete, trailing lines and prepend 'em to the next chunk (in order to yield a valid JSON again - line <tt>13</tt> 'til <tt>22</tt>).

{{< highlight javascript "linenos=table">}}
fetch('/streamoutput/api/resource/stream')
    .then(response => response.body)
    .then(body => {
        const reader = body.getReader();
        const process = async ({done, value}) => {
        if (done) {console.log('I\'m done with you ...');return;}

        let regexp = /\n|\r|\r\n/gm;
        let startIndex = 0;
        let chunk = new TextDecoder('utf-8').decode(value);
        for (;;) {
            let match = regexp.exec(chunk);
            if (!match) {
                if (done) {break;}

                let remainder = chunk.substr(startIndex);
                ({value, done} = await reader.read());

                chunk = remainder + (chunk ? new TextDecoder('utf-8').decode(value) : "");
                startIndex = regexp.lastIndex = 0;
                this.render();
                continue;
            }
            this.elements.push(JSON.parse(chunk.substring(startIndex, match.index)));
            startIndex = regexp.lastIndex;
        }
    };
    reader.read().then(process);
});
{{< /highlight >}}

When placed in the `connectedCallback`-method of our web-component it'll issue the request when it gets added to the DOM and it'll render each chunk of the response as soon as it was received.<br/>
You can find all sources in [this repository][repo] - clone it, compile and run it using the `buildAndRun.sh`-script (prerequisite is `maven` and `docker`) and open `http://localhost:8080/streamoutput/index.html` in Chrome where you should see a steadily growin UI.

<video width="590" height="622" controls preload="none" poster="/posts/streaming/stream-processing-using-fetch.png">
    <!-- MP4 must be first for iPad! -->
    <source src="/posts/streaming/stream-processing-using-fetch.mp4" type="video/mp4" /><!-- WebKit video    -->
    <!--source src="__VIDEO__.webm" type="video/webm" /--> <!-- Chrome / Newest versions of Firefox and Opera -->
    <!--source src="__VIDEO__.OGV" type="video/ogg" /--> <!-- Firefox / Opera -->
    <!-- fallback image. note the title field below, put the title of the video there -->
    <img src="/posts/streaming/stream-processing-using-fetch.png" width="590" height="622" title="No video playback capabilities, please download the video below" /-->
    </object>
</video>


[web-component]:https://developer.mozilla.org/en-US/docs/Web/Web_Components
[fetch]:https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
[stream]:https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Concepts
[chunked]:https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding
[XMLHttpRequest]:https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
[experimental]:https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API#Browser_compatibility
[body]:https://developer.mozilla.org/en-US/docs/Web/API/Body
[Uint8Array]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array
[repo]:https://github.com/schoeffm/jax-rs-streamingoutput
