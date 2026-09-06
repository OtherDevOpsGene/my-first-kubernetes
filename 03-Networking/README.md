# Kubernetes networking

To get our services talking to each other, we need to expose the them to the
other services.

## kubectl expose

We will do this by exposing the ports they will use to communicate.

```shell
kubectl expose pod reviews --port 9080 --target-port 9080
kubectl expose pod details --port 9080 --target-port 9080
kubectl expose pod productpage --port 9080 --target-port 9080
```

The `--port` option tells the cluster what port number we will be using, and the
`--target-port` specifies the port on the container it will be mapped to. Since
Kubernetes has assigned a separate IP address for each pod, we can use the same
port number for each of the services.

Also, we are explicitly opening `productpage` on a port that is **not** open to
the Internet. More on that later.

BTW, `kubectl expose` is known as the *imperative* form of creating a service-
imperative because we tell the cluster what we want done. Similarly with
`kubectl run` for starting a pod. The imperative form is useful on occasion, for
learning, and on certification exams, but we'll get out of the habit soon
enough. Then we will move to the *declarative* form, where we declare what we
want the end result to be and Kubernetes makes the necessary adjustments to get
us there. The declarative form has the strong advantage of not depending what
the starting conditions were, only on what it needs to end up as.

Let's see the services we created.

## ClusterIP

```shell
kubectl get services
```

We see four services- the three we just started and one that `k3s` already had
running. And although we didn't specify, they all have a type of `ClusterIP`.

`ClusterIP` is the default networking option and means that the networking is
handled entirely within the cluster. The IP address it uses is non-routable- it
cannot be accessed from outside the cluster. You would have to be on the node or
within a pod to access it.

Let's test it out. We'll need the `ClusterIP` for the `productpage` service so
we can put together a URL.

## Parsing output

If we don't want the entire list of resources from the `kubectl get`
*resource_type* command, we can be more specific and ask for a particular
*resource_name*.

```shell
kubectl get services productpage
```

We can grab the `ClusterIP` from the table ourselves, or we could get fancy with
output and use [jq](https://jqlang.github.io/jq/) to parse the JSON output to
spit out only the value we want.

```shell
kubectl get services productpage -o json | jq -r '.spec.clusterIP'
```

If we didn't want to involve another utility like `jq`, we could stay within
`kubectl` and use [JSONPath](https://goessner.net/articles/JsonPath/).

```shell
kubectl get services productpage -o jsonpath='{.spec.clusterIP}'
```

Or we could stay away from JSON altogther and use a [Go
template](https://pkg.go.dev/text/template).

```shell
kubectl get services productpage --template='{{.spec.clusterIP}}'
```

Which formatting option to use is very much a personal choice, although each
does have their own control loops and other functions available to them.

## Back to networking

Replacing `10.777.888.999` with the `ClusterIP` for `productpage`, try to access
the URL from the command line.

```shell
curl http://10.777.888.999:9080/productpage
```

It should work, since we are on a cluster node. It would not work from your
neighbor's system, since their node is not part of this cluster, even if the
network was open to port 9080 between them.

This is the networking that Kubernetes uses internally, although it doesn't
bother looking up IP addresses. Internally, *service discovery* just uses DNS.

Our node is just an EC2 instance on the Internet, so it configured to use the
normal DNS servers we are accustomed to. But we can see the internal DNS in
action if we look from the container's point of view.

```shell
kubectl exec reviews -- curl -sS http://productpage:9080/productpage
```

The `kubectl exec` command runs the follow commands (everything after `--`) on
the pod requested. Whatever command we run has to be available on the container
we are running it from- there is no magic installation of utilities happening.
But that can make `kubectl exec` a powerful troubleshooting tool.

If needed, we could even stand up a pod within the cluster that had the tool or
tools we wanted to use, just so we'd be *inside* the cluster while
troubleshooting.

```console
$ kubectl run -it --rm --restart=Never curl --image=alpine/curl sh
All commands and output from this session will be recorded in container logs, including credentials and sensitive information passed through the command prompt.
If you don't see a command prompt, try pressing enter.
# curl -sS http://productpage:9080/productpage
...
# exit
pod "curl" deleted from default namespace
```

or

```console
$ kubectl run -it --rm --restart=Never alpine-bind-tools --image=docker.io/otherdevopsgene/alpine-bind-tools nslookup productpage
;; Got recursion not available from 10.43.0.10
Server:         10.43.0.10
Address:        10.43.0.10#53

Name:   productpage.default.svc.cluster.local
Address: 10.43.18.190
;; Got recursion not available from 10.43.0.10

All commands and output from this session will be recorded in container logs, including credentials and sensitive information passed through the command prompt.
If you don't see a command prompt, try pressing enter.
Session ended, resume using 'kubectl attach alpine-bind-tools -c alpine-bind-tools -n default -i -t' command
pod "alpine-bind-tools" deleted from default namespace
```

Again, this only works within the cluster itself.

```console
$ curl http://productpage:9080/productpage
curl: (6) Could not resolve host: productpage
$ curl http://productpage.default.svc.cluster.local:9080/productpage
curl: (6) Could not resolve host: productpage.default.svc.cluster.local
```

## NodePort

The next networking choice for Kubernetes is `NodePort`. A service exposed using
`NodePort` gets a high port (30000 to 32768) from the cluster and uses the IP
address of the node itself.

```shell
kubectl delete service productpage
kubectl expose pod productpage --port 9080 --target-port 9080 --type NodePort
kubectl get services
```

Replace `33333` with the port listed for the `productpage` service.

```shell
curl -sS http://${PRIVATE_IPV4}:33333/productpage
```

If the high ports for `NodePort` were open between systems, you could use your
neighbor's private IP address to access their copy of the app. In our case,
those port **are** open, specifically for this experiment. Normally, there would
be no good reason to keep them open, and very good reasons not to.

Ask your neighbor for their private IP address, and replace `172.111.222.444`
with that address and `33333` with the `NodePort` for `productpage`.

```shell
curl -sS http://172.111.222.444:33333/productpage
```

## Exercise

Craft a statement to pull the reviews and details for book ID 2 (or whatever
integer, it literally doesn't matter) by passing it as path info on the URL
(i.e., add `/2` to the end of the URL). The response will be JSON.

<!-- curl -sS http://$(kubectl get service reviews -o jsonpath='{.spec.clusterIP}'):$(kubectl get service reviews -o jsonpath='{.spec.ports[0].port}')/reviews/2 | jq -->

Then modify the command to run from one of the other pods. E.g., pull the
details while on the `reviews` pod.

<!-- kubectl exec reviews -- curl -sS http://details:$(kubectl get service details -o jsonpath='{.spec.ports[0].port}')/details/2 | jq -->

## Clean up

There are more ways to make our services available to outside of the cluster,
but we'll get to those in a later lesson. For now, let's clean up.

```shell
kubectl delete service details productpage reviews
kubectl delete pod details productpage reviews
```

The `kubectl delete` command works just like most of the others.

## End of lesson

We will move away from plain pods onto something more robust in
[04-Deployments](../04-Deployments/README.md).
