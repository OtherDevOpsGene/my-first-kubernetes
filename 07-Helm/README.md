# Helm

Most of the time we will use `kubectl apply` to declare and implement the
desired state for our cluster. But if we are using third-party code, often we'll
use a package manager called [Helm](https://helm.sh/) to allow for more flexible
deployments in various environments. With vanilla manifests, even remote ones,
especially remote ones, all the settings are hard-coded.

Helm v4 is preinstalled, but if you needed to
[install Helm](https://helm.sh/docs/intro/install/) elsewhere you could install
it with the following.

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

## Using Helm to install workflow

We will use Helm to install [Kiali](https://kiali.io/), a visualization tool
usually used with [Istio](https://istio.io/).

There are just a few commands we need to install using Helm, but in practice
there are a lot of repeated names in the commands so it might be helpful to
look at them argument-by-argument.

- `helm repo add kiali https://kiali.org/helm-charts`

  This adds the repository to our system so Helm knows about it and the charts
  it contains.

  - `helm repo add`- the command
  - `kiali`- **repo name**
  - `https://kiali.org/helm-charts`- repo URL

  We only need to do this once.

- `helm repo update`

  This downloads the latest version of all the repositories on our system so
  Helm has access to the latest charts in each. We will do this whenever we plan
  to use the repository after a few days have passed. No harm in doing it more
  frequently.

- `helm install --namespace istio-system --create-namespace kiali-server kiali/kiali-server`

  This command follows the chart instructions to install an application or
  service or whatever on our system. This command is the whole point of using
  Helm in the first place.

  - `helm install`- the command
  - `--namespace istio-system`- install into the `istio-system` namespace
  - `--create-namespace`- create the namespace if it doesn't exist
  - `kiali-server`- **release name**
  - `kiali/kiali-server`- **repo name**/**chart name**

  The release name is a name we give to the results of the chart installing
  whatever it installs. Nobody sees it but us. But we'll use it constantly.

  This command finishes when the chart is installed. There might be a lot still
  going on behind the scenes, acting on the chart commands. Some systems are
  ready in seconds, some in minutes, and some charts you should kick off
  installing before going to lunch.

- `helm status kiali-server --namespace istio-system`

  This looks like a great way to find out what it going on behind the scenes. It
  is not. It is really for reminding us what we did yesterday before we quit for
  the evening, or maybe reminding us if we installed last week or just planned
  to install.

  I only mention it here to save you from false hope when we want to understand
  if the installation is done.

  - `helm status`- the command
  - `kiali-server`- **release name**
  - `--namespace istio-system`- we installed into a specific namespace, so we
    need to specify that anytime we refer to the release

## Port forwarding

We installed Kiali using Helm to get a chance to try Helm. Even though it looks
like things worked, we should look at Kiali itself to see if it is useable.

Kiali is accessible as a web application once it is installed, but as we
remember from [05-Ingress](../05-Ingress/README.md), there are things we have to
do to make it available from outside the cluster.

Before we get that far, though, we can check if we can access it from within the
cluster. For that, we can try `kubectl port-forward`.

Open a new terminal using <kbd>Alt</kbd>-<kbd>T</kbd>, and in that window start
the port forwarding. First, we'll need to know where the service is listening.

```shell
kubectl get services -n istio-system
kubectl port-forward service/kiali 15001:20001 -n istio-system
```

That will forward traffic sent to `localhost` on port `15001` to the `kiali`
service on port `20001`. Leave that running.

Remembering back to our experiences in
[03-Networking](../03-Networking/README.md), we know that means *within the
cluster* includes from our development environment. So back in the original
terminal, see if we can access Kiali from the command line.

```console
$ curl -sS http://localhost:15001/
<a href="/kiali/">Found</a>.
```

Success-ish. Since we don't have access to a UI-based browser here, it doesn't
provide any real value to us other than to prove that we can forward traffic
temporarily without going through the trouble of setting up an ingress. This is
often helpful for troubleshooting or for accessing administration or
configuration consoles on an infrequent basis. Oh, it also shows us that Kiali
was installed and is running.

The port forwarding will remain active until we kill it by closing the terminal
or using <kdb>Ctrl</kdb>-<kbd>C</kbd>.

## Kiali

We are practically experts when it comes to setting up an ingress, having done
it successfully once. And we can even add one with `https` access. We need
`Middleware` for Traefik and a new `Ingress` for Kiali, and we'll be running it
from the `istio-system` namespace. The `ClusterIssuer` was cluster-wide, so we
can reuse that.

Create and apply `kiali-ingress.yaml`, making sure to apply it to the correct
namespace.

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect
  namespace: istio-system
spec:
  redirectScheme:
    scheme: https
    permanent: true
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kiali-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cert
    traefik.ingress.kubernetes.io/router.middlewares: istio-system-redirect@kubernetescrd
spec:
  rules:
  - host: william.techbash.otherdevopsgene.dev # replace william with your username
    # or william.techbash.otherdevopsgene.work or william.techbash.otherdevopsgene.xyz
    http:
      paths:
      - path: /kiali
        pathType: Prefix
        backend:
          service:
            name: kiali
            port:
              number: 20001
  tls:
  - hosts:
    - william.techbash.otherdevopsgene.dev # match the host above
    secretName: acme-tls-cert
```

```shell
kubectl apply -f kiali-ingress.yaml -n istio-system
```

This manifest file is different than the others we've created and looked at. It
has multiple YAML documents, separated by three dashes (`---`) that signal the
start of each one. Technically, we could have been leading off our manifests
with them, but they haven't been needed until now. To be honest, we could skip
the first separator now, too, but it helps remind us that this is a multiple
document YAML file.

We added the namespace to the `kubectl apply` command. We could have used
`--namespace istio-system` to be clearer, but namespaces are so common that we
almost always shorten the flag to `-n` to save typing.

Along with that, we can see all the ingresses (both of them) across the
namespaces by querying `--all-namespaces`, or `-A` for short.

```shell
kubectl get ingress -A
```

And we can now reach Kiali from our laptop at
`https://william.techbash.otherdevopsgene.dev/kiali`, replacing `william` with
your username as we did earlier with our first ingress and using `.work` or
`.xyz` as the top-level domain if we didn't use `.dev`.

## Creating a token

And we are stopped at the front door by Kiali asking for a token, aka a service
account token. That is a string representing us as a particular user, or service
account. Once we prove who we are, Kiali will let us in.

The Kiali documentation describes [how to obtain a token when logging in via
token auth
strategy](https://kiali.io/docs/faq/authentication/#how-to-obtain-a-token-when-logging-in-via-token-auth-strategy).
And when we try that, we see an error.

```console
$ kubectl -n istio-system create token kiali-service-account
error: failed to create token: serviceaccounts "kiali-service-account" not found
```

*Notice* that the namespace can go almost anywhere in the command. Earlier is
often better for the shell to help with command completion.

So the docs and the Helm chart aren't 100% in sync. The crux is that we need
to know the name of the service account for the token. But there is enough
context in the error message to point us in the right direction.

```shell
kubectl -n istio-system get serviceaccounts
```

And from the output of that, we can guess that the service account name is
simply `kiali`. Therefore the token can be created with:

```shell
kubectl -n istio-system create token kiali
```

Copy-and-pasting that long string gets us into Kiali, although there are a few
scattered errors about `prometheus.istio-system` not being found.

[Prometheus](https://prometheus.io/) is a very popular metrics collector. We can
look for a Helm chart to install that as well by searching Google or search the
Artifact Hub for Helm.

```shell
helm search hub prometheus
```

That returns us way more options than we could sort through. The first one
happens to be the one we want, but we also need the URL for the repo so we can
add it.

```console
$ helm search hub prometheus --list-repo-url | head -2
URL                                                     CHART VERSION           APP VERSION             DESCRIPTION                                             REPO URL                                          
https://artifacthub.io/packages/helm/prometheus...      29.27.1                 v3.14.0                 Prometheus is a monitoring system and time seri...      https://prometheus-community.github.io/helm-charts
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories
```

Now we can see the repository has been added and then we can search the
`prometheus-community` repo for the right chart.

```shell
helm repo list
helm repo update
helm search repo prometheus-community
```

Again, an unwieldy list. In many cases, searching with Google will give more
information about choosing the appropriate Helm charts than Helm can itself. In
our case, we need the `prometheus` chart (from the `prometheus-community` repo).

## Finding Helm charts and tools the easy way

It is nice of Helm to get us some search tools, but I don't think I have ever
used them and actually found what I was looking for.

In my experience, The best way to find the Helm chart for a particular tool is a
generic Internet search (e.g., `prometheus helm chart`). Sometimes, there are
many options of various countenance and differing quality. But most of the time,
there is a vendor or community-supported official or official-ish Helm chart to
use.

When it comes to finding a tool that can be installed with Helm, I usually turn
to the [Cloud Native Landscape](https://landscape.cncf.io/). Not every tool on
that list is a Kubernetes tool, but most are and almost any Kubernetes tool you
need will be listed there. And they almost always have a Helm chart.

## Exercise

1. Use Helm to install Prometheus in the `istio-system` namespace.
2. Update our `kiali-ingress` to send `/prometheus` traffic to port `80` on
   the `prometheus` service.

<!-- helm install prometheus prometheus-community/prometheus -n istio-system -f values.html -->

<!--  
      - path: /prometheus
        pathType: Prefix
        backend:
          service:
            name: prometheus-server
            port:
              number: 80
-->

## Aftermath

While that was good practice using Helm, our little cluster might not be big
enough to handle everything we've thrown at it. When we run out of memory on our
Cloud9 instance, some of the pods we are using are now or will very soon be
`Evicted`, kicked out for something the Kubernetes scheduler thinks is more
important.

Fixing the Prometheus installation would be the right way forward, but we would
need a bigger cluster to do that. Instead, let's use Helm to undo the damage by
uninstalling Prometheus and Kiali. Or at least to practice some more commands.

```shell
helm uninstall prometheus -n istio-system
helm uninstall kiali-server -n istio-system
```

We might have overwhelmed our cluster so much that it won't fix itself even
then, at least not right away. But this makes a good stopping point.

## End of lesson
