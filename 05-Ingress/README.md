# Working with Ingresses

When we looked at networking before, we focused on networking within the cluster
opening a view into an application that might be suited for development or
debugging, since you need to know which node the application was running on.

Now, we are going to look at something that might be appropriate for production.

## Load balancers

**Spoiler**: We aren't going to spend any time on load balancers. Although they
are truly the next level of networking in the Kubernetes world, after
`ClusterIP` and `NodePort`, they are often implemented outside the cluster
itself. That presents some logistical challenges, and frankly cost in a workshop
with many students.

They work operate very much like traditional load balancers, and in
Kubernetes they also can be used to make an application within the cluster
accessible to the world outside.

In turn, they have a significant overlap with an `Ingress`, which also can be
used to make an application accessible from outside the cluster. And an ingress
is often implemented within the cluster. For this reason, ingresses are a
popular alternative to load balancers in the Kubernetes world.

In our particular case, `k3s` provides a `LoadBalancer` implementation within
the cluster as well as an `Ingress`, but the `Ingress` will be more interesting
to see. It does this via an included tool called *Traefik*.

## Ingress

An ingress provides some simple routing to a service.

Create and apply `ingress.yaml`, and then examine what it produces.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookinfo-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: productpage
            port:
              number: 9080
```

```shell
kubectl apply -f ingress.yaml
kubectl get ingresses
kubectl describe ingress bookinfo-ingress
```

Note that the class is listed as `traefik`, which is the ingress controller that
is distributed with `k3s`. The ingress is assigned the IP address of the system
we are using, albeit listed as the private IP.

As configured, all traffic (path prefix '/') is sent to port 9080 on the
`productpage` service. We can try it.

**NOTE:** Because of the way the BookInfo application is written, it is
expecting the `/productpage` on the URL. That is a BookInfo quirk, not the way
our ingress is behaving.

```shell
curl -sS http://${PRIVATE_IPV4}/productpage
```

Like the NodePort configuration, ingresses work even from our neighbor's
environment. It won't work as is from your laptop since `k3s` only
knows about the private IP address. But if we had a public IP address (we do),
and the firewalls were open (they are), we could use the public IP.

```shell
curl -sS http://${PUBLIC_IPV4}/productpage
```

If we use the value for the public IP address, it works from the command line on
our laptops. Or, we can visit the same URL from a laptop browser.

On the dev environment:

```shell
echo ${PUBLIC_IPV4}
```

Then on the laptop, use `http://111.222.333.444/productpage`, replacing
`111.222.333.444` with the value of our public IP.

Back in the first lesson, we discussed that could use a domain name if we mapped
it to the public IP address (which we did). Try
`http://william.techbash.otherdevopsgene.dev/productpage`. Replace `william`
with the username you used to login to AWS and use `.work` or `.xyz` as the
top-level domain if you didn't use `.dev`. If you get an error, your browser is
probably protecting you by *fixing* the URL to use `https`. If you get a warning
that your connection is not private, your browser is *definitely* protecting you
by using `https`.

Turns out, we can make that work, too.

## Namespaces

Because Kubernetes is a well-defined interface, people can write code that
works with that framework and API and adds significant capability, without
having to know which implementation of Kubernetes you are using (usually).

Also, `kubectl apply` can take a URL rather than just a filename. Since the
declarative form doesn't expect a certain starting state, we can use a manifest
for a third-party plugin to install itself on our cluster.

```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.1/cert-manager.yaml
```

That isn't a single resource, but a file with many resources in it. A lot of the
types are new to us, but there are some services at the end, so let's start by
looking at them.

```shell
kubectl get services
```

Probably not what we expected. The `kubectl apply` said it created
`cert-manager-cainjector`, `cert-manager`, and `cert-manager-webhook`, but they
aren't there. So where did they go?

Well, the first line of the `kubectl apply` said it created
`namespace/cert-manager`, so let's try `namespaces`.

```shell
kubectl get namespaces
```

We can see there are a few, most of which have been around since we started
(based on their age), and then the one new namespace called `cert-manager`.

Namespaces are a way to logically segment applications and resources. Until now,
we've been working in the default namespace, conveniently called `default`. But
almost every command we've used so far can also take a `--namespace` option.

```shell
kubectl get services,deployments,pods --namespace cert-manager
```

We could also look across all namespaces with the `--all-namespaces` or `-A`
option.

```shell
kubectl get pods -A
```

We can see the new pods that `cert-manager` installed, the pods in the `default`
namespace that we've been creating, and some parts of the system that `k3s`
brought in to start in the `kube-system` namespace, including the `traefik`
ingress controller we worked with above.

Now we need a certificate. And we'll see some new resource types on the way.

Create and apply `cluster-issuer.yaml`, making sure to replace the email address
appropriately.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cert
spec:
  acme:
    email: otherdevopsgene@portinfo.com # Replace with your email
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-cert-private-key
    solvers:
      - http01:
          ingress:
            class: traefik
```

Notice from the `apiVersion` that this is not a standard Kubernetes resource.
This is a custom resource that `cert-manager` defined, specifically
`customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io`.
Custom resource definitions, or CRDs, are a way to extend the Kubernetes API. We
won't get into writing a CRD, but obviously we are starting to use them.

```shell
kubectl apply -f cluster-issuer.yaml
kubectl get clusterissuer
kubectl describe clusterissuer
```

The `ClusterIssuer` doesn't live in a namespace because it is cluster-wide. If
you need it, though, there is a version called an `Issuer` that is
namespace-specific.

Aside from the information we supplied, we can see some status that says we
registered with the ACME server, which is the certificate authority called
[Let's Encrypt](https://letsencrypt.org/).

## Secrets

Some other resources got created as well. Let's look into one in particular.

```shell
kubectl get secrets -n cert-manager
kubectl describe secret -n cert-manager letsencrypt-cert-private-key
```

It is called a secret and has one piece of data (therefore, technically a datum)
called `tls.key`. If you know anything about how certificates work, you might
recognize that as the private key that keeps a certificate secure.
Unfortunately, despite the name, a Kubernetes `secret` is not actually secret,
at least not by default. It is just a collection of Base64-encoded name-value
pairs.

Try either (or both) of:

```shell
kubectl get secret -n cert-manager letsencrypt-cert-private-key --template="{{index .data \"tls.key\"}}" | base64 -d
kubectl get secret -n cert-manager letsencrypt-cert-private-key -o json | jq '.data | map_values(@base64d)'
```

The complexity around the Go template is because the single name-value pair
`tls.key` has a dot in the name, so `--template={{.data.tls.key}}` would look
like an attribute named `key` further in the hierarchy.

But the point is that this secret is just encoded, not encrypted, so is obscured
from casual viewing, but not secret at all. There are ways to change that, but
we won't have time to explore further. Just remember, without some changes, a
`secret` isn't secret.

A `ConfigSet` is almost the same thing, without pretending to be secret. Just a
name-value pair store.

## TLS traffic

Update and apply `ingress.yaml` to know about our `cert-manager` and more
importantly our `ClusterIssuer` which will handle getting certificates from
Let's Encrypt for us. The ingress also needs to know what fully-qualified domain
name (FQDN) to use.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookinfo-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cert # name from cluster-issuer
spec:
  rules:
  - host: william.techbash.otherdevopsgene.dev # replace william with your username
    # or william.techbash.otherdevopsgene.work or william.techbash.otherdevopsgene.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: productpage
            port:
              number: 9080
  tls:
  - hosts:
    - william.techbash.otherdevopsgene.dev # match the host above
    secretName: acme-tls-cert
```

To get around some Let's Encrypt rate limitations, we might have to spread our
certificate requests around several domains:

- `otherdevopsgene.dev`
- `otherdevopsgene.work`
- `otherdevopsgene.xyz`

```shell
kubectl apply -f ingress.yaml
kubectl describe ingress bookinfo-ingress
```

We can see that the certificate was successfully created in the `Events`
section.

Notice that the `TLS` section says that `acme-tls-cert` (our certificate)
terminates our FQDN. That means that once the TLS-encrypted traffic arrives at
our ingress, the ingress will pass unencrypted, plain HTTP traffic on to the
service in our cluster. So our service (or services) doesn't need to handle TLS
traffic, but external users still get the protections of TLS.

There are other options for protecting the internal traffic and ways for Traefik
to not terminate if your use case calls for those. It is not something we are
going to cover.

Let's take a quick look at the certificate.

```shell
kubectl get secrets
kubectl describe secrets acme-tls-cert
```

This time, our secret shows two entries- a public key (`tls.crt`) and a private
key (`tls.key`). We could decode these if we wanted, but the important part here
is that the certificate manager automatically requested and installed them.

This is an understated value of using a well-defined framework like Kubernetes.
We installed `cert-manager` and pointed it to Let's Encrypt as the certificate
authority (CA). We redeployed our ingress to include a hostname and the
configuration (`cluster-issuer`) we wanted to use. We generated a certificate
request, requested a cert with the CA, received and installed the certificate,
set up the ingress for TLS termination. That's a lot for not much work on our
part.

## HTTP redirection

All that's left to make this configuration really useable is to make `http`
redirect to `https`.

On your laptop, check the current behavior.

```shell
curl --head -sS http://william.techbash.otherdevopsgene.dev/productpage
```

Notice the `200 OK`.

Now we can configure our ingress. This step happens to be Traefik-specific, as
we'll see from the `apiVersion`.

Install a few CRDs we'll need for Traefik:

```shell
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v3.7/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
```

Then, create and apply `middleware.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect
  namespace: default
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

```shell
kubectl apply -f middleware.yaml
```

The naming format for Traefik to reference the middleware we created is
`namespace-middlewarename@kubernetescrd`. Add an annotation to our ingress
configuration:

```yaml
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd
```

And apply.

```shell
kubectl apply -f ingress.yaml
```

And let's test from our laptops again.

```shell
curl --head -sS http://william.techbash.otherdevopsgene.dev/productpage
```

Notice the `308 Permanent Redirect`

```shell
curl --head -sS https://william.techbash.otherdevopsgene.dev/productpage
```

Works as expected. We can confirm the behavior with our web browsers, too.

## Exercise

Modify the `bookinfo-ingress` to allow access to the `details` and `reviews`
services from outside the cluster using `https`.

Note that the `reviews` service requires a book ID (e.g., `/2`) or it will
return a **404 Not Found** error. Again, this is a BookInfo behavior.

## End of lesson

Now, back to deployments in
[06-Rollouts](../06-Rollouts/README.md).
