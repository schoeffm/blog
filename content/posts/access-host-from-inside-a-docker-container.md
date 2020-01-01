+++
title = "Access Host From Inside a Docker Container"
date = 2020-01-01T12:00:56+01:00
tags = [ "docker", "dev", "docker4mac" ]
+++

This is more a bookmark than a post - but I'd like to write it down explicitly since everytime I need this I have to google it (and cutting through the false positivies sucks).

> From 18.03 onwards our recommendation is to connect to the special DNS name `host.docker.internal`, which resolves to the internal IP address used by the host.

This was taken from the [docker-for-mac documentation][docu] were you can find more details.

__Notice:__ This also works for locally mapped ports of other containerized services (and thus is handy for using containerized CLI tools with existing stacks). 

__Notice:__ This is not just a docker-for-mac thing but also works for [docker-for-win][docu-win]

[docu]:https://docs.docker.com/docker-for-mac/networking/#i-want-to-connect-from-a-container-to-a-service-on-the-host
[docu-win]:https://docs.docker.com/docker-for-windows/networking/#i-want-to-connect-from-a-container-to-a-service-on-the-host
