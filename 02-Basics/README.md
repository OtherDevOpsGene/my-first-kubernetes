# Kubernetes basics

Now, let's stand up the same application using Kubernetes.

## Cluster

We will start with a cluster already available to us. We'll discuss standing up
a cluster from scratch later.

We will use the simple but full-featured [k3s](https://k3s.io/). There are
plenty of other options for Kubernetes control planes, the parts of Kubernetes
that handle all the Kubernetes-ness, from
[minikube](https://minikube.sigs.k8s.io/docs/) or
[kind](https://kind.sigs.k8s.io/) to [Amazon Elastic Kubernetes
Service](https://aws.amazon.com/eks/). Almost everything we do here will work
with any control plane since we are working through the Kubernetes API.

`k3s` was already installed for you, but if you need it in a different
environment, all it takes is `curl -sfL https://get.k3s.io | sh -`

We can tell that our cluster running by accessing the Kubernetes API through our
soon-to-be favorite tool, `kubectl`.

```shell
kubectl cluster-info
```

Technically, `kubectl` just made an API call to the control plane, running
locally on port 6443, and parsed and displayed the response. At least we know
the cluster is running and accessible to us.

We can see a little more by checking out the nodes that are part of our cluster,
in this case just a single node.

```shell
kubectl get nodes
```

A node is just where Kubernetes is running. Nodes can be part of the control
plane, a worker node that hosts containers, or both.

Still not much to see, but this does introduce one of the most common commands
we'll use, `kubectl get` *resource_type*. That provides us a list of all the
resources of the requested type with some basic information. There are caveats
to it, but that is enough to understand for now.

Now let's get our application running.

## Pods

Pods are a wrapper around one or more container images that are deployed and
managed together, very often just one. They are one of the most basic building
blocks for Kubernetes.

```shell
kubectl get pods
```

Currently, we have not started any pods.

We can run the same **BookInfo** microservices application as before, but this
time using Kubernetes.

<img src="../bookinfo-basic.svg">

As a reminder, requests come into `productpage`. That service makes requests to
`reviews` and `details` to gather the information to display. And again, we'll
ignore `ratings` for now.

```shell
kubectl run reviews --image docker.io/istio/examples-bookinfo-reviews-v1:1.20.3
kubectl run details --image docker.io/istio/examples-bookinfo-details-v1:1.20.3
kubectl run productpage --image docker.io/istio/examples-bookinfo-productpage-v1:1.20.3
```

That isn't significantly different than starting the containers with Docker,
other than ignoring our manual configuration via environment variables. And
because of that it isn't quite functional. These are just independently running
containers that aren't specifically configured to find and talk to each other at
this point.

We can see what we've started by getting the list of the running pods.

```shell
kubectl get pods
kubectl get pods -o wide
```

If you are quick enough, you might notice the status of the containers going
from `ContainerCreating` to `Running`. You might also see the ready containers
go from 0 ready of 1 requested (`0/1`) to all 1 of 1 ready (`1/1`).

Adding the output format (`-o wide`) gets us a little more information about
each pod. Notice that the cluster has assigned an internal IP address to each of
the pods.

We can also see that the pods have all been *scheduled* on the same node, not
that there were any alternatives at this point.

If we wanted to really deep-dive into a pod, we can use the `kubectl describe`
*resource_type* *resource_name* command.

```shell
kubectl describe pod productpage
```

We see that the control plane has made a lot of decisions behind the scenes. We
can use a more explicit version of the `kubectl get` command to get a similar
level of detail in different formats.

```shell
kubectl get pod productpage -o json
kubectl get pod productpage -o yaml
```

For now, these details aren't immediately important to us. Just be aware that we
can get more details about a resource from `kubectl describe` or `kubectl get`
if we want.

## Exercise

1. Find the labels that were automatically assigned to each of the three pods.
2. Use the [Kubectl
   Reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
   to craft a `kubectl get` command to show only the `reviews` and `details`
   pods using the label selectors.

<!-- run=reviews, run=details, run=productpage -->

<!-- kubectl get pods -l 'run!=productpage' -->

## End of lesson

We will get the **BookInfo** application actually running in the next lesson,
[03-Networking](../03-Networking/README.md).
