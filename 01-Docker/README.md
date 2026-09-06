# Running a microservices app with Docker

Nothing requires us to use an orchestration framework. We could use vanilla
Docker to deploy our application.

## About our environment

This isn't specific to Kubernetes or Docker at all, but we will need to know
some information about the development environment we are working from. Since we
are on AWS, we can ask the local AWS Metadata Service for information.

```shell
METADATA_TOKEN=$(curl -sS -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -sSH "X-aws-ec2-metadata-token: ${METADATA_TOKEN}" http://169.254.169.254/latest/meta-data/
```

That `curl` request will show us list of the information we can ask for. We need
the private (aka local) and public IP addresses.

```shell
PRIVATE_IPV4=$(curl -sSH "X-aws-ec2-metadata-token: ${METADATA_TOKEN}" http://169.254.169.254/latest/meta-data/local-ipv4)
echo PRIVATE_IPV4 = ${PRIVATE_IPV4}
PUBLIC_IPV4=$(curl -sSH "X-aws-ec2-metadata-token: ${METADATA_TOKEN}" http://169.254.169.254/latest/meta-data/public-ipv4)
echo PUBLIC_IPV4 = ${PUBLIC_IPV4}
```

Now we can get back to the problem at hand.

## BookInfo

For example, let's consider the fairly simple application shown below, known as
**BookInfo**.

<img src="../bookinfo-basic.svg">

Requests come into `productpage`. That service makes requests to `reviews` and
`details` to gather the information to display. We'll ignore `ratings` for now.

The authors of `productpage` allowed for us to configure the networking via
environment variables. Kubernetes calls this *service discovery*, which we are
doing manually.

```shell
docker run --rm -d -p 9081:9080 docker.io/istio/examples-bookinfo-reviews-v1:1.20.3
docker run --rm -d -p 9082:9080 docker.io/istio/examples-bookinfo-details-v1:1.20.3
docker run --rm -d -e REVIEWS_HOSTNAME=${PRIVATE_IPV4} -e REVIEWS_SERVICE_PORT=9081 \
  -e DETAILS_HOSTNAME=${PRIVATE_IPV4} -e DETAILS_SERVICE_PORT=9082 \
  -p 8080:9080 docker.io/istio/examples-bookinfo-productpage-v1:1.20.3
```

We exposed the `productpage` service on port 8080 to avoid permissions problems
by running on a privileged port (i.e., under 1024) and to coincide with a hole
we have in the security group for our instance.

So we can look at the application locally:

```shell
curl http://${PRIVATE_IPV4}:8080/productpage
```

Or we can use the public IP address to see the application from our local
browser at `http://111.222.333.444:8080/productpage` (note the URL uses `http`,
not `https`). Replace `111.222.333.444` with your public IP address.

If we mapped a domain name to the public IP address (which we did), we could use
that instead. We can try with
`http://username.techbash.otherdevopsgene.dev:8080/productpage`. Replace
`username` with the username you used to login to AWS. If you get an error, your
browser is probably protecting you by *fixing* the URL to use `https`.

To fix that, we'd need to use `https` and that means setting up a certificate,
preferably one that is signed by someone that our browser trusts. Too much work
for now.

Let's clean up and move on to Kubernetes.

## Clean up

```shell
docker stop $(docker ps -q)
docker ps
```

## End of lesson

We will recreate this with Kubernetes in the next lesson,
[02-Basics](../02-Basics/README.md).
