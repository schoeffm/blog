+++
title = "Easy en-/decoding using base64"
date =  2019-07-13T14:58:39+02:00
publishDate = 2019-07-13
tags = ["base64", "openshift", "kubernetes", "cli"]
+++

When dealing with [Kubernetes secrets][secrets] or when testing REST-services using i.e. `curl` it's nice to have a fast'n'easy way of de- and encoding characters using [base64][base64]. 
<!--more-->
{{< highlight bash >}}
$ echo -n "Hello World" | base64         # SGVsbG8gV29ybGQ=
$ echo -n "SGVsbG8gV29ybGQ=" | base64 -D # Hello World
{{< /highlight >}}

To make it even more convenient you can define properly named functions in your `.aliases` (or `.bashrc`/`.zshrc`):
    
{{< highlight bash >}}
function enc() { echo -n "$1" | base64 }
function dec() { echo -n "$1" | base64 -D }
{{< /highlight >}}

Now the initial example can be accomplished using:
    
{{< highlight bash >}}
enc "Hello World"                        # SGVsbG8gV29ybGQ=
dec SGVsbG8gV29ybGQ=                     # Hello World
{{< /highlight >}}


[base64]:https://de.wikipedia.org/wiki/Base64
[secrets]:https://kubernetes.io/docs/concepts/configuration/secret/
