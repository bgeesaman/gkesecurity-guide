# Container Images

## Binary Authorization
By default, Kubernetes will pull any container images specified in the `PodSpec`'s `image:` as long as Kubernetes has access, and often the only identifier is the registry URL as well as the common name of the image.  If a cluster is compromised and the attacker has access to api-resources like the `Pod` or `Deployment` resouces, they could potentially pull unknown or malicious images into the environment and further elevate their privileges.  Additionally, an attacker could gain access to your container registry and upload a similarly named container (e.g. `my-usual-api-image-name`) and your Deployment could pull this container image on creation without it having authorization to run in your environment. Both of these threat vectors introduce the need for having your container images signed and verified (with a hash) and then applying an accompanying policy that enforces this at the cluster-level (any time a new image is requested). Binary Authorization uses a system of attestations (validity of the hash digest of the image) and enforcements (verifying attestations match up with a policy provided) to help accomplish this. See [GKE Binary Authorization](https://cloud.google.com/binary-authorization/) for more information.

## Dockerfile Best Practices

Whenever possible, employ least privilege in your Dockerfile (and your container images).  There are three main surface vectors that allow an attacker to do more than they should with a pod:

1. Root Access
2. Shell Access
3. Package Managers

To minimize this risk, it's worth taking out as many of these as possible, check out [Gooogle Container Tools distroless](https://github.com/GoogleContainerTools/distroless) for references and examples.

## Base Images

## Package Vulnerabilities

## Application Vulnerabilities
