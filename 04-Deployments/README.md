# Using Deployments

Pods may be a basic building block for Kubernetes, but they don't add much
beyond Docker containers. If we want to take advantage of Kubernetes
orchestration capabilities, we need to use something more powerful.

## Deployments

Deployments wrap pods with the ability to monitor and restart pods,
scale up or down, and to roll out and roll back updates.

Creating deployments is almost the same as running the pods- instead of
`kubectl run` use `kubectl create deployment`.

```shell
kubectl create deployment reviews --image docker.io/istio/examples-bookinfo-reviews-v1:1.20.3
kubectl create deployment details --image docker.io/istio/examples-bookinfo-details-v1:1.20.3
kubectl create deployment productpage --image docker.io/istio/examples-bookinfo-productpage-v1:1.20.3
kubectl get deployments
kubectl get pods
```

We can see the naming has become somewhat more involved. We will explore that
once we have the application fully running with deployments.

Also, notice that we are still using the imperative form. Not for much longer.

```shell
kubectl expose deployment reviews --port 9080 --target-port 9080
kubectl expose deployment details --port 9080 --target-port 9080
kubectl expose deployment productpage --port 9080 --target-port 9080
kubectl get services
```

Nothing different here. It looks and works just like it did with pods, just with
a different resource type.

```shell
curl -sS http://10.777.888.999:9080/productpage
```

Replace `10.777.888.999` with the `ClusterIP` for `productpage` as before. And
there still doesn't seem to be a difference.

Get the name of one of the pods (any of the three), now with a not-so-simple
name, and delete that pod.

```shell
kubectl get pods
kubectl delete pod details-644c575c58-cpxw5
kubectl get pods
```

The pod we deleted has been replaced with another pod with a similar name. In
fact, only last random string is different. Notice the `AGE` column.

Let's make sure it is still working the same before we get into what happened
behind the scenes.

```shell
kubectl get services
curl -sS http://10.777.888.999:9080/productpage
```

We didn't recreate the service- it stayed intact. And everything still works
networking-wise.

We can delete pods and let them be recreated, and we'll see that the last random
string changes each time, but the middle random string remains consistent.

```shell
kubectl get replicasets
```

That middle is the identifier from the replica set. A deployment wraps a replica
set which wraps pods. The replica set keeps a specified number of replicas
running at any given time. That number currently is 1, but we can change that.

```shell
kubectl scale deployment details --replicas=2
kubectl get deployments,replicasets,pods
```

There is still only one `deployment` and one `replicaset` each, but `details` now
has 2 `pods`. That is main function of a replica set, but we will probably
never deal with a replica set on its own. We will use it via a deployment, and
we'll check out some of the other capabilities of deployments later.

## Declarative

For now, let's look at a better way to define and manage our resources.

```shell
kubectl edit deployment productpage
```

This is a Kubernetes manifest. At first glance, we might notice that this looks
very much like the representation of the pod when we were outputting the YAML
format. The differences are that we are in an editor (`vi`) and the hierarchy is
a little deeper.

If you aren't familiar with `vi`, you can usually use the arrow keys to move
around. If they aren't working, <kbd>h</kbd> (left), <kbd>l</kbd> (right),
<kbd>j</kbd> (down), and <kbd>k</kbd> (up) almost always work.

The type of resource is defined (usually at the top) by the `apiVersion` and the
`kind`. After that, `metadata` has the details of this resource, while `spec`
encapsulates the contents of the resource. Within the `spec`, a `template` shows
the contents that will be replicated as needed.

For now, find the `replicas` field under the outermost `spec`. Change the value
from `1` to `2` (<kbd>s2</kdb><kbd>ESC</kbd>) and then save the file
(<kbd>:wq</kbd><kbd>ENTER</kbd>).

```shell
kubectl get deployments,replicasets,pods
```

It did what we would have expected, and it was actually less convenient than the
imperative form, especially if you aren't a `vi` fan. Especially since we have
access to a mouse or trackpad and a modern editor on the screen.

Much like `kubectl edit`, we can write the current state of a resource out to a
file and then edit that. There are a few extra steps, but some significant
advantages that we'll discuss and demonstrate in a moment.

```shell
kubectl get deployment reviews -o yaml > reviews.yaml
```

Double-click on `reviews.yaml` in the left-hand tool bar. Now you can use the
visual editor to change to 2 replicas and save. (An &#x2715; in the tab will
replace the white &#x9675; when the editor is saved.)

```shell
kubectl apply -f reviews.yaml
kubectl get deployment reviews
```

We'll get a warning, but the message points out that it is resolved
automatically. And we can see that the `reviews` deployment was configured.

Before anything else changes, let's apply the file again.

```shell
kubectl apply -f reviews.yaml
```

This time we get an error. The metadata in the `reviews.yaml` file says
that the last time the resource was updated, and the actual resource says it was
updated more recently (when we first applied the file). So it errors out.

Edit the file to pare it down to a smaller set of attributes.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: reviews
  name: reviews
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: reviews
  template:
    metadata:
      labels:
        app: reviews
    spec:
      containers:
      - image: docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
        name: examples-bookinfo-reviews-v1
```

Now let's apply it again.

```shell
kubectl apply -f reviews.yaml
```

This time it works without error. And let's try it one more time.

```shell
kubectl apply -f reviews.yaml
```

No error, but it did point out that nothing changed. This is **really**
important in working with Kubernetes manifests.

Firstly, this is the declarative form that we've been leading to. We specified
what we wanted the end result to be and Kubernetes made the necessary
adjustments to get us to that state. That means we can write manifests that work
in a multitude of starting states, since we are only declaring the end state.

Secondly, applying these manifests are *idempotent*. That means that if there is
nothing to do, no changes will be made. We can apply that same manifest over and
over, and unless something else changes the state, Kubernetes won't do anything
since we are already in the desired state.

`kubectl apply` is the main way we'll interact with K8s resources and workloads.

With manifests in files, we can store them in source control and easily publish
and share them. Since they are idempotent, we can automate their application
without worrying about undoing a state we just got to. And we can review those
files manually and using static analysis tools to not only let us know if we are
using the right format, but also if we are following recommended practices.

## Static analysis

[Checkov](https://www.checkov.io/) is a static analysis tool for many types of
infrastructure-as-code (IaC), including Kubernetes manifests. It has a lot of
suggestions, most well beyond our current understanding of Kubernetes resources.

To save time, it is already installed for you. But installing Checkov is just
`pip install checkov` or `pip install -U checkov` to upgrade.

```shell
checkov -f reviews.yaml
```

Most of the checks, passed and failed, are accompanied by a URL to explain the
whys and hows of the recommendation.

We might want to review the suggestions to help us better understand what we
*should* be doing. In a production system, we should probably follow all or
almost all of Checkov's suggestions. At least, we should consider them all and
make an informed decision.

## Writing manifests

Aside from capturing the YAML from existing resources and searching on the web,
Kubernetes does give us some guidance as to the attributes and their meanings.

```shell
kubectl explain deployments
kubectl explain deployments --recursive
kubectl explain deployments.spec.template.spec.containers
```

## Exercise

1. Update the `reviews` manifest file to specify the image pull policy per
   Checkov's recommendation (`CKV_K8S_15`). Apply and retest using Checkov.
2. All three services respond to health checks that can be used for
   [liveness](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/#liveness-probe)
   and
   [readiness](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/#readiness-probe)
   probes on the path `/health`. Configure liveness and readiness
   checks for `reviews` using that HTTP endpoint. Apply and retest with Checkov
   (checks `CKV_K8S_8` and `CKV_K8S_9`).

<!-- imagePullPolicy: Always -->

<!--
        livenessProbe:
          httpGet:
            path: /health
            port: 9080
        readinessProbe:
          httpGet:
            path: /health
            port: 9080
-->

## End of lesson

We will get back to the capabilities of deployments soon. But before that, we'll
do some more with Kubernetes manifests and networking in
[05-Ingress](../05-Ingress/README.md).
