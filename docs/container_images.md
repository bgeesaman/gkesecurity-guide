# Container Images

## Binary Authorization
By default, Kubernetes will pull any container images specified in the `PodSpec` as if they are guaranteed to have been uploaded by the intended person.  However, if your container registry is compromised or the API Server is tricked into pulling the wrong images in any way, this could bring in malicious containers.  One highly recommended approach is to have your container images signed and verified (with a Hash) with an accomapnying policy that enforces this at the cluster level. See [GKE Binary Authorization](https://cloud.google.com/binary-authorization/) for more information.

## Dockerfile Best Practices

Whenever possible, employ least privilege in your Dockerfile (and your container images).  There are three main surface vectors that allow an attacker to do more than they should with a pod:

1. Root Access
2. Shell Access
3. Package Managers

To minimize this risk, it's worth taking out as many of these as possible, check out [Gooogle Container Tools distroless](https://github.com/GoogleContainerTools/distroless) for references and examples.

## Base Images

## Package Vulnerabilities

## Application Vulnerabilities
