---
layout:      post
title:       Unrecognized Name issues with HomeAssistant
subtitle:    How to resolve the Unrecognized Name issues within Kubernetes Pods
description: >
    Recently I ran into an issue while trying to create a new instance of HomeAssistant on a K3s install of Kubernetes. 
    Getting HomeAssistant up and running and connected to an MQTT server within the same cluster worked as expected.
    However, the issue came up when I attempted to get it to connect to NabuCasa or make connections out to any TLS/SSL (HTTPS) addresses. 
    Viewing the HomeAssistant logs from the `Settings->System->Logs` area, I noticed I was getting a very vague error of `tlsv1 unrecognized name`. 
date:        2024-03-20 11:00:00 -0600
author:      Nik Verbeck
image:       ""
tags:
    - kubernetes 
    - k3s 
    - k8s 
    - iot 
    - homelab 
    - self-hosted 
    - homeassistant 
    - has 
    - tls 
    - ssl 
    - https 
    - dns
categories:
    - HomeLab
    
---

Recently I ran into an issue while trying to create a new instance of HomeAssistant on a K3s install of Kubernetes. 
Getting HomeAssistant up and running and connected to an MQTT server within the same cluster worked as expected.
However, the issue came up when I attempted to get it to connect to NabuCasa or make connections out to any TLS/SSL (HTTPS) addresses. 
Viewing the HomeAssistant logs from the `Settings->System->Logs` area, I noticed I was getting a very vague error of `tlsv1 unrecognized name`. 
Searching around the internet for the issue affecting HomeAssistant just kept turning up people having issues with DNS but no actual "What the DNS issue was".
That was until I came across a [Kubernetes StackOverflow Question](https://stackoverflow.com/questions/66630929/kubenetes-pod-curl-works-only-if-domain-name-ends-with) where the person was getting the same issue but with a completely different application.

## The Issue

In the above-mentioned StackOverflow question, the user noted that if he added the final `.`, ex `https://example.com` vs `https://example.com.`, to the addresses would actually allow the connection to work as expected. 
This led me to test with some simple `CURL` commands to see if I was facing the same issue, see below for those simple tests.
Now that I had verified that I was getting the same issue, I could move on to the fix.

### Verifying the Issue

1. Get a shell on the HomeAssistant pod with `kubectl exec <pod-id> -n <namespace-of-pod> -it -- /bin/sh`
2. Executing `curl -vvI https://google.com` will result in a failed connection with the error `tlsv1 unrecognized name`
3. Executing `curl -vvI https://google.com.` doesn't result in any errors and would succeed as you'd expect

## The Fix

Reading further down the StackOverflow Questions, an answer was provided.
In the answer, they noted that the issue lies with the `ndots` configuration within `/etc/resolve.conf`. 
With newer Kubernetes versions, a change was made to adjust `ndots` to 5 from its default of 1.
Which helps with resolving addresses internally in the Kubernetes cluster but prevents proper external DNS queries from resolving correctly.
If you want to understand the issue more, read ["DNS Lookups in Kubernetes"](https://mrkaran.dev/posts/ndots-kubernetes/), also linked in the StackOverflow answer.

In an attempt to resolve the issue I tried adjusting `dnsPolicy: Default` on the HomeAssistant pod. 
Which, while it worked to resolve my initial issue, it created a new issue in that I couldn't reach my MQTT server anymore.
This was because the DNS servers would now show up in `/etc/resolve.conf` as those of the Kubernetes Node rather than the internal CoreDNS address.
Reverting that change, and instead doing a `dnsConfig` with `ndots=1` did resolve my initial issue and still allowed MQTT to be resolvable.

### Example HomeAssistant Deployment with the fix applied

{{< gist nerdynick f4d650b15a91cbe31927f6220c216779 homeassistant.yml >}}

## End

I hope this helps all those that run into the same issue and had a hard time finding answers on the internet.
Plus a shootout [mrbm](https://stackoverflow.com/users/1143428/mrbm) for the SO Answer and [Karan Sharma](https://mrkaran.dev/posts/ndots-kubernetes/) for the in-depth writeup on the issue.