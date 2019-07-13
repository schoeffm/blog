+++
title = "SSH Config Format"
date = 2019-07-11T15:35:15+02:00
publishDate = 2019-07-11
draft = false
tags = ["ssh", "devops", "cli", "configuration"]
+++

Dealing with SSH-connections can become cumbersome quickly especially when configurations change frequently, too. 

Luckily, you can pre-configure most of a connections settings in a user specific config-file located in your `~/.ssh`-folder called `config`.
<!--more-->
For example:

{{< highlight bash >}}
# content of ~/.ssh/config
Host gateway
    HostName 160.46.188.153
    User cloud
    IdentityFile ~/.ssh/id_rsa
{{< /highlight >}}

This entry is equivalent to the inline command `ssh cloud@160.46.188.153 -i ~/.ssh/id_rsa` but can be leveraged now by: 

{{< highlight bash >}}
$ ssh gateway
{{< /highlight >}}
Much shorter and even more expressive than a cryptic IP-address.

Of course, you can add a lot more settings - for special use-cases lookup the [documentation for ssh-config][docu] files. Here are some usual suspects that come handy:

{{< highlight bash "linenos=table,hl_lines=6-8 11">}}
# content of ~/.ssh/config
Host gateway
    HostName 160.46.188.153
    User cloud
    IdentityFile ~/.ssh/id_rsa
    ForwardX11 yes
    ForwardAgent yes
    LocalForward 9906 127.0.0.1:443

Host monitoring
    ProxyCommand ssh cloud@160.46.188.153 -W %h:%p
    HostName 10.135.0.20
    User cloud
    IdentityFile ~/.ssh/id_rsa
    ForwardAgent yes
{{< /highlight >}}

- line 6: we'd like to use X11 forwarding (using GUI-based tools executed on the remote machine)
- line 7: we activated [agent forwarding][forward-agent] so when connected to `gateway` we can re-use our (local) SSH-key (without placing it on the remote machine) in order to i.e. do further SSH-hops or pull a GIT repo (via SSH).
- line 8: For the `gateway`-host we created a local port-forwarding so we can access the HTTPS-port on that machine via our local port `9906`. 

- line 11: In those cases where we have to go through a SSH-gateway (a.k.a. jumphost) we would have to make two SSH-hops in a row (first, SSH into the machine that has a public IP - then SSH further into the actual destination machine using its internal IP). <br/>
  In order to do this in one go we've setup a `ProxyCommand` - so by issuing `ssh monitoring` we directly do two hops and end up at the destination host.

[forward-agent]:https://dev.to/levivm/how-to-use-ssh-and-ssh-agent-forwarding-more-secure-ssh-2c32
[docu]:https://linux.die.net/man/5/ssh_config
