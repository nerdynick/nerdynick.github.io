---
layout: post
title:  Exposing Kubernetes Services with DNS External using CoreDNS
date:   2023-12-28 14:30:00 -0600
categories: kubernetes k3s k8s iot dns coredns homelab self-hosted externaldns
---

In my efforts to run a self-hosted Kubernetes cluster(s) in my lab. 
I needed a way to easily expose services outside of the cluster for easy use by me and the family/friends.
In my research, I stumbled upon the Kubernetes Sig [ExternalDNS](https://github.com/kubernetes-sigs/external-dns){:target="_blank"}.
Its goal is to leverage K8 Service Annotates to provide the desired external DNS entry values and the respective configs to go along with it. 
However, in my attempt to make this work in a self-hosted environment with minimal deployment in a small footprint on small hardware (RPis mostly), I came across many challenges.
Many of the integrations that ExternalDNS supports are focused on Cloud Provider DNS services rather than self-hosted one. 
Of the self-hosted DNS options, many have large footprints in their deployments. 
Such as the need for Databases or other services. 
This left me looking at my favorite DNS service, CoreDNS. 
However, in an attempt to get the CoreDNS ExternalDNS integration to work. 
I found out that it requires an [etcd](https://etcd.io/){:target="_blank"} service to be available. 
As well as that the plugins the integration relies on at both the ExternalDNS and CoreDNS sides appear to not be maintained much anymore.
Yet alone, the etcd approach is seen as a deprecated one in its own right. 

This led me to explore other options.
That's when I came across External Plugin for CoreDNS [k8s_gateway](https://github.com/ori-edge/k8s_gateway){:target="_blank"}.
It offered everything I was looking for, all in a single package.
However, unfortunately at the time, it only supported its proprietary Annotations.
Which as I found out not long after, would be an issue.

Many projects have embraced ExternalDNS as the defacto service for exposing DNS externally.
From what I can gather, that's mostly because it is installed in all 3 of the major cloud providers' kube clusters by default.
What this all means is that for many of these projects, they will accept a DNS entry value but won't let you change the annotations it's going to use. 
Resulting in the k8s_gateway being ineffective at the goal I was attempting to solve.
However, the framework was already there to do what was needed and a simple [PR](https://github.com/ori-edge/k8s_gateway/pull/170) later and the support for ExternalDNS' annotations was added.

# The Deployment

Now for what you most likely came here for, the deployment itself.
I've broken it up into 2 parts, but everything here can be put into one `kubectl` deployment.

## Part 1: Create our Service Account and RBAC Permissions

Since we will be leveraging the Kube APIs. 
We will need to allow access to the respective needed APIs to poll the information needed to find services and lookup their tags.

{% gist 5718e7b706b7990c2cbc76881690a8c8 coredns_permissions.yml%}

## Part 2: Create the Deployment and Service

{% gist 5718e7b706b7990c2cbc76881690a8c8 coredns.yml%}

> **_NOTE:_**  You will notice that there is an *nodeAffinity* resitrcting the pods to be deployed only on amd64 machines. This is because, as of this writing, the pre-built docker image for [k8s_gateway](https://github.com/ori-edge/k8s_gateway){:target="_blank"} is only being built for AMD64 archatectures. Hopefully one day arm will be added to the pre-builds.