+++
title = "Troubleshooting Containerized Java Applications"
date = 2020-05-02T13:18:06+02:00
draft = true
tags = ["java", "jmx", "flightrecorder", "missioncontrol", "zmc", "visualvm", "docker", "payara", "performance", "analysis" ]
+++

When developing JEE applications these days we not only deploy 'em as containerized apps but (at least in my teams) also use containers for local development. 

One issue I was confronted with the other day was a performance bug with one of our apps. Back in the days I'd have just fired up [VisualVM][visualvm] to connect to the local process causing trouble - but now, the process to attach to doesn't run locally anymore.
<!--more-->
## JMX pre-requisite
The solution is still pretty simple - just treat the containerized process as an external app (i.e. like an app that was deployed on a remote server) and connect via JMX to it.
For that to work you'll have to add the followin' JVM-options to your containerized java-process:

{{< highlight bash>}}
-Dcom.sun.management.jmxremote 
-Dcom.sun.management.jmxremote.port=9090
-Dcom.sun.management.jmxremote.rmi.port=9090
-Djava.rmi.server.hostname=<CONTAINER_IP>
# only disable transport-layer-encryption and authentation
# for a local, non-productive setup
-Dcom.sun.management.jmxremote.ssl=false 
-Dcom.sun.management.jmxremote.authenticate=false 
{{< /highlight >}}

The `CONTAINER_IP` has to be the external IP-address assigned to the container. So for your local docker environment this would be the hosts address (i.e. `192.168.178.33`).

Setting this up in a [payara/server-full][payara-full]-derived image you could either set the `$JVM_ARGS` accordingly or use `asadmin`-commands via `$POSTBOOT_COMMANDS` like in the followin' snippet:

{{< highlight bash>}}
asadmin create-jvm-options '-Dcom.sun.management.jmxremote.port=9090'
asadmin create-jvm-options '-Dcom.sun.management.jmxremote.rmi.port=9090'
asadmin create-jvm-options '-Djava.rmi.server.hostname=192.168.178.33'
asadmin create-jvm-options '-Dcom.sun.management.jmxremote.ssl=false'
asadmin create-jvm-options '-Dcom.sun.management.jmxremote.authenticate=false'
{{< /highlight >}}

## Gettings started with tooling

To test the setup just start your container, add a port-mapping for the JMX-port 
{{< highlight bash>}}
docker run -d -p 8080:8080 -p 9090:9090 schoeffm/payara-app
{{< /highlight >}}

and use i.e. `jconsole` (which is most probably still part of your JDK-installation) to connect to the respective JMX-socket on `localhost:9090` (_confirm any security warnings_).

{{< figure width="60%" src="/posts/troubleshooting-containerized-java-applications/jconsole.png" alt="jconsole">}} 

## Using VisualVM

`jconsole` is nice, but the aforementioned [VisualVM][visualvm] is able to give us deeper insights. 

Depending on what JDK you're using VisualVM is not necessarily part of your installation (except you're using [GraalVM][graalvm]). Since I use [sdkman][sdkman] to manage parallel JDK-installations I incidentally already had a GraalVM installation which included VisualVM. 

If that is not the case you can [download VisualVM][visualvm] as a separate tool or use your package manager (like `brew cask install visualvm`).

* **VisualVM of GraalVM**: here it was enough to just start the app from within the `bin`-directory: `<GRAAL_HOME>/bin/jvisualvm`
* **dedicated VisualVM**: since I still had an ancient Mac-Installation of JDK I explicitly had to point it to the correct JDK: `/Applications/VisualVM.app/Contents/MacOS/visualvm --jdkhome ~/.sdkman/candidates/java/14.0.1-zulu` 

{{< figure width="100%" src="/posts/troubleshooting-containerized-java-applications/visualvm.png" alt="VisualVM">}}

Using the CPU-sampler I was finally able to identify and fix the mentioned performance issue.

## Using MissionControl and FlightRecorder

Although using [VisualVM][visualvm] most of the time is enough to pinpoint (simple) performance issues I was curious about the integration with [MissionControl and FlightRecorder][jmc].

> Java Flight Recorder is a profiling and event collection framework [...] Mission Control is an advanced set of tools 
> that enables efficient and detailed analysis of the extensive data collected by Java Flight Recorder.

The tools are conceived to impose as little overhead to a running application as possible and thus can even be used in production environments - well, and are a [commercial feature hence][jmc-com].

Fortunately, after further googling I found out that [Azul][azul], the company behind the [Zulu-OpenJDK][zulu-jdk] (which is also the JDK used in the [payara-image][payara-full] (and maaaany others)), also has an open source version of Mission Control called [Zulu Mission Control][zmc-download]. It's open source, free to download and ... 

> ... you can use Zulu Mission Control on Zulu Enterprise, Zing, and Zulu Community builds of OpenJDK ...

You don't even have to use the infamous flag `-XX:+UnlockCommercialFeatures` anymore. Very nice!

So, let's [download and unpack ZMC (Zulu Mission Control)][zmc-download]. Just like VisualVM also ZMC had an issue with may ancient Mac-Java - so I had to explicitly point it to the correct JDK. For that, find the `zmc.ini`-file (for Mac this is `<APP-FOLDER>/Contents/Eclipse/zmc.ini`) and add the `-vm` option: 

{{< highlight bash>}}
-vm
/Users/schoeffm/.sdkman/candidates/java/14.0.1-zulu/bin
{{< /highlight >}}

This is already enough to get started. Start ZMC, connect it via JMX to the running, containerized process and have look around

{{< figure width="100%" src="/posts/troubleshooting-containerized-java-applications/zmc.png" alt="Zulu Mission Control">}}

Finally, when working with an application server there are still some JVM-options that are worth changing/setting to get better results. Similar to the JMX-options we've set above we could inject the followin' `asadmin`-commands (in case of Payara - otherwise, just pass 'em to the JVM):

{{< highlight bash>}}
asadmin create-jvm-options '-XX\:FlightRecorderOptions=stackdepth=2048'
asadmin create-jvm-options '-XX\:+UnlockDiagnosticVMOptions'
asadmin create-jvm-options '-XX\:+DebugNonSafepoints'
{{< /highlight >}}

`FlightRecorderOptions` increases the stackdepth that gets recorded by FlightRecorder (otherwise stack traces would be truncated). `DebugNonSafepoints` improves fidelity as explained [here][fidelity].

To get more information about ZMC and its background have a look at [this webinar recording][zmc-video].

Happy sampling!     

[bien-jmx]:https://www.adam-bien.com/roller/abien/entry/how_to_establish_jmx_connection
[zmc-download]:https://www.azul.com/products/zulu-mission-control/
[zmc-video]:https://www.azul.com/presentation/azul-webinar-open-source-flight-recorder-and-mission-control-managing-and-measuring-openjdk-8-performance/
[jmc]:https://www.oracle.com/technetwork/java/javaseproducts/mission-control/index.html
[jmc-com]:https://docs.oracle.com/javacomponents/jmc-5-5/jfr-runtime-guide/about.htm#JFRRT107
[jmc-install]:https://www.oracle.com/technetwork/java/javase/jmc-install-6415206.html
[jmc-download]:http://jdk.java.net/jmc/
[jmc-8-download]:https://adoptopenjdk.net/jmc
[jmc-download-oracle]:https://www.oracle.com/java/technologies/javase-downloads.html
[git-repo]:https://github.com/openjdk/jmc
[visualvm]:https://visualvm.github.io/
[payara-full]:https://hub.docker.com/r/payara/server-full/
[jconsole]:https://openjdk.java.net/tools/svc/jconsole/
[sdkman]:https://sdkman.io/
[graalvm]:https://www.graalvm.org/
[zulu-jdk]:https://www.azul.com/downloads/zulu-community/?architecture=x86-64-bit&package=jdk
[azul]:https://www.azul.com/
[fidelity]:https://docs.oracle.com/javacomponents/jmc-5-5/jfr-runtime-guide/about.htm#JFRRT111