# Container Images

## Dockerfile Best Practices

Whenever possible, employ least privilege in your Dockerfile (and your container images).  There are three main surface vectors that allow an attacker to do more than they should with a pod:

1. Root Access
2. Shell Access
3. Package Managers

To minimize this risk, it's worth taking out as many of these as possible, check out [Gooogle Container Tools distroless](https://github.com/GoogleContainerTools/distroless) for references and examples.

## Base Images

## Package Vulnerabilities

## Application Vulnerabilities
