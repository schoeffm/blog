+++
title = "Network Debugging of Containerized Java Apps"
date = 2019-09-19T16:27:21+02:00
tags = ["mitmproxy", "proxy", "cli", "devops", "network", "tools" ]
+++

The other day, I had the need to look closer into the network communication of a (dockerized) JEE application. My usual [tcpdump approach]({{< relref "tcpdump-common-commands.md" >}}) wasn't applicable though because the traffic was TLS-encrypted.<br/>
Finally, I ended up with a setup where the app used a [man-in-the-middle proxy][mitm] for all its communication - that proxy transparently re-encrypted the traffic which gave me the chance to inspect every single message exchanged. 
<!--more-->
``` 
┌──────────┐ TLS #1  ┌────────────┐ TLS #2  ┌────────────┐
│  Payara  ├────────>│ MITM Proxy ├────────>│ WebService │
└──────────┘         └────────────┘         └────────────┘
runs in docker       runs on localhost      runs anywhere
```
To use this you'll have to prepare a few things though:

### 1. install mitm

If not already done install [mitmproxy][mitm] - a free and open source proxy available for all major platforms. After installed locally on our dev-machine, when starting the proxy we instruct it to listen for connections on port `1080`:

{{< highlight bash >}}
mitmproxy --listen-port 1080
{{< /highlight >}}

### 2. trust proxy certificates

As you can see in the setup depicted above from a JEE app perspective the trusted communication partner is not the web-service anymore but mitmproxy (_which terminates the apps TLS connection now_). In order to allow that, without the JEE app complaining, you'll have to convince the JVM to trust the certificate used by [mitmproxy][mitm].<br/>
Luckily, during installation [mitm][mitm-certs] will generate and place its certificates in a hidden folder in your home directory `$HOME/.mitmproxy`.

To trust it you'll have to import this certificate into the JVMs cert-store. Since we use a [dockerized payara][payara-image] we just create a custom `Dockerfile` and add that step to it:

{{< highlight Dockerfile >}}
FROM payara/server-full:5.192

# Import the SSL certificates of mitm
USER root
COPY $HOME/.mitmproxy/mitmproxy-ca-cert.cer .
RUN keytool -importcert -file ./mitmproxy-ca-cert.cer -keystore $PAYARA_DIR/glassfish/domains/production/config/cacerts.jks -alias MITM -storepass changeit -noprompt
USER payara
...
{{< /highlight >}}

### 3. make use of the proxy

Right now, the containerized app isn't aware of a proxy running on `localhost:1080`. To make use of it you'll have to set a few system-properties for the application to pick up - and for payara you'll have to use the `asadmin`-tool for that job.<br/>Let's add that again to our `Dockerfile` 
{{< highlight Dockerfile >}}
FROM payara/server-full:5.192
...
RUN $ASADMIN start-domain && \
    $ASADMIN create-system-properties http.proxyHost=host.docker.internal && \
    $ASADMIN create-system-properties http.proxyPort=1080 && \
    $ASADMIN create-system-properties https.proxyHost=host.docker.internal && \
    $ASADMIN create-system-properties https.proxyPort=1080 && \
    $ASADMIN stop-domain
{{< /highlight >}}
Notice the value for the `proxyHost`: `host.docker.internal`. <br/>
As of Docker(-for-mac|-for-windows) 18.03 this will allow connections from within a running container to the host machine. You can find further details in this [stackoverflow post][docker-localhost].

### 4. start debugging

And that's it ... when starting our payara container and interacting with the web-service we should see respective entries in the proxies UI.<br/>
For later analysis you can even stream the requests to a file while looking at 'em:

{{< highlight bash >}}
mitmproxy --listen-port 1080 -w mitm.log
{{< /highlight >}}

Which can then be loaded later on for further analysis

{{< highlight bash >}}
mitmproxy -r mitm.log
{{< /highlight >}}
    
[mitm]:https://mitmproxy.org
[mitm-certs]:https://docs.mitmproxy.org/stable/concepts-certificates/#ca-and-cert-files
[payara-image]:https://hub.docker.com/r/payara/server-full
[docker-localhost]:https://stackoverflow.com/questions/24319662/from-inside-of-a-docker-container-how-do-i-connect-to-the-localhost-of-the-mach/24326540#24326540