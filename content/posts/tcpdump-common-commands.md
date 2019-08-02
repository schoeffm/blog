+++
title = "Tcpdump Common Commands"
date = 2019-07-30T10:08:07+02:00
publishDate = 2019-07-30
tags = ["tcpdump", "cli", "devops", "network", "tools" ]
+++

When debugging network related issues the CLI tool `tcpdump` is a valuable assistant. I usually use a variation of this base command:

{{< highlight bash>}}
    sudo tcpdump -A -i lo0 -n -s0 -v port 8080
{{< /highlight >}}
<!--more-->
- __-A__: Outputs the captured packed in ASCII. Since most of the time I use it for debugging web-apps or REST-interfaces this is a life-safer.
- __-n__: doesn't convert addresses to names (which is of not much value when debugging localhost-traffic)
- __-i lo0__: select the interface whose traffic you'd like to capture (i.e. `lo0` for loopback interface a.k.a. localhost)
- __-s0__: deactivate a fixed snapshot-length (or more precisely, fallback to the internal default) to not drop packages 'cause of their size.
- __port 8080__: limit capturing to this port (also valuable since you'll notice that there is a bunch of noise flying around)
- __-v(vv)__: Varies verbosity of the output

Again, this is the usual base command I start with. There are a gazillion more options and tweaks at your disposal.

#### Write/Read a capture file

You can also write a dump-file for later use or to import that file into a GUI-tool like [Wireshark][wireshark].

{{< highlight bash>}}
    sudo tcpdump -A -i en0 -w network.dump.pcap # write file
    sudo tcpdump -r network.dump.pcap # read file
{{< /highlight >}}

#### Capture traffic by host

When debugging beyond `lo0` it's also valuable to focus on just one specific network partner. For that you can use `host`, `src` and `dst` respectively.

{{< highlight bash>}}
    sudo tcpdump -A -i en0 host 192.168.178.24 # from and to
    sudo tcpdump -A -i en0 dst 192.168.178.24  # only to
    sudo tcpdump -A -i en0 src 192.168.178.24  # only from
{{< /highlight >}}

#### There's still more

Although this is just the tip of the iceberg most of the time it is already sufficient for my use cases. <br/>
But of course there's more to reveal - you can find more (complex) examples in this excellent [post at hackertarget.com][post]. And to understand what you're actually typing I recommend comparing the examples with the man-pages of `tcpdump`.

[post]:https://hackertarget.com/tcpdump-examples/
[wireshark]:https://www.bing.com/search?q=wireshark&FORM=AWRE
