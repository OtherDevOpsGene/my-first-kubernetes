# Rollouts and rollbacks

We mentioned earlier that there were more capabilities of deployments other than
just scaling. So now we'll take a look at how Kubernetes deployments handle
*rollout* and *rollback*.

## Failed deployments

Until now, we've been ignoring the `ratings` microservice in the **BookInfo**
application.

<img src="../bookinfo-basic.svg">

Let's start with `ratings` by seeing what a common failed deployment would look
like. We'll use the wrong image.

Create and apply `ratings.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ratings
  name: ratings
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ratings
  template:
    metadata:
      labels:
        app: ratings
    spec:
      containers:
      - image: wrongratings
        name: wrongratings
```

```shell
kubectl apply -f ratings.yaml
kubectl get pods
```

After a short while, we'll see that the pod has failed to run.

```console
$ kubectl get pods -l app=ratings
NAME                       READY   STATUS             RESTARTS   AGE
ratings-598854949c-gq45f   0/1     ImagePullBackOff   0          63s
```

We have 0 of 1 containers ready and the status is `ImagePullBackOff`. Let's look
at some detail for more information.

```shell
kubectl describe pod -l app=ratings
```

Towards the end we can see the failed events that led to the `ImagePullBackOff`
status.

```console
$ kubectl describe pod -l app=ratings
Name:             ratings-598854949c-bk77t
Namespace:        default
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  74s                default-scheduler  Successfully assigned default/ratings-598854949c-bk77t to ip-172-31-0-195.us-east-2.compute.internal
  Normal   Pulling    36s (x3 over 74s)  kubelet            Pulling image "wrongratings"
  Warning  Failed     36s (x3 over 74s)  kubelet            Failed to pull image "wrongratings": failed to pull and unpack image "docker.io/library/wrongratings:latest": failed to resolve reference "docker.io/library/wrongratings:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     36s (x3 over 74s)  kubelet            Error: ErrImagePull
  Normal   BackOff    1s (x5 over 74s)   kubelet            Back-off pulling image "wrongratings"
  Warning  Failed     1s (x5 over 74s)   kubelet            Error: ImagePullBackOff
```

Not surprising, since we arranged the situation deliberately, the cluster
couldn't pull the non-existent image we specified, then gave us an `ErrImagePull`
and eventually `ImagePullBackOff` once it gave up trying.

The key lesson is to check the `kubectl describe` and look at what happened if
the pods aren't running. More often than not the `ImagePullBackOff` is a
misnamed image, perhaps a permissions issue when dealing with a private image
registry.

The fix is simple. In `ratings.yaml`, update the image to
`docker.io/istio/examples-bookinfo-ratings-v1:1.20.3` (and preferably update
the `wrongratings` name accordingly).

Because it is a deployment, as soon as we apply the changed manifest, Kubernetes
will orchestrate a *rollout*. It has always been doing that for us, we just
haven't been paying attention.

```shell
kubectl apply -f ratings.yaml
kubectl rollout status deployment ratings
```

A rollout of one replica, replacing a deployment that isn't even running, is not
very eventful. On top of that, version 1 of `reviews` doesn't use `ratings`. So
let's do some rollouts we can see.

## Rollouts

We'll start by exposing `ratings`, this time using declarative form.

Create and apply `ratings-service.yaml`.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ratings
  name: ratings
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: ratings
  ports:
  - port: 9080
    protocol: TCP
```

```shell
kubectl apply -f ratings-service.yaml
```

Next, increase both `reviews` to 4 replicas and `ratings` to 2 replicas in their
respective manifests and apply those changes.

```shell
kubectl apply -f reviews.yaml
kubectl apply -f ratings.yaml
kubectl rollout status deployment reviews ratings
```

After a few seconds, success.

At this point, we should load up the page in our browser
(`https://william.techbash.otherdevopsgene.dev/productpage`) so we can remind
ourselves what version 1 of `reviews` looks like.

Next, in the `reviews` manifest, switch the `image` and `name` to `v2`:

```yaml
    spec:
      containers:
      - image: docker.io/istio/examples-bookinfo-reviews-v2:1.20.3
        name: examples-bookinfo-reviews-v2
```

We'll apply it and check the rollout status in a chained command to increase the
chances of seeing the behavior during the switch. It might not help, but it
might.

```shell
kubectl apply -f reviews.yaml && kubectl rollout status deployment reviews
```

If we reload the page in our browser, we should see some block star ratings
about the reviews.

We can update further to version 3 to see red stars.

```yaml
    spec:
      containers:
      - image: docker.io/istio/examples-bookinfo-reviews-v3:1.20.3
        name: examples-bookinfo-reviews-v3
```

```shell
kubectl apply -f reviews.yaml && kubectl rollout status deployment reviews
```

Let's go one more to version 4.

```yaml
    spec:
      containers:
      - image: docker.io/istio/examples-bookinfo-reviews-v4:1.20.3
        name: examples-bookinfo-reviews-v4
```

```shell
kubectl apply -f reviews.yaml && kubectl rollout status deployment reviews
```

This one gets stuck part way. Use <kbd>Ctrl</kbd>-<kbd>C</kbd> to exit the
`kubectl rollout status` command and see what the problem might be.

## Exercise

1. Find out why the rollout to version 4 of `reviews` stalled.
2. Use `kubectl rollout -h` to find the command to rollback to a working
   state with red star reviews.
3. Use similar command or commands to get us back to black star ratings.

<!-- 
  kubectl get pods -l app=reviews
  kubectl describe pod reviews-f78bc974c-hf266
  ImagePullBackOff, there is no version 4 of the image
-->

<!-- kubectl rollout undo deployment reviews -->

<!-- 
  kubectl rollout history deployment reviews
  kubectl rollout undo deployment reviews --to-revision=2
-->

## End of lesson

We will work on installing some new tools in the next lesson,
[07-Helm](../07-Helm/README.md).
