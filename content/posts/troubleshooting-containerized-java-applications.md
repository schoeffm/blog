+++
title = "Troubleshooting Containerized Java Applications via JMX"
date = 2020-05-02T13:18:06+02:00
draft = true
tags = ["java", "jmx", "flightrecorder", "missioncontrol", "zmc", "visualvm", "docker", "payara", "performance", "analysis" ]
+++

When developing JEE applications these days we not only deploy 'em as containerized apps but (at least in my teams) also use containers for local development. 

One issue I was confronted with the other day was a performance bug with one of our apps. Back in the days I'd have just fired up [VisualVM][visualvm] to connect to the local process causing trouble - but now, the process to attach to doesn't run locally anymore.
<!--more-->
## JMX pre-requisite
At the end of the day the solution is pretty simple - just treat the containerized process as an external app (i.e. like an app that was deployed on a remote server) and connect to it via JMX. Unfortunately, this can become quite tricky. 

1) __Turn on JMX__
{{< highlight bash>}}
-Dcom.sun.management.jmxremote 
-Dcom.sun.management.jmxremote.port=9090
-Dcom.sun.management.jmxremote.rmi.port=9090
{{< /highlight >}}
This activates JMX and binds the JMX-server to port 9090 ([why you need to repeat the ports][jmx-ports])
2) __Turn off authentication and transport-layer security__ 
{{< highlight bash>}}
-Dcom.sun.management.jmxremote.ssl=false 
-Dcom.sun.management.jmxremote.authenticate=false 
{{< /highlight >}}
You should think twice before you turn off those security features. It's way easier to set things up this way at the expense of security of course. IMHO for my use-cases the threat is negligible:
    - use JMX locally for debugging purposes
    - use it within a kubernetes setup where I forward that port to my local machine and don't expose the JMX server to others (more on that later).

    Anyways, you have to make that decision yourself in the context of your project.
3) __Setting up the IP/Host__: This is the tricky part and has [sth. to do with JMX on linux][jmx-issues] (which your container is most probably running). When you execute `hostname -i` within your container it'll resolve to an IP-address not know to/resolvable by the outside world.<br/> I found two solutions to this issue:

    1) setting the JMX-host explicitly
{{< highlight bash>}}
-Djava.rmi.server.hostname=<CONTAINER_IP>
{{< /highlight >}}
`CONTAINER_IP` has to be the external IP-address assigned to the container. So for your local docker environment this would be the hosts address (sth. like `192.168.178.33`).<br/> This approach though is less portable since all of your collegues will have to change that address to match their enviornment.
    2) setting the hostname via `docker`
{{< highlight bash>}}
docker run --rm -ti --hostname localhost payara/server-full
{{< /highlight >}}
When starting your container (either via `docker-compose` or plain `docker`) you can pass in the hostname the container should have. If you turn that to `localhost` the container will resolve its hostname to:
{{< highlight bash>}}
payara@localhost:~$ hostname -i
127.0.0.1 172.26.0.4 ::1
{{< /highlight >}}

Lets take all this toghether and start a `payara/server-full` container with active JMX:

{{< highlight bash>}}
docker run --rm -ti \
    -e JVM_ARGS="-Dcom.sun.management.jmxremote.port=9090 -Dcom.sun.management.jmxremote.rmi.port=9090 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false" \
    --hostname localhost \
    -p 9090:9090 -p 8080:8080 \
    payara/server-full
{{< /highlight >}}
or you set the JMX-host explicitly (assuming your machines IP is `192.168.178.22`):
{{< highlight bash>}}
docker run --rm -ti \
    -e JVM_ARGS="-Djava.rmi.server.hostname=192.168.178.22 -Dcom.sun.management.jmxremote.port=9090 -Dcom.sun.management.jmxremote.rmi.port=9090 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false" \
    -p 9090:9090 -p 8080:8080 \
    payara/server-full
{{< /highlight >}}

In both cases you'd be able to connect to your containerized app via:
{{< figure width="60%" src="/posts/troubleshooting-containerized-java-applications/jconsole.png" alt="jconsole">}} 

### Connecting to pods running on k8s or openshift

So far we just looked at how to connect to a container running in your local docker-host. But when dealing with JMX (and the capabilities waiting beyond, like FlightRecorder and MissionControl) it's maybe even more interesting to connect to productive applications running in a kubernetes-ish orchestrator (nothing else seems to be relevant anyways).

Assuming you've setup the container within the pod to start a JMX-server that binds to `127.0.0.1:9090` (using the means from above) you could forward that port to your local machine with:
{{< highlight bash>}}
kubectl port-forward <POD-NAME> 9090:9090
{{< /highlight >}}

Subsequently, you're again able to connect to `localhost:9090` using JConsole or VisualVM exploring information about the remote container.

## Leverage tools now

### JConsole

To test the setup just start your container, add a port-mapping for the JMX-port 
{{< highlight bash>}}
docker run --rm -ti \
    -e JVM_ARGS="-Dcom.sun.management.jmxremote.port=9090 -Dcom.sun.management.jmxremote.rmi.port=9090 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false" \
    --hostname localhost \
    -p 9090:9090 -p 8080:8080 \
    payara/server-full
{{< /highlight >}}

and use i.e. `<JDK>/bin/jconsole` (which is most probably still part of your JDK-installation) to connect to the respective JMX-socket on `localhost:9090` (_confirm any security warnings_).

{{< figure width="60%" src="/posts/troubleshooting-containerized-java-applications/jconsole.png" alt="jconsole">}} 

### VisualVM

`jconsole` is nice, but the aforementioned [VisualVM][visualvm] is able to give us deeper insights. 

Unless you're using [GraalVM][graalvm], your JDK either doesn't contain VisualVM anymore or it contains an ancient version. It was shipped as [part of Oracle JDK until version 8][visualvm-history] but was discontinued and is [distributed now as independent tool][visualvm]. Since I use [sdkman][sdkman] to manage parallel JDK-installations I incidentally already had a GraalVM installation which included VisualVM. 

If that is not the case you can [download VisualVM][visualvm] as a separate tool or check your package manager (like `brew cask install visualvm`).

* **VisualVM of GraalVM**: here it was enough to just start the app from within the `bin`-directory: `<GRAAL_HOME>/bin/jvisualvm`
* **dedicated VisualVM**: since I still had an ancient Mac-Installation of JDK I explicitly had to point it to the correct JDK: `/Applications/VisualVM.app/Contents/MacOS/visualvm --jdkhome ~/.sdkman/candidates/java/14.0.1-zulu` 

Via _Add JMX Connection_ you can then connect using `localhost:9090` to your containerized app again:

{{< figure width="100%" src="/posts/troubleshooting-containerized-java-applications/visualvm.png" alt="VisualVM">}}

In contrast to JConsole, VisualVM offers a CPU- and Memory-Sampling Profiler which is very useful to identify bottlenecks and general performance hotspots.

### Using MissionControl and FlightRecorder

Although using [VisualVM][visualvm] most of the time is enough to pinpoint (simple) performance issues I was curious about the integration with [MissionControl and FlightRecorder][jmc].

> Java Flight Recorder is a profiling and event collection framework [...] Mission Control is an advanced set of tools 
> that enables efficient and detailed analysis of the extensive data collected by Java Flight Recorder.

The tools are conceived to impose as little overhead to a running application as possible and thus can even be used in production environments - well, and were a [commercial feature hence][jmc-com].

Things have changed though and starting [from Oracle JDK 11 on][jmc-faq] FlightRecorder was (along with the actual JDK) open-sourced. Additionally, the guys from [Azul][azul] backported that feature also to their [JDK 8 offering][zmc-download] (even for the community build).

> ... you can use Zulu Mission Control on Zulu Enterprise, Zing, and Zulu Community builds of OpenJDK ...

Since in our examples we used `payara/server-full` which is based on `azul/zulu-openjdk` we're able to leverage that tools. So, let's [download and unpack ZMC (Zulu Mission Control)][zmc-download]. 

Just like VisualVM also ZMC had an issue with may ancient Mac-Java - so I had to explicitly point it to a JDK to be used. For that, find the `zmc.ini`-file (for Mac this is `<APP-FOLDER>/Contents/Eclipse/zmc.ini`) and add the `-vm` option: 

{{< highlight bash>}}
-vm
/Users/schoeffm/.sdkman/candidates/java/14.0.1-zulu/bin
{{< /highlight >}}

After that, start the tool, connect it via JMX to the running, containerized process and have look around

{{< figure width="100%" src="/posts/troubleshooting-containerized-java-applications/zmc.png" alt="Zulu Mission Control">}}

For Zulu-JDK you don't even have to set `-XX:+FlightRecorder` anymore to activate the feature since it's activated by default already. But when working with an application server there are still some JVM-options worth changing/setting to get better results. 

{{< highlight bash>}}
-XX:FlightRecorderOptions=stackdepth=2048
-XX:+UnlockDiagnosticVMOptions
-XX:+DebugNonSafepoints
{{< /highlight >}}

`FlightRecorderOptions` increases the stackdepth that gets recorded (otherwise stack traces would be truncated) and `DebugNonSafepoints` (which has to be combined with `UnlockDiagnosticVMOptions`) improves fidelity as explained [here][fidelity].

You could even start a continous recording that will be started along with the process and that stores its information in a ring-buffer waiting happily for being dumped and analyzed on demand by adding:

{{< highlight bash>}}
-XX:StartFlightRecording=disk=true,name=core,dumponexit=true,maxage=3h,settings=profile 
{{< /highlight >}}

To get more information about ZMC and its background have a look at [this webinar recording][zmc-video].

Happy troubleshooting!     

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
[jmx-ports]:https://stackoverflow.com/questions/20884353/why-java-opens-3-ports-when-jmx-is-configured#answer-21552812
[jmx-issues]:https://docs.oracle.com/javase/8/docs/technotes/guides/management/faq.html#linux1
[visualvm-history]:https://visualvm.github.io/javavisualvm.html
[jmc-faq]:https://wiki.openjdk.java.net/display/jmc/JMC+FAQ
